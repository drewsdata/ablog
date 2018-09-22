---
title: Get and Update Okta Application Credentials for Assigned User
author: Drew Coughlin
date: '2018-09-22'
tags:
  - API
  - httr
  - Okta
  - R
  - jsonlite
slug: get-and-update-okta-application-credentials-for-assigned-user
image: post_okta_update.png
header:
  caption: ''
  image: ''
highlight: yes
math: no
---
Much of my work lately involves interacting with the Okta API.  It is a well documented API for those familiar with RESTful API development in their language of choice.  If that doesn't describe you (yet) then hopefully this post will help those of you with some R programming knowledge.

Scenario:  You want to find out which credential a particular Okta user profile is using when accessing a specfic Okta application and change a credential value (in this example 'userName') if the current value doesn't match the desired value.  Obviously, you can log into the Okta administration interface then find, view and update the information there.  But what if you want to monitor the assigned credential value and change it if some event overwrites the existing value or you wish to make this update programmatically for some other reason?  This is exactly what the following R script will do for you.

```r
library(jsonlite)
library(httr)

# Okta API token.  You need to secure this - DO NOT store in your script. 
# There are various ways of doing it:
# https://cran.r-project.org/web/packages/httr/vignettes/secrets.html
oToken <- "<your Okta API token string>"

# Okta API endpoint URL to retrieve user details for a specific app 
appUserURL <- "https://<your okta domain>.okta.com/api/v1/apps/<the Okta app ID>/users/<the Okta user ID>"

# Make the API call 
initGet <- httr::GET(appUserURL,
                     config = (
                       add_headers(Authorization = oToken))
                     )

# Retrieve the retuned content 
initData <- fromJSON(httr::content(initGet, as = "text"), flatten = TRUE)

# Retrieve the current 'userName' credential and define the desired 'userName' credential
currentAppCredential <- initData$credentials$userName
correctAppCredential <- "<credential to assign>"

# Test the current 'userName' and assign the desired 'userName' if they differ
if (currentAppCredential == correctAppCredential) {
  print(paste0("The current credential is ",currentAppCredential,".  This is correct so no action taken."))
  
} else {
  print(paste0("The current credential is ",currentAppCredential,".  This is incorrect so let's update it with ", correctAppCredential))
  
  okta_post_body <- list(credentials = c(list(userName = correctAppCredential)))
  httr::POST(url = appUserURL,
             config = (
               add_headers(Authorization = oToken)),
             body = okta_post_body, encode = "json",
             verbose(data_in = TRUE, info = TRUE)
             )
  }
  ```
