---
layout: post
title:  "Variables declaration ordering error in Informatica and Talend"
date:   2017-07-29 18:00:00
meta_description: "This post explains the difference in Informatica and Talend with respect to variables declaration errors"
categories: 
- ETL
tags:
- job
- design
- tMap
- expression
- Informatica
- Talend
- variables
excerpt_separator: <!--more-->
---

Understanding how in-flow variables are declared by your ETL tool is important. Probably all of them requires the user to declare the variables in the right orders. But what happens when there is an ordering error in a component of your flow.

<!--more-->

You can declare variables in Informatica expression transformations. Talend offers the same functionality with variables in the tMap components. Both tools need that you declare the variables you use that depend on one another in the right order, for example:

{% highlight java %}

Variable1 = input 
Variable2 = 2xVariable2

{% endhighlight %} 

is perfectly fine but: 

{% highlight java %}

Variable1 = 2xVariable2 
Variable2 = input

{% endhighlight %} 

will bring issue. This is totally normal and fine.

The difference between the two ETL tools is that Informatica will validate the mapping with no error, then execute it throught the corresponding workflow also with no error, but as you may imagine `Variable1` will be empty (NULL-valued) during run-time.

Talend will fail the build phase. Hereafter you can see a simple input to output job (corresponding to a mapping in Informatica):

![Example of a job with a variable dependency error](/images/variables-dep/flow.png)

with the content of the tMap (similar to an expression transformation in this case):

![Example of a tmap with a variable dependency error](/images/variables-dep/tmap.png)

the build error:

![Example of a job with a variable dependency error: build error](/images/variables-dep/build_error.png)

and the error message from the source code:

![Example of a job with a variable dependency error](/images/variables-dep/code_error.png)

Informatica is the market leader and has been out here for quite a while, and yet this simple developer mistake is not spotted anywhere from development to run-time. This issue, amongst others, is a reason why Informatica (9.x) is not doing well in a truly CD/CI environment. Any standard programming IDE would also have spotted this issue and the compiler would have failed, just like Talend and the Java compiler.
