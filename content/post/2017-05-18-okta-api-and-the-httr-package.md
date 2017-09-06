---
title: "Okta API and the httr package"
author: "Drew Coughlin"
date: '2017-05-18'
image: 'okta_httr.jpg'
tags:
- Okta
- API
- httr
slug: okta-api-and-the-httr-package
---
Okta is a popular single sign-on (SSO) service provider that enables secure application connections.  It has a [well documented API](https://www.okta.com/products/developer/) I reference when creating client reports  based on data in their Okta instances. 

REST API endpoint services typically limit the number of records returned per call via pagination. Okta API [pagination](https://developer.okta.com/docs/api/getting_started/design_principles#pagination) is cursor based and pagination links are included in the [link headers](https://developer.okta.com/docs/api/getting_started/design_principles#link-header) of responses. This basic example uses the [httr](https://cran.r-project.org/web/packages/httr/index.html) and [jsonlite](https://cran.r-project.org/web/packages/jsonlite/index.html) packages to illustrate getting all records via a particular Okta API GET request by retrieving, parsing and following the link header values.  This works well enough to return all Okta application records in a given Okta instance but it isn't ideal becauase the underlying URL structure could conceivably change in the future if the Okta API is altered significantly.

```r
library(jsonlite)
library(httr)

# List placeholder for GET content
  initContent <- list()

# Initial URL contstruction for first URL call
  apiLimit <- as.character("200")
  initUrl <- 
  paste0("https://<okta-instance-domain>.okta.com/api/v1/apps/?limit=",apiLimit)

# The 'next' URL construction for paginated URLs
  nextURL <- "https://<okta-instance-domain>.com/api/v1/apps?after="
  limitURL <- "&limit=200"

# Pass initial URL to get first batch
  initGet <- httr::GET(initUrl,
                     config = (
                       add_headers(Authorization = "SSWS <Okta API key>")
                     )
                   )
# Unlist the 'all_headers' element from the URL
  initHeaders <- as.character(unlist(initGet$all_headers))

# Append the content object
  initContent <- append(initContent,content(initGet))


# Not needed but good to test and 
# find the URL, if it exists, of the "next" header element
# nextHead <- subset(initHeaders, grepl('"next\"',initHeaders))

# Iterate paginated results
  while (
    grep('"next\"',initHeaders)
  )
  
  { 
  # Extract the cursor and follow the pagination until the end
    parsenext <- 
    regmatches(nextHead, gregexpr('(?<=after=).*?(?=&limit)',
    nextHead, perl=T))[[1]]
  
  # Create URL
    oktaURLnext <- paste0(nextURL,parsenext,limitURL)
  
    initGet <- httr::GET(oktaURLnext,
                          config = (
                            add_headers(Authorization = "SSWS <Okta API key>")))
    initHeaders <- as.character(unlist(initGet$all_headers))
    initContent <- append(initContent,content(initGet))

    next
   } 

# Append the content list placeholder
  initContent <- append(initContent,content(initGet))

# Get a data frame
  initContent <- toJSON(initContent, simplifyDataFrame = TRUE, flatten = TRUE, 
                      recursive = TRUE)
  initContent <- fromJSON(initContent, simplifyDataFrame = TRUE, flatten = TRUE)
```
