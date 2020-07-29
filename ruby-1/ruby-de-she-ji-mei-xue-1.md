# Ruby 的设计美学

> There are many ways to do it.
>
> 最小惊讶原则 \(Matz\)
>
> 璀璨微笑理念 \(DHH\)

### 对象与类的构成

#### 面向对象设计

> 在 ruby 中，一些皆是对象

哪怕是基本数据类型

```text
2.5.1 :001 > 1.class
 => Integer
2.5.1 :002 > true.class
 => TrueClass
2.5.1 :006 > "a".class
 => String
```

甚至是类

```text
2.5.1 :003 > class A
2.5.1 :004?> end
 => nil
2.5.1 :005 > A.class
 => Class
```

简单归结起来，一个 ruby 程序其实就是由一组对象和这组对象彼此间发送的消息组成的

> Ruby是面向对象语言，面向对象最基本的特性之一，就是消息传递。

当然这并不是说，在使用 ruby 时你只能应用面向对象设计 事实上 ruby 借鉴了 lisp 的部分理念，这就是 ruby 中的 block \(闭包\) 及 lambda 设计，函数在 ruby 中是作为一等公民出现的

> Ruby 的 block 本质上和高阶函数是一样的，高阶函数是接受函数作为参数的函数

```text
def rescue_error
  # do some things
  begin
    yield
  rescue => e
  end
end

2.5.1 :008 > rescue_error do
2.5.1 :009 >     puts "I am err"
2.5.1 :010?>   end
I am err
 => nil

# or
succ = lambda { |x| x + 1 }
# ||
succ = ->(x){ x + 1 }
```

#### 深入对象内部

> 每个 Ruby 对象都是类指针和实例变量数组的组合

让我们深入对象内部看看它到底是如何构成的

每个自定义对象都保存在 RObject 的结构体中 \(c struct\)

RObject 中包含了内部 RBasic 结构体和一些特有信息

```text
# include/ruby/ruby.h

struct RUBY_ALIGNAS(SIZEOF_VALUE) RBasic {
    VALUE flags;
    const VALUE klass;
};

struct RObject {
    struct RBasic basic;
    union {
        struct {
            uint32_t numiv;
            VALUE *ivptr;
            void *iv_index_tbl; /* shortcut for RCLASS_IV_INDEX_TBL(rb_obj_class(obj)) */
        } heap;
        VALUE ary[ROBJECT_EMBED_LEN_MAX];
    } as;
};
```

_`open ruby-robject.png`_

_一组 flags 的布尔值，用来存储各种内部专用的值_

所有自定义对象的底层存储结构皆是如此

上面曾说过基础数据类型也是对象，那么它们的底层存储也是如此吗

```text
# include/ruby/ruby.h

struct RString {
    struct RBasic basic;
    union {
        struct {
            long len;
            char *ptr;
            union {
                long capa;
                VALUE shared;
            } aux;
        } heap;
        char ary[RSTRING_EMBED_LEN_MAX + 1];
    } as;
};

struct RArray {
    struct RBasic basic;
    union {
        struct {
            long len;
            union {
                long capa;
                VALUE shared;
            } aux;
            const VALUE *ptr;
        } heap;
        const VALUE ary[RARRAY_EMBED_LEN_MAX];
    } as;
};

struct RRegexp {
    struct RBasic basic;
    struct re_pattern_buffer *ptr;
    const VALUE src;
    unsigned long usecnt;
};

struct RFile {
    struct RBasic basic;
    struct rb_io_t *fptr;
};

struct RData {
    struct RBasic basic;
    void (*dmark)(void*);
    void (*dfree)(void*);
    void *data;
};

...
```

Ruby 中使用一些不一样的结构来保存每个基本数据类型的值

那么简单立即值呢，比如整形，true，false 事实上，这些简单立即值是没有结构体的，它们的值直接存储在

```text
VALUE
```

中，ruby 会在 `VALUE` 的前几个 bits 保存一串比特标记来标明这些值的类，这类 `VALUE` 就不是指针了，而是立即值本身

```text
0100...100|00000001
```

```text
#if USE_FLONUM
    RUBY_Qfalse = 0x00,        /* ...0000 0000 */
    RUBY_Qtrue  = 0x14,        /* ...0001 0100 */
    RUBY_Qnil   = 0x08,        /* ...0000 1000 */
    RUBY_Qundef = 0x34,        /* ...0011 0100 */

    RUBY_IMMEDIATE_MASK = 0x07,
    RUBY_FIXNUM_FLAG    = 0x01,    /* ...xxxx xxx1 */
    RUBY_FLONUM_MASK    = 0x03,
    RUBY_FLONUM_FLAG    = 0x02,    /* ...xxxx xx10 */
    RUBY_SYMBOL_FLAG    = 0x0c,    /* ...0000 1100 */
#else
    RUBY_Qfalse = 0,        /* ...0000 0000 */
    RUBY_Qtrue  = 2,        /* ...0000 0010 */
    RUBY_Qnil   = 4,        /* ...0000 0100 */
    RUBY_Qundef = 6,        /* ...0000 0110 */

    RUBY_IMMEDIATE_MASK = 0x03,
    RUBY_FIXNUM_FLAG    = 0x01,    /* ...xxxx xxx1 */
    RUBY_FLONUM_MASK    = 0x00,    /* any values ANDed with FLONUM_MASK cannot be FLONUM_FLAG */
    RUBY_FLONUM_FLAG    = 0x02,
    RUBY_SYMBOL_FLAG    = 0x0e,    /* ...0000 1110 */
#endif
    RUBY_SPECIAL_SHIFT  = 8
};
```

