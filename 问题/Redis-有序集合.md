#  揭秘Redis有序集合底层实现

#  有序集合对象

Redis 有序集合（Sorted Set）是 Redis
中的一种数据结构，它结合了集合和有序列表的特点，既可以存储不重复的元素，又可以为每个元素关联一个分数（score），并按照分数对元素进行排序。

有序集合的编码可以是 ziplist 或者 skiplist 。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53dGETzx9U5WLPb7ZfqM62nYWkjiaxNaG8FibvyFfxagy8UExN8rhicfGTw/640?wx_fmt=png&from=appmsg)

##  ziplist 编码

ziplist 编码的有序集合对象使用压缩列表作为底层实现， 每个集合元素使用两个紧挨在一起的压缩列表节点来保存，
第一个节点保存元素的成员（member）， 而第二个元素则保存元素的分值（score）。


​    
    redis> ZADD price 8.5 apple 5.0 banana 6.0 cherry  
    (integer) 3  


ziplist 编码的有序集合：
![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia539WBUj7GzCKzibq0yTQibGw8GKxzObHSlj6RIicDJcSSb52icoh0ZkbAs2Q/640?wx_fmt=png&from=appmsg)

压缩列表中布局如下：
![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53ppkQ1CK2n2xqNK7TbrAfsktTbvPA9DnbQickxZSOB6ia7ibjojHf8oyuQ/640?wx_fmt=png&from=appmsg)

##  skiplist 编码

skiplist 编码的有序集合对象使用 zset 结构作为底层实现， 一个 zset 结构同时包含一个字典和一个跳跃表：


​    
    typedef struct zset {  
        // 字典，键为成员，值为分值  
        // 用于支持 O(1) 复杂度的按成员取分值操作  
        dict *dict;  
        // 跳跃表，按分值排序成员  
        // 用于支持平均复杂度为 O(log N) 的按分值定位成员操作  
        // 以及范围操作  
        zskiplist *zsl;  
    } zset;  


zset 结构中的 zsl 跳跃表按分值从小到大保存了所有集合元素， 每个跳跃表节点都保存了一个集合元素：跳跃表节点的 object 属性保存了元素的成员，
而跳跃表节点的 score 属性则保存了元素的分值。通过这个跳跃表， 程序可以对有序集合进行范围型操作， 比如 ZRANK 、 ZRANGE
等命令就是基于跳跃表 API 来实现的。

除此之外， zset 结构中的 dict 字典为有序集合创建了一个从成员到分值的映射， 字典中的每个键值对都保存了一个集合元素：字典的键保存了元素的成员，
而字典的值则保存了元素的分值。通过这个字典， 程序可以用 O(1) 复杂度查找给定成员的分值， ZSCORE 命令就是根据这一特性实现的，
而很多其他有序集合命令都在实现的内部用到了这一特性。

有序集合每个元素的成员都是一个字符串对象， 而每个元素的分值都是一个 double 类型的浮点数。值得一提的是， 虽然 zset
结构同时使用跳跃表和字典来保存有序集合元素， 但这两种数据结构都会通过指针来共享相同元素的成员和分值，
所以同时使用跳跃表和字典来保存集合元素不会产生任何重复成员或者分值， 也不会因此而浪费额外的内存。

##  编码的转换

当有序集合对象可以同时满足以下两个条件时， 对象使用 ziplist 编码：

  1. 有序集合保存的元素数量小于 128 个； 
  2. 有序集合保存的所有元素成员的长度都小于 64 字节； 

不能满足以上两个条件的有序集合对象将使用 skiplist 编码。


​    
    # 向有序集合添加一个成员只有三字节长的元素  
    redis> ZADD blah 1.0 www  
    (integer) 1  
    redis> OBJECT ENCODING blah  
    "ziplist"  
    # 向有序集合添加一个成员为 66 字节长的元素  
    redis> ZADD blah 2.0 oooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo  
    (integer) 1  
    # 编码已改变  
    redis> OBJECT ENCODING blah  
    "skiplist"  


#  跳跃表

跳跃表（skiplist）是一种有序数据结构， 它通过在每个节点中维持多个指向其他节点的指针， 从而达到快速访问节点的目的。跳跃表支持平均 O(log N)
最坏 O(N) 复杂度的节点查找， 还可以通过顺序性操作来批量处理节点。

在大部分情况下， 跳跃表的效率可以和平衡树相媲美， 并且因为跳跃表的实现比平衡树要来得更为简单， 所以有不少程序都使用跳跃表来代替平衡树。

Redis 使用跳跃表作为有序集合键的底层实现之一：如果一个有序集合包含的元素数量比较多， 又或者有序集合中元素的成员（member）是比较长的字符串时，
Redis 就会使用跳跃表来作为有序集合键的底层实现。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53wLX1X67tbt2OzCW8cgao1QSEXNPSyWcoGBZ8b0DRmqb0Mic1LsicMN5w/640?wx_fmt=png&from=appmsg)

