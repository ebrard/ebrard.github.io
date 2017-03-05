---
layout: post
title:  "Incremental data loading with Talend"
date:   2017-02-28 18:00:00
categories: ETL
excerpt_separator: <!--more-->
---

One of the unavoidable challenge in any data integration architecture is choosing the right loading technique. Push change data capture (CDC) is more than often not available. Full (re)load, batch delta load and incremental load (CDC pull) are potential candidates. In this post, I will describe a possible implementation of incremental load using Talend. Only the relevant part for CDC will be described (no processing of insourcing data).

<!--more-->


