---
layout: post
title: "User Cost Prediction"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### 1. Algorithm

#### Assumption
  
  We assume the cost of a user for a day is related to the cost of the yesterday.
  Therefore, to predict cost for a specific day. e.g. 2017-05-02, we can use the information of the day before the prediction day, e.g. 2017-05-01.
  We can use a vector where each element is the number of spent times for each subcategory for the day as the information.
  (1, 0, 0, 2, 0, 4, ....), then we search for the top k similar days by cosine similarity, 
  
  
  e.g.
  
  * 2017-03-01 (1, 0, 0, 1, 0, 4, ...)
  * 2017-01-21 (1, 0, 0, 2, 0, 3, ...)
  * ...
  
  
  Since the cost for a day is related to its yesterday, we can summarize the day after the top k similar days,
  
  
  e.g.
  
  * 2017-03-02 (0, 1, 3, 1, 0, 0)
  * 2017-01-22 (1, 0, 2, 1, 0, 1)
  * ....
  
  
  We use average on these vectors 
  
  e.g. `(0.3, 0.02, ....)`
  
  Finally, we calculate the average cost for each subcategory based on the users spending history 
  
  e.g. `(245.2, 1000.5, 391.2,...)`
  
  and dot product these two vectors `(0.3, 0.02, ...) * (245.2, 1000.5, 391.2, ...) = 10402`, then it's the prediction result.
 
2. Code
 
