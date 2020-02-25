---
:title: Rails探索：Active Record 篇
:date: 2018-03-16 03:38 UTC
:slug: Active_Record
:author: Johnson
:tags: ''
:category: techpost
:image:
  :original: post/image/137/hannah-wei-84051-unsplash.jpg
  :preview: post/image/137/preview_hannah-wei-84051-unsplash.jpg
  :thumb: post/image/137/thumb_hannah-wei-84051-unsplash.jpg

---





## 動手做 Active Record！

探索 Active Record 最直接的方法可能就是自己動手做一個吧!?

這篇文章將會做一個簡單版的 Active Record ，讀者有興趣的話可以跟著說明一起來做一個。這自製的 Active Record 有部分寫法是直接參考現有 Rails 的 Active Record ，也有部分則是依據我們需要的功能直接寫的程式，整體來說不管在效能、複雜度還是嚴謹性都和 Rails 的  Active Record  差很多，但整個 gem 的製作過程讓我對 Rails 還有 Ruby 都有更深一層的認識，所以在這裡寫成文章分享一下。

>這篇文章預設讀者已經有安裝 `PostgreSQL` `Rails` `Bundler` 這些東西囉！所以還沒安裝的人可能還需要先安裝一下~

---
### 目錄

* 準備工作
  * 新增Gem
  * 加入測試
  * 資料庫設定
* 程式碼實作
  * 步驟0: 環境設定
  * 步驟1: 通過 attrbutes 測試
  * 步驟2: 通過 save 測試
  * 步驟3: 通過 find 測試
  * 步驟4: 通過 where 測試
* 補充
* 結論

---
## 準備工作

### 新增Gem

Rails 的 Active Record 是一個 gem ，我們也將做一個名為 `activerecord_0216002` 的 gem ，使用 bundler 來幫我們快速產生 gem 吧，首先在 terminal 使用 bundler 新增一個 gem

```
bundler gem activerecord_0216002
```

第一次設定時可以設定要選擇 rspec 還是 minitest ( 文章範例使用 rspec )

```vim
Do you want to generate tests with your gem?
Type 'rspec' or 'minitest' to generate those test files now and in the future. rspec/minitest/(none):
```

建立完成專案後我們可以用 `rspec` 指令來看看專案還缺少甚麼東西所以不能跑測試

輸入 `rspec` 看看會發生什麼事？

```vim
The validation error was '"FIXME" or "TODO" is not a description'
```

這時候要到 `activerecord0216002.gemspec` 檔案去設定一下，
修改有 `"TODO"` 字樣的選項，這邊我偷懶一點就是都改成`''`

```
  spec.summary       = ''
  spec.description   = ''
  spec.homepage      = ''
```

完成後再一次輸入 `rspec` ，這時後可能會發現這專案有缺少的 gem 所以就再輸入 `bundle install` ，接著再輸入 `rspec` 就能正常測試囉！


---
### 加入測試
首先我們要覆蓋掉 bundler 幫我預設的測試，將我們的 `activerecord_0216002_spec.rb` 改成下面這樣，以下測試的的項目也就是我們的 gem 要實作的功能唷！

`activerecord_0216002_spec.rb`

```ruby
require 'rspec'
require 'byebug'
# require "activerecord_0216002"
require "active_record"

ActiveRecord::Base.establish_connection(
  adapter: 'postgresql',
  database: 'my_db'
)

class User < ActiveRecord::Base
end

RSpec.describe "Test User" do
  before(:all)do
    User.new(name: 'Ancestor', phone: '0932445631', age: 12).save
  end

  it '1 test model with attrbutes' do
    user = User.new(name: 'Bob', phone: '0932445631', age: 12)
    expect(user.name).to eq('Bob')
    expect(user.phone).to eq('0932445631')
    expect(user.age).to eq(12)
  end

  it '2 test model save record' do
    user = User.new(name: 'Bob')
    expect{user.save}.to change{User.all.count}.by(1)
    expect{user.save}.to change{User.all.count}.by(0)
  end

  it '2 test model save with value' do
    User.new(name: 'Peter').save
    expect(User.last.name).to eq('Peter')
  end

  it '3 test find' do
    user = User.find(User.first.id)
    expect(user.id).to eq(User.first.id)
  end

  it '4 test where chain' do
    User.new(name: 'A', age: 20, phone: '000').save
    User.new(name: 'A', age: 30, phone: '6666').save
    User.new(name: 'B', age: 30, phone: '000').save
    user = User.where("name = 'A'").where("age > 25").where("age > 25").where("age > 25").first
    expect(user.phone).to eq('6666')
  end
end
```

