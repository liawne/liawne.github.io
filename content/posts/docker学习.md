---
title: docker学习
date: 2022-03-10 22:01:26
tags: ['云原生', 'docker', 'cloud']
categories: ['技术', '学习笔记']
description: "介绍容器的实现,讲解docker常用命令和Dockerfile的使用"
---

### \#. 容器是什么 ###
1, LXC(linux container): linux容器

2, 虚拟化类型:
   - 主机级虚拟化:
     Type-I: 直接在硬件上安装一个hypervisor,再在hypervisor上使用虚拟机
     Type-II: virtualbox,vmware workstation,kvm等;先存在一个宿主机,安装好host os,在host os上再安装vmm(virtual machine manager),vmm上再启动虚拟机,还需要自己安装操作系统 
   - 容器级虚拟化: 所有的用户空间不再分属于不同内核,从属于同一内核
     - 主机级虚拟化可以在创建时就天然限制资源(CPU,MEM)
     - CPU资源属于可压缩资源,MEM不属于,当进程申请MEM时没有,则会触发OOM
     - 内核级必须实现一种功能,来限制用户空间中每一个进程所有可用资源总量
       - 可以实现在整体资源上按比例分配,也可以在单一用户空间上,实现做核心绑定,通过CGrouop(control group)来实现

3, 内核和用户空间
   - 能够有产出的都是用户空间的应用程序,像部署一个nginx服务,http服务,再对外提供服务
   - 内核是协调资源的作用,出于通用目的而设计的资源管理平台
   - 现代软件基本都是对内核的系统调用和库调用,应用程序依赖着内核和操作系统

4, kvm等虚拟化
   - 资源需要经过两层调度,虚拟机操作系统层面是一层,宿主机层面又是一层,浪费,存在额外的开销,效率低
   - 优点是资源隔离彻底,某台虚拟机出现问题,不要影响到其他的虚拟机
   - 提高效率的方式: 减少中间层

5, 容器最早出现的形式是freebsd里面的jail,迁移到linux中后,变成了vserver(chroot)

6, 用户空间的隔离类型,都是在内核级别实现的
   - UTS: CLONE_NEWUTS,2.6.19之后支持;主机名和域名,在内核当中以名称命名,并且单独隔离开;可以分别各自使用,而不影响主的,真正的命名空间
   - Mount: CLONE_NEWNS,2.4.19之后支持;文件系统
   - IPC: CLONE_NEWIPC,2.6.19之后支持;信号量,消息队列和共享内存,进程间通信
   - PID: CLONE_NEWPID,2.6.24之后支持;进程号 
   - User: CLONE_NEWUSER,3.8之后支持;用户(组)等相关信息也要做隔离,以root用户说明,只在容器内部是root用户,在宿主机上实际是普通用户
   - Net:CLONE_NEWNET,2.6.29之后支持;网络设备,网络栈,端口等,有自己专有的TCP/IP协议栈

7, 一个操作系统运行存在两棵树: 进程树,文件系统树
   - 进程管理从来都是白发人送黑发人,父进程只有把所有子进程,子进程的子进程...都送完才能放心离开
   - 每个用户空间都有一个init进程(docker中自己定义),当init进程结束时,docker容器实例也停止
   - 一个docker容器中可以存在多个进程,但需要是由同一个最原始的init进程生成

8, linux现已在内核中原生支持名称空间(namespaces),并且通过系统调用向外输出
   - clone()
   - setns(): 进程创建完成后放到某个命名空间中去
   - unshare(): 将进程从某个命名空间中拿出来

9, 要想使用容器,得靠用于支撑用户空间的内核级的内核资源的名称空间隔离来实现

10, CGroup(control group)控制组:
  把系统资源分为多个组,然后把每个组的资源指派或者分配到特定的用户空间的进程上去,分类如下:
  - blkio: 块设备IO
  - cpu: CPU
  - cpuacct: CPU资源使用报告
  - cpuset: 多处理器平台上的CPU集合
  - devices: 设备访问
  - freezer: 挂起或恢复任务
  - memory: 内存使用量及报告
  - perf_event: 对cgroup中的任务进行统一性能测试
  - net_cls: cgroup中的任务创建的数据报文的类别标识符

11, 可以给一个系统上所有运行的进程分门别类的进行分类,每个类就是一个组,叫做控制组
  - 每个控制组内部还可以继续进行划分,分配完成后,每个子组当中的进程自动拥有该控制组分配的资源,除非进一步对子组进行分类

12, 容器级虚拟化相对主机级虚拟化,安全隔离相差很多,相应的为了防止利用漏洞使用其他用户空间的资源,docker也相应的做了一些加固
  - 结合selinux使用

13, 容器实现的三个核心技术:
  - chroot
  - namespaces
  - cgroups

14, 一个容器只运行一个主进程,相应的优缺点
  - 缺点: 
    - 空间占用更多
    - 工具不能共用了,如果需要调试工具,需要在容器中安装
    - 不方便运维,需要专门的编排,不然使用容器部署应用比单独安装服务更麻烦
  - 优点:
    - 方便研发,一次打包,到处运行

15, docker分层构建,联合挂载
  - 每一个镜像都是只读的,多层联合挂载后提供的视图就是当前使用的镜像
  - 每一个容器有一个自己的可写层,在联合挂载的最顶层附加一个新层,能读能写,容器专有的
    - 如果要删除一个文件,且刚好属于上一层镜像中的文件,实现方式为标记为不可见
    - 需要修改一个文件,该文件属于上一层或多层的文件,则通过写时复制的方式实现
  - 持久化使用的数据,通过在容器外部挂载一个共享存储(ceph,glusterfs,iscsi等)实现

16, 在docker的基础之上,能够把这些应用程序之间的依赖关系,从属关系,隶属关系等等反映在启动,关闭时的次序和管理逻辑中,这种功能叫做容器编排
  - docker-compose(单机),docker-swarm(多机集群),docker-machine
  - mesos+marathon
  - kubernetes --> k8s

