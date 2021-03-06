
.. _design:

========
架构设计
========

.. _intro:

--------
设计概述
--------

EMQ X 开源MQTT消息服务器在1.x版本的基础上，首先分离前端协议(FrontEnd)与后端集成(Backend)，其次分离了消息路由平面(Flow Plane)与监控管理平面(Monitor/Control Plane)。EMQ 2.0消息服务器将在稳定支持100万MQTT连接的基础上，向可管理可监控坚如磐石的稳定性方向迭代演进:

.. image:: _static/images/design_1.png

EMQ X 在开源EMQ X 2.0版本基础上，大幅改进系统集群设计，采用Scalable RPC机制，分离节点间的集群与数据转发通道，以支持更稳定的节点集群与更高性能的消息路由。

EMQ X 企业版在Backend端支持MQTT消息数据存储Redis、MySQL、PostgreSQL、MongoDB、Cassandra多种数据库，支持桥接转发MQTT消息到Kafka、RabbitMQ企业消息中间件。

100万连接
---------

多核服务器和现代操作系统内核层面，可以很轻松支持100万TCP连接，核心问题是应用层面如何处理业务瓶颈。

EMQ消息服务器在业务和应用层面，解决了承载100万连接的各类瓶颈问题。连接测试的操作系统内核、TCP协议栈、Erlang虚拟机参数: http://docs.emqtt.cn/zh_CN/latest/tune.html

全异步架构
----------

EMQ消息服务器是基于Erlang/OTP平台的全异步的架构：异步TCP连接处理、异步主题(Topic)订阅、异步消息发布。只有在资源负载限制部分采用同步设计，比如TCP连接创建和Mnesia数据库事务执行。

一条MQTT消息从发布者(Publisher)到订阅者(Subscriber)，在EMQ X消息服务器内部异步流过一系列Erlang进程Mailbox:

.. image:: _static/images/design_2.png

消息持久化
----------

EMQ 1.0版本不支持服务器内部消息持久化，这是一个架构设计选择。首先，EMQ解决的核心问题是连接与路由；其次，我们认为内置持久化是个错误设计。

传统内置消息持久化的MQ服务器，比如广泛使用的JMS服务器ActiveMQ，几乎每个大版本都在重新设计持久化部分。内置消息持久化在设计上有两个问题:

1. 如何平衡内存与磁盘使用？消息路由基于内存，消息存储是基于磁盘。

2. 多服务器分布集群架构下，如何放置Queue？如何复制Queue的消息？

Kafka在上述问题上，做出了正确的设计：一个完全基于磁盘分布式commit log的消息服务器。

EMQ X 企业版本支持消息持久化到Redis、MySQL、PostgreSQL、MongoDb、Cassandra等数据库或Kafka。

设计上分离消息路由与消息存储职责后，数据复制容灾备份甚至应用集成，可以在数据层面灵活实现。

NetSplit问题
------------

