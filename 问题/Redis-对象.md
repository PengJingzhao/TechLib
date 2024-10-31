![cover_image](https://mmbiz.qpic.cn/mmbiz_jpg/2kOTFMdShwvnhI3MGfiaMbVMANrztVJtr0V3qMV00YHp6IQNibbxlrpGJAW4q7ocUrr3hxH41kiaSAiaiagAYakEMnQ/0?wx_fmt=jpeg)

#  揭秘Redis对象底层实现

原创  简单的土拨鼠  [ 简单的土拨鼠 ](javascript:void\(0\);)

__ _ _ _ _

#

  

“关注、分享、点赞、在看”支持一波 ![](https://res.wx.qq.com/t/wx_fed/we-
emoji/res/v1.3.10/assets/Expression/Expression_80@2x.png)

对象的类型  

##  Redis 对象

Redis 使用对象来表示数据库中的键和值， 每次当我们在 Redis 的数据库中新创建一个键值对时， 我们至少会创建两个对象，
一个对象用作键值对的键（键对象）， 另一个对象用作键值对的值（值对象）。

Redis 中的每个对象都由一个 redisObject 结构表示， 该结构中和保存数据有关的三个属性分别是 type 属性、 encoding 属性和
ptr 属性：

    
    
    typedef struct redisObject {  
        // 类型  
        unsigned type:4;  
        // 编码  
        unsigned encoding:4;  
        // 对象最后一次被访问的时间  
        unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */  
        // 引用计数  
        int refcount;  
        // 指向实际值的指针  
        void *ptr;  
    } robj;  
                    
    

##  Redis 类型

###  **类型枚举**

对象的 type 属性记录了对象的类型

    
    
    // 对象类型  
    #define REDIS_STRING 0  
    #define REDIS_LIST 1  
    #define REDIS_SET 2  
    #define REDIS_ZSET 3  
    #define REDIS_HASH 4  
                    
    

对于 Redis 数据库保存的键值对来说， 键总是一个字符串对象， 而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种， 因此：

  * 当我们称呼一个数据库键为“字符串键”时， 我们指的是“这个数据库键所对应的值为字符串对象”； 

  * 当我们称呼一个键为“列表键”时， 我们指的是“这个数据库键所对应的值为列表对象”， 

###  TYPE 命令

当我们对一个数据库键执行 TYPE 命令时， 命令返回的结果为数据库键对应的值对象的类型， 而不是键对象的类型：

    
    
    # 键为字符串对象，值为字符串对象  
    redis> SET msg "hello world"  
    OK  
    redis> TYPE msg  
    string  
      
    # 键为字符串对象，值为列表对象  
    redis> RPUSH numbers 1 3 5  
    (integer) 6  
    redis> TYPE numbers  
    list  
                    
    

不同类型值对象的 TYPE 命令输出：

对象  |  对象 type 属性的值  |  TYPE 命令的输出   
---|---|---  
字符串对象  |  REDIS_STRING  |  "string"   
列表对象  |  REDIS_LIST  |  "list"   
哈希对象  |  REDIS_HASH  |  "hash"   
集合对象  |  REDIS_SET  |  "set"   
有序集合对象  |  REDIS_ZSET  |  "zset"   
  
#  对象的 **编码**

##  对象编码

对象的 ptr 指针指向对象的底层实现数据结构， 而这些数据结构由对象的 encoding 属性决定。

    
    
    // 对象编码  
    #define REDIS_ENCODING_RAW 0     /* 简单动态字符串 */  
    #define REDIS_ENCODING_INT 1     /* long 类型的整数 */  
    #define REDIS_ENCODING_HT 2      /* 字典 */  
    #define REDIS_ENCODING_ZIPMAP 3  /* 压缩列表 */  
    #define REDIS_ENCODING_LINKEDLIST 4 /* 双端链表 */  
    #define REDIS_ENCODING_ZIPLIST 5 /* 压缩列表 */  
    #define REDIS_ENCODING_INTSET 6  /* 整数集合 */  
    #define REDIS_ENCODING_SKIPLIST 7  /* 跳跃表和字典 */  
    #define REDIS_ENCODING_EMBSTR 8  /* embstr 编码的简单动态字符串 */  
    

##  对象类型与编码的对应关系

每种类型的对象都至少使用了两种不同的编码：

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwvnhI3MGfiaMbVMANrztVJtrsttFAibrLshFbvicx8rEoo4VPwBt6Z2RfLPUFNAaOvJOAxjh7mEUzTFg/640?wx_fmt=png&from=appmsg)

##  OBJECT ENCODING 命令

