---
layout : mylayout
title : Dan Pincas
---

<ul class="posts">
    {% for post in site.posts %}
		<li>
			<div class="idea">
				<h2><a class="postlink" href="{{ post.url }}">{{ post.title }}</a></h2>
				<span class="subtitle"><strong>{{ post.date | date: "%e %B, %Y"  }}</strong> filed under
					{% for tag in post.tags %}
					   <a href="/tag/{{ tag }}">{{ tag }}</a>{% if forloop.rindex0 > 0 %}, {% endif %}
					{% endfor %}
                </span>
                <div class="postpreview">
                    {{ post.content | strip_html | truncate: 300 }}<a href="{{ post.url }}">view more</a>
                </div>
			</div>
		</li>
    {% endfor %}
</ul>

<script type="text/javascript">
//<![CDATA[
(function() {
    var links = document.getElementsByTagName('a');
    var query = '?';
    for(var i = 0; i < links.length; i++) {
    if(links[i].href.indexOf('#disqus_thread') >= 0) {
        query += 'url' + i + '=' + encodeURIComponent(links[i].href) + '&';
    }
    }
    document.write('<script charset="utf-8" type="text/javascript" src="http://disqus.com/forums/pandincus/get_num_replies.js' + query + '"></' + 'script>');
})();
//]]>
</script>