---
layout: post
title: "Mapping the Longest Commericial Flights in R"
published: true
excerpt: >
  Mapping the longest regularly scheduled commercial flights in the world using
  R and ggplot2. Includes a discussion of the challenges associated with maps
  for which the central meridian is not at Greenich.
category: spatial
tags: r spatial gis
---





On more than one occasion I've taken the brutally long-haul flight from Toronto to Hong Kong with Air Canada. Given that I'm totally unable to sleep on planes, almost 16 hours crammed into a tiny economy class seat is pretty rough! This got me thinking: what is the longest regularly scheduled, commericial long-haul flight?

Wikipedia has the answer (no surprise) in the form of a table listing the [top 30 longest flights by distance](https://en.wikipedia.org/wiki/Non-stop_flight#Longest_flights). Turns out the longest flight is from Dallas to Syndey, clocking in at almost 17hours. This is 1.5 hours longer than my Hong Kong-Toronto flight, which comes in at number 24 on the list.

Of course, I couldn't resist scraping these data from Wikipedia and mapping the flights. I'm trying to improve my ggplot mapping skills, so I'll try to pre-processing the data as much as possible and stick to ggplot.

## Required packages


```r
library(sp)
library(raster)
library(rgeos)
library(geosphere)
library(plyr)
library(dplyr)
library(rvest)
library(stringr)
library(tidyr)
library(broom)
library(lubridate)
library(ggplot2)
library(scales)
library(ggmap)
library(ggrepel)
library(ggalt)
library(viridis)
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
  mutate(route = paste(from, to, sep = "-")) %>% 
  dplyr::select(rank, route, from, to, airline, flight_no, distance, 
         duration = duration_max, duration_summer, duration_winter,
         first_flight)
```

Now the table is in a nice clean format and ready for display.


```r
dplyr::select(flights, rank, route, airline, distance, duration) %>% 
  kable(format.args =  list(big.mark = ','),
        col.names = c("rank", "route", "airline", "distance (km)", "duration (min)"))
```



| rank|route                   |airline               | distance (km)| duration (min)|
|----:|:-----------------------|:---------------------|-------------:|--------------:|
|    1|Dallas-Sydney           |Qantas                |        13,804|          1,015|
|    2|Johannesburg-Atlanta    |Delta Air Lines       |        13,582|          1,000|
|    3|Abu Dhabi-Los Angeles   |Etihad Airways        |        13,502|            990|
|    4|Dubai-Los Angeles       |Emirates              |        13,420|            995|
|    5|Jeddah-Los Angeles      |Saudia                |        13,409|          1,015|
|    6|Doha-Los Angeles        |Qatar Airways         |        13,367|            985|
|    7|Dubai-Houston           |Emirates              |        13,144|            980|
|    8|Abu Dhabi-San Francisco |Etihad Airways        |        13,128|            975|
|    9|Dallas-Hong Kong        |American Airlines     |        13,072|          1,025|
|   10|Dubai-San Francisco     |Emirates              |        13,041|            950|
|   11|New York-Hong Kong      |Cathay Pacific        |        12,983|            975|
|   12|Newark-Hong Kong        |United Airlines       |        12,980|            960|
|   13|Newark-Hong Kong        |Cathay Pacific        |        12,980|            950|
|   14|Abu Dhabi-Dallas        |Etihad Airways        |        12,962|            980|
|   15|Doha-Houston            |Qatar Airways         |        12,951|            980|
|   16|Dubai-Dallas            |Emirates              |        12,940|            980|
|   17|New York-Guangzhou      |China Southern        |        12,878|            965|
|   18|Boston-Hong Kong        |Cathay Pacific        |        12,827|            950|
|   19|Johannesburg-New York   |South African Airways |        12,825|            965|
|   20|Houston-Taipei          |EVA Air               |        12,776|            955|
|   21|Doha-Dallas             |Qatar Airways         |        12,764|            980|
|   22|Los Angeles-Melbourne   |Qantas                |        12,748|            950|
|   23|Los Angeles-Melbourne   |United Airlines       |        12,748|            950|
|   24|Toronto-Hong Kong       |Cathay Pacific        |        12,569|            930|
|   25|Toronto-Hong Kong       |Air Canada            |        12,569|            935|
|   26|New York-Taipei         |EVA Air               |        12,566|            970|
|   27|New York-Taipei         |China Airlines        |        12,566|            920|
|   28|Mumbai-Newark           |United Airlines       |        12,565|            965|
|   29|Mumbai-Newark           |Air India             |        12,565|            955|
|   30|Chicago-Hong Kong       |United Airlines       |        12,542|            925|
|   31|Chicago-Hong Kong       |Cathay Pacific        |        12,542|            925|

# Geocoding

If I'm going to map these flights, I'll need coordinates for each city in the dataset. Fortunately, the `ggmaps` package has a function for geocoding locations based on their name using Google Maps.


```r
cities <- c(flights$from, flights$to) %>% 
  unique
cities[cities == "Melbourne"] <- "Melbourne, Australia"
cities <- cities %>% 
  cbind(city = ., geocode(., output = "latlon", source = "google"))
cities <- cities %>% 
  mutate(city = as.character(city),
         city = ifelse(city == "Melbourne, Australia", "Melbourne", city))
```

Now I bring these coordinates into the `flights` dataframe.


```r
flights <- flights %>% 
  left_join(cities, by = c("from" = "city")) %>% 
  left_join(cities, by = c("to" = "city")) %>% 
  rename(lng_from = lon.x, lat_from = lat.x, lng_to = lon.y, lat_to = lat.y)
```

# Flight paths

A [great circle](https://en.wikipedia.org/wiki/Great_circle) is the path on a spherical surface (such as the Earth) that gives the shortest distance between two points. Although I have no way of knowing what the actual flight path is for these routes, it's likely to be reasonably approximated by a great circle. First I subset the flights dataset to only include unique routes.


```r
flights_unique <- flights %>% 
  group_by(route) %>% 
  filter(row_number(desc(duration)) == 1)
```

Then I use the `geosphere` package to get great circle routes for each of the above flights. Since flights over the pacific cross the International Date Line, I use `breakAtDateLine = TRUE` so ensure the great circle lines are broken as they cross.


```r
gc_routes <- gcIntermediate(flights_unique[c("lng_from", "lat_from")],
                            flights_unique[c("lng_to", "lat_to")],
                            n = 360, addStartEnd = TRUE, sp = TRUE, 
                            breakAtDateLine = TRUE)
gc_routes <- SpatialLinesDataFrame(gc_routes, 
                                   data.frame(rank = flights_unique$rank,
                                              route = flights_unique$route,
                                              stringsAsFactors = FALSE))
row.names(gc_routes) <- as.character(gc_routes$rank)
```

The `geosphere` package also provides a function to calculate the maximum latitude reached on a great circle route. This will be useful for identifying routes that pass close to the north pole. I also identify routes crossing the international date line. These routes may need to be mapped with a different projection.


```r
max_lat <- gcMaxLat(flights_unique[c("lng_from", "lat_from")],
                    flights_unique[c("lng_to", "lat_to")])
flights_unique$max_lat <- max_lat[, "lat"]
date_line <- readWKT("LINESTRING(180 -90, 180 90)", p4s = projection(gc_routes))
flights_unique$cross_dl <- gIntersects(gc_routes, date_line, byid=T) %>% 
  as.logical
```

# Global map

As a background on which to map the flight paths, I'll use the global map provided by [Natural Earth](http://www.naturalearthdata.com).


```r
base_url <- 'http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/'
tf <- tempfile()
download.file(paste0(base_url, '110m/cultural/ne_110m_admin_0_countries_lakes.zip'), tf)
unzip(tf, exdir = 'data/long-flights/', overwrite = TRUE)
unlink(tf)
world <- shapefile('data/long-flights/ne_110m_admin_0_countries_lakes.shp')
```

`ggplot` can't handle spatial objects directly, it only works with data frames. So, I use the `tidy()` function from the `broom()` package to convert each spatial object to a data frame ready for plotting.


```r
world_df <- tidy(world)
gc_routes_df <- tidy(gc_routes)
```

# Mapping

Now that all the data are prepared, I'll create the map. Rather than just showing the final product, I'll build it up in steps in the hope that it'll be instructive.

## First attempt

All coordinates are currently in unprojected (i.e. lat/long) coordinates, I project them to the [Winkel tripel projection](https://en.wikipedia.org/wiki/Winkel_tripel_projection), a nice compromise projection for global maps, which is used by National Geographic. Typically, I'd project all my spatial data with `sp::spTransform()` before plotting, but here I'll make use of the new `coord_proj()` function from the [`ggalt` package](https://github.com/hrbrmstr/ggalt) package, which projects coordinates on the fly.


```r
ggplot() +
  geom_polygon(data = world_df, aes(long, lat, group = group), 
               fill = "grey80", color = "grey60", size = 0.1) +
  geom_point(data = cities, aes(lon, lat), color = "grey20", size = 0.5) +
  geom_path(data = gc_routes_df, 
            aes(long, lat, group = group), alpha = 0.5, color = "#fa6900") +
  geom_text(data = cities, aes(lon, lat, label = city),
            size = 3, color = "grey20", alpha = 0.9, nudge_y = 2, 
            check_overlap = TRUE) +
  coord_proj("+proj=natearth +lon_0=0") +
  #coord_proj("+proj=wintri", ylim = c(-50, 90)) +
  scale_x_continuous(breaks = seq(-180, 180, 30)) +
  scale_y_continuous(breaks = seq(-90, 90, 15)) +
  theme(panel.grid.major = element_line(size = 0.5, linetype = 2),
        axis.title = element_blank(),
        axis.text = element_blank(),
        axis.ticks = element_blank())
```

<img src="/figures//2016-02-01-long-flights_first-map-1.svg" title="plot of chunk first-map" alt="plot of chunk first-map" style="display: block; margin: auto;" />

Looks OK, but there's tons of room for improvement.

## Changing central meridian

The default Winkel-Tripel projection takes the Greenich Prime Meridian as it's central meridian. This is a poor choice in this case since every route has a North American City at one end, which results in many of the routes going off the edge of the map. Centering the map on the US (around 90°W) seems the best bet, but this puts the edges of the map at 90°E, right in the middle of Asia. This makes a mess of the polygons spanning the edge.


```r
central_meridian <- -90
#proj <- sprintf("+proj=kav7 +lon_0=%i", central_meridian)
# proj <- sprintf("+proj=wag5 +lon_0=%i", central_meridian)
# proj <- sprintf("+proj=natearth +lon_0=%i", central_meridian)
proj <- sprintf("+proj=wintri +lon_0=%i", central_meridian)
ggplot() +
  geom_polygon(data = world_df, aes(long, lat, group = group), 
               fill = "grey80", color = "grey60", size = 0.1) +
  geom_point(data = cities, aes(lon, lat), color = "grey20", size = 0.5) +
  geom_path(data = gc_routes_df, 
            aes(long, lat, group = group), alpha = 0.5, color = "#fa6900") +
  geom_text(data = cities, aes(lon, lat, label = city),
            size = 3, color = "grey20", alpha = 0.9, nudge_y = 2, 
            check_overlap = TRUE) +
  coord_proj(proj, ylim = c(-50, 90)) +
  scale_x_continuous(breaks = seq(-180, 180, 30)) +
  scale_y_continuous(breaks = seq(-90, 90, 15)) +
  theme(panel.grid.major = element_line(size = 0.5, linetype = 2),
        axis.title = element_blank(),
        axis.text = element_blank(),
        axis.ticks = element_blank())
```

<img src="/figures//2016-02-01-long-flights_messy-boundary-1.svg" title="plot of chunk messy-boundary" alt="plot of chunk messy-boundary" style="display: block; margin: auto;" />

To fix this I've defined a function that removes a narrow sliver around the new edge of the map. Then, once the map is projected, there won't be any artifacts at the boundary.


```r
split <- function(x, lng_center, edge_tol = 1e-8) {
  if (lng_center < -180 || lng_center > 180) {
    stop("invalid longitude")
  }
  
  if (is.projected(x) || is.na(is.projected(x))) {
    stop("split only works with unprojected coordinates")
  }
  
  edge <- (lng_center + 360) %% 360 - 180
  clip <- as(extent(edge - edge_tol, edge + edge_tol, -90, 90), "SpatialPolygons")
  projection(clip) <- projection(x)
  row.names(clip) <- "edge"
  gd <- gDifference(x, clip, byid = TRUE)
  # return features ids to original values
  row.names(gd) <- gsub(" edge$", "", row.names(gd))
  # bring back attribute data
  if (inherits(x, "SpatialPolygonsDataFrame")) {
    gd <- SpatialPolygonsDataFrame(gd, x@data, match.ID = TRUE)
  } else if (inherits(x, "SpatialLinesDataFrame")) {
    gd <- SpatialLinesDataFrame(gd, x@data, match.ID = TRUE)
  }
  gd
}
world_split <- split(world, central_meridian)
world_split_df <- tidy(world_split)
```

And, plotting the flight routes using this re-centered projection.


```r
ggplot() +
  geom_polygon(data = world_split_df, aes(long, lat, group = group), 
               fill = "grey80", color = "grey60", size = 0.1) +
  geom_point(data = cities, aes(lon, lat), color = "grey20", size = 0.5) +
  geom_path(data = gc_routes_df, 
            aes(long, lat, group = group), alpha = 0.5, color = "#fa6900") +
  geom_text(data = cities, aes(lon, lat, label = city),
            size = 3, color = "grey20", alpha = 0.9, nudge_y = 2, 
            check_overlap = TRUE) +
  coord_proj(proj, ylim = c(-60, 90)) +
  scale_x_continuous(breaks = seq(-180, 180, 30)) +
  scale_y_continuous(breaks = seq(-90, 90, 15)) +
  theme(panel.grid.major = element_line(size = 0.5, linetype = 2),
        axis.title = element_blank(),
        axis.text = element_blank(),
        axis.ticks = element_blank())
```

<img src="/figures//2016-02-01-long-flights_meridian-1.svg" title="plot of chunk meridian" alt="plot of chunk meridian" style="display: block; margin: auto;" />

## Bounding box and graticules

`ggplot` and `coord_proj` colour the whole background the same colour, include the corners which aren't actually part of the globe. To fix this I'll use the bounding box and graticules from [Natural Earth](http://www.naturalearthdata.com), instead of the default ones from ggplot.


```r
tf <- tempfile()
download.file(paste0(base_url, '50m/physical/ne_50m_graticules_all.zip'), tf)
unzip(tf, exdir = 'data/long-flights/', overwrite = TRUE)
unlink(tf)
bb <- shapefile('data/long-flights/ne_50m_wgs84_bounding_box.shp')
grat <- shapefile('data/long-flights/ne_50m_graticules_30.shp')
```

These will also need to be split at the edge and converted to data frames for ggplot.


```r
bb_df <- split(bb, central_meridian) %>% 
  tidy
grat_df <- split(grat, central_meridian) %>% 
  tidy
```

Including these in the plot.


```r
blank_theme <- theme(
  axis.line = element_blank(),
  axis.text.x = element_blank(), axis.text.y = element_blank(),
  axis.ticks = element_blank(),
  axis.title.x = element_blank(), axis.title.y = element_blank(),
  panel.background = element_blank(),
  panel.border = element_blank(),
  panel.grid.major = element_blank(),
  panel.grid.minor = element_blank(),
  plot.background = element_blank()
  )
ggplot() +
  geom_polygon(data = bb_df, aes(long, lat, group = group),
               fill = "light blue", color = NA) +
  geom_path(data = grat_df, aes(long, lat, group = group),
            color = "grey40", size = 0.1) +
  geom_polygon(data = world_split_df, aes(long, lat, group = group), 
               fill = "grey60", color = "grey80", size = 0.1) +
  geom_point(data = cities, aes(lon, lat), color = "grey20", size = 0.5) +
  geom_path(data = gc_routes_df, 
            aes(long, lat, group = group), alpha = 0.5, color = "#fa6900") +
  geom_text(data = cities, aes(lon, lat, label = city),
            size = 3, color = "grey20", alpha = 0.9, nudge_y = 2, 
            check_overlap = TRUE) +
  coord_proj(proj, ylim = c(-90, 90)) +
  blank_theme +
  theme(panel.border = element_rect(colour = "grey20", fill = NA),
        panel.background = element_rect(fill = 'grey20'))
```

<img src="/figures//2016-02-01-long-flights_grats-1.svg" title="plot of chunk grats" alt="plot of chunk grats" style="display: block; margin: auto;" />

## Colouring routes by distance

To increase the amount of information in the plot, I'll color the route according to their length. This requires joing the flight attribute data to the data frame of spatial data.


```r
routes_df <- mutate(flights_unique, id = as.character(rank)) %>% 
  left_join(gc_routes_df, ., by = "id")
```

Then applying a color gradient to the routes.


```r
ggplot() +
  geom_polygon(data = bb_df, aes(long, lat, group = group),
               fill = "light blue", color = NA) +
  geom_path(data = grat_df, aes(long, lat, group = group),
            color = "grey40", size = 0.1) +
  geom_polygon(data = world_split_df, aes(long, lat, group = group), 
               fill = "grey60", color = "grey80", size = 0.1) +
  geom_point(data = cities, aes(lon, lat), color = "grey20", size = 0.5) +
  geom_path(data = routes_df, 
            aes(long, lat, group = group, color = distance), size = 0.4, alpha = 0.8) +
  geom_text(data = cities, aes(lon, lat, label = city),
            size = 3, color = "grey20", alpha = 0.9, nudge_y = 2, 
            check_overlap = TRUE) +
  coord_proj(proj, ylim = c(-90, 90)) +
  scale_color_viridis(name = "Route Length (km)", labels = comma, option = "D") +
  guides(color = guide_colorbar(nbin = 256, title.position = "top", title.hjust = 0.5,
                                barwidth = unit(20, "lines"), barheight = unit(1, "lines"))) +
  blank_theme +
  theme(panel.border = element_rect(colour = "grey20", fill = NA),
        #panel.background = element_rect(fill = 'grey20'),
        plot.margin = unit(c(0,0.5,0,0.5), "lines"),
        legend.position = 'bottom',
        legend.margin = unit(0,"cm"))
```

<img src="/figures//2016-02-01-long-flights_dist-colors-1.svg" title="plot of chunk dist-colors" alt="plot of chunk dist-colors" style="display: block; margin: auto;" />