17, docker后面研发libcontainer,替换了lxc,后面libcontainer进化为runc(run container)
  - lxc(linux container) --> libcontainer --> runc
  - 为了标准化容器相关(容器引擎,容器底层镜像格式)
  - runc,容器运行时的环境标准
  - 容器镜像格式标准OCF(open container format),事实上的工业标准

### \#. docker基础用法 ###
1, 相关名词
  - OCI(open container initiative),开放容器协会,主要目的是围绕容器格式和运行时制定一个开放的工业化标准,由两部分组成:
    - runtime-spec(The Runtime Specification): 运行时标准
    - image-spec(The Image Specification): 镜像格式标准
  - OCF(open container format),开放容器镜像格式
  - runC是一个cli工具,依照OCI标准来运行容器
    - 容器以runC子程序的方式启动,可以被集成到各种不同的操作系统,而且可以不需要守护进程存在
    - runC是从libcontainer发展而来的

2, docker架构,由一下几部分组成
  
  docker可以认为是一个C/S的架构,无论时server端还是client端,都是由一个docker程序来提供,这个docker程序有很多子程序
  - docker client
    - docker pull
    - docker build
    - docker run
  - docker host
    - docker daemon: docker众多子程序中的一个,运行为daemon,即守护进程
      - mysql接入用户请求一般有两种方式,分别是:
        a,标准的TCP/IP协议,IP+端口的方式
        b,使用socket文件,在本机接入
      - 类似于mysql监听,docker也监听在套接字上,为了安全起见,默认时只监听本机,支持三种套接字
        a,ipv4套接字(ipv4地址+端口)
        b,ipv6套接字(ipv6地址+端口)
        c,unix socket file,监听在本机的一个文件上,上面两个没有放开的话,就会只监听本机(127.0.0.1)
    - docker containers
      - 启动容器时,就是基于镜像来启动,在已有的镜像之上,为一个容器启动一个专用的可写层
    - docker images
      - 不确定要用哪些,具体要用的时候再下载
      - 要用的时候就必须要加载到本地
  - docker registry(docker镜像仓库,默认就是dockerhub)
    - 支持http/https,默认必须使用https,为了安全
    - 要使用http,必须在配置文件中明确定义,才能正常使用
    - registry而不是repository,因为registry有三个功能
      - repository: 存储镜像的仓库
      - auth: 用于认证
      - catalog: 提供已有镜像的索引,即仓库名(nginx) + tag(1.10)


3, docker支持restful风格的调用,docker中的资源对象
  - images和containers的关系可以类比与程序(software)和进程(process),镜像是静态的,容器是动态的,存在生命周期
  - restful风格的资源对象都是可以增删改查的
    - images
    - containers
    - networks
    - volumes
    - plugins  
    - other objects

4, docker的安装使用
  - 为了能够正常使用docker,要求环境的内核版本必须要在3.10以上(user namespace是在3.8之后才支持的)
  - 红帽在2.6.32之后可以安装docker,打了补丁补进去了(但还有很多不确定与因素,建议不要跑在centos6上)
  - 建议不要使用Centos-Base.repo这个源中的docker,因为版本较老,有很多软件要求使用新版本docker的特性,但也不要使用最新的docker版本(k8s不支持最新版),使用docker-ce.repo中stable版
  - 当前docker的配置文件是/etc/docker/daemon.json
    - 使用images时,建议使用加速器(docker cn/阿里云)
  - 需要查看当前docker的详细信息,可以使用docker version来查看
    - docker info输出中的Storage Drive这项很重要
      a,docker镜像需要使用到多层构建,联合挂载,这两项要求必须使用特殊的文件系统才能实现,专用的文件驱动
      b,一般包含overlay2和aufs;centos7.4之前没有使用可能和内核版本有关系,之前的还不支持
      c,centos7之前还有使用到dm(弱爆了),就是lvm的实现,使用dm的方式,docker的性能很差,而且还不稳定

5, docker镜像及容器
  - alpine: 专门用于构建非常小的镜像文件的一个微型发行版,能够给程序提供运行环境,但体积非常小(缺少相应的调试工具)
  - 停掉一个容器的方式有:
    - stop: 相当于sigterm
    - kill: 相当于sitkill
  - 启动一个容器时,不能让其跑在后台,不然启动就会停止(一个容器就是为了一个进程,如果跑后台,终端没有任何程序,那docker就认为已经结束了)

### \#. docker镜像管理基础 ###
1, docker镜像含有启动容器所需要的文件系统及其内容,容器镜像用于创建并启动docker容器
  - 采用分层构建的机制,最底层为bootfs,其次为rootfs
  	- bootfs: 用于系统引导的文件系统,包括bootloader和kernel,容器启动完成后会被卸载以节约内存资源
      - bootfs只在容器启动的时候存在,当容器已经挂载了rootfs后,bootfs层将会被卸载(不是被删除)
      - 存在一个问题,如果存在bootfs这一层,那是否拉起的容器可以支持宿主机内核不支持的特性?
    - rootfs: 位于bootfs之上,表现为docker容器的根文件系统
      - 传统模式中,系统启动时,内核挂载rootfs时会首先将其挂载为只读模式,完整性自检完成后将其重新挂载为读写模式
      - docker中,rootfs由内核挂载为'只读'模式,而后通过联合挂载技术而外挂载成一个'可写'层
    - 最上层为'可读写'层,其下的均为'只读'层
      - 容器启动后,所有的写操作都是在最上一层的'可读写'层完成的,当删除一个容器时,可读写层一并被释放

2, 联合挂载文件系统分类
  - aufs(advanced multi-layered unification filesystem): 高级多层统一文件系统
    - 用于为linux文件系统实现'联合挂载'
    - aufs是之前的UnionFS的重新实现,但未能整合到linux内核中(据说是代码很烂),所以如果需要使用,需要给内核打补丁(redhat系不支持,因为不允许)
    - docker最初使用aufs作为容器文件系统层,目前仍作为存储后端之一来支持
  - aufs的竞争产品为overlay,从3.18版本开始被合并到linux内核中
  - docker的分层镜像,除了aufs,docker还支持btrfs,devicemapper和vfs等
    - 在ubuntu系统下,docker默认ubuntu的aufs
    - 在centos7上,用的时devicemapper(性能差还不稳定)
  - 目前推荐使用overlay2
    - overlay是抽象文件系统,属于二级文件系统,需要先有一层文件系统(xfs,ext...)