使用 OBJECT ENCODING 命令可以查看一个数据库键的值对象的编码：

    
    
    redis> SET msg "hello wrold"  
    OK  
    redis> OBJECT ENCODING msg  
    "embstr"  
    redis> SET story "long long long long long long ago ..."  
    OK  
    redis> OBJECT ENCODING story  
    "raw"  
    redis> SADD numbers 1 3 5  
    (integer) 3  
    redis> OBJECT ENCODING numbers  
    "intset"  
                    
    

OBJECT ENCODING 对不同编码的输出：

对象所使用的底层数据结构  |  编码常量  |  OBJECT ENCODING 命令输出   
---|---|---  
整数  |  REDIS_ENCODING_INT  |  "int"   
embstr 编码的简单动态字符串  |  REDIS_ENCODING_EMBSTR  |  "embstr"   
简单动态字符串  |  REDIS_ENCODING_RAW  |  "raw"   
字典  |  REDIS_ENCODING_HT  |  "hashtable"   
双端链表  |  REDIS_ENCODING_LINKEDLIST  |  "linkedlist"   
压缩列表  |  REDIS_ENCODING_ZIPLIST  |  "ziplist"   
整数集合  |  REDIS_ENCODING_INTSET  |  "intset"   
跳跃表和字典  |  REDIS_ENCODING_SKIPLIST  |  "skiplist"   
  
#  **类型检查与命令多态**

Redis 中用于操作键的命令基本上可以分为两种类型：

  * 其中一种命令可以对任何类型的键执行， 比如说 DEL 命令、 EXPIRE 命令、 RENAME 命令、 TYPE 命令、 OBJECT 命令， 等等。 

  * 而另一种命令只能对特定类型的键执行， 比如说： 

命令和对象类型不匹配时，Redis 将向我们返回一个类型错误

    
    
    redis> SET msg "hello world"  
    OK  
    redis> LLEN msg  
    (error) WRONGTYPE Operation against a key holding the wrong kind of value  
                    
    

##  类型检查的实现

在执行一个类型特定的命令之前， Redis 会先检查输入键的类型是否正确， 然后再决定是否执行给定的命令。

类型特定命令所进行的类型检查是通过 redisObject 结构的 type 属性来实现的：

  * 在执行一个类型特定命令之前， 服务器会先检查输入数据库键的值对象是否为执行命令所需的类型， 如果是的话， 服务器就对键执行指定的命令； 

  * 否则， 服务器将拒绝执行命令， 并向客户端返回一个类型错误。 

以 LLEN 命令为例：

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwvnhI3MGfiaMbVMANrztVJtrQFLwLzBvdibrOROI7AnQ2e3Ykib9LRLw8dyAckMssbDia6eMmTCBibeVxw/640?wx_fmt=png&from=appmsg)

##  多态命令的实现

Redis 除了会根据值对象的类型来判断键是否能够执行指定命令之外， 还会根据值对象的编码方式， 选择正确的命令实现代码来执行命令。

同样以 LLEN 命令为例：

  * 如果列表对象的编码为 ziplist ， 那么说明列表对象的实现为压缩列表， 程序将使用 ziplistLen 函数来返回列表的长度； 

  * 如果列表对象的编码为 linkedlist ， 那么说明列表对象的实现为双端链表， 程序将使用 listLength 函数来返回双端链表的长度； 

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwvnhI3MGfiaMbVMANrztVJtrAyqCOBjefia9BrP41BjzdlV3U1WvCZvTvVUIysJGIGOrToYDDe8F6uw/640?wx_fmt=png&from=appmsg)

#  对象的其他属性

##  引用计数

因为 C 语言并不具备自动的内存回收功能， 所以 Redis 在自己的对象系统中构建了一个引用计数（reference
counting）技术实现的内存回收机制， 通过这一机制， 程序可以通过跟踪对象的引用计数信息， 在适当的时候自动释放对象并进行内存回收。

    
    
    typedef struct redisObject {  
        // ...  
        // 引用计数  
        int refcount;  
        // ...  
    } robj;  
                    
    

对象的引用计数信息会随着对象的使用状态而不断变化：

  * 在创建一个新对象时， 引用计数的值会被初始化为 1 ； 

  * 当对象被一个新程序使用时， 它的引用计数值会被增一； 

  * 当对象不再被一个程序使用时， 它的引用计数值会被减一； 

  * 当对象的引用计数值变为 0 时， 对象所占用的内存会被释放。 

Redis 封装了一系列函数用来操作应用计数：

