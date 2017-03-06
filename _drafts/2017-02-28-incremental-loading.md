---
layout: post
title:  "Incremental data loading with Talend"
date:   2017-03-08 18:00:00
categories: ETL
excerpt_separator: <!--more-->
---

One of the unavoidable challenge in any data integration architecture is choosing the right loading technique. Push change data capture (CDC) is more than often not available. Full (re)load, batch delta/daily load and incremental load (CDC pull) are potential candidates. In this post, I will describe a possible implementation of an incremental load using Talend. Only the relevant part for CDC will be described (no processing of the data).

<!--more-->

Incremental load is an integration technique in which only the created/modified records since the last integration execution will be loaded. This can be based on an auto-incremental key or a modification timestamp attribute. This technique is of particular interest for data synchronization between operational systems. The usage of this technique is more rare in data warehouse integration for which there are usually dependencies between multiple sources or entities within the same source.

The idea is simple: the integration flow needs to persists the last "modification timestamp" it processeed, or it needs to be able to retrieve it back if the origin source system timestamp is transfered to the target. 

I have implemented the first solution. The last processeed "timestamp" will be stored in the following table `etl_jobs_delta_loading`. 

{% highlight sql %}
CREATE TABLE monitoring.etl_jobs_delta_loading
(
    id INTEGER DEFAULT nextval('monitoring.etl_jobs_delta_loading_id_seq'::regclass) NOT NULL,
    job_name VARCHAR(500) NOT NULL,
    loaded_until TIMESTAMP,
    etl_execution_time TIMESTAMP,
    execution_status VARCHAR(30)
);
{% endhighlight %}

Along with this piece of information, we store:
- **the name** of the Talend job, 
- **the execution status** of the job (success or failure): if the job ends in failure, then the next execution needs to resume on the previously last timestamp (to re-run the failed integration). 
- **the timestamp of the execution**

The first step of the flow is to retrieve the last properly integrated record timestamp. To do so we query our table:
{% highlight sql %}
SELECT
  job_name,
  max(loaded_until) as loaded_until
from monitoring.ETL_jobs_delta_loading
where job_name = \'"+jobName+"\'
and execution_status = 'success' 
group by job_name ;
{% endhighlight %}

We need to perists this information in the flow using the GlobalMap and the TSetGlobalVar component. This is a simple in memory key-value store. All keys and values need to be persisted as String. We also store the execution time of the job.

{% highlight java %}
delta_date = TalendDate.formatDate("yyyy-MM-dd HH:mm:ss",to_gmap.loaded_until) ;
execution_date = TalendDate.formatDate("yyyy-MM-dd HH:mm:ss",TalendDate.getCurrentDate()) ;
{% endhighlight %}

In the first execution of the job, the query will return no result. We need to take care of that:

{% highlight java %}
if (globalMap.get("delta_date") == null) {
  System.out.println("First execution of "+jobName)  ;
	globalMap.put("delta_date", "1990-01-01 00:00:01") ;
	globalMap.put("execution_date", TalendDate.formatDate("yyyy-MM-dd HH:mm:ss",TalendDate.getCurrentDate()) ) ;
}
{% endhighlight %}

If this is the first run, then we set the initial delta date to "1990-01-01 00:00:01" (depends on requirements). We then use the `delta_date` to filter the origin source system. I have simulated a simple source with a tRowGenerator, and its filtering with a tFilterRow. If your source is a RDMS/NoSQL then you need to adapt the query and apply the filtering at the query level.

The next step is to separate the flow into two branches. One of them will be used only to keep track of the "modification timestamp" of the record being processed (_to_agg_ branch). The other one will be used for the actual _useful_ processing.

We use the _to_agg_ to find the last -- in time, not in processing ordering -- value of the "timestamp", basically the maximum value. We use a tAggregateRow with the max function on that field. You may think that this will be memory-intensive: aggregating all value in memory to find the maximum. But Talend implement it the smart way:

{% highlight java %}
if (to_agg.updated_at != null) { // G_OutMain_AggR_546
	if (
	operation_result_tAggregateRow_1.updated_at_max == null
			|| to_agg.updated_at
					.compareTo(operation_result_tAggregateRow_1.updated_at_max) > 0
	) {
		operation_result_tAggregateRow_1.updated_at_max = to_agg.updated_at;
	}
}
{% endhighlight %}

The component keeps track of the maximum value only, not of all the values (basically the right implementation to be expected).

We store this value in the GlobalMap as:

{% highlight java %}
"delta_date_new" = TalendDate.formatDate("yyyy-MM-dd HH:mm:ss",max_update.updated_at)
{% endhighlight %}

If this job runs we need modified records in the source system with value will be empty (null), we take that into consideration with the last tJava component:

{% highlight java %}
if (globalMap.get("delta_date_new") == null) {
	globalMap.put("delta_date_new", globalMap.get("delta_date") ) ;
}
{% endhighlight %}

Finally, we need to persist this value in our specific table by issuing a SQL command against that table.

{% highlight sql %}
insert into monitoring.ETL_jobs_delta_loading(job_name, loaded_until, etl_execution_time, execution_status) 
VALUES 
(\'"+jobName+"\',
\'"+(String) globalMap.get("delta_date_new")+"\',
\'"+(String) globalMap.get("execution_date")+"\',
\'success\') ;
{% endhighlight %}

or in case of failure:
{% highlight sql %}
insert into monitoring.ETL_jobs_delta_loading(job_name, loaded_until, etl_execution_time, execution_status) 
VALUES 
(\'"+jobName+"\',
\'"+(String) globalMap.get("delta_date_new")+"\',
\'"+(String) globalMap.get("execution_date")+"\',
\'failure\') ;
{% endhighlight %}

We have now finished our meta job which generated about 7600 LOC and still no actuall processing was done !
