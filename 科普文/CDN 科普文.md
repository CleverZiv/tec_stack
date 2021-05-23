## CDN是什么

- ### 概念

内容分发网络（Content Delivery Network，CDN）是建立并覆盖在承载网上，由不同区域的服务器组成的分布式网络。将源站资源缓存到全国各地的边缘服务器，供用户就近获取，降低源站压力，提升用户体验。

- 无CDN的情况

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103359799.png" alt="image-20210517103359799" style="zoom:50%;" />

- 有CDN的情况

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103421961.png" alt="image-20210517103421961" style="zoom:50%;" />

- ### 举例：阿里云CDN的分布概况

全球：2800+节点，中国大陆：2300+节点   （注：一个节点是一个机房，包含几十台至几百台服务器）

中国大陆：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103520604.png" alt="image-20210517103520604" style="zoom:50%;" />

海外、中国香港、中国澳门和中国台湾

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103548510.png" alt="image-20210517103548510" style="zoom:50%;" />



## CDN工作原理

- ### 用户的访问流程

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103610811.png" alt="image-20210517103610811" style="zoom:50%;" />

1. 当终端用户（北京）向www.a.com下的某资源发起请求时，首先向LDNS（本地DNS）发起域名解析请求。
2. LDNS检查缓存中是否有www.a.com的IP地址记录。如果有，则直接返回给终端用户；如果没有，则向授权DNS查询。
3. 当授权DNS解析www.a.com时，返回域名CNAME www.a.tbcdn.com对应IP地址。
4. 域名解析请求发送至DNS调度系统，并为请求分配最佳节点IP地址。
5. LDNS获取DNS返回的解析IP地址。
6. 用户获取解析IP地址。
7. 用户向获取的IP地址发起对该资源的访问请求。

- 如果该IP地址对应的节点已缓存该资源，则会将数据直接返回给用户，例如，图中步骤7和8，请求结束。

- 如果该IP地址对应的节点未缓存该资源，则节点向源站发起对该资源的请求。获取资源后，结合用户自定义配置的缓存策略，将资源缓存至节点，例如，图中的北京节点，并返回给用户，请求结束



上图中，网站开发者管理的模块是：（1）源站；（2）网站授权DNS；

CDN厂商管理的模块是：（1）DNS调度系统；（2）CDN节点设备



- ### 配置一个CDN加速域名

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103632376.png" alt="image-20210517103632376" style="zoom:50%;" />

申请成功之后，会得到一个CDN的CNAME域名，业务在自己的域名管理系统中，将www.a.com域名CNAME到该CDN的CNAME域名。

#### 什么是 CNAME

https://zhuanlan.zhihu.com/p/147723106

可以简单理解为是原本域名的一个别名，这个别名会由 DNS 调度系统管理，用于 CDN 网络通过 DNS 调度系统的智能分析，把这个 CNAME 指向一个（离用户地理位置最近的）CDN提供商的服务器IP，让用户就近取到想要的资源（如访问网站）



## CDN的主要模块

- ### CDN模块大图

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103652277.png" alt="image-20210517103652277" style="zoom:50%;" />

- 一级节点：最靠近用户的节点，节点数最多

- 二级、三级节点：为了减少回源次数，降低源站压力，采用多级节点；三级节点可以采用多线机房的方式（移动、电信、联通等）

- 调度系统：有DNS调度、302调度等方式，上图仅展示DNS调度；调度的作用：质量优化、容灾、成本优化；



- ### 单机房节点视图

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103715037.png" alt="image-20210517103715037" style="zoom:50%;" />

- LVS（四层负载均衡）：VIP、容灾、网卡性能优化

- Nginx（七层负载均衡）：Host管理、防盗链、URL封禁、一致性Hash到Cache、热点打散

- Cache：内容缓存、回源、流媒体处理

注：Nginx+Cache可以同机部署



## CDN接入层

### Nginx简介

Nginx是一款轻量级的Web服务器、反向代理服务器，它的内存占用少，启动极快，高并发能力强。

