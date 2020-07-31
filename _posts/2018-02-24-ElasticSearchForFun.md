---
layout: post
comments: true
title: "Elasticsearch and BeautifulSoup for Fun"
categories: blog
descriptions: Using python beautiful soup and elastic search to index summer camp program 
tags: 
  - elasticsearch
  - elk
  - python
date: 2018-02-25T10:39:55-04:01
---


My wife had been trying to find a summer camp program that may be suitable for our daughter by looking at township website. There are around 200+ program listed, but the search filter is very minimum. We have to click each link to go inside each program and then go to different tabs to see what is the description, age requirement, fee, schedule and other informations. This is kind of annoying task. 

I was curious and start to inspect the html code and found out that they have pretty much consistent layout. It also seems pretty easy to parse. So, i was thinking, if i can crawl and parse this website, i can put them into my elasticsearch instance at my home lab and from there it should be easy to filter the information. 

Long story short, i created the simple crawler using python and beautiful soup 4 and then feed the result to elastic search using elasticsearch python module. 

Here is the snapshot of my elasticsearch data:

![elk_summer_camp]({{site.baseurl}}/assets/images/elk_summer_camp.png)


