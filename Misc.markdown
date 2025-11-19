---
layout: page
title: Misc
permalink: /Misc/
---

<p class="misc-intro">
These articles aren't displayed on the homepage as they represent personal interests and side topics, rather than my main focus on computer engineering and cybersecurity projects. Consider them a curious peek into other subjects that catch my attention!
</p>

{%- if site.posts.size > 0 -%}
<ul class="post-list">
    {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    {%- for post in site.posts -%}
        {%- if post.tags contains "Misc" -%}
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
