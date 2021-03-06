
<!-- README.md is generated from README.Rmd. Please edit that file -->

[![lifecycle](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)

> Disclaimer: this package is still at an experimental level and under
> active development. You should only use it for testing, reporting bugs
> (which are to be expected), proposing changes to the code, requesting
> features, sending ideas… As long as this package is in “Experimental”
> mode, there might be bugs, and changes to the API are to be expected.
> Read the [NEWS.md](NEWS.md) to be informed of the last changes.

# neo4r

The goal of {neo4r} is to provide a modern and flexible Neo4J driver for
R.

It’s modern in the sense that you results are returned as tibbles
whenever possible, relies on modern tools, and is designed to work with
the pipe. Our goal with this package is to provide a driver that can be
easily integrated in a data analysis workflow, especially by providing
an API that can work smoothly with other data analysis (dplyr or purrr)
and graph packages (igraph, ggraph, visNetwork…).

It’s flexible in the sense that it is rather unopinionated regarding the
way it returns the results, by trying to stay as close as possible to
the way Neo4J returns data. That way, you have the control over the way
you will compute the results. At the same time, the result is not too
complex, so that the “heavy lifting” of data wrangling is not left to
the user.

The connexion object is also an easy to control R6 method, allowing you
to update and query information from the API.

## Installation

You can install {neo4r} from GitHub with:

``` r
# install.packages("devtools")
devtools::install_github("neo4j-rstats/neo4r")
```

## Create a connexion object

Start by creating a new connexion object with `neo4j_api$new`

``` r
library(neo4r)
con <- neo4j_api$new(url = "http://localhost:7474", 
                     user = "plop", password = "pouetpouet")
```

This connexion object is designed to interact with you Neo4J API.

It comes with some methods to retrieve information from it :

``` r
# Test the endpoint, that will not work :
con$ping()
#> [1] 401
```

Being an R6 object, `con` is flexible in the sense that you can change
`url`, `user` and `password` at any time:

``` r
con$reset_user("neo4j")
con$ping()
#> [1] 200
# Or with 
con$password <- "pouetpouet"
```

That means you can at any time connect to another url without having to
generate a new connexion object. (`con$reset_url()`).

``` r
# Get Neo4J Version
con$get_version()
#> [1] "3.3.3"
# List constaints (if any)
con$get_constraints()
#> # A tibble: 3 x 3
#>   label      type       property_keys
#>   <chr>      <chr>      <chr>        
#> 1 Maintainer UNIQUENESS name         
#> 2 Author     UNIQUENESS name         
#> 3 Package    UNIQUENESS name
# Get a vector of labels (if any)
con$get_labels()
#> # A tibble: 3 x 1
#>   labels    
#>   <chr>     
#> 1 Maintainer
#> 2 Package   
#> 3 Author
# Get a vector of relationships (if any)
con$get_relationships()
#> # A tibble: 1 x 1
#>   relationships
#>   <chr>        
#> 1 MAINTAINS
# Get schema 
con$get_schema()
#> # A tibble: 3 x 2
#>   label      property_keys
#>   <chr>      <chr>        
#> 1 Package    name         
#> 2 Author     name         
#> 3 Maintainer name
```

## Call the API

You can either create a separate query or insert it inside the
`call_api` function.

The `call_api` function takes several arguments :

  - query : the cypher query
  - con : the connexion object
  - type : “rows” or “graph”: wether to return the results as a list of
    results in tibble, or as a graph object (with `$nodes` and
    `$relationships`)
  - output : the output format (r or json)
  - include\_stats : whether or not to include the stats about the call
  - meta : wether or not to include the meta arguments of the nodes when
    calling with “rows”

> At the end of the developping process of all the packages planned, you
> will be able to write your queries and pipe them with {cyphersugar},
> which offers a syntactic sugar on top of cypher.

### “rows” format

When you’re calling the API, you can choose to returns a list of
tibbles. You get as many objects as you specified in the RETURN cypher
statement.

``` r
library(magrittr)

'MATCH (p:Package) RETURN p.name AS nom LIMIT 5' %>%
  call_api(con)
#> $nom
#> # A tibble: 5 x 1
#>   value      
#>   <chr>      
#> 1 A3         
#> 2 abbyyR     
#> 3 abc        
#> 4 ABCanalysis
#> 5 abc.data
```

By default, results are returned as an R list of tibbles. We choose to
implement it this way, as we think this is the more “truthful” regarding
the way you call Neo4J.

For example, when you want to return two nodes, you’ll get two results,
in the form of two tibbles (p.name and dep.name
here):

``` r
'MATCH (p:Package) <-[:MAINTAINS]-(main:Maintainer) RETURN p.name AS nom, main.name AS maintainer LIMIT 5' %>%
  call_api(con)
#> $nom
#> # A tibble: 5 x 1
#>   value     
#>   <chr>     
#> 1 A3        
#> 2 abbyyR    
#> 3 abc.data  
#> 4 abc       
#> 5 AdaptGauss
#> 
#> $maintainer
#> # A tibble: 5 x 1
#>   value             
#>   <chr>             
#> 1 scott fortmann-roe
#> 2 gaurav sood       
#> 3 blum michael      
#> 4 blum michael      
#> 5 florian lerch
```

The result is a two elements list with each element being labelled as
what you specified in the cypher query.

Results can also be returned in
JSON:

``` r
'MATCH (p:Package) <-[:MAINTAINS]-(main:Maintainer) RETURN p.name AS nom, main.name AS maintainer LIMIT 5' %>%
  call_api(con, output = "json")
#> [
#>   [
#>     {
#>       "row": [
#>         ["A3"],
#>         ["scott fortmann-roe"]
#>       ],
#>       "meta": [
#>         {},
#>         {}
#>       ]
#>     },
#>     {
#>       "row": [
#>         ["abbyyR"],
#>         ["gaurav sood"]
#>       ],
#>       "meta": [
#>         {},
#>         {}
#>       ]
#>     },
#>     {
#>       "row": [
#>         ["abc.data"],
#>         ["blum michael"]
#>       ],
#>       "meta": [
#>         {},
#>         {}
#>       ]
#>     },
#>     {
#>       "row": [
#>         ["abc"],
#>         ["blum michael"]
#>       ],
#>       "meta": [
#>         {},
#>         {}
#>       ]
#>     },
#>     {
#>       "row": [
#>         ["AdaptGauss"],
#>         ["florian lerch"]
#>       ],
#>       "meta": [
#>         {},
#>         {}
#>       ]
#>     }
#>   ]
#> ]
```

If you turn the `type` argument to `graph`, you’ll get a graph result:

``` r
'MATCH p=()-[r:MAINTAINS]->() RETURN p LIMIT 5' %>%
  call_api(con, type = "graph")
#> $nodes
#> # A tibble: 9 x 3
#>   id    label     properties
#>   <chr> <list>    <list>    
#> 1 0     <chr [1]> <list [5]>
#> 2 1     <chr [1]> <list [1]>
#> 3 2     <chr [1]> <list [5]>
#> 4 3     <chr [1]> <list [1]>
#> 5 4     <chr [1]> <list [5]>
#> 6 5     <chr [1]> <list [1]>
#> 7 6     <chr [1]> <list [5]>
#> 8 7     <chr [1]> <list [1]>
#> 9 8     <chr [1]> <list [5]>
#> 
#> $relationships
#> # A tibble: 5 x 5
#>   id    type      startNode endNode properties
#>   <chr> <chr>     <chr>     <chr>   <list>    
#> 1 0     MAINTAINS 1         0       <list [0]>
#> 2 1     MAINTAINS 3         2       <list [0]>
#> 3 2     MAINTAINS 5         4       <list [0]>
#> 4 3     MAINTAINS 7         6       <list [0]>
#> 5 4     MAINTAINS 5         8       <list [0]>
#> 
#> attr(,"class")
#> [1] "neo"  "list"
```

The result is returned as one node or relationship by row.

Due to the specific data format of Neo4J, there can be more than one
label and propertie by node and relationship. That’s why the results is
returned, by design, as a list-data.frame.

We have designed several functions to unnest this :

  - #### unnest\_nodes, that can unnest a node dataframe :

<!-- end list -->

``` r
res <- 'MATCH p=()-[r:MAINTAINS]->() RETURN p LIMIT 5' %>%
  call_api(con, type = "graph")
unnest_nodes(res$nodes)
#> # A tibble: 9 x 7
#>   id    label      date       license            name   title      version
#>   <chr> <chr>      <chr>      <chr>              <chr>  <chr>      <chr>  
#> 1 0     Package    2015-08-15 GPL (>= 2)         A3     "Accurate… 1.0.0  
#> 2 1     Maintainer <NA>       <NA>               scott… <NA>       <NA>   
#> 3 2     Package    NA         MIT + file LICENSE abbyyR Access to… 0.5.1  
#> 4 3     Maintainer <NA>       <NA>               gaura… <NA>       <NA>   
#> 5 4     Package    2015-05-04 GPL (>= 3)         abc    Tools for… 2.1    
#> 6 5     Maintainer <NA>       <NA>               blum … <NA>       <NA>   
#> 7 6     Package    2017-03-13 GPL-3              ABCan… Computed … 1.2.1  
#> 8 7     Maintainer <NA>       <NA>               flori… <NA>       <NA>   
#> 9 8     Package    2015-05-04 GPL (>= 3)         abc.d… Data Only… 1.0
```

Note that this will return NA for the properties not in a node. For
example here, we have no ‘licence’ information for the Maintainer node
(that makes sense).

On the long run, and this is not {neo4r} specific but Neo4J related, a
good practice is to have a “name” propertie on each node, so this column
will be full here.

You can also either unnest only the properties or the labels :

``` r
res$nodes %>%
  unnest_nodes(what = "properties")
#> # A tibble: 9 x 7
#>   id    label     date       license            name    title      version
#>   <chr> <list>    <chr>      <chr>              <chr>   <chr>      <chr>  
#> 1 0     <chr [1]> 2015-08-15 GPL (>= 2)         A3      "Accurate… 1.0.0  
#> 2 1     <chr [1]> <NA>       <NA>               scott … <NA>       <NA>   
#> 3 2     <chr [1]> NA         MIT + file LICENSE abbyyR  Access to… 0.5.1  
#> 4 3     <chr [1]> <NA>       <NA>               gaurav… <NA>       <NA>   
#> 5 4     <chr [1]> 2015-05-04 GPL (>= 3)         abc     Tools for… 2.1    
#> 6 5     <chr [1]> <NA>       <NA>               blum m… <NA>       <NA>   
#> 7 6     <chr [1]> 2017-03-13 GPL-3              ABCana… Computed … 1.2.1  
#> 8 7     <chr [1]> <NA>       <NA>               floria… <NA>       <NA>   
#> 9 8     <chr [1]> 2015-05-04 GPL (>= 3)         abc.da… Data Only… 1.0
```

``` r
res$nodes %>%
  unnest_nodes(what = "label")
#> # A tibble: 9 x 3
#>   id    properties label     
#>   <chr> <list>     <chr>     
#> 1 0     <list [5]> Package   
#> 2 1     <list [1]> Maintainer
#> 3 2     <list [5]> Package   
#> 4 3     <list [1]> Maintainer
#> 5 4     <list [5]> Package   
#> 6 5     <list [1]> Maintainer
#> 7 6     <list [5]> Package   
#> 8 7     <list [1]> Maintainer
#> 9 8     <list [5]> Package
```

  - `unnest_relationships`

There is only one nested column in the relationship table, so the
function is quite straightforward :

``` r
unnest_relationships(res$relationships)
#> # A tibble: 5 x 5
#>   id    type      startNode endNode properties
#>   <chr> <chr>     <chr>     <chr>   <chr>     
#> 1 0     MAINTAINS 1         0       <NA>      
#> 2 1     MAINTAINS 3         2       <NA>      
#> 3 2     MAINTAINS 5         4       <NA>      
#> 4 3     MAINTAINS 7         6       <NA>      
#> 5 4     MAINTAINS 5         8       <NA>
```

  - `unnest_graph`

This function takes a graph results, and does `unnest_nodes` and
`unnest_relationships`.

``` r
unnest_graph(res)
#> $nodes
#> # A tibble: 9 x 7
#>   id    label      date       license            name   title      version
#>   <chr> <chr>      <chr>      <chr>              <chr>  <chr>      <chr>  
#> 1 0     Package    2015-08-15 GPL (>= 2)         A3     "Accurate… 1.0.0  
#> 2 1     Maintainer <NA>       <NA>               scott… <NA>       <NA>   
#> 3 2     Package    NA         MIT + file LICENSE abbyyR Access to… 0.5.1  
#> 4 3     Maintainer <NA>       <NA>               gaura… <NA>       <NA>   
#> 5 4     Package    2015-05-04 GPL (>= 3)         abc    Tools for… 2.1    
#> 6 5     Maintainer <NA>       <NA>               blum … <NA>       <NA>   
#> 7 6     Package    2017-03-13 GPL-3              ABCan… Computed … 1.2.1  
#> 8 7     Maintainer <NA>       <NA>               flori… <NA>       <NA>   
#> 9 8     Package    2015-05-04 GPL (>= 3)         abc.d… Data Only… 1.0    
#> 
#> $relationships
#> # A tibble: 5 x 5
#>   id    type      startNode endNode properties
#>   <chr> <chr>     <chr>     <chr>   <chr>     
#> 1 0     MAINTAINS 1         0       <NA>      
#> 2 1     MAINTAINS 3         2       <NA>      
#> 3 2     MAINTAINS 5         4       <NA>      
#> 4 3     MAINTAINS 7         6       <NA>      
#> 5 4     MAINTAINS 5         8       <NA>      
#> 
#> attr(,"class")
#> [1] "neo"  "list"
```

## Convert for common graph packages

Unless otherwise specified, the functions do an `unnest_graph` before
being transformed to a graph object.

### {igraph}

To be converted to a graph object,

  - the nodes need an id, and a name. By defaut, the node name is
    assumed to be found in the “name” property return by the graph, but
    you can specify any other column. The “label” column from Neo4J is
    renamed “group”.

  - relationships needs a start and end, which are startNode and endNode
    in the Neo4J results.

<!-- end list -->

``` r
res %>%
  convert_to("igraph")
#> IGRAPH 0088b29 DN-- 9 5 -- 
#> + attr: name (v/c), group (v/c), date (v/c), license (v/c), title
#> | (v/c), version (v/c), type (e/c), id (e/c), properties (e/x)
#> + edges from 0088b29 (vertex names):
#> [1] scott fortmann-roe->A3          gaurav sood       ->abbyyR     
#> [3] blum michael      ->abc         florian lerch     ->ABCanalysis
#> [5] blum michael      ->abc.data
```

Which means that you can :

``` r
'MATCH p=()-[r:MAINTAINS]->() RETURN p LIMIT 5' %>%
  call_api(con, type = "graph") %>% 
  convert_to("igraph") %>%
  plot()
```

<img src="man/figures/README-unnamed-chunk-15-1.png" width="100%" />

This can also be used with ggraph :

``` r
library(ggraph)
#> Loading required package: ggplot2
'MATCH p=()-[r:MAINTAINS]->() RETURN p LIMIT 10' %>%
  call_api(con, type = "graph") %>% 
  convert_to("igraph") %>%
  ggraph() + 
  geom_node_label(aes(label = name, color = group)) +
  geom_edge_link() + 
  theme_graph()
#> Using `nicely` as default layout
```

<img src="man/figures/README-unnamed-chunk-16-1.png" width="100%" />

### {visNetwork}

``` r
network <- 'MATCH p=()-[r:MAINTAINS]->() RETURN p LIMIT 10' %>%
  call_api(con, type = "graph") %>% 
  convert_to("visNetwork")
visNetwork::visNetwork(network$nodes, network$relationships)
```

## Sending data to the api

You can simply send queries has we have just seen, by writing the cypher
query and call the api.

### Sending an R data.frame

  - `as_nodes` turns a dataframe into a series of nodes :

// Coming soon

  - `as_relationships`

// Coming soon

### Reading and sending a cypher file :

  - `read_cypher` reads a cypher file and returns a tibble of all the
    calls

<!-- end list -->

``` r
read_cypher("data-raw/create.cypher")
#> # A tibble: 53 x 1
#>    cypher                                                                 
#>    <chr>                                                                  
#>  1 CREATE CONSTRAINT ON (p:Band) ASSERT p.name IS UNIQUE;                 
#>  2 CREATE CONSTRAINT ON (p:City) ASSERT p.name IS UNIQUE;                 
#>  3 CREATE CONSTRAINT ON (p:record) ASSERT p.name IS UNIQUE;               
#>  4 CREATE CONSTRAINT ON (p:artist) ASSERT p.name IS UNIQUE;               
#>  5 CREATE (ancient:Band {name: 'Ancient' ,formed: 1992}), (acturus:Band {…
#>  6 CREATE CONSTRAINT ON (p:Person) ASSERT p.name IS UNIQUE;               
#>  7 MATCH (band:Band) WHERE band.formed < 1995 RETURN *;                   
#>  8 MATCH (b:Band) WHERE b.formed = 1990 RETURN *;                         
#>  9 MATCH (b:Band {formed: 1990}) RETURN *;                                
#> 10 MATCH (b:Band) WHERE b.formed < 1995 RETURN *;                         
#> # ... with 43 more rows
```

  - `send_cypher` reads a cypher file, and send it the the API. By
    default, the stats are returned.

<!-- end list -->

``` r
send_cypher("data-raw/constraints.cypher", con)
```

### Sending csv dataframe to Neo4J

The `load_csv_with_headers` sends an csv from an url to the Neo4J
browser.

The args are :

  - `on_load` : the code to execute on load
  - `con` : the connexion object
  - `url` : the url of the csv to send
  - `header` : wether or not the csv has a header
  - `periodic_commit` : the volume for PERIODIC COMMIT
  - `as` : the AS argument for LOAD CSV
  - `format` : the format of the result
  - `include_stats` : whether or not to include the stats
  - `meta` : whether or not to return the meta information

<!-- end list -->

``` r
# Create the constraints
call_api("CREATE CONSTRAINT ON (a:artist) ASSERT a.name IS UNIQUE;", con)
call_api("CREATE CONSTRAINT ON (al:album) ASSERT al.name IS UNIQUE;", con)
```

``` r
# List constaints (if any)
con$get_constraints()
#> # A tibble: 3 x 3
#>   label      type       property_keys
#>   <chr>      <chr>      <chr>        
#> 1 Maintainer UNIQUENESS name         
#> 2 Author     UNIQUENESS name         
#> 3 Package    UNIQUENESS name
# Create the query that will create the nodes and relationships
on_load_query <- 'MERGE (a:artist { name: csvLine.artist})
MERGE (al:album {name: csvLine.album_name})
MERGE (a) -[:has_recorded] -> (al)  
RETURN a AS artists, al AS albums;'
# Send the csv 
load_csv(url = "https://raw.githubusercontent.com/ThinkR-open/datasets/master/tracks.csv", 
         con = con, header = TRUE, periodic_commit = 50, 
         as = "csvLine", on_load = on_load_query)
#> $artists
#> # A tibble: 2,367 x 1
#>    name           
#>    <chr>          
#>  1 Eminem         
#>  2 Eurythmics     
#>  3 Queen          
#>  4 The Police     
#>  5 A$AP Rocky     
#>  6 Tears For Fears
#>  7 Foals          
#>  8 Bag Raiders    
#>  9 Bright Eyes    
#> 10 Bob Dylan      
#> # ... with 2,357 more rows
#> 
#> $albums
#> # A tibble: 2,367 x 1
#>    name                           
#>    <chr>                          
#>  1 Curtain Call (Deluxe)          
#>  2 Sweet Dreams (Are Made Of This)
#>  3 The Game (2011 Remaster)       
#>  4 Synchronicity (Remastered)     
#>  5 LONG.LIVE.A$AP (Deluxe Version)
#>  6 Songs From The Big Chair       
#>  7 Holy Fire                      
#>  8 Bag Raiders (Deluxe)           
#>  9 I'm Wide Awake, It's Morning   
#> 10 Highway 61 Revisited           
#> # ... with 2,357 more rows
#> 
#> $stats
#> # A tibble: 12 x 2
#>    type                  value
#>    <chr>                 <dbl>
#>  1 contains_updates         1.
#>  2 nodes_created         1975.
#>  3 nodes_deleted            0.
#>  4 properties_set        1975.
#>  5 relationships_created 1183.
#>  6 relationship_deleted     0.
#>  7 labels_added          1975.
#>  8 labels_removed           0.
#>  9 indexes_added            0.
#> 10 indexes_removed          0.
#> 11 constraints_added        0.
#> 12 constraints_removed      0.
```

Please note that this project is released with a [Contributor Code of
Conduct](CODE_OF_CONDUCT.md). By participating in this project you agree
to abide by its terms.
