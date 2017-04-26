##  由单台Master扩展至多台Master

   为了避免单台master失败，应当配置多台master。

   注意：向配置了多台master的集群中添加新的master是不安全的，不推荐这么做。
    
所有命令行必须在kudu用户下执行
### 迁移准备工作
* 确保一个维护时间段（一个小时足够了），在这期间kudu集群无法使用
* 确定master的总个数并且为奇数，推荐个数为3个或5个

* 在已存在的msters上执行如下操作：
  * 获取master的数据目录路径，默认为、var/lib/kudu/master,还可以通过  fs_wal_dir 和fs_data_dirs设置，需要注意，fs_wal_dir和fs_data_dirs必须在同一父目录下。
  
  * 获取rpc地址，即rpc_bind_address设置的值
  * 获取master的UUID
  *      $ kudu fs dump uuid --fs_wal_dir=<master_data_dir> 2>/dev/null
  *      
        $ kudu fs dump uuid --fs_wal_dir=/var/lib/kudu/master 2>/dev/null
             4aab798a69e94fab8d77069edff28ce0
  * 可选项：为master主机配置DNS别名，可以是已配置的DNS,可以是IP地址，可以是在/etc/hosts配置的别名
 * 在新的master机器上执行如下操作：
   * 确保安装kudu-master以及kudu
   
   * 配置数据目录
   * 配置RPC地址
   * 配置别名
### 执行迁移
   * 停止集群中所有kudu进程
   * 获取新的master机器的UUID
   *     $ kudu fs format --fs_wal_dir=<master_data_dir>
         $ kudu fs dump uuid --fs_wal_dir=<master_data_dir> 2>/dev/null
   * 在原有master上重新进行raft 配置
   *      kudu local_replica cmeta rewrite_raft_config                 --fs_wal_dir=<master_data_dir> <tablet_id> <all_masters>
         其中 tablet_id 为 00000000000000000000000000000000
         all_masters为新增以及原有master的集合，格式为 <uuid>:<hostname>:<port>
        举例如下：
   *        $ kudu local_replica cmeta rewrite_raft_config --fs_wal_dir=/var/lib/kudu/master 00000000000000000000000000000000 4aab798a69e94fab8d77069edff28ce0:master-1:7051 f5624e05f40649b79a757629a69d061e:master-2:7051 988d8ac6530f426cbe180be5ba52033d:master-3:7051
   * 修改所有master(新增及原有)的master_addresses，为所有master的地址列表，单个格式为 <hostname>:<port> ，中间逗号隔开
   * 启动原有master进程
   * 拷贝master数据至新的master，在新的master机器上执行如下命令：
   *     $ kudu local_replica copy_from_remote --fs_wal_dir=<master_data_dir> <tablet_id> <existing_master>
        举例如下：
   *         $ kudu local_replica copy_from_remote --fs_wal_dir=/var/lib/kudu/master 00000000000000000000000000000000 master-1:7051
   * 启动所有新增加的master
   * 配置所有Tablet Server的 tserver_master_addrs ， 以逗号隔开，单个
格式如上
   * 启动所有Tablet Server
