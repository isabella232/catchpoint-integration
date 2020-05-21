## Overview

This document outlines steps required to integrate [SignalFx](https://www.signalfx.com) (a Splunk company) and [Catchpoint](https://www.catchpoint.com/) systems with regards to Synthetic Monitoring and Real User Metrics.

In both cases we are going to use the [simple-nodejs-weather-app](https://github.com/signalfx/simple-nodejs-weather-app) as a sample application monitored and tested by SignalFx and Catchpoint. Before you continue, make sure to deploy this application on a machine reachable from the Internet so that Catchpoint system can test it.

## Synthetic Monitoring

The goal is to have Synthetic Monitoring configured on Catchpoint's side to push synthetic tests related metrics to [SignalFx Infrastructure Monitoring](https://www.splunk.com/en_us/software/infrastructure-monitoring.html) platform.

There are three key components of this setup in Catchpoint system: 
  * *Product* - represents your website, web application or server monitored by Catchpoint.
  * *Test* - represents a set of instructions required to collect performance and availability metrics of your *Product*. 
  * *Test Data Webhook* - a mechanism used to push metric data to SignalFx.

The following procedure describes how to create all those components starting with *Test Data Webhook*.
  
### Setup

1. Login into the Catchpoint Portal and go to *Settings* &rarr; *API*.

1. In the *Test Data Webhook* section click *Add URL*.

   1. Specify *Name* (you will use it later to select this webhook).
   1. Enter SignalFx Data Ingest address in the *URL* field: `https://ingest.{{realm}}.signalfx.com/v2/datapoint`, for example `https://ingest.signalfx.com/v2/datapoint` (us0) or `https://ingest.us1.signalfx.com/v2/datapoint` (us1). 
   
      Note your Data Ingest server name may be different - you may check the actual value on your SignalFx profile page.
   1. Select *Template* for *Format*
   1. Add a new template
   1. Enter the *Template Name* e.g. SignalFx and set the Format to JSON.
   1. Paste the contents of [template.json](template.json) to the webhook template body.
   
      This template can be modified to use other metrics and/or details. The various possible macros can be found at https://support.catchpoint.com/hc/en-us/articles/360008476571
      
   1. Enter an email address into *On Failure Alert* and set the *Notification Trigger* value. If Catchpoint is not able to send data to the configured endpoint several times, you will get a notification to this address.
   
   1. Finally add the following headers:
   
      | Field | Value |
      | :--- | :--- |
      | `Content-Type` | `application/json` |
      | `X-SF-TOKEN` | `<YOUR SIGNALFX ACCESS TOKEN>` |
      
      Refer to the [SignalFx documentation](https://docs.signalfx.com/en/latest/admin-guide/tokens.html#working-with-access-tokens) to check how to obtain the Access Token.
   1. Hit *Save*.
   
1. In Catchpoint Portal go to the *Tests* section and click *Create* &rarr; *Product*.
   
   1. Specify your product *Name*.
   1. Enable *Test Data Webhook* and select the webhook created in step #2 earlier.
   1. Select one or more *Nodes* in the *Default Targeting and Scheduling* section.
   1. Hit *Save*.
   
1. Now click *Create* &rarr; *Transaction*.
   
   1. Specify test *Name*.
   1. Paste the following sample test commands in the *Script* field:
   
      ```javascript
      // change the following address to point to your instance of the tested app
      open("http://www.example.com/");
      var city = ${SequentialListByLocation('New York', 'London', 'Malaga')};
      type("//*[@name=\"city\" or @class=\"ghost-input\"]", "${var(city)}");
      setStepName("Type city name");
      
      clickAndWait("//*[@class=\"ghost-button\"]");
      setStepName("Get temperature.");
      ```
   1. Tick *Enable Test Data Webhook* checkbox.
   1. Hit *Save*.
   
1. Login to SignalFx and go to Dashboards section. Select the green plus icon in the upper right corner and click *Import* &rarr; *Dashboard Group*. Select [dashboard.json](dashboard.json) to import a sample dashboard showing several metrics related to Catchpoint's synthetic test results.

## Real User Metrics

The goal is to use the [sample weather app](https://github.com/signalfx/simple-nodejs-weather-app) instrumented with [signalfx-tracing library](https://github.com/signalfx/signalfx-nodejs-tracing) to push Real User Metrics to Catchpoint system including [SignalFx Microservices APM (µAPM)](https://www.splunk.com/en_us/software/microservices-apm.html) trace ID in the RUM data. 

Having SignalFx µAPM traceID in Catchpoint, we will be able to navigate to SignalFx µAPM application and analyze complete data flow both on the backend and frontend sides.

Refer to the [sample weather app](https://github.com/signalfx/simple-nodejs-weather-app) repository to find out how to instrument code and how to include µAPM trace ID in the RUM data. 

There are two key components of this setup in Catchpoint system: 
  * *Insight* - represents a property associated with your metrics you may later use to breakdown the RUM data.
  * *App* - represents your application you want to gather Real User Metrics for. 

Optionally you may also install a Google Chrome extension to make it easier to navigate from Catchpoint app to the SignalFx µAPM app.

### Setup

1. To create *Trace ID* *insight*, login into the Catchpoint Portal and go to *Settings* &rarr; *Insight* and enter the following values in the first empty row:

   | Name | Token | Status | HTTP Section | Format |
   | :--- | :---- | :----- | :----------- | :----- |
   | `Trace ID` | `traceid` | `Active` | `Headers`, other options disabled | `traceid`, JSON option disabled |
   
1. Hit *Save*.

1. Now go to *Settings* &rarr; *Apps* and click *Create* &rarr; *Site*.

   1. Enter a *Name* of your site.
   1. Enter a *Domain Name* that points to the sample weather app deployment.
   1. In the *Insight* section select *Trace ID* in the *Tracepoint* field.
   
      Note: the JavScript code shown in the *Tracking Code* section is the code that was copied to the sample weather app [view file](https://github.com/signalfx/simple-nodejs-weather-app/blob/master/views/index.ejs).
   1. Hit *Save*.

1. TBD: describe Google Chrome extension setup.

1. Finally in the Catchpoint app you can navigate to the *Analysis &rarr; Real User &rarr; Explorer &rarr; Page* section. This is where you can inspect RUM metrics from your app breaking them down by the SignalFx µAPM trace ID.
   1. In the top-left corner use the dropdown control to select your app.
   1. In the *Breakdown* &rarr; *1st by* select *Trace ID*.
   1. Adjust *Time Frame* if needed.
   1. Hit *Apply* button on the bottom of the left column.
      
      Once your app reports some metrics you will be able to see some charts here. Right above the chart is the SignalFx µAPM trace ID (e.g. `315a9f77d0101f4a`). 
      
      If you have the Google Chrome extension installed, you can click the trace ID to go to the SignalFx µAPM app to inspect the trace data.
      
      Otherwise you need to copy the trace ID and paste it in the SignalFx µAPM UI to inspect the trace data:
      *  go to *µAPM* &rarr; *Troubleshooting* &rarr; *View Trace ID* 
      * or in case of the µAPM PG (Previous Generaton): go to *µAPM PG* &rarr; *View Trace ID* &rarr; *Trace ID*;    