3,docker registry分类
  - registry 用于保存docker镜像,包括镜像的层次结构和元数据
  - 用户可以自建registry,也可以使用官方的dockerhub
  - 分类:
    - sponsor registry: 第三方registry,供客户和docker社区使用
    - mirror  registry: 第三方registry,供客户使用
    - vendor registry : 由发布docker镜像的供应商提供的registry
    - private registry: 通过设有防火墙和额外的安全层的私有实体提供的registry
  - docker registry中的镜像通常由开发人员制作,而后推送至'公共'或'私有'registry上保存
  - 获取镜像的标准格式: docker pull <repository>:[:port]/[<namespace>/]/<name>:<tag>

4,云原生的解释
  - 面向云环境中去运行这个程序而调云系统本身自有的功能而开发的应用程序,就是为了云环境运行而生的
  - docker容器本身如果通过修改配置文件的方式来改配置,相对麻烦,故想了一种方式,通过运行时传入环境变量的方式来运行(早期的解决方式)

5,docker hub的使用
  docker hub主要提供如下的功能特性:
  - image repository: 镜像仓库
  - automated builds: 自动构建 dockerfile(change/push) --> github --> build image --> docker hub
  - webhooks: 当dockerfile提交成功时,自动触发钩子构建镜像

6,镜像的生成途径
  - Dockerfile
  - 基于容器制作
    - 基于先有容器制作镜像时,建议先暂停该镜像,再commit,防止文件数据的缺失
      ```bash
      # 命令为:
      docker commit -p <container> [repository[:tag]]

      # 替换原有镜像的CMD或者其它参数
      docker commit -p <container> -c "CMD ['/bin/httpd', '-f', '-h', '/data/html']" [repository[:tag]]
      ```
  - Docker Hub automated build(也是基于Dockerfile)

7,不通过registry,直接导入导出镜像文件
  - docker save -o <filename>
  - docker load -i <filename>

### \#. 容器虚拟化网络 ###
1,网络名称空间主要是实现网络设备(网卡,协议栈)的隔离
  - 当网络设备不够用时,可以通过命令生成虚拟网络设备,linux内核支持实现生成两种网络设备:
    - 二层网络设备
      - 成对出现,类似一根网线两头,一根接入网络命名空间,一头接入虚拟交换机(虚拟网桥),就可以模拟一台主机连接到一台交换机的场景
    - 三层网络设备
  - linux内核原生支持二层网桥虚拟设备(软件构建交换机),虚拟化网络
  - linux上实现路由的方式,可以通过直接增加一个netns实现,将相应网桥的网卡接入该netns,然后在netns中配置路由规则即可
    - 桥接不安全,跨机容易出现风暴

2,virtualbox/vmware桥接模式的实现
  - 将物理网卡当作交换机来使用,虚拟机的流量都是通过物理网卡转发过去的,对于物理机本身的流量,是通过对本机增加一个虚拟网卡,物理网卡流量转到该网卡实现的

3,overlay network(叠加网络)
  - 叠加网络(隧道实现),用于实现跨机同网段通信
    - C1(container)在H1(host)上,C1 IP 172.17.0.11
    - C2(container)在H2(host)上,C2 IP 172.17.0.22
    - 通过隧道实现直接通信,省去了两次NAT的动作(先SNAT,再DNAT),类似12inN中的vxlan实现(报文两次三层封装)

4,容器类型,及容器类型
  - bridged container(bridge): 桥接容器,默认模式,每起一个容器,新增一对veth peer,一个接在容器中,一个接入docker0
  - open container(host): 开放式容器,共享宿主机的协议栈,网卡,主机名,进程见通信(UTC,IPC,NET)
  - closed container(none): 封闭式容器,拉起的容器只有lo,不需要网络
  - joined container: 联盟式容器,几个容器共享网络名称空间(UTC,IPC,NET名称空间是共享的)

5,可以使用inspect命令查看任何一个docker对象的情况
  - docker network inspect <network name>
  - docker container inspect <container name>
  - docker volume inspect <volume name>

### \#. 容器网络 ###
1,容器间可以共享IPC,UTC,NET命名空间

2,可以手动操作网络名称空间,因为由ip命令
  - 想模拟容器的网络名称空间,只靠ip命令就能完成

3,容器启动时可以指定的一些参数
  - -h指定hostname
  - --dns指定dns服务器(/etc/resolv.conf文件会记录)
  - --add-host直接外部注入host解析内容

4,opening inbound communication(如何开放nat桥桥接式服务到宿主机外部提供服务,可以使用-p参数来实现)
  - 使用-p来实现,有以下@sh几种方式:
    ```bash
    # -p <containerPort>: 将指定的容器端口映射至主机所有地址的一个动态端口
    docker run --name myweb --rm -p 80 mageedu/httpd:v0.2
    # -p <hostPort>:<containerPort>: 将容器端口映射至指定的主机端口
    docker run --name myweb --rm -p 172.20.0.67::80 mageedu/httpd:v0.2
    # -p <ip>::<containerPort>: 将指定的容器端口映射至主机指定<ip>的动态端口
    docker run --name myweb --rm -p 80:80 mageedu/httpd:v0.2
    # -p <ip>:<hostPort>:<containerPort>:将指定的容器端口映射至主机指定<ip>的端口<hostPort>
    docker run --name myweb --rm -p 172.20.0.67:8080:80 mageedu/httpd:v0.2
    # 动态端口指的是随机端口,具体的映射结果可以使用docker port命令查看

    ## 使用-P时,可以不用指定端口,默认会将容器监听的端口暴露出去
    ```