>整份測試會做一個 User Model 然後測試自製的 new、 save、 find 和 where這些功能

完成後再一次輸入 `rspec` 看看我們還缺少甚麼？
可能會發現少了一些 gem

```
LoadError:
  cannot load such file -- pg
```

所以我們再一次打開 `activerecord_0216002.gemspec` 設定我們的 gem 要使用哪些其他 gem

`activerecord_0216002.gemspec`

```ruby
  spec.add_development_dependency "pg"
  spec.add_development_dependency "byebug"
```

* pg 是讓我們的 Ruby 程式碼可以更方便的使用 postgresql 的 gem
* byebug 是幫助我追蹤程式碼使用的 gem


再一次輸入 `rspec` 會發現有一些沒設定的常數，因為我們還沒開始設計我們的 Active Record 嘛！

```
NameError:
  uninitialized constant ActiveRecord
```

在開始設計 Active Record 之前，我們可以測試一下
我們的測試在 Rails 原本的 Active Record 下能不能正常運作？

老樣子到 `activerecord_0216002.gemspec` 加上
`spec.add_development_dependency "activerecord"`
接著修改一下測試，讓我們直接用 Rails 原本的 Active Record

```ruby
# require "activerecord_0216002"
require "active_record"
```

使用 rspec 跑跑看！ 這時候會發現...

```
ActiveRecord::NoDatabaseError
```

其實一般在 Rails 專案裡面的時候， Rails 幫我們做完太多事了
還記得在 Rails 專案的時候我們常會輸入 `rails db:create` (建立資料庫)和`rails db:migrate` (設定資料表)這些指令嗎？

現在我們要自己到 psql 的介面手動設定這些東西啦><

---
### 資料庫設定

到 PostgreSQL 的介面建立資料庫和資料表
在 terminal 輸入 `psql` 進入 PostgreSQL 資料庫
進入 psql 介面後有以下指令可以使用

常用指令說明:

* `\l` 列出資料庫
* `\q` 離開psql介面
* `\c` 連接資料庫
* `\d` 列出該資料庫的所有資料表
* `\d 資料表名`  列出資料表內的欄位

首先我們要建立名為 `my_db` 的資料庫
在資料庫介面輸入 `CREATE DATABASE my_db;`
建立完成後可以輸入 `\c my_db` 進入 my_db 資料庫
進入後可以看到輸入介面變成`my_db=#`
接著輸入` \d `可以發現我們的 `my_db` 資料庫應該是空的還沒有資料表


我們要做兩個資料表，一個是 users 資料表，另一個是 users_id_seq 資料表，這是 users 資料表的 id 欄位
輸入以下 SQL 指令

```
CREATE SEQUENCE users_id_seq START 1;
CREATE TABLE users(id bigint DEFAULT nextval('users_id_seq') PRIMARY KEY
,name character varying, phone character varying, age integer);
```
輸入完後再用` \d `確認是否新增資料表成功，
也可以在使用 `\d users` `\d users_id_seq` 確認這兩張資料表的欄位
確認完成後輸入 `\q` 離開psql介面！

---
## 程式碼實作

以上資料庫設定完成後，讓我們再輸入一次 `rspec` 看看吧！
沒意外的話應該是能順利通過這五個測試
接下來把測試的3,4行調整成

```ruby
require "activerecord_0216002"
# require "active_record"
```
也就是我們要用自己的 Active Record 啦

---
提示：

* 接下來的教學會分成 0-4 個步驟來一步步完成
* 每個步驟也會對應到1-2個測試
* 每個測試名稱開頭的號碼就是第幾個步驟
* 每個 method 上面也會註解一個數字，代表這是第幾步驟需要用到

### 步驟0 : 環境設定

