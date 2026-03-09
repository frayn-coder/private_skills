---
name: warehouse-emergency
description: "Handle warehouse emergency situations such as fire, flood, power outage, or facility closure. Use this skill when the user reports that a warehouse is unavailable, damaged, or unsafe, and needs to reroute pending orders, find alternative stock sources, and notify affected parties. Trigger words: 仓库失火, 仓库关闭, 仓库事故, warehouse fire, emergency, facility unavailable, 紧急情况."
metadata:
  {
    "copaw":
      {
        "emoji": "🔥",
        "requires": {}
      }
  }
---

# 仓库紧急情况处理

## 概述

当某个仓库因火灾、洪水、断电、事故或其他原因无法正常运作时，执行本 skill 以将影响降到最低。目标：
1. 立即关闭受影响的仓库
2. 找出所有受影响的在途/待处理订单
3. 将订单切换到备用仓库
4. 标记无法自动处理的异常，提交人工审核

> **前置条件**：供应链 API 服务需已启动（`python server.py`），默认监听 `http://localhost:8765`

---

## 何时触发

满足以下任意一条时激活本 skill：
- 用户提及某仓库"失火"、"洪水"、"停工"、"关闭"、"不可用"
- 用户问"某仓库出事了，怎么办"
- 物流监控系统报警：仓库失联超过 30 分钟

---

## 执行步骤

### Step 1 — 确认受影响仓库

如果用户只说了城市或仓库名，先查询完整的仓库代码：

```bash
curl -s "http://localhost:8765/warehouses"
```

在返回的列表中找到对应的 `code`（如 `WH-BJ-01`）。
向用户确认："您是指 **北京朝阳仓 (WH-BJ-01)** 吗？"，得到确认后继续。

---

### Step 2 — 执行一键应急处理

```bash
curl -s -X POST "http://localhost:8765/workflows/warehouse-emergency" \
     -H "Content-Type: application/json" \
     -d "{\"warehouse_code\": \"WH-BJ-01\", \"reason\": \"库房失火\"}"
```

此接口会自动完成：
- 将仓库状态改为 `closed`
- 扫描全部状态为 `pending / confirmed / picked` 的关联订单
- 对每个订单，找有足够库存的备选仓并自动切换

---

### Step 3 — 解读并汇报结果

从返回 JSON 的 `data` 字段提取关键信息：

```json
{
  "ok": true,
  "data": {
    "closed_warehouse": "WH-BJ-01",
    "closure_reason": "库房失火",
    "total_affected_orders": 3,
    "rerouting_results": [
      {"so_number": "SO-20260306-006", "success": true,  "new_warehouse": "WH-SH-01"},
      {"so_number": "SO-20260307-007", "success": false, "reason": "No single warehouse has sufficient stock"}
    ]
  }
}
```

向用户汇报：
```
✅ 已自动切仓：N 个订单
⚠️ 需人工处理：M 个订单（库存不足，无法自动切仓）
```

---

### Step 4 — 手动处理无法自动切仓的订单

对 `success: false` 的订单，先查看订单详情：

```bash
curl -s "http://localhost:8765/sales-orders/SO-20260307-007"
```

再查询每个商品的全网备货情况：

```bash
curl -s "http://localhost:8765/inventory/alternatives?sku=SKU-E005&qty=40&exclude=WH-BJ-01"
```

根据结果告诉用户哪个 SKU 在哪个仓库有库存，建议拆单或部分发货。
手动切仓：

```bash
curl -s -X PATCH "http://localhost:8765/sales-orders/SO-20260307-007/warehouse" \
     -H "Content-Type: application/json" \
     -d "{\"warehouse_code\": \"WH-GZ-01\", \"reason\": \"北京仓失火，手动切仓\"}"
```

---

### Step 5 — 配送记录更新

若受影响订单已有配送记录（`preparing` 状态），需中止：

```bash
curl -s -X PATCH "http://localhost:8765/shipments/SFXXXXXXXXXX/status" \
     -H "Content-Type: application/json" \
     -d "{\"status\": \"failed\", \"notes\": \"仓库紧急关闭，配送中止\"}"
```

---

### Step 6 — 汇总输出（必须）

最终向用户输出：

```
📍 受影响仓库：北京朝阳仓 (WH-BJ-01)
⚡ 紧急原因：库房失火
📦 受影响订单总数：X 个
✅ 已自动切仓：X 个（已转至上海浦东仓）
⚠️ 需人工介入：X 个
  - SO-20260307-007：SKU-E005 全网库存不足，建议紧急采购
📞 建议下一步：联系受影响客户告知延迟，并启动紧急补货流程
```

---

## 边界情况

| 情况 | 处理方式 |
|------|----------|
| API 服务未启动 | 提示用户先运行 `python server.py`（在 scope 目录下） |
| 仓库代码不存在 | 调用 `GET /warehouses` 列出所有仓库让用户选择 |
| 所有备用仓均无货 | 报告全部缺货，建议调用 low-stock-replenishment skill |
| 订单已是 `shipped/delivered` | 无需处理，说明已在途 |
| 恢复仓库 | `PATCH /warehouses/WH-BJ-01/status` with `{"status": "operational"}` |

---

## API 快查

| 动作 | 命令 |
|------|------|
| 查询所有仓库 | `GET /warehouses` |
| 仓库应急处理 | `POST /workflows/warehouse-emergency` |
| 查看订单详情 | `GET /sales-orders/{so_number}` |
| 手动切仓 | `PATCH /sales-orders/{so_number}/warehouse` |
| 查找备用仓 | `GET /inventory/alternatives?sku=X&qty=N&exclude=WH-XX` |
| 更新配送状态 | `PATCH /shipments/{tracking}/status` |
| 仪表盘总览 | `GET /dashboard` |
