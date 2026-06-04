# ChallengeModes 模块文档

概述
- 此模块为 AzerothCore 的一个扩展模块，实现了“挑战模式”（Challenge Modes）玩法（如硬核、传奇、工匠、铁人等）。
- 文件：`ChallengeModes.cpp` / `ChallengeModes.h`。
- 通过服务器配置读取一系列开关、奖励与倍增参数，玩家可在角色创建或与指定 gameobject 交互时开启挑战模式（可选客户端同步的临时选择）。

主要类型与变量
- 枚举 `ChallengeModeSettings`
  - `SETTING_HARDCORE` (0)、`SETTING_SEMI_HARDCORE` (1)、`SETTING_SELF_CRAFTED` (2)、`SETTING_ITEM_QUALITY_LEVEL` (3)、`SETTING_SLOW_XP_GAIN` (4)、`SETTING_VERY_SLOW_XP_GAIN` (5)、`SETTING_QUEST_XP_ONLY` (6)、`SETTING_IRON_MAN` (7)、`HARDCORE_DEAD` (8)
- 单例 `ChallengeModes`（访问宏：`sChallengeModes`）
  - 控制全局开关：`challengesEnabled`、各模式 `...Enable`
  - 各模式禁用等级：`...DisableLevel`
  - 经验倍率：`...XpBonus`
  - 奖励映射：`...TitleRewards`、`...TalentRewards`、`...ItemRewards`、`...AchievementReward`
  - `rewardConfigMap`：配置项名到对应奖励 map 的映射（用于读取配置）

配置与加载
- 在 `ChallengeModes_WorldScript::LoadConfig()` 中读取配置，典型的配置项示例（见 `rewardConfigMap` 中的键）：
  - `"Hardcore.TitleRewards"`、`"SemiHardcore.ItemRewards"`、`"IronMan.AchievementReward"` 等。
- 奖励字符串解析函数：`LoadStringToMap(std::unordered_map<uint8,uint32>&, const std::string&)`
  - 配置的字符串格式为逗号分隔的若干对：每对由两个以空格分隔的数字组成（例如 `"10 1234,20 2345"` 表示等级 10 给奖励 1234，等级 20 给奖励 2345）。

核心接口（重要函数）
- `ChallengeModes::instance()`：获取单例。
- `ChallengeModes::enabled()`：模块是否启用。
- `ChallengeModes::challengeEnabled(ChallengeModeSettings)`：某类挑战是否在配置中启用。
- `ChallengeModes::challengeEnabledForPlayer(ChallengeModeSettings, Player*)`：检查玩家是否开启某挑战（且模块与该挑战已整体启用）。
- `getTitleMapForChallenge/getTalentMapForChallenge/getItemMapForChallenge/getAchievementMapForChallenge`：返回对应模式下的奖励映射（等级 -> 奖励 id）。
- `getXpBonusForChallenge` / `getItemRewardAmount` / `getDisableLevel`：分别用于获取经验倍率、发奖数量、自动取消模式的等级阈值。

玩家相关脚本与行为
- `ChallengeMode_PlayerCreateScript::OnPlayerLogin(Player*)`
  - 处理客户端创建角色时的临时选择（`temp_challenge_flag`）。
  - 使用一个 8 位的掩码数组 `bitMask[8] = {1,2,8,16,32,64,128,256}` 对应客户端选择位，开启对应模式并给玩家记录被动技能（`spellList`）。
  - 开启前会基于 `CanOfferChallenge` 的互斥逻辑检测冲突（例如硬核与传奇互斥，工匠和铁人互斥，缓速与乌龟互斥等）。
- `ChallengeMode`（基类，继承自 `PlayerScript`）
  - 提供通用钩子：
    - `OnPlayerGiveXP`：按模式调整经验（乘以 `getXpBonusForChallenge`）。
    - `OnPlayerLevelChanged`：在升一级时发放奖励（称号 / 天赋点 / 物品通过邮件 / 成就）并检查是否达到自动禁用等级，达到则清除该模式。
  - 辅助：`mapContainsKey` 检查奖励 map 是否包含某等级的奖励。
