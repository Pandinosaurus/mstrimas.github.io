---
layout: post
title: "Under eBirded counties: filling the gaps in eBird data"
published: true
excerpt: >
  Highlighting the least eBirded counties in the US by state.
category: r
tags: r spatial gis ebird
leaflet: true
---

After seeing my [previous post](/r/ebird-count/), one of my colleagues suggested it would be interesting to see a map of the least eBirded counties in the US. Highlighting these counties might encourage more eBirders to visit them, thereby filling in the geographic gaps in eBird data. So, here it is, the counties within each state having the fewest checklists. I'll also throw on the county with the most checklists for comparison. There are much more sophisticated ways of looking at gaps in eBird data, but this is a good start.


```r
library(dplyr)
library(sf)
library(leaflet)
library(leaflet.extras)
library(here)

# load the data generated from the previous post
ebird_counties <- here("_source", "data", "ebird-county", "ebird-county_sf.rds") %>% 
  readRDS() %>% 
  # filter to counties in lower 48
  filter(str_detect(region_code, "^US"), state != "AK")

# fewest checklists by state
ebird_counties_least <- ebird_counties %>% 
  group_by(state) %>% 
  top_n(-1, n_checklists) %>% 
  ungroup() %>% 
  mutate(label = "Fewest checklists")
# most checklists by state
ebird_counties_most <- ebird_counties %>% 
  group_by(state) %>% 
  top_n(1, n_checklists) %>% 
  ungroup() %>% 
  mutate(label = "Most checklists")
# combine
ebird_counties <- rbind(ebird_counties_least, ebird_counties_most)

# add popups
popup_maker <- function(code, name, n_species, n_checklists) {
  link_style <- "target='_blank' style='text-decoration: none;'"
  region_url <- paste0("<strong><a href='http://ebird.org/ebird/subnational2/",
                       code, "?yr=all' ", link_style, ">", name, "</a></strong>")
  counts <- paste0("<strong>", n_checklists, " checklists</strong> | ", 
                   n_species, " species")
  paste(region_url, counts, sep = "<br/>")
}
ebird_counties <- ebird_counties %>% 
  mutate(region_name = paste(region_name, state, sep = ", "),
         popup = popup_maker(region_code, region_name, 
                             n_species, n_checklists)) %>% 
  select(label, popup)

# leaflet map
p <- colorFactor(c("#e41a1c", "#4daf4a"), 
                 domain = c("Fewest eBird checklists", "Most eBird checklists"))
leaflet(ebird_counties) %>%
  addProviderTiles("OpenMapSurfer.Roads") %>% 
  addFullscreenControl() %>% 
  addPolygons(color = ~ p(label), opacity = 1.0,
              weight = 2, smoothFactor = 0.5,
              fillColor = "#555555", fillOpacity = 0.2,
              popup = ~ popup) %>% 
  addLegend("bottomright", pal = p, values = ~ label,
            title = "Counties with fewest/most eBird checklists",
            opacity = 1)
```



<iframe src="/assets/leaflet/under-ebirded.html" style="border: none; width: 800px; height: 600px"></iframe>
<a href="/assets/leaflet/under-ebirded.html" target="_blank"><strong>Fullscreen</strong></a> | Data source: [eBird](http://ebird.org/)
