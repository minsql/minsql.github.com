---
layout: page
title: About MINSQL
excerpt: "밥먹고 살기 힘든 부부 DBA"
modified: 2014-08-08T19:44:38.564948-04:00

---


<div class="author-columns">
<ul>
    {% for author in site.data.authors %}

    <li>
        <h3>{{ author[1].name  }}</h3>
        {% if author[1].picture contains 'http' %}
          <img src="{{ author[1].picture }}" class="author-avatar u-photo" alt="{{ author[1].name }} bio photo"/>
        {% elsif author[1].picture %}
          <img src="{{ author[1].picture }}" class="author-avatar u-photo" alt="{{ author[1].name }} bio photo"/>
        {% endif %}
        {{ author[1].bio  }}
        <p></p>
        	{% if author[1].twitter %}<a href="https://twitter.com/{{ author[1].twitter }}" title="{{ author[1].name}} on Twitter" target="_blank"><i class="fab fa-twitter-square"></i></a>{% endif %}
        	{% if author[1].facebook %}<a href="https://facebook.com/{{ author[1].facebook }}" title="{{ author[1].name}} on Facebook" target="_blank"><i class="fab fa-facebook-square"></i></a>{% endif %}
        	{% if author[1].google.plus %}<a href="https://plus.google.com/+{{ author[1].google.plus }}" title="{{ author[1].name}} on Google+" target="_blank"><i class="fab fa-google-plus-square"></i></a>{% endif %}
        	{% if author[1].google.userid %}<a href="https://plus.google.com/u/0{{ author[1].google.userid }}" title="{{ author[1].name}} on Google+" target="_blank"><i class="fab fa-google-plus-square"></i></a>{% endif %}          
        	{% if author[1].linkedin %}<a href="https://linkedin.com/in/{{ author[1].linkedin }}" title="{{ author[1].name}} on LinkedIn" target="_blank"><i class="fab fa-linkedin"></i></a>{% endif %}
        	{% if author[1].stackexchange %}<a href="{{ author[1].stackexchange }}" title="{{ author[1].name}} on StackExchange" target="_blank"><i class="fab fa-stack-exchange"></i></a>{% endif %}
        	{% if author[1].instagram %}<a href="https://instagram.com/{{ author[1].instagram }}" title="{{ author[1].name}} on Instagram" target="_blank"><i class="fab fa-instagram"></i></a>{% endif %}
        	{% if author[1].flickr %}<a href="https://www.flickr.com/photos/{{ author[1].flickr }}" title="{{ author[1].name}} on Flickr" target="_blank"><i class="fab fa-flickr"></i></a>{% endif %}
        	{% if author[1].github %}<a href="https://github.com/{{ author[1].github }}" title="{{ author[1].name}} on Github" target="_blank"><i class="fab fa-github-square"></i></a>{% endif %}
        	{% if author[1].tumblr %}<a href="http://{{ author[1].tumblr }}.tumblr.com" title="{{ author[1].name}} on Tumblr" target="_blank"><i class="fab fa-tumblr-square"></i></a>{% endif %}
          {% if author[1].pinterest %}<a href="https://www.pinterest.com/{{ author[1].pinterest }}/" title="{{ author[1].name}} on Pinterest" target="_blank"><i class="fab fa-pinterest"></i></a>{% endif %}
        	{% if author[1].weibo %}<a href="https://www.weibo.com/u/{{ author[1].weibo }}/" title="{{ author[1].name}} on Weibo" target="_blank"><i class="fab fa-weibo fa-2x"></i></a>{% endif %}

    </li>
    {% endfor %}
</ul>
</div>
