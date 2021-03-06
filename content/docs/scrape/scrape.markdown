---
title: "Web Scraping Archaeology Jobs from Indeed.co.uk"

draft: false
toc: true
toc_float: true
type: docs

linktitle: Scraping Job Posts
menu:
  docs:
    parent: Web Scraping
    weight: 2
---



## Background

Following my PhD, I spent quite a lot of time and effort applying for jobs in the UK. Each day, I'd scroll through numerous job forums that were organized in different fashions, trying to identify the best employment opportunities. While a lot of these sites had the specific information I wanted, they weren't particularly helpful when it came to visualizing where these jobs were and how they stacked up to one another in terms of salary. So I decided to explore web scraping with `rvest` as an option to automate the process of searching for new jobs on **Indeed.co.uk**, using `leaflet` as a GIS interface to plot the postings across the UK. 


```r
library(tidyverse)
library(xml2)
library(rvest)
library(stringr)
library(jsonlite)
library(leaflet)
library(htmltools)
library(kableExtra)
library(knitr)
library(widgetframe)
```

### Scraping and Dataframe Construction

The first stage in this process is to set the `url` to the indeed.co.uk link, and then assign that to a new variable called `webpage`. With that complete, we can use `dplyr` chaining to identify the nodes of interest on the webpage (I find the [SelectorGadget tool](https://chrome.google.com/webstore/detail/selectorgadget/mhjhnkcfbdhnjickkkdbjoemdmbfginb) incredibly useful for this) and extract these to a dataframe called `df`:


```r
url <- "https://www.indeed.co.uk/jobs?q=Archaeology"
webpage <- read_html(url)

df <- webpage %>% 
    html_nodes('.jobsearch-SerpJobCard') %>%   
    map_df(~list(Title = html_nodes(.x, '.jobtitle') %>% 
                     html_text() %>% 
                     {if(length(.) == 0) NA else .},    
                 Salary = html_nodes(.x, '.salary') %>% 
                     html_text() %>% 
                     {if(length(.) == 0) NA else .},
                 Employer = html_nodes(.x, '.company') %>% 
                     html_text() %>% 
                     {if(length(.) == 0) NA else .},
                 Location = html_nodes(.x, '.location') %>% 
                     html_text() %>% 
                     {if(length(.) == 0) NA else .},
                 Postdate = html_nodes(.x, '.date') %>% 
                     html_text() %>% 
                     {if(length(.) == 0) NA else .}))
```

Extract only the numbers from the `Salary` node:


```r
df <- df %>% mutate(salary2=parse_number(Salary))
```

### Geocoding

In order to map the jobs now compiled in our database, we have to be able to geolocate the opportunities to identify their latitude and longitude for mapping. Michael Hainke's [`nominatim`-based solution](http://www.hainke.ca) provides the base function for performing this task using `OpenStreetMap`: 


```r
nominatim_osm <- function(address = NULL)
{
  if(suppressWarnings(is.null(address)))
    return(data.frame("NA"))
  tryCatch(
    d <- jsonlite::fromJSON(flatten = TRUE, 
      gsub('\\@addr\\@', gsub('\\s+', '\\%20', address), 
           'http://nominatim.openstreetmap.org/search/@addr@?format=json&addressdetails=1&limit=1')
    ), error = function(c) return(data.frame("NA"))
  )
  if(length(d) == 0) return(data.frame("NA"))
  return(data.frame(lon = as.numeric(d$lon), 
                    lat = as.numeric(d$lat), 
                    District = if(is.null(d$address.state_district)){paste("Not Available")} else {as.character(d$address.state_district)}, 
                    State = if(is.null(d$address.state)){paste("Not Available")} else {as.character(d$address.state)}, 
                    Country = as.character(d$address.country)))
}
```

We can then use the function to look up the addresses of the job posts from our `df$Location` vector and create a new dataframe with the geolocation information:


```r
addresses <- df$Location
d <- suppressWarnings(lapply(addresses, function(address) {
  api_output <- nominatim_osm(address)
  return(data.frame(address = address, api_output))
  }) %>%
bind_rows() %>% data.frame())

test2<-cbind(d,df)
# Jitter points in case multiple jobs are in the same city
test2$lat <- jitter(test2$lat, factor = 1.5)
test2$lon <- jitter(test2$lon, factor = 1.5)
```

Here's what the dataframe head looks like:

```r
head(test2) %>%
  kable %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> address </th>
   <th style="text-align:right;"> lon </th>
   <th style="text-align:right;"> lat </th>
   <th style="text-align:left;"> District </th>
   <th style="text-align:left;"> State </th>
   <th style="text-align:left;"> Country </th>
   <th style="text-align:left;"> Title </th>
   <th style="text-align:left;"> Salary </th>
   <th style="text-align:left;"> Employer </th>
   <th style="text-align:left;"> Location </th>
   <th style="text-align:left;"> Postdate </th>
   <th style="text-align:right;"> salary2 </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Cambridge </td>
   <td style="text-align:right;"> 0.1673908 </td>
   <td style="text-align:right;"> 52.29787 </td>
   <td style="text-align:left;"> East of England </td>
   <td style="text-align:left;"> England </td>
   <td style="text-align:left;"> United Kingdom </td>
   <td style="text-align:left;"> Academic Services Librarian (Human and Social Sciences) (Fix... </td>
   <td style="text-align:left;"> £36,914 - £49,553 a year </td>
   <td style="text-align:left;"> University of Cambridge </td>
   <td style="text-align:left;"> Cambridge </td>
   <td style="text-align:left;"> 9 days ago </td>
   <td style="text-align:right;"> 36914 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Edinburgh </td>
   <td style="text-align:right;"> -3.1664502 </td>
   <td style="text-align:right;"> 56.01302 </td>
   <td style="text-align:left;"> Not Available </td>
   <td style="text-align:left;"> Scotland </td>
   <td style="text-align:left;"> United Kingdom </td>
   <td style="text-align:left;"> Treasure Trove Officer </td>
   <td style="text-align:left;"> £26,576 - £28,897 a year </td>
   <td style="text-align:left;"> National Museums Scotland </td>
   <td style="text-align:left;"> Edinburgh </td>
   <td style="text-align:left;"> 2 days ago </td>
   <td style="text-align:right;"> 26576 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Liverpool </td>
   <td style="text-align:right;"> -2.9942645 </td>
   <td style="text-align:right;"> 53.42331 </td>
   <td style="text-align:left;"> North West England </td>
   <td style="text-align:left;"> England </td>
   <td style="text-align:left;"> United Kingdom </td>
   <td style="text-align:left;"> Administrative Co-ordinator </td>
   <td style="text-align:left;"> £17,942 a year </td>
   <td style="text-align:left;"> Liverpool Museums </td>
   <td style="text-align:left;"> Liverpool </td>
   <td style="text-align:left;"> Just posted </td>
   <td style="text-align:right;"> 17942 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Edinburgh </td>
   <td style="text-align:right;"> -3.1637507 </td>
   <td style="text-align:right;"> 55.95144 </td>
   <td style="text-align:left;"> Not Available </td>
   <td style="text-align:left;"> Scotland </td>
   <td style="text-align:left;"> United Kingdom </td>
   <td style="text-align:left;"> Student Support Assistant </td>
   <td style="text-align:left;"> NA </td>
   <td style="text-align:left;"> University of Edinburgh </td>
   <td style="text-align:left;"> Edinburgh </td>
   <td style="text-align:left;"> 4 days ago </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Gloucester GL1 1HP </td>
   <td style="text-align:right;"> -2.2744676 </td>
   <td style="text-align:right;"> 51.85793 </td>
   <td style="text-align:left;"> South West England </td>
   <td style="text-align:left;"> England </td>
   <td style="text-align:left;"> United Kingdom </td>
   <td style="text-align:left;"> Engagement Officer </td>
   <td style="text-align:left;"> £23,369 a year </td>
   <td style="text-align:left;"> Gloucestershire County Council </td>
   <td style="text-align:left;"> Gloucester GL1 1HP </td>
   <td style="text-align:left;"> 10 days ago </td>
   <td style="text-align:right;"> 23369 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> London </td>
   <td style="text-align:right;"> -0.1458877 </td>
   <td style="text-align:right;"> 51.48963 </td>
   <td style="text-align:left;"> Greater London </td>
   <td style="text-align:left;"> England </td>
   <td style="text-align:left;"> United Kingdom </td>
   <td style="text-align:left;"> Graduate Research Analyst - Degree, Analyst, Writing </td>
   <td style="text-align:left;"> NA </td>
   <td style="text-align:left;"> Modis </td>
   <td style="text-align:left;"> London </td>
   <td style="text-align:left;"> 2 days ago </td>
   <td style="text-align:right;"> NA </td>
  </tr>
</tbody>
</table>

### Customizing the Layout

So we now have our jobs scraped with relevant information like salary, title, organization, etc. as well as the salient geolocation data for each opportunity. Before plotting in `leaflet`, however, I wanted a way of classifying the jobs based on the advertised salary. To do this, I used `mutate` to create a new variable assigning a color to different salary bands:


```r
test2 <- test2 %>%
  mutate(Color_code = case_when(
    is.na(test2$salary2) ~ "gray",
    test2$salary2 == 0.00 ~ "gray",
    test2$salary2 <= 20000 ~ "red",
    test2$salary2 >= 20000 & test2$salary2 <= 29999 ~ "orange",
    test2$salary2 >= 30000 & test2$salary2 <= 39999 ~ "blue",
    test2$salary2 >= 40000 ~ "green"
  ))
```


```r
test2 <- test2 %>% rename(Salary_low=salary2,Post_date=Postdate)
test2 <- test2 %>% select(Title,Location,Employer,Salary,lon,lat,Salary_low,Color_code,District,Country,State,Post_date)
```

Lastly, I like tooltips. A lot. So this code sets the labels for each job to include the respective `Employer`, `Title` and `Salary` information when we hover over the `leaflet` markers:


```r
labs <- lapply(seq(nrow(test2)), function(i) {
  paste0( '<p>', "<b>Employer:</b>",test2[i, "Employer"], '<p></p>', 
          "<b>Position: </b>",test2[i, "Title"], ' ', 
          test2[i, "location"],'</p><p>', 
          "<b>Salary: </b>",test2[i, "Salary"], '</p>' ) 
})
```

### Plotting the Output

The final step is to simply plug in our variables and final `test2` dataframe into `leaflet` to visualize the results:


```r
l2<-leaflet(test2) %>% 
  addTiles() %>%
  addAwesomeMarkers(~lon, ~lat, popup = ~Salary, 
                    icon = awesomeIcons(icon = 'ion-ionic', 
                                        library = 'ion', 
                                        markerColor = test2$Color_code), 
                    label = lapply(labs, htmltools::HTML),
                    labelOptions = labelOptions(textsize = "15px",
                      style=list(
                        'background'='rgba(243, 241, 239, 1)',
                        'border-color' = 'rgba(46, 49, 49, 1)',
                        'border-radius' = '2px',
                        'border-style' = 'solid',
                        'border-width' = '2px'))) %>% 
  addLegend("bottomright", 
            colors =c("#70AF28",  "#38ADDF", "	#F79530", "#CC3E24", "#575556"),
            labels= c("Over £40,000","£30,000 - £39,999","£20,000 - £29,999","Below £20,000","Rate Unavailable"),
            title= "Salary",
            opacity = 1)

htmlwidgets::saveWidget(frameableWidget(l2),'leafletscrape.html')
```

<iframe seamless src="../leafletscrape.html" width="100%" height="500"></iframe>

### Final Thoughts

This is a fairly simplistic setup, and could be built-out as needed. Scraping multiple pages is the next obvious step, and creating a more nuanced system for dealing with salary ranges would be helpful. But this workflow helped to achieve my goals at the time, and can hopefully be of some use to you!