- #### Nginx多进程

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103732717.png" alt="image-20210517103732717" style="zoom:50%;" />

- Master进程：启动服务、停止服务、重新加载配置文件、平滑升级、管理worker进程（如worker挂掉后重新拉起）等。

- Worker进程：多个worker进程（例如与CPU数相等）处理请求，无阻塞、无休眠，负载几乎只与内存大小有关。



- #### Nginx的配置

典型的Nginx配置示例如下：

```
user tiger;

worker_processes 16;

events {

    use epoll;

    worker_connections 50000;

}



//以下是HTTP的配置

http { //main配置

    include mime.types;

    log_format main '$http_host $remote_addr [$time_local] $request_time '

    '"$request" $status $body_bytes_sent ......';



    upstream backend { //upstream配置

        server 127.0.0.1:8080;

    }



    gzip on;

    server { //server块配置

        listen 8020; //如果带default，则不带域名时，默认走此项

        server_name web toutiao.com *.toutiao.cn; //类似于TLB的域名组

        ......

        location /webstatic { //location块配置

            gzip off;

            ...

        }

        location /files {

            ...

            proxy_pass http://backend;

        }

        location ~ '\.m3u8' {

            access_by_lua_file 'lua/access.lua';  //ACCESS阶段的lua

            ...

            content_by_lua_file 'lua/file_process.lua'; //CONTENT阶段的lua

        }

    } 

}
```

由上面的配置可以看到，HTTP配置分为主要几大类：http主配置、upstream、server、location；分别配置在http{}、upstream{}、server{}、location{}，其中http、server、location有层次关系，同一个配置以哪个层次的为准，以该模块的代码实现为准。

Location的优先级关系，详见[官网](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)



- #### Nginx的模块

在Nginx中，除了少量的核心代码，其他一切皆为模块。ngx_module_t作为所有模块的通用接口，定义如下：

```
typedef struct ngx_module_s      ngx_module_t;

struct ngx_module_s {

    ngx_uint_t            ctx_index; //模块在分类模块中的序号

    ngx_uint_t            index; // 当前模块在ngx_modules数组中的序号

//......

    void                 *ctx;  //用于指向一类模块的上下文结构体，五种类型的模块，各自的上下文分别定义

    ngx_command_t        *commands; //配置文件中的指令

    ngx_uint_t            type; //模块的类型，5种：NGX_HTTP_MODULE、NGX_CORE_MODULE、NGX_CONF_MODULE、NGX_EVENT_MODULE、NGX_MAIL_MODULE

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);

    void                (*exit_thread)(ngx_cycle_t *cycle);

    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

//......

};
```

其中：init_master、init_module、init_process、init_therad、exit_thread、exit_process、exit_master这7个回调方法，定义了模块的初始化和退出。

type是模块的类型，Nginx有多层次、分类别的模块。Nginx的五大类型的模块：核心模块、配置模块、事件模块、HTTP模块、mail模块（NGX_CORE_MODULE、NGX_CONF_MODULE、NGX_EVENT_MODULE、NGX_HTTP_MODULE、NGX_MAIL_MODULE）。

commands用于处理nginx.conf中的配置项；

一段解析HTTP模块配置的代码（以ngx_http_header_filter_module为例），典型如下：

