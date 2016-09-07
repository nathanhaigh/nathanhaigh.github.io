---
layout: page
title: Gluten for Punishment&#58; A Wheat Bioinformaticians Blog
tagline: 
---
{% include JB/setup %}

Welcome to my blog "Gluten for Punishment: A Wheat Bioinformaticians Blog". I named it this, since bread wheat
contains gluten, proteins responsible for the elastic texture of dough. And since the genome of bread wheat is
17Gbp, highly repetitive and hexaploid (think 3 more-or-less homozygous diploid genomes in one), you really
need to be a [glutton for punishment](http://www.oxforddictionaries.com/definition/english/a-glutton-for-punishment)
if you want to work in wheat bioinformatics!

I work as a Research Fellow in Bioinformatics at the Australian Centre for Plant Functional Genomics
([ACPFG](http://www.acpfg.com.au)) in Adelaide, Australia. Much of my work is covered by IP, so I have to be somewhat
careful with what I post. 

Here is a list of blog posts:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
