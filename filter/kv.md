# Key-Value 切分

在很多情况下，日志内容本身都是一个类似于 key-value 的格式，但是格式具体的样式却是多种多样的。logstash 提供 `filters/kv` 插件，帮助处理不同样式的 key-value 日志，变成实际的 LogStash::Event 数据。

## 配置示例

```
filter {
    ruby {
        init => "@kname = ['method','uri','verb']"
        code => "event.append(Hash[@kname.zip(event['request'].split(' '))])"
    }
    if [uri] {
        ruby {
            init => "@kname = ['url_path','url_args']"
            code => "event.append(Hash[@kname.zip(event['uri'].split('?'))])"
        }
        kv {
            prefix => "url_"
            source => "url_args"
            field_split => "&"
            remove_field => [ "url_args", "uri", "request" ]
        }
    }
}
```

## 解释

Nginx 访问日志中的 `$request`，通过这段配置，可以详细切分成 `method`, `url_path`, `verb`, `url_a`, `url_b` ...




## 案例详情
经过三斗室大神的细心指导，然后再看官方文档，总是弄明白了kv应该怎么切分nginx的request字段。

nginx日志定义如下：
```    log_format main     "$http_x_forwarded_for ◊ $time_local ◊ $request ◊ $status ◊ $body_bytes_sent ◊ "
                        "$request_body ◊ $content_length ◊ $http_referer ◊ $http_user_agent ◊ $nuid ◊ "
                        "$http_cookie ◊ $remote_addr ◊ $hostname ◊ $upstream_addr ◊ $upstream_response_time ◊ $request_time”;
```
实际日志如下：
第一行：117.136.9.248 ◊ 08/Apr/2015:16:00:01 +0800 ◊ POST /notice/newmessage?sign=cba4f614e05db285850cadc696fcdad0&token=JAGQ92Mjs3--gik_b_DsPIQHcyMKYGpD&did=4b749736ac70f12df700b18cd6d051d5&osn=android&osv=4.0.4&appv=3.0.1&net=460-02-2g&longitude=120.393006&latitude=36.178329&
ch=360&lp=1&ver=1&ts=1428479998151&im=869736012353958&sw=0&sh=0&la=zh-CN&lm=weixin&dt=vivoS11t HTTP/1.1 ◊ 200 ◊ 132 ◊ abcd-sign-v1://dd03c57f8cb6fcef919ab5df66f2903f:d51asq5yslwnyz5t/{\x22type\x22:4,\x22uid\x22:7567306} ◊ 89 ◊ - ◊ abcd/3.0.1, Android/4.0.4, vivo S11t ◊
nuid=0C0A0A0A01E02455EA7CF47E02FD072C1428480001.157 ◊ - ◊ 10.10.10.13 ◊ bnx02.abcdprivate.com ◊ 10.10.10.22:9999 ◊ 0.022 ◊ 0.022

第二行：59.50.44.53 ◊ 08/Apr/2015:16:00:01 +0800 ◊ POST /feed/pubList?appv=3.0.3&did=89da72550de488328e2aba5d97850e9f&dt=iPhone6%2C2&im=B48C21F3-487E-4071-9742-DC6D61710888&la=cn&latitude=0.000000&lm=weixin&longitude=0.000000&lp=-1.000000&net=0-0-wifi&osn=iOS&osv=8.1.3&sh=568.0
00000&sw=320.000000&token=7NobA7asg3Jb6n9o4ETdPXyNNiHwMs4J&ts=1428480001275 HTTP/1.1 ◊ 200 ◊ 983 ◊ abcd-sign-v1://b398870a0b25b29aae65cd553addc43d:72214ee85d7cca22/{\x22nextkey\x22:\x22\x22,\x22uid\x22:\x2213062545\x22,\x22token\x22:\x227NobA7asg3Jb6n9o4ETdPXyNNiHwMs4J\
x22} ◊ 139 ◊ - ◊ Shopping/3.0.3 (iPhone; iOS 8.1.3; Scale/2.00) ◊ nuid=0C0A0A0A81DF2455017D548502E48E2E1428480001.154 ◊ nuid=CgoKDFUk34GFVH0BLo7kAg== ◊ 10.10.10.11 ◊ bnx02.abcdprivate.com ◊ 10.10.10.35:9999 ◊ 0.025 ◊ 0.026

很明显，日志request中的字段，顺序是乱的，第一行，问号之后的第一个字段是sign，第二行问号之后的第一个字段是appv，所以需要将字段进行切分，取出每个字段对应的值。官方自带grok满足不了要求。

经过三斗室大神的指导，详细filter如下：

