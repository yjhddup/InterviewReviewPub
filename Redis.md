
### 五种基本数据结构
##### 1.string(字符串）

 **内部实现**就是字符数组。redis所有数组结构的key都以string作为名称，然后通过唯一的名称来获取对应的value。

**用途** ：缓存用户信息。将用户信息结构体使用json序列化为字符串，然后将序列化后的字符串塞进redis来缓存，获取用户信息的时候反序列化

**动态字符串**，是可以修改的字符串。**预分配冗余空间**（减少内存的频繁分配），当前字符串分配的实际空间capacity一般高于实际字符串长度len。当字符串小于1MB时，加倍现有空间。大于1M时候，多扩1M。最大长度为512MB

`set key value`

`get key` 

**批量键值对**

set key value key value key value …………

**过期和set命令扩展** 可以对key设置过期时间，但时间就会被自动删除，通常 用来**控制缓存的失效时间**

。具体详见**过期策略**

` set key value`

expire name 5 #5s后过期

**setex** name 5 code #5s后过期，相当于set + expire

setnx key value #如果name不存在就执行set创建

**计数**

如果 value是一个整数可以进行自增操作。自增范围在signed long最小值和最大值之间，超出会报错

`set age 30`

`incr age`

`incrby age 5`

##### 2.list列表

相当于**linkedlist**，是**链表**而不是数组。插入删除比较快，索引定位很慢。**双向指针**可以向前也可以向后遍历。具体底层数据结构为快速链表

通常用来做异步队列。将需要延后处理的任务结构体序列化或字符串，塞进redis列表，另一个线程从这个列表轮询数据进行处理。

【右进左出：队列】

`repush key value1 value2 value3 ……`

`lpop key`

【右进右出：栈】

`repush key value1 value2 value3 ……`

`rpop key`

【慢操作】

lindex：获取对应下标的值，需要对链表进行遍历，性能随着参数index增大而变差

ltrim：保留参数sart_index到start_index之间的值。O(n) 慎用

index可以为负数，-1表示倒数第一个元素

llen：元素个数

【快速链表】

待详细补充

##### 3. hash（字典）

相当于hashmap，是一个无序的字典，也是一个数组加链表的数据结构。键只能是字符串

**渐进式rehash**: 在rehash的时候，保留新旧两个hash结构，查询时会同时查询两个hash结构，然后在后续的定时任务以及hash操作指令中，循序渐进地将旧hash的内容一点点地迁移到新的hash结构中。搬迁完成之后，就会使用新得结构取而代之。

当hash**移除**了**最后一个元素**之后，该数据结构被**自动删除**，内存被回收。

可以用来存储用户信息，可以对用户结构中每个字段单独存储。整个字符串一次性读全部，浪费网络流量

hash结构的的存储消耗高于单个字符串

`hset key sub-key1 value`

`hset key sub-key2 value (如果key已经存在，当前操作为更新)`

`hgetall key   # entries ,ke和value间隔出现`

`hlen key`

`hget key sub-key`

`hmset key sub-key1 value sub-key2 value #批量set`

hincrby sub-key可以进行自增操作

##### 4.set（集合）

相当于hashset，内部的键值对是**无序的、唯一的**。内部实现相当于一个特殊的字典，所有的value都是null

**最后一个元素被移除之后，数据结构会被删除。**

可以用来存储中奖用户ID，有去重功能，可以保证同一个用户不会中奖两次

`sadd key value`

`sadd key value value2`

`smembers key   # 和插入顺序不一致`

`sismember key value #查询value是否存在`

`scard books #获取长度`

`spop key #弹出一个`

##### 5.zset(有序列表)

相较于set可以给每个value赋予排序权重score。内部实现**跳表**

最后一个value被移除之后，数据结构自动删除，内存被回收

`zadd key score value`

`zrange key start_index end_index # 按score排序列出，参数区间为排名范围`

`zrevrange key start_index end_index #按score逆序列出`

`zcard key #相当于count`

`zrank key value #  排名`

`zrangebyscore key min_score max_score #按分值区间进行遍历`

`zrem key value #删除value`

【**跳跃链表**】

待补充

##### 容器型数据结构的通用规则

list、set、hash、zset是容器型数据就结构

1. create if not exits：如果容器不存在，那就创建一个，再进行操作。比如rpush操作刚开始是没有列表的，redis就会自动创建一个，然后再rpush进去新元素
2. dorp id no elements：如果容器里的元素都没有了，那么立即删除容器，释放内存

##### 过期时间

redis所有的数据结构都可以设置过期时间，时间到了就自动删除对应的对象（不是某个子key的过期）。如果设置过期时间之后使用set方法修改它，过期时间会消失。

### 分布式锁