所謂環境設定也就是我想要通過測試檔的5-8行
要做出 `ActiveRecord::Base` 且讓他有 `establish_connection` 方法
這其實就是跟 Active Record 說你可以去找哪個資料庫

```ruby
ActiveRecord::Base.establish_connection(
  adapter: 'postgresql',
  database: 'my_db'
)
```

首先我們要先把 Active Record 的框架做出來
我們會將程式碼拆成6個rb檔

```
 lib
  ├── activerecord_0216002.rb
  └── activerecord_0216002
    ├── base.rb
    ├── method.rb
    ├── relation.rb
    ├── persistence.rb
    └── connection_adapter.rb
```


`activerecord_0216002.rb`

```ruby
module ActiveRecord
  autoload :Base, "activerecord_0216002/base"
  autoload :Method, "activerecord_0216002/method"
  autoload :Relation, "activerecord_0216002/relation"
  autoload :Persistence, "activerecord_0216002/persistence"
  autoload :ConnectionAdapter, "activerecord_0216002/connection_adapter"
end
```

在 `activerecord_0216002.rb` 中我們會把需要的檔案都 `autoload` 進來，以第三行 `autoload :Base, "activerecord_0216002/base"` 來說就是程式執行後看到常數譬如`ActiveRecord::Base` 就知道要去找`activerecord_0216002/base` 的檔案，`autoload`就和 `require` 很像，不會重複讀取同份檔案多次浪費資源，而且`autoload` 比 `require` 更聰明的地方是他的動作就像是 "登記"，假如今天 `ActiveRecord::Base` 這常數沒被使用到，那 `activerecord_0216002/base` 檔案也不會被讀取

再來將`activerecord_0216002`資料夾下的5個rb檔案新增出來

`base.rb`

```ruby
module ActiveRecord
  class Base
    include Persistence
    extend Method
  end
end
```


在Rails中有著這樣的關係 `YourModel << ActiveRecord::Base` ，也就是說我們平常我們在操作的 model 其實也就是繼承了 `ActiveRecord::Base` 的 class 。在 `base.rb` 中我將 instance method 放在 `persistence.rb` 中，而class method 放在 `method.rb` ，因此分別使用了 `include` 和 `extend`

`method.rb`

```ruby
module ActiveRecord
  module Method
    #0
    def establish_connection(option)
      case option[:adapter]
      when 'postgresql'
        @@connection = ConnectionAdapter::PostgreSQLAdapter.new(option[:database])
      when 'sqlite'
        #TODO
      end
    end

    #0
    def connection
      @@connection
    end
  end
end
```
> `method.rb` 裡面的每個method都是 `ActiveRecord::Base` 的 class method 唷！


在 `establish_connection` 這動作中會設定我們的 `ActiveRecord::Base` 將如何連線到資料庫，用 `option[:adapter]` 決定"哪種"資料庫，用`option[:database]` 決定"哪個"資料庫!可以看到測試那邊選了 `postgresql` 這種資料庫下面的 `my_db` 資料庫！

另一個比較特別的是這邊使用 `@@connection` 類別變數來儲存資料，原因是我們需要用 "類別變數" 來讓之後繼承了 `ActiveRecord::Base` 的 `User` 這個 class 可以使用這個變數


`connection_adapter.rb`

```ruby
module ActiveRecord
  module ConnectionAdapter
    class PostgreSQLAdapter
      def initialize(dbname)
        require 'pg'
        @db = PG.connect(dbname: dbname)
      end

      def execute(sql)
        @db.exec(sql)
      end
    end
  end
end

```
在Rails中我們可能使用不同的資料庫(PostgreSQL, SQLite等等)，為了讓`ActiveRecord`能支援多種資料庫而且不浪費記憶體，`ConnectionAdapter` 會依據使用者選了不同類型的資料庫然後 `require` 不同的資料庫進來(如第5行) `require 'pg'`



`persistence.rb`

```ruby
module ActiveRecord
  module Persistence
  end
end
```
> 還只是第 0 步可以先這樣寫，而relation.rb 部分可以第二部後再開始XD

### 步驟1 : 通過 attrbutes 測試
`activerecord_0216002_spec.rb`

