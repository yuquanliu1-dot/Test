# 生词列表功能设计规格

## 概述

为 IELTS Vocab App 新增「生词列表」功能，自动记录学习中标记为"不认识"或"模糊"的单词，提供独立的生词浏览和专项练习能力。连续 2 次答"认识"后自动移除。

## 需求摘要

- **触发条件**：学习时答"不认识"或"模糊"自动加入生词本
- **移除条件**：连续 2 次答"认识"后自动移除
- **入口**：新增底部「生词」Tab（5 Tab 导航）
- **页面功能**：生词列表展示 + 生词专项练习

## 数据模型

### NewWordEntry

```dart
class NewWordEntry {
  final String wordId;            // 关联 Word.id
  final DateTime addedAt;         // 首次加入生词本的时间
  final int consecutiveCorrect;   // 连续答"认识"次数（0/1/2）
}
```

- 持久化到 SharedPreferences，key 为 `'new_words'`，存储为 JSON 数组
- 序列化格式：`{ "wordId": "xxx", "addedAt": "ISO8601", "consecutiveCorrect": 0 }`

### 生词逻辑规则

| 学习回答 | 对生词本的影响 |
|---------|--------------|
| "不认识" | 如不在生词本则添加；如已存在则重置 consecutiveCorrect = 0，更新 addedAt |
| "模糊" | 如不在生词本则添加；如已存在则重置 consecutiveCorrect = 0 |
| "认识" | 如在生词本中，consecutiveCorrect + 1；达到 2 则从生词本移除 |

## 页面结构

### 底部导航栏（5 Tab）

| 顺序 | 图标 | 标签 |
|------|------|------|
| 1 | `school` | 学习 |
| 2 | `bookmark` | 生词 |
| 3 | `menu_book` | 词库 |
| 4 | `bar_chart` | 统计 |
| 5 | `person` | 我的 |

### 生词页面（NewWordsPage）

顶部 TabBar 切换两个子视图：「列表」和「练习」。

#### 列表视图

- 顶部统计：「共 N 个生词」
- 复用 `_WordTile` 样式展示生词，额外显示添加时间（如「3 天前」）
- 点击查看单词详情，复用 `VocabPage` 的 DraggableScrollableSheet 弹窗
- 列表为空时展示引导文案：「学习时标记为"不认识"或"模糊"的单词会自动出现在这里」

#### 练习视图

- 复用 `WordCard` + `StudyProgressBar` 组件
- 仅对生词列表中的单词进行卡片练习
- 答题逻辑：复用三按钮（不认识/模糊/认识），每次答题更新 consecutiveCorrect 计数
- 练习完成展示完成页面
- 练习中答"认识"达标的单词实时从列表移除

## 文件变更清单

### 新建文件

| 文件 | 说明 |
|------|------|
| `lib/models/new_word_entry.dart` | NewWordEntry 数据模型 |
| `lib/repositories/new_words_repository.dart` | 生词持久化仓库（基于 StorageService） |
| `lib/providers/new_words_provider.dart` | Riverpod StateNotifier，管理生词状态 |
| `lib/pages/new_words_page.dart` | 生词页面（列表 + 练习两个子视图） |

### 修改文件

| 文件 | 修改内容 |
|------|---------|
| `lib/main.dart` | 底部导航栏从 4 Tab 改为 5 Tab，插入生词 Tab |
| `lib/providers/study_provider.dart` | `answerCard()` 中添加生词本联动逻辑 |

## 集成要点

1. `StudyProvider.answerCard()` 答题后调用 `NewWordsNotifier` 更新生词记录
2. `NewWordsProvider` 依赖 `wordRepoProvider` 获取完整 Word 对象
3. 专项练习的答题独立于每日学习进度，不影响 SM-2 的 nextReview 计算，但更新 consecutiveCorrect 计数
4. 不影响收藏、统计、个人中心等现有功能
