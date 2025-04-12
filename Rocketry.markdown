---
layout: default
title: Rocketry
permalink: /Rocketry/
---

<h1>Rocketry</h1>

{%- if site.posts.size > 0 -%}
<ul class="post-list">
    {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    {%- for post in site.posts -%}
        {%- if post.tags contains "PRT" -%}
        <li>
            <span class="post-meta">{{ post.date | date: date_format }}</span>
            <h3>
                <a class="post-link" href="{{ post.url | relative_url }}">
                    {{ post.title | escape }}
                </a>
            </h3>
            {{ post.excerpt }}
        </li>
        {%- endif -%}
    {%- endfor -%}
</ul>
{%- endif -%}

