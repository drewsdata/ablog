---
title: Application budget approver report
author: Drew Coughlin
date: '2017-10-18'
tags:
  - R
  - Shiny
  - SQL
  - RStudio
slug: application-budget-approver-report
---

Some projects evolve naturally through combining components into something new and useful.  This happened recently when I fullfiled client request by extending an existing [Shiny based report] (https://www.gratalis.com/post/shiny-file-upload-and-merge/) using what seemed like an unrelated, separate component.

The requirement was to create a 'budget approver' logic flow and data attribute that augmented an existing Shiny based web report I developed previously.  The existing Shiny application allowed the end user to upload a software vendor inventory file and join that file on key attributes from several database views then returned the aggregated data to the user in a Shiny data table. I built the views using data I ingested from several sources including Okta SSO, Active Directory and the client's corporate directory.  The final report needed to extend the Shiny datatable presented to the user whith a 'budget approver' value for each employee record in the corporate directory based on the client's business rule logic.
