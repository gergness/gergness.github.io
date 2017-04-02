---  
layout: post  
title: Color Scheme  
author: Greg  
---

I spent an embarassingly long time trying to come up with the color
scheme for this website. I swear I didn't realize that it is the default
colors for a 2 group graph in ggplot2

    library(ggplot2)
    #> Warning: package 'ggplot2' was built under R version 3.3.2
    ggplot(mtcars, aes(x = mpg, y = wt, group = am, color = factor(am))) +
      geom_point() + 
      geom_smooth()
    #> `geom_smooth()` using method = 'loess'

![](/images/blog/2017-03-31/unnamed-chunk-2-1.png)
