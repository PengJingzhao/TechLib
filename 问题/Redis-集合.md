#  Redis集合底层实现

#  集合对象

Redis 集合（Set）是一种无序的、不重复的数据结构， 集合对象的编码可以是 intset 或者 hashtable 。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53aH70wXkh5yhDbLyNgvvAyV1fD0Rf5TnKnTiagoX0SyonJ4LpdibyIQVg/640?wx_fmt=png&from=appmsg)

##  hashtable 编码

hashtable 编码的集合对象使用字典作为底层实现， 字典的每个键都是一个字符串对象， 每个字符串对象包含了一个集合元素， 而字典的值则全部被设置为
NULL 。


​    
​    redis> SADD fruits "apple" "banana" "cherry"  
​    (integer) 3  


![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53c85XfuwMmedZQte8fPxT8XyYFx8BFvZ3aFm7B3bYtgMiaOTlDH7huIA/640?wx_fmt=png&from=appmsg)
hashtable 编码的集合对象

##  intset 编码

intset 编码的集合对象使用整数集合作为底层实现， 集合对象包含的所有元素都被保存在整数集合里面。


​    redis> SADD numbers 1 3 5  
​    (integer) 3  


![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53FtNV1bFqZKOPWHtkBG0vYHcQqSQjziaIlZhXFpKLVIUr7aFKj8ia1hdA/640?wx_fmt=png&from=appmsg)
intset 编码的集合对象

##  编码的转换

当集合对象可以同时满足以下两个条件时， 对象使用 intset 编码：

  1. 集合对象保存的所有元素都是整数值； 
  2. 集合对象保存的元素数量不超过 512 个； 

不能满足这两个条件的集合对象需要使用 hashtable 编码。

#  整数集合

整数集合（intset）是集合键的底层实现之一：当一个集合只包含整数值元素， 并且这个集合的元素数量不多时， Redis
就会使用整数集合作为集合键的底层实现。

整数集合（intset）是 Redis 用于保存整数值的集合抽象数据结构， 它可以保存类型为 int16_t 、 int32_t 或者 int64_t
的整数值， 并且保证集合中不会出现重复元素。

##  整数集合结构


​    typedef struct intset {  
​        // 编码方式  
​        uint32_t encoding;  
​        // 集合包含的元素数量  
​        uint32_t length;  
​        // 保存元素的数组  
​        int8_t contents[];  
​    } intset;  


  1. **contents** 是整数集合的底层实现：整数集合的每个元素都是 contents 数组的一个数组项（item）， 各个项在数组中按值的大小从小到大有序地排列， 并且数组中不包含任何重复项。 

  2. **length** 属性记录了整数集合包含的元素数量， 也即是 contents 数组的长度。 

  3. **encoding** 标识contents数组元素的类型，虽然 intset 结构将 contents 属性声明为 int8_t 类型的数组， 但实际上 contents 数组并不保存任何 int8_t 类型的值 —— contents 数组的真正类型取决于 encoding 属性的值： 
     * 如果 encoding 属性的值为 INTSET_ENC_INT16 ， 那么 contents 就是一个 int16_t 类型的数组， 数组里的每个项都是一个 int16_t 类型的整数值 （最小值为 -32,768 ，最大值为 32,767 ）。 
     * 如果 encoding 属性的值为 INTSET_ENC_INT32 ， 那么 contents 就是一个 int32_t 类型的数组， 数组里的每个项都是一个 int32_t 类型的整数值 （最小值为 -2,147,483,648 ，最大值为 2,147,483,647 ）。 
     * 如果 encoding 属性的值为 INTSET_ENC_INT64 ， 那么 contents 就是一个 int64_t 类型的数组， 数组里的每个项都是一个 int64_t 类型的整数值 （最小值为 -9,223,372,036,854,775,808 ，最大值为 9,223,372,036,854,775,807 ）。 

##  类型升级

每当我们要将一个新元素添加到整数集合里面， 并且新元素的类型比整数集合现有所有元素的类型都要长时， 整数集合需要先进行升级（upgrade），
然后才能将新元素添加到整数集合里面。

###  升级的步骤

升级整数集合并添加新元素共分为三步进行：

  1. 根据新元素的类型， 扩展整数集合底层数组的空间大小， 并为新元素分配空间。 
  2. 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变。 
  3. 将新元素添加到底层数组里面。 

举个例子， 假设现在有一个 INTSET_ENC_INT16 编码的整数集合， 集合中包含三个 int16_t 类型的元素

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia536iccZFvicI3OIGKQicSvT0PwDr68wVOgDZYqEbhuoxw9DaMInlGibs3LYw/640?wx_fmt=png&from=appmsg)
底层数组:

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53Y8ibLCI2xJUuhFE9IvzY23RjPzkyghA1XuW7HFSCiatwRRO6I8h8tuyw/640?wx_fmt=png&from=appmsg)

现在， 假设我们要将类型为 int32_t 的整数值 65535 添加到整数集合里面， 因为 65535 的类型 int32_t
比整数集合当前所有元素的类型都要长， 所以在将 65535 添加到整数集合之前， 程序需要先对整数集合进行升级。

