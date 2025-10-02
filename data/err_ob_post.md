---
title: 【Hexo】Butterfly主题站点运行时间实现n年m天格式显示
description: 详细介绍如何修改Hexo Butterfly主题，使站点运行时间显示支持"n年n天"格式，提供更友好的时间展示方式。
date: 2025-10-02 16:30:00
updated: 2025-10-02 17:54:38
tags:
  - Hexo
  - 博客建站
  - AI
categories:
  - - 差生文具多
    - 博客建站
abbrlink: 7397014466
summary: 这里是慕雪的小助手，这篇文章介绍了为Hexo Butterfly主题添加站点运行时间显示格式的功能。慕雪通过添加配置选项，支持传统天数显示和更直观的年天格式（如“3年154天”），并确保在运行时间不足一年时智能显示天数以避免“0年”的情况。文章详细说明了配置文件修改、前端配置传递、核心时间计算逻辑以及显示优化，强调保持向下兼容性，提升用户体验。
---

{% note info modern %}
慕雪前排提醒：本文所有内容包括修改点都是AI分析、修改并总结的，纯粹是AI写的，没有半点人工（除了两张截图是我贴上去的）
{% endnote %}


## 前言

在博客网站的侧边栏中，显示站点运行时间是一个很常见的功能。通常我们会看到"1265 天"这样的显示方式，但对于长时间运行的站点来说，这种显示方式不够直观。最近我为Hexo Butterfly主题增加了一个新功能，支持将站点运行时间显示为"3 年 154 天"的格式，这样用户可以更直观地了解站点的运行历史。

本文将详细介绍这个功能的实现过程，包括配置项的添加、JavaScript代码的修改以及相关文件的调整。

## 功能概述

这个修改主要实现了以下功能：

1. **配置选项**：在主题配置中添加`runtime_format`选项，支持两种格式：
   - `days`：默认格式，显示总天数（如：1265 天）
   - `year_day`：年天格式，显示年数和剩余天数（如：3 年 154 天）

2. **智能显示**：当运行时间不足一年时，仍显示天数格式，避免出现"0 年"的尴尬情况

3. **向下兼容**：保持与原有功能的完全兼容，默认使用原有的天数显示方式

## 实现步骤

### 1. 配置文件修改

首先需要在主题配置文件`_config.butterfly5.yml`中添加新的配置选项：

```yaml
aside:
  card_webinfo:
    # 站点运行时间起始日期
    runtime_date: 04/16/2022 00:00:00
    # 运行时间格式：days（默认）或 year_day
    # days: 显示天数（如：1265 天）
    # year_day: 显示年天格式（如：3 年 154 天）
    runtime_format: year_day
```

同时在`merge_config.js`中添加默认配置：

```javascript
// themes/butterfly5/scripts/events/merge_config.js
aside: {
  card_webinfo: {
    post_count: true,
    last_push_date: true,
    sort_order: null,
    runtime_date: null,
    runtime_format: 'days'  // 新增默认配置
  }
}
```

### 2. 前端配置传递

在`config.pug`模板中将配置传递给前端JavaScript：

```pug
// themes/butterfly5/layout/includes/head/config.pug
script.
  window.GLOBAL_CONFIG = {
    // ... 其他配置
    runtime: '!{theme.aside.card_webinfo.runtime_date ? _p("aside.card_webinfo.runtime.unit") : ""}',
    runtime_format: '!{theme.aside.card_webinfo.runtime_format || "days"}',
    // ... 其他配置
  }
```

### 3. 核心时间计算逻辑

主要的时间计算逻辑在`utils.js`中的`diffDate`函数中实现：

```javascript
// themes/butterfly5/source/js/utils.js
diffDate(date, more = false) {
  const datePost = new Date(date)
  const dateNow = new Date()
  const diffTime = dateNow - datePost
  const diffSecond = Math.round(diffTime / 1000)
  const diffDay = Math.floor(diffSecond / 86400)
  const diffMonth = diffDay / 30
  const { dateSuffix } = GLOBAL_CONFIG

  if (!more) {
    const totalDays = Math.floor(diffDay)

    // 如果启用了年天格式转换
    if (GLOBAL_CONFIG.runtime_format === 'year_day' && totalDays >= 365) {
      const years = Math.floor(totalDays / 365)
      const days = totalDays % 365

      if (years > 0 && days > 0) {
        return `${years} 年 ${days} 天`
      } else if (years > 0) {
        return `${years} 年`
      } else {
        return `${days} 天`
      }
    }

    return totalDays
  }

  // ... 其他时间差处理逻辑
}
```

### 4. 显示逻辑优化

在`main.js`中优化显示逻辑，避免重复显示单位：

```javascript
// themes/butterfly5/source/js/main.js
showRuntime() {
  const $runtimeCount = document.getElementById('runtimeshow')
  if ($runtimeCount) {
    const publishDate = $runtimeCount.getAttribute('data-publishDate')
    const runtimeText = btf.diffDate(publishDate)

    // 如果使用年天格式且已经包含年或天字，则不再添加单位
    if (GLOBAL_CONFIG.runtime_format === 'year_day' &&
        (runtimeText.includes('年') || runtimeText.includes('天'))) {
      $runtimeCount.textContent = runtimeText
    } else {
      $runtimeCount.textContent = `${runtimeText} ${GLOBAL_CONFIG.runtime}`
    }
  }
}
```

## 关键实现细节

### 1. 时间计算逻辑

核心的时间计算逻辑基于以下规则：

- 1年 = 365天（不考虑闰年的复杂性）
- 使用整数除法和取模运算分别计算年数和剩余天数
- 只有当总天数大于等于365天时，才使用年天格式

### 2. 显示格式处理

处理显示格式时需要注意：

- 年天格式已包含单位，无需额外添加
- 传统天数格式仍需添加"天"单位
- 避免显示"0 年"的情况，提升用户体验

### 3. 配置兼容性

为了保持向下兼容：

- 新配置项有默认值
- 旧配置不受影响
- 可以随时切换显示格式

## 效果展示

修改后的效果如下：

**传统格式**：
```
站点运行时间 : 1265 天
```

![](https://img.musnow.top/i/2025/10/843745f8690d07d7fe6143044decef52.webp)

**年天格式**：
```
站点运行时间 : 3 年 154 天
```

![](https://img.musnow.top/i/2025/10/595c1de37856356519361800bc08b63d.webp)


对于运行时间较短的站点（不足一年），两种格式都会显示为天数，避免出现"0 年"的显示。

## 总结

这个功能的实现相对简单，但能显著提升用户体验。通过合理的配置设计和代码实现，我们既保持了向后兼容性，又提供了更灵活的时间显示方式。

主要收获：

1. 配置驱动：通过配置项控制功能，提高灵活性
2. 渐进增强：在不破坏原有功能的基础上添加新特性
3. 用户体验：考虑各种边界情况，提供友好的显示方式
4. 代码组织：合理分离配置、计算和显示逻辑

这种实现方式可以作为其他主题功能扩展的参考模板。如果你也想为自己的Hexo主题添加类似功能，可以参考本文的实现思路。