那么方法呢？

**深入类的内部**

每个对象都是通过保存指向 `RClass` 结构体的指针来标记自身的类 那么 `RClass` 内部又是怎么构成的呢

要搞清楚这个问题，我们需要现在思考一下，类中都保存那些信息。

首先非常容易想到，类中保存有方法定义，实例变量名，常量，以及必不可少的类指针

上面有提到过，RObject 结构体中有存一个 ivptr 的指针指向实例变量值数组，也就是说 RObject 中保存了实例变量的值，但其实并没有保存实例变量的名字，实例变量名实际被保存在对象的类中。

除此之外还有些什么呢？不要忘了类中是可以定义类级的实例变量以及变量的，那么类实例变量和类变量是怎么保存的。

先来看看类实例变量，所谓类实例变量，如果你还没有转过来这个弯的话不妨再回想一下上面反复说道的一句话：ruby 中一切皆对象。这样一来是不是就非常容易理解所谓类实例变量到底是个啥了，类也是对象，是对象就有类指针和实例变量。

其实类级别的实例变量就是在类的上下文中创建的实例变量，如同对象级别的实例变量就是在对象上下文中创建的实例变量一样。

```text
class Myclass
  @variable_alpha

  def self.variable_alpha=(value)
    @variable_alpha = value
  end

  def self.variable_alpha
    @variable_alpha
  end
end

Myclass.variable_alpha
#=> nil
Myclass.variable_alpha = 'go'
Myclass.variable_alpha
#=> "go"
```

如果说类实例变量是把类看做一个对象来在其上下文中创建的实例变量的话，类变量就是把类就看做是类来在其上下文中创建的变量。

_创建类变量时，Ruby 会在该类中创建唯一的值，并在其任意子类中共享该值。 如果是类实例变量，Ruby 会在该类和其子类中创建各自独立使用的值。_

除此之外，在创建类时，Ruby 允许随意指定一个超类来实现单继承。如果没有指定超类，Ruby 会默认指派 Object 类作为超类。这就需要存储一个指向其超类的指针

> Ruby 类就是包含方法定义，属性名称，超类指针和常量表的 Ruby 对象

```text
# include/ruby/ruby.h

struct rb_classext_struct {
    struct st_table *iv_index_tbl;
    struct st_table *iv_tbl;
    struct rb_id_table *const_tbl;
    struct rb_id_table *callable_m_tbl;
    rb_subclass_entry_t *subclasses;
    rb_subclass_entry_t **parent_subclasses;
    /**
     * In the case that this is an `ICLASS`, `module_subclasses` points to the link
     * in the module's `subclasses` list that indicates that the klass has been
     * included. Hopefully that makes sense.
     */
    rb_subclass_entry_t **module_subclasses;
    rb_serial_t class_serial;
    const VALUE origin_;
    VALUE refined_class;
    rb_alloc_func_t allocator;
};

typedef struct rb_classext_struct rb_classext_t;

#undef RClass
struct RClass {
    struct RBasic basic;
    VALUE super;
    rb_classext_t *ptr;
    struct rb_id_table *m_tbl;
};
```

* `m_tbl`: 以方法名为 key，每个方法定义的指针（编译好的 YARV 指令）为 value
* `super`: 指向超类的 RClass 结构体的指针
* `iv_tbl`: 类实例变量和类变量的名字和值
* `allocator`: ruby 内部使用 allocator 为类的每个新的实例分配内存

反复出现的 `st_table` 结构是 ruby 的散列表数据结构 而 `rb_id_table` 其实是 ruby 2.4 时引入的改进，主要用来优化 ID key table 的性能，具体的可以看下面的 feature 描述👇 [https://bugs.ruby-lang.org/issues/11420](https://bugs.ruby-lang.org/issues/11420)

```text
# ruby/id_table.c

struct rb_id_table {
    int capa;
    int num;
    int used;
    item_t *items;
};
```

```text
# include/ruby/st.h

struct st_table {
    /* Cached features of the table -- see st.c for more details.  */
    unsigned char entry_power, bin_power, size_ind;
    /* How many times the table was rebuilt.  */
    unsigned int rebuilds_num;
    const struct st_hash_type *type;
    /* Number of entries currently in the table.  */
    st_index_t num_entries;
    /* Array of bins used for access by keys.  */
    st_index_t *bins;
    /* Start and bound index of entries in array entries.
       entries_starts and entries_bound are in interval
       [0,allocated_entries].  */
    st_index_t entries_start, entries_bound;
    /* Array of size 2^entry_power.  */
    st_table_entry *entries;
};
```

**klass ptr 与 super ptr**

Class 类在理论上是每个 Ruby 类的类

* klass 指针标示该类是哪个类的实例
* super 指针则标示该类的超类

> metaclass & singleton class

首先实例对象的方法是定义在其对应的类中的，那么是不是类的方法将被定义在类的类中呢？

在 ruby 中默认情况下所有类的类都是 Class，显而易见肯定是不能把类方法定义在其中的。实际上当创建新的类的时候，ruby 会创建两个类，类本身以及 metaclass ，然后 ruby 会把新类的 RClass 结构体中的 klass 指针指向 metaclass。而类方法将会被放在类对应的 metaclass 中。

[http://elibinary.com/2017/07/08/XXXI-Ruby-Method-Definition/](http://elibinary.com/2017/07/08/XXXI-Ruby-Method-Definition/)

**常量**

常量以大写字母开头，在当前类的作用域内有效

_类名，module名都是常量_

ps

> A monkey patch is a way for a program to extend or modify supporting system software locally \(affecting only the running instance of the program\).