```ruby
  it '1 test model with attrbutes' do
    user = User.new(name: 'Bob', phone: '0932445631', age: 12)
    expect(user.name).to eq('Bob')
    expect(user.phone).to eq('0932445631')
    expect(user.age).to eq(12)
  end
```
我們要通過第一個 attrbutes 的測試，希望可以讓 Model 的 class 可以 `new` 出一個 instance ，而且 instance 可以帶有參數如上面測試的 name、phone 之類的


`persistence.rb`

```ruby
    #1
    def initialize(attributes = {})
      self.class.set_column_name_to_method_attribute
      @attributes = attributes
    end
```
我們在 `persistence.rb` 加上`initialize` 這個 `instance method` ，這也就是讓 `ActiveRecord::Base` 有這 `instance method` ，這也就像一般在寫`Ruby`時我們會直接寫 `def initialize ...` 而非 `def self.new ...`。

其中我用 `@attributes` 來儲存model初始化的參數，而 `set_column_name_to_method_attribute` 是用來動態為model產生 attribute method ，因為 `set_column_name_to_method_attribute` 本身是`class method`所以我就統一整理在 `method.rb` 囉！

> attribute method 就是說 user.phone、user.name 這些


`method.rb`

```ruby
    #1
    def set_column_name_to_method_attribute
      columns = self.connection.execute("SELECT column_name FROM information_schema.columns
        WHERE table_name= '#{self.table_name}'").map{|m|  m["column_name"]}
      columns.each{ |e| define_method_attribute(e) }
    end

    #1
    def define_method_attribute(name)
      class_eval <<-STR
        def #{name}
          @attributes[:#{name}] || @attributes["#{name}"]
        end

        def #{name}=(value)
          @attributes[:#{name}] = value
        end
      STR
    end

    #1
    def table_name
      name.downcase + "s"
    end
```

在 `set_column_name_to_method_attribute` 方法中我先透過 `columns` 取得一個model在資料表中擁有的欄位，譬如能從User中找到 `["id", "name", "phone", "age"]` 這些欄位，接下來在第二個 `define_method_attribute` 方法中透過`class_eval`動態產生 id、name、phone 這些 attribute method。


為什麼是使用 `class_eval` 來定義 method 呢？

>現在的情境是我們在 class 的 scope 裡定義方法，我們直接定義方法的話都會變成class method，也就是把 id、name 之類的方法直接定義在 `User` 上，而 `clsss_eval` 的功能就是他能讓你在 class 的 scope 裡定義 instance method!

為什麼使用 `<<-STR ... STR`   字串型態而非常見的`do ... end`呢？

>因為我想要定義的method名稱是個變數，他可能是 id、name 之類的，所以透過 STR 我能用`#{name}`這種方法來讓`def`後面的 method 名稱是個傳進來的值。其中這個 STR 你可以換成其他你喜歡的字譬如 <<-abc 之類的


我們可以做個小小的實驗看 `class_eval` 有什麼樣的功能！
執行以下這檔案
`test.rb`

```ruby
module Machine
  def start
    class_eval do
      def go
        puts 'Go!!'
      end
    end

    def stop
      puts 'Stop!!'
    end
  end
end

class Car
  extend Machine
end

c = Car.new
Car.start
c.go
c.stop
```
執行 `ruby test.rb` 結果是...

```
Go!!
test.rb:24:in `<main>': undefined method `stop' for #<Car:0x007fca0c8a03d0> (NoMethodError)
```
`go` 方法成功定義成 instance method ，而 `stop` 方法會變成 Car 的 class method ，因為第16行做了這事 `extend Machine` ，想要在這情況下做出instance method就要靠 `class_eval` 啦！

---

### 步驟2 : 通過 save 測試
`activerecord_0216002_spec.rb`

```ruby
  it '2 test model save record' do
    user = User.new(name: 'Bob')
    expect{user.save}.to change{User.all.count}.by(1)
    expect{user.save}.to change{User.all.count}.by(0)
  end

  it '2 test model save with value' do
    User.new(name: 'Peter').save
    expect(User.last.name).to eq('Peter')
  end
