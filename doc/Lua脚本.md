# Lua语言简介
<code>Lua</code>是一种轻量小巧的脚本语言，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。
## 语言特性
<code>Lua</code>语言拥有以下特性：
- **轻量级** 它用标准C语言编写并以源代码形式开放，编译后仅仅一百余K，可以很方便的嵌入别的程序里。
- **可扩展** Lua提供了非常易于使用的扩展接口和机制：由宿主语言(通常是C或C++)提供这些功能，Lua可以使用它们，就像是本来就内置的功能一样。
- **其它特性** 支持面向过程(procedure-oriented)编程和函数式编程(functional programming)；
  自动内存管理；只提供了一种通用类型的表（table），用它可以实现数组，哈希表，集合，对象；
  语言内置模式匹配；闭包(closure)；函数也可以看做一个值；提供多线程（协同进程，并非操作系统所支持的线程）支持；通过闭包和table可以很方便地支持面向对象编程所需要的一些关键机制，比如数据抽象，虚函数，继承和重载等。

## 应用场景
<code>Lua</code>的应用场景有：
- 游戏开发
- 独立应用脚本
- Web 应用脚本
- 扩展和数据库插件如：MySQL Proxy 和 MySQL WorkBench
- 安全系统，如入侵检测系统
- Redis
# 数据结构
```c
struct redisServer {
	// 忽略其他配置信息
	
    // Lua 环境
    lua_State *lua; /* The Lua interpreter. We use just one for all clients */
    
    // 复制执行 Lua 脚本中的 Redis 命令的伪客户端
    redisClient *lua_client;   /* The "fake client" to query Redis from Lua */

    // 当前正在执行 EVAL 命令的客户端，如果没有就是 NULL
    redisClient *lua_caller;   /* The client running EVAL right now, or NULL */

    // 一个字典，值为 Lua 脚本，键为脚本的 SHA1 校验和
    dict *lua_scripts;         /* A dictionary of SHA1 -> Lua scripts */
    // Lua 脚本的执行时限
    mstime_t lua_time_limit;  /* Script timeout in milliseconds */
    // 脚本开始执行的时间
    mstime_t lua_time_start;  /* Start time of script, milliseconds time */

    // 脚本是否执行过写命令
    int lua_write_dirty;  /* True if a write command was called during the
                             execution of the current script. */

    // 脚本是否执行过带有随机性质的命令
    int lua_random_dirty; /* True if a random command was called during the
                             execution of the current script. */

    // 脚本是否超时
    int lua_timedout;     /* True if we reached the time limit for script
                             execution. */

    // 是否要杀死脚本
    int lua_kill;         /* Kill the script if true. */
};
```
如上，关于<code>Lua</code>在Redis中的数据表示部分均存储在<code>redisServer</code>的Redis服务端中
- **lua** 表示Lua的执行环境，所有的客户端共享**一个**Lua环境
- **lua_client** 复制执行<code>Lua</code>脚本中的 Redis 命令的伪客户端
- **lua_caller** 当前正在执行<code>EVAL</code>命令的客户端，如果没有就是NULL
- **lua_scripts** 这里使用<code>字典</code>存储，键为脚本的<code>SHA1</code>校验和，值为 Lua 脚本，即<code><SHA1,SCRIPT></code>。