##  跳跃表结构


​    
    /*  
     * 跳跃表  
     */  
    typedef struct zskiplist {  
        // 表头节点和表尾节点  
        struct zskiplistNode *header, *tail;  
        // 表中节点的数量  
        unsigned long length;  
        // 表中层数最大的节点的层数  
        int level;  
    } zskiplist;  


  * header 和 tail 指针分别指向跳跃表的表头和表尾节点， 通过这两个指针， 程序定位表头节点和表尾节点的复杂度为 O(1) 。 
  * length 属性来记录节点的数量， 程序可以在 O(1) 复杂度内返回跳跃表的长度。 
  * level 属性则用于在 O(1) 复杂度内获取跳跃表中层高最大的那个节点的层数量， 注意表头节点的层高并不计算在内。 

##  跳跃表节点


​    
    /* ZSETs use a specialized version of Skiplists */  
    /*  
     * 跳跃表节点  
     */  
    typedef struct zskiplistNode {  
        // 成员对象  
        robj *obj;  
        // 分值  
        double score;  
        // 后退指针  
        struct zskiplistNode *backward;  
        // 层  
        struct zskiplistLevel {  
            // 前进指针  
            struct zskiplistNode *forward;  
            // 跨度  
            unsigned int span;  
        } level[];  
    } zskiplistNode;  


###  分值和成员

节点的分值（score 属性）是一个 double 类型的浮点数， 跳跃表中的所有节点都按分值从小到大来排序。节点的成员对象（obj 属性）是一个指针，
它指向一个字符串对象， 而字符串对象则保存着一个 SDS 值。在同一个跳跃表中， 各个节点保存的成员对象必须是唯一的，
但是多个节点保存的分值却可以是相同的。

###  后退指针

节点的后退指针（backward 属性）用于从表尾向表头方向访问节点：跟可以一次跳过多个节点的前进指针不同， 因为每个节点只有一个后退指针，
所以每次只能后退至前一个节点。

虚线展示了如果从表尾向表头遍历跳跃表中的所有节点

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53u6Pvw1c76SZG8fPBf5y7icqntc9PxAfNwPReJWoUtiaVcTh1Ia4CtkVg/640?wx_fmt=png&from=appmsg)
从表尾向表头方向遍历跳表

###  层

跳跃表节点的 level 数组可以包含多个元素， 每个元素都包含一个指向其他节点的指针， 程序可以通过这些层来加快访问其他节点的速度， 一般来说，
层的数量越多， 访问其他节点的速度就越快。

每次创建一个新跳跃表节点的时候， 程序都根据幂次定律 （power law，越大的数出现的概率越小） 随机生成一个介于 1 和 32 之间的值作为
level 数组的大小， 这个大小就是层的“高度”。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53WHF6vxYOrgbXEJWxn5NAicUlZ6Enk9yicOxX7RmxiaXyE4HwYOCicVYLIw/640?wx_fmt=png&from=appmsg)
具有不同层数的节点

###  前进指针

每个层都有一个指向表尾方向的前进指针（level[i].forward 属性）， 用于从表头向表尾方向访问节点。

遍历跳跃表中所有节点的路径：

  1. 迭代程序首先访问跳跃表的第一个节点（表头）， 然后从第四层的前进指针移动到表中的第二个节点。 
  2. 在第二个节点时， 程序沿着第二层的前进指针移动到表中的第三个节点。 
  3. 在第三个节点时， 程序同样沿着第二层的前进指针移动到表中的第四个节点。 
  4. 当程序再次沿着第四个节点的前进指针移动时， 它碰到一个 NULL ， 程序知道这时已经到达了跳跃表的表尾， 于是结束这次遍历。 

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53xAyL3sKGo1JoaLxvSwsh95uRTgqI0BH7y1WPegrMFGuxhXZibhzlhgg/640?wx_fmt=png&from=appmsg)
遍历整个跳表

###  跨度

层的跨度（level[i].span 属性）用于记录两个节点之间的距离：

  1. 两个节点之间的跨度越大， 它们相距得就越远。 
  2. 指向 NULL 的所有前进指针的跨度都为 0 ， 因为它们没有连向任何节点。 

初看上去， 很容易以为跨度和遍历操作有关， 但实际上并不是这样 —— 遍历操作只使用前进指针就可以完成了，
跨度实际上是用来计算排位（rank）的：在查找某个节点的过程中， 将沿途访问过的所有层的跨度累计起来， 得到的结果就是目标节点在跳跃表中的排位。

