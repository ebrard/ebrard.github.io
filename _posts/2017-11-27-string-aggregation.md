---
layout: post
title:  "String aggregation with databases and ETL tools"
date:   2017-11-27
meta_description: "String aggregations with Postgres, MS SQL, Talend and Informatica, a naive implementation in Python"
categories: 
  - data engineer
  - etl
  - databases
tags:
  - postgres
  - ms sql server
  - sql
  - python
  - talend
excerpt_separator: <!--more-->
---

String aggregation is the process of concatenating strings (usualy as a list with a separator) based on a common key. Databases implement their own function to do so such as wm_concat or LISTAGG in Oracle, group_concat in MySQL, array_agg or string_agg with PostGres. Some databases doesn't support it at all, such as MS SQL Server (before the 2017 version) and require workaround. The implementation on the ETL tools side also depends on the vendor.  

<!--more-->

String aggregation is a simple operation to do in any high level programming language. A possible implementation in Python 3 can be:

{% highlight python %} 
records = [
(1, 'Matt'),
(1, 'Rocks'),
(2, 'Stylus'),
(3, 'Foo'),
(3, 'Bar'),
(3, 'Baz')]

agg_data = dict()

for (k, v) in records:
  if k in agg_data:
    agg_data[k].append(v)
  else:
    agg_data[k] = list()
    agg_data[k].append(v)

for k, v in agg_data.items():
  print(k,",".join(v))
{% endhighlight %}

Which gives as expected:

{% highlight text %} 
1 Matt,Rocks
2 Stylus
3 Foo,Bar,Baz
{% endhighlight %}


The same can be obtained with SQL for databases that support string aggregation. For example with PostGres 9.x:

{% highlight sql %} 
CREATE TEMPORARY TABLE records 
(
  k int,
  v varchar(100)
) ;

INSERT INTO records(k, v)
values
(1, 'Matt'),
(1, 'Rocks'),
(2, 'Stylus'),
(3, 'Foo'),
(3, 'Bar'),
(3, 'Baz') ;

select k, string_agg(v, ', ') from records group by k ;
{% endhighlight %}

Or, for Oracle:

{% highlight sql %} 
INSERT INTO records(k, v) values (1, 'Matt')   ;
INSERT INTO records(k, v) values (1, 'Rocks')  ;
INSERT INTO records(k, v) values (2, 'Stylus') ;
INSERT INTO records(k, v) values (3, 'Foo')    ;
INSERT INTO records(k, v) values (3, 'Bar')    ;  
INSERT INTO records(k, v) values (3, 'Baz')    ;

SELECT 
k, 
LISTAGG(v, ',') WITHIN GROUP (ORDER BY v)
from records
group by k ;
{% endhighlight %}

MS SQL Server prior to version 2017 requires a workaround, one of them is the well-known XML method:

{% highlight sql %} 
select  k
        ,Names = stuff((select ', ' + v as [text()]
        from records xt
        where xt.k = t.k
        for xml path('')), 1, 1, '')
from records t
group by k ;
{% endhighlight %}

Unfortunately, the same is true for ETL tools. Informatica (the market leader) has no built-in function for string aggregation and requires a workaround too (see here).

With Talend, there are (at least) two ways to do it. Depending on your background, you may find one of them more intuitive. In Talend, you can aggregate strings using the tAggregateRow transformation or the tDenormalize transformation. 


![Example of a string aggregation with Talend](/images/string-aggregate/string_aggregate_job_talend.png)


Giving the following input (randomly generated),

{% highlight text %} 
.---+----------.
|LogGeneratedData|
|=--+---------=|
|key|value     |
|=--+---------=|
|2  |Woodrow   |
|3  |Herbert   |
|3  |Lyndon    |
|4  |John      |
|4  |Rutherford|
|4  |William   |
|8  |Ronald    |
|9  |Calvin    |
|9  |Grover    |
|9  |Millard   |
'---+----------'
{% endhighlight %}

Both the aggregation on list method

{% highlight text %} 
.---+-----------------------.
|     LogAggregatedData     |
|=--+----------------------=|
|key|value                  |
|=--+----------------------=|
|2  |Woodrow                |
|3  |Herbert,Lyndon         |
|4  |John,Rutherford,William|
|8  |Ronald                 |
|9  |Calvin,Grover,Millard  |
'---+-----------------------'
{% endhighlight %}

or the denormalization will produce the same outputs:

{% highlight text %} 
.---+-----------------------.
|    LogDenormalizedData    |
|=--+----------------------=|
|key|value                  |
|=--+----------------------=|
|2  |Woodrow                |
|3  |Herbert;Lyndon         |
|4  |John;Rutherford;William|
|8  |Ronald                 |
|9  |Calvin;Grover;Millard  |
'---+-----------------------'
{% endhighlight %}