5,joined containers(联盟式容器)
  - 联盟式容器是指使用某个已存在容器的网络接口的容器,接口被联盟内的各容器共享使用; 因此联盟式容器间完全无隔离
    ```bash
    # 创建一个监听2222端口的http服务容器
    docker run -d -it --rm -p 2222 busybox:latest /bin/httpd -p 2222 -f
    # 创建一个联盟式容器,并查看其监听的端口
    docker run -it --rm --net container:web --name joined busybox:latest netstat -tan

    ## another example
    docker run --name b1 -it --rm busybox
    docker run --name b2 --network container:b1 -it --rm busybox
    ```
  - 联盟式容器彼此间虽然共享一个网络名称空间,但其他名称空间如User,Mount等还是隔离的
  - 联盟式容器彼此间存在端口冲突的可能性,因此通常只会在多个容器上的程序需要程序loopback接口互相通信,或对某已存的容器的网络属性进行监控时才使用此种模式的网络模型

6,可以自定义docker的一些配置(修改/etc/docker/daemon.json)
  - 定义docker0网桥的IP段及其他相关配置
    ```bash
    ## docker0使用的网段,会自动识别出所属的网段
    "bip": "192.168.1.5/25"
    ## mtu
    "mtu": 1500
    ## 如果想让容器不获取宿主机的dns,获取指定的dns,可以如下配置
    "dns": ["10.20.1.2", "10.20.1.3"]
    ## 设定容器的默认网关
    "default-gateway": "10.20.1.1"
    ```

7,docker默认只支持在本机使用,通过/var/run/docker.sock通信
  - -H可以指定指向的docker服务器
  - 可以放开选项,在daemon.json文件中放开
    ```bash
    ## 增加本机2375端口的监听
    "hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]
    ```

8,自定义创建docker桥接桥
  - 示例如下:
    ```bash
    ## 新增mybr0桥
    docker network create -d bridge --subnet "172.26.0.0/16" --gateway "172.26.0.1" mybr0
    ## 新起一个容器,网络设置为mybr0
    docker run -it --name t1 --net mybr0 busybox:latest
    ## 默认加入docker0网桥的命令是(default,不加默认就是)
    docker run -it --name t2 --net bridge busybox:latest
    ```

### \#. 容器存储 ###
1,为什么要使用data volume
  - docker镜像由多个只读层叠加而成,启动容器时,docker会加载只读镜像层并在镜像栈顶添加一个读写层
  - 如果运行中的容器修改了现有的一个已经存在的文件,那该文件将会从读写层下面的只读层复制到读写层,该文件的只读版本仍然存在,之时已经被读写层中该文件的副本所隐藏,此即"写时复制(COW)"机制
  - 经过多层联合挂载,IO性能肯定不高,对于一些IO要求比较高的应用,这种联合挂载的方式肯定不满足条件

2,使用容器的可读写层完成相关读写
  - 缺点:
    - 删除容器时,所有内容都被删除了
    - 实现数据存取时,效率比较低
  - 若想绕故上述的限制,可以通过存储卷的方式来实现
    - 可以理解为在特权名称空间中(宿主机),在宿主机本地找一个文件系统上的目录(或者文件)与容器内的某一文件系统目录(或文件)建立绑定关系
    - 绑定的实现是通过mount的bind参数实现的,绑定之后,容器写入相应的目录就是写到了宿主机上对应目录下
    - 使得容器内的进程在数据保存时,能跳过或绕过容器文件系统的限制,从而与宿主机的文件系统建立关联关系,使得可以在容器内与宿主机共享数据或者内容(宿主机可以直接访问容器内内容,宿主机可以给容器直接提供内容)
  - 使用了volume的docker容器,run时加--rm参数,实现就类似进程了
    - 进程是程序的实例(类比容器是镜像的实例)
    - 进程执行过程,生成的数据保存在文件系统,停止后,数据仍然还在(类比docker单独挂载了volume保存数据后,容器停止,数据已经保存在volume中)

3,容器启动时,参数会比较多,当需要完全复现一个容器时,可能不记得当时的启动参数了,这个功能可以通过容器编排工具提供

4,有状态应用和无状态应用
  - 有状态应用是当前这次连接请求是和之前有关联关系的,大多数有状态应用需要持久化数据的
  - 无状态应用是当前请求和之前没有关联关系

5,docker的存储卷默认情况下是使用其所在的宿主机之上的本地文件系统目录的
  - 存储卷是关联到宿主机上的一个目录而已,要单独挂载磁盘,需要先在宿主机上完成才行

6,为什么使用数据卷
  - 关闭并重启容器,其数据不受影响;但删除容器,则其更改将会全部丢失
  - 存在的问题:
    - 存储于联合文件系统中,不利于宿主机访问
    - 容器间共享数据不便
    - 删除容器其数据会丢失
  - 解决方案: 卷(volum)
    - 卷是容器上一个或多个'目录',此类目录可以绕过联合文件系统,与宿主机上的某个目录绑定

7,卷提供了几项有用的特性以支持持久化存储或共享存储
  - volume在容器创建之初就会创建,由base image提供的卷中的数据会于此期间完成复制
    - 存储卷可以在容器之间复用
    - 对卷中的数据更改实时生效
    - 对卷中的数据更改不影响容器镜像的更新
    - 容器删除后,卷中的数据仍然存在
  - volume的初衷是独立于容器的生命周期实现数据持久化,因此删除容器既不会删除卷,也不会对那些未被应用的卷做垃圾回收操作

8,卷为docker提供了独立于容器的数据管理机制
  - 可以把镜像想象成静态文件,例如程序,把卷类比于动态内容,例如数据,于是,镜像可以重用,而卷可以共享
  - 卷实现了'程序(镜像)'和'数据(卷)'的分离,以及'程序(镜像)'和'制作镜像的主机'分离,用户制作镜像时无需再考虑镜像运行的容器所在的主机的环境

