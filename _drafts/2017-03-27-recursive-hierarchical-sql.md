---
layout: post
title:  "Hierarchical and recursive queries in SQL"
date:   2017-03-27 18:00:00
categories: ETL
excerpt_separator: <!--more-->
---

"Hierarchical and recursive queries in SQL"... this is quite a hot topic, so hot that it has its own 
[wikipedia entry](https://en.wikipedia.org/wiki/Hierarchical_and_recursive_queries_in_SQL). The typical example
of its use case are employees, their managers and their hierarchy level.

<!--more-->

I will describe a simple complete use case, the same one as the wikipedia page. Let's first create the (temporary) dataset:

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

in a **single** (yet quite complex) query.

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
