---
layout: post
title:  "Implementing a bridge table with Talend"
date:   2017-01-19 18:00:00
categories: ETL
excerpt_separator: <!--more-->
---

In Kimball's multi-dimensional data model, a bridge table is an analytical solution to a multi-valued dimension fact in a fact table: when a fact in a fact table relates to more than one record in a dimension table (many-to-many). Creating and feeding the bridge table and the associated dimension table can be challenging: here is a solution with a RDBMS and the ETL tool Talend.

<!--more-->

A bridge table is a core implementation of a many-to-many relationship in a star-schema. The pros and cons of such a solution
for analysis purposes are realy well documentented in [Kimball's books](http://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/). This is also covered in Chris Adamson's book and an introduction 
can be read on his [blog post](http://blog.chrisadamson.com/2011/04/bridge-tables-and-many-to-many.html) ("Bridge Tables and Many-to-many Relationships").

What is less abundant in the litterature and on the Internet is how to technically implement and feed a bridge table and the relationships
with the associated dimension and fact tables ([here](http://www.kimballgroup.com/2012/02/design-tip-142-building-bridges/) is a short example completely in SQL).

In what follows, I will describe one possible implementation in a standard star-schema powered by a traditional fully SQL-compliant RDBMS and using the ETL tool Talend. We will assume also that the many-to-many original relationship is stored in a standard source table field, a CSV-like file column, or a JSON value: no matter the exact structure of the source, the relationship has been already flattened[^1].

We will use the "traditional" salesmen case: a sale man can conclude a selling contract alone or with the help of any number of other salesmen. This will bring the many-to-many relationship.

For our example, we will consider the following CSV file describing the sales representatives:

{% highlight text %}
rep_id,rep_name,rep_department
1,"Martin Paul","Electronics"
2,"John Doe","Electronics"
3,"Ed Weiss","Electronics"
4,"Arthur White","Garden"
5,"Robert Stans","Garden"
{% endhighlight %}

This will be "our source" for the dimension table representing our saleman entity.

The following CSV file describes the actual sale "transactions" from our operational source system. Note that the field _rep_id_
is actually a list of ids (the many to many relationship unpacked).

{% highlight text %}
rep_id,sale_id,amount
"1;2",1,10
"1",2,5
"2",3,100
"3",10,0.5
"3;1",5,1000
{% endhighlight %}

Another thing worth noting is that the list value _"3;1"_ is equivalent to _"1;3"_ since the storage ordering has no business meaning. This means that our ETL process needs to be independant on the order of the element in that list. Talend comes with no native function to handle list in Strings. We therefore need to write our own (you can find a ready to import routine [StringHandlingExtended](https://github.com/ebrard/talend-routine/blob/master/StringHandlingExtended.java) on my github repository):

{% highlight java %}
    public static String SortSeparatedField(String field, String separator) {
    	  
  	String[] ArrayField = field.split(separator) ;
      	Arrays.sort(ArrayField) ;
      	return String.join(separator, ArrayField) ; 
    }
{% endhighlight %}

This function takes a list as input, with a separator and returns the same list sorted. For example `SortSeparatedField("3;1",";")` and `SortSeparatedField("1;3",";")` will both return _"1;3"_.

The first step of our ETL integration in to load the representative dimension table defined as such:

{% highlight sql %}
CREATE TABLE dim_representative (
	dwh_rep_id INT NOT NULL PRIMARY KEY,
	src_sys_key INT NOT NULL, -- This is rep_id from the CSV file
	name VARCHAR(50),
	department VARCHAR(50)
) ;
{% endhighlight %}

![Jobs with linear dependencies](/images/bridge-table/load_dim.png)

Nothing special to this point.

The actual loading of our bridge is what's interesting. First this is how we define it:

{% highlight sql %}
CREATE TABLE dimension_bridge (
	group_id INT NOT NULL,
	sorted_group_content VARCHAR(100) NOT NULL,
	group_count INT NOT NULL,
	dwh_rep_id INT NOT NULL
) ;
{% endhighlight %}

Now here are the steps we need to do:

1. Read only the relevant column of the input file (rep_id)
2. Sort the content of the field, Count the number of group members (can be useful depending on the requirements)
3. Normalize the incoming flow: for each member in the group, generate a new row for this member. We use the tNormalize component on the `rep_id` field
4. Create an unique ID for the group: we use the CRC component of Talend on the sorted group field (basically a Hash of the string field), this could be done before (3)
5. Retrieve the surrogate key from the dimension table for the representative
6. Persist this data in the bridge table

![Jobs with linear dependencies](/images/bridge-table/load_bridge.png)

For example, our first record -- `"1;2",1,10` -- creates the following records in the bridge table (as displayed by the tlogrow component):

{% highlight text %}
.----------------------------------.
|       #1. bridge creation        |
+----------------------+-----------+
| key                  | value     |
+----------------------+-----------+
| group_id             | 781051917 |
| sorted_group_content | 1;2       |
| group_count          | 2         |
| dwh_rep_id           | 1         |
+----------------------+-----------+
.----------------------------------.
|       #2. bridge creation        |
+----------------------+-----------+
| key                  | value     |
+----------------------+-----------+
| group_id             | 781051917 |
| sorted_group_content | 1;2       |
| group_count          | 2         |
| dwh_rep_id           | 2         |
+----------------------+-----------+
{% endhighlight %}

After that, we need to load the facts and look-up the group `rep_id` on the bridge table. There are two ways to do so:

- Sort `rep_id` and then look-up the sorted result with the bridge table: `StringHandlingExtended.SortSeparatedField(sales_facts.rep_id,";")
=lkp_bridge.sorted_group_content`. 
- Sort `rep_id`, hash it with the CRC component and then look-up the result value with group_id

Here we are ! Our facts are now associated with a group, and the group with the group member(s). We still have a many-to-many relationship between the fact table and the bridge. But this scenario is usally properly understood by the BI tools, and the BI practionners. If your tool, or RDMS complains about this, you also need to create an [intersect table](http://blog.chrisadamson.com/2011/04/bridge-tables-and-many-to-many.html).

The following remarks has been described everywhere but I will emphasis it once more. Results implying a bridge table need to be considered with great care. The following query yields an incorrect result:

{% highlight sql %}
select
 sum(amount) as total_revenue
from (
select
  sale_id,
  fact_sales.group_id,
  group_count,
  name,
  department,
  amount
from fact_sales 
inner join testing_bridge 
	on fact_sales.group_id = testing_bridge.group_id
inner join dim_representative 
	on testing_bridge.dwh_rep_id = dim_representative.dwh_rep_id 
) as bridge_usage ;
{% endhighlight %}

whereas this one gives the correct result:

{% highlight sql %}
select
 sum(amount) as total_revenue
from (
select
  sale_id,
  fact_sales.group_id,
  group_count,
  name,
  department,
  amount/group_count as amount
from fact_sales 
inner join testing_bridge 
	on fact_sales.group_id = testing_bridge.group_id
inner join dim_representative 
	on testing_bridge.dwh_rep_id = dim_representative.dwh_rep_id 
) as bridge_usage ;
{% endhighlight %}

The complete flow is available from [my github repository](https://github.com/ebrard/talend-examples/tree/master/bridge). You will need to create the two CSV files. This flow uses a PostgreSQL database (you can get a free small db instance on [elephantsql](https://www.elephantsql.com/)).

[^1]: if this is not your case and your source in a RDBMS you may want to look at "string aggregation" features such as group_concat (MySQL), string_agg (PostGres) or XML path (SQL Server).
