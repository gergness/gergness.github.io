---  
layout: post  
title: Introducing osmar2  
author: Greg  
tags: osmar2  
---

osmar2 is a version of the [osmar
package](http://cran.r-project.org/package=osmar) ("OpenStreetMap and
R") rewritten to use xml2 package instead of XML, and some other
adaptations to allow for much quicker reading of .osm files. It's
available on [github](https://github.com/gergness/osmar2).

The part I wanted to spend some time writing about here was the
optimization and debugging process that went into making it faster.
Going into this project, I was pretty sure that the package there were
inefficiencies in reading files because even though the files were
relatively small (10Mb), osmar was already taking several minutes to
load, whereas a `read.csv()` (or better, `readr::read_csv()`) could read
that size in less than a second.

So, I first thought that just switching over to xml2 would be most of
what I needed, but when I had finished rewriting the code using the
original osmar package's approach updated with xml2 funcitons instead of
XML, there wasn't much of an improvement.What ended up speeding things
up was a change in the parsing algorithm.

Here's a sample osm xml file, which I've simplified so that only has a
few nodes and have removed some tags that aren't used (more details on
osm files [here](http://wiki.openstreetmap.org/wiki/OSM_XML)).

    #>  ...
    #>  <node id="506685546" lat="44.9493365" lon="-93.1692413" version="1"/>
    #>  <node id="506685547" lat="44.9496459" lon="-93.1691436" version="1">
    #>      <tag k="highway" v="turning_circle"/>
    #>  </node>
    #>  <node id="530739183" lat="44.9399208" lon="-93.1875805" version="1">
    #>      <tag k="name" v="Davanni&#39;s Pizza &#38; Hot Hoagies"/>
    #>      <tag k="amenity" v="restaurant"/>
    #>      <tag k="cuisine" v="italian"/>
    #>  </node>
    #>  ...

One of the slowest part of parsing the file in the original osmar
package is when creating a dataset of the tag variables. This dataset
needs one row per `tag` node, but includes a variable from that `tag`'s
parent `node` (the `id` variable). `id` from the node. The original way
I attacked this looked something like this:

    old_method <- function() {
      # Find all of the nodes with at least one tag
      nodes <- xml_find_all(osm_xml_root, "/osm/node[./tag]")
      
      # Pull those nodes' ids
      ids <- xml_attr(nodes, "id")
      
      # Pull those nodes' (1 or more) tags into a list of data.frames()
      tags <- lapply(nodes, function(parent_node) {
        tag_nodes <- xml_find_all(parent_node, "./tag")
        
        data_frame(k = xml_attr(tag_nodes, "k"), 
                   v = xml_attr(tag_nodes, "v"))
      })
      
      # And convert that to a single long data.frame
      out_df <- data_frame(ids, 
                           tags = tags) %>%
        unnest()
      
      out_df
    }

    old <- old_method() 
    old %>% print(n = 3)
    #> # A tibble: 4,616 x 3
    #>         ids       k               v
    #>       <chr>   <chr>           <chr>
    #> 1 187845021 highway mini_roundabout
    #> 2 187854927 highway mini_roundabout
    #> 3 187854980  noexit             yes
    #> # ... with 4,613 more rows

As I began to dig into this, I learned that some of the xml2 functions
allow you to perform the entire operations in C and so are very fast,
while others force the code into R objects and so have much slower
looping. So, it doesn't always make sense to try to save your place in
the xml document to avoid having to repeatedly search for the same
spots, because this may force you to loop in R rather than C. For
example:

    system.time(
      osm_xml_root %>% 
        xml_find_all("/osm/node") %>%
        xml_find_all("tag/@k") %>% 
        xml_text()
    )
    #>    user  system elapsed 
    #>   3.005   0.029   3.035

    # Is 30X slower than this:

    system.time(
      osm_xml_root %>%
        xml_find_all("/osm/node/tag/@k") %>%
        xml_text()
    )
    #>    user  system elapsed 
    #>   0.120   0.003   0.123

    # So even though the second method requires us to search from the top
    # of the tree for the k and v variable separately, it is still faster.

The only problem being that we need a way to find out which node id this
particular key/value belong to. This requires some tricky(-ish) xpath.

    # Find all nodes with at least one tag and get their id
    nodes <- xml_find_all(osm_xml_root, "/osm/node[./tag]")
    id <- xml_attr(nodes, "id")
    # Find out how many tags are below each of these nodes and
    # repeat the id that many times
    lens <- xml_find_num(nodes, "count(./tag)")
    id <- rep(id, lens)

Putting it all together, we get:

    new_method <- function() {
      # Find all of the nodes with at least one tag and pull their ids
      nodes <- xml_find_all(osm_xml_root, "/osm/node[./tag]")
      ids <- xml_attr(nodes, "id")
      
      # Find out how many tags are below each of these nodes and
      # repeat the id that many times
      lens <- xml_find_num(nodes, "count(./tag)")
      ids <- rep(ids, lens)
      
      # Pull the tag's keys and values
      keys <- osm_xml_root %>%
        xml_find_all("/osm/node/tag/@k") %>%
        xml_text()
      
      values <- osm_xml_root %>%
        xml_find_all("/osm/node/tag/@v") %>%
        xml_text()
      
      data_frame(ids = ids,
                 k = keys,
                 v = values)
    }
    new <- new_method()
    new %>% print(n = 3)
    #> # A tibble: 4,616 x 3
    #>         ids       k               v
    #>       <chr>   <chr>           <chr>
    #> 1 187845021 highway mini_roundabout
    #> 2 187854927 highway mini_roundabout
    #> 3 187854980  noexit             yes
    #> # ... with 4,613 more rows

Which gives the same result, but much more quickly than the old method:

    microbenchmark(old = old <- old_method(), 
                   new = new <- new_method(), 
                   times = 5)
    #> Unit: milliseconds
    #>  expr       min       lq      mean    median       uq       max neval
    #>   old 3648.1691 3697.593 3906.4411 3949.5129 4015.202 4221.7289     5
    #>   new  354.0688  374.710  380.0945  378.4144  380.622  412.6573     5

    identical(old, new)
    #> [1] TRUE

This makes me wonder whether it'd be easier for xml2 to have a function
along the lines of `xml_find_list()` so that you could more easily
select multiple queries using the quicker C code. I'm not exactly sure
how this would work though. Also, possibly I'm missing something and
there's an easier way to do this. If there is, [let me
know](mailto:gdfe.co.mail@gmail.com).
