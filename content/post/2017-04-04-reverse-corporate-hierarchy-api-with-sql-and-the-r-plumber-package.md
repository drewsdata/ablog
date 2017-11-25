---
title: Reverse corporate hierarchy API with SQL and the R plumber package
author: Drew Coughlin
date: '2017-04-04'
tags:
  - API
  - R
  - SQL
image: hierarchy_api_plumber.jpg
slug: reverse-corporate-hierarchy-api-with-sql-and-the-r-plumber-package
---

Using recursive SQL common table expressions (CTE) to build corporate hierachy reports is [well documented] (https://technet.microsoft.com/en-us/library/ms186243(v=sql.105).aspx) use case for SQL CTE's. I leveraged a SQL CTE to build a Shiny data table report that returns a client's top down hierarchy based on user selection of an employee ID. The Shiny app uses the [squr](https://github.com/smbache/squr) and [RODBC](https://cran.r-project.org/web/packages/RODBC/index.html) pacakges to call the SQL script. 

That report was useful but you never really know what someone wants until they ask.  After announcing its availability, I received a request from another developer asking if I could return the 'reverse hiearchy' path as a REST API endpoint given an employee ID as input.  They wanted me to return a particular JSON string format starting from the submitted employee ID up to their direct manager's employee ID and so on all the way up to the top of the organization.  I came across a SQL CTE example that sent me in the right direction with the underlying SQL query.  Then I remembered reading about the [plumber](https://cran.r-project.org/web/packages/plumber/index.html) package and from there it was just a matter of assembling the parts.  This was internal reporting of non-sensitive data so API restriction and security requirements were minimal.  Note, this example is based on a plumber package version prior to the 0.4x update which introduced major changes.

The following two scripts illustrate getting the data then starting the endpoint that listens for requests and serves the data at http://localhost:9900/eidup?eidget=00012 where '00012' is the employee ID from which you would like the path upwards.  Data returned will be in a format similar to:
```r
["[{\"empID\":\"00012\",\"path\":\"00012/11113/22214/33315/44416\"}]"]

```

R and SQL CTE script called by the plumber::plumb function:
```r
#* @get /eidup

getFunc <- function(eidget){
  
  library(RODBC)
  library(jsonlite)
  library(plumber)
  
  setwd("c:/Projects/orgReverse/")
  
  dbCon <- RODBC::odbcConnect("dbDSN")
  inputEmp <- as.character(eidget)
  dbData <- sqlQuery(dbCon, paste0("
                                   WITH pathup (empID, supervisorID)
                                   AS (
                                   SELECT empID, supervisorID
                                   FROM dbo.cdDBdata
                                   WHERE empID = ","'",inputEmp,"'","
                                   UNION ALL
                                   SELECT cd.empID, cd.supervisorID
                                   FROM dbo.cdDBdata cd
                                   INNER JOIN pathup ON
                                   pathup.supervisorID = cd.empID
                                   )
                                   SELECT empID from pathup"))
  odbcClose(dbCon)
  
  dbData$empID <- as.character(dbData$empID)
  empData <- as.data.frame(dbData$empID[1], stringsAsFactors = FALSE)
  names(empData) <- "empID"
  
  dbDataRet <- paste(dbData$empID, collapse='/')
  dbDataRet <- as.data.frame(dbDataRet, stringsAsFactors = FALSE)
  
  names(dbDataRet) <- "path"
  dbDataRet <- as.data.frame(dbDataRet, stringsAsFactors = FALSE)
  
  myDFAll <- data.frame(empData$empID,dbDataRet$path, stringsAsFactors = FALSE)
  names(myDFAll) <- c("empID","path")
  
  pathUpJSON <- toJSON(myDFAll)
}
```

R script that calls the previous script and serves data at the API endpoint:
```r
library(plumber)
setwd("c:/Projects/orgReverse/")
cdpathupemps <- plumb("c:/Projects/orgReverse/empIDUp_api_call.r") 
cdpathupemps$run(port=9900)
```
