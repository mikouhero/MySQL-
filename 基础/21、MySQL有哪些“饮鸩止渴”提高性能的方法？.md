#  21 |  MySQL有哪些“饮鸩止渴”提高性能的方法？

### 短链接风暴
- 短连接模式： 连接到数据库后，执行很少的SQL 语句就断开 下次需要的时候再重新连接，如果使用的是短连接，在业务高峰期的时候，就可能出现连接数突然暴涨的情况。
- 风险： 一旦数据库处理得慢一些，连接数就会暴涨； max_connections 参数，用来控制一个MySQL 实例同时存在的连接数的上限， 超过这个值 提示 “Too many connections ”
> 碰到这种情况时，一个比较自然的想法，就是调高 max_connections 的值。但这样做是有风险  因为设计 max_connections 这个参数的目的是想保护 MySQL，如果我们把它改得太大，让更多的连接都可以进来，那么系统的负载可能会进一步加大，量的资源耗费在权限验证等逻辑上，结果可能是适得其反，已经连接的线程拿不到 CPU 资源去执行业务的 SQL 请求   ？？？

####  先处理掉那些占着连接但是不工作的线程。

max_connections 的计算，不是看谁在 running，是只要连着就占用一个计数位置。对于那些不需要保持的连接，我们可以通过 kill connection 主动踢掉。这个行为跟事先设置
wait_timeout 的效果是一样的。设置 wait_timeout 参数表示的是，一个线程空闲
wait_timeout 这么多秒之后，就会被 MySQL 直接断开连接。

#### 减少连接过程的消耗。
有的业务代码会在短时间内先大量申请数据库连接做备用，，如果现在数据库确认是被连接行为打挂了，那么一种可能的做法，，是让数据库跳过权限验证阶段。
跳过权限验证的方法是：重启数据库，并使用–skip-grant-tables 参数启动。这样，整个 MySQL 会跳过所有的权限验证阶段，包括连接过程和语句执行过程在内。


除了短连接数暴增可能会带来性能问题外，实际上，，我们在线上碰到更多的是查询或者更新语句导致的性能问题。。其中，查询问题比较典型的有两类，一类是由新出现的慢查询导致的，一类是由 QPS（每秒查询数）突增导致的。

#### 慢查询性能问题

- 导致慢查询的第一种可能是，索引没有设计好。  
>这种场景一般就是通过紧急创建索引来解决。MySQL 5.6 版本以后，创建索引都支持 Online 对于那种高峰期数据库已经被这个语句打挂了的情况，最高效的做法就是直接执行 alter table 语句。  

比较理想的是能够在备库先执行。假设你现在的服务是一主一备，主库 A、备库 B，这个方案的大致流程是这样的：  

1、在备库 B 上执行 set sql_log_bin=off，也就是不写 binlog，然后执行 alter table 语句加上索引；
2、执行主备切换；
3、这时候主库是 B，备库是 A。在 A 上执行set sql_log_bin=off 然后执行 alter table 语句加上索引。
这是一个“古老”的 DDL 方案。平时在做变更的时候，你应该考虑类似 gh-ost 这样的方案，更加稳妥。但是在需要紧急处理时，上面这个方案的效率是最高的。

- 导致慢查询的第二种可能是，语句没写好。

- QPS 突增的问题

有时候由于业务突然出现高峰，或者应用程序 bug，导致某个语句的 QPS 突然暴涨，也可能导致 MySQL 压力过大，影响服务

















