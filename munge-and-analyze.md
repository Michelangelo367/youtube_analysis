Youtube Video Data: Cleaning & Analysis
================
Hope Johnson
May 24 2018

``` r
library(tidyverse)
library(jsonlite)
library(stringr)
library(lubridate)
library(magrittr)
library(skimr)
```

Opening
=======

This document contains cleaning functions, summary tables, and plots for all videos uploaded to the Alex Jones Channel between between January 1st, 2015 and May 4th, 2018. Throughout this document, individual code chunks are viewable by pushing the "code"" button the to the right of each section. To view all chunks in this document, select "Show All Code"" from the dropdown menu in the upper right. The data-generating process calls on a raw .json file of videos pulled from Google's Youtube Data API. The code to extract such data lives in `get-dat.py`, available in the code folder.

Munge data
==========

First, I write a number of helper functions to convert a nested list item into a dataframe. I want to build up a [tidy](https://cran.r-project.org/web/packages/tidyr/vignettes/tidy-data.html) dataframe, where each variable has its own column and each video has its own row.

``` r
# Note: The first two functions are not my own. See: 
# https://stackoverflow.com/questions/35444968/read-json-file-into-a-data-frame-without-nested-lists 

col_fixer <- function(x, vec2col = FALSE) {
  if (!is.list(x[[1]])) {
    if (isTRUE(vec2col)) {
      as.data.table(data.table::transpose(x))
    } else {
      vapply(x, toString, character(1L))
    }
  } else {
    temp <- rbindlist(x, use.names = TRUE, fill = TRUE, idcol = TRUE)
    temp[, .time := sequence(.N), by = .id]
    value_vars <- setdiff(names(temp), c(".id", ".time"))
    dcast(temp, .id ~ .time, value.var = value_vars)[, .id := NULL]
  }
}

Flattener <- function(indf, vec2col = FALSE) {
  require(data.table)
  require(jsonlite)
  indf <- flatten(indf)
  listcolumns <- sapply(indf, is.list)
  newcols <- do.call(cbind, lapply(indf[listcolumns], col_fixer, vec2col))
  indf[listcolumns] <- list(NULL)
  cbind(indf, newcols)
}

#' @return flattened video item with a column for each nested piece of information from json
make_tidy <- function(item){
  colNames <- c("categoryId", "channelId", "channelTitle", "defaultAudioLanguage", "description", 
                "liveBroadcastContent", "localized", "tags", "publishedAt", "thumbnails", "title")
  item <- data.frame(item)
  { if (length(colNames) != length(names(item$snippet))) {
      missingCols <- setdiff(union(names(item$snippet), colNames), intersect(names(item$snippet), colNames))
      item$snippet[, missingCols] <- ""   
      if ("tags" %in% missingCols) {item$snippet$tags <- list(0)}
      item$snippet <- item$snippet[colNames]  
      item$snippet[ ,order(names(item$snippet))]
      }  
  } 
  Flattener(item) %>%
  as_data_frame()
}

#' @return clean, flat dataframe from .json file where each unique video is a row
clean_full_dat <- function(file){
  df <- fromJSON(file) %>% 
  pull(items) %>%
  lapply(make_tidy) %>% 
  bind_rows() %>%
  select(-(contains("thumbnails"))) %>%
  rename_(.dots = setNames(names(.), 
                           gsub("snippet.","", names(.)))) %>% 
  rename_(.dots = setNames(names(.), gsub("statistics.","", names(.)))) %>% 
  mutate_at(vars(contains("Count")), funs(as.numeric)) %>%
  mutate(publishedAt = ymd_hms(publishedAt),
         year = year(publishedAt)) %>%
    # the etag has something to do with the video being updated (possibly, edited)
    select(-etag) %>% 
    # keep the video observation with the highest viewCount (likely the most up-to-date)
    arrange(id, desc(viewCount)) %>%
    group_by(id) %>%
    filter(row_number() == 1) %>%
    ungroup()
  return(df)
}
```

I convert the raw data from it's original .json format to one that can be written as a sensible .csv file. To simplify this task, I use the package `jsonlite`, available for download on CRAN. The helper functions that go into creating the main function, `clean_full_dat`, can be found in the first code chunk on this document.

``` r
full_dat <- clean_full_dat("../data/vid_details.json") 
#write_csv(full_dat, "data/video_metadata.csv")
```

`Full_dat` contains video titles, viewer statistics, and other details for all the videos uploaded to theAlexJonesChannel between between January 1st, 2015 and May 4th, 2018.

Filter data
===========

To create a dataset with only the words identified by researchers as relevant, I perform a search of the following attributes of each video: tags, description, and title. If any of the relevant terms show up in any of those three items, I save the video to a secondary dataframe, `ms_dat`. Note that any escape characters attached to relevant\_terms (e.g., "buzzfeed/n", indicating a new paragraph) will still be found in the search function.

``` r
relevant_terms <- c("Mainstream media",
              "MSM",
              "CNN",
              "Mtv",
              "buzzfeed",
              "Nytimes",
              "ny times",
              "New york times",
              "Wapo",
              "Washington post",
              "Msnbc",
              "Lamestream",
              "propaganda")
pattern <- paste0(relevant_terms, collapse = "|")

mainstream_filter <- function(data, feature) {
  select <- str_detect(data[[feature]], regex(paste0("(?i)", pattern)))
  data <- data[select, ]
  return(data)
}

subset_by_tags <- mainstream_filter(full_dat, "tags")
subset_by_description <- mainstream_filter(full_dat, "description")
subset_by_title <- mainstream_filter(full_dat, "title")
```

Below, I stack three dataframes on top of each other. These three dataframes represent the results with keywords in (1) the video description, (2) the video tags, and (3) the video title. Many of the videos in these three dataframes are repetitions of one another. I perform an operation, `distinct`, to only keep unique videos in the mainstream subset.

``` r
ms_dat <- subset_by_description %>%
  bind_rows(subset_by_tags) %>%
  bind_rows(subset_by_title) %>%
  distinct() 
#write.csv(ms_dat, "data/video_metadata_mainstream_media.csv")
```

Exploratory Analysis
====================

In the interest of time, I limit my exploration and analysis to the numeric variables. With more time, I will implement an in-depth analysis of the `description` text for each video. Below, I print out summary information for the numeric variables available to us.

The first table provides **summary statistics for videos with mainstream-media tags**. The second table provides **summary statistics for all videos**. I immediately notice a few things:

1.  `favoriteCount` doesn't provide any information.
2.  The average number of dislikes on the videos in our mainstream subset is higher than the average for all videos. The average number of dislikes in the mainstream subset is 845, whereas the average number of dislikes for all videos is 639. That said, the standard deviation is also much greater for the mainstream subset than for the full dataset. This means that the number of likes are less tightly clustered around the mean for the videos in the mainstream subset relative to the full dataset.
3.  The same is true for number of likes, despite the fact that the average number of views for the videos in the mainstream subset is lower than for the overall videos. The average number of likes for the videos in the mainstream subset is 4854, whereas the average number of likes for all videos is 4508.

Skim summary statistics
n obs: 125
n variables: 19

Variable type: numeric

|      variable     | missing | complete |  n  |   mean  |     sd    |  p0  |  p25  |  p50  |   p75  |  p100 |   hist   |
|:-----------------:|:-------:|:--------:|:---:|:-------:|:---------:|:----:|:-----:|:-----:|:------:|:-----:|:--------:|
|    commentCount   |    0    |    125   | 125 |  1563.2 |   2502.8  |  19  |  317  |  689  |  2084  | 20858 | ▇▁▁▁▁▁▁▁ |
|    dislikeCount   |    0    |    125   | 125 |  844.89 |  5165.73  |   4  |   38  |   97  |   342  | 57228 | ▇▁▁▁▁▁▁▁ |
|   favoriteCount   |    0    |    125   | 125 |    0    |     0     |   0  |   0   |   0   |    0   |   0   | ▁▁▁▇▁▁▁▁ |
|     likeCount     |    0    |    125   | 125 | 4854.27 |  6418.85  |  130 |  969  |  1944 |  6827  | 39234 | ▇▂▁▁▁▁▁▁ |
|     viewCount     |    0    |    125   | 125 |  2e+05  | 383485.94 | 4518 | 26729 | 56265 | 228781 | 3e+06 | ▇▁▁▁▁▁▁▁ |
| Skim summary stat |  istics |          |     |         |           |      |       |       |        |       |          |
|     n obs: 506    |         |          |     |         |           |      |       |       |        |       |          |
|  n variables: 19  |         |          |     |         |           |      |       |       |        |       |          |

Variable type: numeric

|    variable   | missing | complete |  n  |    mean   |     sd    |  p0  |    p25   |  p50  |    p75   |   p100  |   hist   |
|:-------------:|:-------:|:--------:|:---:|:---------:|:---------:|:----:|:--------:|:-----:|:--------:|:-------:|:--------:|
|  commentCount |    0    |    506   | 506 |  1993.57  |  5781.52  |  19  |   315.5  |  610  |   1622   |  92533  | ▇▁▁▁▁▁▁▁ |
|  dislikeCount |    0    |    506   | 506 |   638.5   |   3189.8  |   4  |    37    |  89.5 |  291.25  |  57228  | ▇▁▁▁▁▁▁▁ |
| favoriteCount |    0    |    506   | 506 |     0     |     0     |   0  |     0    |   0   |     0    |    0    | ▁▁▁▇▁▁▁▁ |
|   likeCount   |    0    |    506   | 506 |  4508.17  |  8716.95  |  107 |   885.5  |  1913 |   4930   |  122859 | ▇▁▁▁▁▁▁▁ |
|   viewCount   |    0    |    506   | 506 | 226621.85 | 526681.42 | 4518 | 24896.25 | 55635 | 177095.5 | 6756754 | ▇▁▁▁▁▁▁▁ |

### Proportion of likes/dislikes to views

The summary tables above warrant an exploratory comparison of viewer engagement. With the data available, we quantify viewer engagement with the proportion of likes/dislikes to views. A higher proportion of dis/likes to views suggests increased audience engagement and responsiveness. I first reshape the data to facilitate such a comparison, and then present results in a density plot.

``` r
toPlot <- full_dat %>%
  mutate(main_stream_desc = str_detect(description, regex(paste0("(?i)", pattern))),
         main_stream_title = str_detect(title, regex(paste0("(?i)", pattern))),
         main_stream_tags = str_detect(tags, regex(paste0("(?i)", pattern)))) %>%
  mutate(main_stream = ifelse((main_stream_desc | main_stream_tags | main_stream_title), "Mainstream-related videos", "Non mainstream-related videos"),
         Likes = likeCount/viewCount,
         Dislikes = dislikeCount/viewCount,
         Comments = commentCount/viewCount) %>%
  select(main_stream, Likes, Dislikes, Comments) %>%
  gather(var, prop, Likes:Comments) 

ggplot(toPlot) + 
  geom_density(aes(x=prop, y=..scaled.., fill = main_stream), alpha = 0.5) + 
  facet_wrap(~var) + 
  labs(x = "Proportion of engagement type to views", 
       y = "Scaled density", 
       title = "Viewer engagement by video type") + 
  theme(legend.position = "bottom", legend.direction = "horizontal") + 
  scale_fill_discrete("")
```

![](munge-and-analyze_files/figure-markdown_github/add_props-1.png)

The y-axis on this plot is a scaled density because the sample sizes are different for videos with mainstream-tags and those without. The x-axis shows the proportion of the engagement type (comments, dislikes, likes) to the number of views for each video. The shading indicates whether the video had a mainstream tag or not.

The pink tail to the right within the "Likes" cell suggests that viewer engagement is higher for videos with mainstream tags than those without. This illuminates a potential motivation of mainstream media-related content creation. Previous research suggests that malicious actors creating and spreading disinformation, propaganda, and/or fake news are usually motivated by one or more of the following categories: ideology, money, and/or status or attention \[0-1\]. The differential in likes and views suggests that videos related to mainstream media garner more attention than the average video from the Alex Jones channel.

Closing
=======

``` r
version
```

    ##                _                           
    ## platform       x86_64-apple-darwin15.6.0   
    ## arch           x86_64                      
    ## os             darwin15.6.0                
    ## system         x86_64, darwin15.6.0        
    ## status                                     
    ## major          3                           
    ## minor          5.0                         
    ## year           2018                        
    ## month          04                          
    ## day            23                          
    ## svn rev        74626                       
    ## language       R                           
    ## version.string R version 3.5.0 (2018-04-23)
    ## nickname       Joy in Playing

Citations
=========

\[0\] Alice Marwick and Rebecca Lewis, "Media Manipulation and Disinformation Online" (Data & Society Research Institute, May 17, 2017), <https://datasociety.net/pubs/oh/DataAndSociety_MediaManipulationAndDisinformationOnline.pdf>.

\[1\] Robyn Caplan and danah boyd, "Mediation, Automation, Power" (Data & Society Research Institute, May 15, 2016), <https://datasociety.net/pubs/ap/MediationAutomationPower_2016.pdf>.