整数集合目前有三个元素， 再加上新元素 65535 ， 整数集合需要分配四个元素的空间， 因为每个 int32_t 整数值需要占用 32 位空间，
所以在空间重分配之后， 底层数组的大小将是 32 * 4 = 128 位。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53t3dm7aLVlMicOWXvxaz1tJdzuxS0BSNKjKxia2hclE9G1h9ZxvR2Hc3g/640?wx_fmt=png&from=appmsg)

将这三个元素转换成 int32_t 类型， 并将转换后的元素放置到正确的位上面， 而且在放置元素的过程中， 需要维持底层数组的有序性质不变。

  1. 对元素3进行类型转换，并保存在适当的位置上： 

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53mOyoxuem4AmAdH4JOgibCxu06C3ObfnuxF3gd448s9q84ZvLTQyqmng/640?wx_fmt=png&from=appmsg)

  2. 对元素2进行类型转换，并保存在适当的位置上： 

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53pn2ib9tEmFFRk2JEVbRjUMIRj24hanDDFwT5ZO2Ts2rqa8icqgR8VbbA/640?wx_fmt=png&from=appmsg)

  3. 对元素1进行类型转换，并保存在适当的位置上： ![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53VhTzYvC6GiaPSl4s4ia3Qjz62znhncdayzKvEkzbrruZYoX2LMnXk3mQ/640?wx_fmt=png&from=appmsg)

  4. 然后添加新元素，新元素最大放到最后面： 

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53RiaFccwwkooP78zdV3k7KtricbR0GEzLkDvxppmhENaZJo5ecicujibDyg/640?wx_fmt=png&from=appmsg)

  5. 最后， 程序将整数集合 encoding 属性的值从 INTSET_ENC_INT16 改为 INTSET_ENC_INT32 ， 并将 length 属性的值从 3 改为 4： 

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53DO5sLIZbVOOqLzQiaEVZ5TJD3qXlhte4wZpacicJfqN055tlG8WfLXng/640?wx_fmt=png&from=appmsg)

###  升级的优点

**提升灵活性** 整数集合可以通过自动升级底层数组来适应新元素， 所以我们可以随意地将 int16_t 、 int32_t 或者 int64_t
类型的整数添加到集合中， 而不必担心出现类型错误， 这种做法非常灵活。

**节约内存** 升级既可以让集合能同时保存三种不同类型的值， 又可以确保升级操作只会在有需要的时候进行， 这可以尽量节省内存。

###  降级

整数集合不支持降级操作， 一旦对数组进行了升级， 编码就会一直保持升级后的状态。

即使我们将集合里唯一一个真正需要使用 int64_t 类型来保存的元素 4294967295 删除了， 整数集合的编码仍然会维持
INTSET_ENC_INT64 ， 底层数组也仍然会是 int64_t 类型的。

原始整数集合：
![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53e5J7uHKRgvdILJjrfyIM3mWAg39zgTPNFaaPueGEolr5KUjstMozDQ/640?wx_fmt=png&from=appmsg)

删除 4294967295 后的整数集合：
![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtOosPVYm2kmibwiaBO5V5ia53icccib34RLfgYf3v3DbELH2f5yEhiaUkSX2THoTLGib2UCGNrO8lSrgTkQ/640?wx_fmt=png&from=appmsg)

#  整数集合部分源码

##  添加元素