```
filter {
	ruby {
	init => "@kname = ['http_x_forwarded_for','time_local','request','status','body_bytes_sent','request_body','content_length','
http_referer','http_user_agent','nuid','http_cookie','remote_addr','hostname','upstream_addr','upstream_response_time','request_time'
]"
	code => "event.append(Hash[@kname.zip(event['message'].split('◊'))])"
	}
	if [request] {
        	ruby {
            	init => "@kname = ['method','uri','verb']"
            	code => "event.append(Hash[@kname.zip(event['request'].split(' '))])"
        	}
		if [uri] {
        		ruby {
            		init => "@kname = ['url_path','url_args']"
            		code => "event.append(Hash[@kname.zip(event['request'].split('?'))])"
        		}
        		kv {
            		prefix => "url_"
            		source => "url_args"
            		field_split => "& "
            		remove_field => [ "url_args","uri","request" ]
        		}
    		}
	}
}
```

解释：
第一个init是初始化nginx的日志格式，通过◊进行切分，取出nginx日志定义的每一个大字段；
if [request]，判断request，init初始化request中的信息，以 空格进行切分，在切分完request之后，再通过if [uri]再次以问号进行切分；将uri信息切分成两段，最后通过kv插件，已url_args作为数据源，以&作为分隔符，在每个字段前面加入url_

===最终结果：

```
{
                   "message" => "1.43.3.188 ◊ 08/Apr/2015:16:00:01 +0800 ◊ POST /search/suggest?appv=3.0.3&did=dfd5629d705d400795f698055806f01d&dt=iPhone7%2C2&im=AC926907-27AA-4A10-9916-C5DC75F29399&la=cn&latitude=-33.903867&lm=sina&longitude=151.208137&lp=-1.000000&net=0-0-wifi&osn=iOS&osv=8.1.3&sh=667.000000&sw=375.000000&token=_ovaPz6Ue68ybBuhXustPbG-xf1WbsPO&ts=1428480001567 HTTP/1.1 ◊ 200 ◊ 353 ◊ abcd-sign-v1://a24b478486d3bb92ed89a901541b60a5:b23e9d2c14fe6755/{\\x22key\\x22:\\x22last\\x22,\\x22offset\\x22:\\x220\\x22,\\x22token\\x22:\\x22_ovaPz6Ue68ybBuhXustPbG-xf1WbsPO\\x22,\\x22limit\\x22:\\x2220\\x22} ◊ 148 ◊ - ◊ abcdShopping/3.0.3 (iPhone; iOS 8.1.3; Scale/2.00) ◊ nuid=0B0A0A0A9A64AF54F97634640230944E1428480001.113 ◊ nuid=CgoKC1SvZJpkNHb5TpQwAg== ◊ 10.10.10.11 ◊ bnx02.abcdprivate.com ◊ 10.10.10.26:9999 ◊ 0.070 ◊ 0.071",
                  "@version" => "1",
                "@timestamp" => "2015-04-11T11:10:33.271+08:00",
                      "type" => "nginxapiaccess",
                      "host" => "blog05.abcdprivate.com",
                      "path" => "/home/nginx/logs/api.access.log",
      "http_x_forwarded_for" => "1.43.3.188 ",
                "time_local" => " 08/Apr/2015:16:00:01 +0800 ",
                    "status" => " 200 ",
           "body_bytes_sent" => " 353 ",
              "request_body" => " abcd-sign-v1://a24b478486d3bb92ed89a901541b60a5:b23e9d2c14fe6755/{\\x22key\\x22:\\x22last\\x22,\\x22offset\\x22:\\x220\\x22,\\x22token\\x22:\\x22_ovaPz6Ue68ybBuhXustPbG-xf1WbsPO\\x22,\\x22limit\\x22:\\x2220\\x22} ",
            "content_length" => " 148 ",
              "http_referer" => " - ",
           "http_user_agent" => " abcdShopping/3.0.3 (iPhone; iOS 8.1.3; Scale/2.00) ",
                      "nuid" => " nuid=0B0A0A0A9A64AF54F97634640230944E1428480001.113 ",
               "http_cookie" => " nuid=CgoKC1SvZJpkNHb5TpQwAg== ",
               "remote_addr" => " 10.10.10.11 ",
                  "hostname" => " bnx02.abcdprivate.com ",
             "upstream_addr" => " 10.10.10.26:9999 ",
    "upstream_response_time" => " 0.070 ",
              "request_time" => " 0.071",
                    "method" => "POST",
                      "verb" => "HTTP/1.1",
                  "url_path" => " POST /search/suggest",
                  "url_appv" => "3.0.3",
                   "url_did" => "dfd5629d705d400795f698055806f01d",
                    "url_dt" => "iPhone7%2C2",
                    "url_im" => "AC926907-27AA-4A10-9916-C5DC75F29399",
                    "url_la" => "cn",
              "url_latitude" => "-33.903867",
                    "url_lm" => "sina",
             "url_longitude" => "151.208137",
                    "url_lp" => "-1.000000",
                   "url_net" => "0-0-wifi",
                   "url_osn" => "iOS",
                   "url_osv" => "8.1.3",
                    "url_sh" => "667.000000",
                    "url_sw" => "375.000000",
                 "url_token" => "_ovaPz6Ue68ybBuhXustPbG-xf1WbsPO",
                    "url_ts" => "1428480001567"
}
```
