#  Redis字符串底层实现

编码其实就是底层数据结构，字符串对象的编码可以是 int 、 raw 或者 embstr 。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6PJxXibAv1FPYs5RLoyAVI4HM4AqKkzicwsKb7YRv60FicSUVOReYBDcPIQ/640?wx_fmt=png&from=appmsg)
字符串对象的三种编码

##  int编码

如果一个字符串对象保存的是整数值， 并且这个整数值可以用 long 类型来表示， 那么字符串对象会将整数值保存在字符串对象结构的 ptr 属性里面（将
void* 转换成 long ）， 并将字符串对象的编码设置为 int 。


​    
    redis> SET number 10086  
    OK  
    redis> OBJECT ENCODING number  
    "int"  


![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6Pp9jZIuYG4krWN56fcHkIej4kEXwibbDshxqFRydq6ZS3hVeYiblJzf6A/640?wx_fmt=png&from=appmsg)
int编码的字符串对象

##  raw编码

如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度大于 39 字节， 那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值，
并将对象的编码设置为 raw 。


​    
    redis> SET story "Long, long, long ago there lived a king ..."  
    OK  
    redis> STRLEN story  
    (integer) 43  
    redis> OBJECT ENCODING story  
    "raw"  


![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6Pia1w4gXiaonoME0SDHq8R5UCKN6ZD3V5vTOCWVxdZPbjqBLJekftibesQ/640?wx_fmt=png&from=appmsg)
raw编码的字符串对象

##  embstr编码

如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度小于等于 39 字节， 那么字符串对象将使用 embstr 编码的方式来保存这个字符串值。

使用 embstr 编码的字符串对象来保存短字符串值有以下好处：

  * embstr 编码将创建字符串对象所需的内存分配次数从 raw 编码的两次降低为一次。 
  * 释放 embstr 编码的字符串对象只需要调用一次内存释放函数， 而释放 raw 编码的字符串对象需要调用两次内存释放函数。 
  * 因为 embstr 编码的字符串对象的所有数据都保存在一块连续的内存里面， 所以这种编码的字符串对象比起 raw 编码的字符串对象能够更好地利用缓存带来的优势。 

    
    
    redis> SET msg "hello"  
    OK  
    redis> OBJECT ENCODING msg  
    "embstr"  

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6PEd73daD3bdicnLYia4HA0Q2GHF3ObDLI1X1gXyf1uKcCISGmlREv2Cpw/640?wx_fmt=png&from=appmsg)
embstr编码的字符串对象

##  浮点数

如果我们要保存一个浮点数到字符串对象里面， 那么程序会先将这个浮点数转换成字符串值， 然后再保存起转换所得的字符串值。


​    
    redis> SET pi 3.14  
    OK  
    redis> OBJECT ENCODING pi  
    "embstr"  

在有需要的时候， 程序会将保存在字符串对象里面的字符串值转换回浮点数值， 执行某些操作， 然后再将执行操作所得的浮点数值转换回字符串值，
并继续保存在字符串对象里面。


​    
    redis> INCRBYFLOAT pi 2.0  
    "5.14"  
    redis> OBJECT ENCODING pi  
    "embstr"  


字符串对象保存各类型值的编码方式

##  小结

值  |  编码   
---|---  
可以用 long 类型保存的整数。  |  int   
可以用 long double 类型保存的浮点数。  |  embstr 或者 raw   
字符串值， 或者因为长度太大而没办法用 long 类型表示的整数， 又或者因为长度太大而没办法用 long double 类型表示的浮点数。  |  embstr 或者 raw   

其中 embstr 和 raw 编码都是基于 SDS（简单动态字符串）实现的。

#  简单动态字符串

Redis 没有直接使用 C 语言传统的字符串表示（以空字符结尾的字符数组，以下简称 C 字符串）， 而是自己构建了一种名为简单动态字符串（simple
dynamic string，SDS）的抽象类型， 并将 SDS 用作 Redis 的默认字符串表示。

除了用来保存数据库中的字符串值之外， SDS 还被用作缓冲区（buffer）：AOF 模块中的 AOF 缓冲区， 以及客户端状态中的输入缓冲区， 都是由
SDS 实现的。

##  SDS 的定义


​    
    struct sdshdr {  
        // 记录 buf 数组中已使用字节的数量  
        // 等于 SDS 所保存字符串的长度  
        int len;  
        // 记录 buf 数组中未使用字节的数量  
        int free;  
        // 字节数组，用于保存字符串  
        char buf[];  
    };  


  * free 属性的值为 0 ， 表示这个 SDS 没有分配任何未使用空间。 
  * len 属性的值为 5 ， 表示这个 SDS 保存了一个五字节长的字符串。 
  * buf 属性是一个 char 类型的数组， 数组的前五个字节分别保存了 'R' 、 'e' 、 'd' 、 'i' 、 's' 五个字符， 而最后一个字节则保存了空字符 '\0' 。 

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6PH9KTtz4ApicicPKfv0Eibw796JfQlf1DO4k7gEjMDj4AKickibMrhyNNr2A/640?wx_fmt=png&from=appmsg)

