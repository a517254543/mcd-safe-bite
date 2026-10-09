# 工作上下文记录

本文件用于核验项目是否真实使用 WorkBuddy 开发。

## 开发过程

### 阶段一：赛事与工具调研

使用 WorkBuddy 内置网页抓取能力读取：
- `M-China/mcd-developer-innovation-challenge`（README + activityGuidelines.md + RANKING.md）
- `M-China/mcd-mcp-server`（README 工具列表）

发现规则要点：
- 排名仅按 GitHub 公开 Star 数
- 榜单前 7 名集中在省钱 / 积分 / 热量决策赛道，同质化严重
- 榜首仅 19 星，窗口期充足

据此判断应避开省钱红海，转向未有人做的方向。

### 阶段二：真实接口探查

**这一步改变了项目设计。**

用麦当劳 MCP 逐个工具实测，发现两个关键事实：

1. `list-nutrition-foods` 返回 160 项餐品，字段仅有
   `energyKj / energyKcal / protein / fat / carbohydrate / sodium / calcium`，
   **没有过敏原字段**。

   → 原本设想的「过敏原检测」定位无法成立，
     因为接口根本不含这些数据。改为如实声明能力边界，
     把项目定位收窄到「配料特调可行性」，这是数据支撑得住的部分。

2. `query-meal-detail` 返回 `modification.items[].values[]`，
   内含配料名称与 `selectedKey` / `unselectedKey`。

   实测巨无霸可调配料：巨无霸酱、吉士、生菜、酸黄瓜、洋葱粒。
   实测麦香鸡可调配料：麦香鸡酱、生菜（与巨无霸不同，证明配料表逐品独立）。

   → **这是整个项目的地基。** 把配料拆到可增减粒度，
     正是「饮食限制翻译成可执行改动」所需的唯一数据源。

3. 套餐结构差异：`巨无霸三件套` 顶层 `supportModify=false`，
   但 `rounds[].choices[]` 各轮次分别有自己的 `supportModify` 与 `modification`。
   只看顶层会漏掉套餐内真正可调的部分（汉堡去吉士、饮料换无糖）。

4. `query-meal-detail` 返回的 `modification` 结构中，
   `values` 存在为空数组的情况（如「热浓浓黑巧中杯」`values:[]`），
   需判断为空而非报错。

### 阶段三：约束数据库编写

基于实测到的真实配料名编写 `references/constraint-db.md`：
- 已确认可特调的配料清单（吉士、巨无霸酱、酸黄瓜、洋葱粒、奶油球、白砂糖等）
- 6 类限制的判定规则（牛、猪、乳、蛋、素、控钠控热量）
- 8 项接口无法判定的项目（花生、坚果、麸质、芝麻、贝类、交叉污染、清真认证、素食标识）
- 判定优先级：主料命中直接 ⛔，不看特调

### 阶段四：算价环节排查

`calculate-price` 在当前工具调用层报 `/items: must be array`，
尝试最简合法入参 `[{productCode:"1100",quantity:1}]` 同样失败，
判断为客户端数组参数序列化问题，非接口 schema 变更。

处理方式：在 `references/mcd-tools.md` 第 9 条如实记录，
并要求 Skill 在算价失败时告知用户、给出 `query-meals` 原价作参考，
**不得伪造算价结果**。

## 真实门店数据

测试门店 `1450713`（上海黄浦华旭国际大厦餐厅），经 `query-nearby-stores`
以 `searchType=2, city=上海, keyword=人民广场` 获取。

实测在售菜单约 100+ 项，含「人气热卖」「精选单人餐」「巨无霸牛鱼肉堡」
「安格斯MAX厚牛堡」「小食甜品」「饮品」等分类。

营养数据引用值：
中薯条 289kcal/钠165mg、巨无霸 513kcal/钠961mg、
培根安格斯厚牛堡 707kcal/钠1037mg、芝士双层安格斯厚牛堡 1003kcal/钠1373mg、
大杯玉米杯 87kcal/钠2mg、纯悦 0kcal/钠0mg。

## 环境约束

- `calculate-price` / `create-order` 的数组参数在本环境不可用
- `list-nutrition-foods` 返回中存在内部测试名称（如 yeyeyeye 开头项）
  与重复项，需按名称去重
- `CONTEST_DECLARATION.md` 使用官方原文，内容与文件名均未修改