9,卷的类型,docker有两种类型的卷,每种类型的卷都在容器中存在一个挂载点,但其在宿主机上的位置有所不同
  - bind mount volume(绑定挂载卷): 在宿主机上需要指定挂载路径,在容器中也要指定挂载路径,两个已知路径建立关联关系
    - 指定的宿主机路径不存在,默认情况下,会自动创建出来
      ```bash
      docker run -it -v HOSTDIR:VOLUMEDIR --name bbox2 busybox:latest
      docker inspect -f {% raw %}{{.Mounts}}{% endraw %} bbox2
      ```
  - docker-managed volume(docker管理卷):只需要指定容器中的挂载路径,宿主机路径不指定,由docker自动指定一个空目录,与挂载目录建立关联关系
    ```bash
    docker run -it -v /data --name bbox1 busybox:latest
    docker inspect -f {% raw %}{{.Mounts}}{% endraw %} bbox1
    ```
  - 使用其他容器的卷
    ```bash
    docker run -it --name bbox1 --volume-from bbox2 busybox:latest
    ```

10,使用docker inspect去查看容器的相关信息时,可以使用go模板(go template),类似与ansible的jinja2模板
  - docker inspect -f {% raw %}{{.Mounts}}{% endraw %} b2
  - docker inspect -f {% raw %}{{.NetworkSettings.IPAddress}}{% endraw %} b2

11,joined-container使用共享卷,实现nmt(nginx+mysql+tomcat)
  - 可以先启动一个基础支撑容器(互联网上有专门的这种容器),先提供一个卷,给后面的容器使用
  - nmt三个容器共用一个存储卷,共用(net,uts,ipc)名称空间
  - 只有nginx对外暴露端口,tomcat和mysql监听在lo上
    ```bash
    docker run -it --name infracon -it -v /data/infracron/volume/:/data/web/html/ busybox
    docker run -it --name nginx --network container:infracon --volume-from infracon nginx:latest
    docker run -it --name tomcat --network container:infracon --volume-from infracon tomcat:latest
    docker run -it --name mysql --network container:infracon --volume-from infracon mysql:latest
    ```
  - 对于这种需求,可以使用docker-compose来实现,主要功能是单击编排(还可以做镜像,镜像现做,容器现启动)

### \#. Dockerfile详解(一) ###
1,生产环境下使用nginx场景,修改配置生效的几种方式
  - docker exec CONTAINER vi, reload进入容器修改配置文件,再reload服务
  - 挂载存储卷,配置文件在存储卷中,但修改之后还是需要reload操作
  - 自己制作镜像

2,制作镜像的两种方式
  - 基于已有镜像制作,修改完成后,提交可读写层
    - 日常运维修改很频繁,相应的提交也就很频繁
    - 版本管理混乱,后续想制作同样的镜像很难
  - Dockerfile
    - Dockerfile仅仅就是构建docker镜像的源代码
    - Dockerfile的内容就是一个用户可以在命令行下操作,用来装配好docker镜像的指令集合
    - docker可以通过读取Dockerfile的内容,自动构建docker镜像

3,Dockerfile的组成
  - 格式:
    - INSTRUCTION其实并不区分大小写,但是为了规范且和参数区别,一般大写
    - docker是按照顺序执行Dockerfile中的INSTRUCTION的 
    - Dockerfile的第一行必须是FROM开头,用来指定需要构建的镜像的基础镜像是哪个
    - Dockerfile中可以执行很多shell命令,但是这个命令是docker容器中的命令,不是宿主机上的命令
      ```dockerfile
      # Comment
      INSTRUCTION arguements
      ```
  - 构建规范和逻辑
    - 要有单独的工作路径(目录)
    - 工作目录中,要有Dockerfile文件
    - 需要使用到的文件,需要存放在工作目录下,Dockerfile中引用的路径都是以当前工作目录为起始路径
    - 对于子目录中不需要拷贝的文件,可以使用.dockerignore文件来罗列(文件排除列表)
      - 一行一个文件
      - 支持通配符
  - 环境变量替换
    - 环境变量(以ENV开头的部分)同样能被Dockerfile解析
    - 环境变量在Dockerfile中引用的格式时$variable_name或者${variable_name}
    - ${variable_name}同样支持少量的shell变量替换操作
      - ${variable_name:-word}:变量为空或者变量未设置时,引用的值就是word,变量有值的时候,则使用变量的值
      - ${variable_name:+word}:变量有值则显示为word,没值就什么都没有