函数  |  作用   
---|---  
incrRefCount  |  将对象的引用计数值增一。   
decrRefCount  |  将对象的引用计数值减一， 当对象的引用计数值等于 0 时， 释放对象。   
resetRefCount  |  将对象的引用计数值设置为 0 ， 但并不释放对象， 这个函数通常在需要重新设置对象的引用计数值时使用。   
  
下面是 RANDOMKEY 命令底层实现代码，当KEY过期了就会调用decrRefCount将引用计数减一，然后重新再找一个KEY。

    
    
     robj *dbRandomKey(redisDb *db) {  
        dictEntry *de;  
      
        while(1) {  
            sds key;  
            robj *keyobj;  
      
            // 从键空间中随机取出一个键节点  
            de = dictGetRandomKey(db->dict);  
      
            // 数据库为空  
            if (de == NULL) return NULL;  
      
            // 取出键  
            key = dictGetKey(de);  
            // 为键创建一个字符串对象，对象的值为键的名字  
            keyobj = createStringObject(key,sdslen(key));  
            // 检查键是否带有过期时间  
            if (dictFind(db->expires,key)) {  
                // 如果键已经过期，那么将它删除，并继续随机下个键  
                if (expireIfNeeded(db,keyobj)) {  
                    decrRefCount(keyobj); // 引用计数减一  
                    continue; /* search for another key. This expired. */  
                }  
            }  
      
            // 返回被随机到的键（的名字）  
            return keyobj;  
        }  
    }  
    

##  对象共享

Redis 会在初始化服务器时， 创建一万个字符串对象， 这些对象包含了从 0 到 9999 的所有整数值， 当服务器需要用到值为 0 到 9999
的字符串对象时， 服务器就会使用这些共享对象， 而不是新创建对象。

    
    
    redis> SET A 100  
    OK  
    redis> OBJECT REFCOUNT A  
    (integer) 2  
    redis> SET B 100  
    OK  
    redis> OBJECT REFCOUNT A  
    (integer) 3  
    redis> OBJECT REFCOUNT B  
    (integer) 3  
                    
    

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwvnhI3MGfiaMbVMANrztVJtrPsgib3lPHnB93kwm8aaPKaibvD4wZZbbewNT5DNKfibcgdyyj5AlwrUCg/640?wx_fmt=png&from=appmsg)

> **为什么 Redis 不共享包含字符串的对象？**
>
> 当服务器考虑将一个共享对象设置为键的值对象时， 程序需要先检查给定的共享对象和键想创建的目标对象是否完全相同，
> 只有在共享对象和目标对象完全相同的情况下， 程序才会将共享对象用作键的值对象， 而一个共享对象保存的值越复杂，
> 验证共享对象和目标对象是否相同所需的复杂度就会越高， 消耗的 CPU 时间也会越多：
>
> 1\. 如果共享对象是保存整数值的字符串对象， 那么验证操作的复杂度为 O(1) ；
>
> 2\. 如果共享对象是保存字符串值的字符串对象， 那么验证操作的复杂度为 O(N) ；
>
> 3\. 如果共享对象是包含了多个值（或者对象的）对象， 比如列表对象或者哈希对象， 那么验证操作的复杂度将会是 O(N^2) 。
>
> 因此， 尽管共享更复杂的对象可以节约更多的内存， 但受到 CPU 时间的限制， Redis 只对包含整数值的字符串对象进行共享。

##  对象的空转时长

redisObject 结构包含的最后一个属性为 lru 属性， 该属性记录了对象最后一次被命令程序访问的时间。

OBJECT IDLETIME 命令可以打印出给定键的空转时长， 这一空转时长就是通过将当前时间减去键的值对象的 lru 时间计算得出的：

    
    
    redis> SET msg "hello world"  
    OK  
    # 等待一小段时间  
    redis> OBJECT IDLETIME msg  
    (integer) 20  
    # 等待一阵子  
    redis> OBJECT IDLETIME msg  
    (integer) 180  
    # 访问 msg 键的值  
    redis> GET msg  
    "hello world"  
    # 键处于活跃状态，空转时长为 0  
    redis> OBJECT IDLETIME msg  
    (integer) 0  
                    
    

键的空转时长还有另外一项作用：如果服务器打开了 maxmemory 选项， 并且服务器用于回收内存的算法为 volatile-lru 或者 allkeys-
lru ， 那么当服务器占用的内存数超过了 maxmemory 选项所设置的上限值时， 空转时长较高的那部分键会优先被服务器释放， 从而回收内存。

  

预览时标签不可点

微信扫一扫  
关注该公众号





****



****



×  分析

  收藏

