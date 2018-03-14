[//]: # (Ruby 安全调用运算符 (&.))

今天在阅读discourse源码时，看到其中有很多地方使用`&.`符号，如：
```ruby
result << types[:whisper] if viewed_by&.staff?
```
不明白干啥的，搜索了一下，发现国外一篇文章总结的很好，在此记录一下。

安全调用运算符（&.）是Ruby 2.3.0中最吸引人的新增特性之一。类似的运算符在C#和Groovy语言中早已存在，用的语法略有不同，是符号（?.）。那么这个运算符的作用是什么呢？
# 使用场景
假设有一个account对象，它有一个关联的owner对象，现在想要获取owner的address属性。稳妥的不引发nil异常的写法如下：
```ruby
if account && account.owner && account.owner.address
...
end
```
这种写法很啰嗦、很不爽。ActiveSupport提供的try方法有类似的写法（两者稍微有一点区别，我们稍后讨论）：
```ruby
if account.try(:owner).try(:address)
...
end
```
这段代码完成同样的功能——它要么返回address字段值要么返回nil（当调用链中有nil值时）。但第一个例子的写法也有可能返回false，比如当owner值为false的时候。
# 使用&.
我们可以使用安全调用运算符重写前面的例子：
```ruby
account&.owner&.address
```
虽然语法看起来有点奇怪，但我猜你会很快爱上它，因为它的确让代码更简洁。
# 更多例子
接下来让我们详细比较一下这三种方式：
```ruby
account = Account.new(owner: nil) # account without an owner

account.owner.address
# => NoMethodError: undefined method `address' for nil:NilClass

account && account.owner && account.owner.address
# => nil

account.try(:owner).try(:address)
# => nil

account&.owner&.address
# => nil
```
这个例子没有什么特别的。但如果owner的值为false呢（虽然可能性不大，但在垃圾代码的狂躁世界里并非不可能）？
```ruby
account = Account.new(owner: false)

account.owner.address
# => NoMethodError: undefined method `address' for false:FalseClass `

account && account.owner && account.owner.address
# => false

account.try(:owner).try(:address)
# => nil

account&.owner&.address
# => undefined method `address' for false:FalseClass`
```
**第一个亮点出现了——&.运算符只跳过nil但可以识别false！它与`s1 && s1.s2 && s1.s2.s3`这种写法并不完全一样。**
如果owner不为空但是并没有address属性（方法）呢？
```ruby
account = Account.new(owner: Object.new)

account.owner.address
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>

account && account.owner && account.owner.address
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>`

account.try(:owner).try(:address)
# => nil

account&.owner&.address
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>`
```
**很遗憾，try方法没有检查接收者是否响应给定的方法。这就是为什么应该尽量使用try的严格版本——try!的原因：**
```
account.try!(:owner).try!(:address)
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>`
```
# 注意事项
这段作者也有疑惑，我就自己发挥一下了。第一个例子不用解释。第二个我理解为先调用nil?，此时返回false（主对象main调用nil?的结果）。然后再调用nil?，还是返回false（相当于false.nil?）。**第三个例子表明&.符号在对象为nil时不会再去检查是否响应对应的方法**。
```
nil.nil?
# => true

nil?.nil?
# => false

nil&.nil?
# => nil
```
# Array#dig and Hash#dig
在我看来，dig方法是这个版本中最有用的特性。我们再也不用写下面这样恶心的代码：
```
address = params[:account].try(:[], :owner).try(:[], :address)

# or

address = params[:account].fetch(:owner) .fetch(:address)
```
只需简单的使用Hash#dig来达到同样的目的：
```
address = params.dig(:account, :owner, :address)
```

# 写在后面
我非常不喜欢处理动态语言中的空值，安全调用运算符和dig的出现使得代码可以写得非常简洁。（最后一句不翻了，Ruby 2.3.0早已发布，祝大家编码愉快`^_^`）

英文原文地址：http://mitrev.net/ruby/2015/11/13/the-operator-in-ruby/