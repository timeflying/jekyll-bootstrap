---
layout: post
title: "如何使特定 CSS 排除在 application.css 之外"
description: ""
category:
tags: []
---
{% include JB/setup %}

Rails 使用 [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) 來串聯及最小化 Javascript 和 CSS 資源，預設會建立 **app/assets/stylesheets/application.css** 內容如下

```css
/*
*= require_self
*= require_tree .
*/
```

`require_tree` 這行指令表示要 [Sprockets](https://github.com/sstephenson/sprockets) 將該目錄底下的所有 CSS 檔案輸出

有的時候為了特定頁面的設計，我們想要將某些 CSS 排除在 application.css 外，藉由以下步驟：

首先分別建立兩種不同的 layout

假設有一組 CSS 是只給首頁使用，接著移動這組 CSS 到 **app/assets/stylesheets/landing**；至於其他則移到 **app/assets/stylesheets/conten**，最後將共用的移動到 **app/assets/stylesheets/common**

接著編輯以下三個清單：

``` css
/*
 * application-common.css
 *
 *= require_self
 *= require_tree ./common
 */
```

``` css
/*
 * application-landing.css
 *
 *= require_self
 *= require_tree ./landing
 */
```

``` css
/*
 * application-content.css
 *
 *= require_self
 *= require_tree ./content
 */
```

還需加入以下設定：

``` ruby
config.assets.precompile += %w( application-common.css application-landing.css application-content.css )
```

這樣就能妥善管理 CSS 檔案

## 使用 Rails 4 關鍵字

以上範例，首頁只需要特定的 CSS，而不用載入所有的 CSS 檔案，使用上述方式似乎太多餘了，我們只需要將一個檔案從其他頁面指定到首頁，只要使用 Rails 4 所提供的關鍵字 `stub`

``` css
/*
 * application.css
 *
 *= require_self
 *= require_tree .
 *= stub 'landing'
 */
```

這樣 **application.css** 就只會讀取除了 **landing.css** 以外的檔案

接著在首頁上加上：

``` erb
<%= stylesheet_link_tag  'landing' %>
```

且別忘了加入設定 **config/initializers/assets.rb**

``` ruby
Rails.application.config.assets.precompile += %w( landing.css )
```

這樣我們就達成了首頁只會讀取 **landing.css** 檔案，其他則不會
