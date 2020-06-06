---
layout: page
title: About
description: 九旋之猿
keywords: crystalxmumu, 小飞猪, hk, 孔雀王子, 九旋之猿
comments: false
menu: 关于
permalink: /about/
---

我是小飞猪，编码使我快乐，犹如猪在天上飞。

希望自己可以成为九旋之渊之人。

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
{% if site.url contains 'crystalxmumu' %}
<li>
微信公众号：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ assets_base_url }}/assets/images/qrcode.jpg" alt="九旋之猿" />
</li>
{% endif %}
</ul>


## 技能

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
