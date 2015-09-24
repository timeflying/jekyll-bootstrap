---
layout: post
title: "首發文 簡易說明"
description: ""
category: 
tags: []
---
{% include JB/setup %}

1. 先git clone<br>
git clone https://github.com/timeflying/timeflying.github.com.git<br>

2. 安裝jekyll<br>
gem install jekyll<br>

3. local端執行（打開browser輸入http:localhost:4000)<br>
jekyll serve<br>

4. 發文<br>
rake post title="標題"<br>
會產生檔案
./_posts/2015-09-24-標題.md

5. 上傳<br>
git add .<br>
git commit -m "commit message"<br>
git push<br>

<br>



文章測試:
 [google]: http://google.com/        "Google"
  [yahoo]:  http://search.yahoo.com/  "Yahoo Search"
  [msn]:    http://search.msn.com/    "MSN Search"

I get 10 times more traffic from [Google](http://google.com/ "Google")
than from [Yahoo](http://www.yahoo.com.tw) or
[MSN](http://search.msn.com/ "MSN Search").

