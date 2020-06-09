CivicPlus
================
Amy DiPierro
2020-06-09

  - [Introduction](#introduction)
  - [Scraping documents using rvest](#scraping-documents-using-rvest)
  - [Scaling](#scaling)
  - [Next steps](#next-steps)

``` r
# Libraries
library(tidyverse)
library(rvest)
library(lubridate)
```

# Introduction

CivicPlus is a large software vendors serving counties, municipalities
and a variety of other public bodies in the United States. Public
agencies use CivicPlus as a general information website and, frequently,
as a vehicle to post meeting agendas, minutes and related attachments.

A list of 3,837 potential CivicPlus urls to scrape is available for
download
[here](https://docs.google.com/spreadsheets/d/1SL8qU_1YQPesNyKWacTdT8UhtAKm6pHE7iR-mtpgNZA/edit#gid=1759432592).

Although CivicPlus has APIs, it appears that they are only accessible
with credentials granted internally to website administrators. Public
access appears impossible. Further reading on CivicPlus APIs is
available at the following links:

  - [How to use CivicPlus
    APIs](https://www.civicengage.civicplus.help/hc/en-us/articles/360011464693-Authenticate-a-User)
  - [An example of how to navigate CivicPlus
    APIs](http://www.anaheim.net/api/help/index)

Since API access does not seem to be public, I instead demonstrate here
how to scrape meeting minutes and agendas using the package `rvest`.

# Scraping documents using rvest

`rvest` is relatively easy to use, especially if you are already
familiar with the Tidyverse. The hardest part is often navigating the
HTML itself, to find the parts you need to extract.

To learn more about `rvest`, I suggest consulting [this excellent
overview](https://rvest.tidyverse.org/index.html) and working through
the example code provided.

I start by providing three example URLs. You can test each by changing
which one is uncommented.

``` r
#url_data <- "http://va-newportnews.civicplus.com/AgendaCenter"
#url_root <- "http://va-newportnews.civicplus.com"

url_data <- "https://www.portofgalveston.com/AgendaCenter"
url_root <- "https://www.portofgalveston.com"

#url_data <-"http://ga-savannah.civicplus.com/AgendaCenter"
#url_root <- "http://ga-savannah.civicplus.com"
```

We need to be able to scrape only the documents that apply after a
certain date. The format of dates in CivicPlus is mmddyyyy. We can use
this to our advantage.

``` r
date_vector <-
  # Creates a vector of dates in the next week.
  seq(Sys.Date(), Sys.Date() + ddays(7), by = "1 day") %>%  
  # Rearranges the order of the date
  map_chr(
    ~ str_remove_all(
        str_c(
          str_extract(., "-[:digit:]{2}-"), # Extracts the month
          str_extract(., "-[:digit:]{2}$"), # Extracts the day
          str_extract(., "[:digit:]{4}") # Extractes the year
        ),
        "-" # Removes dashes
    )
  )
```

Here I scrape all of the links to agendas and minutes contained on the
Agenda Center website. Further details are provided below.

``` r
documents <- 
  # Start with the URL you plan to scrape
  url_data %>% 
  # Use this method from the rvest package to read in the URL.
  read_html() %>% 
  # Use these tags to isolate the part of the HTML we want to scrape
  # There are many ways to identify these links; this is just one
  html_nodes("#AgendaCenterContent .catAgendaRow a") %>% 
  # Pluck out all of the links to documents
  html_attr("href") %>% 
  # Turn it all into a tibble for ease of manipulation
  as_tibble() %>% 
  mutate(
    # Construct the urls so that we can download each document
    url = str_c(url_root, value),
    # Fill in blank links with NA
    value = na_if(value, ""),
    # Make a date column
    date = str_remove(str_extract(value, "_[:digit:]{8}"), "_")
  ) %>% 
  # Drop all of the NA values -- the empty links
  drop_na(value) %>% 
  unique() %>%
  # Use these filters to remove previous versions of agendas and minutes
  # and to filter out HTML versions of the same agendas and minutes
  filter(
    # Throw out some bogus values
    str_detect(value, "true", negate = TRUE),
    str_detect(value, "Previous", negate = TRUE),
    str_detect(value, "http://", negate = TRUE),
    str_detect(value, "www.", negate = TRUE),
    # Only keep documents dated in the future
    date %in% date_vector
  )
```

    ## Warning: Calling `as_tibble()` on a vector is discouraged, because the behavior is likely to change in the future. Use `tibble::enframe(name = NULL)` instead.
    ## This warning is displayed once per session.

``` r
documents
```

    ## # A tibble: 1 x 3
    ##   value                       url                                         date  
    ##   <chr>                       <chr>                                       <chr> 
    ## 1 /AgendaCenter/ViewFile/Age… https://www.portofgalveston.com/AgendaCent… 06112…

Note that this process will only identify the URLs of documents listed
on the first page of results. For our purposes, this should be fine,
since going forward we typically only want to capture new documents and
do not need a comprehensive archive of every document posted by a city.

As a final step, I download one document as an example proving that this
is possible:

``` r
download.file(documents$url[1], here::here("docs", "test.pdf"))
```

# Scaling

This approach appears to be relatively easy to scale. The script
`civicplus.R` loops through a list of candidate CivicPlus sites and
downloads all of the documents available. Since this process takes a
long time, and I don’t have space to download all of the documents
locally, `civicplus_short.R` is a demo version of the same script that
runs on the first five URLs in the CivicPlus URL list.

Here’s an .rds file of 2,220 .pdfs downloaded by `civicplus.R` on
`Sys.Date()`

``` r
civicplus <- 
  read_rds(here::here("data", "civicplus.rds")) %>% 
  mutate(
    state_province = str_to_upper(str_remove(str_remove(str_extract(url, "/\\w{2}-"), "/"), "-")),
    body = str_to_title(str_remove(str_remove(str_extract(url, "-\\w+\\."), "-"), "\\.")),
  )
```

Let’s see what these documents cover. Here are the states and provinces
for which we have documents:

``` r
civicplus %>% 
  count(state_province, sort = TRUE)
```

    ## # A tibble: 51 x 2
    ##    state_province     n
    ##    <chr>          <int>
    ##  1 MA               403
    ##  2 CA               161
    ##  3 TX               139
    ##  4 IL               103
    ##  5 WI               100
    ##  6 CT                86
    ##  7 MO                77
    ##  8 OH                75
    ##  9 CO                67
    ## 10 PA                61
    ## # … with 41 more rows

And here are the the bodies for which we have documents:

``` r
civicplus %>% 
  count(state_province, body, sort = TRUE) 
```

    ## # A tibble: 898 x 3
    ##    state_province body               n
    ##    <chr>          <chr>          <int>
    ##  1 MA             Brookline         18
    ##  2 MA             Brookline2        18
    ##  3 MA             Andover           17
    ##  4 MA             Falmouth          17
    ##  5 MA             Concord           16
    ##  6 MA             Hingham           15
    ##  7 MA             Nantucket         14
    ##  8 MA             Whitman           14
    ##  9 ID             Kootenaicounty    13
    ## 10 MA             Mansfield         13
    ## # … with 888 more rows

# Next steps

  - Identify root URLs that are invalid and remove them from our list of
    master URLs.
  - Make a scraper for another CivicPlus product, CivicWeb. [This is an
    example of a CivicWeb
    website.](https://nngov.civicweb.net/Portal/MeetingTypeList.aspx).
    It appears to me that CivicWeb is the latest version of meeting
    software from CivicPlus, and replaces the AgendaCenter websites
    scraped in this guide.
