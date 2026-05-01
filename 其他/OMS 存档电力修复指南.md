# OMS 存档电力修复指南

## 问题背景

TNO 模组中，"黑色联盟"（OMS/鄂木斯克）统一俄罗斯后，电力系统出现严重 bug：

- 生产单位（PU）全面崩盘，全部归零或负数
- 热电厂产能倍率锁死在 0，115 座热电厂产出为 0
- 电力面板显示 `总电力消耗：-15826`、`民用消耗：-15601` 等巨大负数
- 根本原因：统一过程中 TNO 经济系统变量损坏，`kd_consumption_multiplier` 被错误设为 3.69（应为 3）

## 最简单的修复方法（只改一处，100% 生效）

### 文件

```
C:\game\hoi4\hoi4\(h4)1.17.4.1\langou123\mod\2438003901\common\buildings\00_buildings.txt
```

### 第 421 行，改前：

```
local_resources_power = 5
```

### 改后：

```
local_resources_power = 500
```

### 说明

每座热电厂从产 5 电力变为产 500 电力，OMS 产出从 921 直接跳到 92100。重启游戏即生效。

---

## 关于"全球热电厂都翻倍"的问题

是的，这个文件是全局建筑定义，所有国家的热电厂都会翻 100 倍。**但这不重要，因为：**

**AI 国家完全不受电力盈亏影响。** TNO 的电力系统（生产单位分配、民用消耗等）仅通过 GUI 面板触发，AI 不会打开经济面板，`TNO_calculate_power_consumption` 对 AI 根本就不运行。HOI4 引擎层面，电力资源没有像钢铁、石油那样的硬性惩罚机制——缺电不会导致工厂停工、不会降低产出、不会有任何 debuff。电力在 TNO 中纯粹是一个"玩家资源"，多出来对 AI 零影响。

---

## 如果一定要只改 OMS（需要改模组文件）

### 文件

```
C:\game\hoi4\hoi4\(h4)1.17.4.1\langou123\mod\3449417512\common\dynamic_modifiers\2WRW_OMS_dynamic_modifiers.txt
```

### 第 5 行 `OMS_Reintegration_Efforts` 块内，加一行：

```
country_resource_power = 10000
```

改后效果：

```
OMS_Reintegration_Efforts = {
    enable = { always = yes }
    icon = "GFX_Omsk_Open_Chem_Weapons"
    compliance_growth = 0.1
    country_resource_power = 10000
}
```

`country_resource_power` 是 HOI4 原生国家电力加成，不被任何 TNO 脚本覆盖。值可任意调整。

---

## 存档内可改但不推荐的变量（会被游戏运行时覆盖）

以下变量在存档 `gogogo.hoi4` 中，但**每次打开经济面板都会被 TNO 经济系统重新计算覆盖**，改了也白改：

|行号|变量|说明|
|---|---|---|
|1828381|`buildings_power_demand`|总电力消耗|
|1828380|`buildings_power_consumption_display`|建筑电力消耗|
|1829561|`power_demand_gdp`|GDP 民用消耗（修正后）|
|1829562|`power_demand_gdp_display`|GDP 民用消耗（展示值）|
|1832728|`total_state_power_plant_effect`|全国电厂产能池（每开面板重算）|
|1832139|`tno_regulations_thermo_plant_power_factor`|电厂倍率（动态修饰符已注释，不生效）|
|1829350|`kd_consumption_multiplier`|消耗乘数（运行时重置为 3）|

---

## 技术细节：为什么存档改不动

1. **电力产出**由引擎计算：`每座热电厂等级 × local_resources_power(5)`，结果写入各州的 `resources={ power=X }`
2. **电力消耗**由 `TNO_calculate_power_consumption` 每次打开经济面板时重新计算：`GDP × KD_consumption_multiplier × 消费修正`
3. **`total_state_power_plant_effect`** 在 `calculate_power_plant_nominal_gdp_effect` 中被重置为 `(通电州数 ÷ 需电州数) × 0.025`，与存档值无关
4. **`tno_regulations_thermo_plant_power_factor`** 对应的动态修饰符在 `TNO_laws_dynamic_modifiers.txt` 第 78 行已被注释（`# thermo_plant_power_factor = ...`），该变量不产生任何效果