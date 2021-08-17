# Centos8安装Ceph-Pacific（容器化部署）



## 环境要求

系统基础环境如下：

| 工具   | 版本            | 备注                                             |
| ------ | --------------- | ------------------------------------------------ |
| CentOS | 8.4.2105_x86-64 | Centos8后续没有固定版本维护，采用APPStream流方式 |
| Ceph   | Pacific 16.2.5  | 要求centos8，7内核太低不符合要求，且没有7的rpm包 |
| Podman | 3.0.2-dev       | Centos8.4默认版本，8.2采用1.6版本不符合          |

Cephadm与podman版本要求对应关系如下：

| **Ceph**  | **Podman** |         |         |         |         |
| --------- | ---------- | ------- | ------- | ------- | ------- |
|           | **1.9**    | **2.0** | **2.1** | **2.2** | **3.0** |
| <= 15.2.5 | True       | False   | False   | False   | False   |
| >= 15.2.6 | True       | True    | True    | False   | False   |
| >= 16.2.1 | False      | True    | True    | False   | True    |

 虚拟机环境

| Hostname | Ip addr   | 配置          |
| -------- | --------- | ------------- |
| Master   | 10.0.0.11 | 4GB, SSD 10GB |
| Slave1   | 10.0.0.12 | 4GB, SSD 10GB |
| Slave2   | 10.0.0.13 | 4GB, SSD 10GB |

 

## 部署准备

### 环境要求

Ceph pacific要求系统安装如下软件：

- Python 3
- Systemd
- Podman or Docker for running containers
- Time synchronization (such as chrony or NTP)
- LVM2 for provisioning storage devices



### cephadm安装

cephadm是由Ceph社区主导的部署工具脚本，主要是为了统一部署工具，替代`ceph-volume`、`ansible`等。其主要作用是通过容器化方式获取并部署Ceph集群，这样使Ceph更加容易使用和升级，不会因为系统或软件版本问题导致各种错误。

