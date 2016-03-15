---
layout: post
title:  "分享一次排查问题的过程"
date:   2016-03-15 10:34:59 +0800
categories: ruby
---

### 问题描述
 
 发现于一个专门提供API的应用，请求量比较低，几十cpm的水平，通过监控查看到会有极少量请求比较慢，时间在两三秒的样子，这个API是返回用户所拥有优惠券的数量，使用了SQL count实现。监控上看到该API的一般响应时间大约几十毫秒，只是个别请求的响应时间比较长。
 
### 服务器概况
 
Rails4.2, Nginx, Puma, PostgreSQL, Grape, Sidekiq, Redis

服务器、PostgreSQL和redis都用的阿里云。

### 定位问题的过程

#### 1. 看监控

- 监控的位置定位
 
我们使用的是OneAPM作为监控的。
![](https://ruby-china-files.b0.upaiyun.com/photo/2016/81f52e101b03b1c9d85eb3a612aaf629.png)
第一张图片可以看出数据库查询的时间是可以接受的，时间都耗在了Rack上。

![](https://ruby-china-files.b0.upaiyun.com/photo/2016/5410f66d36fede256dc4ac9f0fcd0096.png)
详情中也没有显示具体的慢的位置，没有说明是哪个Rack，范围太大，没什么实际帮助。

当时有过怀疑，是不是OneAPM的监控不准呢？在Nginx日志中是可以显示出响应的处理时间，这个时间相对OneAPM来说要更准确。

在Nginx的配置中为log_format增加$upstream_response_time或者$request_time，默认是没有这2个值得。
这台服务器在搭建环境的时候就已经配置好了。经过确认，响应时间在1-3秒的请求每天都会有几十个。

- 服务器负载分析

  查看OneAPM以及阿里云上的资源监控，没有发现任何异常。请求响应迟钝的那段时间前后，也没有耗时或者有可能造成堵塞的其他请求。
  
#### 2. 分析代码

接口的实现中只有3行代码，逻辑很简单, 单表count查询，grape中也没有去渲染其他数据，返回一个数量。

Puma的问题？ Grape的问题？ Rails？ 数据库？ 锁？…………


#### 3. 是否慢在业务代码的范围内？

用了最笨的办法，在每一行前后打上日志，缩小范围。

结果：Rails默认的请求日志和手动添加的第一条日志，这2个日志的间隔时间比较长。
可以确认的是，已经执行到了Rails的代码中，和Puma的关系不大了。同样，也不是业务代码的原因。

#### 4. 是否慢在了某个Middleware内？
已经打印了请求日志，但是海米进入到接口代码中，因为可以开率是否是因为某个middleware处理慢了导致的。middleware的相关介绍查看[这里](http://blog.gauravchande.com/what-is-rack-in-ruby-rails)。在config/application.rb添加下面的代码：

{% highlight ruby linenos %}
config.after_initialize do
  Rails.configuration.middleware.middlewares.each do |middleware|
    middleware_class = middleware.klass
    if middleware_class.class === Class && middleware_class.method_defined?(:call)
      middleware_class.class_eval do
        alias :old_call :call

        def call env
          Rails.logger.info "-------#{self.class.name}"
          old_call(env)
        end
      end
    end
  end
end
{% endhighlight %}

(未完待续)