```
typedef struct {

    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);

    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);



    void       *(*create_main_conf)(ngx_conf_t *cf);

    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);



    void       *(*create_srv_conf)(ngx_conf_t *cf);

    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);



    void       *(*create_loc_conf)(ngx_conf_t *cf);

    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);

} ngx_http_module_t;  //HTTP模块类型的回调接口



static ngx_http_module_t  ngx_http_headers_filter_module_ctx = {

    NULL,                                  /* preconfiguration */

    ngx_http_headers_filter_init,          /* postconfiguration */



    NULL,                                  /* create main configuration */

    NULL,                                  /* init main configuration */



    NULL,                                  /* create server configuration */

    NULL,                                  /* merge server configuration */



    ngx_http_headers_create_conf,          /* create location configuration */

    ngx_http_headers_merge_conf            /* merge location configuration */

};



ngx_module_t  ngx_http_headers_filter_module = {

    NGX_MODULE_V1,

    &ngx_http_headers_filter_module_ctx,   /* module context */

    ngx_http_headers_filter_commands,      /* module directives */

    NGX_HTTP_MODULE,                       /* module type */

    NULL,                                  /* init master */

    NULL,                                  /* init module */

    NULL,                                  /* init process */

    NULL,                                  /* init thread */

    NULL,                                  /* exit thread */

    NULL,                                  /* exit process */

    NULL,                                  /* exit master */

    NGX_MODULE_V1_PADDING

};



......

//code: ngx_http.c

//初始化HTTP模块的main、server、location配置

    for (m = 0; ngx_modules[m]; m++) {

        if (ngx_modules[m]->type != NGX_HTTP_MODULE) { //判断是否HTTP模块

            continue;

        }



        module = ngx_modules[m]->ctx;

        mi = ngx_modules[m]->ctx_index;



        if (module->create_main_conf) { //解析http{}的主配置

            ctx->main_conf[mi] = module->create_main_conf(cf);

            if (ctx->main_conf[mi] == NULL) {

                return NGX_CONF_ERROR;

            }

        }



        if (module->create_srv_conf) { //解析server{}级别的配置

            ctx->srv_conf[mi] = module->create_srv_conf(cf);

            if (ctx->srv_conf[mi] == NULL) {

                return NGX_CONF_ERROR;

            }

        }



        if (module->create_loc_conf) { //解析location{}级别的配置

            ctx->loc_conf[mi] = module->create_loc_conf(cf);

            if (ctx->loc_conf[mi] == NULL) {

                return NGX_CONF_ERROR;

            }

        }

    }

......
```