在跳跃表中查找分值为 2.0 、 成员对象为 o2 的节点时， 沿途经历的层：在查找节点的过程中， 程序经过了两个跨度为 1 的节点， 因此可以计算出，
目标节点在跳跃表中的排位为 2 。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53riah19qSCVxlGqKF9GQXJxLD7yfibHn5MXlwU9o8D10knXiaiahdBBeuNg/640?wx_fmt=png&from=appmsg)

简化跳表示意图

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53vozBOgkpxANAKYI1h79ST5X1tVdP3tOOg4mdXpQCZl2l4duIUgbvcg/640?wx_fmt=png&from=appmsg)

#  跳表源码

##  创建跳表


​    
    /*  
     * 创建并返回一个新的跳跃表  
     *  
     * T = O(1)  
     */  
    zskiplist *zslCreate(void) {  
        int j;  
        zskiplist *zsl;  
      
        // 分配空间  
        zsl = zmalloc(sizeof(*zsl));  
      
        // 设置高度和起始层数  
        zsl->level = 1;  
        zsl->length = 0;  
      
        // 初始化表头节点  
        // T = O(1)  
        zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);  
        for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {  
            zsl->header->level[j].forward = NULL;  
            zsl->header->level[j].span = 0;  
        }  
        zsl->header->backward = NULL;  
      
        // 设置表尾  
        zsl->tail = NULL;  
      
        return zsl;  
    }  


##  创建节点


​    
    /*  
     * 创建一个层数为 level 的跳跃表节点，  
     * 并将节点的成员对象设置为 obj ，分值设置为 score 。  
     *  
     * 返回值为新创建的跳跃表节点  
     *  
     * T = O(1)  
     */  
    zskiplistNode *zslCreateNode(int level, double score, robj *obj) {  
          
        // 分配空间  
        zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));  
      
        // 设置属性  
        zn->score = score;  
        zn->obj = obj;  
      
        return zn;  
    }  


##  插入节点


​    
    /*  
     * 创建一个成员为 obj ，分值为 score 的新节点，  
     * 并将这个新节点插入到跳跃表 zsl 中。  
     *   
     * 函数的返回值为新节点。  
     *  
     * T_wrost = O(N^2), T_avg = O(N log N)  
     */  
    zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {  
        zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;  
        unsigned int rank[ZSKIPLIST_MAXLEVEL];  
        int i, level;  
      
        redisAssert(!isnan(score));  
      
        // 在各个层查找节点的插入位置  
        // T_wrost = O(N^2), T_avg = O(N log N)  
        x = zsl->header;  
        for (i = zsl->level-1; i >= 0; i--) {  
      
            /* store rank that is crossed to reach the insert position */  
            // 如果 i 不是 zsl->level-1 层  
            // 那么 i 层的起始 rank 值为 i+1 层的 rank 值  
            // 各个层的 rank 值一层层累积  
            // 最终 rank[0] 的值加一就是新节点的前置节点的排位  
            // rank[0] 会在后面成为计算 span 值和 rank 值的基础  
            rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];  
      
            // 沿着前进指针遍历跳跃表  
            // T_wrost = O(N^2), T_avg = O(N log N)  
            while (x->level[i].forward &&  
                (x->level[i].forward->score < score ||  
                    // 比对分值  
                    (x->level[i].forward->score == score &&  
                    // 比对成员， T = O(N)  
                    compareStringObjects(x->level[i].forward->obj,obj) < 0))) {  
      
                // 记录沿途跨越了多少个节点  
                rank[i] += x->level[i].span;  
      
                // 移动至下一指针  
                x = x->level[i].forward;  
            }  
            // 记录将要和新节点相连接的节点  
            update[i] = x;  
        }  
      
        /* we assume the key is not already inside, since we allow duplicated  
         * scores, and the re-insertion of score and redis object should never  
         * happen since the caller of zslInsert() should test in the hash table  
         * if the element is already inside or not.   
         *  
         * zslInsert() 的调用者会确保同分值且同成员的元素不会出现，  
         * 所以这里不需要进一步进行检查，可以直接创建新元素。  
         */  
      
        // 获取一个随机值作为新节点的层数  
        // T = O(N)  
        level = zslRandomLevel();  
      
        // 如果新节点的层数比表中其他节点的层数都要大  
        // 那么初始化表头节点中未使用的层，并将它们记录到 update 数组中  
        // 将来也指向新节点  
        if (level > zsl->level) {  
      
            // 初始化未使用层  
            // T = O(1)  
            for (i = zsl->level; i < level; i++) {  
                rank[i] = 0;  
                update[i] = zsl->header;  
                update[i]->level[i].span = zsl->length;  
            }  
      
            // 更新表中节点最大层数  
            zsl->level = level;  
        }  
      
        // 创建新节点  
        x = zslCreateNode(level,score,obj);  
      
        // 将前面记录的指针指向新节点，并做相应的设置  
        // T = O(1)  
        for (i = 0; i < level; i++) {  
              
            // 设置新节点的 forward 指针  
            x->level[i].forward = update[i]->level[i].forward;  
              
            // 将沿途记录的各个节点的 forward 指针指向新节点  
            update[i]->level[i].forward = x;  
      
            /* update span covered by update[i] as x is inserted here */  
            // 计算新节点跨越的节点数量  
            x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);  
      
            // 更新新节点插入之后，沿途节点的 span 值  
            // 其中的 +1 计算的是新节点  
            update[i]->level[i].span = (rank[0] - rank[i]) + 1;  
        }  
      
        /* increment span for untouched levels */  
        // 未接触的节点的 span 值也需要增一，这些节点直接从表头指向新节点  
        // T = O(1)  
        for (i = level; i < zsl->level; i++) {  
            update[i]->level[i].span++;  
        }  
      
        // 设置新节点的后退指针  
        x->backward = (update[0] == zsl->header) ? NULL : update[0];  
        if (x->level[0].forward)  
            x->level[0].forward->backward = x;  
        else  
            zsl->tail = x;  
      
        // 跳跃表的节点计数增一  
        zsl->length++;  
      
        return x;  
    }  


