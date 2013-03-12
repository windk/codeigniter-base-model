codeigniter-base-model
=====================================

[![Build Status](https://secure.travis-ci.org/jamierumbelow/codeigniter-base-model.png?branch=master)](http://travis-ci.org/jamierumbelow/codeigniter-base-model)

这是中文的README.md的翻译.

CodeIgniter Base Model是可以扩展你的应用中的 CI_Model 类的插件. 为你提供基础和完整的CURD, 基于事件观察者的系统, model内的数据验证, 智能数据库表名猜测和软删除(soft delete), 使数据库交互更加简单快捷.


摘要
--------

```php
class Post_model extends MY_Model { }

$this->load->model('post_model', 'post');

$this->post->get_all();

$this->post->get(1);
$this->post->get_by('title', 'Pigs CAN Fly!');
$this->post->get_many_by('status', 'open');

$this->post->insert(array(
    'status' => 'open',
    'title' => "I'm too sexy for my shirt"
));

$this->post->update(1, array( 'status' => 'closed' ));

$this->post->delete(1);
```

安装/使用
------------------
下载MY\_Model.php文件, 放到项目的_application/core_文件夹中.
CodeIgniter会自动载入并初始化.

让你的model类继承`MY_Model`, 所有的方法就能使用了.


命名约定
------------------
继承自MY_Model的类, 会根据类命名自动猜测应该使用的数据库的表名

例如:

    class Post_model extends MY_Model { }

…那么猜出的表名就是 `posts` . 后缀换成`_m`也可以:

    class Book_m extends MY_Model { }

…猜出的表名是 `books`.

如果你需要设置自己的表名, 可以在类中定义 _$\_table_ 属性 初始化为你希望设置的表名:

    class Post_model extends MY_Model
    {
        public $_table = 'blogposts';
    }

有一些 CURD 函数还会假设你的数据表主键的列名为_'id'_. 你可以通过为类增加_$primary\_key_属性覆盖掉默认值:

    class Post_model extends MY_Model
    {
        public $primary_key = 'post_id';
    }

回调/观察者
-------------------
在插入或者返回数据的时候, 你里有很多机会,可以修改model的数据. 
这些情境通常是是增加一个时间戳, 引入一个关系或者删除独立的行.
MVC设计模式中,这样的操作应该在model中进行. 为了简化, **MY_Model** 包含一系列在特定时间调用的方法也叫:回调或者观察者.

完整的观察者列表如下:

* $before_create
* $after_create
* $before_update
* $after_update
* $before_get
* $after_get
* $before_delete
* $after_delete

这些变量实例通常定义在class中, 类包含有在特定时机调用的方法数组, 例如:

```php
class Book_model extends MY_Model
{
    public $before_create = array( 'timestamps' );
    
    protected function timestamps($book)
    {
        $book['created_at'] = $book['updated_at'] = date('Y-m-d H:i:s');
        return $book;
    }
}
```
**记住, 永远永远永远返回你传递的`$row`对象. 每一个观察者都会以他们定义的顺序覆盖调用前的自己**

观察者也可以接受参数, 形式和CodeIgniter的 Form Validation 库很相似. 如果传递了参数, 可以通过`$this->callback_parameters`访问:

    public $before_create = array( 'data_process(name)' );
    public $before_update = array( 'data_process(date)' );

    protected function data_process($row)
    {
        $row[$this->callback_parameters[0]] = $this->_process($row[$this->callback_parameters[0]]);

        return $row;
    }

验证
----------
MY_Model使用CodeIgniter的内置表单验证系统验证数据插入.

你可以通过设置 `$validate` 实例数组, 开启表单验证.
:

    class User_model extends MY_Model
    {
        public $validate = array(
            array( 'field' => 'email', 
                   'label' => 'email',
                   'rules' => 'required|valid_email|is_unique[users.email]' ),
            array( 'field' => 'password',
                   'label' => 'password',
                   'rules' => 'required' ),
            array( 'field' => 'password_confirmation',
                   'label' => 'confirm password',
                   'rules' => 'required|matches[password]' ),
        );
    }
所有表单验证库中可用的在这里都可以用.更多可用的规则, 请 [访问库文档](http://codeigniter.com/user_guide/libraries/form_validation.html#validationrulesasarray).

通过这个规则数组, 每一个调用`insert()` 或者 `update()`都会在执行数据库查询之前验证数据.
**和 CodeIgniter 验证库不同, 这里不会验证 POST 的数据, 只会验证直接传递给他的数据.**
可以用 `skip_validation()` 方法跳过验证:

    $this->user_model->skip_validation();
    $this->user_model->insert(array( 'email' => 'blah' ));

此外还可以给 `insert()` 传递一个 `TRUE` 或者 `FALSE`:

    $this->user_model->insert(array( 'email' => 'blah' ), TRUE);

这个就是控制是否调用`validate()`.

Protected 属性
--------------------
如果你像我一样懒, 你可以直接将<from>表单里的数据直接扔进model里. 
一些陷阱可以通过validatoin避免, 这样做对于输入数据来说十分危险; model中的任何属性都可以被修改.
为了防止发生悲剧, MY_Model支持protected属性. 这样的属性无法被更改

可以为 `$protected_attributes` 赋值, 声明被保护的属性:

    class Post_model extends MY_Model
    {
        public $protected_attributes = array( 'id', 'hash' );
    }

现在, 当调用`insert` 或者 `update` 的时候, 属性会被自动移除, 实现保护:

    $this->post_model->insert(array(
        'id' => 2,
        'hash' => 'aqe3fwrga23fw243fWE',
        'title' => 'A new post'
    ));

    // SQL: INSERT INTO posts (title) VALUES ('A new post')

关系
-------------

**MY\_Model** 现在支持 基本的 _belongs\_to_ 和 \_many 关系. 这些关系很容易定义:

    class Post_model extends MY_Model
    {
        public $belongs_to = array( 'author' );
        public $has_many = array( 'comments' );
    }
上例会假设: 和它有单一关系的, 一个继承自MY_Model的model已经定义. 默认情况下, 就是 `relationship_model`. 那么, 上面的例子就会引用其他两个model:

    class Author_model extends MY_Model { }
    class Comment_model extends MY_Model { }

如果你想自定义, 你可以传递model名称作为参数实现:

    class Post_model extends MY_Model
    {
        public $belongs_to = array( 'author' => array( 'model' => 'author_m' ) );
        public $has_many = array( 'comments' => array( 'model' => 'model_comments' ) );
    }

然后就可以使用`with()`方法访问相关数据:

    $post = $this->post_model->with('author')
                             ->with('comments')
                             ->get(1);

相关数据会从被嵌入`get`返回值中:

    echo $post->author->name;

    foreach ($post->comments as $comment)
    {
        echo $message;
    }
查询(queries)会分别执行去获取数据, 如果性能很重要, 那么建议使用独立的JOIN 和 SELECT调用.
你也可以设置主键. 对于 _belongs\_to_ 调用, 关联字段在当前的对象上而不是其他对象, 伪代码:

The primary key can also be configured. For _belongs\_to_ calls, the related key is on the current object, not the foreign one. Pseudocode:

    SELECT * FROM authors WHERE id = $post->author_id

...而对于一个 _has\_many_ 请求则是这样的:

    SELECT * FROM comments WHERE post_id = $post->id

为了改变主键,可以配置的时候可以使用`primary_key`:

    class Post_model extends MY_Model
    {
        public $belongs_to = array( 'author' => array( 'primary_key' => 'post_author_id' ) );
        public $has_many = array( 'comments' => array( 'primary_key' => 'parent_post_id' ) );
    }

数组 vs 对象
-----------------
默认情况下, MY_Model被设置为返回CodeIgniter的QB的`row()`和`result()`方法返回的对象. 如果你想用数组返回值, 这里有几个实现方法:
如果希望所有的请求都是用数组方法, 可以设置`$return_type`的值为`array`:
    class Book_model extends MY_Model
    {
        protected $return_type = 'array';
    }
如果只是希望明确指示 _下一个_ 请求的返回类型, 可以这么做:

    $this->book_model->as_array()
                     ->get(1);
    $this->book_model->as_object()
                     ->get_by('column', 'value');

软删除
-----------
默认情况下, 删除的原理是使用SQL的 `DELETE` 语句. 然而你可能不想破坏数据, 你可能想执行一个'软删除'.

如果启动软删除, 删除的行会被标记为 `deleted` 而不是真的从数据库删除.
举个栗子, 一个 `Book_model` :

    class Book_model extends MY_Model { }

通过设置 `$this->soft_delete` 属性启动软删除:

    class Book_model extends MY_Model
    { 
        protected $soft_delete = TRUE;
    }
默认情况下, MY_Model回去找一个叫做`deleted`, `TINYINT`或者`INT` 的列.
如果需要修改, 可以更改`$soft_delete_key`的值:

    class Book_model extends MY_Model
    { 
        protected $soft_delete = TRUE;
        protected $soft_delete_key = 'book_deleted_status';
    }

现在, 当你使用任何`get_`方法的时候, 需要加上一个非删除限定:

    => $this->book_model->get_by('user_id', 1);
    -> SELECT * FROM books WHERE user_id = 1 AND deleted = 0
如果想包括删除的列, 可以使用 `with_deleted()`:

    => $this->book_model->with_deleted()->get_by('user_id', 1);
    -> SELECT * FROM books WHERE user_id = 1

内置的观察者
-------------------

**MY_Model** contains a few built-in observers for things I've found I've added to most of my models.

The timestamps (MySQL compatible) `created_at` and `updated_at` are now available as built-in observers:

    class Post_model extends MY_Model
    {
        public $before_create = array( 'created_at', 'updated_at' );
        public $before_update = array( 'updated_at' );
    }

**MY_Model** also contains serialisation observers for serialising and unserialising native PHP objects. This allows you to pass complex structures like arrays and objects into rows and have it be serialised automatically in the background. Call the `serialize` and `unserialize` observers with the column name(s) as a parameter:

    class Event_model extends MY_Model
    {
        public $before_create = array( 'serialize(seat_types)' );
        public $before_update = array( 'serialize(seat_types)' );
        public $after_get = array( 'unserialize(seat_types)' );
    }

单元测试
----------
MY_Model 包含一个健壮的单元测试集合以保证系统正常运行.
使用 Composer 安装测试框架(PHPUnit):

    $ curl -s https://getcomposer.org/installer | php
    $ php composer.phar install

可以使用`vendro/bin/phpunit` 程序执行测试:

    $ vendor/bin/phpunit tests/MY_Model_test.php


为 MY_Model 贡献代码
------------------------
如果你发现bug或者想增加新的特性到MY_Model, 那太好了! 为了让我更快更容易的验证和合并更改, 你要是能按下面的步骤来贡献代码, 我将不胜感激:

1. Fork 这个项目
2. **建立一个新的分支. `git checkout -b name_of_new_feature_or_bug`**
3. 增加功能, 或者修bug.
4. **为更改增加测试. 这太重要了我以后也会一直这么要求.**
5. commit.
6. 给我发一个 pull request!


其他文档
-------------------

* My book, The CodeIgniter Handbook, talks about the techniques used in MY_Model and lots of other interesting useful stuff. [Get a copy now.](https://efendibooks.com/books/codeigniter-handbook/vol-1)
* Jeff Madsen has written an excellent tutorial about the basics (and triggered me updating the documentation here). [Read it now, you lovely people.](http://www.codebyjeff.com/blog/2012/01/using-jamie-rumbelows-my_model)
* Rob Allport wrote a post about MY_Model and his experiences with it. [Check it out!](http://www.web-design-talk.co.uk/493/codeigniter-base-models-rock/)
* I've written a write up of the new 2.0.0 features [over at my blog, Jamie On Software.](http://jamieonsoftware.com/journal/2012/9/11/my_model-200-at-a-glance.html)

Changelog
---------

**Version 2.0.0**
* Added support for soft deletes
* Removed Composer support. Great system, CI makes it difficult to use for MY_ classes
* Fixed up all problems with callbacks and consolidated into single `trigger` method
* Added support for relationships
* Added built-in timestamp observers
* The DB connection can now be manually set with `$this->_db`, rather than relying on the `$active_group`
* Callbacks can also now take parameters when setting in callback array
* Added support for column serialisation
* Added support for protected attributes
* Added a `truncate()` method

**Version 1.3.0**
* Added support for array return types using `$return_type` variable and `as_array()` and `as_object()` methods
* Added PHP5.3 support for the test suite
* Removed the deprecated `MY_Model()` constructor
* Fixed an issue with after_create callbacks (thanks [zbrox](https://github.com/zbrox)!)
* Composer package will now autoload the file
* Fixed the callback example by returning the given/modified data (thanks [druu](https://github.com/druu)!)
* Change order of operations in `_fetch_table()` (thanks [JustinBusschau](https://github.com/JustinBusschau)!)

**Version 1.2.0**
* Bugfix to `update_many()`
* Added getters for table name and skip validation
* Fix to callback functionality (thanks [titosemi](https://github.com/titosemi)!)
* Vastly improved documentation
* Added a `get_next_id()` method (thanks [gbaldera](https://github.com/gbaldera)!)
* Added a set of unit tests
* Added support for [Composer](http://getcomposer.org/)

**Version 1.0.0 - 1.1.0**
* Initial Releases
