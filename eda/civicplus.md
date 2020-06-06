CivicPlus
================
Amy DiPierro
2020-06-06

  - [Introduction](#introduction)
  - [Scraping documents using rvest](#scraping-documents-using-rvest)
  - [Next steps: Scaling](#next-steps-scaling)

``` r
# Libraries
library(tidyverse)
library(rvest)
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
url_data <- "http://va-newportnews.civicplus.com/AgendaCenter"
url_root <- "http://va-newportnews.civicplus.com"

#url_data <- "https://www.portofgalveston.com/AgendaCenter"
#url_root <- "https://www.portofgalveston.com"

#url_data <-"http://ga-savannah.civicplus.com/AgendaCenter"
#url_root <- "http://ga-savannah.civicplus.com"
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
    value = na_if(value, "")
  ) %>% 
  # Drop all of the NA values -- the empty links
  drop_na(value) %>% 
  # Use these filters to remove previous versions of agendas and minutes
  # and to filter out HTML versions of the same agendas and minutes
  filter(
    str_detect(value, "true", negate = TRUE),
    str_detect(value, "Previous", negate = TRUE)
  )
```

    ## Warning: Calling `as_tibble()` on a vector is discouraged, because the behavior is likely to change in the future. Use `tibble::enframe(name = NULL)` instead.
    ## This warning is displayed once per session.

``` r
documents
```

    ## # A tibble: 45 x 2
    ##    value                         url                                            
    ##    <chr>                         <chr>                                          
    ##  1 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ##  2 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ##  3 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ##  4 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ##  5 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ##  6 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ##  7 /AgendaCenter/ViewFile/Minut… http://va-newportnews.civicplus.com/AgendaCent…
    ##  8 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ##  9 /AgendaCenter/ViewFile/Minut… http://va-newportnews.civicplus.com/AgendaCent…
    ## 10 /AgendaCenter/ViewFile/Agend… http://va-newportnews.civicplus.com/AgendaCent…
    ## # … with 35 more rows

Note that this process will only identify the URLs of documents listed
on the first page of results. For our purposes, this should be fine,
since going forward we typically only want to capture new documents and
do not need a comprehensive archive of every document posted by a city.

As a final step, I download one document as an example proving that this
is possible:

``` r
download.file(documents$url[1], here::here("docs", "test.pdf"))
```

# Next steps: Scaling

I believe this approach will be relatively easy to scale. Here are the
next tasks that need to be done:

  - Verify each of the candidate CivicStar URLs. Although most are named
    via the convention state-city.civicplus.com or
    state-county.civicplus.com, there are many variations. It might
    require some manual work to correctly identify the name of the body
    that uses each URL. It might also be useful to check whether each of
    the candidates agencies uses CivicStar to post agendas – although,
    the worst that can happen is that the scraped returns no documents.

  - Loop through the list of each CivicPlus site in order to compile a
    list of all candidate documents for download.

  - Download each of the documents identified by the scraper and add
    them to our archive.

  - Adapt this workflow going forward so that we download new documents
    that do not already appear in our archive and ignore old documents
    that we’ve already downloaded.
