[//]: # (Rails中的MIME类型解析规则)

本文缘于在项目中遇到的一个问题，查阅了网上的资料和Rails源码后有一点收获，简单做个总结，有些地方不够全面，欢迎大家补充指正。

# 相关背景
Rails项目中经常可以看到类似如下代码：

```ruby
respond_to do |format|
  format.html
  format.json { render json: @users }
  format.xml  { render xml: @users }
end
```

如果想获取xml格式的数据，就在请求路径后面增加`.xml`扩展名，比如`localhost:3000/users.xml`，这样就可以拿到xml格式的返回。

**路径扩展名是Rails中MIME类型解析的一个影响因素，另一个影响因素是HTTP头字段Accept。**

## HTTP头字段Accept

当浏览器发送请求的时候，它也会通知服务器自己能处理的内容类型。访问网站的时候可以通过浏览器的开发者工具查看它们发送的Accept头。

下面是我的机器上不同浏览器发送的Accept头：

```
Chrome: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Firefox: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Safari: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
```

让我们看看Chrome的Accept头：

```
Chrome: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
```

Chrome的头表明它可以处理html文档（text/html）、xhtml文档（application/xhtml+xml）、xml文档（application/xml）、webp和apng格式的图片，以及其它别的格式。

Accept头里面有一个q值，它表示优先级。HTTP标准里面是这么描述的（[https://tools.ietf.org/html/rfc7231#section-5.3.1](https://tools.ietf.org/html/rfc7231#section-5.3.1)）：
> 5.3.1.  Quality Values

>   Many of the request header fields for proactive negotiation use a common parameter, named "q" (case-insensitive), to assign a relative "weight" to the preference for that associated kind of content.  This weight is referred to as a "quality value" (or "qvalue") because the same parameter name is often used within server configurations to assign a weight to the relative quality of the various representations that can be selected for a resource.

>   The weight is normalized to a real number in the range 0 through 1, where 0.001 is the least preferred and 1 is the most preferred; a value of 0 means "not acceptable".  If no "q" parameter is present, the default weight is 1.

简单说就是为了表示Accept中内容类型的优先级，使用一个参数q来作为权重，q是一个0到1之间的实数，最小是0.001，最大是1。如果是0表示不接受这种内容类型。如果没有指定，q默认值为1。

Accept内容类型的优先级除了与q值和顺序有关，还与类型具体化程度有关，越具体的类型优先级越高（[https://tools.ietf.org/html/rfc7231#section-5.3.2](https://tools.ietf.org/html/rfc7231#section-5.3.2)）：

> Media ranges can be overridden by more specific media ranges or
   specific media types.  If more than one media range applies to a
   given type, the most specific reference has precedence. 

举RFC里的例子：

```
Accept: text/*, text/plain, text/plain;format=flowed, */*
```
这个Accept头里，类型优先级如下：

1. text/plain;format=flowed
2. text/plain
3. text/*
4. */*

关于Accept优先级到此为止，不继续深究，有兴趣的同学可以阅读RFC原文。

# 遇到的问题
我们的项目采用前后端分离的写法，后端controller中有如下代码：

```ruby
respond_to do |format|
  format.html do
    flash[:alert] = '用户密码被重置，请重新登录'
    redirect_to new_user_session_path
  end

  format.json do
    render(json: {
              code: 0,
              location: new_user_session_path,
              msg: '用户密码被重置，请重新登录'
            })
  end
end
```

前端发送的请求头中Accept字段如下：

`Accept: application/json, text/plain, */*`

按我的理解，接口应该返回json数据，结果返回的是html重定向。
经过后来几次测试、跟踪源码，现象如下：

Accept | 返回
----|------
`application/json, text/plain, */*` | html重定向  
`application/json, text/plain` | json数据 
`*/*` | 与respond_to代码块中声明格式的顺序有关，即`format.html`、`format.json`哪个在前，返回哪个

除了第一条，后面两条基本还是符合直觉的。接下来看看Rails判断返回格式的规则到底是什么样的？

# Rails对MIME的解析

答案当然要从Rails源码中找（Read the fucking source code ^\_^），涉及的函数如下（在文件`rails/actionpack/lib/action_dispatch/http/mime_negotiation.rb`中，代码细节先不用深究，后文还有分析）：

```ruby
# order是在respond_to代码块中声明的返回格式数组，formats函数返回前端需要的格式数组，见后面的定义
def negotiate_mime(order)
  formats.each do |priority|
    if priority == Mime::ALL
      return order.first
    elsif order.include?(priority)
      return priority
    end
  end

  order.include?(Mime::ALL) ? format : nil
end
```

```ruby
# formats函数解析前端需要的格式，返回解析后的格式数组
# 其中第二个条件分支中的accepts函数会解析Accept字段，注意前提是有Accept头并且校验合法
def formats
  fetch_header("action_dispatch.request.formats") do |k|
    params_readable = begin
                        parameters[:format]
                      rescue ActionController::BadRequest
                        false
                      end

    v = if params_readable
      Array(Mime[parameters[:format]])
    elsif use_accept_header && valid_accept_header
      accepts
    elsif extension_format = format_from_path_extension
      [extension_format]
    elsif xhr?
      [Mime[:js]]
    else
      [Mime[:html]]
    end
    set_header k, v
  end
end
```

```ruby
# accepts函数调用Mime::Type.parse完成真正的解析工作
def accepts
  fetch_header("action_dispatch.request.accepts") do |k|
    header = get_header("HTTP_ACCEPT").to_s.strip

    v = if header.empty?
      [content_mime_type]
    else
      Mime::Type.parse(header)
    end
    set_header k, v
  end
end
```

```ruby
private

  # 这个正则表达式是用来判断是否是浏览器发出的请求，不是浏览器发出的请求才返回true
  BROWSER_LIKE_ACCEPTS = /,\s*\*\/\*|\*\/\*\s*,/

  # 校验函数，在formats中被调用，用来检查Accept是否正确
  def valid_accept_header # :doc:
    (xhr? && (accept.present? || content_mime_type)) ||
      (accept.present? && accept !~ BROWSER_LIKE_ACCEPTS)
      # 此处是问题的关键，如果accept头里包含*/*和其他类型（如：Accept: application/json, text/plain, */*）则此函数返回false，
      # 因此formats函数里不会解析Accept，Accept不起作用，formats函数继续往下走，走到了else分支返回html。
      # 如果只有*/*（Accept: */*），此函数返回true，negotiate_mime函数会选择respond代码块里的第一个格式类型返回（详细分析见下文）
  end
```

通过以上代码，可以大体看出Rails对返回格式的判断流程：

1. 判断请求是否带format后缀，如果带，则返回相应格式。如：http://localhost:3000/users.xml。
1. 判断请求是否有Accept头并且是合法头，如果是，则解析Accept，返回Accept中的格式。
1. 判断请求是否有`format_from_path_extension`的格式，不好意思，这个暂时没来得及研究是啥，欢迎大神们补充^\_^。
1. 判断请求是否是Ajax调用，如果是，返回javascript格式数据。
1. 以上都不满足，返回html格式。

## 对浏览器的特殊处理

从上面的代码里看到，Rails对浏览器发出的请求做了特殊处理，如果是浏览器发出的，则不解析Accept头。
为什么要对浏览器的请求做特殊处理？查阅的资料显示是因为早期浏览器设计不规范，
大部分浏览器的请求头Accept字段第一个值是`application/xml`，
如果按照Accept解析就会给浏览器用户返回xml格式的数据，
而这通常不是浏览器用户想要的。因此判断如果是浏览器就直接忽略Accept头。

最后总结一下Accept的三种情形。

## Accept三种情形
### 1. Accept不包含\*/\*

假设接口中有如下代码：

```ruby
respond_to do |format|
  format.html { render html: '<p>this is html</p>'.html_safe }
  format.json { render json: { data: 'this is json' } }
end
```

Accept的值为：

```
application/json, text/plain
```

此时`formats`函数中`valid_accept_header`返回true，Rails会解析Accept，`formats`函数最终返回如下格式顺序的数组：

```
json
plain
```

`negotiate_mime`函数中的`order`数组中包含的格式是：

```
html
json
```

这种情况下，代码遍历`formats`，如果`order`中有匹配的格式就返回。`format`的第一个格式是json，`order`中有匹配，此时返回json数据。

**结论：Accept不包含\*/\*时，按照Accept的优先级返回。**

### 2. Accept头是\*/\*

Accept的值为：

```
*/*
```

接口代码如下：

```ruby
respond_to do |format|
  format.html { render html: '<p>this is html</p>'.html_safe }
  format.json { render json: { data: 'this is json' } }
end
```

此时返回html格式。

把`respond_to`代码块调整一下顺序：

```
respond_to do |format|
  format.json { render json: { data: 'this is json' } }
  format.html { render html: '<p>this is html</p>'.html_safe }
end
```

此时返回json格式。

Accept头是\*/\*时，解析Accept的结果为`Mime::ALL`类型，从`negotiate_mime`函数代码中可以明显看出，当请求类型为`Mime::ALL`时，选择`order`中的第一个格式返回，此时会返回`respond_to`代码块中的第一个格式。

如果没有`respond_to`代码块呢？如果没有`respond_to`代码块，但是在view目录下有`test.html.erb`、`test.json.jbuilder`两个文件，Rails返回什么格式？这种情况下，Rails按顺序遍历所有注册的Mime格式，找对应的匹配文件，一旦找到文件就返回该格式。这种情况下的返回格式依赖Mime格式的注册顺序，下面是Mime格式注册的代码（`rails/actionpack/lib/action_dispatch/http/mime_types.rb`）：

```ruby
Mime::Type.register "text/html", :html, %w( application/xhtml+xml ), %w( xhtml )
Mime::Type.register "text/plain", :text, [], %w(txt)
Mime::Type.register "text/javascript", :js, %w( application/javascript application/x-javascript )
Mime::Type.register "text/css", :css
Mime::Type.register "text/calendar", :ics
Mime::Type.register "text/csv", :csv
Mime::Type.register "text/vcard", :vcf
Mime::Type.register "text/vtt", :vtt, %w(vtt)

Mime::Type.register "image/png", :png, [], %w(png)
Mime::Type.register "image/jpeg", :jpeg, [], %w(jpg jpeg jpe pjpeg)
Mime::Type.register "image/gif", :gif, [], %w(gif)
Mime::Type.register "image/bmp", :bmp, [], %w(bmp)
Mime::Type.register "image/tiff", :tiff, [], %w(tif tiff)
Mime::Type.register "image/svg+xml", :svg

Mime::Type.register "video/mpeg", :mpeg, [], %w(mpg mpeg mpe)

Mime::Type.register "audio/mpeg", :mp3, [], %w(mp1 mp2 mp3)
Mime::Type.register "audio/ogg", :ogg, [], %w(oga ogg spx opus)
Mime::Type.register "audio/aac", :m4a, %w( audio/mp4 ), %w(m4a mpg4 aac)

Mime::Type.register "video/webm", :webm, [], %w(webm)
Mime::Type.register "video/mp4", :mp4, [], %w(mp4 m4v)

Mime::Type.register "font/otf", :otf, [], %w(otf)
Mime::Type.register "font/ttf", :ttf, [], %w(ttf)
Mime::Type.register "font/woff", :woff, [], %w(woff)
Mime::Type.register "font/woff2", :woff2, [], %w(woff2)

Mime::Type.register "application/xml", :xml, %w( text/xml application/x-xml )
Mime::Type.register "application/rss+xml", :rss
Mime::Type.register "application/atom+xml", :atom
Mime::Type.register "application/x-yaml", :yaml, %w( text/yaml ), %w(yml yaml)

Mime::Type.register "multipart/form-data", :multipart_form
Mime::Type.register "application/x-www-form-urlencoded", :url_encoded_form

# https://www.ietf.org/rfc/rfc4627.txt
# http://www.json.org/JSONRequest.html
Mime::Type.register "application/json", :json, %w( text/x-json application/jsonrequest )

Mime::Type.register "application/pdf", :pdf, [], %w(pdf)
Mime::Type.register "application/zip", :zip, [], %w(zip)
Mime::Type.register "application/gzip", :gzip, %w(application/x-gzip), %w(gz)
```

很明显，`text/html`是第一个格式，因此在本例中返回`test.html.erb`文件内容。

**结论：Accept头是\*/\*时，返回`respond_to`代码块中的第一个格式。没有`respond_to`代码块时，按Mime格式注册顺序寻找对应文件返回。**

### 3. Accept头包含\*/\*和其他内容

Accept的值为：

```
application/json, text/plain, */*
```

接口代码如下：

```
respond_to do |format|
  format.json { render json: { data: 'this is json' } }
  format.html { render html: '<p>this is html</p>'.html_safe }
end
```

暂时不考虑流程中的3、4，此时返回html，不会解析Accept头（原因在于`valid_accept_header`函数返回false，对浏览器的特殊处理），也不会受`respond_to`代码块顺序影响。

**结论：Accept头包含\*/\*和其他内容，返回html。**

文笔不好，内容又多，写的有点乱，大家见谅。

参考资料：

[https://blog.bigbinary.com/2010/11/23/mime-type-resolution-in-rails.html](https://blog.bigbinary.com/2010/11/23/mime-type-resolution-in-rails.html)

[https://github.com/rails/rails/issues/9940](https://github.com/rails/rails/issues/9940)

[https://tools.ietf.org/html/rfc7231#section-5.3.1](https://tools.ietf.org/html/rfc7231#section-5.3.1)

[https://tools.ietf.org/html/rfc7231#section-5.3.2](https://tools.ietf.org/html/rfc7231#section-5.3.2)