```

26-30行的測試是測驗 `user` 呼叫 save 之後，能不能成功新增一筆資料，同時第29行測第二次呼叫save後會不會重複新增資料，32-35行的測試要看 save 後能否將正確的 attributes 存入資料庫。

步驟二的程式分成部分，第一部分是 "儲存" 資料，第二部分是 "讀取" 資料

---
`persistence.rb` 完全解鎖

```ruby
module ActiveRecord
  module Persistence
    #1
    def initialize(attributes = {})
      self.class.set_column_name_to_method_attribute
      @attributes = attributes
      @new_record = true
    end

    #2
    def new_record?
      @new_record
    end

    #2
    def save
      if new_record?
        self.class.connection.execute("INSERT INTO #{self.class.table_name}
        (#{@attributes.keys.join(',')}) VALUES (#{"'"+@attributes.values.join("', '")+"'"})")
        @new_record = false
        true
      else
        false
      end
    end
  end
end

```
`persistence` 的程式就是在做 "儲存" 資料的工作，在 `initialize` 部分我們新增了 `@new_record` 用來檢查這個 model 有沒有被 save 過，在下方 `save` method也可以看到對應的判斷。

`save` method 中將使用 connection 方法來讓 SQL 指令直接對資料表插入資料，而我們能取的的原始資料( `@attributes.keys`、`@attributes.kes` )格式還需經過 `.join(',')` 來轉換，
`@attributes.keys` 和 `@attributes.values` 轉換方式如下

>[:name, :phone, :age] => "name, phone, age"

>["Ancestor", "0932445631", 12] => "'Ancestor', '0932445631', '12'"

`method.rb`

```ruby
    #2
    def all
      Relation.new(self).records
    end

    #2
    def last
      all.last
    end

    #2
    def find_by_sql(sql)
      connection.execute(sql).map do |attributes|
        new(attributes)
      end
    end
```
`relation.rb`

```ruby
module ActiveRecord
  class Relation
    #2
    def initialize(klass)
      @klass = klass
    end

    #2
    def to_sql
      "SELECT * FROM #{@klass.table_name}"
    end

    #2
    def records
      @records ||= @klass.find_by_sql(to_sql)
    end
  end
end
```

講完儲存資料後，接下來是 "讀取" 資料的部分。

這邊你會發現我們特別做了一個 `Relation` 的 class 來處理 "讀取" 資料的部分，為什麼要這麼大費周章呢？其實是在幫步驟4 `where` 的功能鋪路，記得在Rails中 `where` 或是 `all` 方法會回傳什麼東西嗎？不是 array 而是 `XX_model::ActiveRecord_Relation` ，現在我們也就是要做出類似的功能唷！

在 `method.rb` 的 `all` 方法中我們將 `User` 這個 class 丟進參數中，做出一個 `User` 的 `Relation` 的物件，現階段這個物件功能最主要就是 `records` 方法，他會再呼叫再 `method.rb` 中定義的 `find_by_sql
` 來回傳一個 User的物件陣列。

在這邊我很偷懶的直接讓 `method.rb` 中的 `all` 方法就是等於直接回傳呼叫 `records` 的結果，也就是說測試中的 `User.all` 其實是直接回傳一個陣列喔！這跟一般Rails裡面的狀況不一樣！


---

### 步驟3 : 通過 find 測試
`activerecord_0216002_spec.rb`

```ruby
  before(:all)do
    User.new(name: 'Ancestor', phone: '0932445631', age: 12).save
  end
```
```ruby
  it '3 test find' do
    user = User.find(User.first.id)
    expect(user.id).to eq(User.first.id)
  end
```
到第三個測試時我們就會使用 `before(:all)` 來確保測試之前資料庫裡面至少有一筆資料XD，步驟3 相對前面的步驟相對簡單，只要在做出 `find` 功能就好囉！

`method.rb`

```ruby
    #3
    def first
      all.first
    end

    #3
    def find(id)
      find_by_sql("SELECT * FROM #{table_name} WHERE id = #{id.to_i}").first
    end
```

這個 `first` 就和步驟2的 `last` 一樣意思，就是把 `all` 回傳的陣列直接用 `ruby` 的方式取出第一個或最後一個值。

`find` 其實跟 `relation.rb` 裡的 `records` 方法很像，就是透過 `find_by_sql` 來跑一段SQL指令而已，這時候會回傳一個陣列所以我在最後面加上 `.first` 。


---
### 步驟4 : 通過 where 測試
`activerecord_0216002_spec.rb`

```ruby
  it '4 test where chain' do
    User.new(name: 'A', age: 20, phone: '000').save
    User.new(name: 'A', age: 30, phone: '6666').save
    User.new(name: 'B', age: 30, phone: '000').save
    user = User.where("name = 'A'").where("age > 25").first
    expect(user.phone).to eq('6666')
  end
```


我們的 `where` 支援 Chain，也就是你能一直 `where` 下去`User.where(..).where(..)`，乍看之下 `where` 像是 User 的 class method 但是他也是 Relation 的 instance method ，究竟是怎麼回事呢？讓我們看下去...

>我們的 `where` 相當基本，他只能直接用SQL來查詢，像這樣`where('id = 1')` ，沒辦法用Hash查詢，就是沒有 `where(id:1)` 這種方法


`method.rb`

```ruby
    #4
    def where(*args)
      Relation.new(self).where(*args)
    end
```

`method.rb` 裡的 where 就是 User 的 class method ，裡面在做的事情也就是做出一個 Relation 物件然後再用 Relation 物件呼叫 Relation 的 where ，這也就是我們能一直 where 下去的原因啦！
無論是第一個 User 叫出來的where還是第 n 個 ActiveRecord_Relation 叫出來的 where 都是一樣的！

`relation.rb`

```ruby
module ActiveRecord
  class Relation
    #2
    def initialize(klass)
      @klass = klass
      @where_values = []
    end

    #2
    def to_sql
      sql = "SELECT * FROM #{@klass.table_name}"
      if @where_values.any?
        sql += " WHERE " + @where_values.join(" AND ")
      end
      sql
    end

    #2
    def records
      @records ||= @klass.find_by_sql(to_sql)
    end

    #4
    def where(condition)
      @where_values += [condition]
      self
    end

    #4
    def first
      records.first
    end
  end
end
```
我們將步驟2做的 `Relation` 又更加擴充了，加上了 `@where_values` 來儲存多個 where 的資料，如何儲存的呢？在每一次 where 後都會 `@where_values += [condition]` 讓這個陣列越來越長，最後在呼叫 `records` 方法時，就會在 `to_sql` 方法中用 `@where_values.join(" AND ")` 把陣列的值都取出來然後用 `AND` 接再一起！

> to_sql 方法就是把測試的資料從
> where("name = 'A'").where("age > 25")
> 變成
>"SELECT * FROM users WHERE name = 'A' AND age > 25"


最後最後，再一次輸入 `rspec` 看看會發生什麼事 ?

```vim
Test User
  1 test model with attrbutes
  2 test model save record
  2 test model save with value
  3 test find
  4 test where chain

Finished in 0.46207 seconds (files took 0.52716 seconds to load)
5 examples, 0 failures
```

就這樣我們完成了四個步驟，通過了五個測試，做出了自己的 Active Record 啦 :)

## 補充

範例程式碼放在我的 [GitHub](https://github.com/shes50103/activerecord_0216002)
如果你想把整份 gem 安裝在自己電腦裡可以輸入

`gem install activerecord_0216002`

把這個 gem 裝在自己的電腦裡，裝完後可以再輸入

`gem env`

然後找 `INSTALLATION DIRECTORY:` 對應的位置就能知道你電腦安裝 gem 的位置在哪裡唷！裝好這 gem 後你就能輕鬆的在你的 Ruby 檔案中 `require "activerecord_0216002"` 啦！

---
## 結論

在學習 Rails 的過程中我們會好奇各種 Rails 的魔法是怎麼做出來的？這時候去看了下 Rails 的原始碼才發現裡面博大精深我真的是看不懂。後來隨著對 Ruby 有越來越多的認識後，再加上網路上找到的一些資料，總算能做出一個超簡單 ActiveRecord ，雖然東西是做出來了，但能進步的空間還很大！希望未來在寫 Rails 時候遇到問題後我除了可以上網查外，還可以因為對 Rails 的本身的了解，從 Rails 的原始法裡找答案，如果這樣可以比較快的話XD
