---
layout: page
title: 关于
description: 人在码途
keywords: tongkun
comments: true
menu: 关于
permalink: /about/
---

人学习有三种途径，一种是自书本上学前人的知识，一种是自身边的人身上学其先进，一种是向自己过去的经验和教训学习。

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

