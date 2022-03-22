数据库查询中通常会用到各种`timeout`配置，现在来看下常见的有哪些。


- 建立链接池的Connection
    - connectTimeout
mysql官方文档：`Timeout for socket connect (in milliseconds), with 0 being no timeout.`
建立socket链接时使用，单位毫秒，默认值为0，表示无限等待（Linux中提供了操作系统级timeout）
    - socketTimeout
mysql官方文档：`Timeout (in milliseconds) on network socket operations (0, the default means no timeout).`
执行socket操作（socket read）时使用，单位毫秒，默认值为0，表示无限等待（Linux中提供了操作系统级timeout）

        > connectTimeout和socketTimeout的区别：
connectTimeout是建立TCP连接的时间，包括三次握手等；
socketTimeout是TCP连接请求响应（执行sql）的时间，应该设置的比最长的sql执行时间更长一些。
    - loginTimeout
数据库驱动的等待时长，单位秒，默认值为0，表示无限等待。
- 从连接池获取Connection的等待时间
    - Hikari中`connectionTimeout`，默认30s，最小250ms。
    - DBCP中`maxWaitMillis`，默认一直等待（-1）。
- 校验Connection
    - Hikari中`validationTimeout`，默认5s，最小250ms。
    - DBCP中`validationQueryTimeout`。默认一直等待。
- 执行一条SQL的时间，包括读SQL结果

    > socketTimeout控制所有statement的执行时间，该指标表示特定Statement的执行时间

    - DBCP中`queryTimeout`，默认为nul
    - HIkariCP中不支持。



> 参考链接：
[https://brightinventions.pl/blog/database-timeouts/#init](https://brightinventions.pl/blog/database-timeouts/#init)[https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-connp-props-connection-authentication.html](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-connp-props-connection-authentication.html)