EMQ 1.0消息服务器集群，基于Mnesia数据库设计。NetSplit发生时，节点间状态是：Erlang节点间可以连通，互相询问自己是否宕机，对方回答你已经宕机:(

NetSplit故障发生时，EMQ X消息服务器的log/emqx_error.log日志，会打印critical级别日志::

    Mnesia inconsistent_database event: running_partitioned_network, emqx@host

EMQ集群部署在同一IDC网络下，NetSplit发生的几率很低，一旦发生又很难自动处理。所以EMQ .0版本设计选择是，集群不自动化处理NetSplit，需要人工重启部分节点。

.. _architecture:

--------
系统架构
--------

概念模型
--------

EMQ X 消息服务器概念上更像一台网络路由器(Router)或交换机(Switch)，而不是传统的企业级消息服务器(MQ)。相比网络路由器按IP地址或MPLS标签路由报文，EMQ X 按主题树(Topic Trie)发布订阅模式在集群节点间路由MQTT消息:

.. image:: ./_static/images/design_3.png

设计原则
--------

1. EMQ X 消息服务器核心解决的问题：处理海量的并发MQTT连接与路由消息。

2. 充分利用Erlang/OTP平台软实时、低延时、高并发、分布容错的优势。

3. 连接(Connection)、会话(Session)、路由(Router)、集群(Cluster)分层。

4. 消息路由平面(Flow Plane)与控制管理平面(Control Plane)分离。

5. 支持后端数据库或NoSQL实现数据持久化、容灾备份与应用集成。

系统分层
--------

1. 连接层(Connection Layer)： 负责TCP连接处理、MQTT协议编解码。

2. 会话层(Session Layer)：处理MQTT协议发布订阅消息交互流程。

3. 路由层(Route Layer)：节点内路由派发MQTT消息。

4. 分布层(Distributed Layer)：分布节点间路由MQTT消息。

5. 认证与访问控制(ACL)：连接层支持可扩展的认证与访问控制模块。

6. 钩子(Hooks)与插件(Plugins)：系统每层提供可扩展的钩子，支持插件方式扩展服务器。

.. _connection_layer:

----------
连接层设计
----------

连接层处理服务端Socket连接与MQTT协议编解码：

1. 基于 `eSockd`_ 框架的异步TCP服务端
2. TCP Acceptor池与异步TCP Accept
3. TCP/SSL, WebSocket/SSL连接支持
4. 最大并发连接数限制
5. 基于IP地址(CIDR)访问控制
6. 基于Leaky Bucket的流控
7. MQTT协议编解码
8. MQTT协议心跳检测
9. MQTT协议报文处理

.. _session_layer:

----------
会话层设计
----------

会话层处理MQTT协议发布订阅(Publish/Subscribe)业务交互流程：

1. 缓存MQTT客户端的全部订阅(Subscription)，并终结订阅QoS

2. 处理Qos0/1/2消息接收与下发，消息超时重传与离线消息保存

3. 飞行窗口(Inflight Window)，下发消息吞吐控制与顺序保证

4. 保存服务器发送到客户端的，已发送未确认的Qos1/2消息

5. 缓存客户端发送到服务端，未接收到PUBREL的QoS2消息

6. 客户端离线时，保存持久会话的离线Qos1/2消息

消息队列与飞行窗口
------------------

会话层通过一个内存消息队列和飞行窗口处理下发消息:

.. image:: _static/images/design_4.png

飞行窗口(Inflight Window)保存当前正在发送未确认的Qos1/2消息。窗口值越大，吞吐越高；窗口值越小，消息顺序越严格。

当客户端离线或者飞行窗口(Inflight Window)满时，消息缓存到队列。如果消息队列满，先丢弃Qos0消息或最早进入队列的消息。

报文Id与消息Id
--------------

MQTT协议定义了一个16bits的报文ID(PacketId)，用于客户端到服务器的报文收发与确认。MQTT发布报文(PUBLISH)进入消息服务器后，转换为一个消息对象并分配128bits消息ID(MessageId)。

全局唯一时间序列消息ID结构：

.. image:: _static/images/design_5.png

1. 64bits时间戳: erlang:system_time if Erlang >= R18, otherwise os:timestamp
2. Erlang节点ID: 编码为2字节
3. Erlang进程PID: 编码为4字节
4. 进程内部序列号: 2字节的进程内部序列号

端到端消息发布订阅(Pub/Sub)过程中，发布报文ID与报文QoS终结在会话层，由唯一ID标识的MQTT消息对象在节点间路由:

.. image:: _static/images/design_6.png

.. _route_layer:

----------
路由层设计
----------

路由层维护订阅者(Subscriber)与订阅关系表(Subscription)，并在本节点发布订阅模式派发(Dispatch)消息:

.. image:: ./_static/images/design_7.png

消息派发到会话(Session)后，由会话负责按不同QoS送达消息。

.. _distributed_layer:

----------
分布层设计
----------

分布层维护全局主题树(Topic Trie)与路由表(Route Table)。主题树由通配主题构成，路由表映射主题到节点:

.. image:: ./_static/images/design_8.png

分布层通过匹配主题树(Topic Trie)和查找路由表(Route Table)，在集群的节点间转发路由MQTT消息:

.. image:: ./_static/images/design_9.png

.. _hook:

--------------
钩子(Hook)设计
--------------

钩子(Hook)定义
--------------

*EMQ X* 消息服务器在客户端上下线、主题订阅、消息收发位置设计了扩展钩子(Hook):

+----------------------+----------------------+
|         钩子         |         说明         |
+======================+======================+
| client.authenticate  | 客户端认证           |
+----------------------+----------------------+
| client.check_acl     | 客户端 ACL 检查      |
+----------------------+----------------------+
| client.connected     | 客户端上线           |
+----------------------+----------------------+
| client.subscribe     | 客户端订阅主题前     |
+----------------------+----------------------+
| client.unsubscribe   | 客户端取消订阅主题   |
+----------------------+----------------------+
| session.subscribed   | 客户端订阅主题后     |
+----------------------+----------------------+
| session.unsubscribed | 客户端取消订阅主题后 |
+----------------------+----------------------+
| message.publish      | MQTT 消息发布        |
+----------------------+----------------------+
| message.deliver      | MQTT 消息投递前      |
+----------------------+----------------------+
| message.acked        | MQTT 消息回执        |
+----------------------+----------------------+
| client.disconnected  | 客户端连接断开       |
+----------------------+----------------------+

钩子(Hook) 采用职责链设计模式(`Chain-of-responsibility_pattern`_)，扩展模块或插件向钩子注册回调函数，系统在客户端上下线、主题订阅或消息发布确认时，触发钩子顺序执行回调函数:

.. image:: ./_static/images/design_10.png

不同钩子的回调函数输入参数不同，用户可参考插件模版的 `emqx_plugin_template`_ 模块，每个回调函数应该返回:

+----------------+----------------------+
|      返回      |         说明         |
+================+======================+
| ok             | 继续执行             |
+----------------+----------------------+
| {ok, NewAcc}   | 返回累积参数继续执行 |
+----------------+----------------------+
| stop           | 停止执行             |
+----------------+----------------------+
| {stop, NewAcc} | 返回累积参数停止执行 |
+----------------+----------------------+

钩子(Hook)实现
--------------

emqx 模块封装了 Hook 接口:

.. code-block:: erlang

    -spec(hook(emqx_hooks:hookpoint(), emqx_hooks:action()) -> ok | {error, already_exists}).
    hook(HookPoint, Action) ->
        emqx_hooks:add(HookPoint, Action).

    -spec(hook(emqx_hooks:hookpoint(), emqx_hooks:action(), emqx_hooks:filter() | integer())
        -> ok | {error, already_exists}).
    hook(HookPoint, Action, Priority) when is_integer(Priority) ->
        emqx_hooks:add(HookPoint, Action, Priority);
    hook(HookPoint, Action, Filter) when is_function(Filter); is_tuple(Filter) ->
        emqx_hooks:add(HookPoint, Action, Filter);
    hook(HookPoint, Action, InitArgs) when is_list(InitArgs) ->
        emqx_hooks:add(HookPoint, Action, InitArgs).

    -spec(hook(emqx_hooks:hookpoint(), emqx_hooks:action(), emqx_hooks:filter(), integer())
        -> ok | {error, already_exists}).
    hook(HookPoint, Action, Filter, Priority) ->
        emqx_hooks:add(HookPoint, Action, Filter, Priority).

    -spec(unhook(emqx_hooks:hookpoint(), emqx_hooks:action()) -> ok).
    unhook(HookPoint, Action) ->
        emqx_hooks:del(HookPoint, Action).

    -spec(run_hook(emqx_hooks:hookpoint(), list(any())) -> ok | stop).
    run_hook(HookPoint, Args) ->
        emqx_hooks:run(HookPoint, Args).

    -spec(run_fold_hook(emqx_hooks:hookpoint(), list(any()), any()) -> any()).
    run_fold_hook(HookPoint, Args, Acc) ->
        emqx_hooks:run_fold(HookPoint, Args, Acc).

钩子(Hook)使用
--------------

`emqx_plugin_template`_ 提供了全部钩子的使用示例，例如端到端的消息处理回调:

.. code-block:: erlang

    -module(emqx_plugin_template).

    -export([load/1, unload/0]).

    -export([on_message_publish/2, on_message_deliver/3, on_message_acked/3]).

    load(Env) ->
        emqx:hook('message.publish', fun ?MODULE:on_message_publish/2, [Env]),
        emqx:hook('message.deliver', fun ?MODULE:on_message_deliver/3, [Env]),
        emqx:hook('message.acked', fun ?MODULE:on_message_acked/3, [Env]).

    on_message_publish(Message, _Env) ->
        io:format("publish ~s~n", [emqx_message:format(Message)]),
        {ok, Message}.

    on_message_deliver(Credentials, Message, _Env) ->
        io:format("deliver to client ~s: ~s~n", [Credentials, emqx_message:format(Message)]),
        {ok, Message}.

    on_message_acked(Credentials, Message, _Env) ->
        io:format("client ~s acked: ~s~n", [Credentials, emqx_message:format(Message)]),
        {ok, Message}.

    unload() ->
        emqx:unhook('message.publish', fun ?MODULE:on_message_publish/2),
        emqx:unhook('message.acked', fun ?MODULE:on_message_acked/3),
        emqx:unhook('message.deliver', fun ?MODULE:on_message_deliver/3).

.. _auth_acl:

------------------
认证与访问控制设计
------------------

*EMQ X* 消息服务器支持可扩展的认证与访问控制，通过挂载 ``client.authenticate`` and ``client.check_acl`` 两个钩子实现。

编写鉴权钩子回调函数
--------------------

挂载回调函数到 ``client.authenticate`` 钩子:

.. code-block:: erlang

    emqx:hook('client.authenticate', fun ?MODULE:on_client_authenticate/1, []).

钩子回调函数必须接受一个 ``Credentials`` 参数，并且返回一个新的 Credentials:

.. code-block:: erlang

    on_client_authenticate(Credentials = #{password := Password}) ->
        {ok, Credentials#{result => success}}.

``Credentials`` 结构体是一个包含鉴权信息的 map:

.. code-block:: erlang

    #{
      client_id => ClientId,     %% 客户端 ID
      username  => Username,     %% 用户名
      peername  => Peername,     %% 客户端的 IP 地址和端口
      password  => Password,     %% 密码 (可选)
      result    => Result        %% 鉴权结果，success 表示认证成功,
                                 %% bad_username_or_password 或者 not_authorized 表示失败.
    }

编写 ACL 钩子回调函数
----------------------

挂载回调函数到 ``client.authenticate`` 钩子:

.. code-block:: erlang

    emqx:hook('client.check_acl', fun ?MODULE:on_client_check_acl/4, []).

回调函数必须可接受 ``Credentials``, ``AccessType``, ``Topic``, ``ACLResult`` 这几个参数， 然后返回一个新的 ACLResult:

.. code-block:: erlang

    on_client_check_acl(#{client_id := ClientId}, AccessType, Topic, ACLResult) ->
        {ok, allow}.

AccessType 可以是 ``publish`` 和 ``subscribe`` 之一。
Topic 是 MQTT topic。
ACLResult 要么是 ``allow``，要么是 ``deny``。

``emqx_mod_acl_internal`` 模块实现了基于 etc/acl.conf 文件的 ACL 机制，etc/acl.conf 文件的默认内容：

.. code-block:: erlang

    %%%-----------------------------------------------------------------------------
    %%%
    %%% -type who() :: all | binary() |
    %%%                {ipaddr, esockd_access:cidr()} |
    %%%                {client, binary()} |
    %%%                {user, binary()}.
    %%%
    %%% -type access() :: subscribe | publish | pubsub.
    %%%
    %%% -type topic() :: binary().
    %%%
    %%% -type rule() :: {allow, all} |
    %%%                 {allow, who(), access(), list(topic())} |
    %%%                 {deny, all} |
    %%%                 {deny, who(), access(), list(topic())}.
    %%%
    %%%-----------------------------------------------------------------------------

    {allow, {user, "dashboard"}, subscribe, ["$SYS/#"]}.

    {allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}.

    {deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.

    {allow, all}.

由 emqx 提供的 Auth/ACL 插件:

+-----------------------+--------------------------------+
| Plugin                | Authentication                 |
+-----------------------+--------------------------------+
| emqx_auth_username    | Username and Password          |
+-----------------------+--------------------------------+
| emqx_auth_clientid    | ClientID and Password          |
+-----------------------+--------------------------------+
| emqx_auth_ldap        | LDAP                           |
+-----------------------+--------------------------------+
| emqx_auth_http        | HTTP API                       |
+-----------------------+--------------------------------+
| emqx_auth_mysql       | MySQL                          |
+-----------------------+--------------------------------+
| emqx_auth_pgsql       | PostgreSQL                     |
+-----------------------+--------------------------------+
| emqx_auth_redis       | Redis                          |
+-----------------------+--------------------------------+
| emqx_auth_mongo       | MongoDB                        |
+-----------------------+--------------------------------+
| emqx_auth_jwt         | JWT                            |
+-----------------------+--------------------------------+

.. _plugin:

----------------
插件(Plugin)设计
----------------

插件是一个可以被动态加载的普通 Erlang 应用(Application)。插件主要通过钩子(Hook)机制扩展服务器功能，或通过注册扩展模块方式集成认证访问控制。

emqx_plugins 模块实现插件机制，提供加载卸载插件 API ::

    -module(emqx_plugins).

    -export([load/1, unload/1]).

    %% @doc Load a Plugin
    load(PluginName :: atom()) -> ok | {error, any()}.

    %% @doc UnLoad a Plugin
    unload(PluginName :: atom()) -> ok | {error, any()}.

用户可通过 `./bin/emqx_ctl` 命令行加载卸载插件::

    ./bin/emqx_ctl plugins load <plugin name>

    ./bin/emqx_ctl plugins unload <plugin name>

开发者请参考模版插件: http://github.com/emqx/emqx_plugin_template

-----------------
Mnesia/ETS 表设计
-----------------

+--------------------------+--------+------------------+
|          Table           |  Type  |   Description    |
+==========================+========+==================+
| emqx_conn                | ets    | 连接表           |
+--------------------------+--------+------------------+
| emqx_metrics             | ets    | 统计表           |
+--------------------------+--------+------------------+
| emqx_session             | ets    | 会话表           |
+--------------------------+--------+------------------+
| emqx_hooks               | ets    | 钩子表           |
+--------------------------+--------+------------------+
| emqx_subscriber          | ets    | 订阅者表         |
+--------------------------+--------+------------------+
| emqx_subscription        | ets    | 订阅表           |
+--------------------------+--------+------------------+
| emqx_admin               | mnesia | Dashboard 用户表 |
+--------------------------+--------+------------------+
| emqx_retainer            | mnesia | Retained 消息表  |
+--------------------------+--------+------------------+
| emqx_shared_subscription | mnesia | 共享订阅表       |
+--------------------------+--------+------------------+
| emqx_session_registry    | mnesia | 全局会话注册表   |
+--------------------------+--------+------------------+
| emqx_alarm_history       | mnesia | 告警历史表       |
+--------------------------+--------+------------------+
| emqx_alarm               | mnesia | 告警表           |
+--------------------------+--------+------------------+
| emqx_banned              | mnesia | 禁止登陆表       |
+--------------------------+--------+------------------+
| emqx_route               | mnesia | 路由表           |
+--------------------------+--------+------------------+
| emqx_trie                | mnesia | Trie 表          |
+--------------------------+--------+------------------+
| emqx_trie_node           | mnesia | Trie Node 表     |
+--------------------------+--------+------------------+
| mqtt_app                 | mnesia | App 表           |
+--------------------------+--------+------------------+

.. _erlang:

---------------
Erlang 设计相关
---------------

1. 使用 Pool, Pool, Pool... 推荐 GProc 库: https://github.com/uwiger/gproc

2. 异步，异步，异步消息...连接层到路由层异步消息，同步请求用于负载保护

3. 避免进程 Mailbox 累积消息

4. 消息流经的 Socket 连接、会话进程必须 Hibernate，主动回收 binary 句柄

5. 多使用 Binary 数据，避免进程间内存复制

6. 使用 ETS, ETS, ETS... Message Passing vs. ETS

7. 避免 ETS 表非键值字段 select, match

8. 避免大量数据 ETS 读写, 每次 ETS 读写会复制内存，可使用 lookup_element, update_counter

9. 适当开启 ETS 表 {write_concurrency, true}

10. 保护 Mnesia 数据库事务，尽量减少事务数量，避免事务过载(overload)

11. 避免对 Mnesia 数据表非索引、或非键值字段 match, select

.. _eSockd: https://github.com/emqx/esockd
.. _Chain-of-responsibility_pattern: https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern
.. _emqx_plugin_template: https://github.com/emqx/emqx_plugin_template/blob/master/src/emqx_plugin_template.erl

