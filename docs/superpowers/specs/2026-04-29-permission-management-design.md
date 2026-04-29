# 权限管理功能设计文档

## 1. 概述

### 1.1 目标
为 IELTS Vocab App 增加内容访问控制，区分免费用户和付费（高级）用户，实现双层分级的内容权限管理。

### 1.2 核心原则
- **简洁标记式**：通过 `isPremium` 字段标记内容，`isPremiumUser` 标记用户状态
- **本地优先**：当前阶段仅本地状态存储，不对接支付系统
- **渐进演进**：设计上预留支付集成点，未来可平滑对接 IAP/激活码

### 1.3 术语
| 术语 | 含义 |
|------|------|
| 免费用户 | `isPremiumUser = false`，仅可访问基础卡片学习功能 |
| 高级用户 | `isPremiumUser = true`，可访问全部内容和功能 |
| 付费内容 | `isPremium = true` 的词汇及相关高级记忆方法 |

## 2. 数据模型变更

### 2.1 Word 模型
新增 `isPremium` 字段：

```dart
Word {
  id: String
  word: String
  phonetic: String
  meanings: [{ partOfSpeech: String, definition: String }]
  examples: [String]
  roots: String
  mnemonic: String
  topic: String
  level: String
  frequency: Int
  isPremium: Bool          // 新增：是否为付费内容，默认 false
}
```

### 2.2 UserSettings 模型
新增 `isPremiumUser` 字段：

```dart
UserSettings {
  dailyNewWords: Int
  reminderTime: Time
  soundEnabled: Bool
  isPremiumUser: Bool      // 新增：用户是否已解锁高级版，默认 false
}
```

### 2.3 词汇数据标记策略
在本地 JSON 词汇数据中，通过 `isPremium` 字段标记每个单词：
- 基础级别 + 高频词（frequency 排名靠前）→ `isPremium: false`（免费）
- 进阶级别 + 专项话题包 → `isPremium: true`（付费）

## 3. 权限控制层

### 3.1 PremiumProvider
基于 Riverpod 的权限状态管理：

```dart
// 用户高级状态 Provider
@riverpod
bool isPremiumUser(IsPremiumUserRef ref) {
  return ref.watch(settingsProvider).isPremiumUser;
}

// 内容访问权限检查
@riverpod
bool canAccessContent(CanAccessContentRef ref, {required bool contentIsPremium}) {
  if (!contentIsPremium) return true;
  return ref.watch(isPremiumUserProvider);
}
```

### 3.2 解锁操作
通过 SettingsProvider 更新用户状态：

```dart
// 解锁高级版（当前为本地模拟，未来替换为支付验证）
void unlockPremium() {
  settings.isPremiumUser = true;
  // 持久化到本地存储
}
```

### 3.3 工作流程
1. UI 组件调用 `canAccessContent(contentIsPremium: word.isPremium)`
2. 若返回 `true`，正常展示内容
3. 若返回 `false`，弹出 `PremiumUpgradeDialog` 引导升级
4. 用户点击"立即解锁" → 调用 `unlockPremium()` → 状态更新 → UI 刷新

## 4. 免费与付费内容边界

| 功能/内容 | 免费用户 | 高级用户 |
|-----------|---------|---------|
| 基础卡片学习（英文+发音+中文释义） | 可用 | 可用 |
| 全部词汇包 | 仅免费词汇（isPremium=false） | 全部词汇 |
| 词根词缀拆解 | 不可用，弹窗引导升级 | 可用 |
| 联想记忆 | 不可用，弹窗引导升级 | 可用 |
| 语境例句 | 不可用，弹窗引导升级 | 可用 |
| 图片辅助 | 不可用，弹窗引导升级 | 可用 |
| 学习数据统计 | 可用（掌握度分布、打卡天数） | 可用（全部：分布、打卡、趋势图、正确率、增长曲线） |
| 搜索与浏览 | 可用 | 可用 |
| 每日学习/复习 | 可用 | 可用 |
| 数据备份 | 可用 | 可用 |

## 5. UI 交互设计

### 5.1 付费内容锁定标识
- **词库浏览页**：付费单词卡片右上角显示小锁图标（`Icons.lock`）
- **学习页**：遇到付费单词时，高级记忆方法区域（词根词缀、联想记忆、例句）显示模糊遮罩 + 锁图标

### 5.2 升级弹窗（PremiumUpgradeDialog）
- **触发时机**：免费用户点击或触达付费内容时自动弹出
- **弹窗内容**：
  - 标题："解锁完整内容"
  - 说明：简要列出高级版包含的内容（全部词汇、词根词缀、联想记忆、例句等）
  - 主按钮："立即解锁" → 将 `isPremiumUser` 设为 `true` 并持久化
  - 次按钮："暂不" → 关闭弹窗
- **设计风格**：沿用现有紫蓝渐变风格，卡片圆角 14-16px

### 5.3 个人中心状态展示
- 在"我的"页面顶部用户信息区域显示状态徽章
- 免费用户显示"免费版"标签，点击可触发升级弹窗
- 高级用户显示"高级版"标签

## 6. 涉及文件

| 文件 | 变更类型 | 说明 |
|------|---------|------|
| `lib/models/word.dart` | 修改 | 新增 `isPremium` 字段 |
| `lib/models/user_settings.dart` | 修改 | 新增 `isPremiumUser` 字段 |
| `lib/providers/premium_provider.dart` | 新增 | 权限检查 Provider |
| `lib/providers/settings_provider.dart` | 修改 | 增加 `unlockPremium()` 方法 |
| `lib/widgets/premium_upgrade_dialog.dart` | 新增 | 升级弹窗组件 |
| `lib/widgets/word_card.dart` | 修改 | 集成权限检查和锁定标识 |
| `lib/widgets/study_progress_bar.dart` | 可能修改 | 学习页进度相关调整 |
| 词汇 JSON 数据文件 | 修改 | 为每个单词添加 `isPremium` 标记 |

## 7. 验收标准

1. 免费用户在词库页可看到付费词汇，但带有锁定图标
2. 免费用户在学习页仅能看到基础卡片内容（英文+发音+中文释义），高级记忆方法区域显示锁定遮罩
3. 点击付费内容时弹出升级弹窗，点击"立即解锁"后状态持久化，重启 App 后仍为高级用户
4. 高级用户可正常访问所有内容，无任何锁定标识或弹窗
5. 权限状态变更后 UI 立即刷新，无需手动切换页面
6. 现有免费功能（基础卡片学习、搜索、每日学习/复习、数据备份）不受影响

## 8. 技术约束与注意事项

- 当前阶段"立即解锁"为本地模拟操作，实际将 `isPremiumUser` 直接设为 `true`
- 预留 `unlockPremium()` 作为未来支付集成的替换点——对接 IAP 或激活码时只需修改此方法内部实现
- `isPremiumUser` 状态需持久化到本地存储（Isar/Hive），确保 App 重启后状态不丢失
- 词汇 JSON 数据中的 `isPremium` 字段需在数据准备阶段完成标记
