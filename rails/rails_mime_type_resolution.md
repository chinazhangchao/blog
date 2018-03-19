[//]: # (Rails中的MIME类型解析)

在项目中遇到了一个MIME解析相关的问题，在此记录一下。

# 相关背景
Rails项目中经常可以看到如下代码：
```ruby
respond_to do |format|
  format.html
  format.json { render json: @users }
  format.xml  { render xml: @users }
end
```

如果想获取xml格式的输出，就要在请求后面增加`.xml`扩展比如`localhost:3000/users.xml`，这样就可以拿到xml格式的输出。
现在我们看到的只是这个问题的一个影响因素，另一个影响因素是HTTP RFC里定义的HTTP头字段Accept。

## HTTP头字段Accept

当浏览器发送请求的时候，它也会告知服务器自己能处理的内容类型。下面是一些浏览器Accept头的例子：

```
text/plain

image/gif, images/x-xbitmap, images/jpeg, application/vnd.ms-excel, application/msword,
application/vnd.ms-powerpoint, */*

text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

application/vnd.wap.wmlscriptc, text/vnd.wap.wml, application/vnd.wap.xhtml+xml,
application/xhtml+xml, text/html, multipart/mixed, */*
```

当你访问网站的时候可以通过浏览器的开发者工具查看他们发送的Accept头。

这是我的机器上不同浏览器发送的Accept

```
Chrome: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Firefox: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Safari: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
```

让我们看看Chrome的Accept头

```
Chrome: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
```

Chrome的头表明它可以处理html文档（text/html），xml文档（application/xml），可以处理webp和apng格式的图片，以及其他别的格式。

头里面有一个q值，它表示优先级。HTTP标准里面是这么描述的（[https://tools.ietf.org/html/rfc7231#section-5.3.1](https://tools.ietf.org/html/rfc7231#section-5.3.1)）
>5.3.1.  Quality Values

>   Many of the request header fields for proactive negotiation use a
   common parameter, named "q" (case-insensitive), to assign a relative
   "weight" to the preference for that associated kind of content.  This
   weight is referred to as a "quality value" (or "qvalue") because the
   same parameter name is often used within server configurations to
   assign a weight to the relative quality of the various
   representations that can be selected for a resource.

>   The weight is normalized to a real number in the range 0 through 1,
   where 0.001 is the least preferred and 1 is the most preferred; a
   value of 0 means "not acceptable".  If no "q" parameter is present,
   the default weight is 1.

     weight = OWS ";" OWS "q=" qvalue
     qvalue = ( "0" [ "." 0*3DIGIT ] )
            / ( "1" [ "." 0*3("0") ] )

   >A sender of qvalue MUST NOT generate more than three digits after the
   decimal point.  User configuration of these values ought to be
   limited in the same fashion.
>

简单说就是为了表示选择内容类型的优先级，使用一个参数q来作为权重，q是一个0到1之间的实数，最小是0.001，最大是1。如果是0表示不接受这种类型。如果没有指定，q默认值为1。

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

按常规的理解，接口应该返回json数据，结果返回了html重定向。
经过后来几次测试、跟踪源码，现象如下：

Accept | 返回
----|------
`application/json, text/plain, */*` | html重定向  
`application/json, text/plain` | json数据 
`*/*` | 与respond_to代码块中声明格式的顺序有关，即`format.html`、`format.json`哪个在前，返回哪个

产生上述现象的原因是什么？Rails的解析规则是什么？

# Rails对MIME的处理

上面的现象需要从Rails源码中找原因，涉及的函数如下（在文件`rails/actionpack/lib/action_dispatch/http/mime_negotiation.rb`中）：


```ruby
# order是在respond_to代码库中声明的返回类型数组，formats函数返回前端需要的类别，见下面的定义
# [#<Mime::Type:0x007ffded079048 @synonyms=["application/xhtml+xml"], @symbol=:html, @string="text/html", @hash=-4499798999559200672>, #<Mime::Type:0x007ffdedb9bd40 @synonyms=["text/x-json", "application/jsonrequest"], @symbol=:json, @string="application/json", @hash=-3893990215891182727>]

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
# 解析前端需要的格式，其中accepts函数会解析Accept字段，注意前提是有Accept头并且校验合法
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

  # 校验函数
  def valid_accept_header # :doc:
    (xhr? && (accept.present? || content_mime_type)) ||
      (accept.present? && accept !~ BROWSER_LIKE_ACCEPTS)
      # 此处是关键，如果accept头里包含*/*和其他类型（如：Accept: application/json, text/plain, */*）则此函数返回false，
      # 不会解析accept，因此accept不起作用，在formats函数里会继续往下走，此时大多数情况下会返回html
      # 如果只有*/*（Accept: */*），此函数返回true，negotiate_mime函数会选择respond里的第一个返回
  end
```

通过以上代码，可以看出，Rails对返回格式的判断过程。

1. 请求是否带format后缀，如果带，则使用相应格式。如：http://localhost:3000/users.xml 
2. 请求是否有Accept头并且是合法头，如果是，则解析Accept。

## 原因
Rails为什么要对浏览器的请求做特殊处理？搜索的资料显示是因为早起浏览器设计不规范，大部分请求头Accept字段第一个值是`application/xml`，如果按照规则就会给浏览器用户返回xml格式的数据，而不通常不是浏览器用户想要的。

## Accept总结
### 1. Accept头是\*/\*
### 2. Accept不包含\*/\*
### 3. Accept头包含\*/\*和其他内容
### 4. Ajax


参考资料：

[https://blog.bigbinary.com/2010/11/23/mime-type-resolution-in-rails.html](https://blog.bigbinary.com/2010/11/23/mime-type-resolution-in-rails.html)

[https://github.com/rails/rails/issues/9940](https://github.com/rails/rails/issues/9940)

[https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept)

[https://tools.ietf.org/html/rfc7231#section-5.3.1](https://tools.ietf.org/html/rfc7231#section-5.3.1)