- 每个模式有一个派生类覆盖相关钩子来实现具体规则：
  - `ChallengeMode_Hardcore`
    - 玩家死亡后标记 `HARDCORE_DEAD`，并通过踢出会话等方式阻止继续游戏（示例：`KillPlayer()` + `KickPlayer()`）。
    - 在 `OnPlayerLogin` 如果已处于 `HARDCORE_DEAD` 状态则强行杀死并踢下线。
  - `ChallengeMode_SemiHardcore`
    - 死亡时摧毁玩家装备并清零金币（循环 `EQUIPMENT_SLOT_END`）。
  - `ChallengeMode_SelfCrafted`
    - 只允许装备带有签名且创建者为玩家自身的物品（检查 `ITEM_FIELD_CREATOR`）。
  - `ChallengeMode_ItemQualityLevel`
    - 只允许装备品质 <= `ITEM_QUALITY_NORMAL` 的装备。
  - `ChallengeMode_SlowXpGain` / `ChallengeMode_VerySlowXpGain`
    - 通过基类的经验倍率实现（默认配置分别 0.5 和 0.25）。
  - `ChallengeMode_QuestXpOnly`
    - 击杀怪物不再给玩家经验（但给宠物经验），仅保留非击杀来源的经验。
  - `ChallengeMode_IronMan`
    - 禁止复活（复活立即死亡）、移除自由天赋点、禁止学习多数贸易技能、禁止使用药剂/食物增益（通过检查 `ItemTemplate::Spells`）以及禁止组队邀请/接受等。

GameObject（神像）交互
- 脚本类 `gobject_challenge_modes`（继承 `GameObjectScript`）
  - 常量：
    - `GOSSIP_SENDER_CHALLENGE_CONFIRM = 1`
    - `GOSSIP_ACTION_CHALLENGE_CANCEL = 1000`
  - `GetPassiveSkillForChallenge` / `GiveChallengeRecordSkill`：按 `ChallengeModeSettings` 返回并授予对应的被动记录 spell id（数组 `PassiveSkillIds`）。
  - `GetChallengeDescription`：返回每个模式的中文说明（用于 gossip 界面）。
  - `CanOfferChallenge`：判断是否允许为玩家在该位置提供开启该模式（使用玩家当前设置来避免互斥冲突）。
  - `OnGossipHello`：展示可供选择的挑战列表（动态基于 `challengeEnabled` 与 `CanOfferChallenge`）。
  - `SendChallengeConfirm`：显示确认对话（包含说明与确认/取消选项）。
  - `OnGossipSelect`：处理确认后的动作，最终调用 `player->UpdatePlayerSetting("mod-challenge-modes", action, 1)` 并授予记录技能。

注意事项与扩展点
- 奖励配置解析较为简单（以空格分隔等级与奖励 id，每组以逗号分隔），修改配置格式需同步更新 `LoadStringToMap`。
- `HARDCORE_DEAD` 是一个特殊的设置索引（用于记录硬核已死亡状态），在 `ChallengeModes::challengeEnabled` 等函数中不作为普通开关处理。
- 若想添加新模式：
  - 在 `ChallengeModeSettings` 枚举中注册新索引，添加对应的配置变量和奖励 map，更新 `rewardConfigMap`，并实现模式的 `PlayerScript` 派生类以覆盖需要的钩子。
- 本模块大量直接调用玩家对象的方法（如 `UpdatePlayerSetting`、`learnSpell`、`KillPlayer` 等），在修改行为时请注意服务器端持久化与线程安全（基于 AzerothCore 的脚本模型一般已处理）。

快速定位
- 主要实现：`ChallengeModes.cpp`
- 接口声明：`ChallengeModes.h`
- 配置读取：`ChallengeModes_WorldScript::LoadConfig`
- 玩家创建端选择处理：`ChallengeMode_PlayerCreateScript::OnPlayerLogin`
- 神像交互：`gobject_challenge_modes::OnGossipHello/OnGossipSelect`

示例（配置字符串）
- `Hardcore.TitleRewards = "10 5000,20 5001"` 表示在等级 10 和 20 分别授予 title id 5000、5001。