---
title: Removing Data Barriers
author: Drew Coughlin
date: '2019-02-24'
slug: removing-data-barriers
categories: []
tags:
  - httr
  - R
  - Okta
  - Log
  - gmailr
  - htmlTable
  - jsonlite
header:
  caption: ''
  image: ''
highlight: yes
math: no
---
Okta continuously improves administration features.  Recent additions include new email [notification options](https://support.okta.com/help/s/article/Improved-Admin-Email-Notification-Experience) for administrative functions.  These additions are helpful but customers often have more refined needs that require some development effort.  Fortunately, the [Okta system log API](https://support.okta.com/help/s/article/Improved-Admin-Email-Notification-Experience) provides access to nearly any event you might want to monitor or use to extend reporting processes that add value in your organization and enhance information transparency.  

While the Okta API provides integration capabilities for several security information and event management (SIEM) products, those solutions are generally focused on threat analysis and general monitoring.  There is often no substitute for building something yourself rather than relying on third-party, often expensive solutions that may require their own additional development or complex configuration.  Eliminate as many barriers as possible between your customer’s business information needs and the source data if you want to deliver real value.  

A solution I recently built for a customer uses the Okta system log API to fetch deprovisioning events during a given time range then joins the returned data with information from several different sources and delivers a formatted HTML report to the application owner’s email distribution group.  This solution has three components:

1. Monitor deprovisioning events for a particular application
2. Enrich returned data with information from different data sources
3. Send email to the distribution group address of the team that owns the application

Once data is retrieved via the Okta API, it’s just a matter of a shaping and filtering then joining against other data sources, structuring a nice report table and sending the email.  In my case, I used the [htmlTable package](https://cran.r-project.org/web/packages/htmlTable/vignettes/tables.html) for the formatted report table and the [gmailr package](https://github.com/jimhester/gmailr) for sending email but there are many options in how you create and deliver this information.  This flexibility is what makes developing your own solutions the most powerful way to help your customers by removing barriers to information they need to succeed.

```r
library(httr)
library(jsonlite)
library(tidyr)

  # Set date range for filtering log data, in this case the past 10 days
  since_date <- format((Sys.Date() - 10),"%Y-%m-%d")

  # Set Okta application of interest to retrieve target log event data
  okta_app_id <- as.character(<your Okta application value>)

  # Construct the API call to get removal events from the given Okta app
  okta_app_users_removed <- paste0("https://<yourdomain>.okta.com/api/v1/logs?since=",since_date,"T00:00:00.000Z&filter=event_type+eq+%22application.user_membership.remove%22+an   d+target.id+eq+",okta_app_id)

  # Send the request and get the content
  get_removed <- GET(okta_app_users_removed,
                      config = (
                        add_headers(Authorization = "<key>")))

  app_content <- content(get_removed)

  # Transform the serialized string into workable JSON and nested dataframe
  app_content_to_json <- toJSON(app_content, simplifyDataFrame = TRUE, flatten = TRUE, recursive = TRUE)
  json_from_app_content <- fromJSON(app_content_to_json, simplifyDataFrame = TRUE, flatten = TRUE)
  
  # Dataframe of lists filtered for just the row we need
  app_uname_rem <- unnest(json_from_app_content, target) %>% filter(type == 'AppUser') %>% unique()
  app_uname_rem <- as.data.frame(app_uname_rem)
  app_uname_rem$uKey <- as.character(seq.int(nrow(app_uname_rem)))
  
  # More filtering to get unique instances for the user events
  uname_rem <- unnest(json_from_app_content, target) %>% filter(type == 'User') %>% unique()
  uname_rem <- as.data.frame(uname_rem)
  uname_rem$uKey <- as.character(seq.int(nrow(uname_rem)))
  
  # Join the filtered records to get a tidy data set of just the app removal events per user
  au_rem_joined_data <- dplyr::left_join(app_uname_rem,uname_rem, by = "uKey")
```