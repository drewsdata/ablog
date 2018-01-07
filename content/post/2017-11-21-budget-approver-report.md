---
title: Budget approver report
author: Drew Coughlin
date: '2017-11-21'
slug: budget-approver-report
categories: []
tags:
  - R
  - RStudio
  - Shiny
  - SQL
---
Some projects evolve naturally by combining previous efforts into something new and useful.  This happened recently when I fullfiled a client request by extending an existing [RStudio Shiny application] (https://www.gratalis.com/post/shiny-file-upload-and-merge/) using what initially seemed like unrelated, separate components.

End users could upload a software vendor inventory text file in the existing Shiny application.  Key attributes in that file were joined across several database views and a Shiny data table report returned the aggregated data to the user. I created the database views using data from several sources including Okta, Active Directory and the client's corporate directory.  I had also built separate [hierarchy and reverse hierarchy reports and API's] (https://www.gratalis.com/post/reverse-corporate-hierarchy-api-with-sql-and-the-r-plumber-package/) that returned hierarchy information for individual employee records.

The new request required extending the Shiny data table report with a 'budget approver' value for each employee record based on the client's business rules and data in their corporate directory.  I fulfilled this requirement by extending and merging the hierarchy report components into the text file upload and merge report.  Now when the end user uploads the software vendor inventory text file the Shiny application will iterate through each employee record's key attribute using the reverse hiearchy logic and extend the data table to return the 'budget approver' value for each record.

This small R code section illustrates iterating over the employee records to generate the entire possible budget approver list via SQL.  The full solution includes a subsequent series of SQL case statements implementing the business logic determining the budget approver for each employee record based on the approver's hierarchical relationship with the employee.

```r
library(RODBC)
library(plyr)
library(dplyr)

dbCon <- RODBC::odbcConnect("dbDSN")

# Get a character vector of all non-terminated employee IDs from the database
input_Emp_ID <- sqlQuery(dbCon,as.is=TRUE, "select emp_ID from emp_directory
                where [status] !='T'")
input_Emp_ID <- as.vector(input_Emp_ID$emp_ID)

# Create list placeholder for data
emp_ID_List <- list()

# SQL CTE iteration generating reverse hierarchy for each emp_ID 
# in the input_Emp_ID vector - append results to the list placeholder
for (i in input_Emp_ID) {
  get_Each_Emp_ID <- 
                    sqlQuery(dbCon,as.is=TRUE, paste0("
                    WITH pathup (emp_ID, supervisor_ID)
                    AS (SELECT emp_ID, supervisor_ID
                    FROM dbo.emp_directory
                    WHERE emp_ID = ","'",i,"'","
                                                    
                    UNION ALL
                    SELECT cd.emp_ID, cd.supervisor_ID
                    FROM  dbo.emp_directory cd
                    INNER JOIN pathup ON
                    pathup.supervisor_ID = cd.emp_ID)
                    select emp_ID from pathup"))
  
  get_Each_Emp_ID$emp_ID <- as.character(get_Each_Emp_ID$emp_ID)
  emp_ID_List <- append(emp_ID_List,get_Each_Emp_ID)
  
  next
}

odbcClose(dbCon)

# Create data frame from the list values
budget_Approver_Emp_ID <- 
  rbind.fill(lapply(emp_ID_List,function(y){as.data.frame(t(y),stringsAsFactors=FALSE)}))

# Rename data frame columns to match reporting hierarchy depth level
empDFnames <- c("empBase","empS1","empS2","empS3","empS4","empS5")
names(budget_Approver_Emp_ID) <- empDFnames

# Upload data frame to database for further  processing
dbCon <- odbcConnect("dbDSN")
sqlDrop(dbCon, "budget_Approver_Emp_ID", errors = FALSE)
sqlSave(dbCon, `budget_Approver_Emp_ID`, tablename = "budget_Approver_Emp_ID", 
  append = FALSE, rownames = FALSE,
  safer=FALSE)
```