​    
​    intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {  
​      
```c
    // 计算编码 value 所需的长度  
    uint8_t valenc = _intsetValueEncoding(value);  
    uint32_t pos;  
  
    // 默认设置插入为成功  
    if (success) *success = 1;  
  
    // 如果 value 的编码比整数集合现在的编码要大  
    // 那么表示 value 必然可以添加到整数集合中  
    // 并且整数集合需要对自身进行升级，才能满足 value 所需的编码  
    if (valenc > intrev32ifbe(is->encoding)) {  
        /* This always succeeds, so we don't need to curry *success. */  
        // T = O(N)  
        return intsetUpgradeAndAdd(is,value);  
    } else {  
        // 运行到这里，表示整数集合现有的编码方式适用于 value  
  
        /* Abort if the value is already present in the set.  
         * This call will populate "pos" with the right position to insert  
         * the value when it cannot be found. */  
        // 在整数集合中查找 value ，看他是否存在：  
        // - 如果存在，那么将 *success 设置为 0 ，并返回未经改动的整数集合  
        // - 如果不存在，那么可以插入 value 的位置将被保存到 pos 指针中  
        //   等待后续程序使用  
        if (intsetSearch(is,value,&pos)) {  
            if (success) *success = 0;  
            return is;  
        }  
  
        // 运行到这里，表示 value 不存在于集合中  
        // 程序需要将 value 添加到整数集合中  
      
        // 为 value 在集合中分配空间  
        is = intsetResize(is,intrev32ifbe(is->length)+1);  
        // 如果新元素不是被添加到底层数组的末尾  
        // 那么需要对现有元素的数据进行移动，空出 pos 上的位置，用于设置新值  
        // 举个例子  
        // 如果数组为：  
        // | x | y | z | ? |  
        //     |<----->|  
        // 而新元素 n 的 pos 为 1 ，那么数组将移动 y 和 z 两个元素  
        // | x | y | y | z |  
        //         |<----->|  
        // 这样就可以将新元素设置到 pos 上了：  
        // | x | n | y | z |  
        // T = O(N)  
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);  
    }  
  
    // 将新值设置到底层数组的指定位置中  
    _intsetSet(is,pos,value);  
  
    // 增一集合元素数量的计数器  
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);  
  
    // 返回添加新元素后的整数集合  
    return is;  
}  
```


##  升级并添加


​    /* Upgrades the intset to a larger encoding and inserts the given integer.   
     *  
         * 根据值 value 所使用的编码方式，对整数集合的编码进行升级，  
         * 并将值 value 添加到升级后的整数集合中。  
             *
             * 返回值：添加新元素之后的整数集合  
                 *
                 * T = O(N)  
                     */  
                    static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {  

```c
    // 当前的编码方式  
    uint8_t curenc = intrev32ifbe(is->encoding);  
  
    // 新值所需的编码方式  
    uint8_t newenc = _intsetValueEncoding(value);  
  
    // 当前集合的元素数量  
    int length = intrev32ifbe(is->length);  
  
    // 根据 value 的值，决定是将它添加到底层数组的最前端还是最后端  
    // 注意，因为 value 的编码比集合原有的其他元素的编码都要大  
    // 所以 value 要么大于集合中的所有元素，要么小于集合中的所有元素  
    // 因此，value 只能添加到底层数组的最前端或最后端  
    int prepend = value < 0 ? 1 : 0;  
  
    /* First set new encoding and resize */  
    // 更新集合的编码方式  
    is->encoding = intrev32ifbe(newenc);  
    // 根据新编码对集合（的底层数组）进行空间调整  
    // T = O(N)  
    is = intsetResize(is,intrev32ifbe(is->length)+1);  
  
    /* Upgrade back-to-front so we don't overwrite values.  
     * Note that the "prepend" variable is used to make sure we have an empty  
     * space at either the beginning or the end of the intset. */  
    // 根据集合原来的编码方式，从底层数组中取出集合元素  
    // 然后再将元素以新编码的方式添加到集合中  
    // 当完成了这个步骤之后，集合中所有原有的元素就完成了从旧编码到新编码的转换  
    // 因为新分配的空间都放在数组的后端，所以程序先从后端向前端移动元素  
    // 举个例子，假设原来有 curenc 编码的三个元素，它们在数组中排列如下：  
    // | x | y | z |   
    // 当程序对数组进行重分配之后，数组就被扩容了（符号 ？ 表示未使用的内存）：  
    // | x | y | z | ? |   ?   |   ?   |  
    // 这时程序从数组后端开始，重新插入元素：  
    // | x | y | z | ? |   z   |   ?   |  
    // | x | y |   y   |   z   |   ?   |  
    // |   x   |   y   |   z   |   ?   |  
    // 最后，程序可以将新元素添加到最后 ？ 号标示的位置中：  
    // |   x   |   y   |   z   |  new  |  
    // 上面演示的是新元素比原来的所有元素都大的情况，也即是 prepend == 0  
    // 当新元素比原来的所有元素都小时（prepend == 1），调整的过程如下：  
    // | x | y | z | ? |   ?   |   ?   |  
    // | x | y | z | ? |   ?   |   z   |  
    // | x | y | z | ? |   y   |   z   |  
    // | x | y |   x   |   y   |   z   |  
    // 当添加新值时，原本的 | x | y | 的数据将被新值代替  
    // |  new  |   x   |   y   |   z   |  
    // T = O(N)  
    while(length--)  
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));  
  
    /* Set the value at the beginning or the end. */  
    // 设置新值，根据 prepend 的值来决定是添加到数组头还是数组尾  
    if (prepend)  
        _intsetSet(is,0,value);  
    else  
        _intsetSet(is,intrev32ifbe(is->length),value);  
  
    // 更新整数集合的元素数量  
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);  
  
    return is;  
}  
  
/* Resize the intset   
 *  
 * 调整整数集合的内存空间大小  
 *  
 * 如果调整后的大小要比集合原来的大小要大，  
 * 那么集合中原有元素的值不会被改变。  
 *  
 * 返回值：调整大小后的整数集合  
 *  
 * T = O(N)  
 */  
static intset *intsetResize(intset *is, uint32_t len) {  
  
    // 计算数组的空间大小  
    uint32_t size = len*intrev32ifbe(is->encoding);  
  
    // 根据空间大小，重新分配空间  
    // 注意这里使用的是 zrealloc ，  
    // 所以如果新空间大小比原来的空间大小要大，  
    // 那么数组原有的数据会被保留  
    is = zrealloc(is,sizeof(intset)+size);  
  
    return is;  
}  
```

