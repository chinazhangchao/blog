[//]: # (Rails中的MIME类型解析)

https://blog.bigbinary.com/2010/11/23/mime-type-resolution-in-rails.html
https://github.com/rails/rails/issues/9940
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept

Rails项目中经常可以看到如下代码：
```ruby
respond_to do |format|
  format.html
  format.xml  { render :xml => @users }
end
```
如果想获取xml格式的输出，就要在请求后面增加`.xml`扩展比如`localhost:3000/users.xml`，这样就可以拿到xml格式的输出。
现在我们看到的只是这个问题的一个影响因素，另一个影响因素是HTTP RFC里定义的HTTP头字段Accept。

# HTTP头字段Accept

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

Chrome: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Firefox: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Safari: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

让我们看看Chrome的Accept头

Chrome: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8

Chrome的头表明它可以处理html文档（text/html），xml文档（application/xml），可以处理webp和apng格式的图片，以及其他别的格式。

头里面有一个q值，它表示优先级。HTTP标准里面是这么描述的
>5.3.1.  Quality Values

>   Many of the request header fields for proactive negotiation use a
   common parameter, named "q" (case-insensitive), to assign a relative
   "weight" to the preference for that associated kind of content.  This
   weight is referred to as a "quality value" (or "qvalue") because the
   same parameter name is often used within server configurations to
   assign a weight to the relative quality of the various
   representations that can be selected for a resource.

>Fielding & Reschke           Standards Track                   [Page 37]
 
>RFC 7231             HTTP/1.1 Semantics and Content            June 2014


>   The weight is normalized to a real number in the range 0 through 1,
   where 0.001 is the least preferred and 1 is the most preferred; a
   value of 0 means "not acceptable".  If no "q" parameter is present,
   the default weight is 1.

>     weight = OWS ";" OWS "q=" qvalue
>     qvalue = ( "0" [ "." 0*3DIGIT ] )
            / ( "1" [ "." 0*3("0") ] )

   >A sender of qvalue MUST NOT generate more than three digits after the
   decimal point.  User configuration of these values ought to be
   limited in the same fashion.
>

https://tools.ietf.org/html/rfc7231#section-5.3.2

rails/actionpack/lib/action_dispatch/http/mime_negotiation.rb

```ruby
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

  BROWSER_LIKE_ACCEPTS = /,\s*\*\/\*|\*\/\*\s*,/

  def valid_accept_header # :doc:
    (xhr? && (accept.present? || content_mime_type)) ||
      (accept.present? && accept !~ BROWSER_LIKE_ACCEPTS)
      # 此处是关键，如果accept头里包含*/*和其他类型（如：Accept: application/json, text/plain, */*）则此函数返回false，
      # 不会解析accept，因此accept不起作用，在formats函数里会继续往下走，此时大多数情况下会返回html
      # 如果只有*/*（Accept: */*），此函数返回true，negotiate_mime函数会选择respond里的第一个返回
  end
```