cephadm就是一个python脚本，单个文件，可以通过RPM包方式安装（具体下载路径在[download.ceph.com/](https://download.ceph.com/) 下的对应版本），也可以通过github内获取源码的方式获取。如下：

```bash
#1 获取cephadm脚本，但curl github在国内可能行不通，可以直接访问网站拷贝下这个脚本
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm

chmod +x cephadm

#2 添加ceph repo源
./cephadm add-repo --release pacific

#3 安装cephadm包，这步不是必须，也可以一直 ./cephadm 执行或者把刚才的脚本放在 /usr/bin/目录下也行
./cephadm install

#4 安装ceph其他包，如 ceph-common，由于容器化，所以宿主机其实不需要，也可以不安装
./cephadm install ceph-common
```



cephadm提供了容器shell命令，让我们可以在不安装ceph-common包的情况下获取ceph状态，其原理是启动一个ceph容器，引用`/etc/ceph/ceph.conf`配置，执行并返回命令，如下。

```bash
# ceph -s 命令
cephadm shell -- ceph -s

# 如果没有安装ceph-common，为了保持习惯，可以在bash里生成alias，在 /root/.bashrc 脚本中将alias永久配置，这样宿主机就能够保持干净
alias ceph='cephadm shell -- ceph'
```



### 部署Ceph集群



#### 初始化集群（Bootstrap）

使用cephadm工具，其策略是先在单个节点上初始化一个Ceph环境，然后cephadm任务就完成了，最后通过orchestrator集群管理工具进行扩展的方式部署一个高可靠的Ceph集群环境。其操作如下：

```bash
# 执行 cephadm bootstrap --mon-ip *<mon-ip>* ， 如
cephadm bootstrap --mon-ip 10.0.0.11

# 正常输出如下：
Verifying xxx...
Cluster fsid: 07d15412-f5b5-11eb-8264-000c29c81876
Verifying IP 10.0.0.11 port 3300 ...
Verifying IP 10.0.0.11 port 6789 ...
Mon IP `10.0.0.11` is in CIDR network `10.0.0.0/24`
Pulling container image docker.io/ceph/ceph:v16...
Ceph version: ceph version 16.2.5 (0883bdea7337b95e4b611c768c0279868462204a) pacific (stable)
Creating initial xxx...

Ceph Dashboard is now available at:

	     URL: https://wayne-master:8443/
	    User: admin
	Password: 10whm9lr7h
```

这个命令会完成如下操作：

- 在本地节点新建一个ceph cluster，并创建一个 Mon 和 Mgr daemon；
- 为该集群生成一个 SSH key 并添加到 root 用户的  `/root/.ssh/authorized_keys` 文件中，并复制其公钥到`/etc/ceph/ceph.pub`文件；
- 生成最小化的 `/etc/ceph/ceph.conf`配置文件；
- 生成`/etc/ceph/ceph.client.admin.keyring`的管理员 client.admin 密钥
- 将该节点添加为 `_admin`标签。

可用通过 `cephadm bootstrap -h`查看一些Option配置，这里不详说，可添加 `--apply-spec` 指定 cluster 的集群配置，使得bootstrap后按照spec文件进行集群管理配置。



这样，一个简单的Ceph容器化环境就好了，cephadm作用就到此为止了。剩下的就是使用orchestrator组件扩展host节点和配置OSD了。我们可以通过`orch`子命令和`podman ps -a`查看当前的情况。也可以通过刚才部署完的Dashboard进行网页可视化的集群扩展配置。

```bash
[root@wayne-master ~]# ceph orch ls
NAME           PORTS        RUNNING  REFRESHED  AGE  PLACEMENT  
alertmanager   ?:9093,9094      1/1  2m ago     2h   count:1    
crash                           1/1  2m ago     2h   *          
grafana        ?:3000           0/1  -          2h   count:1    
mgr                             1/2  2m ago     2h   count:2    
mon                             1/5  2m ago     2h   count:5    
node-exporter  ?:9100           1/1  2m ago     2h   *          
prometheus     ?:9095           1/1  2m ago     2h   count:1  

[root@wayne-master ~]# ceph orch host ls
HOST          ADDR       LABELS  STATUS  
wayne-master  10.0.0.11  _admin      
```

有几个知识点可以提前了解：

- orch管理Ceph集群，包括每个组件运行多少个实例，由PLACEMENT配置决定
- orch维护host列表，并根据策略和实例个数可自动化的在host里拉起/删除ceph组件
- 采用prometheus进行节点和服务监控
- crash容器是用于组件崩溃数据日志收集，应该是用于定位问题的，具体以后可以研究一下



#### 配置集群Host节点

首先，要配置ssh免密登录，前面部署完后，在master节点上会创建一个`/etc/ceph/ceph.pub`的SSH密钥文件，把这个文件拷贝到其他的节点上。

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@*<new-host>*

# 示例
ssh-copy-id -f -i /etc/ceph/ceph.pub root@slave1
ssh-copy-id -f -i /etc/ceph/ceph.pub root@slave2
```



然后，通过ceph命令把节点添加到orch host列表中。

```bash
ceph orch host add *<newhost>* [*<ip>*] [*<label1> ...*]

# 示例
ceph orch host add slave1 10.0.0.12 _admin
ceph orch host add slave2 10.0.0.13 _admin

# PS: label 是host的一个标签，可以自定义，也有一些是ceph默认的，如 _admin 表明这是个管理节点，
# 会把 ceph.conf、client.admin keyring 文件拷贝到 _admin 标签节点的 /etc/ceph 目录下。
```

Label标签可参考如下说明：

The following host labels have a special meaning to cephadm. All start with `_`.

- `_no_schedule`: *Do not schedule or deploy daemons on this host*.

  This label prevents cephadm from deploying daemons on this host. If it is added to an existing host that already contains Ceph daemons, it will cause cephadm to move those daemons elsewhere (except OSDs, which are not removed automatically).

- `_no_autotune_memory`: *Do not autotune memory on this host*.

  This label will prevent daemon memory from being tuned even when the `osd_memory_target_autotune` or similar option is enabled for one or more daemons on that host.

- `_admin`: *Distribute client.admin and ceph.conf to this host*.

  By default, an `_admin` label is applied to the first host in the cluster (where bootstrap was originally run), and the `client.admin` key is set to be distributed to that host via the `ceph orch client-keyring ...` function. Adding this label to additional hosts will normally cause cephadm to deploy config and keyring files in `/etc/ceph`.



#### 配置Mon

一般来说，Mon在增加完节点后，就会默认根据bootstrap的IP地址寻找合适的节点进行Mon组件的添加。由cephadm自动完成，不需要手动参与。但是我们仍然可以对Mon进行一些配置，来实现一些定制化的配置。

```bash
# 配置 mon 不自动管理
ceph orch apply mon --unmanaged

# 配置 mon 的public_network, 可以接受多个networks，但目前不清楚意义在哪
ceph config set mon public_network 10.1.2.0/24,192.168.0.1/24

# 可以指定 mon 的部署节点，需要unmanage，不然mon就会给自动部署起来了
ceph orch apply mon --unmanaged
ceph orch daemon add mon newhost1:10.1.2.123
ceph orch daemon add mon newhost2:10.1.2.0/24

# 配置 mon 的最大部署数为3， ceph官方建议3个或5个，默认5个。
ceph orch apply mon 3

# 配置 mon 按照placement的定义部署，这样mon就会部署到对应的placement上，具体可参照placement章节
ceph orch apply mon --placement="newhost1,newhost2,newhost3"

# 删除 mon daemon ： ceph orch daemon rm *mon.<oldhost1>*
ceph orch daemon rm mon.node1
```

对于Mon修改网络IP，Ceph官网建议新增加对应网络的Mon，然后删除旧的，但是两个网段也不通，两个不同网段的Mon如何通信进行数据同步，暂时没想好，还需要进一步研究（一种可能性是需要物理网络通过配置打通两种网段，使之能够通信，比如采用网络转发，默认网关之类）。



#### 部署OSD

配置完host节点后，Ceph其他组件如mon、mgr就会根据预定义的Placement进行自动扩展，在有效的节点内拉起对应的容器镜像。但是OSD部署还是需要执行一些操作的。一般地，使用下面的命令，cephadm会自动根据各个节点的device设备信息，选择可用的硬盘直接部署成OSD服务。

```bash
ceph orch apply osd --all-available-devices

# 可通过如下命令查看可用的devices
[~] ceph orch device ls
Hostname      Path      Type  Serial  Size   Health   Ident  Fault  Available  
wayne-master  /dev/sdb  hdd           10.7G  Unknown  N/A    N/A    Yes        
wayne-slave1  /dev/sdb  hdd           10.7G  Unknown  N/A    N/A    Yes        
wayne-slave2  /dev/sdb  hdd           10.7G  Unknown  N/A    N/A    Yes  
```

硬盘设备可用的条件：

- The device must have no partitions.
- The device must not have any LVM state.
- The device must not be mounted.
- The device must not contain a file system.
- The device must not contain a Ceph BlueStore OSD.
- The device must be larger than 5 GB.

设备信息应该是通过监控插件 node-exporter 获取的，然后交由 mgr module：cephadm 进行定时refresh并配置 service 服务。下面是一些OSD的管理命令，后接 `--dry-run` 参数表示预演命令，并不会实际产生效果：

```bash
# zap 格式化 osd 盘： ceph orch device zap <hostname> <path>
ceph orch device zap wayne-master /dev/sdb

# 指定创建 OSD： ceph orch daemon add osd *<host>*:*<device-path>*
ceph orch daemon add osd wayne-master：/dev/sdb

# 自动部署可用的设备为 OSD
ceph orch apply osd --all-available-devices

# 删除 OSD： ceph orch osd rm <osd_id(s)> [--replace] [--force]
# --replace 指在删除osd时保留osd-id等一些信息，使该osd保留在CRUSH中，并等待add osd时赋予这些信息
# 这样，就相当于采取了replace替换一个 osd 盘的操作。
ceph orch osd rm 0
ceph orch osd rm 4 --replace

# 配置 OSD daemon的内存限制，先设置autotune内存为false，然后设置
ceph config set osd.123 osd_memory_target_autotune false
ceph config set osd.123 osd_memory_target 16G
```

如果不想采用`--all-available-devices`方式，除了一个个添加外，也可以为osd写一个yaml的osd-specs文件，来定义osd的配置部署方式，详见placement章节。



### 采用yaml文件部署Ceph

在执行 cephadm bootstrap时，接`--apply-spec`可以在初始化完后进行 orch 的集群配置调整。其相当于分两步走：先执行`cephadm bootstrap`后，执行`ceph orch apply -i cluster.yml`。Service描述的具体定义要求可查看官方文档service-management相关章节。

```yaml
---
service_type: host
hostname: wayne-master
addr: 10.0.0.11
labels:
- _admin

---
service_type: host
hostname: wayne-slave1
addr: 10.0.0.12
labels:
- _admin

---
service_type: host
hostname: wayne-slave2
addr: 10.0.0.13
labels:
- _admin

---
service_type: mgr
placement:
  count: 2
  
---
service_type: mon
placement:
  label: '_admin'
  
---
service_type: osd
service_id: all_avail_osd
placement:
  host_pattern: '*'
data_devices:
  all: true
```



### 删除集群

cephadm并没有提供一键删除命令，其提供了 `rm-cluster` 命令，可以清理本节点的所有Ceph相关的组件、数据和配置。

```bash
# 清理本节点集群环境，本命令主要：
# 1. 删除 containers
# 2. 删除 /var/lib/ceph/07d15412-f5b5-11eb-8264-000c29c81876
# 3. 删除 /etc/ceph/ 文件
cephadm rm-cluster --fsid 07d15412-f5b5-11eb-8264-000c29c81876 --force
```



因此，要清理整个集群环境，只能在所有节点都执行该操作。最好同时进行，因为不然的话，ceph orch有可能恰好定期更新集群状态，并把 rm-cluster 的节点当成是一个全新的节点，调度创建新的daemon。



# #附录

## #问题及解决方法

 ### #1 ceph-grafana:6.7.4镜像下载manifest unknown问题

配置国内源，在执行`ceph orch ls`时发现，grafana未能正常运行，由于manifest unknown，导致镜像无法下载。如下所见：

```txt
[root@wayne-master ~]# podman pull ceph-grafana:6.7.4
Resolving "ceph-grafana" using unqualified-search registries (/etc/containers/registries.conf)
Trying to pull docker.io/library/ceph-grafana:6.7.4...
  manifest unknown: manifest unknown
Error: Error initializing source docker://ceph-grafana:6.7.4: Error reading manifest 6.7.4 in hub-mirror.c.163.com/library/ceph-grafana: manifest unknown: manifest unknown
```



起始以为是国内源不能用了，但下载其他镜像都好使的，然后排除这个问题。然后查看了一下镜像的版本，居然区分了x86和aarch64版本的，尝试性的下载x86版本，然后将tag改为6.7.4（cephadm默认版本号），发现就可以用了。如下：

```bash
[root@wayne-master ~]# curl -XGET http://hub-mirror.c.163.com/v2/ceph/ceph-grafana/tags/list
{"name":"ceph/ceph-grafana","tags":["6.6.2","6.7.4-aarch64","6.7.4-x86_64","6.7.4","latest"]}

# 下载 6.7.4-x86_64 对应的镜像，修改tag
podman image tag 557c83e11646 docker.io/ceph/ceph-grafana:6.7.4
```

在hub.docker.com官网，`ceph-grafana:6.7.4`是没有hash code的，应该是选用特定系统架构版本的镜像即可，cephadm默认版本号没有及时进行修改。



### #2 手动运行OSD（OS reinstalled）

当重装系统或者执行 `cephadm rm-cluster`后，其实osd的盘没有zap，那么是可以重新加入到ceph集群的。

在以前，可以使用 `ceph-volum lvm activate --all` 命令去激活OSD。但是现在使用容器运行OSD，所以直接通过  `cephadm ceph-volum lvm activate --all` 是有问题的，因为这相当于在 ceph-volume的容器内运行OSD的容器，显然和我们预想的不一样。

查阅官方手册及bug，给予了两种方式：

一是直接通过 unit.run 的脚本，在 `/var/lib/ceph/<cluster-fsid>/<service-name>/unit.run` 路径上。这中方式没有试验，但感觉也可以，容器起起来就行。

二是手工方式执行：（参考 [bugfix-46691](https://tracker.ceph.com/issues/46691)）

```bash
# 1. OSD Host: 获取osd id 和 osd-fsid
cephadm ceph-volume lvm list

# 2. Admin Host: 获取osd keyring
ceph auth get osd.XYZ

# 3. Admin Host: 生成一个ceph配置文件 
ceph config generate-minimal-conf

# 4. Admin Host：获取osd的容器镜像 
ceph config get "osd.xyz" container_image

# 5. OSD Host：生成 config json， 内容如下。

cat config-json.json
{
"config": "# minimal ceph.conf for 8255263a-a97e-4934-822c-00bfe029b28f\n[global]\n\tfsid = 8255263a-a97e-4934-822c-00bfe029b28f\n\tmon_host = [v2:192.168.178.28:40483/0,v1:192.168.178.28:40484/0]\n",
"keyring": "[osd.XYZ]\n\tkey = AQDSb0ZfJqfDNxAAhgAVuH8Wg0yaRUAHwXdTQA==\n" 
}

# 6. OSD ： cephadm deploy
#  PS：
#      1. --image 可以不写，当时试验时写上了反而失败了
#      2. --config-json 也可以用 --config /etc/ceph/ceph.conf 替换
cephadm --image <container-image> deploy --fsid <fsid> --name osd.xyz --config-json config-json.json --osd-fsid <osd-fsid>
```



在执行时，有时会遇到如下的错误。

```txt
[ceph: root@wayne-master /]# ceph -s
  cluster:
    id:     b0f4f1f2-fb4b-11eb-8153-000c29c81876
    health: HEALTH_WARN
            1 stray daemon(s) not managed by cephadm
```



原因是之前记录了daemon，现在cephadm无法看到了（cache缓存里没有），认为没有纳入到管理中，通过 `ceph orch ps`可以看到，新增的osd没有在里面。这个过一会集群就能自动解决的。（具体可以查看下cephadm中的 `_check_for_strays`相关代码。）



## #一些配置

### #1 国内容器镜像源配置

国内容器镜像源地址如下，使用网易或者中科大镜像效果不错。

```txt
Docker中国官方镜像加速
--registry-mirror=https://registry.docker-cn.com
网易163镜像加速
--registry-mirror=http://hub-mirror.c.163.com
中科大镜像加速
--registry-mirror=https://docker.mirrors.ustc.edu.cn
阿里云镜像加速
--registry-mirror=https://{your_id}.mirror.aliyuncs.com
daocloud镜像加速
--registry-mirror=http://{your_id}.m.daocloud.io 
```



Docker管理工具源配置步骤：

```bash
# 1. 创建文件夹
sudo  mkdir -p /etc/docker
# 2. 编辑/etc/docker/daemon.json文件，并输入国内镜像源地址
# xxx 为上面的国内镜像源地址
sudo  vi /etc/docker/daemon.json
{
    "registry-mirrors": ["xxx", "xxx"]
}  
```



Podman管理工具源配置步骤：

```bash
# PS: 在安装docker-ce时，按照国内源替换操作，总是报registry-1.docker.io reset peer error. 按照上述方法可以解决并成功使用国内源。

# /etc/containers/registries.conf中，配置prefix 前缀为docker.io可正常使用，如下：
mv /etc/containers/registries.conf  /etc/containers/registries.conf.bak

cat << EOF > /etc/containers/registries.conf
unqualified-search-registries =  ["docker.io"]

[[registry]] 
prefix = "docker.io" 
location = "docker.mirrors.ustc.edu.cn"
EOF
```

最后，执行`podman | docker system info` 查看`registry`是否生效，或者pull一个镜像看下速度。



## #新的认识/总结

### #1 ceph-pacific管理节点实例启停

当我们执行命令 `ceph orch host add xxx` 操作后，我们发现ceph能够自动在新增加的节点中自动部署容器化实例。这引起我的好奇心，一个全新的节点，没有镜像，也没有cephadm工具，它是根据什么策略实现容器部署管理的。

> PS：为什么会对其感兴趣？因为实验环境是联网的，但是工作的环境在内网，不能连接互联网，所以没法方便的下载镜像。对其进行理解能够对后续的工作起到一定的作用。

为了解决疑惑，大致看了下ceph的源码，并做下了一些笔记总结。这里先简单总结：

1. host add 操作将一个节点新添加到ceph的 KV 数据库中
2. ceph-mgr的module：`cephadm`是一个main-loop服务，定期更新数据并根据策略进行管理控制。
3. mgr module cephadm执行 `_apply_all_services`进行容器的创建或者删除。
4. 如果是在远端节点，通过ssh方式将cephadm脚本文件拷贝过去，然后通过cephadm脚本进行容器pull等各种环境配置及部署操作，然后再进行服务部署。



下面，我们详细地进行了解。先从命令入口点进行一步步地追踪。

从 `pybind/mgr/orchestrator`为切入点进行，但毫无头绪，经过仔细研究，发现`orchestrator`仅仅是一个抽象类库，其一个具体实现为`CephadmOrchestrator`，在`pybind/mgr/cephadm`里定义。在这里，为了先解答ceph容器部署的管理策略，我们先略过前序逻辑描述，只需要知道最终处理的函数是定义在`CephadmOrchestrator`内即可。

我们知道，对于mgr modules来说，主要基于实现以下方法来实现不同的需求。

>- a `serve` member function for server-type modules. This function should block forever.
>- a `notify` member function if your module needs to take action when new cluster data is available.
>- a `handle_command` member function if your module exposes CLI commands. But this approach for exposing commands is deprecated. For more details, see [Exposing commands](https://docs.ceph.com/en/latest/mgr/modules/#mgr-module-exposing-commands).

其实`ceph orch host add`是个幌子，作用仅仅是把节点信息记录起来。而真正做处理的是`CephadmServe`，实现了serve方法，定时对数据进行更新并进行处理，如下。（如无特别说明，代码都是经过删减，只保留较为关键的环节。）

```python
# pybind/mgr/cephadm/module.py
class CephadmOrchestrator(orchestrator.Orchestrator, MgrModule,
                          metaclass=CLICommandMeta):
    def serve(self) -> None:
        serve = CephadmServe(self)
        serve.serve()
     
# pybind/mgr/cephadm/serve.py
class CephadmServe:

    def __init__(self, mgr: "CephadmOrchestrator"):
        self.mgr: "CephadmOrchestrator" = mgr

    def serve(self) -> None:
        while self.mgr.run:
            try:
                self.convert_tags_to_repo_digest()  # 通过digest来确保所有daemon使用镜像相同

                # refresh daemons
                self._refresh_hosts_and_daemons()

                if not self.mgr.paused:
                    # 这步是关键，前面的都是数据更新，这是daemon（即容器）部署配置都在这里实现
                    # 循环调用 _apply_service(...) 为每个spec进行检查并进行service调度
                    if self._apply_all_services():
                        continue  # did something, refresh

                    self._check_daemons()
                    self._purge_deleted_services()

            except OrchestratorError as e:
                ...

            self._serve_sleep()  # 基于配置睡眠时间或者event事件触发循环
```

`_apply_service`方法太长，这里不详细说明，里面做了很多操作。简单点就是`HostAssignment`分配了节点需要增加或者删除的daemon，然后再一系列流程判断后，最终执行create_daemon的操作。

```python
# pybind/mgr/cephadm/serve.py
class CephadmServe:
    def _apply_service(self, spec: ServiceSpec) -> bool:
        """
        Schedule a service.  Deploy new daemons or remove old ones, depending
        on the target label and count specified in the placement.
        """
        svc = self.mgr.cephadm_services[service_type]
        daemons = self.mgr.cache.get_daemons_by_service(service_name)

		# HostAssignment定义在cephadm/shecdule.py内，分配合适的host
        ha = HostAssignment(
            spec=spec,
            hosts=self.mgr._schedulable_hosts(),
            daemons=daemons,
            networks=self.mgr.cache.networks,
            filter_new_host=(
                matches_network if service_type == 'mon'
                else None
            ),
            allow_colo=svc.allow_colo(),
            primary_daemon_type=svc.primary_daemon_type(),
            per_host_daemon_type=svc.per_host_daemon_type(),
            rank_map=rank_map,
        )

        try:
            all_slots, slots_to_add, daemons_to_remove = ha.place()
        except OrchestratorError as e:
            return False

        self.log.debug('Hosts that will receive new daemons: %s' % slots_to_add)
        self.log.debug('Daemons that will be removed: %s' % daemons_to_remove)

        # create daemons
        for slot in slots_to_add:

            # deploy new daemon
            daemon_id = slot.name
            daemon_spec = svc.make_daemon_spec(
                slot.hostname, daemon_id, slot.network, spec,
                daemon_type=slot.daemon_type,
                ports=slot.ports,
                ip=slot.ip,
                rank=slot.rank,
                rank_generation=slot.rank_generation,
            )
            self.log.debug('Placing %s.%s on host %s' % (
                slot.daemon_type, daemon_id, slot.hostname))

            try:
                # svc 是 cephadm/services/cephadmservice.py内定义的不同service-type的实例
                # 如mon、mgr、osd等，不同type对应不同的一些配置。
                daemon_spec = svc.prepare_create(daemon_spec)
                # 根据daemon_spec进行真正执行容器部署
                self._create_daemon(daemon_spec)
            except (RuntimeError, OrchestratorError) as e:
                continue
        return r
```

在`_create_daemon`方法中，执行了容器部署的流程，最终会调用cephadm脚本的`cephadm deploy`命令。所有cephadm命令都是通过`_run_cephadm`方法执行的。其大致流程如下：

```python
# pybind/mgr/cephadm/serve.py
class CephadmServe:
    def _run_cephadm(self,
                     host: str,
                     entity: Union[CephadmNoImage, str],
                     command: str,
                     args: List[str],
                     addr: Optional[str] = "",
                     stdin: Optional[str] = "",
                     no_fsid: Optional[bool] = False,
                     error_ok: Optional[bool] = False,
                     image: Optional[str] = "",
                     env_vars: Optional[List[str]] = None,
                     ) -> Tuple[List[str], List[str], int]:

        with self._remote_connection(host, addr) as tpl:
            conn, connr = tpl
            # mgr.mode分为 ’root‘ 和 ’cephadm-package‘模式，其实就是cephadm脚本路径：
            # ’root‘： /var/lib/ceph/{uuid}/cephadm.xxxx
            # ’cephadm-package‘: /usr/bin/cephadm
            # ’root’为默认方式
            if self.mgr.mode == 'root':
                # 'root' 先执行容器命令，如果失败，检查cephadm脚本是否存在，
                # 不存在先部署cephadm脚本后，再尝试执行一遍
                try:
                    out, err, code = remoto.process.check(
                        conn,
                        [python, self.mgr.cephadm_binary_path] + final_args,
                        stdin=stdin.encode('utf-8') if stdin is not None else None)
                    if code == 2:
                        out_ls, err_ls, code_ls = remoto.process.check(
                            conn, ['ls', self.mgr.cephadm_binary_path])
                        if code_ls == 2:
                            self._deploy_cephadm_binary_conn(conn, host)
                            out, err, code = remoto.process.check(
                                conn,
                                [python, self.mgr.cephadm_binary_path] + final_args,
                                stdin=stdin.encode('utf-8') if stdin is not None else None)

                except RuntimeError as e:
                    ...
            elif self.mgr.mode == 'cephadm-package':
                try:
                    out, err, code = remoto.process.check(
                        conn,
                        ['sudo', '/usr/bin/cephadm'] + final_args,
                        stdin=stdin)
                except RuntimeError as e:
                    ...
            else:
                assert False, 'unsupported mode'
            return out, err, code
```



### #2 Orchestrator的神秘通信--转换到特定的实例执行

之前在解析代码的时候，发现orchestrator的一些神秘用法。先说一下总结：

1. Orchestrator类是一个接口，用户可自定义实现其子类实例；
2. OrchestratorCli 提供了通用的管理命令，其会将方法转换成remote Orchestrator，即特定具体实例实现方法执行。
3. 在这里remote调用应该将数据通过序列化或远程调用的方式进行的，但是很明显的，CephadmOrchestrator并没有监听端口，因此ceph开发者采用了一种极其不推荐的方式（原话：magic method），即因为module都是通过python的sub interpreter运行的，它们共用了同一个GIL和GC的对象管理空间，因此可以在ceph-mgr底层（引用记录了各个运行的pymoudles），在python解释器的底层，直接通过`PyObject_Call`方式调用method来执行特定Orchestrator具体实例的方法。



下面详细地走读一下源码，以`ceph orch host ls`为例说明。

之前解析源码的时候，是从`pybind/mgr/orchestrator`的命令切入点进行的。而我们知道实际执行操作是在`CephadmOrchestrator`定义的方法中执行的，这其中使用了一些神奇的魔法。首先，我们看下ceph-mgr的modules定义以及类之间的继承关系。

```python
# pybind/mgr/ochestartor/_interface.py
class Orchestrator: pass
class OrchestratorClientMixin(Orchestrator)： pass

# pybind/mgr/ochestartor/module.py
class OrchestratorCli(OrchestratorClientMixin, MgrModule,
                      metaclass=CLICommandMeta):
    
    # 大部分命令都定义在 OrchestratorCli 里面
    @_cli_write_command('orch host add')
    def _add_host(...): pass
 
# pybind/mgr/cephadm/module.py
class CephadmOrchestrator(orchestrator.Orchestrator, MgrModule,
                          metaclass=CLICommandMeta):
    pass
```

基于上述类的继承关系，我们可以发现下面备注的问题。

> 仔细点看会发现`CephadmOrchestrator`和`OrchestratorCli`类都是`Orchestrator`的子类，并没有继承关系。但是命令行都定义在`OrchestratorCli`内，而实际执行时却会找到`CephadmOrchestrator`的实现函数。那么函数是怎么重定向的？

不仔细看，还以为`CephadmOrchestrator`继承了`OrchestratorCli`类，但其实不是。它们是ceph-mgr的两个module，是相互独立的，但是又有不一般的联系。

```python
# pybind/mgr/ochestartor/module.py
class OrchestratorCli(OrchestratorClientMixin, MgrModule,
                      metaclass=CLICommandMeta):
	@_cli_read_command('orch host ls')
    def _get_hosts(self, format: Format = Format.plain) -> HandleCommandResult:
        """List hosts"""
        completion = self.get_hosts()
        ...
```

如上述代码所示，`OrchestratorCli`定义了获取host列表的方法，其调用了`Orchestrator`的`get_hosts`方法。`OrchestratorCli`和`OrchestratorClientMixin`都没有实现`get_hosts`方法。但是`Orchestrator`是抽象类，其`get_hosts`方法没有实现的，那么`self.get_hosts()`具体实现在哪里呢。

此时，我们需要追溯下父类的实现。

```python
# pybind/mgr/orchestrator/_interface.py
def _mk_orch_methods(cls: Any) -> Any:
    # Needs to be defined outside of for.
    # Otherwise meth is always bound to last key
    def shim(method_name: str) -> Callable:
        def inner(self: Any, *args: Any, **kwargs: Any) -> Any:
            completion = self._oremote(method_name, args, kwargs)
            return completion
        return inner

    for meth in Orchestrator.__dict__:
        if not meth.startswith('_') and meth not in ['is_orchestrator_module']:
            setattr(cls, meth, shim(meth))
    return cls


@_mk_orch_methods
class OrchestratorClientMixin(Orchestrator):
    """
    A module that inherents from `OrchestratorClientMixin` can directly call
    all :class:`Orchestrator` methods without manually calling remote.

    Every interface method from ``Orchestrator`` is converted into a stub method that internally
    calls :func:`OrchestratorClientMixin._oremote`

    >>> class MyModule(OrchestratorClientMixin):
    ...    def func(self):
    ...        completion = self.add_host('somehost')  # calls `_oremote()`
    ...        self.log.debug(completion.result)

    .. note:: Orchestrator implementations should not inherit from `OrchestratorClientMixin`.
        Reason is, that OrchestratorClientMixin magically redirects all methods to the
        "real" implementation of the orchestrator.

    """
    
    def __get_mgr(self) -> Any:
        try:
            return self.__mgr
        except AttributeError:
            return self
        
    def _oremote(self, meth: Any, args: Any, kwargs: Any) -> Any:
        """
        Helper for invoking `remote` on whichever orchestrator is enabled

        :raises RuntimeError: If the remote method failed.
        :raises OrchestratorError: orchestrator failed to perform
        :raises ImportError: no `orchestrator` module or backend not found.
        """
        # 其实就是自己实例
        mgr = self.__get_mgr()

        try:
            o = mgr._select_orchestrator()
        except AttributeError:
            o = mgr.remote('orchestrator', '_select_orchestrator')

        try:
            return mgr.remote(o, meth, *args, **kwargs)
        except Exception as e:
            pass
```

首先，`_mk_orch_methods`修饰器把`OrchestratorClientMixin`中的所有`Orchestrator`接口方法替换成remote的调用方式。从上面`OrchestratorClientMixin._oremote`实现可以知道，先获取`Orchestrator`特定实例，然后执行remote调用。

```python
# pybind/mgr/ochestartor/module.py
class OrchestratorCli(OrchestratorClientMixin, MgrModule,
                      metaclass=CLICommandMeta):
    def _select_orchestrator(self) -> str:
        # 选择orchestrator，相当于执行下面命令，返回的是 cephadm 这个 ceph-mgr module
        # [~root] ceph config get mgr mgr/orchestrator/orchestrator
        return cast(str, self.get_module_option("orchestrator"))
    
# pybind/mgr_module.py
class MgrModule(ceph_module.BaseMgrModule, MgrModuleLoggingMixin):
    def remote(self, module_name: str, method_name: str,
               *args: Any, **kwargs: Any) -> Any:
        """
        Invoke a method on another module.  All arguments, and the return
        value from the other module must be serializable.
        """
        return self._ceph_dispatch_remote(module_name, method_name,
                                          args, kwargs)
```

remote方法是MgrModule的实现，其后面就是MGR的CPython实现了。具体如下：

```cpp
// src\mgr\BaseMgrModule.cc
PyMethodDef BaseMgrModule_methods[] = {
    {"_ceph_dispatch_remote", (PyCFunction)ceph_dispatch_remote,
    METH_VARARGS, "Dispatch a call to another module"},
}

static PyObject *
ceph_dispatch_remote(BaseMgrModule *self, PyObject *args)
{
  char *other_module = nullptr;
  char *method = nullptr;
  PyObject *remote_args = nullptr;
  PyObject *remote_kwargs = nullptr;

  // Drop GIL from calling python thread state, it will be taken
  // both for checking for method existence and for executing method.
  PyThreadState *tstate = PyEval_SaveThread();

  // self->py_modules 应该是 ActivePyModules 的一个实例
  auto result = self->py_modules->dispatch_remote(other_module, method,
      remote_args, remote_kwargs, &err);

  PyEval_RestoreThread(tstate);
  return result;
}
```

而ActivePyModules以及具体的call方法如下：

```cpp
// src\mgr\ActivePyModules.cc
PyObject *ActivePyModules::dispatch_remote(
    const std::string &other_module,
    const std::string &method,
    PyObject *args,
    PyObject *kwargs,
    std::string *err)
{
  // 这里就是查找特定的 orchestraotr modules，即 cephadm
  auto mod_iter = modules.find(other_module);
  ceph_assert(mod_iter != modules.end());
  return mod_iter->second->dispatch_remote(method, args, kwargs, err);
}

// src\mgr\ActivePyModule.cc
// 最终调用查找到的 module 的具体方法
PyObject *ActivePyModule::dispatch_remote(
    const std::string &method,
    PyObject *args,
    PyObject *kwargs,
    std::string *err)
{
  // Rather than serializing arguments, pass the CPython objects.
  // Works because we happen to know that the subinterpreter
  // implementation shares a GIL, allocator, deallocator and GC state, so
  // it's okay to pass the objects between subinterpreters.
  // But in future this might involve serialization to support a CSP-aware
  // future Python interpreter a la PEP554

  Gil gil(py_module->pMyThreadState, true);

  // Fire the receiving method
  auto boundMethod = PyObject_GetAttrString(pClassInstance, method.c_str());

  dout(20) << "Calling " << py_module->get_name() << "." << method << "..." << dendl;

  auto remoteResult = PyObject_Call(boundMethod, args, kwargs);
  Py_DECREF(boundMethod);

  return remoteResult;
}
```

至此，就是通过`PyObject_Call`方法调用特定`Orchestrator`的具体实例方法（在这里是cephadm）。实现该方法的注释也说明了，采用了subinterperter的魔法，但是后续应该是要基于PEP554的序列化方式进行修改的。



### #3 ceph-mgr的module command注册解析

目前，为一个ceph-mgr模块注册command命令有两种方式，一是使用 `@CLICommand` 修饰特定的方法，该类定义在 mgr_module.py 文件内，如下：

```python
@CLICommand('antigravity send to blackhole', perm='rw')
def send_to_blackhole(self, oid: str, blackhole: Optional[str] = None, inbuf: Optional[str] = None):
    '''
    Send the specified object to black hole
    '''
    obj = self.find_object(oid)
    if obj is None:
        return HandleCommandResult(-errno.ENOENT, stderr=f"object '{oid}' not found")
    if blackhole is not None and inbuf is not None:
        try:
            location = self.decrypt(blackhole, passphrase=inbuf)
        except ValueError:
            return HandleCommandResult(-errno.EINVAL, stderr='unable to decrypt location')
    else:
        location = blackhole
    self.send_object_to(obj, location)
    return HandleCommandResult(stdout=f'the black hole swallowed '{oid}'")
```

第一个参数是命令的 prefix ，也可以说是命令名称，方法中的参数会自动成为command的一部分，`inbuf` 用于传递 `ceph --in-file` 参数内容。`docstring`的内容会自动成为command的描述，命令在  `ceph --help`时将会显示如下：

```
antigravity send to blackhole <oid> [<blackhole>]  Send the specified object to black hole
```



第二种方法是在module类定义的时候，直接在类属性 `COMMANDS` 定义命令（其命令格式与Ceph Mon 和 admin socket命令的解析定义一样，具体参照 `mon/MonCommands.h`说明）:

```python
COMMANDS = [
    {
        "cmd": "foobar name=myarg,type=CephString",
        "desc": "Do something awesome",
        "perm": "rw",
        # optional:
        "poll": "true"
    }
]
```

命令返回 tuple `(retval, stdout, stderr)`。

- `retval` is an integer representing a libc error code (e.g. EINVAL, EPERM, or 0 for no error), 

- `stdout` is a string containing any non-error output,
- stderr` is a string containing any progress or error explanation output. Either or both of the two strings may be empty.



接下来看看`CLICommand`的定义，prefix就是命令前缀，perm标记读写（估计是ceph在执行时会有些权限控制吧），poll默认是false，表示用户需要重复执行命令以获取结果输出（估计用于异步执行或需要长时间等待的需求吧）：

```python
# pybind/mgr_module.py
class CLICommand(object):
    COMMANDS = {}  # type: Dict[str, CLICommand]

    def __init__(self, prefix: str, perm: str = 'rw', poll: bool = False):
        self.prefix = prefix
        self.perm = perm
        self.poll = poll
        self.func = None  # type: Optional[Callable]
        self.arg_spec = {}    # type: Dict[str, Any]
        self.first_default = -1

    KNOWN_ARGS = '_', 'self', 'mgr', 'inbuf', 'return'
    
    def __call__(self, func: HandlerFuncType) -> HandlerFuncType:
        self.store_func_metadata(func)
        self.func = func
        self.COMMANDS[self.prefix] = self
        return self.func
    
    def call(self,
             mgr: Any,
             cmd_dict: Dict[str, Any],
             inbuf: Optional[str] = None) -> HandleCommandResult:
        kwargs = self._collect_args_by_argspec(cmd_dict)
        if inbuf:
            kwargs['inbuf'] = inbuf
        assert self.func
        return self.func(mgr, **kwargs)
```

在`OrchestratorCli`类中，其命令并没有像官方文档所描述一样使用，而是采用了很多很多`@`修饰。代码非常隐晦难懂。这里作为学习还是简单地对其进行分析吧。

`Orchestrator`的子类基本都使用了`metaclass=CLICommandMeta`的元类，使用修饰器`@_cli_read_command`或者`@_cli_write_command`进行command定义。下面先看下这两个方法的定义，最后再看metaclass。其源码如下：

```python
# pybind/orchestrator/module.py
class OrchestratorCli(OrchestratorClientMixin, MgrModule,
                      metaclass=CLICommandMeta):
    @_cli_write_command('orch host set-addr')
    def _update_set_addr(self, hostname: str, addr: str) -> HandleCommandResult: pass

    @_cli_read_command('orch host ls')
    def _get_hosts(self, format: Format = Format.plain) -> HandleCommandResult: pass

# pybind/orchestrator/_interface.py
def _cli_command(perm: str) -> InnerCliCommandCallable:
    def inner_cli_command(prefix: str) -> Callable[[FuncT], FuncT]:
        return lambda func: handle_exception(prefix, perm, func)
    return inner_cli_command

_cli_read_command = _cli_command('r')
_cli_write_command = _cli_command('rw')
```

由此可知，`_cli_read_command`和`_cli_write_command`只是记录了CLICommand的perm参数值，使用户只需要关心prefix参数，即前面的`orch host ls`这个命令。直白点就相当于：

```python
  _cli_read_command('orch host ls')(_get_host)
= _cli_command('r')('orch host ls')(_get_host)
= inner_cli_command('orch host ls')(_get_host)
= lambda func : handle_exception('orch host ls', 'r', func)(_get_host)
= handle_exception('orch host ls', 'r', _get_host)

# 等价于类 OrchestratorCli._get_hosts 的定义如下，
OrchestratorCli._get_hosts = handle_exception(prefix, perm, _get_hosts)
```

然后 handle_exception修饰器做了如下操作，返回的是一个`wrapper_copy`的`lambda`方法：

```python
# pybind/orchestrator/_interface.py
def handle_exception(prefix: str, perm: str, func: FuncT) -> FuncT:
    @wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        try:
            return func(*args, **kwargs)
        except (OrchestratorError, SpecValidationError) as e:
            # Do not print Traceback for expected errors.
            return HandleCommandResult(e.errno, stderr=str(e))
        except ImportError as e:
            return HandleCommandResult(-errno.ENOENT, stderr=str(e))
        except NotImplementedError:
            msg = 'This Orchestrator does not support `{}`'.format(prefix)
            return HandleCommandResult(-errno.ENOENT, stderr=msg)

    # misuse lambda to copy `wrapper`
    wrapper_copy = lambda *l_args, **l_kwargs: wrapper(*l_args, **l_kwargs)  # noqa: E731
    wrapper_copy._prefix = prefix  # type: ignore
    wrapper_copy._cli_command = CLICommand(prefix, perm)  # type: ignore
    wrapper_copy._cli_command.store_func_metadata(func)  # type: ignore
    wrapper_copy._cli_command.func = wrapper_copy  # type: ignore

    return cast(FuncT, wrapper_copy)
```

也就是最后`OrchestratorCli._get_hosts = wrapper_copy`，而这个`wrapper_copy`具有`_cli_command`属性，细心的就会发现其是一个`CLICommand`类的实例，其属性 func 就是 `wrapper_copy`，这也就相当于在这基础上做了异常处理操作。如下：

```python
try:
    @CLICommand('orch host ls', 'r')
    def _get_host(...): pass
except: pass
```

最后，我们知道ceph-mgr都是通过handle_command来进行分发处理客户端命令的，但是`OrchestratorCli`及其父类都没有实现该方法。所以，我们把目光转向元类`CLICommandMeta`。可以看到，`CLICommandMeta`为其增加了`handle_command`属性。

```python
# pybind/orchestrator/_interface.py
class CLICommandMeta(type):
    """
    This is a workaround for the use of a global variable CLICommand.COMMANDS which
    prevents modules from importing any other module.

    We make use of CLICommand, except for the use of the global variable.
    """
    def __init__(cls, name: str, bases: Any, dct: Any) -> None:
        super(CLICommandMeta, cls).__init__(name, bases, dct)
        dispatch: Dict[str, CLICommand] = {}
        for v in dct.values():
            try:
                dispatch[v._prefix] = v._cli_command
            except AttributeError:
                pass

        def handle_command(self: Any, inbuf: Optional[str], cmd: dict) -> Any:
            if cmd['prefix'] not in dispatch:
                return self.handle_command(inbuf, cmd)

            return dispatch[cmd['prefix']].call(self, cmd, inbuf)

        cls.COMMANDS = [cmd.dump_cmd() for cmd in dispatch.values()]
        cls.handle_command = handle_command
```

同时，可以发现，`CLICommandMeta`也在创建时把所有具有`_cli_command`属性的对象绑定到dispatch中，这样，就可以通过命令的前缀找到对应的处理方法进行执行了。即相当于执行了`CLICommand.call`方法，并将func重定向为wrapper_copy里的方法。



### #4 ceph配置统一管理

在Nautilus后续版本，ceph将其配置文件放在了Mon的KV store内，进行统一管理。这样，用户就可以统一修改配置，不用每个进程去修改了。

```bash
# 查看所有配置
ceph config ls

# get 某个配置 ceph config get *target* *target_key*
ceph config get mon public_network

# set 某个配置 ceph config set *target* *target_key* *target_value*
ceph config set mon public_network 10.0.0.0/24
```

个人设想，并没有实际阅读源码验证，组件（mon，osd，mgr）等启动的时候，会自动通过kv获取配置，加上用户自定义的ceph.conf配置文件的内容，形成自己的config配置。



### #5 关于Ceph容器镜像问题

在docker Hub中，有很多ceph镜像，其可在官方镜像地址中获取。如下：

```txt
https://quay.io/repository/ceph/ceph
https://hub.docker.com/r/ceph
```

这些镜像都是通过 ceph/ceph-container 的github库生成的。

根据手册，我们部署时下载的镜像为 ceph/ceph，但会发现除了ceph/ceph镜像外，还有ceph/daemon、ceph/daemon-base等多种镜像，那么它们之间又有什么关系呢。下面详细描述。



#### CEPH/CEPH 镜像

它是目前官方文档部署时指定的下载镜像，在cephadm脚本中默认，其版本号如下。

| Tag                  | Meaning                                                      |
| -------------------- | ------------------------------------------------------------ |
| vRELNUM              | Latest release in this series (e.g., *v14* = Nautilus)       |
| vRELNUM.2            | Latest *stable* release in this stable series (e.g., *v14.2*) |
| vRELNUM.Y.Z          | A specific release (e.g., *v14.2.4*)                         |
| vRELNUM.Y.Z-YYYYMMDD | A specific build (e.g., *v14.2.4-20191203*)                  |

#### CEPH/DAEMON-BASE 镜像

- 安装了ceph所有组件需要的安装包和依赖，是ceph的一个基础镜像容器。
- 其实就是Ceph/Ceph，就是tag不同
- 备注：马上作为ceph/ceph镜像的一个别名（alias）

| Tag                  | Meaning                                                 |
| -------------------- | ------------------------------------------------------- |
| latest-master        | Build of master branch a last ceph-container.git update |
| latest-master-devel  | Daily build of the master branch                        |
| latest-RELEASE-devel | Daily build of the *RELEASE* (e.g., nautilus) branch    |

#### CEPH/DAEMON镜像

- 在ceph/daemon-base镜像的基础上，整合了很多的BASH脚本，用于ceph-nano 和 ceph-ansible等工具对ceph集群进行管理。

| Tag                  | Meaning                                                 |
| -------------------- | ------------------------------------------------------- |
| latest-master        | Build of master branch a last ceph-container.git update |
| latest-master-devel  | Daily build of the master branch                        |
| latest-RELEASE-devel | Daily build of the *RELEASE* (e.g., nautilus) branch    |



### #6 通过ceph-container制作ceph容器

Ceph各版本的容器都是通过ceph-container代码进行制作的，目前使用的最新tag版本为v6.0.4，其主要为Ceph创建ceph/daemon 和 ceph/daemon-base两个镜像。其工作原理，简单点就是定义了一个宏Dockerfile文件，文件内基本由一些宏参数定义，然后根据stage对其内容进行替换，生成实际可用 docker build 的 Dockerfile。

下面描述一下其用法。

```bash
# 1. 首先， 下载 ceph-container 源
git clone https://github.com/ceph/ceph-container.git

# 2. 然后安装 make 和 git 工具
dnf install make git

# 3. 执行 make， 可执行 make help 打印帮助信息
# 根据 Makefile 的内容，FLAVORS 表示要制作容器对应ceph的版本、系统镜像版本
# -d 为 debug 模式
make FLAVORS='pacific,centos,8' -d

# 4. make 执行成功后，会生成 staging 目录，里面包括最终的 Dockerfile
[root@ceph-container]# ls staging/
pacific-centos-8-x86_64

# 5. 在 staging/pacific-centos-8-x86_64/daemon-base 目录下，执行 docker build
podman build -t ceph/daemon:v16.2.5 ./

# 6. 查看 docker 镜像
podman images
```



为了能够看懂 Makefile里的语法，瞄了一眼 make 官方手册，内容实在太多了，只看了一些大概，尽量进行描述。下面的是 Makefile 主要部分，由于 make 命令执行时会自动发现并执行第一个 `target` ，因此，在没有指定的情况下，应该就是把 `do.images.%` 作为target，而它依赖了 `stage.%` ，因此会先执行 `stage.%`.

```makefile
# ==============================================================================
# Internal definitions
include maint-lib/makelib.mk

# All flavor options that can be passed to FLAVORS
ALL_BUILDABLE_FLAVORS := \
        pacific,centos,8

# ==============================================================================
# Build targets
.PHONY: all stage build build.parallel build.all push push.parallel push.all

stage.%:
        @$(call set_env_vars,$*) sh -c maint-lib/stage.py

# Make daemon-base.% and/or daemon.% target based on IMAGES_TO_BUILD setting
#do.image.%: | stage.% $(foreach i, $(IMAGES_TO_BUILD), $(i).% ) ;
do.image.%: stage.%
        $(foreach i, $(IMAGES_TO_BUILD), \
                $(call set_env_vars,$*); $(MAKE) $(call set_env_vars,$*) -C $$STAGING_DIR/$(i) \
                        $(call set_env_vars,$*) $(MAKECMDGOALS)$(\n))

stage: $(foreach p, $(FLAVORS), stage.$(HOST_ARCH),$(p)) ;
build: $(foreach p, $(FLAVORS), do.image.$(HOST_ARCH),$(p)) ;
push:  $(foreach p, $(FLAVORS), do.image.$(HOST_ARCH),$(p)) ;
```

`stage.%` 的 recipe 是执行 `stage.py` 的脚本，该脚本主要做了两件事情，一是将需要的所有文件拷贝到 staging 的相关目录下，如`staging/pacific-centos-8-x86_64/daemon-base`里的文件有：

```txt
[root@wayne-master daemon-base]# ll
total 100
-rw-r--r--. 1 root root   651 Aug 16 16:55 __CEPH_BASE_PACKAGES__
-rw-r--r--. 1 root root    81 Aug 16 16:55 __CEPHFS_PACKAGES__
-rw-r--r--. 1 root root    52 Aug 16 16:55 __CEPH_GRAFANA_PACKAGES__
-rw-r--r--. 1 root root    55 Aug 16 16:55 __CEPH_IMMUTABLE_OBJECT_CACHE_PACKAGE__
-rw-r--r--. 1 root root   301 Aug 16 16:55 __CEPH_MGR_PACKAGES__
-rw-r--r--. 1 root root    20 Aug 16 16:55 __CRIMSON_PACKAGES__
-rw-r--r--. 1 root root    82 Aug 16 16:55 __CSI_PACKAGES__
-rw-r--r--. 1 root root 11382 Aug 16 16:56 Dockerfile
-rw-r--r--. 1 root root  1886 Aug 16 16:55 Dockerfile.bak
-rw-r--r--. 1 root root  1590 Aug 16 16:55 __DOCKERFILE_CLEAN_COMMON__
-rw-r--r--. 1 root root  4837 Aug 16 16:56 __DOCKERFILE_INSTALL__
-rw-r--r--. 1 root root    39 Aug 16 16:55 __DOCKERFILE_MAINTAINER__
-rw-r--r--. 1 root root   222 Aug 16 16:55 __DOCKERFILE_POSTINSTALL_CLEANUP__
-rw-r--r--. 1 root root   451 Aug 16 16:55 __DOCKERFILE_POSTINSTALL_TWEAKS__
-rw-r--r--. 1 root root     0 Aug 16 16:55 __DOCKERFILE_PREINSTALL__
-rw-r--r--. 1 root root   669 Aug 16 16:55 __DOCKERFILE_TRACEABILITY_LABELS__
-rw-r--r--. 1 root root    30 Aug 16 16:55 __DOCKERFILE_VERIFY_PACKAGES__
-rw-r--r--. 1 root root    92 Aug 16 16:55 __GANESHA_PACKAGES__
-rw-r--r--. 1 root root    38 Aug 16 16:55 __ISCSI_PACKAGES__
-rwxr-xr-x. 1 root root  1208 Aug 16 16:55 Makefile
-rw-r--r--. 1 root root    86 Aug 16 16:55 __RADOSGW_PACKAGES__
-rw-r--r--. 1 root root    21 Aug 16 16:55 __SCIKIT_LEARN__
-rw-r--r--. 1 root root   572 Aug 16 16:55 STAGING_ENV_VARS.md
```

二是对 Dockerfile 里的宏定义进行内容替换，其代码在 `maint-lib/stagelib/replace.py` 内。

```python
def do_variable_replace(replace_root_dir):
    """
    For all files recursively in the replace root dir, do 2 things:
     1. For each __VARIABLE__ in the File, look for a corresponding __VARIABLE__ file in the
        replace root dir, and replace the __VARIABLE__ in the File with the text from the
        corresponding __VARIABLE__ file.
     2. For each __ENV_[<ENV_VAR>]__ variable in the File, replace the variable in the File with
        the content of the environment variable ENV_VAR.
    Variables are allowed to contain nested variables of either type.
    """
    logging.info('    Replacing variables')
    for dirname, subdirs, files in os.walk(replace_root_dir, topdown=True):
        for f in files:
            # do not process the tarball
            # this prevents the following error:
            # UnicodeDecodeError: 'utf-8' codec can't decode byte 0x8b in position 1: invalid start byte
            if "Sree-" in f:
                continue
            if re.match(VARIABLE_FILE_PATTERN, f):
                logging.debug(PARENTHETICAL_LOGTEXT.format(
                    '[Skip __VARIABLE__ replace]', os.path.join(dirname, f)))
                continue
            file_path = os.path.join(dirname, f)
            total_replacements, rendered_text = _do_replace_on_file(file_path, replace_root_dir)
            if total_replacements > 0:
                # Only save if there have been changes
                _save_with_backup(file_path, rendered_text)
```

如上方法注释所示，主要替换 `__VARIABLE__` 和 `__ENV_[<ENV_VAR>]__` 两种类型的宏变量。具体替换的内容和其文件所在位置，可以通过查询 `staging/pacific-centos-8-x86_64/stage.log `。

所有拷贝的文件（或者说宏变量定义文件）大致在两个地方 ： `ceph-releases` 和 `src` 目录里面。src 目录里的Dockerfile 都是通过宏变量编写的，这样做的好处显而易见，可以适配不同版本和系统的依赖，然后制作不同版本的ceph容器镜像，而 Dockerfile 不需要重新定义多份。



**PS：如果想修改ceph镜像，但是又看不懂ceph-container的代码，很简单，make完后，直接修改对应的 Dockerfile 文件即可，这是一条比较简单的路子。**