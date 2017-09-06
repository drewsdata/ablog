---
title: Shiny file upload and merge
author: Drew Coughlin
date: '2017-06-05'
image: 'upload_merge.jpg'
tags:
  - Shiny
  - SQL
slug: shiny-file-upload-and-merge
---

I built a Shiny application recently for a client that accepts uploaded files and matches data attributes in those files to corporate directory information stored on the server in an RDS file.  After some trial and error I was able to piece together something useful and produce a self-service Shiny solution.  This example illustrates the key concept of properly referencing reactive data objects when accessing them outside the function that created them in the following "contents()" and "mergedData()" references.

```r
library(shiny)
library(shinydashboard)
library(DT)

setwd("c:/rprojects/homedash/")

# Read in the server data file
  serverData <- readRDS("serverData.rds")

# Force lower case for later merging / matching
  serverData <- as.data.frame(lapply(serverData, tolower))

# Shinydashboard app
  ui <- dashboardPage(
    dashboardHeader(title = "File Match Example"),
    dashboardSidebar(width=225,
      sidebarMenu(
        menuItem("Sidebar Title", tabName = "dashboard", icon = icon("globe")),
        hr(),
        menuItem("File upload merging - Click Me!", tabName = "sDataA", 
        icon = icon("table")),
        fileInput('fileUpload', 'Choose file to upload'),
        hr()
      )
    ),
    dashboardBody(
      tabItems(
        tabItem(tabName = "sDataA", icon = icon("dashboard"),
                h4("File upload matched with other data sources on email field."),
                fluidRow(DT::dataTableOutput('tblMerged'))
        )
      )
    )
  )

  server <- function(input, output) {
# Reactive function to upload the file we will merge / match with our server data
    contents <- reactive({
      inputFile <- input$fileUpload
      if (is.null(inputFile))
        return()
      read.csv(inputFile$datapath, header = TRUE)
    })

# Reactive function joining the input data with the server data
    mergedData <- reactive({
      if(is.null(contents()))
        return()
    
# Matching the email address field in our uploaded file with that in our server data 
    merge(x=contents(),y=serverData,by.x="email",
          by.y = "serverDataEmail",all.x=TRUE)
    })
  
# Reactive function creating the DT output object
    output$tblMerged <- DT::renderDataTable({
    if(is.null(mergedData()))
      return()
 
    DT::datatable(mergedData(), 
                  filter = 'top', 
                  extensions = 'Buttons',
                  options = list(
                    dom = 'Blftip',
                    buttons = 
                      list('colvis', list(
                        extend = 'collection',
                        buttons = list(list(extend='csv',
                                            filename = 'results'),
                                       list(extend='excel',
                                            filename = 'results'),
                                       list(extend='pdf',
                                            filename= 'results')),
                        text = 'Download'
                      )),
                    scrollX = TRUE,
                    pageLength = 5,
                    lengthMenu = list(c(5, 15, -1), list('5', '15', 'All')),
                    rownames = FALSE
                    )
                  )
    })

  }
  shinyApp(ui, server)
```