##  删除节点


​    
    /* Delete all the elements with score between min and max from the skiplist.  
     *  
     * 删除所有分值在给定范围之内的节点。  
     *  
     * Min and max are inclusive, so a score >= min || score <= max is deleted.  
     *   
     * min 和 max 参数都是包含在范围之内的，所以分值 >= min 或 <= max 的节点都会被删除。  
     *  
     * Note that this function takes the reference to the hash table view of the  
     * sorted set, in order to remove the elements from the hash table too.  
     *  
     * 节点不仅会从跳跃表中删除，而且会从相应的字典中删除。  
     *  
     * 返回值为被删除节点的数量  
     *  
     * T = O(N)  
     */  
    unsigned long zslDeleteRangeByScore(zskiplist *zsl, zrangespec *range, dict *dict) {  
        zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;  
        unsigned long removed = 0;  
        int i;  
      
        // 记录所有和被删除节点（们）有关的节点  
        // T_wrost = O(N) , T_avg = O(log N)  
        x = zsl->header;  
        for (i = zsl->level-1; i >= 0; i--) {  
            while (x->level[i].forward && (range->minex ?  
                x->level[i].forward->score <= range->min :  
                x->level[i].forward->score < range->min))  
                    x = x->level[i].forward;  
            update[i] = x;  
        }  
      
        /* Current node is the last with score < or <= min. */  
        // 定位到给定范围开始的第一个节点  
        x = x->level[0].forward;  
      
        /* Delete nodes while in range. */  
        // 删除范围中的所有节点  
        // T = O(N)  
        while (x &&  
               (range->maxex ? x->score < range->max : x->score <= range->max))  
        {  
            // 记录下个节点的指针  
            zskiplistNode *next = x->level[0].forward;  
            zslDeleteNode(zsl,x,update);  
            dictDelete(dict,x->obj);  
            zslFreeNode(x);  
            removed++;  
            x = next;  
        }  
        return removed;  
    }  
      
    /* Internal function used by zslDelete, zslDeleteByScore and zslDeleteByRank   
     *   
     * 内部删除函数，  
     * 被 zslDelete 、 zslDeleteRangeByScore 和 zslDeleteByRank 等函数调用。  
     *  
     * T = O(1)  
     */  
    void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {  
        int i;  
      
        // 更新所有和被删除节点 x 有关的节点的指针，解除它们之间的关系  
        // T = O(1)  
        for (i = 0; i < zsl->level; i++) {  
            if (update[i]->level[i].forward == x) {  
                update[i]->level[i].span += x->level[i].span - 1;  
                update[i]->level[i].forward = x->level[i].forward;  
            } else {  
                update[i]->level[i].span -= 1;  
            }  
        }  
      
        // 更新被删除节点 x 的前进和后退指针  
        if (x->level[0].forward) {  
            x->level[0].forward->backward = x->backward;  
        } else {  
            zsl->tail = x->backward;  
        }  
      
        // 更新跳跃表最大层数（只在被删除节点是跳跃表中最高的节点时才执行）  
        // T = O(1)  
        while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)  
            zsl->level--;  
      
        // 跳跃表节点计数器减一  
        zsl->length--;  
    }  




“关注、分享、点赞、在看”支持一波 ![](https://res.wx.qq.com/t/wx_fed/we-
emoji/res/v1.3.10/assets/Expression/Expression_80@2x.png)

预览时标签不可点

微信扫一扫  
关注该公众号





****



****



×  分析

  收藏

