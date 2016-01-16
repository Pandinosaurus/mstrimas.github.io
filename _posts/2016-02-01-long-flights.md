---
layout: post
title: "Mapping the Longest Commericial Flights in R"
published: true
excerpt: >
  Mapping the longest regularly scheduled commercial flights in the world using
  R and ggplot2.
category: spatial
tags: r spatial gis
---





On more than one occasion I've taken the brutally long-haul flight from Toronto to Hong Kong with Air Canada. Given that I'm totally unable to sleep on planes, almost 16 hours crammed into a tiny economy class seat is pretty rough! This got me thinking: what is the longest regularly scheduled, commericial long-haul flight?

Wikipedia has the answer (no surprise) in the form of a table listing the [top 30 longest flights by distance](https://en.wikipedia.org/wiki/Non-stop_flight#Longest_flights). Turns out the longest flight is from Dallas to Syndey, clocking in at almost 17hours. This is 1.5 hours longer than my Hong Kong-Toronto flight, which comes in at number 24 on the list.

Of course, I couldn't resist scraping these data from Wikipedia and mapping the flights. I'm trying to improve my ggplot skills, so I'll stick to this package for the visualization.

## Required packages


```r
library(sp)
library(raster)
library(rgeos)
library(plyr)
library(dplyr)
library(rvest)
library(stringr)
library(tidyr)
library(lubridate)
library(ggplot2)
library(ggmap)
library(ggrepel)
```

# Scraping and cleaning

The `rvest` package makes web scraping a breeze. I just read the html, extract out any tables, pick the first table on the page (this is the only one I'm interested in), and parse it into a dataframe with `html_table()`.


```r
flights <- read_html('https://en.wikipedia.org/wiki/Non-stop_flight') %>% 
  html_nodes('.wikitable') %>% 
  .[[1]] %>% 
  html_table(fill = TRUE)
```

As usual there are some issues with the imported data. First, the Wikipedia table has cells spanning multiple rows corresponding to flights on the same route with different airlines. The `rvest` explicitely states that it can't handle rows spanning multiple columns. In addition, the column headers are not nice variable names.

<img src="/img/multi-row-cell.png" style="display: block; margin: auto;" />

I fix these issues below.


```r
# variable names
names(flights) <- c("rank", "from", "to", "airline", "flight_no", "distance",
                    "duration", "aircraft", "first_flight")
# cells spanning multiple rows
row_no <- which(is.na(flights$first_flight))
problem_rows <- flights[row_no, ]
fixed_rows <- flights[row_no - 1, ]
fixed_rows$rank <- problem_rows[, 1]
fixed_rows$airline <- problem_rows[, 2]
fixed_rows$flight_no <- problem_rows[, 3]
fixed_rows$duration <- problem_rows[, 4]
fixed_rows$aircraft <- problem_rows[, 5]
flights <- flights[-row_no, ]
flights <- rbind(flights, fixed_rows) %>% 
  arrange(rank)
```

The next step is cleaning the data, and there are a variety of issues here:
1. Footnotes need to be cleaned out of some cells 
2. Destinations sometimes have city and airport
3. Some routes have multiple flight numbers for the same airline
4. Distances are given in three units all within the same cell
5. Durations aren't in a nice format to work with
5. Some routes have different durations for winter and summer

Nothing `stringr` and some regular expressions can't handle!


```r
flights <- flights %>% 
  mutate(rank = as.integer(str_extract(rank, "^[:digit:]+")),
         from = str_extract(from, "^[[:alpha:] ]+"),
         to = str_extract(to, "^[[:alpha:] ]+"))
# make multiple flight numbers comma separated
flights$flight_no <- str_replace_all(flights$flight_no, "[:space:]", "") %>% 
  str_extract_all("[:alpha:]+[:digit:]+") %>% 
  laply(paste, collapse = ",")
# only consider distances in km, convert to integer
flights$distance <- str_extract(flights$distance, "^[0-9,]+") %>% 
  str_replace(",", "") %>% 
  as.integer
# convert duration to minutes and separate into summer/winter schedules
flights <- str_match_all(flights$duration, "([:digit:]{2}) hr ([:digit:]{2}) min") %>% 
  llply(function(x) {60 * as.integer(x[, 2]) + as.integer(x[, 3])}) %>% 
  llply(function(x) c(x, x)[1:2]) %>% 
  do.call(rbind, .) %>% 
  data.frame %>% 
  setNames(c("duration_summer", "duration_winter")) %>% 
  mutate(duration_max = pmax(duration_summer, duration_winter)) %>% 
  cbind(flights, .)
# first_flight to proper date
flights$first_flight <- str_extract(flights$first_flight, "^[0-9-]+") %>% 
  ymd %>% 
  as.Date
flights <- flights %>% 
  dplyr::select(rank, from, to, airline, flight_no, distance, 
         duration = duration_max, duration_summer, duration_winter,
         first_flight)
```

Now the table is in a nice clean format and ready for display.


```r
dplyr::select(flights, rank, from, to, airline, distance, duration) %>% 
  kable(format.args =  list(big.mark = ','),
        col.names = c("rank", "from", "to", "airline", 
                      "distance (km)", "duration (min)"))
```



| rank|from         |to            |airline               | distance (km)| duration (min)|
|----:|:------------|:-------------|:---------------------|-------------:|--------------:|
|    1|Dallas       |Sydney        |Qantas                |        13,804|          1,015|
|    2|Johannesburg |Atlanta       |Delta Air Lines       |        13,582|          1,000|
|    3|Abu Dhabi    |Los Angeles   |Etihad Airways        |        13,502|            990|
|    4|Dubai        |Los Angeles   |Emirates              |        13,420|            995|
|    5|Jeddah       |Los Angeles   |Saudia                |        13,409|          1,015|
|    6|Doha         |Los Angeles   |Qatar Airways         |        13,367|            985|
|    7|Dubai        |Houston       |Emirates              |        13,144|            980|
|    8|Abu Dhabi    |San Francisco |Etihad Airways        |        13,128|            975|
|    9|Dallas       |Hong Kong     |American Airlines     |        13,072|          1,025|
|   10|Dubai        |San Francisco |Emirates              |        13,041|            950|
|   11|New York     |Hong Kong     |Cathay Pacific        |        12,983|            975|
|   12|Newark       |Hong Kong     |United Airlines       |        12,980|            960|
|   13|Newark       |Hong Kong     |Cathay Pacific        |        12,980|            950|
|   14|Abu Dhabi    |Dallas        |Etihad Airways        |        12,962|            980|
|   15|Doha         |Houston       |Qatar Airways         |        12,951|            980|
|   16|Dubai        |Dallas        |Emirates              |        12,940|            980|
|   17|New York     |Guangzhou     |China Southern        |        12,878|            965|
|   18|Boston       |Hong Kong     |Cathay Pacific        |        12,827|            950|
|   19|Johannesburg |New York      |South African Airways |        12,825|            965|
|   20|Houston      |Taipei        |EVA Air               |        12,776|            955|
|   21|Doha         |Dallas        |Qatar Airways         |        12,764|            980|
|   22|Los Angeles  |Melbourne     |Qantas                |        12,748|            950|
|   23|Los Angeles  |Melbourne     |United Airlines       |        12,748|            950|
|   24|Toronto      |Hong Kong     |Cathay Pacific        |        12,569|            930|
|   25|Toronto      |Hong Kong     |Air Canada            |        12,569|            935|
|   26|New York     |Taipei        |EVA Air               |        12,566|            970|
|   27|New York     |Taipei        |China Airlines        |        12,566|            920|
|   28|Mumbai       |Newark        |United Airlines       |        12,565|            965|
|   29|Mumbai       |Newark        |Air India             |        12,565|            955|
|   30|Chicago      |Hong Kong     |United Airlines       |        12,542|            925|
|   31|Chicago      |Hong Kong     |Cathay Pacific        |        12,542|            925|

# Geocoding

If I'm going to map these flights, I'll need coordinates for each city in the dataset. Fortunately, the `ggmaps` package has a function for geocoding locations based on their name using Google Maps. I convert the resulting cities and coordinates to a SpatialPointsDataFrame so I can later change the projection of the city coordinates as necessary.


```r
cities_wgs <- c(flights$from, flights$to) %>% 
  unique
cities_wgs[cities_wgs == "Melbourne"] <- "Melbourne, Australia"
cities_wgs <- cities_wgs %>% 
  cbind(city = ., geocode(., output = "latlon", source = "google"))
cities_wgs <- cities_wgs %>% 
  mutate(city = as.character(city),
         city = ifelse(city == "Melbourne, Australia", "Melbourne", city))
coordinates(cities_wgs) <- ~ lon + lat
```

# Global map

As a background on which to map the flight paths, I'll use the global map provided by [Natural Earth](http://www.naturalearthdata.com). First I download, unzip, and load shapefiles for country boundaries, graticules, and a bounding box.


```r
base_url <- 'http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/110m/'
tf <- tempfile()
download.file(paste0(base_url, 'cultural/ne_110m_admin_0_countries.zip'), tf)
unzip(tf, exdir = 'data/long-flights/', overwrite = TRUE)
unlink(tf)
download.file(paste0(base_url, 'physical/ne_110m_graticules_all.zip'), tf)
unzip(tf, exdir = 'data/long-flights/', overwrite = TRUE)
unlink(tf)

world_wgs <- shapefile('data/long-flights/ne_110m_admin_0_countries.shp')
bbox_wgs <- shapefile('data/long-flights/ne_110m_wgs84_bounding_box.shp')
grat_wgs <- shapefile('data/long-flights/ne_110m_graticules_20.shp')
```

This shapfile is more granular than I require, so I aggregate it so that each ISO alpha-2 code corresponds to a single polygon.


```r
world_wgs <- gUnaryUnion(world_wgs, id = world_wgs$iso_a2)
```

These shapefiles are currently in unprojected coordinates (i.e. lat/long), so I project them to the [Winkel tripel projection](https://en.wikipedia.org/wiki/Winkel_tripel_projection), a nice compromise projection for global maps, which is used by National Geographic. I also project the city coordinates.


```r
world_wt <- spTransform(world_wgs, '+proj=wintri')
bbox_wt <- spTransform(bbox_wgs, '+proj=wintri')
grat_wt <- spTransform(grat_wgs, '+proj=wintri')

projection(cities_wgs) <- projection(world_wgs)
cities_wt <- spTransform(cities_wgs, '+proj=wintri')
```

Finally, `ggplot` can't handle spatial objects directly, it only works with data frames. So, I use the `fortify()` function to convert each spatial object to a data frame ready for plotting.


```r
world_wt_df <- fortify(world_wt)
bbox_wt_df <- fortify(bbox_wt)
grat_wt_df <- fortify(grat_wt)
cities_wt_df <- as.data.frame(cities_wt)
```

# Mapping

Now that all the data are prepared, I'll create the map. I build it up in steps here.


```r
set.seed(1)
ggplot() +
  geom_polygon(data = bbox_wt_df, aes(long, lat, group = group), 
               fill = "light blue") +
  geom_path(data = grat_wt_df, aes(long, lat, group = group, fill = NULL), 
            linetype = "dashed", color = "grey70", size = 0.25) +
  geom_polygon(data = world_wt_df, aes(long, lat, group = group), 
               fill = "grey40", color = "grey80", size = 0.05) +
  geom_point(data = cities_wt_df, aes(lon, lat)) +
  geom_text_repel(data = cities_wt_df, aes(lon, lat, label = city),
                  segment.color = "black", segment.size = 0.25,
                  box.padding = unit(0.1, 'lines'), force = 0.5,
                  fontface = "bold", size = 3) +
  coord_equal() +
  theme_nothing()
```

<img src="/figures//2016-02-01-long-flights_map-1.svg" title="plot of chunk map" alt="plot of chunk map" style="display: block; margin: auto;" />