# 协作组件
## 伪客户端
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5efd59e4fc1846bab5e49dc5bcb0a092~tplv-k3u1fbpfcp-zoom-1.image)
在Redis中，所有的客户端的Lua执行环境只有一个，这个Lua执行环境是专门处理Lua语言脚本相关功能的，体现了执行环境较好的**隔离性**和**独立性**，而它又是作为Redis的一种寄存语言存在的，Redis作为Lua的宿主存在，因此在利用Lua的各种语言特性和优势的同时，也要兼容宿主Redis语言的命令支持，为了避免耦合，Redis在这里采用了一种**伪客户端**的形式来联通Redis与Lua，使得Lua脚本可以轻松调用Redis提供的命令支持。
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e201e42d9ca344e7a0ed16782092d46e~tplv-k3u1fbpfcp-zoom-1.image)
Lua脚本中可以通过<code>redis.call</code>或<code>redis.pcall</code>函数进行Redis命令的调用，执行步骤如下：
- **Lua环境**将<code>redis.call</code>或<code>redis.pcall</code>函数要执行的命令传递给**伪客户端**
- **伪客户端**将执行命令传递给Redis的**命令执行器**
- **命令执行器**执行命令，并将执行结果返回给**伪客户端**
- **伪客户端**接收返回结果，将其返回给**Lua环境**
- 返回结果会从调用发起的<code>redis.call</code>或<code>redis.pcall</code>函数中返回给调用者
## 脚本缓存
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f831e7db4e0b42ca8a0d499e543f670a~tplv-k3u1fbpfcp-zoom-1.image)
Lua脚本缓存会以<code>字典</code>形式进行存储，键Key为脚本文件的<code>SHA1校验和</code>，值Value为<code>脚本文件</code>，即脚本内容。
> 由于Lua脚本对于Redis命令来说可能是巨大且批量的命令，为了减少脚本执行耗费在网络传输上的性能损耗，这里会进行缓存，通过文件校验和来匹配已缓存脚本数据提高性能
# 命令实现
## eval
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d96ce646fe64180a294f5228ff671f0~tplv-k3u1fbpfcp-zoom-1.image)
如上，Lua脚本执行过程大体分为三个主要步骤
- **（1）定义脚本函数**
    * ① 计算脚本<code>SHA1</code>文件校验和
    * ② 以<code>f_SHA1</code>格式构造脚本函数，函数体内容是脚本本身，如
```c
f_1as8dsdsjz8sww9(){
	return 1 + 2;
}
```
> **使用函数体来保存脚本的优点是：**
> - 调用简单，通过函数SHA1便可定位到
> - 可以利用函数的局部性特点让Lua环境保持清洁，减少了垃圾回收工作，也避免了全局变量
> - 可以提高脚本利用率，如果已经进行了脚本函数定义，那么之后的函数执行只需要进行SHA1调用即可，免去了重复性处理
>
- **（2）缓存脚本函数**
    * ③ 在<code>lua_scripts</code>字典中进行<code><SHA1,SCRIPT></code>映射关系存储
- **（3）执行脚本函数**
    * ④ 在执行脚本函数之前会将脚本参数进行加载
    * ⑤ 设置脚本执行超时的钩子，可以在执行<code>script skill</code>或<code>shutdown</code>终止脚本执行
    * ⑥ 执行脚本
    * ⑦ 移除脚本执行超时的钩子
    * ⑧ 保存脚本执行结果到客户端缓冲区
    * ⑨ Lua环境垃圾回收操作
## evalsha
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f1642bce30e44e7b4c3bba6fc0946dd~tplv-k3u1fbpfcp-zoom-1.image)
- 计算脚本SHA1校验和
- 拼接SHA1方法函数
- 校验是否存在函数
## script flush
清空<code>lua_scripts</code>字典中缓存的脚本，并且重置Lua环境
## script load
加载脚本到<code>lua_scripts</code>字典中
## script exists
校验脚本是否存在于<code>lua_scripts</code>字典中
## script kill
通过脚本钩子停止当前执行的脚本

# Lua与Redis命令区别
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/365359d17de047bebe3c002bd7167e6a~tplv-k3u1fbpfcp-zoom-1.image)
Lua相当于是一系列Redis命令的集合，由于运行在独立的Lua环境中，因此Lua脚本内容执行的命令是原子执行的，因此运行在单线程环境下，这与Redis原生命令相比客户端执行一系列命令时不会被其他客户端命令**插队**。
# Lua脚本优点
- Lua脚本在Redis中是原子执行的，执行过程中间不会插入其他命令。
- Lua脚本可以帮助开发和运维人员创造出自己定制的命令，并可以将这 些命令常驻在Redis内存中，实现复用的效果。
- Lua脚本可以将多条命令一次性打包，有效地减少网络开销
# 总结
- Lua脚本由于执行环境的天然隔离性，在执行过程中能对批量命令和复杂逻辑进行封装而拥有较好的原子性，这是对Redis事务机制的极大补充，为批执行的原子性操作提供了使用方式
- 由于Lua与Redis通信是采用了伪客户端传递形式，理论上执行性能要比Redis原生执行命令差，需要在实际使用中根据实际情况来进行取舍
# 参考
《Redis设计与实现》

https://www.runoob.com/lua/lua-tutorial.html
