![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9Gr0G9aADP34MkvMqSANmULicgf0W4gNKcf7a0mmBbStb1iaxEGw9Ud1XQ/0?wx_fmt=jpeg)

#  布隆过滤器原理

##  布隆过滤器

###  1\. 是什么？

  * 它是一个二进制bit数组，初始为 0 

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9GEkYEDEj4eDv1HqB3fd4zFoaleVUMUffoaWicYo1xrjygpY9oGibkhs2A/640?wx_fmt=png&from=appmsg)

  * 采用位存储数据结构，节省存储空间 
  * 1 存在，0 不存在 
  * 可以添加，但是不能删除 
  * 存在误判 

###  2\. 干什么？

用于 ` 快速查找 ` 一个集合中是否存在某个元素。尤其是大数据量中快速查找判断是否存在的问题。目的就是“ ` 大海捞针 ` ”

使用场景：

  * 从10亿个手机号中如何快速判断10万个号码是否存在？ 
  * 白名单设置，如何正确识别合法用户？ 
  * 黑名单校验垃圾短信 
  * ...... 

###  3\. 为什么？

因为它主要的作用在于快速查找，我们通过元素查找的过程说明具体的 ` 原理 ` 和可能存在的 ` 问题 ` 。

假设，现有一个集合【码，道】，我们查找某元素是否存在？

**1、存储逻辑**

使用多个不同hash散列函数对“码”“道”进行计算，对应下标 标记为 1

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9GpCvIHKd7rTTLicWDdS3WyQMGYEdCNUFkic5tI5thRxNHXtLLqjic3mRHQ/640?wx_fmt=png&from=appmsg)

**2、查找逻辑**

1、查找示例

例如，在这个集合中，我们查找两一个元素“易”。

2、查找步骤

  * 使用多个不同hash散列函数对“易”进行计算 

  * 查找标记为 ` 全为 1 ` 代表存在，有一个为 0 代表不存在 

  * 如图示，“易”字在最后的位置计算的hash值对应为0，判定不存在集合中。 

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9GOichicH38R7GibqRsohWwe0ibKy4OsNUdgZqWvWLSeiaRJFFQLictHoHFsww/640?wx_fmt=png&from=appmsg)

###  4\. 有什么问题？

**（1）什么是误判？**

因为hash计算会产生hash冲突（或者hash碰撞）的问题。这就意味着： ` 不同的字符可能存在相同的hash结果值 ` 。如下图，“ 有” 和“道”
经过多个hash计算出相同的值。那么可能判定“有” 也存在于集合中，从而产生误判。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9GFFyKA7hQyOe1qicZJMgCBU2H7kl4fjhsehsKmibDoeSYKqlpaOKtqncA/640?wx_fmt=png&from=appmsg)

关于误判的解读，有句话挺有意思。鉴赏一下，哈哈~~
![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9GRAiaGX28FOauZcooOujSX6urVMDdepibFb69uzeotdICiaV5vrGsfCBVQ/640?wx_fmt=png&from=appmsg)

**（2）为什么不能删？**

布隆过滤器（Bloom Filter）是一种空间效率极高的 ` 概率型 ` 数据结构，它允许一定的 ` 误报率 ` 。但是它 ` 不支持删除 ` 操作。

主要由于哈希函数可能存在冲突（即不同的元素映射到相同的位置），因此一旦某个位置被设置为1，它就可能表示多个元素。如果尝试删除一个元素，那么就需要将该元素映射到的所有位置都重置为0。但是，这样做可能会影响到其他也映射到这些位置的元素，因为无法确定哪些1是由当前要删除的元素设置的，哪些是由其他元素设置的。

例如，紧跟上边的思路，要是删除“有”，就要将下标【3,7,8】同时置为 0 。显然误删了 “道”。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9G5Bh90u98HicIhYHUSy88bybKYl6pzqRCE97ibTo2kK9ImbIyXnJcoaBw/640?wx_fmt=png&from=appmsg)

**（3）有没有办法解决Hash冲突？**

关于这个问题。只能说，减小出现hash碰撞的概率。而不能彻底杜绝. 对此，有同学肯定想：

  * 拉长点：增加bit位数 

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9GWZJLvVr44QBhQDZiaThPPceSia3QDnJ6DchWEM4KMEAfrH3qSNSudRHg/640?wx_fmt=png&from=appmsg)

  * 更散列：再多用点hash函数 

