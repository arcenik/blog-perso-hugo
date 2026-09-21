+++
date = '2026-09-21T14:46:13+02:00'
draft = false
title = 'Victoria Logs Grafana Dashboard'
+++

Create the first dashboard to access Victoria Logs in Grafana

## Adding the datasource

First you need to add the [victoriametrics-logs-datasource](https://grafana.com/grafana/plugins/victoriametrics-logs-datasource/) plugin.
Then you can add the **datasource**.

![Victoria Logs datasource](/images/grafana-victorialogs-datasource.png)

Set the URL, then click on the **Save & Test** buttom at the bottom of the page.

![Victoria Logs datasource](/images/grafana-victorialogs-datasource-settings.png)

<!--more-->

## Create the first dashboard

Now, let's create a simple dashboard.

![](/images/gf-viclogs-dashboard.png)

The dashboard contains one variable and two pannels: a simple graph to display logs volume and the logs themself.

### Hostname variable

In the **dashboard settings**, hit the **+ Add Variable** button and select **Query**.

Set the following parameters
* Name: ```hostname```
* Refresh: ```On time range change```
* Multi-value: ✅
* Include All value: ✅
* Custom all value: ```*```

Open the variable editor and set the following:

* Target data source: ```victoriametrics-logs-datasource```
* Query type: ```field value```
* Field: ```_HOSTNAME```
* Sort: ```Alphabetical (asc)```

Click the **Run query** buttom on the top righ and check the returned values on top.

Finish by clicking on the **Apply** button on the bottom right.

### Logs volume pannel

Create the first pannel, change the type to **Time series** and set the settings

* Queries / A
  * code: ```_HOSTNAME:$hostname | stats by (_HOSTNAME) count() total```
  * type: ```Raw Logs```
  * legend: ```{{ _HOSTNAME }}```
* Pannel Options
  * Title: ```Log Count```
* Axis
  * Scale: ```Logarithmic```
  * Log base: ```10```
* Graph Style
  * Style: ```Points```
* Standard options
  * Unit: ```short```

### Logs pannel

The second pannel, that display logs is simmilar. The type is *Logs*

* Queries / A
  * code: ```_HOSTNAME:$hostname```
  * type: ```Raw Logs```
* Pannel Options
  * Title: ```Logs Lines```
* Logs
  * Show timestamps: ✅
  * Display log level: ✅
  * Wrap lines: ✅
  * Enable logs details: ✅
  * Enable fields selector: ✅

Expend the field tab on the top left: ```_HOSTNAME```, ```SYSLOG_IDENTIFIER```, ```Log line```

![Logs pannel](/images/gf-viclogs-pannel.png)

Finally, you can **Save**, **Go Back** and **Exit Edit**
