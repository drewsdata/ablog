---
title: Analytics solutions for everyone
author: ''
date: '2017-03-17'
tags:
  - API
  - Go
  - Google
  - R
image: 'analytics_everyone.jpg'
slug: analytics-solutions-for-all
---
[Google Data Studio](https://www.google.com/analytics/data-studio/) is a basic, yet functional report building and dashboard service for when you don't need or want the power of something like RStudio Shiny, Power BI or Tableau.  It has many data connectors and is tightly integrated into Google's authentication ecosystem.   I recently used it to build a simple yet effective business intelligence dashboard report for a client that saved them significant annual spend on a content analytics service.   

Underlying report data comes from a Google Sheet updated nightly with data from a Go program I wrote using Google's Admin SDK API for Google Drive activity [reporting](https://developers.google.com/admin-sdk/reports/v1/guides/manage-audit-drive).  I massage that data in R after extracting it with the Go program and before pushing it to Google Sheets using the extremely useful [googlesheets](https://cran.r-project.org/web/packages/googlesheets/) package.  The Google Data Studio report I designed and built does the rest and satisfies the client's basic needs for analytics on their content stored on Google Drive and served on their Google site.

There are several parts to this overall solution and it took a while to fully complete mostly because it was my first time writing a program in the Go language.  This is just the R script that tidies up the data and appends the Google Sheet.  

```r
library(dplyr)
library(stringr)
library(googlesheets)

setwd("c:/Projects/Go/")

# Read in the nightly file produced by the Go program that extracts 
# one day of Drive activity
  gDriveLogData <- read.csv("C:/Projects/Go/gDriveLogData.csv", header=FALSE, sep=","
  ,stringsAsFactors=FALSE)
  names(gDriveLogData) <- c("DateTime","Actor","Action","Document")

# Trim leading and trailing spaces from the Drive filenames
  gDriveLogData <- apply(gDriveLogData,2,str_trim)
  gDriveLogData <- as.data.frame(gDriveLogData, stringsAsFactors = FALSE)

# Convert the time stamp to a date / time class and change time zone from UTC to PST
  gDriveLogData$DateTime <- as.POSIXct(gDriveLogData$DateTime,
  "%Y-%m-%dT%H:%M:%S", tz="UTC")
  attr(gDriveLogData$DateTime, "tzone") <- "America/Los_Angeles"

# Get unique entries with  hours minutes seconds - deduplicate
  gDriveLogData <- unique(gDriveLogData)

# Trim hours minutes seconds
  gDriveLogData$DateTime <- strptime(gDriveLogData$DateTime, "%Y-%m-%d")
  gDriveLogData$DateTime <- as.POSIXct(gDriveLogData$DateTime)

# Filter out all other Google Drive events except views and downloads for the report
  target <- c("view", "preview", "download")
  gDriveLogData <- filter(gDriveLogData, Action %in% target)

# Write output for recordkeeping
  write.csv(gDriveLogData, "gDriveLogDataSiteUpdate.csv", row.names = FALSE)

# Rename the file from the nightly Go program output that we read in at the start
# in case the GO job fails - we won't create duplicate entries in the Google sheet
  file.rename("gDriveLogData.csv","last_gDriveLogData.csv")

# Google Sheet update (assumes the gs_auth steps already completed)
  gDriveLogDataAppend <- gs_key("<target Google Sheet GUID")
  gs_add_row(gDriveLogDataAppend, ws = 1, input = gDriveLogData, verbose = FALSE)
```