带有未使用空间的SDS：

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6PFGQP5MGykSkRWmarkwcVrFRaJQACySqWoSFLrXSLJbXSp8N0Sk5CIA/640?wx_fmt=png&from=appmsg)
带有未使用空间的示例

##  SDS 与 C 字符串的区别

###  常数复杂度获取字符串长度

因为 C 字符串并不记录自身的长度信息， 所以为了获取一个 C 字符串的长度， 程序必须遍历整个字符串， 对遇到的每个字符进行计数，
直到遇到代表字符串结尾的空字符为止， 这个操作的复杂度为 O(N) 。

和 C 字符串不同， 因为 SDS 在 len 属性中记录了 SDS 本身的长度， 所以获取一个 SDS 长度的复杂度仅为 O(1) 。

设置和更新 SDS 长度的工作是由 SDS 的 API 在执行时自动完成的， 使用 SDS 无须进行任何手动修改长度的工作。

###  杜绝缓冲区溢出


​    
    char *strcat(char *dest, const char *src);  


因为 C 字符串不记录自身的长度， 所以 strcat 假定用户在执行这个函数时， 已经为 dest 分配了足够多的内存， 可以容纳 src
字符串中的所有内容， 而一旦这个假定不成立时， 就会产生缓冲区溢出。

与 C 字符串不同， SDS 的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当 SDS API 需要对 SDS 进行修改时， API 会先检查 SDS
的空间是否满足修改所需的要求， 如果不满足的话， API 会自动将 SDS 的空间扩展至执行修改所需的大小， 然后才执行实际的修改操作， 所以使用 SDS
既不需要手动修改 SDS 的空间大小， 也不会出现前面所说的缓冲区溢出问题。

###  减少修改字符串时带来的内存重分配次数

每次增长或者缩短一个 C 字符串， 程序都总要对保存这个 C 字符串的数组进行一次内存重分配操作：

  * 如果程序执行的是增长字符串的操作， 比如拼接操作（append）， 那么在执行这个操作之前， 程序需要先通过内存重分配来扩展底层数组的空间大小 —— 如果忘了这一步就会产生缓冲区溢出。 
  * 如果程序执行的是缩短字符串的操作， 比如截断操作（trim）， 那么在执行这个操作之后， 程序需要通过内存重分配来释放字符串不再使用的那部分空间 —— 如果忘了这一步就会产生内存泄漏。 

**空间预分配**

空间预分配用于优化 SDS 的字符串增长操作：当 SDS 的 API 对一个 SDS 进行修改， 并且需要对 SDS 进行空间扩展的时候， 程序不仅会为
SDS 分配修改所必须要的空间， 还会为 SDS 分配额外的未使用空间。

额外分配的未使用空间数量由以下公式决定：

  * 如果对 SDS 进行修改之后， SDS 的长度（也即是 len 属性的值）将小于 1 MB ， 那么程序分配和 len 属性同样大小的未使用空间， 这时 SDS len 属性的值将和 free 属性的值相同。举个例子， 如果进行修改之后， SDS 的 len 将变成 13 字节， 那么程序也会分配 13 字节的未使用空间， SDS 的 buf 数组的实际长度将变成 13 + 13 + 1 = 27 字节（额外的一字节用于保存空字符）。 

  * 如果对 SDS 进行修改之后， SDS 的长度将大于等于 1 MB ， 那么程序会分配 1 MB 的未使用空间。举个例子， 如果进行修改之后， SDS 的 len 将变成 30 MB ， 那么程序会分配 1 MB 的未使用空间， SDS 的 buf 数组的实际长度将为 30 MB + 1 MB + 1 byte 。 

例如：


​    
    sdscat(s, " Cluster");  


那么 sdscat 将执行一次内存重分配操作， 将 SDS 的长度修改为 13 字节， 并将 SDS 的未使用空间同样修改为 13 字节。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6PoiblSJbaicYQbTxEqjv4SeS34sg1w8XZ3zqPDsyJU310Pcib4RAoqk9GA/640?wx_fmt=png&from=appmsg)
未使用sdscat之前的SDS
![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6PQlo12pflibl5eKKjQjGcpF1j6ZThgjZyTRozJ0fcILZeXAqAYT7SVpQ/640?wx_fmt=png&from=appmsg)
使用sdscat之后的SDS

如果这时， 我们再次对 s 执行：


​    
    sdscat(s, " Tutorial");  


