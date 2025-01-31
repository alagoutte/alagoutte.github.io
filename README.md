Welcome !

Les derniers articles :

<ul>
{% for post in site.posts %}
  <li>
    <a href="{{ post.url }}">{{ post.title }}</a> - <small>{{ post.date | date: "%d %B %Y" }}</small>
  </li>
{% endfor %}
</ul>