4,Dockerfile中的指令
  - FROM
    - FROM指令是最重要的一个,且必须是Dockerfile文件开篇的第一个非注释行,用于为影响文件构建过程指定基准镜像,后续的指令基于此基准镜像提供的运行环境
    - 基准镜像可以使用任何镜像文件,默认情况下,docker build会在docker主机上查找指定的镜像文件,在其不存在时,则会从docker hub registry上拉取所需要的镜像文件(如果找不到指定的镜像文件,则返回报错)
    - 语法:
      ```dockerfile
      FROM <repository>:<tag>
      ## 或者
      FROM <repository>@<digest>
      ## digest是镜像的哈西码,为的是避免同名镜像被替换
      ```
  - MAINTANIER(depreacted已废弃)
    - 用于让镜像制作者提供本人信息
    - Docker并不限制MAINTANIER出现的位置,建议放在FROM后面
    - 语法:
      ```dockerfile
      MAINTANIER <author's details>
      ```
  - LABEL
    - LABEL指令用于给镜像增加元数据
    - 一个镜像可以有多个LABEL
    - 可以在一行中指定多个label(键值对)
    - 语法:
      ```dockerfile
      LABEL <key>=<value> <key>=<value> <key>=<value>
      ## MAINTANIER可以作为LABEL中的一个键值数据存在
      ```
  - COPY
    - 用于从Docker主机复制文件至创建的新映像文件
    - 语法如下:
      ```dockerfile
      COPY <src>...<dest>或者
      COPY ["<src>",..."<dest>"]
          # <src>:要复制的源文件或者目录,支持使用通配符
          # <dest>:目标路径,即正在创建的image的文件系统路径;建议为<dest>使用绝对路径,否则,COPY指定则以WORKDIR为其起始路径
          # 注意: 在路径中有空白字符时,通常使用第二种格式
      ```
    - 文件复制准则:
      - <src>必须是build上下文中的路径,不能是父目录中的文件
      - 如果<src>是目录,则其内部文件或子目录会被递归复制,但<src>目录自身不会被复制
      - 如果指定了多个<src>,或在<src>中使用了通配符,则<dest>必须是一个目录,且必须以/结尾
      - 如果<dest>事先不存在,他将会被自动创建,这包括岐阜目录路径
  - ADD
    - 类似于COPY指令,ADD支持使用TAR文件和URL路径
    - 语法如下:
      ```dockerfile
      ADD <src>...<dest>或者
      ADD ["<src>",..."<dest>"]
      ```
    - 文件复制准则:
      - 同COPY指令
      - 如果<src>为URL且<dest>不以/结尾,则<src>指定的文件将被下载并直接被创建为<dest>;如果<dest>以/结尾,则文件名URL指定的文件将被下载下来并保存为<dest>/<filename>
      - 如果<src>是一个本地文件系统上的压缩格式的tar文件,它将被展开为一个目录,其行为类似于'tar -x'命令;然而,通过URL获取到的tar文件将不会自动展开
      - 如果<src>有多个,或其间接或直接使用了通配符,则<dest>必须是一个以/结尾的目录路径;如果<dest>不以/结尾,则其被视作一个普通文件,<src>的内容将被直接写入到<dest>
  - WORKDIR
    - 用于为Dockerfile中所有的RUN,CMD,ENTRYPOINT,COPY和ADD指定设定工作目录
    - 语法如下:
      ```dockerfile
      WORKDIR <dirpath>
          # 在Dockerfile文件中,WORKDIR指令可以出现多次,其路径也可以为相对路径,不过,其是相对此前一个WORKDIR指令指定的路径
          # 另外,WORKDIR也可调用由ENV指定定义的变量
      # 示例:
      WORKDIR /var/log
      WORKDIR $STATEPATH
      ```
  - VOLUME
    - 用于在image中创建一个挂载点目录,以挂载Docker host上的卷或其他容器上卷
    - 语法如下:
      ```dockerfile
      VOLUME <mountpoint>或者
      VOLUME ["<mountpoint>"]
      ```
    - 如果挂载点目录下此前在文件存在,docker run命令会在卷挂载完成后将此前的所有文件复制到新挂载的卷中
    - 与run命令使用的volume不同,Dockerfile中指定VOLUME只能指定docker挂载目录
  - EXPOSE
    - 用于为容器打开指定要监听的端口以实现与外部通信
    - 语法如下:
      ```dockerfile
      EXPOSE <port>[/<protocol>][<port>[/<protocol>]...]
          # <protocol>用于指定传输层协议,可为tcp或者udp二者之一,默认为TCP协议
          # 注意:写在Dockerfile中的端口暴露,只是提示可以暴露,而不是以该镜像启动的容器就暴露端口
          # 在需要暴露时,docker run指定-P选项,会读取到哪些端口需要暴露,并将其暴露出去
          # 示例:(构建惊险的Dockerfile中已经存在EXPOSE 80/tcp)
          #  docker run --name tinyweb1 --rm tinyhttpd:v0.1-6 /bin/httpd -f -h /data/web/html
          #  docker port tinyweb1 --> none
          #  docker run --name tinyweb1 --rm -P tinyhttpd:v0.1-6 /bin/httpd -f -h /data/web/html
          #  docker port tinyweb1 --> 80/tcp -> 0.0.0.0:32768
      ```
    - EXPOSE可以一次指定多个端口,如:
      - EXPOSE 11211/udp 11211/tcp
    - 写在镜像中的端口,在运行容器时没有指定则被称为待暴露端口,而不会真正暴露,除非在运行容器时加上-P(默认暴露端口)选项
  - ENV
    - 用于为镜像定义所需的环境变量,并可以被Dockerfile文件中位于其后的其他指令(如ENV,ADD,COPY)所调用
    - 语法如下:
      ```dockerfile
      ENV <key> <value> 或者
      ENV <key>=<value>...
      # 第一种格式中,<key>之后的所有内容均会被视作其<value>的组成部分,因此,一次只能设置一个变量
      # 第二种格式可以用于一次设置多个变量,每个变量为一个"<key>=<value>"的键值对,如果<value>中包含空格,可以以反斜线(\)进行转意,亦可通过对<value>加引号进行标识;另外,反斜线也可用于续行.
      # 定义多个变量时,建议使用第二种方式,以便在同一层中完成所有功能
      ```
    - 在Dockerfile中ENV定义的变量,可以在依照其构建出的镜像启动的容器中直接使用 

5,Dockerfile中请惜字如金,因为每一条指令都会生成一个新的层

### \#. Dockerfile详解(二) ###
1,在docker的上下文当中,有一个重要的特点,一个容器只是为了运行单个程序或者单个应用

2,在命令行下通过&符放后台执行的程序都是当前shell的子进程,当退出当前shell时,该shell的子进程会被杀掉,相应的该shell下的后台进程也被关闭
  - 如果要实现退出shell时,仍然可以正常运行,需要增加nuhup command &,剥离command与当前shell的关系,相当于把启动这个进程的父进程安排给了init
  - 在一个用户空间中,不是以shell的子进程去启动一个程序,很多的经验和参数则不能使用
    ```bash
    ls /var/*                      # --> 不在shell下执行的话,*无法被内核解析
    # 变量替换                        --> 同样不能被解析
    # 管道,输入输出重定向等             --> 无法识别
    ```

