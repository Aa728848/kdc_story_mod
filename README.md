[README.md](https://github.com/user-attachments/files/34738775/README.md)
# KDC 纳米包 · 纳米机制扩展开发蓝图

- 本目录是 **KDC 纳米包的扩展蓝图**，作为舰R（Warship Girls R）的**纳米子层**。
- 所有代码以 `kdcne_` 前缀命名，只新增、不改舰R 体系；
- 已获得作者权限 直接在本体上改了 有点废话了
- 现在由DrxClef（github）、某个关于爱弥斯的逆天id的人在维护、开发此项目，其实是同一个神人

> 整理日期：2026-11-12。
---

## 一、整体思路

1. **侧重**

   KDC目前只会添加一个纳米侵蚀机制，侵蚀机制只会给debuff
   着重故事线与沉浸感，战斗爽可以放一放

2. **标记驱动（tag-driven）**

   有新buff新东西打了 `kdcne_` 不波及原版舰船或其他 mod

3. **参考**

| 来源 | 参考内容 | 代码方向 |
|---|---|---|
| 舰R | 领袖特质、舰船船体、组件 tag、舰娘先驱者/Vector 事件链 | fleet_modifier + ship_size+component_tags + 考古链 |

---

## 二、当前保留的核心

当前开发与验证优先级是**纳米云投射巨构**和**纳米云侵蚀减益**。普通命中云和远程全星系覆盖共用暴露、减益与恢复规则。恒星改造、星笼和扩展新增船坞已停用；隐藏的舰船同化开关仍默认关闭。

### 2.1 纳米云来源

| 来源 | 当前行为 | 主要入口 |
|---|---|---|
| 纳米鱼雷命中 | 在目标命中位置生成半径30、持续30天的小云；每支攻击舰队每3天最多生成一次，每来源国每星系最多24朵 | `common/component_templates/kdcne_components.txt`、`common/scripted_effects/kdcne_cloud_effects.txt` |
| 发生器 / 纳米舰载机 | 装有 A 槽云发生器或 H 槽纳米舰载机的母舰造成伤害时生成同样的小云，共用每舰队 3 天间隔；目前无法区分母舰的哪种武器造成了伤害 | `events/kdcne_events.txt` 中的 `kdcne.9` |
| 远程投射巨构 | 玩家选中可定位的恒星系后覆盖整个星系360天；不改变天体 | `common/megastructures/kdcne_megastructures.txt`、`events/kdcne_cloud_control_events.txt` |

远程投射普通模式消耗10000能量与2000纳米，共用30天帝国冷却。操作面板可以打开、锁定目标、返回或关闭；只有确认发射时才扣费并启动冷却。发射前重新检查投射台、目标、资源和冷却。普通模式不要求宣战、接触或调查，目标只需能定位到星系。当前代码没有反制方案中的“60天后AI回应/自动宣战”事件。

阵列造价50000合金＋10000纳米，基础工期5400天，维护费每月100能量。通过巨构升级入口使用，操作基础工期 1 天、完成后再等 1 天弹出面板，需要解除暂停。`control` 与 `control_repeat` 两阶段自然交替，保留同一巨构实例；旧阵列型号异常才使用 `event kdcne_cloud.25` 的维护入口。懒得删了 嘻嘻

远程云的视觉使用原版太空云模型，缩放 2 倍，偏离恒星 70 距离、旋转 45 度；视觉位置不代表完整判定边界。覆盖期限到达后清除覆盖状态，已经产生的侵蚀按恢复规则逐日消退。

### 2.2 暴露与减益

来源国拥有或控制的舰船免疫此云；其他来源的舰船均可受影响，包括和平或友好 AI、未接触国家、野怪、民用舰与空间站。不同来源的云重叠不会让同一舰船在一天内重复增加暴露。

| 规则 | 当前数值 |
|---|---:|
| 每日暴露增加 | 无抗性时 +1；抗性按比例降低 |
| 暴露上限 | 30 |
| 减益强度 | 每点有效暴露一层，最多 20 层；允许小数层 |
| 离开云区后的宽限 | 最后接触后约 2 天开始恢复 |
| 恢复速度 | 每天减少 1 点暴露，直至清零 |

| 每层舰船修正 | 数值 |
|---|---:|
| 武器伤害 | -5% |
| 航速 | -4% |
| 装甲上限 | -5% |
| 护盾上限 | -5% |
| 闪避 | -3 个百分点 |
| 装甲穿透乘数 | -5% |
| 护盾穿透乘数 | -5% |

20 层合计为武器伤害、装甲和护盾上限各-100%，航速-80%，闪避 -60 个百分点，双穿乘数各 -100%。这些修正与原版及其他 Mod 同类修正合算，最终战斗属性仍受其他加成影响。

减益挂在敌方舰船上，不由固定时长 modifier 自行清除；恢复事件依据暴露值更新强度。完全免疫不会新增暴露，并会在恢复结算时清除减益。具体事件调度、旧存档兼容与脚本回归记录见 `tools/test-kdcne-cloud.mjs`、`tools/kdcne-cloud-all-targets-evidence.json`。

### 2.3 吸血核对

纳米云侵蚀本身**没有吸血，也不定时扣除船体再回给发射方**。当前纳米鱼雷通过 `triggered_ship_modifier` 给载舰增加 `component_lifesteal_add = 0.1`，需要所属国家拥有 `tech_kdcne_lifesteal`；配置位于 `common/component_templates/kdcne_components.txt`。这是武器命中吸血的修正来源，不能据此认定云减益会回血。它挂在载舰修正上，不能保证只作用于这一枚鱼雷；多枚鱼雷是否叠加、回复船体/护甲的实际表现还需单独实测。本轮保留该旧配置。

另外，进化核心的低血修复与科技刷新给予的临时装甲/护盾恢复也可能造成回血观感；它们同样不来自侵蚀减益。

### 2.4 隐藏内容

纳米同化/夺船由控制台事件 `event kdcne_cloud.20` 控制，默认关闭；原法令入口已隐藏。事件界面明确提示兼容性 BUG 和不可预测的事件链中断风险，适合实在难以继续游戏时自行开启。它不是本轮核心平衡目标。

开启还需要 `tech_kdcne_assimilation`。目标累积 30 个有效暴露日且仍在对应来源云区时，延迟 1 天执行设计复制；成功生成并核对我方归属后才移除原舰，失败保留原对象。旧船身份、领袖关系、经验、事件标记和临时修正无法靠复制完整继承。关闭开关不撤销已有云区或减益，也不把已经生成的舰船退回。

### 2.5 保留的旧功能（本轮不新增或调平衡）

| 功能 | 当前代码行为 |
|---|---|
| 击杀进化 | 装备进化核心、拥有进化科技的纳米舰每 5 次击杀升 1 级，最高 20 级；每级伤害 +5%、船体 +3% |
| 低血修复 | 受击后仍存活、船体低于 20% 且有击杀计数时，消耗 1 点计数调用修复；不保证抵挡致命一击 |
| 科技刷新 | 扩展科技完成时，对带纳米标记的舰船添加 30 天临时修正：伤害 +20%、装甲/护盾回复；不属于吸血 |
| 击杀收益 | 带纳米标记舰船击杀后，国家获得 1 纳米 |
| 星系纳米化巨构 | `kdcne_mega_nanofication` 转化符合条件的未殖民自然岩质行星，保留恒星、气态巨行星、人工世界与剧情地点；与已废弃的恒星转化不同 |
| 循环科技 | 六项循环仍走纳米开发局；`weight = 0`，当前不进入普通研究池 |
| 行星建筑 | 保留进化实验室，以及降低 25% 轨道轰炸伤害的纳米防护设施；后者不产生舰船云区 |

已确认：本轮保留上述旧功能；后续若精简，单独决定并更新本表，不能把“本轮只做云减益”误写成这些功能已经被禁用。

---

## 三、功能状态与停用范围

| 功能 | 状态 | 说明 |
|---|---|---|
| 远程投射普通模式 | 保留、可用 | 消耗 10000 能量和 2000 纳米，30 天冷却，覆盖 360 天 |
| 命中云与侵蚀减益 | 保留、可用 | 共享每日暴露、20 层减益上限及离云恢复 |
| 纳米反制 | 已废弃 | 当前活动脚本目录未找到反制实现；只在历史设计/压缩资料中出现，不参与运行 |
| 远程恒星转化模式 | 已停用 | 控制台入口已移除，旧确认事件只允许安全关闭；保留普通投射模式 |
| 巨像 W 槽恒星转化器 | 已停用 | 不再允许瞄准恒星或装备新组件；对应命中回调已解除 |
| 纳米恒星与纳米星笼 | 已停用新建 | 科技和巨构入口关闭；保留旧存档类定义、模型刷新与保守迁移代码 |
| 扩展新增的 `kdcne_mega_shipyard` | 已停用新建 | 巨构入口关闭。舰R/KDC 原有船坞链是独立内容，不在本轮停用范围内 |
| 星系纳米化 `kdcne_mega_nanofication` | 仍保留 | 与纳米恒星转化、星笼是不同功能；本轮按截图范围未停用 |
| H/X/L/W/T 槽武器压制构想 | 待评估、未实现 | 目前减益不逐槽阻止武器开火；先评估引擎支持与兼容性，再决定是否纳入 |
| AIII 领袖事件链 | 设计稿 | 尚无活动事件、遗迹或领袖定义 |

停用采用入口/定义门控、无操作兼容回调与旧实现注释：星笼/船坞的 `potential` 和 `possible` 均拒绝新建，星笼科技从开发局按钮与完成钩子撤下，恒星转换效果和星笼完工效果保留为空入口。旧恒星类、星笼类、模型和巨构键值继续加载，避免存档对象出现缺失引用。原法令隐藏后，同化仍可通过独立事件手动控制。

已有纳米恒星和扩展船坞保留原对象。已有星笼继续沿用旧兼容修复：已殖民、正在殖民或附带轨道设施的世界保留身份与内容；未使用的旧偏移星笼可能由迁移逻辑在恒星处重建，并清理其施工基址。该迁移不属于新建玩法，仍应另存后验证。

---

## 四、代码组织

所有新增代码使用 `kdcne_` 前缀。舰船、国家、星系和行星标记分别记录在各自作用域，避免用全局标记误伤其他 Mod。

| 责任 | 文件入口 |
|---|---|
| 纳米云鱼雷与科技门控吸血 | `common/component_templates/kdcne_components.txt` |
| 远程投射台与控制阶段 | `common/megastructures/kdcne_megastructures.txt` |
| 投射控制、确认发射与冷却 | `events/kdcne_cloud_control_events.txt`、`common/button_effects/kdcne_cloud_buttons.txt`、`interface/kdcne_cloud_control.gui` |
| 云的生成、暴露、恢复与覆盖效果 | `common/scripted_effects/kdcne_cloud_effects.txt`、`events/kdcne_cloud_events.txt` |
| 每层侵蚀修正 | `common/static_modifiers/zz_kdcne_static_modifiers.txt` |
| 天体旧存档兼容 | `events/kdcne_stellar_events.txt`、`common/scripted_effects/kdcne_stellar_effects.txt` |
| 科技与纳米开发局接入 | `common/technology/kdcne_tech.txt`、`common/button_effects/`、`events/kdcne_tech_events.txt` |
| 简体中文本地化 | `localisation/simp_chinese/kdcne_*.yml` |

本地化说明、AIII 设计和美术替换信息以本 README 为总入口。脚本测试和存档证据只用于静态回归/实现记录，不替代游戏内验证。

### 4.1 命名与作用域

- 新增键统一 `kdcne_`，科技为 `tech_kdcne_`，美术为 `GFX_kdcne_`；保留已存在的旧键，不为统一命名而改存档键。
- `kdcne_is_nano_ship` 是进化/收益体系的纳米舰标记；侵蚀减益针对外来舰船，不要求受影响舰船带该标记。
- 舰船：`kdcne_cloud_exposure`、`kdcne_cloud_severity`、`kdcne_cloud_recent`、`kdcne_cloud_recovery_running`。
- 国家：`kdcne_assimilation_enabled`、`kdcne_projection_cooldown`；星系：`kdcne_projection_active`、`kdcne_projection_loop_running`。
- 舰队：`kdcne_cloud_spawn_cooldown`；进化变量实际使用 `kdcne_kill_count` / `kdcne_evo_lv`。
- `weapon_type_nano` 是现有纳米武器类别；组件自己的 `kdcne_nano_torpedo` / `kdcne_nano_aircraft` 等用于精确识别。继承舰R武器加成仍须核对类别/组件限制，前缀本身不会自动继承效果。

### 4.2 科技接入

| 开发局 ID / 科技 | 当前用途与状态 |
|---|---|
| 4000 / `tech_kdcne_evolution_core` | 进化核心，保留 |
| 4001 / `tech_kdcne_nano_corrosion` | 纳米侵蚀鱼雷，保留 |
| 4002 / `tech_kdcne_nano_cloud` | 云发生器、纳米舰载机、防护建筑，保留 |
| 4003 / `tech_kdcne_assimilation` | 隐藏事件开启夺船的科技条件，保留；不会自动打开开关 |
| 4004 / `tech_kdcne_lifesteal` | 纳米鱼雷载舰吸血修正，保留 |
| 4005 / `tech_kdcne_system_nanofication` | 远程投射阵列与旧星系纳米化巨构，保留；不能随恒星转化一起删除 |
| 4006 / `tech_kdcne_nano_starcage` | 已停用；旧科技键值和空回调保留 |
| 六项 `tech_repeat_*` | 纳米、合金、消费品、科研、纳米武器、凝聚力循环，仍在开发局；定义见 `common/technology/zz_kdcne_technology.txt` |
| `tech_kdcne_a3_judgment_carrier` | AIII 设计占位，尚无代码；计划考古奖励科技卡并推进 10% |

---

## 五、剧情
### 5.1.1 远星之歌原版剧情结构

   第一幕 怪谈      1.1.x 母星异象（Omega物质→核弹→世界吞噬者→飓风）
                    1.2.x 星团探索（纳米战舰→纳米星球→孤立星门）
                           ↓ 故事链【贯穿古今的神秘现象】集满线索
   第二幕 接触      2.1.1 二选一：找操控者 / 找控制方法
                    ├─ 2.1.2 守护者线：荒野初遇→勋章→两万年合约
                    └─ 2.1.3 控制权线：指挥室对峙
                           ├─ 同意 → 并入守护者线
                           └─ 拒绝 → 2.3.1-2 覆盖→失控
                                     ├─ 坚持 → 末日之后（自主线，灰风永不出现）
                                     └─ 停止 → 亡羊补牢→海边垂钓→试用契约（支配者线，好感-2）
   第三幕 拓荒      2.3.1 全球生态纳米框架（守护者/支配者）
                 2.3.2 纳米星球改造：灰风线 / 自主线 / 覆水难收（灰风跑路→转自主线）
   第四幕 寻钥      2.4.1 灰风自白 → 故事链【L星门的线索】
                 五处遗迹 × 五段 gal：工厂／建筑群／卫星／研究设施／**坠毁星舰→资料库**
   第五幕 开门      2.5 德萨努的最后留言："你自由了"
   后日谈           3.1 称后   3.2 政治线（帝制／亲外／排外→安全宣言→鸿门宴→HE/BE/黑化）
                    纳米飞升（机制清单）   深海线 gal（独立短线，结局互相求婚）

### 5.1.2 后续修改方向

   于**坠毁星舰→资料库**处替换为前文明德萨努的审判者AIII，替换最后的留言，运用灰风在主线中“赎罪、忏悔”的任务弧光  processing
   后日谈移除，替换为记录玩家出门之后都干啥了，有没有四处图图外星人，本国殖民地运营情况等来进行事件分支  not sure
   新增AIII与灰风的联动对话   not sure


### 5.2.1 AIII 领袖设计（未开发）
  AIII 是古代舰娘先驱者文明制造的“审判代行者”，定位为舰队司令。审判之眼对应索敌、反隐和破隐；“遮蔽双眸＝审判的开端”为人物意象。战斗方向为炮击与舰载机并用。领袖特质可承载帝国级纳米未  果，但数值尚未定稿。

### 5.2.2 背景与发现

   古代先驱者文明（德萨努）制造纳米机器人，纳米机器人叛乱，在与纳米舰队作战时逆向其技术，制造AIII作为决战兵器。AIII曾深入L星团灰蛊风暴控制中心的三星系统，最终坠毁于该星系的一颗纳米星球上并沉睡。该背景借用原版L星团/灰蛊叙事，不改变原版灰风事件链。

计划在纳米星球上生成三级异常“地表巨舰”；调查后开启遗迹 `site_kdcne_a3`，遗迹揭示背景并触发觉醒。异常和遗迹文本、阶段数量尚未完成。

### 5.3 事件链与奖励草案

1. 探索发现纳米索敌脉冲异常。
2. 定位坠毁的超级航空战舰并生成考古遗迹。
3. 通过多阶段考古唤醒 AIII。
4. 发放审判母舰科技卡 `tech_kdcne_a3_judgment_carrier`，并给予 10% 科技进度。
5. 玩家选择接纳 AIII 为领袖，或让她继续沉睡并领取纳米索敌科技/资源保底。
6. AIII 服役后提供舰队技能与帝国级内阁词条；审判母舰作为独立舰体奖励。

### 5.4 待定项
  
- 领袖的四项航战技能、数值和技能触发方式未定。
- “必定暴击”建议先按必定命中与额外伤害模拟，尚未确认。
- 瘫痪累积速度、阈值和持续时间未定；不应在设计阶段假定可直接移除舰船 AI 行为。
- AIII 立绘、头像及审判母舰舰体美术尚未提供。
- 原草案的审判母舰参数为船体 400000、`size_multiplier = 200`、X×2 / T×2 / H×16，仅作未实现的历史数值；故事与技能确认后再平衡。
- 异常阶段数、出现保底与最终刷新位置需要统一，不能一边指定控制中心、一边宣称任意纳米球必刷；当前只保留候选方案。
- 参考舰R 列克星顿的遗迹奖励链、星极的考古/事件链。索敌/反隐优先接入舰R现有能力；纳米云可复用本项目效果，母舰沿用舰R所需的反应堆、FTL、推进器、传感器、火控和光环组件集。
- 旧反制方案中的定向瞬移巨构及 AIII 独立奖励/测试入口随反制方案搁置；当前活动目录没有该奖励实现，不视为 AIII 既有奖励。

### 5.5 非远星起源的玩家

- 生成异常时避开已带原版灰风异常/遗迹的纳米星球。
- 保底草案包括“未调查纳米星球剩余 ≤2 时确保出现”，或异常长期未被调查后给追踪线索；还未选择最终方式，避免玩家无法找到事件。

---

## 六、美术素材现状与替换清单

只列当前保留功能的素材和已停用内容的旧存档素材。引用原版资源不代表复制了原版文件；替换模型时要同时检查实体注册、网格、比例、枢轴、点击区域和旧存档刷新。

| 对象 | 当前素材/状态 | 代码入口 | 替换方向 |
|---|---|---|---|
| 远程投射阵列及操作阶段 | 原版戴森球第一阶段实体；施工阶段用原版施工平台。操作阶段共用同一实体 | `common/megastructures/kdcne_megastructures.txt` | 制作专属阵列、施工阶段和窗口背景；三个阶段保持一致 |
| 远程投射云 | 原版 `space_cloud_entity`，偏离恒星显示 | `common/ambient_objects/kdcne_cloud.txt`、`common/scripted_effects/kdcne_cloud_effects.txt` | 可换专属云模型/粒子；保持不可点选，不挡住中央天体 |
| 旧星系纳米化巨构 | 同样借用原版戴森球第一阶段框架 | `common/megastructures/kdcne_megastructures.txt` 中 `kdcne_mega_nanofication` | 若后续继续美术开发，与投射阵列分别设计 |
| 纳米恒星（旧存档） | 旧定义刷新到原版 `black_hole_entity`；不再生成新恒星 | `common/planet_classes/kdcne_nano_star.txt`、`common/scripted_effects/kdcne_stellar_effects.txt` | 暂不替换；若恢复该设计，再复用纳米星球表面并验证引擎渲染 |
| 纳米星笼（旧存档） | 旧星球包装戴森球第五阶段 mammalian 模型；新建入口关闭 | `common/planet_classes/kdcne_planet_classes.txt`、`gfx/models/planets/kdcne_nano_starcage.asset` | 暂不替换；恢复设计时再制作专属可殖民外壳 |
| 扩展船坞（旧存档） | 原版 `mega_shipyard_01_stage_3_entity`；新建入口关闭 | `common/megastructures/kdcne_megastructures.txt` 中 `kdcne_mega_shipyard` | 暂不替换，保留旧对象显示 |
| AIII 立绘/异常/母舰图标 | 尚无实现或专属素材 | 未来 `GFX_kdcne_a3_portrait`、`GFX_kdcne_a3_anomaly`、`GFX_kdcne_judgment_carrier_icon` | 先完成事件和舰体设计，再按界面尺寸制作 |
| 远星之歌灰风新立绘 | 已经约稿 | 棍木 | 减少差分数量，精简一下 |

### 6.1 可复用素材位置

- 远程阵列实体注册：`D:\SteamLibrary\steamapps\common\Stellaris\gfx\models\ships\megastructures\dyson_sphere\_dyson_sphere_entities.asset`。
- 戴森球网格注册：同目录 `_dyson_sphere_meshes.gfx`；第一阶段框架为 `dyson_sphere_frame.mesh`。
- mammalian 戴森球贴图：同目录 `mammalian_dyson_sphere_01_diffuse.dds`、`mammalian_dyson_sphere_01_normal.dds`、`mammalian_dyson_sphere_01_specular.dds`。
- 施工平台实体：`D:\SteamLibrary\steamapps\common\Stellaris\gfx\models\ships\megastructures\construction_platform\construction_platform_01.asset`，网格 `construction_platform_01.mesh`。
- 远程云实体注册和网格：`D:\SteamLibrary\steamapps\common\Stellaris\gfx\models\ships\other\_other_entities.asset`、`_other_meshes.gfx`、`space_cloud.mesh`。
- 旧纳米星球表面：`2737603165\gfx\models\planets\wg_gray_goo_planet_01_diffuse.dds` 和 `wg_gray_goo_planet_01_normal.dds`；未启用候选实体 `2737603165\gfx\models\planets\kdcne_nano_star.asset` 上次实测不可见。重新设计?
- 旧纳米恒星当前黑洞素材：`D:\SteamLibrary\steamapps\common\Stellaris\gfx\models\planets\black_hole_new.asset`、`black_hole_new.mesh`；附属圆环/引力透镜资源也在同目录。
- 旧星笼外壳：原版戴森球网格 `dyson_sphere_frame.mesh` 与 `dyson_sphere_part_1.mesh` 至 `dyson_sphere_part_4.mesh`，当前包装实体为 `2737603165\gfx\models\planets\kdcne_nano_starcage.asset`。
- 旧扩展船坞：`D:\SteamLibrary\steamapps\common\Stellaris\gfx\models\ships\megastructures\mega_shipyard\mega_shipyard_01.asset`；网格注册 `mega_shipyard_01.gfx`，主体网格 `mega_shipyard_01_stage_3.mesh`，外观对应贴图在同目录。
- 通用巨构窗口背景继续使用原版 `GFX_megastructure_construction_background`，注册于 `D:\SteamLibrary\steamapps\common\Stellaris\interface\megastructure_view.gfx`。
- 背景贴图：`D:\SteamLibrary\steamapps\common\Stellaris\gfx\interface\planetview\megastructure_background\megastructure.dds`。
- 旧纳米天体界面背景/天空：`2737603165\gfx\portraits\environments\pc_wg_graygoo.dds`、`pc_wg_graygoo_sky.dds`；类型图标为 `interface/WG_planet.gfx` 注册的 `GFX_wg_planet_type_graygoo`，引用 `gfx/interface/icons/wg_planet_type_icons.dds` 第 10 帧。
- 当前投射台三个阶段均使用 `dyson_sphere_phase_01_entity` 后缀，由国家舰船外观选择实际实体（如 `mammalian_01_dyson_sphere_phase_01_entity`），XY 偏移为 -27/-27。替换时三个阶段一起处理。
- 星笼包装实体附着原版 `mammalian_01_dyson_sphere_phase_05_entity`，`entity_scale = 1.0`、固定比例且无轨道线；旧黑洞显示使用 `entity_scale = 20.0`。二者重叠遮挡问题在该设计恢复前不继续开发。
- 旧星笼定位参考：`reference_mods/2660548454/common/megastructures/AOW_4_starcage_world.txt`、`reference_mods/2660548454/common/planet_classes/AOW_planet_classes.txt`；当前未从该参考 Mod 复制模型或贴图。

### 6.2 图标与待写文本

| 对象 | 当前图标/文件 | 后续替换 |
|---|---|---|
| 纳米鱼雷 | `GFX_wg_gray_g`；`gfx/interface/icons/ship_parts/wg_gray_g.dds` | 专属鱼雷图标 |
| A 槽发生器/进化核心 | `GFX_wg_gray_ia`；`gfx/interface/icons/ship_parts/wg_gray_ia.dds` | 两种组件分别制作图标 |
| 纳米舰载机 | `GFX_wg_gray_h`；`gfx/interface/icons/ship_parts/wg_gray_h.dds`；实体 `gatebuilder_wsg_fighters_strikecraft_entity` 注册于 `gfx/models/ships/gatebuilder_01/wsg_gatebuilder_entities.asset` | 专属载机图标/模型 |
| 投射台建造/大纲图标 | `interface/kdcne_gfx.gfx` 中三阶段 GFX，复用 `gfx/interface/icons/ascension_perks/ap_become_the_gray.dds` | 三阶段对应图标 |
| 侵蚀实际减益图标 | 原版 `gfx/interface/icons/modifiers/mod_ship_evasion_add.dds` | 专属侵蚀图标；同时检查静态修正中的真实引用 |
| 其他纳米修正占位 | `interface/kdcne_gfx.gfx` 多数复用 `gfx/interface/icons/resources/nanites_large.dds` | 进化、免疫、抗性等专属图标；GFX 注册存在不等于已被界面使用 |
| AIII 文本 | `kdcne_a3_anomaly_name`、`kdcne_a3_anomaly_desc`、`site_kdcne_a3_intro`、`kdcne_a3.*` | 异常、考古阶段、接纳/沉睡选项、领袖与舰体描述，均待写 |

本地化正式文件使用 UTF-8 BOM，并放在 `localisation/simp_chinese/`。GFX 注册和组件实际引用都要改，不能只增加一份同名素材表。

上述本机原版素材路径用于开发核对；不同安装位置需要按本机 Stellaris 安装目录调整。所有正式模型替换都要完全重启游戏，并验证新建对象和已有存档对象。

---

## 七、后续工作顺序

1. 游戏内回归普通远程投射：打开/关闭面板不扣费，只有确认发射扣费；30 天内无法重复发射，第 30 天恢复；覆盖持续 360 天。 check
2. 回归命中小云和远程覆盖的减益、重叠云每日去重、离云恢复、免疫与抗性。 check
3. 单独测试纳米鱼雷 10% 吸血科技的船体/护甲回血，并确认没有把回血误归因给云减益。 check
4. 检查旧存档中已存在的纳米恒星、星笼与扩展船坞仍能读取；确认新建入口和旧恒星转换入口已关闭。 not check
5. 评估槽位武器压制是否能稳定区分 H/X/L/W/T，并估计兼容成本；确认前只保留当前通用减益。 not check
6. 完成 AIII 机制与剧情取舍后，再进入事件实现和美术制作。 processing
7. kdc远星之歌剧情扩展。 processing
8. 借鉴隔壁的轨道纳米云巨构，本地化可以写为类似舰rmod的巨型轨道站pro，加速舰船制造速度、给星球buff

### 7.1 方便复测

- 本机静态检查：在工作区根目录执行 `node tools/check-kdcne.mjs`；云机制回归：`node tools/test-kdcne-cloud.mjs`。它们不启动群星，也不模拟引擎绘制、真实武器回复或完整 AI 行为。
- 测试存档可用 `event kdc_test.2` 获取现有纳米测试科技（会发放多项科技，仅用于测试存档）；同化开关单独执行 `event kdcne_cloud.20`，减益测试保持关闭。
- 通过投射台正常升级入口打开面板；不要用事件直接开窗代替阵列绑定。确认只有“锁定星系/纳米云覆盖”，没有恒星转化按钮。
- 第 0～29 天尝试再次启动操作/确认发射，第 30 天复测；观察目标舰船每日侵蚀，20 层封顶。离云后约第 2 天开始减暴露；满 30 暴露时约第 31 天完全清除。
- 旧存档可用 `event kdcne_stellar.2` 调用现有外观兼容修复；新建星笼与恒星转化已关闭，不再作为普通测试项目。
- 生成靶舰：先选中一支己方舰队，再运行 `event kdcne_test.6`，在附近生成高耐久测试靶舰；测试结束在该星系选中己方舰队执行 `event kdcne_test.3`，清理尚属测试国的靶舰。只在测试存档执行。
- 美术或脚本同步到实际加载目录后完全重启游戏。当前修改先在工作区验收，未自动改写 Steam 工坊副本。

### 7.2 本轮检查记录

- 2026-11-11：78 项脚本子集回归通过，覆盖费用、30 天冷却、360 天覆盖、暴露与恢复、隐藏同化、废弃入口以及旧对象兼容；详细结果为 `tools/kdcne-cloud-tests.json`。
- 结构/引用/本地化/贴图检查结果为 `tools/kdcne-static-check.json`；不把静态检查等同于群星引擎实测。
- CWTools 的本地桥接配置读取被权限阻止，未取得本轮有效语义诊断。未运行游戏或声称模型/吸血效果已通过实机验证。
- 工作区目前的 Mod、idea 和 tools 均处于 Git 未跟踪状态；文件落盘不等于已纳入版本记录。本轮未自动暂存或提交。

## 八、已隐藏

- 纳米同化/夺船：事件 `kdcne_cloud.20`，默认关闭；法令入口已隐藏，事件有明确风险提示。原存档里曾手动启用的开关仍按原状态保留，可通过事件手动关闭。

## 九、已废弃或暂停

- 纳米反制：当前活动目录无实现；历史压缩文件/设计稿不属于活动代码。
- 纳米恒星远程转化、巨像恒星转化器、纳米星笼：新入口已关闭；保留旧存档识别和模型刷新定义。
- 扩展新增的 `kdcne_mega_shipyard`：新建入口已关闭；不影响舰R/KDC 原有船坞。
- H/X/L/W/T 槽武器禁用效果：仍是待评估构想，没有脚本实现。

