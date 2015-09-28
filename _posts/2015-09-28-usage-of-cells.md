---
layout: post
title: "Usage of Cells"
description: ""
category:
tags: [cache, payment]
---


介紹使用 **Cells** 來製作 Cache ，這篇以怎麼引用 Cells 在我們的專案為主，讓之後如果有需要用到的話，可以縮短熟悉的時間。

----------

##Issue##

> [https://github.com/timeflying/andromoney_server/issues/39](https://github.com/timeflying/andromoney_server/issues/39)

##Source##

> [https://github.com/apotonick/cells](https://github.com/apotonick/cells)

##Installation##

Gemfile 加上這兩個，因為我們使用 haml 就是 cells-haml

- gem 'cells', "~> 4.0.0"
- gem 'cells-haml'


##Intro##

以下以我做 payment 的部分為例

首先 因為是要減少帳戶總覽 Query 的次數，也就是除非使用者修改了以下項目，否則理論上帳戶頁面是不會變動的，也就是可以製作 Cache

1.  使用者的  payment
2.  使用者這些 payment 的 Record
3.  Main Currency

再加上所有使用者的 cache html file 都是放在 tmp/cache/ 底下
，所以需要區分 user，這邊是使用 user_id
<br>另外，帳戶列表有區分計入總覽和不計入總覽，所以要再加上區分這兩種的 token ，所以總共需要 5 個變數來區分每份 cache key。

##Start##

> rails generate cell payment

在 app 目錄底下會產生 cells 的資料夾，會有以下的結構
<br>
 app<br>
 ├── cells<br>
 │   ├── payment<br>
 │   │   ├── show.haml<br>
 │   ├── payment_cell.rb<br>

 接下來要在專案引用的地方 (payments/index.html.haml)
 將原先的 code 換成

Origin

> %table ...

To

>  \= cell(:payment).(:show, current_user, icon_path, true)

指的是要透過 payment cell 中的 show function 來取得 view

> class PaymentCell < Cell::ViewModel<br>
>   &nbsp;&nbsp;cache :show do |user, icon_path, countIn|<br>
>   &nbsp;&nbsp;end<br>
>   <br>
>   &nbsp;&nbsp;def show(user, icon_path, countIn)<br>
>   &nbsp;&nbsp;end<br>
> end<br>

Cache 失效觸發時機可以走

1.  expire time
2.  自訂 key ，不符 key 就不使用 cache

我們的需求是自訂key，所以在 cache :show do ...
需要定義 key 後回傳，而定義的 key 組合是

1.  user id
2. payment 最後更新時間
3. record 最後更新時間
4. main currency
5. 計入或不計入

key 的取得會先執行，之後才會執行 show function

在 show function 裡取要的 payment 資料後，Cells 會直接取 show.haml 來顯示，<br>show.haml 裡就是剛剛被取代的 %table ...
show.haml 最後取得的程式就會被做成 cache存放。

##Reference##
> [http://trailblazerb.org/gems/cells/caching.html](http://trailblazerb.org/gems/cells/caching.html)<br>
> [http://blog.xdite.net/posts/2013/09/01/cells-partial-cache-1](http://blog.xdite.net/posts/2013/09/01/cells-partial-cache-1)
