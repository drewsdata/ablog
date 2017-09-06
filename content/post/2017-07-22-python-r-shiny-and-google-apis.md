---
title: Python, R, Shiny and Google APIs
author: Drew Coughlin
date: '2017-07-22'
tags:
  - API
  - Google
  - Python
  - Shiny
  - R
image: 'python_args_shiny_r.jpg'
slug: python-r-shiny-and-google-apis
---
A client wanted an easy way to quickly view upcoming event lists for any of their G Suite domain's Google Resource Calendars.  I tried keeping the solution entirely within R by using the [googleAuthR](https://cran.r-project.org/web/packages/googleAuthR/index.html) package but domain-wide delegated authentication didn't work properly for me so I reverted to Python.  Without too much anguish I wrote a working parameterized script that returns Google resource calendar event details based on resource calendar ID and  event count script arguments.  Google provides helpful calendar API documentation and a basic reference example [here] (https://developers.google.com/google-apps/calendar/v3/reference/events/list)

In [RStudio Shiny](https://shiny.rstudio.com/), I capture the Python script arguments from Shiny UI user inputs, call the Python script with those argument parameters, return and parse the results then display those results in a dynamic data table.  Here is the key Shiny portion:
```r
# Create Google Calendar Python API script args from user inputs
 gcalEmailP <- reactive({
		gcalList <- input$showCalDrop
		if (is.null(gcalList))
		  return()
 calEmailMatch <-
 gresourcecals$resourceEmail[gresourcecals$resourceName==input$showCalDrop]
 })
	
 gcalEventCountP <- reactive({
		gcalCt <- input$gcaleventcount
		if (is.null(gcalCt))
		  return()
 gcalCt <- input$gcaleventcount
 })

 # Call the Google Calendar API Python script with the params created above
 # only when the actionButton is clicked
	 gcalPythonScriptCall <- eventReactive(input$goCalButton,{    
		isolate({      
		input$goCalButton
		setwd("c:/Projects/Python")
		command <- "python.exe"
 # (Single and double quotes in the string needed if paths have spaces)
		path2script <- "c:/Projects/Python/googleCalAPIs.py"
		gaddress <- gcalEmailP()
		mtgcount <- gcalEventCountP()
		args <- paste0(" --gaddress ",gaddress, " --mtgcount ",mtgcount)
 # Add path to script as first arg
		allArgs <- c(path2script, args)
		output <- system2(command, args=allArgs, stdout=TRUE)
 # Parse the output into a dataframe for use in the data table
		output <- string.break.line(output)
		output <- as.data.frame.character(unlist(output))
		names(output) <- c("allresults")
		output <- separate(output, allresults,
		          into = c("DateTime", "Event", "Organizer", "Creator"), 
		          sep = ",", remove = TRUE, extra = "drop")
	})
 })
```