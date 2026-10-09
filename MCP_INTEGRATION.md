# MCP 集成说明

## 用到的麦当劳 MCP Server

- 接入地址：`https://mcp.mcd.cn`
- 传输协议：Streamable HTTP
- 鉴权：请求头 `Authorization: Bearer <MCP Token>`
- 限流：每 Token 每分钟 600 次，超限返回 429

本项目实际调用 6 个 Tool，全部来自麦当劳中国官方 MCP Server。

## 实际使用的 Tool

| Tool | 用途 | 是否关键 |
|---|---|---|
| `query-nearby-stores` | 按城市与位置定位门店，取 `storeCode` | 关键前置 |
| `query-meals` | 拉取该门店当前真实在售菜单、分类与价格 | 关键 |
| `query-meal-detail` | 拉取餐品配料轮次与特调能力 | **核心** |
| `list-nutrition-foods` | 能量、蛋白质、脂肪、碳水、钠、钙 | 关键 |
| `calculate-price` | 计算特调后的实际价格 | 可选 |
| `create-order` | 用户明确要求时才下单 | 可选 |

核心洞察：整个项目建立在 `query-meal-detail` 返回的 `modification` 结构上。
该字段把配料拆到可增减的粒度，并给出保留/去重的 key，这是把「饮食限制」翻译成「可执行改动」的唯一数据来源。

## 调用流程

```
① query-nearby-stores(searchType=2, beType=1, city, keyword)
      └─ 取 storeCode

② query-meals(storeCode, orderType=1, beType=1)
      └─ 取菜单：code / name / currentPrice / tags
      └─ 按 references/constraint-db.md 粗筛出候选餐品

③ 对每个候选餐品：
   query-meal-detail(storeCode, orderType=1, beType=1, code)
      └─ 读 rounds[].choices[].modification.items[].values[]
      └─ 命中限制的配料 → 判定为 ⚠️ 或 ⛔

④ list-nutrition-foods()
      └─ 补能量/钠/蛋白，用于控钠与控热量排序

⑤ calculate-price(storeCode, orderType=1, beType=1, items[].modification)
      └─ 用户确认方案后，算改动后价格
      └─ 注意：含 unselectedKey 的特调组，每一项都要传

⑥ create-order(...)
      └─ 仅用户明确要求时
```

## 特调传参规则

含 `unselectedKey` 的特调组，该组所有配料都必须传入：

| 配料状态 | key |
|---|---|
| 保留 | `selectedKey` |
| 去掉 | `unselectedKey` |

以「巨无霸去吉士」为例，五项配料全传，吉士用 `unselectedKey`：

```json
{
  "code": "1100",
  "quantity": 1,
  "modification": { "values": [
    { "code": "100116", "key": "0-1", "quantity": 1 },
    { "code": "100141", "key": "0-0", "quantity": 1 },
    { "code": "100168", "key": "0-1", "quantity": 1 },
    { "code": "100202", "key": "0-1", "quantity": 1 },
    { "code": "100212", "key": "0-1", "quantity": 1 }
  ]}
}
```

漏传任一项，后端会按默认全配计算，用户拿到的餐品与方案不符。
套餐通过 `items[].roundList` 传，`round` 为轮次名（如 `选择套餐内饮料`）。

## 业务价值

饮食限制在菜单面前是**不可见**的。用户在菜单上看到「巨无霸」，无法知道它含吉士、酸黄瓜、洋葱粒，
也无法知道去掉吉士后价格是否变化。本项目把这个不可见的过程变成可见的改动清单。

具体价值：

- **过敏人群**：把「这道能不能吃」变成「删掉哪三样」。
- **宗教饮食**：明确告知哪些改动可行，并提示需向门店确认认证状态。
- **控钠控热量人群**：从 160 项营养数据里筛出低钠低热量组合。
- **门店视角**：特调能力被整理成结构化输出，减少沟通成本。

## 数据边界（如实声明）

`list-nutrition-foods` 返回字段为：

```
productName, nutritionDescription, energyKj, energyKcal,
protein, fat, carbohydrate, sodium, calcium
```

**不包含过敏原、配料表、素食标识、清真认证字段。**

因此本项目结论的适用范围是「配料特调层面能否规避」，
不覆盖「是否含有某过敏原」。涉及花生、坚果、麸质、芝麻、贝类等，
以及交叉污染、素食认证、清真认证，接口均无数据，项目会明确标注需现场确认。

项目不做医疗建议。涉及医嘱的场景只提供菜单选项参考，并提示遵医嘱。
