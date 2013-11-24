---
layout: docsdefault
title: Example applications
permalink: /example-apps.html

partof: documentation
num: 7
outof: 8
---


This section contains walkthroughs for several benchmark applications.
These walkthroughs are meant to show you the capabilities of the collection API,
in the same time instructing you how to use (parallel) collections to create larger applications.
All the examples in this section are executable and can be run or tweaked by downloading
the [source code](TODO) repository.
Some of the applications even contain applets runnable directly from the website.

<br/>

<div>
</div>



{% for pg in site.pages %}
  {% if pg.partof == "examples" and pg.outof %}
    {% assign totalPagesTour = pg.outof %}
  {% endif %}
{% endfor %}
{% if totalPagesTour %}
<ul class="bullets">
  {% for i in (1..totalPagesTour) %}
    {% for pg in site.pages %}
      {% if pg.partof == "examples" and pg.num and pg.num == i %}
      <div class="examples">
        <a class="examples" href="{{ homedir }}/{{ pg.url }}"><table>
          <tr>
            <td><img class="imageframe-icon" src="{{ pg.image }}" width="48" height="48"/></td>
            <td><h2>
              {{ pg.title }}
            </h2></td>
          </tr>
        </table></a>
        <div>{{ pg.description }}</div>
      </div>
      {% endif %}
    {% endfor %}
  {% endfor %}
</ul>
{% else %}
  **ERROR**. Couldn't find the total number of pages in this set of tutorial articles. Have you declared the `outof` tag in your YAML front matter?
{% endif %}

