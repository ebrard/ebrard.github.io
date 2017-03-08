---
layout: post
title:  "Hierarchical and recursive queries in (T-)SQL"
date:   2017-03-27 18:00:00
categories: TSQL
excerpt_separator: <!--more-->
---

"Hierarchical and recursive queries in SQL"... this is quite a hot topic, so hot that it has its own 
[wikipedia entry](https://en.wikipedia.org/wiki/Hierarchical_and_recursive_queries_in_SQL). The typical example
of its use case are employees, their managers and their hierarchy level.

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

The main idea behing a "hierarchical and recursive query" are 

- to make sense of this hierarchy 
- to be able to retrieve for each record (here employee), its parent (here manager) record information

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

The way to do it with Microsoft SQL Server dialect, TSQL, is to use a [CTE](https://msdn.microsoft.com/en-us/library/ms175972.aspx) (Common Table Expression) query.

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
  ORDER BY ordering, level ;
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

The `level` part comes from a built-in feature and is understood by the parser/optimzer when parsing the query.
