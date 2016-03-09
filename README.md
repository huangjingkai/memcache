# Memcached Client for Golang
golang版本的memcached客户端，使用二进制协议，支持分布式，支持连接池，支持多种数据格式

### 特性
* 支持多server集群
* 与memcached使用二进制协议通信
* 支持连接池
* 存储value支持golang基本数据类型：string、[]byte、int、int8、int16、int32、int64、bool、uint8、uint16、uint32、uint64、float32、float64、map、结构体，不需要单独转为string存储
* Replace、Increment/Decrement、Delete、Append/Prepend命令支持cas原子操作

### 分布式集群
默认开启分布式集群，key按照一致性哈希算法分配到各server，当server无法连接时如果设置了SetRemoveBadServer(true)则自动被剔除server列表，等到恢复正常时再重新加入server列表

### 使用
##### 下载

    go get github.com/pangudashu/memcache

##### 导入

    package main

    import(
        "fmt"
        "github.com/pangudashu/memcache"
    )

    func main(){
        
        /**
         * server配置
         * Address  string          //host:port
         * Weight   int             //权重        
         * InitConn int:            //初始化连接数 < MaxCnt
         * MaxConn  int:            //最大连接数
         * IdleTime time.Duration:  //空闲连接有效期
         */

        s1 := &memcache.Server{Address: "127.0.0.1:12000", Weight: 50}
        s2 := &memcache.Server{Address: "127.0.0.1:12001", Weight: 20}
        s3 := &memcache.Server{Address: "127.0.0.1:12002", Weight: 20}
        s4 := &memcache.Server{Address: "127.0.0.1:12003", Weight: 10}

        //初始化连接池
        mc, err := memcache.NewMemcache([]*memcache.Server{s1, s2, s3, s4})
        if err != nil {
            fmt.Println(err)
            return
        }

        //设置是否自动剔除无法连接的server，默认不开启(建议开启)
        //如果开启此选项被踢除的server如果恢复正常将会再次被加入server列表
        mc.SetRemoveBadServer(true)
        //设置连接、读写超时
        mc.SetTimeout(time.Second*2, time.Second, time.Second)

        mc.Set("test_key",true)
        fmt.Println(mc.Get("test_key"))

        //...

        mc.Close()
    }

##### 示例
github.com/pangudashu/memcache/example/example.go

### 命令列表
###### Get
    
    根据key检索一个元素

    【说明】
    Get(key string [, format_struct interface{} ])(value interface{}, cas uint64, err error)
    
    【参数】
    key    要检索的元素的key
    format 用于存储的value为map、结构体时，返回值将直接反序列化到format

    【返回值】
    value为interface，取具体存储的值需要断言
    存储的value为map、结构体时,value将返回nil 
    
        type User struct {
            //...
        }

        var user User
        if _, _, e := mc.Get("pangudashu_struct", &user); e != nil {
            fmt.Println(e)
        } else {
            fmt.Println(user)
        }


###### Set
    
    向一个新的key下面增加一个元素

    【说明】
    Set(key string, value interface{} [, expire ...uint32 ]) (res bool, err error)

    【参数】
    key    用于存储值的键名
    value  存储的值，可以为string、[]byte、int、int8、int16、int32、int64、bool、uint8、uint16、uint32、uint64、float32、float64、map、struct等类型
    expire 过期时间，默认0

    【返回值】
    设置成功res返回true，err返回nil，否则res返回false，err返回memcache.ErrNotStord

    【注意】
    int类型长度与系统位数相关，所以实际存储转为string，建议尽量使用具体长度的类型：int8、int16、int32、int64替换

        //demo
        var value uint32 = 360000000000
        mc.Set("test_value", value, 1800)

###### Add

    向一个新的key下面增加一个元素,与Set类似，但是如果 key已经在服务端存在，此操作会失败

    【说明】
    Add(key string, value interface{} [, expire uint32 ]) (res bool, err error)

    【参数】
    同Set

    【返回值】
    同Set。
    如果key已经存在，res返回false，err返回memcache.ErrKeyExists

###### Replace
    
    替换已存在key下的元素,类似Set，但是如果服务端不存在key，操作将失败

    【说明】
    Replace(key string, value interface{} [, expire uint64 [, cas uint64 ]]) (res bool, err error)

    【参数】
    key    用于存储值的键名
    value  存储的值
    expire 过期时间
    cas    数据版本号，原子替换，如果数据在此操作前已被其它客户端更新，则替换失败

        _,cas,_ := mc.Get("test_key")

        res, er := mc.Replace("test_key", "new value~", 0, cas) //每次更新操作数据的cas都会变，所以如果这个值在Get后被其它client更新了则返回false，err返回memcache.ErrKeyExists

###### Delete
    
    删除一个元素

    【说明】
    Delete(key string [, cas uint64 ]) (res bool, err error)

    【参数】
    key 要删除的key
    cas 数据版本号，如果数据在此操作前已被其它客户端更新，则删除失败

    【返回值】
    成功时返回 true，或者在失败时返回 false，如果key不存在err返回 memcache.ErrNotFound

###### Increment

    增加数值元素的值,如果key不存在则操作失败

    【说明】
    Increment(key string [, delta int [, cas int ]]) (res bool, err error)

    【参数】
    key   要增加值的元素的key
    delta 要将元素的值增加的大小,默认1
    cas   数据版本号，只有当服务端cas没有变化时此操作才成功

    【返回值】
    成功时返回 true，或者在失败时返回 false，如果key不存在则初始化为0，如果cas版本号已变err返回memcache.ErrKeyExists

    【注意】
    Increment/Decrement只能操作value类型为int的值，其它任何类型均无法操作。(原因是memcached中在Incr/Decr处理时首先使用strtoull将value转为unsigned long long再进行加减操作，所以只有将数值存为字符串strtoull才能将其转为合法的数值)

###### Decrement
    
    减小数值元素的值
    
    【说明】
    Decrement(key string [, delta int [, cas int ]]) (res bool, err error)

    【参数】
    同Increment

###### Flush
    
    删除缓存中的所有元素

    【说明】
    Flush([ delay uint32 ]) (res bool, err error)

    【参数】
    delay 在flush所有元素之前等待的时间（单位秒）

    【返回值】
    成功时返回 true， 或者在失败时返回 false

###### Append
    
    向已存在string元素后追加数据

    【说明】
    Append(key string, value string [, cas uint64 ]) (res bool, err error)

    【参数】
    key   用于存储值的键名
    value 将要追加的值

    【返回值】
    成功时返回 false， 或者在失败时返回 false。 如果key不存在err返回memcache.ErrNotFound

    【注意】
    Append/Prepend只能操作string类型，尽管操作其它类型时也能转化为string，但是将导致数据原来的类型失效，也就是说Append/Prement能够成功，但是Get时将出错

###### Prepend

    向已存在string元素前追加数据

    【说明】
    Prepend(key string, value string [, cas uint64 ]) (res bool, err error)

    同Append

###### Version
    
    获取memcached服务端版本

    【说明】
    Version(server *memcache.Server) (v string, err error)


    【参数】
    server server配置结构

    【返回值】
    memcached version

### 错误编码
* ErrNotConn     : Can't connect to server
* ErrNotFound    : Key not found
* ErrKeyExists   : Key exists
* ErrInval       : Invalid arguments
* ErrNotStord    : Item not stored
* ErrDeltaBadVal : Increment/Decrement on non-numberic value
* ErrMem         : Out of memery
* ErrInvalValue  : Unkown value type
* ErrInvalFormat : Invalid format struct
* ErrNoFormat    : Format struct empty
* ErrUnkown      : Unkown error
