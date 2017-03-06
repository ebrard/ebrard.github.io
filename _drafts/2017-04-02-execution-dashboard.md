---
layout: post
title:  "Creating an execution dashboard for Talend jobs"
date:   2017-04-02 18:00:00
categories: ETL
excerpt_separator: <!--more-->
---

Talend Open Studio is a great open source solution. Its "Cost vs. functionalities" is quite good. 
But because this is the free version, as opposed to the enterprise edition, it lacks a few core functionalities such as 
a proper [workflow manager](http://dataeng.ninja/etl/workflow/2016/11/15/workflow-manager/) or a simple execution dashboard.
But as it turns out, creating an execution dashboard is actually quite easy.

<!--more-->

There are two functionnalities that EVERY projects based on the "Open Studio" version should have turned on: 

- "Use statistics" (tStatCatcher)
- "Use volumetrics" (tFlowMeterCatcher)

The first option will tell all jobs of the project to send execution statistics 
through tStatCatcher to the target datastore. This way we can collect some metrics such as:
- the job start timestamp
- the job end timestamp and duration
- the job execution status ('failure', 'success')

If we tick the option "Catch component statistics", we will also collect component execution statistics.

The second option will route the outputs of all tFlowMeter components through the tFlowMeterCatcher to the target datastore. The tFlowMeter
components can be used to catch "inner-join"/"look-up" failure metrics of the flow. This can be usefull to detect dimension look-up
errors for example, or data quality issues.

To be able to report using these, we need the datastore to be simple RDMS tables. Just by selecting  the "On Databases" option 
(and configuring it), Talend will automatically create the defined two tables as:

{% highlight sql %}
CREATE TABLE ETL_jobs_execution_monitoring
(
    moment DATETIME NOT NULL,
    pid VARCHAR(20) NOT NULL,
    father_pid VARCHAR(20) NOT NULL,
    root_pid VARCHAR(20) NOT NULL,
    system_pid BIGINT NOT NULL,
    project VARCHAR(50) NOT NULL,
    job VARCHAR(255) NOT NULL,
    job_repository_id VARCHAR(255) NOT NULL,
    job_version VARCHAR(255) NOT NULL,
    context VARCHAR(50) NOT NULL,
    origin VARCHAR(255),
    message_type VARCHAR(255),
    message VARCHAR(255),
    duration BIGINT
);
{% endhighlight %}

and,
{% highlight sql %}
CREATE TABLE ETL_jobs_volume_monitoring
(
    moment DATETIME NOT NULL,
    pid VARCHAR(20) NOT NULL,
    father_pid VARCHAR(20) NOT NULL,
    root_pid VARCHAR(20) NOT NULL,
    system_pid BIGINT NOT NULL,
    project VARCHAR(50) NOT NULL,
    job VARCHAR(255) NOT NULL,
    job_repository_id VARCHAR(255) NOT NULL,
    job_version VARCHAR(255) NOT NULL,
    context VARCHAR(50),
    origin VARCHAR(255),
    label VARCHAR(255),
    count INT,
    reference INT,
    thresholds VARCHAR(255)
);
{% endhighlight %}

Regarding the execution statistics, there is actually nothing else to do. Talend will feed the data to the execution table automatically. Collecting volume information
requires establishing development rules and conventions. For example, any dimension look-up implementations should send failed join records
to a tFlowMeter as in this integration job part:

![Example of a job flow to tFlowMeter](/images/execution-dash/volume_example.png)

I have established the following development conventions:
- All source component (files, APIs, RDMS) flows should first go through a tFlowMeter with a custom label starting with the keyword
_in_ and a short description of the source.

![Example of a job flow volume through tFlowMeter](/images/execution-dash/volume_src_example.png)

- Any look-up (`tMap` or `tJoin`) between datasets or dimension look-up should have a failed records flow going to or through a tFlowMeter
with a custom label starting with the keyword _error_ and following by a very short description.

The first development convention helps detecting issues in source systems communication or internal, 
if on a particular execution, the label value `count` drops to zero, there is definitely something wrong.

The second convention helps dealing with data quality issues, early facts (or late dimensions), integration errors etc.

We can now create a dashboard on these metrics with any reporting tool (example for Tableau). We want to blend both datasets into
the same dashboard. A "simple" way to do so with Tableau is to create a datasource with ETL_jobs_execution_monitoring in a left
join with ETL_jobs_volume_monitoring. To avoid duplications, we need to filter the ETL_jobs_execution_monitoring with:
- `origin` is null (We want the job, not the components)
- `message_type = 'end'` 

Metrics (`duration` and `count`) will be aggregated by the reporting tool, a smart decision here is to select the average
function, just to be sure.

In any case, to prevent the reporting solution from aggregating metrics, we need to add the the `pid` (Tableau internal) or the 
`system_pid` to the dimensions we use. Both are unique (at least within a day).

With all these in mind, it's quite straightforward to build an execution dashboard like this one:

![Example of an execution dashboard with Tableau](/images/execution-dash/dash_execution.png)
