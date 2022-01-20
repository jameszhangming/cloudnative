# Nginx执行步骤

Nginx处理每一个用户请求时，都是按照若干个不同阶段(phase)依次处理的，而不是根据配置文件上的顺序。

Nginx处理请求的过程一共划分为11个阶段，按照执行顺序依次是post-read、server-rewrite、find-config、rewrite、post-rewrite、 preaccess、access、post-access、try-files、content、log。

**1、post-read**
读取请求内容阶段，nginx读取并解析完请求头之后就立即开始运行；例如模块 ngx_realip 就在 post-read 阶段注册了处理程序，它的功能是迫使 Nginx 认为当前请求的来源地址是指定的某一个请求头的值。

**2、server-rewrite**
server请求地址重写阶段；当ngx_rewrite模块的set配置指令直接书写在server配置块中时，基本上都是运行在server-rewrite 阶段

**3、find-config**
配置查找阶段，这个阶段并不支持Nginx模块注册处理程序，而是由Nginx核心来完成当前请求与location配置块之间的配对工作。

**4、rewrite**
location请求地址重写阶段，当ngx_rewrite指令用于location中，就是再这个阶段运行的；另外ngx_set_misc(设置md5、encode_base64等)模块的指令，还有ngx_lua模块的set_by_lua指令和rewrite_by_lua指令也在此阶段。

**5、post-rewrite**
请求地址重写提交阶段，当nginx完成rewrite阶段所要求的内部跳转动作，如果rewrite阶段有这个要求的话；

**6、preaccess**
访问权限检查准备阶段，ngx_limit_req和ngx_limit_zone在这个阶段运行，ngx_limit_req可以控制请求的访问频率，ngx_limit_zone可以控制访问的并发度；

**7、access**
访问权限检查阶段，标准模块ngx_access、第三方模块ngx_auth_request以及第三方模块ngx_lua的access_by_lua指令就运行在这个阶段。配置指令多是执行访问控制相关的任务，如检查用户的访问权限，检查用户的来源IP是否合法；

**8、post-access**
访问权限检查提交阶段；主要用于配合access阶段实现标准ngx_http_core模块提供的配置指令satisfy的功能。satisfy all(与关系),satisfy any(或关系)

**9、try-files**
配置项try_files处理阶段；专门用于实现标准配置指令try_files的功能,如果前 N-1 个参数所对应的文件系统对象都不存在，try-files 阶段就会立即发起“内部跳转”到最后一个参数(即第 N 个参数)所指定的URI.

**10、content**
内容产生阶段，是所有请求处理阶段中最为重要的阶段，因为这个阶段的指令通常是用来生成HTTP响应内容并输出 HTTP 响应的使命；

**11、log**
日志模块处理阶段；记录日志

# ngx_lua运行指令

ngx_lua属于nginx的一部分，它的执行指令都包含在nginx的11个步骤之中了，不过ngx_lua并不是所有阶段都会运行的；

**1、init_by_lua、init_by_lua_file**
运行阶段：loading-config
当nginx master进程在加载nginx配置文件时运行指定的lua脚本，通常用来注册lua的全局变量或在服务器启动时预加载lua模块：

**2、init_worker_by_lua、init_worker_by_lua_file**
运行阶段：starting-worker
在每个nginx worker进程启动时调用指定的lua代码。如果master 进程不允许，则只会在init_by_lua之后调用。

**3、set_by_lua、set_by_lua_file**
运行阶段：rewrite
传入参数到指定的lua脚本代码中执行，并得到返回值到res中。中的代码可以使从ngx.arg表中取得输入参数(顺序索引从1开始)。

这个指令是为了执行短期、快速运行的代码因为运行过程中nginx的事件处理循环是处于阻塞状态的。耗费时间的代码应该被避免。

**4、rewrite_by_lua、rewrite_by_lua_file**
运行阶段：rewrite tail
作为rewrite阶段的处理，为每个请求执行指定的lua代码。注意这个处理是在标准HtpRewriteModule之后进行的

**5、access_by_lua，access_by_lua_file**
运行阶段：access tail
为每个请求在访问阶段的调用lua脚本进行处理。主要用于访问控制，能收集到大部分的变量。

**6、content_by_lua，content_by_lua_file**
运行阶段：content
作为”content handler”为每个请求执行lua代码，为请求者输出响应内容。

**7、header_filter_by_lua，header_filter_by_lua_file**
运行阶段：output-header-filter

一般用来设置cookie和headers。

**8、body_filter_by_lua，body_filter_by_lua_file**
运行阶段：output-body-filter
输入的数据时通过ngx.arg[1](https://www.hi-linux.com/posts/作为lua的string值)，通过ngx.arg[2]这个bool类型表示响应数据流的结尾。

**9、log_by_lua，log_by_lua_file**
运行阶段：log
在log阶段调用指定的lua脚本，并不会替换access log，而是在那之后进行调用。



