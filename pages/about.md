---
layout: page
title: About
description: 个人杂谈
keywords: 我是真的狗, 胡国强, huguoqiang, 我是真的GOU
comments: true
menu: 关于
permalink: /about/
---

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
{% if site.url contains 'huguoqiang0520.github.io' %}
<li>
微信公众号：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ site.url }}/assets/images/qrcode.jpg" alt="我是真的GOU" />
</li>
{% endif %}
</ul>

## Skill Keywords

{% for skill in site.data.skills %}

### {{ skill.name }}

<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
