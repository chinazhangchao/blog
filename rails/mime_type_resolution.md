[//]: # (Rails MIME解析)

https://blog.bigbinary.com/2010/11/23/mime-type-resolution-in-rails.html
https://github.com/rails/rails/issues/9940
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept

rails/actionpack/lib/action_dispatch/http/mime_negotiation.rb
```ruby
private

  BROWSER_LIKE_ACCEPTS = /,\s*\*\/\*|\*\/\*\s*,/
  def valid_accept_header # :doc:
    (xhr? && (accept.present? || content_mime_type)) ||
      (accept.present? && accept !~ BROWSER_LIKE_ACCEPTS)
      # 此处是关键，如果accept头里包含*/*（如：Accept: application/json, text/plain, */*）则此函数返回false，
      # 不会解析accept，因此accept不起作用，在formats函数里会继续往下走，此时大多数情况下会返回html
  end
```