![](https://mmbiz.qpic.cn/sz_mmbiz_png/VhiaUjy5R5VD0C2e9uEQicC6KYsoKEUM9GYlqfkPbVFckgLTahkbicjCW3kPzsicpV4IiaS6PBZgiaOU2jcN5StcaVKg/640?wx_fmt=png&from=appmsg)

事实上，在实际使用中，我们也会根据预估的业务数据，设置 ` 总位数 ` 和 ` 容错率 ` 这两个重要参数。这就是 ` 怎么用 ` 的问题。

###  5\. 怎么用？

> 一、使用 Google Guava库中的布隆过滤器

使用Google Guava库中的布隆过滤器（Bloom Filter）功能展示如何创建一个布隆过滤器，并如何使用它来检查元素是否可能存在于集合中。

  * 在 ` pom.xml ` 中添加依赖： 

    

    ````xml
    <dependencies>  
        <dependency>  
            <groupId>com.google.guava</groupId>  
            <artifactId>guava</artifactId>  
            <version>30.1-jre</version> <!-- Use the latest version -->  
        </dependency>  
    </dependencies>  
    ````

    

  * 编写Java代码来创建和使用布隆过滤器： 

    
    
    ````java
    import com.google.common.hash.BloomFilter;  
    import com.google.common.hash.Funnels;  
    
    public class BloomFilterExample {  
        public static void main(String[] args) {  
            // 初始化布隆过滤器  
            // 预期插入的元素数量  
            long expectedInsertions = 10000L;  
            // 期望的误报率  
            double fpp = 0.03; // 3%的误报率  
    
            // 使用Funnels.stringFunnel()来处理String类型的数据  
            BloomFilter<String> bloomFilter = BloomFilter.create(Funnels.stringFunnel(Charsets.UTF_8), expectedInsertions, fpp);  
    
            // 插入元素  
            bloomFilter.put("element1");  
            bloomFilter.put("element2");  
            // ... 可以继续插入其他元素  
    
            // 检查元素是否可能存在于集合中  
            boolean mightContainElement1 = bloomFilter.mightContain("element1");  
            System.out.println("Element1 might be present: " + mightContainElement1); // 应该输出 true  
    
            boolean mightContainElement3 = bloomFilter.mightContain("element3");  
            System.out.println("Element3 might be present: " + mightContainElement3); // 可能输出 true 或 false，取决于误报率  
    
            // 注意：布隆过滤器可能会产生误报（false positives），但不会产生误报（false negatives）  
            // 如果 mightContain 返回 false，那么该元素肯定不在集合中  
            // 如果 mightContain 返回 true，那么该元素可能在集合中，但也可能不在  
        }  
    } 
    ````
    
    

> 二、使用 Redisson中的布隆过滤器

使用 Redisson 创建布隆过滤器，插入元素，并检查某个元素是否存在。

  * 在 ` pom.xml ` 文件中添加 Redisson 依赖： 

    

    ````xml
    <dependencies>  
        <dependency>  
            <groupId>org.redisson</groupId>  
            <artifactId>redisson</artifactId>  
            <version>3.16.1</version> <!-- 使用最新版本 -->  
        </dependency>  
    </dependencies>
    ````

    

  * 编写代码来创建布隆过滤器，插入元素，并检查元素： 

    
    
    ````java
    import org.redisson.Redisson;  
    import org.redisson.api.RBloomFilter;  
    import org.redisson.api.RedissonClient;  
    import org.redisson.config.Config;  
    
    public class RedissonBloomFilterExample {  
    
        public static void main(String[] args) {  
            // 配置 Redisson 客户端  
            Config config = new Config();  
            config.useSingleServer()  
                  .setAddress("redis://127.0.0.1:6379"); // 替换为你的 Redis 服务器地址  
    
            // 创建 Redisson 客户端实例  
            RedissonClient redisson = Redisson.create(config);  
    
            // 创建布隆过滤器  
            // 注意：这里的名称 "myBloomFilter" 是布隆过滤器的唯一标识，你可以根据需要更改  
            RBloomFilter<String> bloomFilter = redisson.getBloomFilter("myBloomFilter");  
    
            // 初始化布隆过滤器，设置预期插入的元素数量和误报率  
            bloomFilter.tryInit(10000L, 0.03);  
    
            // 插入元素  
            bloomFilter.add("码");  
            bloomFilter.add("道");  
    
            // 查找元素  
            boolean mightContainYi = bloomFilter.contains("易");  
            System.out.println("布隆过滤器中可能包含'易'：" + mightContainYi);  
    
            // 关闭 Redisson 客户端  
            redisson.shutdown();  
        }  
    }  
    ````
    
    

> _注意_ ：由于布隆过滤器的特性， ` contains ` 方法返回 ` true ` 并不意味着元素一定存在，而返回 ` false `
> 则意味着元素一定不存在。

对于 ` bloomFilter.tryInit(10000L, 0.03); ` 的参数设置，应根据实际业务给出。

##  

