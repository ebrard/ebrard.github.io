---
layout: post
title:  "Workflow manager"
date:   2016-11-15 19:00:00
meta_description: "This post describes the importance of having a good workflow manager (such as Luigi, Airflow...) when dealing with ETL or data integration. It goes through a simple implementation using Unix Bash, and a more advanced one using Python. It then presents the possible off-the-shelf solutions."
tags:
  - ETL
  - Workflow manager
  - Job manager
  - auto-retry
  - Bash
  - Python
categories: 
  - ETL
  - Workflow
---

If you have used Informatica, or to some extent Talend Studio (that is, not the free version), you know that you can chain jobs together. Now, if you need to chain jobs which use diffent technologies, or if you need more than just linear chaining and dependencies, this is where a robust workflow manager comes in handy.

# What's a workflow manager

Your company runs lots of data jobs. You use an ETL tool like [Talend](http://www.talend.com) or Pentaho [data integration](http://community.pentaho.com/projects/data-integration/) to speed up the jobs development. 

But sometimes you have to hand-code some of the jobs because they use a specific API and no connector is available for your favorite tool. These code-specific jobs do not integrate very well with your main ETL tool that has some limited job workflow built-in features (no data passing for instance). 

You integrate data from a a wide range of data sources with lots of dependencies but your ETL tool allows only for linear dependency:

![Jobs with linear dependencies](/images/workflow-manager/linear-dep.png)

This means that you have to resolve the dependencies yourself by having the proper execution order (A before B and B before C).

And what if a job fails ? Most of the open source and freemium based ETL tool will restart the complete (linear) flow. That is if _job C_ fails, they would restart from _job A_ even if _A_ and _B_ were actually successful.

# Minimum requirements implementations

These three factors:

* Integration of different technologies/languages
* Job failure handling
* Complex dependencies

They are why we need a proper and (technology) independent workflow manager.

Integrating different technologies is usually "easy", you just need a "launcher" which can simply call the job executable as long as the job can be packaged into a standalone application (Talend java package for instance) ; or can be called with an external wrapper ([kitchen](http://wiki.pentaho.com/display/EAI/Kitchen+User+Documentation) for Pentaho data integration).

Now if a job fails, the minimum we want is a retry (or more). A job can fail because of a network issue, or a time-out, it is worth trying again following a reasonable delay. The implementation logic would be if _job_ fails and the retry number limit is not reached then retry, otherwise fail the complete flow. This is illustrated in the following figure:

![](/images/workflow-manager/job-flow-with-failure-handling---Page-1-2.png)

This can be implemented with a simple bash script:

```bash
Retry_attemp_number=5

for job in ${some_job_list}
do
	exit_code=1
	let "current_attemp_number=0"

	while [[ ! $exit_code -eq 0 ]]
	do
		let "current_attemp_number=current_attemp_number+1"

		if [[ $current_attemp_number == $Retry_attemp_number ]]
			then
			echo "Max attemp reach for job ${job}"
			exit 1
		fi

		if [[ ! $exit_code -eq 0 ]]
			then
		 	echo "${job} starting" 
		else
			echo "${job} restarting" 
		fi

		# Command to launch $job
		echo "Hello world from ${job}"
		exit_code=$?

		if [[ ! $exit_code -eq 0 ]]
			then
			echo "Waiting 120 seconds before restarting job"
			sleep 120
		fi
	done

done
```

Now this implementation assumes that every job is critical and cannot be "skipped": therefore in case of an error, the complete flow will always fail. We also assume that each job needs the same maximum retry number. Wouldn't it be great if we could have all the settings in a file separated from the main script ?

# A simple workflow manager in 300 LOC

It would indeed be great. This is why I wrote a [simple workflow manager](https://github.com/ebrard/simple-workflow-manager) in about 300 LOC. This workflow manager handle:

* □ Linear dependencies
* ✓ Complete or partial failures
* ✓ Retry number specific for each job
* ✓ Time-out specific for each job 
* ✓ Skip successful jobs in case of complete restart 
* □ Single daily execution 
* ✓ Data backend: anything that SQLAlchemy handle 
* ✓ JSON external configuration file 

Here is an example of a workflow definition file:

{% highlight json %}
{  
   
   "description":"A jobs flow example",

   "retry_number":2,
   "package_dir":"/home/etl/jobs/",
   "if_failed":"skip",
   "time_out":10,

   "monitoring_db":{
      "db_type":"mssql+pymssql",
      "db":"DWH",
      "server":"DWH",
      "table":"ETL_jobs_execution_monitoring",
      "port":"1433",
      "user":"XXXX",
      "password":"xxxxx"
      },

   "jobs":[  

      {  
         "name":"job 1",
         "job_executable":"sh /home/etl/jobs/job1/run.sh",
         "if_failed":"skip",
         "retry_number":5,
         "active":true,
         "time_out":120,
         "monitored_by_workflow":true
      },

      {  
         "name":"Talend job 2",
         "job_executable":"sh /home/etl/jobs/job2/job2_run.sh",
         "parameters":"--context_param StartDate={{ StartDate }}"
         "if_failed":"failed",
         "retry_number":5,
         "active":true,
         "time_out":120,
         "monitored_by_workflow":false
      }

   ]
}
{% endhighlight %}

# Complete solutions

Although this workflow manager could be enough for an integration with a limited number of jobs (and therefore of dependencies), it does not handle complex dependencies and flow branching (if job _N_ fails then do completely something else, otherwise continue normal flow). And probably it never will because there are better workflow managers available in the market.

There are the Hadoop ones like the famous [Oozie](http://oozie.apache.org) or [LinkedIn/Azkaban](https://azkaban.github.io/), and the general purposes and [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) oriented ones.

Amongst them are three of particular interest:

* [Spotify/Luigi](https://github.com/spotify/luigi)
* [AirBnB/Airflow](https://github.com/apache/incubator-airflow)
* [pinterest/pinball](https://github.com/pinterest/pinball)

There are debates on which one to use. You can find a comparison on [Marton Trencseni's blog](http://bytepawn.com/luigi-airflow-pinball.html) (Data engineer at Facebook). I mostly agree with the conclusions. Basically the most robust and production ready is Luigi. The most features-rich and easy to use is Airflow. Pinball is not ready yet and it lacks documentation.

Note that Airflow does have real time monitoring in its web UI and alerting of failed dags/tasks by email.

As Marton Trencseni said in his post header, " If I had to build a new ETL system today from scratch, I would use Airflow", I cannot agree more with that, Airflow supports out of the box:

* Programmatic DAG-oriented workflow creation, with automatic cycle detections (yes it does happen !)
* Dynamic tasks creation (one task per file to process for instance, good isolation !)
* Multi-DAG supports (comes in handy to separate workflow by department, data avaibility...)
* Task dependencies and intra-dag dependencies (using ExternalSensorTask) 
* Tasks retry
* Flow branching 
* Technologies independent (as long as it runs on Linux and can be called by the shell)
* In-flow communication between tasks (output of one is input of another one[^1])

A quick hint here to conclude this section, all of these solutions are around 20,000 LOC Python code ...  

# And the scheduler ?

A workflow manager is not a scheduler ! That being said, the boundaries between the two tools are blurry. _Airflow_ come with its own scheduler. _Luigi_ is a workflow manager only and you need a scheduler to actually run your workflow at specific times or by specific event triggering.

To make it more complex, some schedulers have workflow management features. [Rundeck](http://rundeck.org) has linear job dependencies but no retry: if a job (step in _Rundeck_) fails, the complete workflow (job) fails, no second chance. [JobScheduler](https://www.sos-berlin.com/jobscheduler) also offers some workflow (job chains) functionalities. And if you are willing to  pay for your tool (and for much more than you need), [control-m](http://www.bmc.com/it-solutions/control-m.html) can almost do it all (except maybe for data passing between jobs).

[^1]: This is typically perfect for DoubleClick data integration for which you can have a task triggering a remote report and another task download it with data passing.
