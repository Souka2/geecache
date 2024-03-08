### 项目介绍

+ 本项目是基于groupcache实现的一个分布式缓存，主要分为了Group模块，缓存模块、分布式一致性等模块。

##### Group模块

+ Group模块是对外提供服务接⼝的部分，⼀个Group就是⼀个缓存空间。其要实现对缓存的增删查方法。

```go
//Group 提供命名管理缓存/填充缓存的能⼒
type Group struct {
  name             string               // 缓存空间的名字
  getter           Getter               // 数据源获取数据
  mainCache        *cache               // 主缓存，并发缓存
  hotCache         *cache               // 热点缓存
  server           Picker               // ⽤于获取远程节点请求客户端
  flight           *singleflight.Flight // 避免对同⼀个key多次加载造成缓存击穿
  emptyKeyDuration time.Duration        // getter返回error时对应空值key的过期时间
}
```

##### 缓存模块

+ 缓存模块是数据实际存储的位置，其中实现缓存淘汰算法，过期机制，回调机制等，缓存模块与其他部分是解耦的。  

##### byteview模块

+ byteview是对实际缓存的一层封装，因为实际的缓存值是一个byte切片存储的，而切片的底层是一个指向底层数组的指针，一个记录长度的变量和一个记录容量的变量。   

+ 如果获取缓存值时直接返回缓存值的切片，那个切片只是原切片三个变量的拷贝，真正的缓存值就可能被外部恶意修改。 

+ 所以用byteView进行一层封装，返回缓存值时的byteView则是一个原切片的深拷贝。 

```go
type ByteView struct {
  b      []byte
  expire time.Time // 过期时间
}
```

##### 分布式一致性模块

+ 分布式一致性模块实现了一致性哈希算法，将机器节点组成哈希环，为每个节点提供了从其他节点获取缓存的能力。

```go
type Consistence struct {
  hash     Hash           // 哈希函数依赖
  replicas int            // 虚拟节点倍数
  ring     []int          // 哈希环
  hashMap  map[int]string // 虚拟节点hash到真实节点名称的映射
}
```

#### 流程

+ 当Group模块接收到请求时，当前节点先到本地的主缓存中查找是否有目标值。

+ 如果没有再查看此节点的热点缓存中是否有，如果也没有，就需要到远程伙伴节点去获取。

+ 通过一致性哈希计算该key的哈希值，找到哈希环上相应的节点，执行该节点的远程调用函数获取值，并把该值加入到当前节点的热点缓存中。

+ 如果该key在哈希环中对应的就是当前节点，那么就需要从本地的数据源去获取数据后加载到当前节点的主缓存中。
