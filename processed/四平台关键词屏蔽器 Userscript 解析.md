---
title: 四平台关键词屏蔽器 Userscript 解析
source: raw_20260712_210952_abf4460c
source_url: https://www.zhihu.com/question/1920892831981106398/answer/1930304825150669292
created: '2026-07-12'
updated: '2026-07-12'
tags:
- Userscript
- 信息过滤
- 工具脚本
- 注意力管理
- 内容农场
- 知乎
status: processed
note_type: content
related: []
---

# 四平台关键词屏蔽器 Userscript 解析

## 核心结论
> 这是一款由用户「化纤粒」开发并托管在 Greasy Fork 的 Tampermonkey 用户脚本，可在知乎、微博、B站、小红书四大平台按关键词过滤低营养信息流内容。脚本通过 MutationObserver 监听 DOM 变化，并采用「整容器删除」而非隐藏的方式避免空白占位，还附带可视化的屏蔽词管理面板与按站点开关功能。

## 详细内容
## 背景与起源

脚本回答的原始知乎问题是「在知乎上泡这么久，工作、生活、工资还是没有起色是什么原因？」，作者化纤粒的切入点是：与其反思自己，不如直接用技术手段屏蔽掉那些情绪化、对立性、低营养的内容流。本脚本 Fork 自两位原作者的开源作品，并入了 MIT 协议：

- [[三无用户]] 的「知乎屏蔽词修改器」
- [[Andante]] 的「微博/B站/小红书屏蔽器」

## 安装与运行环境

- 脚本管理器：[[Tampermonkey]]（篡改猴）或 [[脚本猫]]，Chrome / Edge 均可
- 脚本市场：[[Greasy Fork]]，当前版本 2.2.1
- 匹配域名：`zhihu.com` / `xiaohongshu.com` / `bilibili.com` / `weibo.com`

## 核心实现机制

### 关键词与站点配置

通过 `getCurrentSite()` 根据 `hostname` 判断当前平台，每个站点维护独立的 CSS 选择器配置（`siteConfigs`）：

| 平台 | 容器选择器 | 标题选择器 |
|---|---|---|
| 知乎 | `.ContentItem` | `.ContentItem-title a` |
| 小红书 | `section.note-item` | `a.title, .title` |
| B站 | `.bili-feed-card, .bili-video-card` | `.bili-video-card__info--tit` |
| 微博 | `.wbpro-scroller-item` | `.wbpro-feed-content .detail_wbtext_4CRf9` |

默认关键词覆盖约 40 个高频干扰词（如「985」「211」「原生家庭」「生物爹」「漫展」「健身」等），用户自定义词条通过 `localStorage` 的 `keyword_blocker_words` 键持久化。

### 「整容器删除」策略

与简单的 `display: none` 不同，`processContentElement()` 在匹配到关键词后，会向上查找外层卡片并直接 `.remove()`：

- 知乎 → `.Card.TopstoryItem.TopstoryItem-isRecommend`
- B站 → `.feed-card` / `.bili-feed-card`
- 小红书 → `.note-item`

这是该脚本相对同类工具的关键改进——彻底避免屏蔽后留下空白占位框。B站还额外清理了「反屏蔽提示容器」（带 `is-rcmd` 但缺失 `enable-no-interest` 的卡片）。

### 实时性保障

三层触发机制应对无限滚动与异步加载：

1. [[MutationObserver]] 监听 `document.body` 的 `childList` 与 `subtree`，新增节点即时处理
2. `scroll` 事件（passive + 1s 防抖）兜底
3. `setInterval(processAllContent, 5000)` 定时扫描

防抖函数 `debounce`（500ms）防止高频触发。

## UI 与异常处理

页面左侧浮动「屏蔽词管理」竖排按钮，点击展开面板：

- **新增**：输入框支持逗号或斜杠分隔的批量添加，去空格后去重插入数组头部
- **删除**：点击「删除」→ 显示「确认删除 / 手滑了」二次确认组
- **按站点禁用**：写入 `keyword_blocker_disabled_sites`，下次刷新生效；主要用于绕过小红书屏蔽后白色空框的已知 bug

`init()` 中先检查 `isCurrentSiteDisabled()`，若已停用则只渲染管理 UI、不执行屏蔽逻辑。

## 适用场景与局限

**适用**：希望减少信息流中性别对立、学历焦虑、原生家庭、消费主义话题曝光的用户，本质是一种 [[注意力管理]] 工具。

**局限**：
- 关键词匹配为子串包含，存在误伤（如屏蔽「妈」会波及「妈妈再打我一次」类无关内容）
- 依赖平台 DOM 结构，UI 改版后需更新选择器
- 小红书容器删除仍存在 bug，需依赖「停用」按钮绕过

## 相关笔记

- [[Tampermonkey 用户脚本开发笔记]]
- [[信息流内容农场识别与过滤]]
- [[注意力管理工具集]]

## 相关笔记
- (待关联引擎填充)