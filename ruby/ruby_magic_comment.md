[//]: # (Ruby 2.3 中的魔法注释 # frozen_string_literal: true)

# 背景
Ruby中冻结的对象只会创建一次，以后遇到相同的对象会复用之前创建的对象，这样可以减少对象创建次数和垃圾回收次数。Ruby中的符号、整数、浮点数默认都是冻结的，字符串字面量目前还不是。

为了提高程序性能，**在Ruby 3中，字符串字面量在所有文件中默认被冻结。**

为了过渡，Ruby2.3增加了一个魔法注释：
```ruby
# frozen_string_literal: true
```
# 作用
它告诉Ruby，文件中的所有字符串字面量都被隐式冻结，不可修改，就像每一个字符串都调用了`freeze`方法一样。 
```ruby
# frozen_string_literal: true
s = "string"
puts s.frozen?      => true
s << "12"           => can't modify frozen String (RuntimeError)
```
# 使用方式

* 把该注释加在文件的第一行。

* 另外，在Ruby 2.3中使用`--enable=frozen-string-literal`标志运行ruby，也会默认冻结所有文件中的字符串字面量。在单个文件中可以通过`# frozen_string_literal: false`覆盖全局设置（Ruby 3中也可以用此方式覆盖全局设置）。
```shell
ruby --enable=frozen-string-literal t.rb
```

# 如何修改
如果想要修改字符串字面量怎么办？

**无论全局或每个文件如何设置，可以使用一元+运算符（注意优先级）来产生非冻结字符串或调用`dup`方法来对其进行复制**：
```ruby
# frozen_string_literal: true
"".frozen?
=> true
(+"").frozen?
=> false
"".dup.frozen?
=> false
```
还可以使用一元-运算符冻结可变（未冻结）字符串。
```ruby
(-"").frozen?
=> true
```

参考资料：

http://flats.github.io/blog/2016/01/03/frozen-strings/

http://ruby-performance-book.com/blog/2016/02/is-ruby-2-3-faster-frozen-string-literals-performance.html
