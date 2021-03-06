---
title: "GIS with Leaflet"
draft: no
toc: yes
toc_float: yes
type: docs
linktitle: Leaflet for Census Data
menu:
  docs:
    parent: GIS
    weight: 1
---



While software platforms like ArcGIS and QGIS are the industry standard for complex geopspatial analysis, they're a bit cumbersome for basic visualization purposes. [Leaflet](https://leafletjs.com/) is an open-source JavaScript library that enables rapid production of interactive maps that are easily embedded in a variety of contexts. Leaflet maps are dynamic and include features like:

* Interactive panning/zooming
* Choropleth capabilities
* Pop-up tooltips and labels
* Highlighting/selecting regions
* Custom map icons

## Usage

A basic usage example employing `leaflet` and the [`acs`](https://github.com/cran/acs) package to visualize U.S. Census data is demonstrated below. 

In this particular tutorial, I'll be using the [2011 Nativity And Citizenship Status In The United States](https://www.census.gov/programs-surveys/acs/technical-documentation/table-and-geography-changes/2011/1-year.html) dataframe, which provides information on the citizenship status of U.S. residents. The aim here is to produce GIS output that maps the tract density of non-U.S. citizens residing in the state of Rhode Island as of 2011.

### Fetching `acs` Data


```r
library(tidyverse)
library(tigris)
library(acs)
library(leaflet)
library(htmltools)
library(widgetframe)
```

Our first step is to initialize a `tracts` vector that defines the state of interest for `acs` calls. We can then make a basic `acs.fetch` call using the relevant table number for our dataset of interest (in this case, B05001):


```r
# Select Rhode Island for tracts
tracts <- tracts(state = 'RI', cb=TRUE)
# Fetch ACS data 
ri <- acs.fetch(
  geography = geo.make(state = "RI", county="*", tract = "*"),
  endyear = 2011, span = 5, 
  table.number = "B05001", 
  col.names = "pretty")    
```

Then we can see what the column names are in the dataframe:


```r
# View column names
attr(ri, "acs.colnames") 
```

```
## [1] "Nativity and Citizenship Status in the United States: Total:"                                                
## [2] "Nativity and Citizenship Status in the United States: U.S. citizen, born in the United States"               
## [3] "Nativity and Citizenship Status in the United States: U.S. citizen, born in Puerto Rico or U.S. Island Areas"
## [4] "Nativity and Citizenship Status in the United States: U.S. citizen, born abroad of American parent(s)"       
## [5] "Nativity and Citizenship Status in the United States: U.S. citizen by naturalization"                        
## [6] "Nativity and Citizenship Status in the United States: Not a U.S. citizen"
```

### Dataframe Construction

With our initial data fetched, we can create a new structure to hold geolocation variables and our main data of: 

1) the overall population in the state of RI, and 
2) the total number of non-citizens in the state


```r
# Create new dataframe including geolocation data and estimates
ri_df <- data.frame(
  paste0(
    str_pad(ri@geography$state,  2, "left", pad="0"),
    str_pad(ri@geography$county, 3, "left", pad="0"),
    str_pad(ri@geography$tract,  6, "left", pad="0")),
  ri@estimate[,c(
    "Nativity and Citizenship Status in the United States: Total:", 
    "Nativity and Citizenship Status in the United States: Not a U.S. citizen")],
  stringsAsFactors = FALSE)
# Select subset of initial dataframe
ri_df <- dplyr::select(ri_df, 1:3) %>% tbl_df()
# Rename rows
names(ri_df) <- c("GEOID", "total", "non_citizen")
# Calculate percentage of non-U.S. citizens
ri_df$percent <- 100*(ri_df$non_citizen/ri_df$total)
```

### Perform spatial join

We can now spatially join our initial `tracts` vector with our new dataframe, using the `GEOID` parameter:


```r
# Spatial join
df_merged <- geo_join(tracts, ri_df, "GEOID", "GEOID")
# Remove any tracts with no land area
df_merged <- df_merged[df_merged$ALAND>0,]
```

### Plot Output

Finally, we set our popup labels and color palette for the final `leaflet` output, to allow clicking on a tract to get information on the percentage on non-citizens:


```r
# Set popup labels
popup <- paste0("GEOID: ", df_merged$GEOID, "<br>", "Percent of non-citizens: ", round(df_merged$percent,2))
# Set color palette
pal <- colorNumeric(
  palette = "plasma",
  domain = df_merged$percent
)
# Plot
RI<-leaflet(df_merged) %>% 
  addTiles() %>%
  addPolygons(fillColor = ~pal(percent),
              color = "#b2aeae",
              fillOpacity = 0.7,
              weight = 1,
              smoothFactor = 0.2,
              popup = popup) %>% 
  addLegend(pal = pal,
            values = df_merged$percent,
            position = "bottomright",
            title = "Percent of<br>non-citizens",
            labFormat = labelFormat(suffix = "%"))

# Workaround for Leaflet bug with NA in legend
css_fix <- "div.info.legend.leaflet-control br {clear: both;}" 
html_fix <- htmltools::tags$style(type = "text/css", css_fix)  
RI<-RI %>% htmlwidgets::prependContent(html_fix)
htmlwidgets::saveWidget(frameableWidget(RI),'RI.html')
```

<iframe seamless src="../RI.html" width="100%" height="500"></iframe>

### Final Thoughts

This brief overview highlights some of `leaflet`'s functionality to map continuous data in an interactive fashion, as the popups allow us to quickly see the exact percentage of non-citizens with a single click. Moreover, this process shows how spatial joining can be performed between government datasets and the GIS interface for quick data visualization. There are plenty of other ways to manipulate data for exciting spatial output in `leaflet`, and for further ideas on map types I'd recommend looking at the [Leaflet for R page](https://rstudio.github.io/leaflet/).
