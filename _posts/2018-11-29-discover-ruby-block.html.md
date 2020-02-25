---
:title: Ruby 探索：Blocks 深入淺出
:date: 2018-11-29 00:00:00 UTC
:slug: discover-ruby-block
:author: Johnson
:tags: 5xruby, 心得分享, 五倍紅寶石, Ruby, Block
:category: techpost
:image:
  :original: post/image/2018-11-30-discover-ruby-block/cover.jpg
  :preview: post/image/2018-11-30-discover-ruby-block/cover.jpg
  :thumb: post/image/2018-11-30-discover-ruby-block/cover.jpg
---
> 圖片作者：Duncan WJ Palmer    ，
> [來源連結](https://www.flickr.com/photos/blatez/16802876298/)

# Ruby 探索：Blocks 深入淺出

在學習 Ruby 的過程中，Block 是其中很重要的一環，如果對 Block 有多一點的認識就能更了解一些深奧的 code 到底是什麼意思啦！這篇文章主要是在看完 Metaprogramming Ruby 2 中的 Block 章節後的讀書心得，以及一個我自己找的一個小題目來練習學到的觀念。

---
### 目錄

* Scope 介紹
  * 什麼是 Scope ?
  * Scope Gate
* Block 介紹
  * Block 從頭開始
  * Proc 基本介紹
  * procs 和 lambdas 怎麼分？！
  * & 符號解析
* Block 實作
  * 自己做一個 before action
  * 第一步：試試看在 before 中執行 block
  * 第二步：在執行 before 的時候只有紀錄…
  * 第三步：重新定義要執行 before 的 method
  * 第四部：只有第一次執行 execute_before
* 結論


---

### Scope 介紹

#### 什麼是 Scope ?
在介紹 Block 之前，我們需要先認識 ruby 中的 scope。
誒！什麼是 scope 呢？大概就是哪個變數可以在哪邊被看到的規則，舉例來說

```ruby
x = 10
def method
  puts x
end

method
```
執行上面這段程式後會發生...

```text
test.rb:3:in 'method': undefined local variable or method 'x' for main:Object (NameError)
	from test.rb:6:in '<main>'
```

在 method 方法內和外是不同的兩個 scope，所以 method 方法不能看到它外面的變數 x。如果想知道在自己的 scope 中可以使用哪些變數可以用 `local_variables` 來看！以下面這段 code 為例：


```ruby
v1 = 1
class MyClass
  v2 = 2
  local_variables #[:v2]
  def my_method
    v3 = 3
    local_variables #[:v3]
  end
  local_variables # [:v2]
end
local_variables # [:v1, :obj]
obj = MyClass.new
obj.my_method
```
> local_variables 會回傳現在的 scope 中可以看到的區域變數

在上面的例子我們可以發現，定義 `class MyClass` 到 `end` 中間這段區域看不到變數 v1 ，同時 `def my_method` 到它的 `end` 這段也看不到變數 v2，這是因為他們在不同的 scope。

#### Scope Gates

在 Ruby 中有三個 Scope Gates：`Class definitions`, `Module definitions` 還有 `Methods` 他們就像一道牆，牆外牆內彼此看不見那裡有哪些變數。

當然也是有一些方法可以突破這樣子的限制，可以換個方式定義 Class 還有 Methods 讓他們不再是 Scope Gates，這方法叫做 Flattening the Scope (其實方法的名字好像也不是很重要)，反正就是突破那個 scope 的限制啦！

```ruby=
my_var = "Success"
MyClass = Class.new do
  puts "#{my_var} in the class definition"
  define_method :my_method do
    puts "#{my_var} in the method"
  end
end

MyClass.new.my_method
```
可以成功印出

```
Success in the class definition
Success in the method
```

### Block 介紹

#### Block 從頭開始

除了上述方法可以突破 scope 的限制外，其實 block 也是可以做到類似的效果的，它讓我們可以先寫一段程式，然後再方法裡面呼叫，就像這樣


```ruby=
def my_method
  x = "Goodbye"
  yield("cruel")
end

x = "Hello"

my_method do |y|
  puts "#{x}, #{y} world"
end
```
會印出

```
Hello, cruel world
```

上面的例子，在呼叫 `my_method` 的時候我們傳了一個 block (do 到 end 的那段)，大概可以想像他是 `my_method` 這方法的一個參數，接著 `my_method` 可以用 yield 這方法來執行這個 block。這個 block 厲害的地方在於它可以帶著當下的 bindings`(註解1)` 到 method 裡面去執行，所以在 `my_method` 可以看到 `x` 變數內的值是 Hello。

> 註解1

> bindings 指的是當下 scope 可以使用的 variable, methos 還有 self

在 Ruby 的世界裡 block 是個特殊的成員，他不是物件也不能單獨存在，但其實還是有方法可以把 block 轉成物件讓我們處理的，那就是 Proc。

#### Proc 基本介紹

```ruby=
inc = Proc.new {|x| x + 1 }
inc.call(2) # 回傳 3
```

在 Proc.new 後面接一個 block 就可以產生一個 Proc 的物件，接著可以使用 call 方法來呼叫。有很多種方式可以產生 Proc 的物件，而且用不同方式產生的 Proc 也會不太一樣，首先來介紹 proc 還有 lambda ～

#### procs 和 lambdas 怎麼分？！

```ruby=
l1 = lambda {|x| x + 1 }
l2 = ->(x){ x + 1 }
p1 = Proc.new {|x| x + 1 }
p2 = proc {|x| x + 1 }

puts "l1: #{l1.lambda?}, #{l1.class}" # l1: true, Proc
puts "l2: #{l2.lambda?}, #{l2.class}" # l2: true, Proc
puts "p1: #{p1.lambda?}, #{p1.class}" # p1: false, Proc
puts "p2: #{p2.lambda?}, #{p2.class}" # p2: false, Proc
```
上面的 `l1`, `l2`, `p1`, `p2` 都可以使用 call 方法來執行，其中我們可以用 `lambda?` 來知道它是不是 lambda 如果不是那它就是 proc 啦！除了用 `lambda` 跟 `->()` 產生的 Proc 物件是 lambda 其他都是 proc 唷！

說了那麼久怎麼產生跟判斷 proc 和 lambda，當然就要來說他們有什麼不同啦！主要有兩個不同的地方，第一個是 return 的值，第二個是參數的判斷方式

##### procs 和 lambdas 有不同的 return

```ruby=
def double(callable_object)
  callable_object.call * 2
end

# return from lambdas
la = lambda { return 10 }

# return from the scope which defined the procs(main)
pr = proc { return 10 }

puts double(la) # 20
puts double(pr) # LocalJumpError...
```
執行程式後會

```
 20
 test.rb:12:in `block in <main>': unexpected return (LocalJumpError)
 	from p92.rb:22:in `double'
 	from p92.rb:29:in `<main>'
```

lambda 的 return 是從 lambda return，而 proc 則是從定義 proc 的 scope return 意思就是如果把 proc 用一個方法包起來的話...

```ruby=
def lambda_double
  la = lambda { return 10 }
  result = la.call
  return result * 2
end

def proc_double
  pr = Proc.new { return 10 }
  result = pr.call
  return result * 2  # unreachable code!
end

puts lambda_double # => 20
puts proc_double   # => 10
```

`lambda_double` 可以執行完方法中的每行程式，而 `proc_double` 則會在 `pr.call` 那行就 return 了，因為 proc 的 return 是從定義 proc 的地方也就是 `proc_double` 出發的。

##### procs 和 lambdas 處理參數的方式

procs 處理參數的方式比較有彈性，而 lambdas 則是比較嚴格，說起來不清楚!? 看 code 最快

```ruby
puts "以下是 proc"
pr = proc {|a,b| [a,b]}
p pr.call('a', 'b')
p pr.call('a')
p pr.call('a', 'b', 'c')

puts "以下是 lambda"
la = lambda {|a,b| [a,b]}
p la.call('a', 'b')
p la.call('a')
p la.call('a', 'b', 'c')
```

執行程式後會

```
以下是 proc
["a", "b"]
["a", nil]
["a", "b"]
以下是 lambda
["a", "b"]
test.rb:8:in `block in <main>': wrong number of arguments (given 1, expected 2) (ArgumentError)
	from test.rb:10:in `<main>'
```

procs 遇到多的參數會自動丟掉，而不夠就補 nil，lambdas 則要求參數的數量正確才能執行。



#### & 符號解析

> 促使我想要多瞭解 ruby 中的 block 就是這個 & 符號了，第一次在 rails 專案裡面看到了這種符號真是既挫折又期待，覺得有很多東西可以學啦！在 C 語言中 & 是來拿記憶體位址的符號，在 ruby 裡面 & 符號到底是拿來做什麼的呢？


& 符號可以把 block 轉成 Proc 或是轉回來，多說無益看 code 就知道

```ruby=
def second_method
  puts 'In second_method'
  yield
end

def first_method(&operator) # 這裡的 & 是把 block 轉成 Proc
  puts 'In first_method'
  operator.call
  second_method(&operator) # 這裡的 & 是把 Proc 轉成 block
end

first_method do
  puts "this is block"
end
```
執行後會印出

```
In first_method
this is block
In second_method
this is block
```

在呼叫 `first_method` 得時候我們傳了一個 block 給他，我們可以把 block 想像成是最後一個參數，只是要在定義 `first_method` 的時候跟他說我們會有一個參數而且他是一個 block，因此就寫 `&operator`，之後就能在 `first_method` 中使用 operator 這個 Proc 啦～

如果想要不是馬上要用這個 Proc 還能把他先轉成 block 再傳給下一個 method，這時候就是在 `operator` 前加個 `&` 它就又從 Proc 轉成 block 囉！所以 second_method 使用 yield 才能成功呀！

以上就是關於 Block 的基本介紹啦，接下來做個小小的練習來應用一下學到的知識啦！


### Block 實作:
#### 自己做一個 before action

目標是讓下面的程式碼

```ruby=
class Example
  extend BeforeAction
  before :method_a, :method_b do
    puts 'the code before method'
  end

  def method_a
    puts 'this is method a'
  end

  def method_b
    puts 'this is method b'
  end
end

instance_1 = Example.new
instance_1.method_a
instance_2 = Example.new
instance_2.method_b
```

可以有這樣的執行結果

```ruby
the code before method
this is method a
the code before method
this is method b
```

> 要做一個 `BeforeAction` 的 Module ，在裡面有個 before 方法可以讓指定的 method 在執行之前，先執行 before 後面的 block

#### 第一步：試試看在 before 中執行 block
```ruby=
module BeforeAction
  def before(*methods, &block)
    methods.each do |method|
      block.call
    end
  end
end
```
> 現在我們可以依據定義了幾個 method 給 before 來執行幾次 block


#### 第二步：在執行 before 的時候只有紀錄...

要做到 before action 就必須覆寫裡面的 `method_a`, `method_b` 之類的，但是在執行 before 的時候根本還沒有定義 `method_a` 是什麼，所以在這邊只能先把要執行的 block 還有要覆寫的 method 名字存起來！

```ruby=
module BeforeAction
  def new
    execute_before
    super
  end

  def before(*methods, &block)
    # 先存起來之後用
    @methods = methods
    @block = block
  end

  def execute_before
    # to do ...
  end
end
```
> 存起來之後我們可以在 Example 用了 new 方法時，再來處理我們的 block，讓那些需要執行 before action 的 method 可以被覆寫

#### 第三步：重新定義要執行 before 的 method

```ruby=
module BeforeAction
  ...
  def execute_before
    @methods.each do |method|
      # 幫原本的 method 取一個新名字
      newname = "new_#{method}"
      alias_method newname, method
      block = @block
      # 重新定義每個 method 要做什麼
      define_method method do
        # 執行 block
        block.call
        # 原本的 method
        send(newname)
      end
    end
  end
end
```
> 這個步驟比較麻煩一點，重新定義方法後就會把原本的方法全部蓋過去。所以我們幫原本的方法取了個新名字(newname)，在重新定義方法後就可以在呼叫(send)來執行原本方法的程式了！

到目前為止，我們的程式已經可以讓 Example 這個 class 做這樣的事囉！如果只有 new 一次都能正常執行～

```ruby=
instance_1 = Example.new
instance_1.method_a
instance_1.method_b
```

執行結果為

```ruby
the code before method
this is method a
the code before method
this is method b
```

#### 第四部：只有第一次執行 execute_before

因為只有在第一次 new 的時候需要執行 execute_before，所以我們可以做一點小小的修改...

```ruby=
module BeforeAction
  def new
    execute_before if first_time?
    super
  end

  def first_time?
    return false if @not_first_time
    @not_first_time = true
  end

  ...
end

```

#### 最後：大功告成

完整程式碼如下

```ruby=
module BeforeAction
  def new
    # 只有第一次執行 new 時要 execute_before
    execute_before if first_time?
    super
  end

  def first_time?
    return false if @not_first_time
    @not_first_time = true
  end

  def before(*methods, &block)
    # 先存起來之後用
    @methods = methods
    @block = block
  end

  def execute_before
    @methods.each do |method|
      # 幫原本的 method 取一個新名字
      newname = "new_#{method}"
      alias_method newname, method
      block = @block
      # 重新定義每個 method 要做什麼
      define_method method do
        # 執行 block
        block.call
        # 原本的 method
        send(newname)
      end
    end
  end
end

class Example
  extend BeforeAction
  before :method_a, :method_b do
    puts 'the code before method'
  end

  def method_a
    puts 'this is method a'
  end

  def method_b
    puts 'this is method b'
  end
end

instance_1 = Example.new
instance_1.method_a
instance_2 = Example.new
instance_2.method_b
```

執行後結果

```
the code before method
this is method a
the code before method
this is method b
```


在這範例練習到把 block 轉成 Proc ，然後存起來之後使用，在每次呼叫方法的時候都先執行這個 Proc


### 結論

透過寫這篇文章好好的整理了 block 相關的觀念，不然以前總是略懂略懂，但要回想的時候又要翻書查看半天...雖然這些東西可能不是馬上需要用到，但以後如果有需要用到這些技巧的時候，再回來看這篇自己寫的文章應該可以很快回憶起來吧！
