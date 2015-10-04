---
layout: post
title: "Development環境下如何建立假使用者與假資料"
description: ""
category: 
tags: []
---
###緣由：
在製作Calendar view時，必須要有假使用者以及假資料看來畫面呈現。

###1. 建立假使用者：

config/initializers/omniauth.rb:

    case Rails.env
      when "production"
        ... # config in production env
      when "development" 
              
	    Rails.application.config.middleware.use OmniAuth::Builder do
	      provider :google_oauth2,
               'OAUTH_CLIENT_ID',
               'OAUTH_CLIENT_SECRET',
               {name: "google_login", approval_prompt: ''}
    	end
    
	    OmniAuth.config.test_mode = true # 讓登入直接導到 google_login/callback
	
    	OmniAuth.config.mock_auth[:google_login] = { # 設定google_login的預設auth hash
        	"provider" => "google_oauth2",
	        "uid" => "123456789",
    	    "info" => {
        	    "name" => "John Doe",
            	"email" => "john@andromoney.com",
	            "first_name" => "John",
    	        "last_name" => "Doe",
        	    "image" => "https://lh3.googleusercontent.com/url/photo.jpg"
	        },
    	    "credentials" => {
        	    "token" => "token",
            	"refresh_token" => "another_token",
	            "expires_at" => 2041008560,
    	        "expires" => true
        	}
	    }	  
	else
        ... # config in test env
	end

當這樣設定完後，在google_login/callback (SessionsController#create)拿到的 request.env["omniauth.auth"] 就會是mock_auth所設定的auth hash了。	

###2. 建立假資料

1. 使用[Fabricator gem](http://www.fabricationgem.org/)跟[Faker gem](https://github.com/stympy/faker)：

		gem install fabrication
		gem install faker
	
2. 在/spec/fabricators定義各個model的fabricator：

	user_fabricator:
	
		Fabricator(:user) do #使用model的名字當參數
		  email { Faker::Internet.email }
		  name { Faker::Name.name }
		end

	sample_user_fabricator:

		#若要新增一個特殊的fabricator，必須指定是從哪個fabricator繼承來的。
		Fabricator(:sample_user, from: :user) do 	
		  name {"John Doe"}
		  email {"john@andromoney.com"}
		  provider {"google_login"}
		end
	
	record_fabricator:
	
		Fabricator(:record) do
		  mount { Faker::Number.between(100, 10000) } #使用Faker隨機產生資料
		  amount_to_main { Faker::Number.between(100, 10000) }
		  date { Time.zone.today + (Faker::Number.between(-20, 20) * -1).days + \
		  	(Faker::Number.between(0, 23)).hours }
		  hash_key { Faker::Lorem.characters(20) }
		  is_delete false
		  remark { Faker::Lorem.paragraph(sentence_count=7) }
		  device_uuid { Faker::Lorem.characters(20) }
		end

	其他fabricator不一一詳列。

3. 製作建立假資料的腳本/lib/sample.rb

		#建立時model時可以動態指定所要建立的model的attributes
		currency1= Fabricate(:currency, currency_code: 'TWD', sequence_status: 1)
		currency2= Fabricate(:currency, currency_code: 'GBP', sequence_status: 2)
		currency3= Fabricate(:currency, currency_code: 'AZN', sequence_status: 0)
		
		sample_user = Fabricate(:sample_user)
		sample_user.currencies << currency1
		sample_user.currencies << currency2
		sample_user.currencies << currency3

		...以下不詳列
		
4. 新增一個rake task在 /lib/tasks/dev.rake中
	
		desc "generate dev seed"
		task :import_sample => :environment do
    	  Rake::Task["db:reset"].invoke #Reset DB
		  load File.join(pwd,'lib','sample.rb') #呼叫sample.rb建立假資料。
		end	

這樣之後要建立development環境下的假資料只要下
	
	rake dev:import_sample
	
就可以馬上產生了。

{% include JB/setup %}
