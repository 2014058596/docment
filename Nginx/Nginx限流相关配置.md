### Nginx限流相关配置

#### 为什么需要限流

​	防止外部暴力扫描，减少密码被暴力破解的可能性，也可以解决流量突发问题，保护服务器不会因为承受不住一时刻的大量请求而宕机

- ##### 限制访问频率

  - [ngx_http_limit_req_module](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html) 模块提供限制请求处理速率能力，使用了漏桶算法(leaky bucket)。下面例子使用 nginx limit_req_zone 和 limit_req 两个指令，限制单个IP的请求处理速率

  - > 格式：limit_req_zone key zone rate

    ```nginx
    http {
        limit_req_zone $binary_remote_addr zone=myRateLimit:10m rate=10r/s;
       	server {
            location / {
                limit_req zone=myRateLimit burst=20 nodelay;
                proxy_pass http://my_upstream;
            }
    	}	 
    }
    ```

    - **key** ：定义限流对象，binary_remote_addr 是一种key，表示基于 remote_addr(客户端IP) 来做限流，binary_ 的目的是压缩内存占用量。
    - **zone**：定义共享内存区来存储访问信息， myRateLimit:10m 表示一个大小为10M，名字为myRateLimit的内存区域。1M能存储16000 IP地址的访问信息，10M可以存储16W IP地址访问信息。
    - **rate**： 用于设置最大访问速率，rate=10r/s 表示每秒最多处理10个请求。Nginx 实际上以毫秒为粒度来跟踪请求信息，因此 10r/s 实际上是限制：每100毫秒处理一个请求。这意味着，自上一个请求处理完后，若后续100毫秒内又有请求到达，将拒绝处理该请求
    - **burst** 译为突发、爆发，表示在超过设定的处理速率后能额外处理的请求数。当 **rate=10r/s** 时，将1s拆成10份，即每100ms可处理1个请求。
    - **nodelay** 针对的是 burst 参数，burst=20 nodelay 表示这20个请求立马处理，不能延迟，相当于特事特办。不过，即使这20个突发请求立马处理结束，后续来了请求也不会立马处理。burst=20 相当于缓存队列中占了20个坑，即使请求被处理了，这20个位置这只能按 100ms一个来释放

- ##### 限制并发连接数

  - [ngx_http_limit_conn_module](http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html) 提供了限制连接数的能力，利用 **limit_conn_zone** 和 **limit_conn** 两个指令即可。下面是 Nginx 官方例子：

    ```
    http {
        limit_conn_zone $binary_remote_addr zone=perip:10m;
        limit_conn_zone $server_name zone=perserver:10m;
    
        server {
            ...
            limit_conn perip 10;
            limit_conn perserver 100;
        }
    }
    ```