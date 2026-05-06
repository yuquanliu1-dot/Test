# 词根记忆页面设计

## 概述

为 IELTS 词汇应用增加词根记忆浏览功能。用户可以从词库页进入词根词典，按前缀/词根/后缀三类浏览所有词根，查看词根详情（来源、记忆技巧、关联词根），并跳转到包含该词根的单词详情。

## 需求背景

当前应用已有 `Word` 模型中的 `roots` 字段（`Root` 类型，含 type/text/meaning），但缺少独立的词根浏览入口和丰富的词根信息展示。用户希望通过词根系统性学习单词构词规律，提升记忆效率。

## 页面结构

### 1. 入口（词库页改造）

**位置**：`WordLibraryScreen`，搜索框与筛选条之间

**样式**：渐变背景横向卡片，左侧图标 + "词根学习" 文字 + 右侧箭头

**交互**：点击后 `Navigator.push` 进入 `RootListScreen`

**不变项**：底部导航不变，现有筛选逻辑不变，仅新增一个入口卡片。

### 2. 词根列表页（RootListScreen）

**布局**：
- AppBar：标题「词根词缀」，带返回按钮
- TabBar：三个 Tab — `前缀` / `词根` / `后缀`
- TabBarView：每个 Tab 下是词根卡片列表

**词根卡片内容**：
- 词根文本（大号粗体）
- 含义（中英文，如 "out · 向外"）
- 右侧 badge：关联单词数量（如 "3 词"）

**排序**：每个 Tab 内按词根 text 字母升序排列

**交互**：点击卡片 → `Navigator.push` 进入 `RootDetailScreen`，传入 `RootInfo` 对象

### 3. 词根详情页（RootDetailScreen）

**布局**（可滚动 SingleChildScrollView）：

1. **头部区域**：词根文本（32px 粗体）+ 类型 badge（前缀/词根/后缀，彩色标签）+ 含义说明
2. **来源卡片**：显示词根的语言来源（如「源自希腊语 bios」）
3. **记忆技巧卡片**：一段帮助记忆的文字
4. **关联词根**：横向滚动的 Chip 列表，点击可跳转到对应词根详情页
5. **关联单词列表**：
   - 标题「包含此词根的单词」+ 数量
   - 每个单词一行：单词文本 + 首个中文释义
   - 点击单词：调用现有 `showModalBottomSheet` 展示单词完整详情（复用词库页已有的底部弹窗交互）

## 数据模型

### 新增 RootInfo 模型

```dart
class RootInfo {
  final String text;            // 词根文本，如 "bio"
  final String type;            // prefix / root / suffix
  final String meaning;         // 含义，如 "life 生命"
  final String origin;          // 来源，如 "希腊语 bios"
  final String memoryTip;       // 记忆技巧
  final List<String> relatedRoots; // 关联词根的 text 列表
}
```

### 关联逻辑

`RootInfo.text` 与 `Word.roots[].text` 通过精确匹配关联。在词根详情页加载时，遍历所有 `Word`，筛选 `roots` 中包含该 text 的单词。

### 数据文件

新建 `lib/data/root_data.dart`，预置所有词根的 `RootInfo` 列表。数据覆盖当前 `sample_words.dart` 中出现的所有词根，包括：

**前缀 (prefix)**：e-, de-, infra-, contro-, com-, ac-, al-
**词根 (root)**：labor, sustain, phen, curr, structure, bio, diversity, nov, terior, assess, vers, prehens, demo, graph, rod, facil, impl, ubique, ped, agog, lev, auto, nom, quis
**后缀 (suffix)**：-able, -ment, -itate

## 新增文件清单

| 文件 | 说明 |
|------|------|
| `lib/models/root_info.dart` | RootInfo 数据模型 |
| `lib/data/root_data.dart` | 词根详情预置数据 |
| `lib/screens/root_list_screen.dart` | 词根列表页（含 Tab 切换） |
| `lib/screens/root_detail_screen.dart` | 词根详情页 |

## 修改文件清单

| 文件 | 修改内容 |
|------|----------|
| `lib/screens/word_library_screen.dart` | 在搜索框下方新增「词根学习」入口卡片 |

## 设计约束

- 不修改底部导航栏
- 不修改现有 `Word` 和 `Root` 模型
- 不修改现有单词详情交互（底部弹窗）
- 复用 `AppTheme` 中的颜色、渐变和文字样式
- 词根详情数据为内置静态数据，不涉及网络请求