header_filter模块的commands详见[官网](http://nginx.org/en/docs/http/ngx_http_headers_module.html)



Nginx的各个模块关系如下图

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103755224.png" alt="image-20210517103755224" style="zoom:50%;" />





- #### 事件模型

请求的多阶段异步处理（Nginx高性能的由来，区别于apache httpd多进程模型），以请求本地静态文件为例，事件的收集和分发如下图所示：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103813853.png" alt="image-20210517103813853" style="zoom:50%;" />

- 事件循环：worker进程循环地收集、分发事件。

- 异步和分阶段：每个事件消费者都不能有阻塞行为。如果某个处理是阻塞的，则必须拆分成多个顺序的子阶段来处理。



- #### HTTP的处理阶段

Nginx的模块化设计，使得每个HTTP模块可以仅专注于完成一个独立的、简单的功能。一个HTTP请求的完整处理过程可以由无数个HTTP模块共同合作完成。多个HTTP模块流水式的处理同一请求时，单一的顺序无法满足灵活性的需求，每个正在处理请求的模块无法灵活、有效地指定下一个HTTP处理模块是哪一个。因此，Nginx的HTTP框架将HTTP处理流程划分为11个阶段，这11个阶段有些是必备的、有些是可选的，也可以有多个HTTP模块同时介入同一阶段（这时，将会在一个阶段中按照这些HTTP模块的ctx_index顺序来依次执行它们提供的handler处理方法），11个阶段如下所示：

```
 typedef enum {

    //在接收到完整的HTTP头部后处理的HTTP阶段

    NGX_HTTP_POST_READ_PHASE=0,



    /*在还没有查询到URI匹配的location前，这时rewrite重写URL也作为一个独立的HTTP阶段*/

    NGX_HTTP_SERVER_REWRITE_PHASE,



    /*根据URI寻找匹配的location，这个阶段通常由ngx_http_core_module模块实现，不建议其他HTTP模块重新定义这一阶段的行为*/

    NGX_HTTP_FIND_CONFIG_PHASE,



    /*在NGX_HTTP_FIND_CONFIG_PHASE阶段之后重写URL的意义与NGX_HTTP_SERVER_REWRITE_ PHASE阶段显然是不同的，因为这两者会导致查找到不同的location块（location是与URI进行匹配的）*/

    NGX_HTTP_REWRITE_PHASE,



    /*这一阶段是用于在rewrite重写URL后重新跳到NGX_HTTP_FIND_CONFIG_PHASE阶段，找到与新的URI匹配的location。所以，这一阶段是无法由第三方HTTP模块处理的，而仅由ngx_http_core_module模块使用*/

    NGX_HTTP_POST_REWRITE_PHASE,



    //处理NGX_HTTP_ACCESS_PHASE阶段前，HTTP模块可以介入的处理阶段

    NGX_HTTP_PREACCESS_PHASE,



    /*这个阶段用于让HTTP模块判断是否允许这个请求访问Nginx服务器

    NGX_HTTP_ACCESS_PHASE,



    /*当NGX_HTTP_ACCESS_PHASE阶段中HTTP模块的handler处理方法返回不允许访问的错误码时（实际是NGX_HTTP_FORBIDDEN或者NGX_HTTP_UNAUTHORIZED），这个阶段将负责构造拒绝服务的用户响应。所以，这个阶段实际上用于给NGX_HTTP_ACCESS_PHASE阶段收尾*/

    NGX_HTTP_POST_ACCESS_PHASE,



    /*这个阶段完全是为了try_files配置项而设立的。当HTTP请求访问静态文件资源时，try_files配置项可以使这个请求顺序地访问多个静态文件资源，如果某一次访问失败，则继续访问try_files中指定的下一个静态资源。另外，这个功能完全是在NGX_HTTP_TRY_FILES_PHASE阶段中实现的*/

    NGX_HTTP_TRY_FILES_PHASE,



    //用于处理HTTP请求内容的阶段，这是大部分HTTP模块最喜欢介入的阶段

    NGX_HTTP_CONTENT_PHASE,



    /*处理完请求后记录日志的阶段。例如，ngx_http_log_module模块就在这个阶段中加入了一个handler处理方法，使得每个HTTP请求处理完毕后会记录access_log日志*/

    NGX_HTTP_LOG_PHASE



} ngx_http_phases;
```

注：上述标红的是Lua中经常介入的阶段，对应的Lua指令为：access_by_lua_file、rewrite_by_lua_file、content_by_lua_file；



### Nginx与周边模块的交互

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103834935.png" alt="image-20210517103834935" style="zoom:50%;" />



- HTTPS：用户如果通过Https访问，需要经过https_nginx模块先去除https

- Config加载：用户从开放云CDN配置的内容，主要包括：
  - 用户加速的host

- 用户的源站域名或IP

- 是否使用视频拖拽

- 缓存策略：对指定的路径或后缀名设置缓存TTL

- 防盗链配置：Referer、User-Agent等

- Lua：主要处理url、配置解析、url鉴权等

- 开放云CDN配置示例（增加域名不需要重启Nginx）

```
upstream config_center {

    server x.x.x.x:9090;

}

upstream cache_server {

    server x.x.x.x:8060; //多个Cache后端IP

}



server {

    listen 6000 default;//需要配置成default，以便服务所有用户的域名

    ......

    location / {

        set $upstream_addr '';

        ......

        rewrite_by_lua_file 'lua/rewrite.lua'; //读共享内存中的配置，从大量的域名配置中，找出当前域名配置，并在这里把$upstream_addr配置上；如果有鉴权，也在这里实现；

        proxy_set_header X-Upstream-Addr $upstream_addr;

        proxy_pass http://$cache_server;

    }



    location /update_config { //这里由外部定时请求探测触发，获取用户的配置，写到共享内存中

        ...

        proxy_pass http://config_center;

    }

}
```

### Nginx在CDN中的主要功能

#### 一致性Hash upstream到Cache

一致性hash的结构如下图所示：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103854339.png" alt="image-20210517103854339" style="zoom:50%;" />

使用一致性Hash的好处：

- 提高空间利用率，同一个文件在一个节点只存一份

- 提高命中率



#### 后端健康检查

对后端Cache做健康检查，当后端某台Cache出现故障时，自动踢除该机器，保证可用性。



#### 鉴权、URL封禁

- 鉴权方式1：使用用户提供的鉴权库

方案的实现：用户提供一个静态编译的库（.a文件、.h文件），在Nginx中开发一个模块，将用户的库编译进去，并对外提供一个Lua接口。在Lua中调用该接口，传入Url、及从Url中提取的各项参数、解密key等，检测该url是否鉴权通过。



- 鉴权方式2：用户指定鉴权算法，在Lua中实现该算法

用户将鉴权信息（一般包含过期时间等）编码到url中，并提供解码算法和Key，在Lua中解码该url，获得鉴权信息进行校验是否符合要求。



- URL封禁

对于外网发现的违规URL，添加到封禁库中，在Nginx中做封禁处理

#### 限速

根据用户的限速需求，在Nginx做相应的限速处理

#### 用户的特殊需求

- 异步打击系统

- 按视频码率限速

- 等等

## CDN Cache

### 什么是好的CDN Cache

- 功能
  - 尽可能完整地支持http协议

- 尽可能地节省原站带宽

- 可扩展（支持模块开发，如：多种回源方式、流媒体等）

- 性能
  - 响应时间与负载之间呈线性关系，线性常数尽量低

- 使用多核

- 可运维性
  - 方便升级

- 配置简洁

- 自动处理磁盘异常

### Cache的总体结构

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103913448.png" alt="image-20210517103913448" style="zoom:50%;" />

- Master，worker，io都是线程

- 都有独立的epoll循环

- 彼此之间通过管道进行通信



### Master线程

#### Server hash的管理

每个Server对应一个端口，对应于一个缓存业务，单独分配一个Server Hash用于管理缓存item；Server Hash的结构如下图：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103933162.png" alt="image-20210517103933162" style="zoom:50%;" />

- 每个Server监听一个独立的端口

- Key的构成：host、uri、query_string、method、accept_encoding、part_index等，根据用户的配置决定取其中的若干个，然后计算md5；

- 当Hash的条数过大时，由Small Hash分裂为Big Hash，Hash条目数翻倍；

- 每个item是一个指向磁盘位置的结构体（元数据），含path_id、offset、size等，直接写块设备；

- 每个item是一个文件分片，分片大小最大为1MB；

- 每个item记录一个ref_cnt，用于合并回源的引用计数；



#### Store的管理

- Store的结构如下图

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517103950797.png" alt="image-20210517103950797" style="zoom:50%;" />

- Store: Store是一组磁盘的集合，可以按ssd、hdd分为两个Store；

- Path：Path是一块磁盘，分配一个path_id，可以按LRU淘汰Path中的item；

- 每个Server只能配置使用一种Store;

- Server与Store相互有指针可以找到对方，当Store发生踢盘时能通知到Server，当Server从配置中去掉时能通知到Store；



- PATH的空间管理

  <img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517104050134.png" alt="image-20210517104050134" style="zoom:50%;" />

- 初始状态下划分为若干个8GB大小的FreeBlock

- PATH以2KB为单位寻址，节省索引内存使用量

- 回源的数据根据大小从FreeBlock中切分以2KB整数倍的item（2KB-1MB），并串到Path的LRU链表中

- 数据回收时，将item串回到FreeBlock table中，按2KB的整数倍进行分类，回收的空间可以重复利用；

#### 运行状态管理

- Cache的运行状态

| **名称**       | **运行阶段**                          |
| -------------- | ------------------------------------- |
| 加载配置模式   | 启动时                                |
| 加载元数据模式 | 加载完配置文件时                      |
| 并行只读模式   | restart后新、老进程并行服务时（30秒） |
| 正常服务模式   | 正常服务时                            |
| 退出准备1      | 收到restart命令时                     |
| 退出准备2      | dump完元数据后                        |
| 硬退出         | 收到hardquit命令并dump完元数据后      |

- Restart过程

同一个服务端口，多个worker通过REUSEPORT的方式监听和绑定，老进程退出时，新进程仍能使用老进程的fd来监听，平滑Restart过程如下：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517104108551.png" alt="image-20210517104108551" style="zoom:50%;" />



### Work线程

#### Work线程事件循环

- 抢锁情况下的事件循环

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517104125989.png" alt="image-20210517104125989" style="zoom:50%;" />

- 每个端口对应一个socket fd

- 各个worker线程通过抢锁将socket fd放到自己的epoll中，没抢到锁的线程将socket fd从自己的epoll中删除



- REUSEPORT情况下的事件循环

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517104142764.png" alt="image-20210517104142764" style="zoom:50%;" />

- 每个端口对应多个socket fd，每个worker线程将各自的socket fd放入epoll中

- 内核决定哪个socket fd收到epoll事件



#### 请求的处理

HTTP请求主要有以下模块

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517104157810.png" alt="image-20210517104157810" style="zoom:50%;" />



#### 回源的处理

HTTP回源主要有以下模块

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517104214352.png" alt="image-20210517104214352" style="zoom:50%;" />

#### 多媒体处理

- 视频拖拽：根据用户请求的视频时间范围，根据视频文件中的索引将原视频重新封装成新的视频，并响应给用户。例如，用户请求为：http://a.com/video/a.mp4?start=100&end=200，输出是一个起始为100秒，终止为200秒时刻的视频。



- MP4容器：MP4以Box来组织其元数据和媒体数据，一个MP4的Box结构如下图所示

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210517104229809.png" alt="image-20210517104229809" style="zoom:50%;" />



- CDN对视频的处理
  - 回源处理：元数据必须全部缓存，媒体数据根据元数据计算出Range范围，按缓存块大小回源。

- 请求解析和流媒体封装：解析用户的拖拽参数（start、end参数），计算出Range范围，从缓存中读取对应的文件块或从源站回源缺失的片。将索引和媒体数据重新组织并响应给用户。



### Cache其他模块

- IO线程
  - 每个path分配3个io线程，处理worker线程的数据读写请求

- 模块化设计
  - 支持插件式开发，通过HOOK点实现模块功能



### Cache有哪些坑

1. item_key没有记录url，用md5作为key，无法按url前缀删除一批item。（采取了补丁式方案）
2. 按Server来配置使用SSD、HDD，不易扩展分级存储。
3. 多重异步回调，代码流程不连贯，可读性不太好。



## 开放云CDN

传统CDN厂商与互联网CDN有哪些差别？

- 互联网CDN与自有业务结合，成本控制灵活

- 与业务配合提升CDN性能

- CDN与对象存储结合

- 灵活的使用和接入方式



## CDN其他模块和技术

- DNS调度、302调度

DNS调度

302调度

- IP库

- 机器监控

- 用户平台

- 日志和计费系统

- 运维管理平台

- 动态加速

- 预充系统

- 删除系统



## 参考资料

扩展阅读：朱照远-[阿里云CDN技述演进之路](https://wenku.baidu.com/view/63989efc71fe910ef02df86a.html)

扩展阅读：白金-[《一堂TCP加速与CDN陨坑的白金课程》](https://mp.weixin.qq.com/s?__biz=MzA4MDY3MjMyNQ==&mid=2651026071&idx=1&sn=948de5891f8635e5979ff4f03fd8e9f0&scene=2&srcid=07082Lqo0kXDRowBGZhVHQHB&from=timeline&isappinstalled=0#wechat_redirect)

书：《深入理解Nginx：模块开发与架构解析》

[视频编解码分享.pptx](https://bytedance.feishu.cn/space/file/boxcnz7684yp2J53gU2gZrgAT2b) 

[ISO/IEC 14496-12 MP4 Format](http://l.web.umkc.edu/lizhu/teaching/2016sp.video-communication/ref/mp4.pdf)