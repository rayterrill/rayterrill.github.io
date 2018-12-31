---
title:  "How to Use Jekyll and Github Pages"
tags: [jekyll, github]
---

I had a hell of a time trying to get Github Pages going with the Jekyll support. The basics are pretty easy to get going, but I couldn't get some of the more "advanced" features going without digging around. This is my attempt at documenting some of those learnings.

This page has been an excellent resource: https://jekyllrb.com/docs/

##### Configuring Global Settings for Jekyll/Github Pages
Global configuration goes in the _config.yml file at the root of the repo. Stuff like the global theme configuration, title, author, etc go in here.

##### Modifiying the Layout for the Sites
The layout files are in the _layouts folder. default.html is the base configuration, post.html is use for posts.

##### Creating Posts
1. Create posts in the _posts folder, using the naming YEAR-MONTH-DAY-title.md
2. Create posts using the Markdown format

##### Setting Titles for Pages
To set titles for individual pages, start the page with the following metadata:
```
---
title:  "How to Use Github Pages with Jekyll"
---
```

##### Using Images in Posts
1. Upload images to the assets folder
2. Add the image reference to your blog post like the following:
```
![APIGatewayMultipleStages]({{ site.url }}/assets/APIGatewayMultipleStages.PNG)
```

##### Using Tags or Categories in Posts
To set tags for a post, add the tags section to the front matter for a post:
```
---
layout: post
title: A Trip
categories: [blog, travel]
tags: [hot, summer]
---
```

##### Iterating Tags or Categories
```
{% for category in site.categories %}
  <h3>{{ category[0] }}</h3>
  <ul>
    {% for post in category[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}
```

##### Creating a List of Posts and Excerpts
```
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
```

##### Drafts
Drafts go in the _drafts folder without a date in the filename
