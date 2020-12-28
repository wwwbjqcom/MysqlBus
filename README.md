
# Tubing  
  
集中式代理后端数据库的小型读写分离中间件， 功能比较少，只有读写分离以及连接池功能。

# 功能  
  
 1. 可以集中管理后端数据库，根据不同的业务(platform)进行区分
 2. 每个后端节点独立的连接池管理
 3. 可以动态修改连接池大小
 4. 可以和我开发的高可用[MysqlMp](https://github.com/wwwbjqcom/mysqlMP-server)进行兼容，自动获取主从关系并维护对应连接池， 如果后端数据库宕机或者主从切换无需做任何操作，对客户端完全透明
 5. 不同的后端业务可通过set platfrom来设置连接
 6. 支持多用户连接， 用户权限同步后端mysql已存在的用户信息
 7. 支持单主模式mgr集群， 自动发现主从关系
 8. 支持记录sql操作记录
  
## 获取方式：  
  
获取源码进行编译运行 RUST_LOG=debug ./tubing --defaults-file test.toml 日志方式可以选择info或者debug，退出则使用kill pid的方式，程序会平滑关闭
  
## 部分配置： 
 配置文件为标准的toml格式， 可参考test.toml，全局配置为基本信息

 1. mode: 可配置mp或者file， mp既通过高可用进行获取，默认都是file，但是file的方式没有实现动态获取路由关系，也不能通过命令行进行节点修改
 2. user/password: 为管理用户及密码
 3. server_url: 高可用获取路由信息的接口
 4. hook_id: 用于高可用验证的id
 5. cluster: 从高可用获取路由的名称列表
 
 #### platform(既为业务组， 如果多个业务组则需要写多个):
 1. platform: 该业务组名称，如果使用高可用获取的方式则同高可用业务集群名称
 2. platform_sublist: 子列表，代表该套业务db下存在的子业务，也是客户端set platform时的值
 3. user/password: 连接后端mysql所使用的用户密码，需要对应操作所有的权限
 4. min/max:  后端到mysql连接池最小/最大值

#### user_info(配置用户信息，多个则配置多个)

 - user/password: 客户端连接使用的账号密码，该账号密码必须存在于后端mysql， 会对该用户进行权限判断
 - platform: 代表该用户操作的是那个platform， 一个用户只能有一个platform且该platform必须存在于上面的platform_sublist

 ## 管理命令:
 没有专用的管理端或者管理端口，直接使用mysql命令行工具进行连接并通过set platform=admin命令设置就可以。

    MySQL [(none)]> set platform=admin;   
    Query OK, 0 rows affected (0.00 sec)
    
    MySQL [(none)]> show questions;                          
    +----------+-------------------+------------+------------+------------+------------+--------------------+
    | platform | host_info         | com_select | com_update | com_insert | com_delete | platform_questions |
    +----------+-------------------+------------+------------+------------+------------+--------------------+
    | test001 | 192.168.1.80:3306 |          1 |          0 |          0 |          0 | 3                  |
    | test001 | 192.168.1.81:3306 |          0 |          0 |          0 |          0 | 3                  |
    +----------+-------------------+------------+------------+------------+------------+--------------------+
    2 rows in set (0.00 sec)
    
    MySQL [(none)]> show connections;
    +----------+-------------------+------------+------------+---------------+------------+
    | platform | host_info         | min_thread | max_thread | active_thread | pool_count |
    +----------+-------------------+------------+------------+---------------+------------+
    | test001 | 192.168.1.80:3306 |         10 |        100 |             0 |         10 |
    | test001 | 192.168.1.81:3306 |         10 |        100 |             0 |         10 |
    +----------+-------------------+------------+------------+---------------+------------+
    2 rows in set (0.00 sec)

 查询状态目前只支持show connections/questions/status/fuse四个操作，可以看到连接池使用情况以及qps，都可以通过命令where platform='' 添加条件查询。


    MySQL [(none)]> set min_thread=1 where platform="test001" and host_info="192.168.1.80:3306";
    Query OK, 0 rows affected (0.00 sec)
    
    MySQL [(none)]> show connections;
    +----------+-------------------+------------+------------+---------------+------------+
    | platform | host_info         | min_thread | max_thread | active_thread | pool_count |
    +----------+-------------------+------------+------------+---------------+------------+
    | test001 | 192.168.1.80:3306 |          1 |        100 |             0 |         10 |
    | test001 | 192.168.1.81:3306 |         10 |        100 |             0 |         10 |
    +----------+-------------------+------------+------------+---------------+------------+
    2 rows in set (0.00 sec)
可以通过set命令修改连接池大小情况以及开关日志记录(set auth=1/0 whre ....)， sql记录可供临时开启排查问题作用，不建议一直使用， 由于sql语句需要写入文件，所有比较损耗性能，set命令必须带where条件，防止误操作, set 命令支持：

 - set min_thread  修改连接池最小值
 - set max_thread 修改连接池最大值
 - set aut 打开/关闭sql审计
 - set fuse 打开/关闭限流功能，如果打开则限制单个platform最多只能获取连接池中80%的连接，只限于sublist中的paltform名称， 业务db集群的paltform不受限制。

## 内部原理：  
![enter image description here](https://i.niupic.com/images/2020/07/28/8sMM.png)
 1. 连接池每分钟进行心跳检测、每十分钟进行连接池状态维护、每50ms从高可用获取一次节点状态，如变化则修改， 空闲连接超过十分钟同时连接池数量大于最小值则会关闭。
 2. 每个mysql业务组读流量连接通过最小连接算法进行获取，数据库操作基本都是长连接，通过最小算法获取比轮询能更好的均衡流量。
 3. 客户端连接获取到的连接空闲超过200ms才归还到连接池，像插入/更新之后立马需要查询的不会路由到从节点，同时也支持对select语句强制走主库， 例如select /\*force_master*/ a,b,c from t1。
 4. 每个客户端连接创建之后首先需要set platfrom才可以进行操作，因为不设置的话不知道该把操作语句路由到那个业务组，所以该中间件不支持像navicat这些界面工具。
 5. set命令只支持设置names、autocommit、platfrom
 6. 中间件只做流量路由及连接池管理，且内部都是异步处理，如果需要kill线程，需直接在后端进行操作。
 7. 启动时会加载配置所有用户的权限信息， 业务端操作时权限由中间件完成，如果后端mysql用户权限变动需重启中间件
## 限制条件
 1. set命令仅支持set names/autocommit/platform三个命令
 2. 仅支持单条sql语句，如果以; 分隔传入多条语句会引发不可知问题
 3. 连接成功后首先要执行set platform命令才能正常运作
 4. mgr仅支持单主模式

## 特别提醒
该工具已在我司生产环境范围性使用，现在还在初级阶段，如在使用或测试中遇到问题欢迎提issues或者加qq群(479472450)找我

