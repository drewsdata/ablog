---
title: Improved - Okta API and httr package
author: Drew Coughlin
date: '2018-07-03'
slug: improved-okta-api-and-httr-package
categories: []
tags:
  - Okta
  - API
  - httr
header:
  caption: ''
  image: ''
highlight: yes
math: no
---
I finally had some time to revist and improve a [past project](https://www.gratalis.com/post/okta-api-and-the-httr-package/).  This example illustrates retrieving all Okta user profiles assigned to a given application in Okta.  As outlined in my earlier post, Okta limits the number of records returned depending on the API request so follow their cursor based pagination URLs to return all records if the number exceeds the API limit.  

There are many different reasons why you might want to retrieve this data.  You can extend this idea and retrieve even more user profile details by simply using the [list users API request](https://developer.okta.com/docs/api/resources/users#list-all-users) then joining on the 'id' field in both datasets.

```r
library(httr)
library(jsonlite)
library(dplyr)
library(stringr)
library(readr)


# Okta API token.  You need to secure this - DO NOT store in your script. 
# There are various ways of doing it:
# https://cran.r-project.org/web/packages/httr/vignettes/secrets.html
apiKey <- as.character("SSWS <API Token>")

# Initial URL construction for first URL call
appID <- as.character("<Okta application ID of interest>") 
apiLimit <- as.character("200")
initUrl <- paste0("https://<your Okta domain>.okta.com/api/v1/apps/",appID,"/users?limit=",apiLimit)

# Pass initial URL to get first batch
initGet <- httr::GET(initUrl,
                     config = (
                       add_headers(Authorization = apiKey)
                     )
)

# Get the content into a data frame
initContent <- fromJSON(httr::content(initGet, as = "text"), flatten = TRUE)

# Check headers for "next" cursor URL - means we have more
isNextHead <- initGet$headers[grepl('"next\"', initGet$headers)]

# Extract 'next' URL assuming it exists
getNextHead <- str_extract(isNextHead,"(?<=<).*(?=>)")

# If there is a 'next' (paginated cursor) URL, repeat til there isn't
if(length(isNextHead) > 0)
  repeat{
  
    # Pass next URL to get next batch
    initGet <- httr::GET(getNextHead,
                         config = (
                           add_headers(Authorization = apiKey)
                         )
    )
    
    # Add to content data frame
    initContent <- bind_rows(initContent, fromJSON(httr::content(initGet, as = "text"), flatten = TRUE))
    
    # Check headers for "next" cursor URL - means we have more
    isNextHead <- initGet$headers[grepl('"next\"', initGet$headers)]
    
    # Extract 'next' URL assuming it exists
    getNextHead <- str_extract(isNextHead,"(?<=<).*(?=>)")
    
    # Check for break condition
    if(length(isNextHead) == 0){
      break
    }
  }
```