那么这次 sdscat 将不需要执行内存重分配：因为未使用空间里面的 13 字节足以保存 9 字节的 " Tutorial" 。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6Pvack4OM8wfe5XRAmiaQNfbnDCbkuxk3cg8ogy0pQzGhCJ6ERKpOey4Q/640?wx_fmt=png&from=appmsg)
再次执行sdscat之后的SDS

在扩展 SDS 空间之前， SDS API 会先检查未使用空间是否足够， 如果足够的话， API 就会直接使用未使用空间， 而无须执行内存重分配。

通过这种预分配策略， SDS 将连续增长 N 次字符串所需的内存重分配次数从必定 N 次降低为最多 N 次。

**惰性空间释放**

惰性空间释放用于优化 SDS 的字符串缩短操作：当 SDS 的 API 需要缩短 SDS 保存的字符串时，
程序并不立即使用内存重分配来回收缩短后多出来的字节， 而是使用 free 属性将这些字节的数量记录起来， 并等待将来使用。

SDS 值 s 来说， 执行：


​    
    sdstrim(s, "XY");   // 移除 SDS 字符串中的所有 'X' 和 'Y'  


![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6PPwwkEYukmNz85TYw2dpSdcyLakvx3jAVRIFqT1Go4VxXzMbSmnQ17A/640?wx_fmt=png&from=appmsg)
执行 sdstrim 之前的SDS
![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6P8patBqBhB3r0xias0SuHyVukxkYIFNR6a5n0z9HbPy0V66aecU2VTfA/640?wx_fmt=png&from=appmsg)
执行 sdstrim 之后的SDS

###  二进制安全

C 字符串中的字符必须符合某种编码（比如 ASCII）， 并且除了字符串的末尾之外， 字符串里面不能包含空字符，
否则最先被程序读入的空字符将被误认为是字符串结尾 —— 这些限制使得 C 字符串只能保存文本数据， 而不能保存像图片、音频、视频、压缩文件这样的二进制数据。

SDS 的 API 都是二进制安全的（binary-safe）：所有 SDS API 都会以处理二进制的方式来处理 SDS 存放在 buf 数组里的数据，
程序不会对其中的数据做任何限制、过滤、或者假设 —— 数据在写入时是什么样的， 它被读取时就是什么样。

使用 SDS 来保存中间包含空格的字符串没有任何问题， 因为 SDS 使用 len 属性的值而不是空字符来判断字符串是否结束。

![](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwuYzNibI9xbxVG666D4Kfm6Pmkmf0uiaC9NP0Kv45NvHMUfWK7NTS9lCh3JgJnaOfyt2qrIiaaXOpG5g/640?wx_fmt=png&from=appmsg)
保存特殊数据格式的SDS

###  兼容部分 C 字符串函数

SDS遵循 C 字符串以空字符结尾的惯例， 所以SDS 可以在有需要时重用 <string.h> 函数库， 从而避免了不必要的代码重复。


​    
    // 对比字符串  
    strcasecmp(sds->buf, "hello world");  
    // 拼接字符串  
    strcat(c_string, sds->buf);  


##  字符串扩容


​    
    /*  
     * 对 sds 中 buf 的长度进行扩展，确保在函数执行之后，  
     * buf 至少会有 addlen + 1 长度的空余空间  
     * （额外的 1 字节是为 \0 准备的）  
     *  
     * 返回值  
     *  sds ：扩展成功返回扩展后的 sds  
     *        扩展失败返回 NULL  
     *  
     * 复杂度  
     *  T = O(N)  
     */  
    sds sdsMakeRoomFor(sds s, size_t addlen) {  
      
        struct sdshdr *sh, *newsh;  
      
        // 获取 s 目前的空余空间长度  
        size_t free = sdsavail(s);  
      
        size_t len, newlen;  
      
        // s 目前的空余空间已经足够，无须再进行扩展，直接返回  
        if (free >= addlen) return s;  
      
        // 获取 s 目前已占用空间的长度  
        len = sdslen(s);  
        sh = (void*) (s-(sizeof(struct sdshdr)));  
      
        // s 最少需要的长度  
        newlen = (len+addlen);  
      
        // 根据新长度，为 s 分配新空间所需的大小  
        if (newlen < SDS_MAX_PREALLOC)  
            // 如果新长度小于 SDS_MAX_PREALLOC   
            // 那么为它分配两倍于所需长度的空间  
            newlen *= 2;  
        else  
            // 否则，分配长度为目前长度加上 SDS_MAX_PREALLOC  
            newlen += SDS_MAX_PREALLOC;  
        // T = O(N)  
        newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);  
      
        // 内存不足，分配失败，返回  
        if (newsh == NULL) return NULL;  
      
        // 更新 sds 的空余长度  
        newsh->free = newlen - len;  
      
        // 返回 sds  
        return newsh->buf;  
    }  

