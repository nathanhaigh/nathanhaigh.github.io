---
layout: page
title: About
tagline: 
---
{% include JB/setup %}

Welcome to my brand-new blog "Gluten for Punishment". I intend to start blogging about things of relevance to
wheat bioinformatics.

Bread wheat is a hexaploid plant, think 3 diploid genomes in one species, and has a genome size of 17Gbps and
is full of highly repetitive elements. The generation of a bread wheat reference genome is underway by the
International Wheat Genome Sequencing Consortium ([IWGSC](http://www.wheatgenome.org/)). Due to the size and
repetitive nature of wheat,
a BAC-by-BAC assembly approach is being undertaken on individual chromosomal arms. However, this is a long
process. In the meantime, the IWGSC have released Illumina shotgun assemblies of the arms in the form of
ABySS assembled Chromosomal Survey Sequence ([CSS](https://urgi.versailles.inra.fr/download/iwgsc/)) contigs.

It is of great world importance as it is accounts for a large proportion of the calories consumed in the diet.
Wheat is of particular interest to Australia as it is one of the major cereal crops grown here and yield is
affected by abiotic stresses such as drought and salt. Therefore, much effort is being put into delivering new
cultivars which are more drought and salt tolerant.

Here is a list of blog posts:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