3,Dockerfile中的指令
  - CMD
    - 在docker容器中,需要让容器运行的进程变为pid为1的进程
      - 在容器当中,希望以shell的方式启动一个主进程
        > 先在用户空间中启动一个shell(pid为1),在shell中以剥离终端的方式启动需要启动的主进程;解决主进程pid不为一的方式为,在shell中exec command --> exec顶替shell的pid=1,取代shell进程,shell退出,command成为pid=1的进程

      - 在docker容器中,可以不基于shell,直接启动进程;也可以基于shell,通过shell启动进程,但要基于shell启动,并且不违背这个主进程id为1的原则,需要通过exec来实现
    - CMD是定义一个容器启动时,默认需要执行的程序
      - 类似于RUN指令,CMD指令也可用与运行任何命令或应用程序,不过二者的运行时间点不同
        - RUN指令运行于映像文件构建过程中,而CMD指令运行于基于Dockerfile构建出的新镜像文件启动为一个容器时
        - CMD指令的首要目的在于为启动的容器指定默认要运行的程序,且其运行结束后,容器也将终止;不过CMD指定的命令可以被docker run的命令行选项所覆盖
        - 在Dockerfile中,可以存在多个CMD指令,但仅有最后一个会生效
    - 语法:
      ```dockerfile
      CMD <command>
      CMD ["<executable>", "<param1>", "<param2>"]
      CMD ["<param1>", "<param2>"]
      # 前两种语法格式的意义同RUN
      # 第三种则用于为ENTRYPOINT指令提供默认参数
      ```
  - RUN
    - 用于指定docker build过程中运行的程序,其可以是任何命令
    - 语法:
      ```dockerfile 
      RUN <command>
      RUN ["<executable>", "<param1>", "<param2>"]
      # 第一种格式中,<command>通常是一个shell命令,且以'/bin/sh -c'来运行它,这意味着此进程在容器中的PID不为1,不能接收Unix信号,因此,当使用docker stop <container>命令停止容器时,此进程接收不到SIGTERM信号
      RUN ['/bin/bash', '-c', '<executable>', '<param1>']
      # 第二种格式中的参数是一个JSON格式的数组,其中<executable>为要运行的命令,后面的<paramN>为传递给命令的选项或者参数;然而,此种格式指定的命令不会以'/bin/sh -c'来发起,因此常见的shell操作如变量替换以及通配符(?,*等)替换将不会进行;不过,如果要运行的命令依赖于此shell特性的话,可以将其替换为类似下面的格式
      ```
    - 示例用法:
      ```dockerfile
      ENV WEB_DOC_ROOT='/data/web/html/'
      CMD /bin/httpd -f -h ${WEB_DOC_ROOT}                                 # ---> 可用
      CMD ["/bin/httpd", "-f", "-h ${WEB_DOC_ROOT}"]                       # ---> 不可用,如上b所述,该写法不是在shell中执行,不能解析${WEB_DOC_ROOT}
      CMD ["/bin/sh", "-c", "/bin/httpd", "-f", "-h ${WEB_DOC_ROOT}"]      # ---> 可用
      ```
  - ENTRYPOINT
    - 类似CMD指令的功能,用于为容器指定默认运行程序,从而使得容器像是一个单独的可执行程序
    - 与CMD不同的是,由ENTRYPOINT启动的程序不会被docker run命令行指定的参数所覆盖,而且,这些命令行参数会被当作参数传递给ENTRYPOINT指定的程序
      - docker run命令的--entrypoint选项的参数可以覆盖ENTRYPOINT指令指定的程序
    - 语法:
      ```dockerfile
      ENTRYPOINT <command>
      ENTRYPOINT ["<executable>", "<param1>", "<param2>"]
      
      # 示例:
      ENTRYPOINT /bin/sh -c           # --> inspect image会发现存在两层/bin/sh -c,因为默认这种格式就是/bin/sh -c
      ENTRYPOINT ['/bin/sh', '-c']    # --> inspect image正常
      ```
    - docker run命令传入的命令参数会覆盖CMD指令的内容并附加到ENTRYPOINT命令最后,作为其参数使用
    - ENTRYPOINT在Dockerfile中也可以存在多个,但仅有最后一个会生效
  - USER
    - 用于指定运行image时的或运行Dockerfile中任何RUN,CMD或ENTRYPOINT指令指定的程序时的用户名或UID
    - 默认情况下,container的运行身份为root用户
    - 语法:
      ```dockerfile
      USER <UID>|<UserName>
      # 需要注意的是,<UID>可以为任意数字,但实践中必须要为/etc/passwd中某用户的有效UID,否则,docker run命令将运行失败
      ```
  - HEALTHCHECK
    - HEALTHCHECK指令是通过给定内容给docker,令其检查容器是否还在正常运行,检查主进程工作状态健康与否
    - 语法:
      ```dockerfile
      HEALTHCHECK [OPTION] CMD command      # --> 通过在容器中运行command来检测容器是否运行正常
        - 可在CMD前指定的option
          --interval=DURATION(default: 30s)
          --timeout=DURATION(default: 30s)
          --start-period=DURATION(default: 0s) -->等待多长时间(docker run容器起来后,可能还需要初始化动作,这个时候不应该算到检测内容中)
          --retries=N(default: 3)
        - command执行的返回值反映了容器的健康状态,可能的value包括如下:
          0:success    --> 容器健康且能提供服务
          1:unhealthy  --> 容器未能正常运行
          2:reserved   --> 不使用该返回值(预留的,没什么意义)
        # 示例内容:
          HEALTHCHECK --interval=5m --timeout=3s CMD curl -f http://localhost/ || exit 1
      HEALTHCHECK NON禁用任何从容器尽享中继承的healthcheck
      ```
  - STOPSIGNAL
    - STOPSIGNAL指令用于指定发送给容器,用于停掉的容器的system call signal
    - 语法:
      ```dockerfile
      STOPSIGNAL singal
      ```
  - ARG
    - 定义变量,但只能在docker build过程中使用
      - ARG后定义的变量可以在docker build过程中被替换,这种场景适用于一个Dockerfile可以接收不同参数,构建不同镜像的
    - 语法:
      ```dockerfile
      ARG <key>=<value>
      # 使用了之后,可以在构建镜像时使用--build-arg <key>=<value2>替换
      ```
  - ONBUILD
    - 在Dockerfile中定义一个触发器
    - Dockerfile用于build映像文件,此映像文件亦可作为base image被另一个Dockerfile用作FROM指令的参数,并以之构建新的镜像文件
    - 在后面的这个Dockerfile中的FROM指令在build过程中被执行时,将会触发创建其base image的Dockerfile文件中的ONBUILD指令定义的触发器
    - 语法:
      ```dockerfile
      ONBUILD <INSTRUCTION>
      ```
    - 尽管任何指令够可注册成为触发器指令,但ONBUILD不能自我嵌套,且不会触发FROM和ENTRYPOINT指令
    - 使用ONBUILD指令的Dockerfile构建的镜像应该使用特殊的标签,例如ruby:2.0-onbuild
    - 在ONBUILD指令中使用ADD或者COPY指令应该格外小心,因为新构建过程的上下文缺少指定的源文件时会报错

