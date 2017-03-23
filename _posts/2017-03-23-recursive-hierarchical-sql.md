---
layout: post
title:  "Hierarchical and recursive queries in (T-)SQL"
date:   2017-03-23 19:00:00
categories:
  - SQL
  - TSQL
meta_description: This post describes the usage of T-SQL Common Table Expression (CTE) for hierarchical and recursive data queries in MS SQL Server.
tags: SQL,TSQL,CTE,Hierarchical,Recursive,Query
excerpt_separator: <!--more-->
---

"Hierarchical and recursive queries in SQL"... this is quite a hot topic, so hot that it has its own 
[wikipedia entry](https://en.wikipedia.org/wiki/Hierarchical_and_recursive_queries_in_SQL). The typical "hello world"
is employees and managers.

<!--more-->

I will describe a simple use case, the same one as the wikipedia page cited above. Let's first create the (temporary) dataset:

{% highlight sql %}
DECLARE @employees TABLE(
        empno INT NOT NULL,
        employee VARCHAR(20),
        parent_empno INT NULL
) ;

INSERT INTO @employees(empno, employee, parent_empno)
VALUES
  (7839, 'KING', NULL),
  (7566, 'JONES', 7839),
  (7788, 'SCOTT', 7566),
  (7876, 'ADAMS', 7788),
  (7902, 'FORD', 7566),
  (7369, 'SMITH', 7902),
  (7698, 'BLAKE', 7839),
  (7499, 'ALLEN', 7698),
  (7521, 'WARD', 7698),
  (7654, 'MARTIN', 7698),
  (7844, 'TURNER', 7698),
  (7900, 'JAMES', 7698),
  (7782, 'CLARK', 7839),
  (7934, 'MILLER', 7782) ;
{% endhighlight %}

This table contains both the employees information and their direct relationship. This way of storing information is 
probably good enough for applicative usage, but it is clearly limited for exploration/analysis usage.

The main ideas behing a "hierarchical and recursive query" are 

- make sense of the hierarchy 
- be able to retrieve for each record (here employee), its parent (here manager) record information

in a **single** (yet "quite" complex) query.

We have the following table (`@employees`)

| empno  | employee  | parent_empno  | 
|--------|-----------|---------------| 
|  7839  | KING      |      		     |
|  7566  | JONES     |  7839		     |
|  7788  | SCOTT     |  7566		     |
|  7876  | ADAMS     |  7788		     |
|  7902  | FORD      |  7566		     |
|  7369  | SMITH     |  7902		     |
|  7698  | BLAKE     |  7839		     |
|  7499  | ALLEN     |  7698		     |
|  7521  | WARD      |  7698		     |
|  7654  | MARTIN    |  7698		     |
|  7844  | TURNER    |  7698		     |
|  7900  | JAMES     |  7698		     |
|  7782  | CLARK     |  7839		     |
|  7934  | MILLER    |  7782		     |

We want, using a single query the same information the wikipedia page provides (using `CONNECT BY`), that is:

- the employee id
- the absolute hierarchical position
- the relative hierarchical position to the branch: the tree (presented as left padding)
- the manager ~~id~~ name, the manager id would be too easy too retrieve from `@employees.parent_empno`

The way to do it with Microsoft SQL Server dialect -- T-SQL -- is to use a [CTE](https://msdn.microsoft.com/en-us/library/ms175972.aspx) (Common Table Expression) query:

{% highlight sql %}
WITH cte (empno, employee, manager, ordering, level) AS
(
  SELECT
    empno,
    employee,
    employee as manager,
    CAST(ROW_NUMBER() OVER (ORDER BY empno) as VARCHAR(MAX)) as ordering,
    0            AS level
  FROM @employees AS _root
  WHERE parent_empno IS NULL
  UNION ALL
  SELECT
    e.empno,
    e.employee,
    d.employee as manager,
    d.ordering + '.' + CAST(ROW_NUMBER() OVER (PARTITION BY parent_empno ORDER BY e.empno) as VARCHAR(MAX)) as ordering,
    level + 1
  FROM @employees e
    INNER JOIN cte AS d ON e.parent_empno = d.empno
)
SELECT
  empno,
  level,
  replicate(' ',level) + employee as employee,
  case manager when employee then null else manager end as [direct manager]
FROM cte
  ORDER BY ordering ;
{% endhighlight %}

{% highlight text %}
| empno | level | employee | direct manager |
|-------+-------+----------+----------------|
| 7839  | 0     | KING     |                |
| 7566  | 1     |  JONES   | KING           |
| 7788  | 2     |   SCOTT  | JONES          |
| 7876  | 3     |    ADAMS | SCOTT          |
| 7902  | 2     |   FORD   | JONES          |
| 7369  | 3     |    SMITH | FORD           |
| 7698  | 1     |  BLAKE   | KING           |
| 7499  | 2     |   ALLEN  | BLAKE          |
| 7521  | 2     |   WARD   | BLAKE          |
| 7654  | 2     |   MARTIN | BLAKE          |
| 7844  | 2     |   TURNER | BLAKE          |
| 7900  | 2     |   JAMES  | BLAKE          |
| 7782  | 1     |  CLARK   | KING           |
| 7934  | 2     |   MILLER | CLARK          |
{% endhighlight %}

The `level` part comes from a built-in feature and is understood by the parser/optimzer when parsing the query. The "tree" is constructed using the row_number() aggregating function. The basic idea is too build the hierarchy as a sorting-ready string field for examples: _KING_ is `1`, _JONES_ `1.1`, _SCOTT_ `1.1.1`, _ADAMS_ `1.1.1.1` whereare _FORD_ is `1.1.2` etc. We get the "direct manager" just by following the query relationship as `d.employee` where d is our `cte`. The only trick to add is in the last step: we don't need to know that KING is his own manager:

{% highlight sql %}
case manager when employee then null else manager end as [direct manager]
{% endhighlight %}

Let's now add a new head manager and a subordinate in the dataset:

| empno  | employee  | parent_empno  | 
|--------|-----------|---------------| 
|  ...   | ...       |  ...          |
|  8000  | HENRY     |  7839         |
|  8001  | TOM       |  8000	     |

The query behaves as expected, but we somehow lack one piece of information: the top manager. Let's add it to the query:

{% highlight sql %}
WITH cte (empno, employee, manager, ordering, level, top_manager) AS
(
  SELECT
    empno,
    employee,
    employee as manager,
    CAST(ROW_NUMBER() OVER (ORDER BY empno) as VARCHAR(MAX)) as ordering,
    0            AS level,
    employee AS top_manager
  FROM @employees AS _root
  WHERE parent_empno IS NULL
  UNION ALL
  SELECT
    e.empno,
    e.employee,
    d.employee as manager,
    d.ordering + '.' + CAST(ROW_NUMBER() OVER (PARTITION BY parent_empno ORDER BY e.empno) as VARCHAR(MAX)) as ordering,
      level + 1,
    top_manager
  FROM @employees e
    INNER JOIN cte AS d ON e.parent_empno = d.empno
)
SELECT
  empno,
  level,
  replicate(' ',level) + employee as employee,
  case manager when employee then null else manager end as [direct manager],
  case top_manager
    when employee then null
    else top_manager
  end as top_manager
FROM cte
  ORDER BY ordering, level ;
{% endhighlight %}

The `top_manager` also comes from a built-in feature, and we use the same trick to avoid displaying non-useful information.

We have seen the typical example with an employees table but there are much more situations where the hierarchy between records is stored within the same table. A common example in the e-commerce business is credits granted to customers and their usage. For example, a customer is granted 100 USD that yields a record with `id=1, amount=+100, parent_id=null` in a table somewhere. The customer starts using 60 USD yielding a record with `id=6, amount=-60, parent_id=1` and then uses it
again for 40 USD: `id=200, amount=-40, parent_id=6`. Now we can with a single query see the complete credit usage for the initial deposit _id=1_.
