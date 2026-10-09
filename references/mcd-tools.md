# MCP 工具速查与数据边界

实测环境：`https://mcp.mcd.cn`，测试门店 `1450713`（上海黄浦华旭国际大厦餐厅），2026-10-09。

## 工具清单与本 Skill 的用法

| 工具 | 用途 | 阶段 |
|---|---|---|
| `query-nearby-stores` | 定位门店拿 `storeCode` | 第 2 步 |
| `query-meals` | 拉该店真实在售菜单与价格 | 第 3 步 |
| `query-meal-detail` | 拉配料与特调能力（核心） | 第 4 步 |
| `list-nutrition-foods` | 能量/蛋白/脂肪/碳水/钠/钙 | 第 5 步 |
| `calculate-price` | 算改动后价格 | 第 7 步 |
| `create-order` | 下单（仅用户明确要求时） | 下单 |

## 字段陷阱

### 1. 价格单位不一致

`query-meals` 的 `currentPrice` 是**字符串形式的元**（`"25.5"`）。
`calculate-price` 返回的价格字段单位是**分**，展示要除以 100。
混用会差 100 倍。

### 2. `beType` / `orderType` / `beCode` 组合

| 场景 | beType | orderType | beCode |
|---|---|---|---|
| 到店取餐 | 1 | 1 | 不传（传了报错） |
| 得来速 | 5 | 1 | 必传 |
| 麦乐送 | 2 | 2 | 必传 |
| 团餐 | 6 | 2 | 必传 |

### 3. 单品与套餐的 `modification` 位置不同

- **单品**：`data.modification.items[]`，`data.rounds` 为空数组。
- **套餐**：`data.supportModify` 常为 `false`，但每个 `data.rounds[].choices[]` 各有自己的 `supportModify` 和 `modification`。

实测「巨无霸三件套」：`supportModify=false`，但轮次 1 的巨无霸 `supportModify=true`，
轮次 3 的饮料 `supportModify=true`。**只看顶层 `supportModify` 会漏掉套餐里真正可特调的部分。**

### 4. 必选项不能取消

`minValues=1` 且 `values` 只有 1 项 → 该配料必选，不能提供「不选」选项。
实测：可乐的冰量特调组 `minValues=1`，必须给出去冰/多冰/少冰/标准其一。

### 5. 特调组要全传

含 `unselectedKey` 的组，**所有**配料都要传进 `calculate-price` / `create-order`：
保留的传 `selectedKey`，去掉的传 `unselectedKey`。漏传按默认全配计算。

### 6. 空的 modification

实测「热浓浓黑巧中杯」的 `modification.items` 为
`[{maxValues:0, minValues:0, values:[]}]` — 这是空数组，不是错误。
遇到空 `values` 说明该选项无可调项。

### 7. `nutritionDescription` 基本为 null

160 项餐品中绝大多数为 `null`，少数是类比描述（如「能量约清水1杯」「能量约鲈鱼1条」）。
可以引用作为趣味化表达，但不要当成精确值。

### 8. 名称为空的营养项

实测返回中存在 `yeyeyeye奶冻款`、`yeyeyeye爆珠款` 这类内部测试名称，
以及重复项（`小杯玉米杯`、`纯牛奶（盒装）` 各出现两次）。
输出时按名称去重，异常名称跳过。

### 9. `calculate-price` 的 items 数组在部分客户端下解析失败

实测在 WorkBuddy 当前的工具调用层，`calculate-price` 的 `items` 数组参数会报
`/items: must be array`，无论数组内容是否合法（含最简 `[{productCode, quantity}]` 均失败）。

同一层环境下 `query-nearby-stores`、`query-meals`、`query-meal-detail`、`list-nutrition-foods`
四个工具调用正常。

判断为客户端数组参数序列化问题，非接口 schema 变化。

**影响**：第 7 步的算价在本环境下无法验证，其他步骤均已实测通过。
**处理**：用户在客户端内直接对话时，若算价调用失败，如实告知用户并给出
`query-meals` 返回的原价作为参考，同时说明特调本身通常不改变单价（实测特调项 `price` 多为 0，
部分咖啡加料为 500 分）。不要伪造算价结果。

`list-nutrition-foods` 返回的字段仅有：

```
productName, nutritionDescription, energyKj, energyKcal,
protein, fat, carbohydrate, sodium, calcium
```

**没有过敏原字段，没有配料表，没有素食标识，没有清真认证。**

因此本 Skill 的结论只覆盖「配料特调层面能否规避」，
无法覆盖「是否含有某过敏原」。这条边界必须在输出末尾体现。

## 实测参考值（2026-10-09，非承诺数据）

| 餐品 | energyKcal | protein | sodium |
|---|---|---|---|
| 大杯玉米杯 | 87 | 4 | 2 |
| 纯悦 | 0 | 0 | 0 |
| 浓缩咖啡 | 13 | 1 | 3 |
| 中薯条 | 289 | 4 | 165 |
| 麦香鸡 | 369 | 15 | 731 |
| 巨无霸 | 513 | 27 | 961 |
| 培根安格斯厚牛堡 | 707 | 34 | 1037 |
| 芝士双层安格斯厚牛堡 | 1003 | 58 | 1373 |

控钠场景推中杯玉米杯或苹果片；控热量推纯悦或玉米杯。