4,nginx Dockerfile示例
  - 内容如下:
    ```dockerfile
    FROM nginx:1.14:alpine
    LABLE maintainer="MageEdu <mage@magedu.com>"

    ENV NGX_DOC_ROOT='/data/web/html/'

    ADD entrypoint.sh /bin/

    CMD ['/usr/sbin/nginx', '-g', 'daemon off;']
    ENTRYPOINT ['/bin/entrypoint.sh']
    ##################################
    cat entrypint.sh
    #!/bin/bash
    #
    cat > /etc/nginx/conf.d/www.conf <<EOF
    server {
        server_name $HOSTNAME;
        listen ${IP:-0.0.0.0}:${PORT:-80};
        root ${NGX_DOC_ROOT:-/usr/share/nginx/html};
    }
    exec "$@"
    EOF
    ```

5,在写Dockerfile时,注意json传入时必须要写双引号,单引号会报错

### \#. Docker私有registry ###
1,registry用于保存docker镜像,包括镜像的层次结构和元数据

2,启动registry有几种方式
  - docker run -d xxxx registry
  - yum install docker-registry(在epel源中,实际名称为docker-distribution)
    - 实际上时一个python开发的web自运行程序
    - 数据保存目录在/var/lib/docker下,配置文件在/etc下,可以通过更改/etc下配置文件修改镜像保存位置和监听端口

3,建议使用harbor来管理镜像,搭建harbor需要使用到docker-compose
  - harbor有两种安装方式,离线和在线
    - 离线版安装需要下载tgz文件,文件较大,需要下载一段时间
    - 在线版是在更改完配置文件后,执行安装脚本,安装过程中下载相应镜像
    - 不管离线还是在线安装,需要先完成harbor.cfg文件的配置,而后才能进行部署
  - harbor是一个多容器服务,需要使用到各种容器镜像,编排内容在docker-compose.yml中定义
    - 一般执行docker-compose命令,会在docker-compose.yml文件所在目录执行
    - 根据docker-compose.yml文件启动相应服务  --> docker-compose start
    - 根据docker-compose.yml文件关闭相应服务  --> docker-compose stop
    - 查看docker-compose的帮助命令           --> docker-compose help

4,harbor的一些特性:
  - 多租户登录和验证
  - 安全和风险验证
  - 日志监控
  - RBA(role-base control)
  - 实例间镜像拷贝
  - 扩展API和web UI界面

### \#. Docker的资源限制 ###
1,默认情况下,容器是没有资源限制的,可以使用完宿主机上内核分配给该容器的所有资源

2,docker提供了一个途径,可以用来控制容器可以使用多少CPU,memory,块设备IO(限制有限);可以通过设置运行时(runtime)配置
  - 内存是不可压缩资源,当容器需要使用的内存分配不到时,可能触发容器内的oom
  - CPU的分配可以通过少分配来实现

3,这些资源限制功能的实现依赖于内核是否支持来实现

4,在linux主机上,如果内核探测到但前内存无法满足重要系统功能实现时,会抛出OOME(out of memory exception),并开始杀掉进程来释放空间
  - 一旦发生oome,任何进程都有可能被杀死,包括docker daemon
  - 为此docker特地调整了docker daemon的OOM优先级,以免被内核杀掉,但容器内的优先级并未被调整
  - 每个进程有一个oom_adj,用来计算oom权重(优先级)

5,限制一个容器可以使用的内存
  - -m或者--memory:最少为4M
  - --memory-swap:不使用--memory时,不能使用该项
    ```
        ------------------------------------------------------------------------------------------------
        | --memory-swp | --memory  | 功能
        ------------------------------------------------------------------------------------------------
        |   正数S      |  正数M    | 容器可用总内存为S,其中ram为M,swap为(S-M),若S=M,则无可用swap资源
        ------------------------------------------------------------------------------------------------
        |    0         |  正数M    | 相当于未设置swap(unset)
        ------------------------------------------------------------------------------------------------
        |    unset     |  正数M    | 若主机(Docker Host)启用了swap,则容器的可用swap为2M
        ------------------------------------------------------------------------------------------------
        |    -1        |  正数M    | 若主机(Docker Host)启用了swap,则容器的可用swap可使用到主机上所有swap
        ------------------------------------------------------------------------------------------------
        # 注意:在容器内部使用free命令可以看到的swap空间并不具有其所展现出的空间指示意义
    ```
  - --memory-swappiness
  - --memory-reservation
  - --kernel-memory
  - --oom-kill-disable

6,限制一个容器可以使用的CPU
  - 默认情况下,每个容器可以使用的CPU时间片是不受限制的
  - 可以通过多种方式来限制容器可以使用的CPU资源
  - 大多数用户通过CFS(completely fair scheduler)来调度
  - 在1.13之后版本的docker上,还可以使用realtime调度器
  - --cpus=<value> 限定一个容器最多能够使用几核
  - --cpu-period=<value>
  - --cpu-quota=<value>
  - --cpuset-cpus 限制在哪些核上
  - -- cpu-shares 配置为共享方式(各个容器共享宿主机CPU)

7,下载压测镜像,可以在dockerhub上搜索stress
