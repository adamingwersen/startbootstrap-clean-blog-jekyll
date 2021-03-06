---
layout:     post
title:      "Crawling Wikipedia Search Queries"
subtitle:   "Using rvest and RJSONIO packages in R"
date:       2016-07-16
author:     "Adam"
header-img: "img/docubyte-purple.jpg"
tags:		[R Programming | JSON Scraping | Wikipedia | Bitcoin]
---

<h3><center> What are we doing? </center></h3>

The purpose of this tutorial is to show how to obtain time-series data on any Wikipedia.org article of the number of search-queries towards this particular article. As such, this will not show how to crawl different Wikipedia.org pages for content - but rather how to obtain a single, or, if needed, multiple time-series. 

<h3><center> Which tools are we using? </center></h3>

You need a functioning installation of R with the following packages installed:


{% highlight r %}
pkgs <- c("rvest", 
          "dplyr", 
          "plyr", 
          "stringr", 
          "RJSONIO",
          "ggthemes",
          "knitr")
lapply(pkgs, require, character.only  = TRUE)
{% endhighlight %}

<h3><center> Step One: Create Links </center></h3>

On [this website](http://stats.grok.se/en), there are some graphs on any and all Wikipedia Articles - we are interested in the complete dataset behind one of the 3-monthly time-series presented [here](http://stats.grok.se/en/201601/Bitcoin). I can see a bar-chart for 3 months at a time - but I would like to see a bigger picture...

The solution is to look into the HTML of the website. All the possible dates are contained within a single css.selector. And given the structure of the link, we can do the following:


{% highlight r %}
wikilink = "http://stats.grok.se/en/201601/Bitcoin"
css.selector = "body > form > #year"

BTC_WIKI.dates = read_html(wikilink) %>%
  html_nodes(css = css.selector) %>%
  html_text(trim = FALSE)


BTC_WIKI.dates = sapply(seq(from=1, to=nchar(BTC_WIKI.dates), by=6), function(i) substr(BTC_WIKI.dates, i, i+5))
head(BTC_WIKI.dates, 3)
{% endhighlight %}



{% highlight text %}
## [1] "201607"
## [2] "201606"
## [3] "201605"
{% endhighlight %}

Some dates contain a space at the beginning or end of string, we can fix that by "trimming" the strings:


Now, we can create links for the crawler. We simply want to iterate over all possible dates in the link while keeping the search-term constant. Given that I'm currently interested in Bitcoins, I set my search-term accordingly:



{% highlight r %}
searchterm = "Bitcoin"

groks.base = "http://stats.grok.se/json/en/LANK/"
groks.link = paste0(groks.base, searchterm)

link_str_replace = function(x, y){
  link.replacement = gsub("\\LANK", x, groks.link)
}


links = llply(BTC_WIKI.dates, link_str_replace)
head(links, 3)
{% endhighlight %}



{% highlight text %}
## [[1]]
## [1] "http://stats.grok.se/json/en/201607/Bitcoin"
## 
## [[2]]
## [1] "http://stats.grok.se/json/en/201606/Bitcoin"
## 
## [[3]]
## [1] "http://stats.grok.se/json/en/201605/Bitcoin"
{% endhighlight %}

Now we have a list of viable links. 

<h3><center> Step Two: Deploy crawler </center></h3>

The content on groks.se is formatted as a JSON table. This requires some extra work as the content is nested. Luckily, there's an R-package for that: RJSONIO


{% highlight r %}
crawl_groks = function(x){
  searchinfo_wiki = fromJSON(x, simplifyMatrix = TRUE, flatten = TRUE)
    get.inf = ldply(searchinfo_wiki$daily_views, data.frame)
  return(cbind(get.inf))
}
wiki.jsonget = lapply(links, crawl_groks)

 # Inspecting wiki.get: Nested list of data.frames
 # This can be combatted by ldply to data.frame
wiki.get.df = ldply(wiki.jsonget, data.frame)

## Cleaning data ##
wiki.get.df$date = as.Date(wiki.get.df[,1])                                             # Create new variables 
wiki.get.df$queries = as.numeric(wiki.get.df[,2])

wiki.get.df$X..i.. = NULL                                                               # Deleting old variables
wiki.get.df$.id = NULL
wiki.get.df = wiki.get.df[!(wiki.get.df$queries==0 & wiki.get.df$date>"2010-01-01"),]   # Discard irrelevant dates
wiki.get.df = na.omit(wiki.get.df)

kable(head(wiki.get.df, 5))
{% endhighlight %}



|date       | queries|
|:----------|-------:|
|2016-01-04 |    6170|
|2016-01-05 |    5844|
|2016-01-06 |    5145|
|2016-01-07 |    5827|
|2016-01-01 |    3202|

You now have a dataframe consisting of two columns with a daily search-query count on any Wikipedia Article. 

<h3><center> What do I use it for? </center></h3>

I used this particular data to do a cointegration analysis alongside Google-Trends data and Real Bitcoin Price/Trade Volume data. For illustration purposes, here's a plot:


{% highlight r %}
library(ggplot2)
p = ggplot(data = wiki.get.df, aes(x=date, y=log(queries)))
p = p + geom_line(data = wiki.get.df, aes(y = log(queries)))
p = p + theme_economist() + scale_color_economist() + 
  labs(x = "Date", y = "Log of Queries", title = "Search Queries for Bitcoin \n on Wikipedia - Daily")
plot(p)
{% endhighlight %}

<img src="/blogfigure/source/2016-07-19-Wikipedia-Search/plot-1.png" title="plot of chunk plot" alt="plot of chunk plot" style="display: block; margin: auto;" />
