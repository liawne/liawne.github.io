---
title: ansible学习
date: 2022-03-05 12:07:21
tags: 
- ansible
- notes
description: "讲解ansible及ansible-playbook的使用"
---
# learning ansible
*本文内容参考自[朱双印ansible笔记](https://www.zsythink.net/archives/2481)*

### 配置主机清单 ###
主机清单配置有两种方式
#### \# ini方式
```ini
# 1,如下示例:
[ctl]
10.168.0.2
10.168.0.3
10.168.0.4

[nova:children]
ctl

[nova-compute:children]
nova
```
#### \#  yaml方式
```yaml
# 1,如下示例:
all:
  children:
    k8s:
      hosts:
        master:
          ansible_host: 192.168.110.11
          ansible_port: 22
          ansible_user: root
          ansible_pass: '123.com'
        node2:
          ansible_host: 192.168.110.22
          ansible_port: 22
          ansible_user: root
          ansible_pass: '123.com'
        node3:
          ansible_host: 192.168.110.33
          ansible_port: 22
          ansible_user: root
          ansible_pass: '123.com'
```

### 常用模块 ###
#### \#  文件类操作
1,copy: 将ansible主机上的文件拷贝至远端主机
  - 同fetch类似,不过操作动作相反,fetch是从远端主机拿文件到ansible主机
  - 可使用content直接代替src
  - 可备份远端重名文件
  - 可设定拷贝文件的权限

2,file: 完成一些对文件的基本操作,包括创建,删除,修改文件和目录的权限等
  - path是必须项,需要和state结合使用
  - state参数是核心,对不同类型的文件对应的动作不相同,path如果是文件(touch),是目录(directory); absent时不用管path是什么(目录或者文件)

3,blockinfile: 在文件中插入一段文本,这段文本是被标记过的,方便我们后面对标记内容的操作(删除,修改等)
  - 对文件中块的操作,文件位置定位使用insertbefore,insertafter
  - 使用block/content制定块内容
  - 使用该模块marker会在插入内容前后增加marker内容
  - 可使用regex,匹配文件中内容进行操作
  - 包含backup相关内容

4,lineinfile: 确保某一行文本存在于文件中,或不存在与文本中,可使用regex进行替换
  - line指定行内容
  - 包含regex参数,使用regex来匹配相应的行,当替换文本时，如果有多行文本都能被匹配，则只有最后面被匹配到的那行文本才会被替换，当删除文本时，如果有多行文本都能被匹配，这么这些行都会被删除
  - backrefs参数：默认情况下，当根据正则替换文本时，即使regexp参数中的正则存在分组，在line参数中也不能对正则中的分组进行引用，除非将backrefs参数的值设置为yes
  - 插入内容为insertbefore,insertafter, 包含backup相关内容

5,find: 类似find命令,可在远端机器中找到相应文件
  - 使用方式ansible-doc -s find查看

6,replace: replace模块可以根据我们指定的正则表达式替换文件中的字符串，文件中所有被正则匹配到的字符串都会被替换
  - 使用方式ansible-doc -s replace查看

#### \#  命令类操作
1,command: 在远端主机上执行命令
  - 使用command模块在远程主机中执行命令时，不会经过远程主机的shell处理，在使用command模块时，如果需要执行的命令中含有重定向、管道符等操作时，这些符号也会失效，比如”<“, “>”, “|”, “;” 和 “&” 这些符号，如果你需要这些功能，需要使用shell模块
  - 具体使用方式,参考ansible-doc -s command

2,shell: 在远端主机上执行命令,与command不同的是,shell模块在远端执行命令时,会经过远端主机的/bin/sh程序处理
  - 具体使用方式,参考ansible-doc -s shell

3,scripts: 在远端主机上执行ansible主机上的脚本,脚本在ansible主机上,不需要拷贝到远端主机
  - 具体使用方式,参考ansible-doc -s scripts

#### \#  系统类操作
1,cron: 管理远端主机上的定时任务,功能同crontab
  - 具体使用方式,参考ansible-doc -s cron

2,service: 管理远端主机上的服务
  - 具体使用方式,参考ansible-doc -s service

3,user: 管理远端主机上的用户,类似命令usermod
  - 还可以管理用户的ssh密钥
  - 具体使用方式,参考ansible-doc -s user

4,groupo: 管理远端主机上的组,类似命令groupmod
  - 具体使用方式,参考ansible-doc -s group

#### \#  包管理类操作
1,yum_repository: 管理远端主机上的yum仓库
  - 具体使用方式,参考ansible-doc -s yum_repository

1,yum: 通过远端主机上的yum管理软件包
  - 具体使用方式,参考ansible-doc -s yum

### 认识ansible-playbook ###
1,ansible-playbook的使用,可以理解为ansible -m <module_name> -a 'xxxx'的转换,将命令行使用模块操作的内容写成脚本内容,按照脚本内容完成相关操作
2,上述脚本在ansible-playbook中称作为'playbook',即剧本
  - 每个playbook(剧本)又多个play(桥段)组成,每个剧本是由多个桥段组成的,每个桥段包含有人物,场景,故事
  - 每个play在执行时,都会执行一个默认任务('Gathering Facts),任务会收集当前当前play对应的目标主机的相关信息(IP,hostname,硬件版本,系统版本等),收集完成后才会完成我们定义的相关任务

3,ansible有个重要特性:幂等性
  - 在ansible调用模块或者ansible-playbook执行相应play时,输出内容会有颜色区分,黄色表示有修改,绿色表示么有修改;区别是远端的内容是否满足我们的预期
  - ansible是”以结果为导向的”，我们指定了一个”目标状态”，ansible会自动判断，”当前状态”是否与”目标状态”一致，如果一致，则不进行任何操作，如果不一致，那么就将”当前状态”变成”目标状态”，这就是”幂等性”，”幂等性”可以保证我们重复的执行同一项操作时，得到的结果是一样的

### 使用handlers ###
1,handlers的使用场景:
  - 有个任务需要修改nginx的配置文件,将listen端口由8080改为8088,使用handler可以在nginx配置文件有修改的环境上重启nginx,没有修改的不会出发重启nginx
  - handlers可以理解为另一种tasks,handlers是另一种'任务列表',handlers中的任务会被tasks中的任务调用;
  - handlers被调用并不一定会执行,只有当tasks中的任务真正被执行后(真正的进行了实际操作,造成了实际变化),handlers中的任务才会真正执行
  - 如果tasks中的任务并没有作出任何实际的操作,那么handlers中的任务即使被调用,也不会执行
  - handlers和tasks是平级的,所以缩进相同

2,handlers的调用:
  - handlers需要被关键字notify调用
    如下示例:
```yaml
---
- hosts: all
  remote_user: root
  tasks:
  - name: change nginx configuration
    lineinfile:
      path=/etc/nginx/conf.d/test.conf
      regexp="listen(.*) 8080(.*)"
      line="listen\1 8088\2"
      backrefs=yes
      backup=yes
    notify:
      restart nginx

  handlers:
  - name: restart nginx
    service:
      name=nginx
      state=restarted 
```

3,handlers中可以有多个任务,被tasks中不同的任务调用
  - handler执行的顺序与handler在playbook中定义的顺序相同,与handler被notify的顺序无关(下述内容ht1先触发,ht2后触发),如下示例:
```yaml
---
- hosts: all
  remote_user: root
  tasks:
  - name: make testfile1
    file:
      path=/testdir/testfile1
      state=directory
    notify: ht2
  - name: make testfile2
    file:
      path=/testdir/testfile2
      state=directory
    notify: ht1

  handlers:
  - name: ht1
    file:
      path=/testdir/ht1
      state=touch
  
  - name: ht2
    file:
      path=/testdir/ht2
      state=touch
```
  - 默认情况下,所有tasks执行完成后,才会执行各个handlers
  - 当存在多个同名的handler时,只会执行一个handler

4,若要在执行完某个task后立即执行其对应的handler,需要使用meta模块
  - 如下示例:
```yaml
---
- hosts: all
  remote_user: root
  tasks:
  - name: task1
    file: 
      path=/testdir/testfile
      state=touch
    notify: handler1
  - name: task2
    file:
      path=/testdir/testfile2
      state=touch
    notify: handler2

  - meta: flush handlers

  - name: task3
    file:
      path=/testdir/testfile3
      state=touch
    notify: handler3

  handlers:
  - name: handler1
    file:
      path=/testdir/hd1
      state=touch
  - name: handler2
    file:
      path=/testdir/hd2
      state=touch
  - name: handler3
    file:
      path=/testdir/hd3
      state=touch
```
  - meta可以理解为tasks下一个特殊任务,使用的是meta模块
  - meta: flush_handlers指的是立即执行之前的task对应的handlers,上述例子中flush_handlers之前有两个task,在执行了这两个task之后立即执行他们对应的handler
  - 使用meta配合task和handler,可以让任务调用更加灵活

5,如果需要一次性notify多个handler,需要使用到listen
  - 可以把'listen'理解为'组名',可以把多个handler分成'组',当我们需要一次性notify多个handler时,只要将多个handler分成一组,使用相同的组名即可
  - 如下示例:
```yaml
---
- hosts: all
  remote_user: root
  tasks:
  - name: task1
    file: 
      path=/testdir/testfile
      state=touch
    notify: handler group1

  handlers:
  - name: handler1
    listen: 'handler group1'
    shell: 'echo handler1'
  - name: handler2
    listen: 'handler group1'
    shell: 'echo handler2'
```

### 使用tags ###
1,写了一个很长的playbook,在调试时只想跑其中很少的一部分,tag在这种场景下可以使用,指定执行哪些任务,不执行哪些任务
  - 如下示例:
```yaml
---
- hosts: all
  remote_user: root
  tasks:
  - name: test 1
    file:
      path=/testdir/testfile
      state=touch
    tags: t1
  - name: test 2
    file:
      path=/testdir/testfile2
      state=touch
    tags:t2
  - name: test 3
    file:
      path=/testdir/testfile3
      state=touch
    tags: t3
$ ansible-playbook --tags=t2 test.yaml
$ ansible-playbook --skip-tags=t2 test.yaml
```
  - --tags=t2,只执行tag为t2部分的task,--skip-tags=t2,跳过tag=t2的tag
  - tag相关使用命令:
```bash
# 指定多个tag时,使用命令为
$ ansible-playbook -t tag1,tag2,tag3 test.yaml
# 执行play前,查看当前有哪些tag
$ ansible-playbook --list-tags test.yaml
```
  - ansible预置了几个特殊tag:
```
# always: 如果任务的tags包含always,则该task一定会被执行,除非指定--skip-tags always(该情况也不合理,可能其他任务也包含always,这样其他task也不会执行)
# never: 永远不执行,与always刚好相反

## 下列只在调用标签时生效
# tagged: ansible-playbook --tags tagged test.yaml --> 只执行有标签的task,没有标签的task不会被执行
# untagged: ansible-playbook --tags untagged test.yaml --> 只执行没有tag的task,有tag的不会被执行
# all: 默认使用,所有都会被执行
```
  - 可以为task指定多个tag,如下:
```yaml
# method one
tags:
  - testing
  - t1
# method two
tags: testing, t1
# method three
tags: ['testing', 't1']
```
2,play也可以指定tags,当一个play之指定了tags,这个play下的所有task都包含该tag,若该play下的task还有自己的tags,则该task的实际tags为'play tags' + 'task tags'
```yaml
---
- hosts: test70
  remote_user: root
  tags: httpd
  tasks:
  - name: install httpd package
    tags: ['package']
    yum:
      name=httpd
      state=latest
 
  - name: start up httpd service
    tags:
      - service
    service:
      name: httpd
      state: started
```

### 使用变量(一) ###
1,怎么定义变量
  - 变量由数字,字母,下划线组成,要以字母开头
  - ansible的关键字不能作为变量名

2,变量的定义及引用
  - 变量定义可以使用vars,和tasks同级,如下(普通定义方式):
```yaml
---
- hosts: all
  remote_user: root
  vars:
    testvar1: testfile
    testvar2: testfile2
# 或者使用yaml写法
  vars:
    - testvar1: testfile
    - testvar2: testfile2
  tasks:
  - name: task1
    file:
      path: /testdir/{{testvar1}}
      state: touch
```
  - 使用属性的方式定义
```yaml
- hosts: all
  remote_user: root
  vars:
    nginx:
      conf80: /etc/nginx/conf.d/80.conf
      conf8080: /etc/nginx/conf.d/8080.conf
  tasks:
  - name: task1
    file:
      path: "{{nginx.conf80}}"
      # 或者path: "{{nginx['conf80']}}"
      state: touch
  - name: task2
    file:
      path: "{{nginx.conf8080}}"
      # 或者path: "{{nginx['conf8080']}}"
      state: touch
# 使用=给模块参数赋值时,可以不考虑变量是否加引号
#  - name: task2
#    file:
#      path={{nginx.conf8080}}
#      # 或者path={{nginx['conf8080']}}
#      state=touch
```
> 注意: 上面列举的两个例子有些差别,第一个例子中变量没有加引号,第二个有加引号,因为第一个变量不是'开头'位置,第二个是'开头'位置
> 但实际上也有例外,给模块参数赋值时,可以选择':',也可以选择'=',当使用'='时,可以不用考虑引号的问题

3,在文件中定义变量给playbook使用
  - 将变量在单独的文件中定义的好处是,可以做到变量文件分离,不给看到变量的值
  - 在文件中定义变量时,不用使用vars关键字,直接定义变量即可,如下集中语法:
```yaml
# method one
testvar1: testfile
testvar2: testfile2
# method two
- testvar1: testfile
- testvar2: testfile2
# method three
nginx:
  conf80: /etc/nginx/conf.d/80.conf
  conf8080: /etc/nginx/conf.d/8080.conf
```
  - 在文件中定义完变量,在playbook中使用变量,需要使用vars_files,被导入的文件以'-'开头(可以引入多个变量文件),以yaml中块序列的方式导入
```yaml
---
- hosts: all
  remote_user: root
  vars_files:
  - /testdir/ansible/nginx_vars.yml
  tasks:
  - name: task1
    file:
      path: "{{nginx.conf80}}"
      state: touch
  - name: task2
    file:
      path: "{{nginx['conf8080']}}"
      state: touch
```

### 使用变量(二) ###
1,在playbook执行前,有一个Gathering Facts的动作,调用的是setup模块; 这些信息会保存在对应的变量中，我们在playbook中可以使用这些变量,我们可以称这些信息为facts信息
  - setupa模块的返回值是json格式的,方便返回时内容展示
  - setup模块可以获取远端主机很详尽的信息,若需要过滤相关信息,可以使用setup的filter参数(支持通配符)
```bash
# 显示内存相关信息
ansible all -m setup -a 'filter=ansible_memory_mb'
# 不确定信息相关信息,使用通配符来获取
ansible all -m setup -a 'filter=*mb*'
```
  - setup模块获取的信息都保存在相应的变量中,我们可以通过引用变量获取到这些值

2,setup模块支持获取自定义内容
  - 要求:自定义内容存在于目标主机的/etc/ansible/facts.d/下以.fact结尾的文件; 这些文件需要以json或者ini格式保存变量信息
```bash
$ cat /etc/ansible/facts.d/test.fact
[testmsg]
msg1='test message 1'
msg2='test message 2'
```
  - 这些自定义的变量称为'local facts',可以在setup获取时通过filter=ansible_local获取

3,另一个模块debug,可用于playbook的调试使用,把调试信息打印到控制台上,方便我们查看和定位问题
  - debug模块常用的两个参数分别是var和msg,var用于测试变量,msg用于打印输出是否符合预期
```
# 连接到主机,但是并没有做任何事情,debug引用的是testvar,测试testvar是否能用
---yaml
- hosts: all
  remote_user: root
  vars:
    testvar: value of testvar
  tasks:
  - name: test debug
    debug:
      var: testvar
```

### 使用变量(三) ###
1,变量注册
- ansible模块在执行之后都会有一些返回值,默认情况下,这些返回值不会显示而已;我们可以把这些返回值写入到某个变量中,我们可以通过引用相应的变量获取到这些返回值,将模块的返回值写入到变量中,这种方式称为'注册变量'; 如下为变量注册的的一个范例
```yaml
---
- hosts: all
  remote_user: root
  tasks:
  - name: test register
    shell: "echo test > /tmp/testfile"
    register: testvar
  - name: shell module return values
    debug:
      var: testvar
# 执行完成后,在控制台中看到名为”[shell module return values]”的任务中已经显示了第一个任务的返回值的信息，返回信息:
TASK [shell module return values] ************************************
ok: [master] => {    
    "testvar": {          
        "changed": true,
        "cmd": "echo test > /tmp/testfile",
        "delta": "0:00:00.010963",
        "end": "2021-03-30 09:26:47.432571",                         
        "failed": false,                                            
        "rc": 0,                                                   
        "start": "2021-03-30 09:26:47.421608",                    
        "stderr": "",
        "stderr_lines": [],                                      
        "stdout": "",
        "stdout_lines": []
    }
}
# register的变量testvar其实是一个json格式的返回值,可以通过引用testvar的不同属性获取相应的值
```

2,变量的传入方式,ansible支持多种变量传入的方式,包括:交互式传入,命令行传入,文件传入
  - 交互式传入,需要使用到var_prompt关键字,属于vars的一种,当出现该关键字时,会让你在命令行下输入
    - 普通使用,以下输入内容不会在控制台显示(默认情况下private: yes)
```yaml
---
- hosts: all
  remote_user: root
  vars_prompt:
    - name: 'your_name'
      prompt: "What is your name?"
      # private: no
    - name: 'your_age'
      prompt: "What is your age?"
      # private: no
  tasks:
  - name: output vars
    debug:
      msg: your name is {{your_name}}, your age is {{your_age}}
```
    - 普通使用,可以做到提供选项供选择
    - 特殊使用,例如增加用户,设定密码
```yaml
# 需要使用到vars_prompt下encrypt,指定加密方式,而且该方式需要passlib库,么有的话需要安装
---
- hosts: test70
  remote_user: root
  vars_prompt:
    - name: "hash_string"
      prompt: "Enter something"
      private: no
      encrypt: "sha512_crypt"
  tasks:
   - name: Output the string after hash
     debug:
      msg: "{{hash_string}}"
```
  - 命令行传入,直接在ansible-playbook执行时增加-e 'key=value'即可,可有多个,变量赋值有多种方式,还可以是json格式
  - 文件传入,类似命令行-e "@变量文件绝对路径"

### 使用变量(四),register,set_fact ###
1,配置主机清单时,可以配置主机或主机组变量,但只对配置的主机或主机组生效
```yaml
## 主机配置,配置/etc/ansible/hosts如下
# method one
node2 ansible_host=192.168.110.22 testhostvar=node2_host_var
# method two
all:
  hosts:
    node2:
      ansible_host: 192.168.110.22
      ansible_port: 22
      testhostvar: node2_host_var
      testhostvar2: node2_host_var2
      testhostvar3: 
        thv31: 3.1
        thv32: 3.2
$ ansible node2 -m shell -a 'echo {{testhostvar}}
$ ansible node2 -m shell -a 'echo {{testhostvar3.thv31}}' or '{{testhostvar3['thv32']}}'

## 主机组配置,配置/etc/ansible/hosts如下
# method one
[testB]
node2 ansible_host=192.168.110.22
node3 anisble_host=192.168.110.33
 
[testB:vars]
test_group_var1='group var test'
test_group_var2='group var test2'
# method two
all:
 children:
   testB:
     hosts:
       node2:
         ansible_host: 192.168.110.22
         ansible_port: 22
       node3:
         ansible_host: 192.168.110.33
         ansible_port: 22
     vars:
       test_group_var1: 'group var test1'
       test_group_var2: 'group var test2'
$ ansible testB -m shell -a 'echo {{test_group_var1}}'
```

2,通过set_fact定义变量,set_fact是一个模块,可以通过set_fact模块在task中定义变量
  - 普通使用,直接与task同级定义一个变量
```yaml
---
- hosts: node2
  remote_user: root
  tasks:
  - set_fact:
      testvar: "testtest"
  - debug:
      msg: "{{testvar}}"
```
  - 普通使用,将一个变量的值赋给另一个变量
```yaml
---
- hosts: node2
  remote_user: root
  vars:
    testvar1: test1_string
  tasks:
  - shell: "echo test2_string"
    register: shellreturn
  - set_fact:
      testsf1: "{{testvar1}}"
      testsf2: "{{shellreturn.stdout}}"
  - debug:
      msg: "{{testsf1}} {{testsf2}}"
```
  - 通过set_fact模块创建的变量还有一个特殊性，通过set_fact创建的变量就像主机上的facts信息一样，可以在之后的play中被引用
```yaml
---
- hosts: node2
  remote_user: root
  vars:
    testvar1: tv1
  tasks:
  - set_fact:
      testvar2: tv2
  - debug:
      msg: "{{testvar1}} ----- {{testvar2}}"
 
- hosts: node2
  remote_user: root
  tasks:
  - name: other play get testvar2
    debug:
      msg: "{{testvar2}}"
  - name: other play get testvar1
    debug:
      msg: "{{testvar1}}"
# 这两个变量在第一个play中都可以正常的输出.但是在第二个play中，testvar2可以被正常输出了，testvar1却不能被正常输出，会出现未定义testvar1的错误
```
  - 如果想要在tasks中给变量自定义信息，并且在之后的play操作同一个主机时能够使用到之前在tasks中定义的变量时，则可以使用set_facts定义对应的变量

### 使用变量(五),内置变量,host_vars ###
1,ansible有一些内置变量可供使用,这些变量被ansible保留,我们定义变量时不能使用
  - ansible_version
```bash
$ ansible node2 -m debug -a 'msg={{ansible_version}}'
```

2,hostvars可以在我们操作当前主机时获取到其他主机中的信息
  - 如下示例
```yaml
---
# 第一个什么也没做,只是获取node3的facts内容,这一步是需要的,只有收集过的facts才能被后面的play使用
# 如果没有收到对应主机的facts信息,即使使用hostvars内置变量,也无法获取到对应主机的facts内容
- name: "play 1: gather facts of node3"
  hosts: node3
  remote_user: root
  # 下面的gather_facts默认是yes
  # gater_facts: yes
- name: "play 2: gathter facts of node3 when operating on node2"
  hosts: node2
  remote_user: root
  tasks:
  - debug:
    msg: "{{hostvars['node3'].ansible_enp0s3.ipv4}}"
    # 下面两种也可以
    #msg: "{{hostvars.node3.ansible_enp0s3.ipv4}}"
    #msg: "{{hostvars['node3']['ansible_enp0s3']['ipv4']}}"
```
  - hostvars除了获取到其他主机的facts内容,还可以获取到其他类型的一些变量信息,如其他主机的注册变量,主机变量,组变量等;如下示例
```yaml
---
# 1,通过hostvars内置变量可以直接获取到其他主机中的注册变量
# 2,注册变量并不用像facts信息那样需要事先收集，即可直接通过hostvars跨主机被引用到
# 3,如果你在清单中为node3主机配置了主机变量，或者为node3主机所在的组配置了组变量，也是可以通过hostvars直接跨主机引用
- hosts: node3
  remote_user: root
  gather_facts: no
  tasks:
  - shell: "echo register_var_in_play1"
    register: shellreturn
 
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{hostvars.node3.shellreturn.stdout}}"
```
  - 通过vars关键字定义的变量使用上例中的hostvars方法是无法被跨主机引用
```yaml
# 下列内容会报错,变量需要注册才行,直接使用vars定义的变量无法传递
---
- hosts: node3
  remote_user: root
  gather_facts: no
  vars:
    testvar: testvar_in_3
  tasks:
  - debug:
      msg: "{{testvar}}"
 
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{hostvars.node3.testvar}}"

# 通过set_fact将vars定义的内容注册,则可以
---
- hosts: node3
  remote_user: root
  gather_facts: no
  tasks:
  - set_fact:
      testvar: testvar_in_3
  - debug:
      msg: "{{testvar}}"
 
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{hostvars.node3.testvar}}"
```
  - 通过”set_fact”结合”hostvars”的方式，可以实现跨play获取其他主机中的变量信息

3,内置变量inventory_hostname,获取到被操作的当前主机的主机名称
  - 主机名称并不是linux系统的主机名，而是对应主机在清单中配置的名称,清单中配置的名称即是

4,内置变量inventory_hostname_short,获取到被操作的当前主机的主机名称,简版
  - 无论是IP还是别名，如果清单的主机名称中包含”.”，inventory_hostname_short都会取得主机名中第一个”.”之前的字符作为主机的简短名称

5,内置变量play_hosts,获取到当前play所操作的所有主机的主机名列表
  - 如下示例
```yaml
---
- hosts: node2,node3
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{play_hosts}}"
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => {
    "msg": [
        "node2", 
        "node3"
    ]
}
ok: [node3] => {
    "msg": [
        "node2", 
        "node3"
    ]
}
```

6,内置变量groups,可以获取到清单中”所有分组”的”分组信息”
  - 同inventory_hostname,但能获取到分组的信息
```bash
$ ansible all -m debug -a 'msg={{groups}}'
$ ansible all -m debug -a 'msg={{groups.k8s}}'
```

7,内置变量group_names,获取到当前主机所在分组的组名
  - 如下示例:
```bash
# ansible node2 -m debug -a "msg={{group_names}}"
node2 | SUCCESS => {
    "changed": false, 
    "msg": [
        "k8s"
    ]
}
```

8,内置变量inventory_dir,获取到ansible主机中清单文件的存放路径,默认是/etc/ansible,但也可以自定义
  - 如下示例:
```bash
# ansible node2 -m debug -a "msg={{inventory_dir}}"
node2 | SUCCESS => {
    "changed": false, 
    "msg": "/etc/ansible"
}
```

9,除了直接在hosts文件中定义主机变量和组变量，还有另外一种方法也可以定义主机变量和组变量，我们可以在清单文件的同级目录中创建两个目录，这两个目录的名字分别为”group_vars”和”host_vars”，我们可以将组变量文件放在”group_vars”目录中，将主机变量文件放在”host_vars”目录中，这样ansible就能获取到对应组变量和主机变量

### 使用循环(一),with_items的使用 ###
1,使用with_items处理循环的内容
  - 普通示例:
```yaml
# "with_items"关键字会把返回的列表信息自动处理，将每一条信息单独放在一个名为"item"的变量中，只要获取到名为"item"变量的变量值，即可循环的获取到列表中的每一条信息
# 下列debug被循环3次,每次单独输出相应循环的debug输出内容
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - name: test with_items
    debug:
      msg: "{{item}}"
    with_items: "{{groups.k8s}}"
```

2,with_items可以自定义
  - 列表
```yaml
# method one
with_items:
- 1
- 2
- 3
# method two
with_items: [1, 2, 3]
```
  - 相对复杂的列表
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{item.test1}}"
    with_items:
    - { test1: a, test2: b}
    - { test1: c, test2: d}

```

3,result的使用
  - 当使用了循环以后，每次shell模块执行的返回值放入名为”results”的序列中，”results”也是一个返回值，当模块中使用循环时，模块每次执行的返回值都会追加存放到”results”这个返回值中,如下示例:
```yaml
# 两次debug循环,输出的内容都保存在results序列中,属于results
---
- hosts: node2
  gather_facts: no
  tasks:
  - shell: "{{item}}"
    with_items:
    - "ls /opt"
    - "ls /home"
    register: returnvalue
  - debug:
      var: returnvalue
# 可以使用如下方式,避免所有结果在最后的results中去获取
---
- hosts: node2
  gather_facts: no
  tasks:
  - shell: "{{item}}"
    with_items:
    - "ls /opt"
    - "ls /home"
    register: returnvalue
  - debug:
      msg: "{{item.stdout}}'
    with_items: "{{returnvalue.results}}"
# 先使用循环重复的调用了shell模块，然后将shell模块每次执行后的返回值注册到了变量”returnvalue”中，之后，在使用debug模块时，通过返回值”results”获取到了之前每次执行shell模块的返回值（shell每次执行后的返回值已经被放入到item变量中），最后又通过返回值”stdout”获取到了每次shell模块执行后的标准输出
```
### 使用循环(二),对列表循环的操作 ###
1,对序列循环有几个关键字
  - with_items: 当循环的序列元素也是列表时,展开与预期的有差异,会将所有的列表展开
  - with_list: 其他动作与with_items相同,只有在嵌套列表循环时有差异,子列表将会作为元素使用
  - with_flattened: 与with_items相同
  - with_together: 列对齐,”对齐合并”功能
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{ item }}"
    with_together:
    - [ 1, 2, 3 ]
    - [ a, b, c ]
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => (item=[1, u'a']) => {
    "changed": false,
    "item": [
        1,
        "a"
    ],
    "msg": [
        1,
        "a"
    ]
}
ok: [node2] => (item=[2, u'b']) => {
    "changed": false,
    "item": [
        2,
        "b"
    ],
    "msg": [
        2,
        "b"
    ]
}
ok: [node2] => (item=[3, u'c']) => {
    "changed": false,
    "item": [
        3,
        "c"
    ],
    "msg": [
        3,
        "c"
    ]
}
```
### 使用循环(三),嵌套循环 ###
1,with_cartesian和with_nested
  - 当我们需要两个列表嵌套循环时,可以使用with_cartesian或者with_nested,将每个小列表中的元素按照”笛卡尔的方式”组合后，循环的处理每个组合,如下示例
```yaml
# 下面的例子会在node2下创建6个目录
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - file:
      state: directory
      path: "/testdir/testdir/{{ item.0 }}/{{ item.1 }}"
    with_cartesian:
    - [ a, b, c ]
    - [ test1, test2 ]
```
### 使用循环(四),序列索引循环 ###
1,使用到with_indexed_items,在处理列表中的每一项时，按照顺序为每一项添加了编号,如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{ item }}"
    with_indexed_items:
    - test1
    - test2
    - test3
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => (item=(0, u'test1')) => {
    "changed": false,
    "item": [
        0,
        "test1"
    ],
    "msg": [
        0,
        "test1"
    ]
}
ok: [node2] => (item=(1, u'test2')) => {
    "changed": false,
    "item": [
        1,
        "test2"
    ],
    "msg": [
        1,
        "test2"
    ]
}
ok: [node2] => (item=(2, u'test3')) => {
    "changed": false,
    "item": [
        2,
        "test3"
    ],
    "msg": [
        2,
        "test3"
    ]
}
```
  - ”with_indexed_items”会将嵌套的两层列表”拉平”，”拉平”后按照顺序为每一项编号
  - 当多加了一层嵌套以后，”with_indexed_items”并不能像”with_flattened”一样将嵌套的列表”完全拉平”，第二层列表中的项如果仍然是一个列表，”with_indexed_items”则不会拉平这个列表，而是将其当做一个整体进行编号

### 使用循环(五),with_sequence,with_random_choice ###
1,with_sequence
- 使用with_sequence生成序列,with_sequence可以按照顺序生成数字序列,如下示例:
```yaml
# debug模块被循环调用了5次，msg的值从1一直输出到了5，值的大小每次增加1
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{ item }}"
    with_sequence: start=1 end=5 stride=1
    # 下列写法结果一致
    # with_sequence: count=5
```
  - 使用with_sequence还可以格式化输出
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{item}}"
    with_sequence: start=2 end=6 stride=2 format="number is %0.2f"
```

2,with_random_choice,使用with_random_choice可以从列表的多个值中随机返回一个值
  - 如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{item}}"
    with_random_choice:
    - 1
    - 2
    - 3
    - 4
    - 5
```
### 使用循环(六),with_dict,with_subelements,with_file ###
1,with_dict,循环遍历字典元素,如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      alice:
        name: Alice Appleworth
        gender: female
        telephone: 123-456-7890
      bob:
        name: Bob Bananarama
        gender: male
        telephone: 987-654-3210
  tasks:
  - debug:
      msg: "{{item}}"
    with_dict: "{{users}}"
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => (item={'value': {u'gender': u'male', u'name': u'Bob Bananarama', u'telephone': u'987-654-3210'}, 'key': u'bob'}) => {
    "changed": false,
    "item": {
        "key": "bob",
        "value": {
            "gender": "male",
            "name": "Bob Bananarama",
            "telephone": "987-654-3210"
        }
    },
    "msg": {
        "key": "bob",
        "value": {
            "gender": "male",
            "name": "Bob Bananarama",
            "telephone": "987-654-3210"
        }
    }
}
ok: [node2] => (item={'value': {u'gender': u'female', u'name': u'Alice Appleworth', u'telephone': u'123-456-7890'}, 'key': u'alice'}) => {
    "changed": false,
    "item": {
        "key": "alice",
        "value": {
            "gender": "female",
            "name": "Alice Appleworth",
            "telephone": "123-456-7890"
        }
    },
    "msg": {
        "key": "alice",
        "value": {
            "gender": "female",
            "name": "Alice Appleworth",
            "telephone": "123-456-7890"
        }
    }
}
```

2,with_subelements:
  - with_subelements会将hobby子元素列表中的每一项作为一个整体，将其他子元素作为一个整体，然后组合在一起
```yaml
# 如下示例:
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
    - name: bob
      gender: male
      hobby:
        - Skateboard
        - VideoGame
    - name: alice
      gender: female
      hobby:
        - Music
  tasks:
  - debug:
      msg: "{{ item }}"
    with_subelements:
    - "{{users}}"
    - hobby
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => (item=({u'gender': u'male', u'name': u'bob'}, u'Skateboard')) => {
    "changed": false,
    "item": [
        {
            "gender": "male",
            "name": "bob"
        },
        "Skateboard"
    ],
    "msg": [
        {
            "gender": "male",
            "name": "bob"
        },
        "Skateboard"
    ]
}
ok: [node2] => (item=({u'gender': u'male', u'name': u'bob'}, u'VideoGame')) => {
    "changed": false,
    "item": [
        {
            "gender": "male",
            "name": "bob"
        },
        "VideoGame"
    ],
    "msg": [
        {
            "gender": "male",
            "name": "bob"
        },
        "VideoGame"
    ]
}
ok: [node2] => (item=({u'gender': u'female', u'name': u'alice'}, u'Music')) => {
    "changed": false,
    "item": [
        {
            "gender": "female",
            "name": "alice"
        },
        "Music"
    ],
    "msg": [
        {
            "gender": "female",
            "name": "alice"
        },
        "Music"
    ]
}
```
  - 由于item由两个整体组成，所以，我们通过item.0获取到第一个小整体，即gender和name属性，然后通过item.1获取到第二个小整体，即hobby列表中的每一项
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
    - name: bob
      gender: male
      hobby:
        - Skateboard
        - VideoGame
    - name: alice
      gender: female
      hobby:
        - Music
  tasks:
  - debug:
      msg: "{{ item.0.name }} 's hobby is {{ item.1 }}"
    with_subelements:
    - "{{users}}"
    - hobby
# msg内容如下:
"msg": "bob 's hobby is Skateboard"
"msg": "bob 's hobby is VideoGame"
"msg": "alice 's hobby is Music"
```
### 使用循环(七) with_file, with_fileglob ###
1, ansible主机中有几个文件,若需要获取到这些文件的内容，可以使用with_file关键字，循环的获取到这些文件的内容
  - 如下示例:
```yaml
# 无论目标主机是谁，都可以通过with_file关键字获取到ansible主机中的文件内容
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{ item }}"
    with_file:
    - /testdir/testdir/a.log
    - /opt/testfile
```

2,可以通过with_fileglob关键字，在指定的目录中匹配符合模式的文件名，with_file与with_fileglob相同的地方，它们都是针对ansible主机的文件进行操作，而不是目标主机
  - 如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{ item }}"
    with_fileglob:
    - /testdir/(此处为星号,删除防止下面格式错乱)
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => (item=/testdir/testfile) => {
    "changed": false,
    "item": "/testdir/testfile",
    "msg": "/testdir/testfile"
}
ok: [node2] => (item=/testdir/test.sh) => {
    "changed": false,
    "item": "/testdir/test.sh",
    "msg": "/testdir/test.sh"
}
# 需要注意的是，with_fileglob只会匹配指定目录中的文件，而不会匹配指定目录中的目录
```

### 条件判断(六),with_dict,with_subelements,with_file ###
1,绝大多数语言中，都使用if作为条件判断的关键字，而在ansible中，条件判断的关键字是when
  - 如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  tasks:
  - name: test when
    debug:
      msg: "System is centos"
    # 如果需要获取到facts中的key的值，都是通过引用变量的方式获取的，即"{{ key }}"
    # 在when关键字中引用变量时，变量名不需要加"{{  }}"
    when: ansible_distribution == "CentOS"
```

2,使用when关键字为任务指定条件，条件成立，则执行任务，条件不成立，则不执行任务
  - 如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{item}}"
    with_items:
    - 1
    - 2
    - 3
    when: item > 1
```

3,在ansible中，使用如下比较运算符.
  - 如下所示:
```
==  :比较两个对象是否相等，相等为真
!=  :比较两个对象是否不等，不等为真
>   :比较两个值的大小，如果左边的值大于右边的值，则为真
<   :比较两个值的大小，如果左边的值小于右边的值，则为真
>=  :比较两个值的大小，如果左边的值大于右边的值或左右相等，则为真
<=  :比较两个值的大小，如果左边的值小于右边的值或左右相等，则为真
上述总结的这些运算符其实都是jinja2的运算符，ansible使用jinja2模板引擎，在ansible中也可以直接使用jinja2的这些运算符

上述为比较运算符，再来说说逻辑运算符，可用的逻辑运算符如下:
and  :逻辑与，当左边与右边同时为真，则返回真
or   :逻辑或，当左边与右边有任意一个为真，则返回真
not  :取反，对一个操作体取反
( )  :组合，将一组操作体包装在一起，形成一个较大的操作体
```
  - 如下示例:
```yaml
# 示例1:
---
- hosts: node2
  remote_user: root
  tasks:
  - debug:
      msg: "System release is centos7"
    when:
    # 使用列表,列表中的每一项都是一个条件，列表中的所有条件同时成立时，对应的任务才会执行
    - ansible_distribution == "CentOS"
    - ansible_distribution_major_version == "7"
# 示例2:
---
- hosts: node2
  remote_user: root
  tasks:
  - debug:
      msg: "System release is centos6 or centos7"
    # 比较运算符和逻辑运算符结合使用作为条件判断
    when: ansible_distribution == "CentOS" and
          (ansible_distribution_major_version == "6" or ansible_distribution_major_version == "7")
# 示例3:
---
- hosts: node2
  remote_user: root
  tasks:
  - debug:
      msg: "System release is not centos"
    # 逻辑取反
    when: not ansible_distribution == "CentOS"
# 示例4:
---
- hosts: node2
  remote_user: root
  tasks:
  - name: task1
    shell: "ls /testabc"
    register: returnmsg
  - name: task2
    debug:
      msg: "Command execution successful"
    # 通过shell指令的返回值判断是否执行
    when: returnmsg.rc == 0
  - name: task3
    debug:
      msg: "Command execution failed"
    when: returnmsg.rc != 0
```

3,结合ignore_errors和when来限定playbook的执行
  - 如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  tasks:
  - name: task1
    shell: "ls /testabc"
    register: returnmsg
    # 此处增加了ignore_errors,在不存在/testabc目录的节点就不会报错,playbook不会退出,task2,task3通过rc值来确定是否要执行
    ignore_errors: true
  - name: task2
    debug:
      msg: "Command execution successful"
    when: returnmsg.rc == 0
  - name: task3
    debug:
      msg: "Command execution failed"
    when: returnmsg.rc != 0
```

### 条件判断与tests ###
1,在ansible中也有类似bash中test的用法,不过是借助jinja2的tests，借助tests，可以进行一些判断操作，tests会将判断后的布尔值返回，如果条件成立，返回true，否则返回false，通常在条件判断时使用到tests
  - 如下示例:
```yaml
---
- hosts: me
  gather_facts: no
  remote_user: root
  vars:
    testpath: /testdir
  tasks:
  - name: test tests
    debug:
      msg: "test dir exist"
    # "is exists"中的"exists"就是tests的一种，它与"test -e"命令的作用是相同的，通过"exists"可以判断ansible主机中的对应路径是否存在
    # "is not exists"表示对应路径不存在时返回真
    # 上述内容都是在ansible主机中判断的,和远端目标主机无关
    when: testpath is exists
```

2,判断变量的一些tests
  - defined: 判断变量是否已经定义，已经定义则返回真
  - undefined: 判断变量是否已经定义，未定义则返回真
  - none: 判断变量值是否为空，如果变量已经定义，但是变量值为空，则返回真
  - 上述内容,如下示例:
```yaml
---
- hosts: me
  remote_user: root
  gather_facts: no
  vars:
    testvar: "test"
    testvar1:
  tasks:
  - debug:
      msg: "Variable is defined"
    when: testvar is defined
  - debug:
      msg: "Variable is undefined"
    when: testvar2 is undefined
  - debug:
      msg: "The variable is defined, but there is no value"
    when: testvar1 is none
```

3,判断执行结果的一些tests
  - success或者succeeded: 通过任务的返回信息判断任务的返回状态,任务执行成功则返回真
  - failure或者failed: 通过任务的返回信息判断任务的返回状态,任务执行失败则返回真
  - change后者changed: 通过任务的返回信息判断任务的返回状态,任务执行状态为changed则返回真
  - skip或者skipped: 通过任务的返回信息判断任务的返回状态,当任务没有满足执行条件,而被跳过执行时,则返回真
  - 如下示例:
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars:
    doshell: "yes"
  tasks:
  - name: execute
    shell: cat /testdir/test
    when: doshell == "yes"
    register: retmsg
    ignore_errors: true
  - debug:
      msg: "task success"
    when: retmsg is success
  - debug:
      msg: "task failed"
    when: retmsg is failed
  - debug:
      msg: "task skipped"
    when: retmsg is skip
  - debug:
      msg: "task changed"
    when: retmsg is changed
```

4,判断路径的一些tests
  - 注:如下tests的判断均针对于ansible主机中的路径,与目标主机无关
  - file : 判断路径是否是一个文件,如果路径是一个文件则返回真
  - directory :判断路径是否是一个目录,如果路径是一个目录则返回真
  - link :判断路径是否是一个软链接,如果路径是一个软链接则返回真
  - mount:判断路径是否是一个挂载点,如果路径是一个挂载点则返回真
  - exists:判断路径是否存在,如果路径存在则返回真
  - 如下示例:
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars:
    testpath1: "/testdir/test"
    testpath2: "/testdir/"
    testpath3: "/testdir/testsoftlink"
    testpath4: "/testdir/testhardlink"
    testpath5: "/boot"
  tasks:
  - debug:
      msg: "file"
    when: testpath1 is file
  - debug:
      msg: "directory"
    when: testpath2 is directory
  - debug:
      msg: "link"
    when: testpath3 is link
  - debug:
      msg: "link"
    when: testpath4 is link
  - debug:
      msg: "mount"
    when: testpath5 is mount
  - debug:
      msg: "exists"
    when: testpath1 is exists
```

5,判断字符串的一些tests
  - lower:判断包含字母的字符串中的字母是否是纯小写,字符串中的字母全部为小写则返回真
  - upper:判断包含字母的字符串中的字母是否是纯大写,字符串中的字母全部为大写则返回真
  - 如下示例:
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars:
    str1: "abc"
    str2: "ABC"
  tasks:
  - debug:
      msg: "This string is all lowercase"
    when: str1 is lower
  - debug:
      msg: "This string is all uppercase"
    when: str2 is upper
```

6,判断整除的一些tests
  - even :判断数值是否是偶数,是偶数则返回真
  - odd :判断数值是否是奇数,是奇数则返回真
  - divisibleby(num) :判断是否可以整除指定的数值,如果除以指定的值以后余数为0，则返回真
  - 如下示例:
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars:
    num1: 4
    num2: 7
    num3: 64
  tasks:
  - debug:
      msg: "An even number"
    when: num1 is even
  - debug:
      msg: "An odd number"
    when: num2 is odd
  - debug:
      msg: "Can be divided exactly by"
    when: num3 is divisibleby(8)
```

7,其他一些tests
  - version:可以用于对比两个版本号的大小,或者与指定的版本号进行对比，使用语法为 version(‘版本号', ‘比较操作符')
```yaml
---
- hosts: me
  remote_user: liawne
  vars:
    ver: 7.4.1708
    ver1: 7.4.1707
  tasks:
  - debug:
      msg: "This message can be displayed when the ver is greater than ver1"
    when: ver is version(ver1,">")
  - debug:
      msg: "system version {{ansible_distribution_version}} greater than 7.3"
    when: ansible_distribution_version is version("7.3","gt")
    # ”>”与”gt”都表示”大于”,当使用version时，支持多种风格的比较操作符，你可以根据自己的使用习惯进行选择，version支持的比较操作符如下
    # 大于:>, gt
    # 大于等于:>=, ge
    # 小于:<, lt
    # 小于等于:<=, le
    # 等于: ==, =, eq
    # 不等于:!=, <>, ne
```
  - subset:判断一个list是不是另一个list的子集,是另一个list的子集时返回真
  - superset : 判断一个list是不是另一个list的父集,是另一个list的父集时返回真
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars:
    a:
    - 2
    - 5
    b: [1,2,3,4,5]
  tasks:
  - debug:
      msg: "A is a subset of B"
    when: a is subset(b)
  - debug:
      msg: "B is the parent set of A"
    when: b is superset(a)
  # 注:2.5版本中上述两个tests从issubset和issuperset更名为subset和superset
```
  - string:判断对象是否是一个字符串,是字符串则返回真
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars:
    testvar1: 1
    testvar2: "1"
    testvar3: a
  tasks:
  - debug:
      msg: "This variable is a string"
    when: testvar1 is string
  - debug:
      msg: "This variable is a string"
    when: testvar2 is string
  - debug:
      msg: "This variable is a string"
    when: testvar3 is string
  # 上例playbook中只有testvar2和testvar3会被判断成字符串,testvar1不会
```
  - number:判断对象是否是一个数字,是数字则返回真
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars:
    testvar1: 1
    testvar2: "1"
    testvar3: 00.20
  tasks:
  - debug:
      msg: "This variable is number"
    when: testvar1 is number
  - debug:
      msg: "This variable is a number"
    when: testvar2 is number
  - debug:
      msg: "This variable is a number"
    when: testvar3 is number
 # 上例playbook中只有testvar1和testvar3会被判断成数字,testvar2不会
```
### 条件判断与block ###
1,在ansible中,可以使用"block"关键字将多个任务整合成一个"块",这个"块"将被当做一个整体,我们可以对这个"块"添加判断条件,当条件成立时,则执行这个块中的所有任务
  - 如下示例:
```yaml
---
- hosts: me
  remote_user: root
  tasks:
  - debug:
      msg: "task1 not in block"
  - block:
      - debug:
          msg: "task2 in block1"
      - debug:
          msg: "task3 in block1"
    when: 2 > 1
```

2,block用在错误处理的场景下
  - 之前的方式如下:
```yaml
---
- hosts: me
  remote_user: root
  tasks:
  - shell: 'ls /ooo'
    register: return_value
    ignore_errors: true
  # 通过注册变量,且忽略错误(ignore_errors)来做分支判断
  - debug:
      msg: "I cought an error"
    when: return_value is failed
```
  - 使用block+rescue的方式
```yaml
---
- hosts: me
  remote_user: root
  tasks:
  - block:
      - shell: 'ls /ooo'
    # rescue直接承接上面的block,当block中的内容出现错误时,执行rescue中的内容
    # rescue关键字与block关键字对齐,rescue的字面意思为"救援",表示当block中的任务执行失败时,会执行rescue中的任务进行补救
    # rescue中的内容由自己定义
    rescue:
      - debug:
          msg: 'I caught an error'
```
  - block的优势,如下示例:
```yaml
---
- hosts: me
  remote_user: root
  tasks:
  - block:
  # block可以接多个任务,任何一个任务出错,都会执行rescue中的任务,所以通常使用block和rescue结合,完成"错误捕捉,报出异常"的功能
  # 不仅block中可以有多个任务,rescue中也可以定义多个任务,当block中的任何一个任务出错时,会按照顺序执行rescue中的任务.
      - shell: 'ls /opt'
      - shell: 'ls /testdir'
      - shell: 'ls /c'
    rescue:
      - debug:
          msg: 'I caught an error'
```
  - block+rescue+always的使用示例:
```yaml
# 还能再加入always关键字,加入always关键字以后,无论block中的任务执行成功还是失败,always中的任务都会被执行
---
- hosts: me
  remote_user: root
  tasks:
  - block:
      - debug:
          msg: 'I execute normally'
      - command: /bin/false
      - debug:
          msg: 'I never execute, due to the above task failing'
    rescue:
      - debug:
          msg: 'I caught an error'
      - command: /bin/false
      - debug:
          msg: 'I also never execute'
    always:
      - debug:
          msg: "This always executes"
# 如上例所示,block中有多个任务,rescue中也有多个任务,上例中故意执行"/bin/false"命令,模拟任务出错的情况,当block中的'/bin/false'执行后,其后的debug任务将不会被执行,因为'/bin/false'模拟出错,出错后直接执行rescue中的任务,在执行rescue中的任务时,会先输出 ‘I caught an error',然后又在rescue中使用'/bin/false'模拟出错的情况,出错后之后的debug任务不会被执行,直接执行always中的任务,always中的任务一定会被执行,无论block中的任务是否出错
```
### 条件判断与错误处理 ###
1,使用fail模块来处理出错
  - 如下示例:
```yaml
---
- hosts: me
  remote_user: root
  tasks:
  - shell: "echo 'This is a string for testing--error'"
    register: return_value
  - fail:
      msg: "Conditions established,Interrupt running playbook"
    when: "'error' in return_value.stdout"
  - debug:
      msg: "I never execute,Because the playbook has stopped"

# a, 当使用"in"或者"not in"进行条件判断时,整个条件需要用引号引起,并且,需要判断的字符串也需要使用引号引起,所以,使用'in'或者'not in'进行条件判断时,如下两种语法是正确的:
# when: ' "successful" not in return_value.stdout '
# when: " 'successful' not in return_value.stdout "
# b,fail可以单独使用,不加msg和when,效果是直接报错
```

2,使用failed_when
  - 'failed_when'的作用就是,当对应的条件成立时,将对应任务的执行状态设置为失败
```yaml
---
- hosts: me
  remote_user: root
  tasks:
  - debug:
      msg: "I execute normally"
  - shell: "echo 'This is a string for testing error'"
    register: return_value
    failed_when: ' "error" in return_value.stdout'
  - debug:
      msg: "I never execute,Because the playbook has stopped"
# 'failed_when'关键字与shell关键字对齐,表示其对应的条件是针对shell模块的,'failed_when'对应的条件是 ‘ “error" in return_value.stdout',表示"error"字符串如果存在于shell模块执行后的标准输出中,则条件成立,
# 当条件成立后,shell模块的执行状态将会被设置为失败,由于shell模块的执行状态被设置为失败,所以playbook会终止运行,于是,最后的debug模块并不会被执行
# 'failed_when'虽然会将任务的执行状态设置为失败,但并不代表任务真的失败了,以上例来说,shell模块的确是完全正常的执行了,只不过在执行之后,' failed_when'对应的条件成立了,' failed_when'将shell模块的执行状态设置为失败而已
```

3,使用changed_when
  - ‘changed_when'关键字的作用是在条件成立时,将对应任务的执行状态设置为changed
```yaml
---
- hosts: me
  remote_user: root
  tasks:
  - debug:
      msg: "test message"
    changed_when: 2 > 1
# debug模块在正常执行的情况下只能是"ok"状态,上例中,我们使用'changed_when'关键字将debug模块的执行后的状态定义为了"changed"
```
  - 与handler结合使用
```yaml
# 只有任务作出了实际的操作时（执行后状态为changed）,才会真正的执行对应的handlers
# 而在某些时候,如果想要通过任务执行后的返回值将任务的最终执行状态判定为changed,则可以使用'changed_when'关键字,以便条件成立时,可以执行对应的handlers,
# 'changed_when'除了能够在条件成立时将任务的执行状态设置为"changed",还能让对应的任务永远不能是changed状态,示例如下:
---
- hosts: me
  remote_user: root
  tasks:
  - shell: "ls /opt"
    changed_when: false
# 当将'changed_when'直接设置为false时,对应任务的状态将不会被设置为'changed',如果任务原本的执行状态为'changed',最终则会被设置为'ok',所以,上例playbook执行后,shell模块的执行状态最终为'ok'
```
### 过滤器 ###
1,过滤器是一种能够帮助我们处理数据的工具,其实,ansible中的过滤器功能来自于jinja2模板引擎

2,过滤器有些是jinja2内置的,有些是ansible特有的,如果这些过滤器都不能满足你的需求,jinja2也支持自定义过滤器

3,一些常见的过滤器
  - 字符串相关
```yaml
---
- hosts: me
  remote_user: root
  vars:
    testvar: "abc123ABC 666"
    testvar1: "  abc  "
    testvar2: '123456789'
    testvar3: "1a2b,@#$%^&"
  tasks:
  - debug:
      #将字符串转换成纯大写
      msg: "{{ testvar | upper }}"
  - debug:
      #将字符串转换成纯小写
      msg: "{{ testvar | lower }}"
  - debug:
      #将字符串变成首字母大写,之后所有字母纯小写
      msg: "{{ testvar | capitalize }}"
  - debug:
      #将字符串反转
      msg: "{{ testvar | reverse }}"
  - debug:
      #返回字符串的第一个字符
      msg: "{{ testvar | first }}"
  - debug:
      #返回字符串的最后一个字符
      msg: "{{ testvar | last }}"
  - debug:
      #将字符串开头和结尾的空格去除
      msg: "{{ testvar1 | trim }}"
  - debug:
      #将字符串放在中间,并且设置字符串的长度为30,字符串两边用空格补齐30位长
      msg: "{{ testvar1 | center(width=30) }}"
  - debug:
      #返回字符串长度,length与count等效,可以写为count
      msg: "{{ testvar2 | length }}"
  - debug:
      #将字符串转换成列表,每个字符作为一个元素
      msg: "{{ testvar3 | list }}"
  - debug:
      #将字符串转换成列表,每个字符作为一个元素,并且随机打乱顺序
      #shuffle的字面意思为洗牌
      msg: "{{ testvar3 | shuffle }}"
  - debug:
      #将字符串转换成列表,每个字符作为一个元素,并且随机打乱顺序
      #在随机打乱顺序时,将ansible_date_time.epoch的值设置为随机种子
      #也可以使用其他值作为随机种子,ansible_date_time.epoch是facts信息
      msg: "{{ testvar3 | shuffle(seed=(ansible_date_time.epoch)) }}"
```
  - 数字相关
```yaml
---
- hosts: me
  remote_user: root
  vars:
    testvar4: -1
  tasks:
  - debug:
      #将对应的值转换成int类型
      #ansible中,字符串和整形不能直接计算,比如{{ 8+'8' }}会报错
      #所以,我们可以把一个值为数字的字符串转换成整形后再做计算
      msg: "{{ 8+('8' | int) }}"
  - debug:
      #将对应的值转换成int类型,如果无法转换,默认返回0
      #使用int(default=6)或者int(6)时,如果无法转换则返回指定值6
      msg: "{{ 'a' | int(default=6) }}"
  - debug:
      #将对应的值转换成浮点型,如果无法转换,默认返回'0.0'
      msg: "{{ '8' | float }}"
  - debug:
      #当对应的值无法被转换成浮点型时,则返回指定值'8.8‘
      msg: "{{ 'a' | float(8.88) }}"
  - debug:
      #获取对应数值的绝对值
      msg: "{{ testvar4 | abs }}"
  - debug:
      #四舍五入
      msg: "{{ 12.5 | round }}"
  - debug:
      #取小数点后五位
      msg: "{{ 3.1415926 | round(5) }}"
  - debug:
      #从0到100中随机返回一个随机数
      msg: "{{ 100 | random }}"
  - debug:
      #从5到10中随机返回一个随机数
      msg: "{{ 10 | random(start=5) }}"
  - debug:
      #从5到15中随机返回一个随机数,步长为3
      #步长为3的意思是返回的随机数只有可能是5、8、11、14中的一个
      msg: "{{ 15 | random(start=5,step=3) }}"
  - debug:
      #从0到15中随机返回一个随机数,这个随机数是5的倍数
      msg: "{{ 15 | random(step=5) }}"
  - debug:
      #从0到15中随机返回一个随机数,并将ansible_date_time.epoch的值设置为随机种子
      #也可以使用其他值作为随机种子,ansible_date_time.epoch是facts信息
      #seed参数从ansible2.3版本开始可用
      msg: "{{ 15 | random(seed=(ansible_date_time.epoch)) }}"
```
  - 列表相关
```yaml
---
- hosts: me
  remote_user: root
  vars:
    testvar7: [22,18,5,33,27,30]
    testvar8: [1,[7,2,[15,9]],3,5]
    testvar9: [1,'b',5]
    testvar10: [1,'A','b',['QQ','wechat'],'CdEf']
    testvar11: ['abc',1,3,'a',3,'1','abc']
    testvar12: ['abc',2,'a','b','a']
  tasks:
  - debug:
      #返回列表长度,length与count等效,可以写为count
      msg: "{{ testvar7 | length }}"
  - debug:
      #返回列表中的第一个值
      msg: "{{ testvar7 | first }}"
  - debug:
      #返回列表中的最后一个值
      msg: "{{ testvar7 | last }}"
  - debug:
      #返回列表中最小的值
      msg: "{{ testvar7 | min }}"
  - debug:
      #返回列表中最大的值
      msg: "{{ testvar7 | max }}"
  - debug:
      #将列表升序排序输出
      msg: "{{ testvar7 | sort }}"
  - debug:
      #将列表降序排序输出
      msg: "{{ testvar7 | sort(reverse=true) }}"
  - debug:
      #返回纯数字非嵌套列表中所有数字的和
      msg: "{{ testvar7 | sum }}"
  - debug:
      #如果列表中包含列表,那么使用flatten可以'拉平'嵌套的列表
      #2.5版本中可用,执行如下示例后查看效果
      msg: "{{ testvar8 | flatten }}"
  - debug:
      #如果列表中嵌套了列表,那么将第1层的嵌套列表‘拉平'
      #2.5版本中可用,执行如下示例后查看效果
      msg: "{{ testvar8 | flatten(levels=1) }}"
  - debug:
      #过滤器都是可以自由结合使用的,就好像linux命令中的管道符一样
      #如下,取出嵌套列表中的最大值
      msg: "{{ testvar8 | flatten | max }}"
  - debug:
      #将列表中的元素合并成一个字符串
      msg: "{{ testvar9 | join }}"
  - debug:
      #将列表中的元素合并成一个字符串,每个元素之间用指定的字符隔开
      msg: "{{ testvar9 | join(' , ') }}"
  - debug:
      #从列表中随机返回一个元素
      #对列表使用random过滤器时,不能使用start和step参数
      msg: "{{ testvar9 | random }}"
  - debug:
      #从列表中随机返回一个元素,并将ansible_date_time.epoch的值设置为随机种子
      #seed参数从ansible2.3版本开始可用
      msg: "{{ testvar9 | random(seed=(ansible_date_time.epoch)) }}"
  - debug:
      #随机打乱顺序列表中元素的顺序
      #shuffle的字面意思为洗牌
      msg: "{{ testvar9 | shuffle }}"
  - debug:
      #随机打乱顺序列表中元素的顺序
      #在随机打乱顺序时,将ansible_date_time.epoch的值设置为随机种子
      #seed参数从ansible2.3版本开始可用
      msg: "{{ testvar9 | shuffle(seed=(ansible_date_time.epoch)) }}"
  - debug:
      #将列表中的每个元素变成纯大写
      msg: "{{ testvar10 | upper }}"
  - debug:
      #将列表中的每个元素变成纯小写
      msg: "{{ testvar10 | lower }}"
  - debug:
      #去掉列表中重复的元素,重复的元素只留下一个
      msg: "{{ testvar11 | unique }}"
  - debug:
      #将两个列表合并,重复的元素只留下一个
      #也就是求两个列表的并集
      msg: "{{ testvar11 | union(testvar12) }}"
  - debug:
      #取出两个列表的交集,重复的元素只留下一个
      msg: "{{ testvar11 | intersect(testvar12) }}"
  - debug:
      #取出存在于testvar11列表中,但是不存在于testvar12列表中的元素
      #去重后重复的元素只留下一个
      #换句话说就是:两个列表的交集在列表1中的补集
      msg: "{{ testvar11 | difference(testvar12) }}"
  - debug:
      #取出两个列表中各自独有的元素,重复的元素只留下一个
      #即去除两个列表的交集,剩余的元素
      msg: "{{ testvar11 | symmetric_difference(testvar12) }}"
```
  - 变量未定义时相关操作的过滤器
```yaml
---
- hosts: me
  remote_user: root
  gather_facts: no
  vars:
    testvar6: ''
  tasks:
  - debug:
      #如果变量没有定义,则临时返回一个指定的默认值
      #注:如果定义了变量,变量值为空字符串,则会输出空字符
      #default过滤器的别名是d
      msg: "{{ testvar5 | default('zsythink') }}"
  - debug:
      #如果变量的值是一个空字符串或者变量没有定义,则临时返回一个指定的默认值
      msg: "{{ testvar6 | default('zsythink',boolean=true) }}"
  - debug:
      #如果对应的变量未定义,则报出“Mandatory variable not defined."错误,而不是报出默认错误
      msg: "{{ testvar5 | mandatory }}"
```

4,上述使用到的default参数,default过滤器,还有一个很方便的用法,default过滤器不仅能在变量未定义时返回指定的值,还能够让模块的参数变得"可有可无"
  - 如下示例:
```yaml
# method one,循环了两次
- hosts: me
  remote_user: root
  gather_facts: no
  vars:
    paths:
      - path: /tmp/test
        mode: '0444'
      - path: /tmp/foo
      - path: /tmp/bar
  tasks:
  - file: dest={{item.path}} state=touch mode={{item.mode}}
    with_items: "{{ paths }}"
    when: item.mode is defined
  - file: dest={{item.path}} state=touch
    with_items: "{{ paths }}"
    when: item.mode is undefined

# method two
- hosts: me
  remote_user: root
  gather_facts: no
  vars:
    paths:
      - path: /tmp/test
        mode: '0444'
      - path: /tmp/foo
      - path: /tmp/bar
  tasks:
  - file: dest={{item.path}} state=touch mode={{item.mode | default(omit)}}
    with_items: "{{ paths }}"
# 没有对文件是否有mode属性进行判断,而是直接调用了file模块的mode参数,将mode参数的值设定为了"{{item.mode | default(omit)}}",这是什么意思呢？它的意思是,如果item有mode属性,就把file模块的mode参数的值设置为item的mode属性的值,如果item没有mode属性,file模块就直接省略mode参数,'omit'的字面意思就是"省略",换成大白话说就是:[有就用,没有就不用,可以有,也可以没有]
```
### 变量(六)include_vars ###
1,通过'vars_files'可以将文件中的变量引入playbook,以便在task中使用,但是vars_files加载时是静态的引入变量,即后续在vars_files中新增的变量,无法被引用
  - vars_files变量文件在ansible控制节点中,与目标主机无关
  - 示例如下:
```yaml
---
# 为了更加方便的操作变量文件进行测试,此处将目标主机设置为me,主机me为ansible控制主机
- hosts: me
  remote_user: root
  gather_facts: no
  vars_files:
  - /testdir/ansible/testfile
  # /testdir/ansible/testfile内容为:
  # testvar1: aaa
  # testvar2: bbb
  tasks:
  - debug:
      msg: "{{testvar1}},{{testvar2}}"
  - lineinfile:
      path: "/testdir/ansible/testfile"
      line: "testvar3: ccc"
  - debug:
      msg: "{{testvar1}},{{testvar2}},{{testvar3}}"
# 上述执行出错,因为在playbook载入vars_files对应的变量文件时,文件中只有两个变量,在执行第三个任务执行,并没有重新载入对应的变量文件
```

2,include_vars可以在任务执行过程中,随时的引入变量文件,以便动态的获取到最新的变量文件内容
  - 接上例,调整内容如下:
```yaml
---
# 为了更加方便的操作变量文件进行测试,此处将目标主机设置为me,主机me为ansible控制主机
- hosts: me
  remote_user: root
  gather_facts: no
  vars_files:
  - /testdir/ansible/testfile
  tasks:
  - debug:
      msg: "{{testvar3}}"
  - lineinfile:
      path: "/testdir/ansible/testfile"
      line: "testvar4: ddd"
  - include_vars: "/testdir/ansible/testfile"
  - debug:
      msg: "{{testvar4}}"
```
  - include_vars还有一种使用场景,有些时候,变量文件可能并没有位于ansible主机中,而是位于远程主机中,所以需要先把变量文件从远程主机中拉取到ansible主机中,当通过前面的task拉取到变量文件以后,也可以使用'include_vars'模块加载刚才拉取到的变量文件,以便后面的task可以使用变量文件中的变量.

3,include_vars的一些常用参数
  - file,如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include_vars:
      file: /testdir/ansible/testfile
  # 等效于:
  # - include_vars: "/testdir/ansible/testfile"
  - debug:
      msg: "{{testvar4}}"
```
  - 'include_vars'可以把变量文件中的变量全部赋值给另外一个变量,如下示例:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include_vars:
      file: /testdir/ansible/testfile
      name: trans_var
  - debug:
      msg: "{{trans_var}}"
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [test70] => {
    "msg": {
        "testvar1": "aaa",
        "testvar2": "bbb",
        "testvar3": "ccc",
        "testvar4": "ddd"
    }
}
# 'trans_var'变量的值就是变量文件中的所有变量,可以使用name参数指定一个变量,然后将文件中的所有变量都赋值给这个指定的变量
# 当使用name参数时,要获取到文件中的某一个变量的值,可以使用如下方法
  tasks:
  - include_vars:
      file: /testdir/ansible/testfile
      name: trans_var
  - debug:
      msg: "{{trans_var.testvar4}}"
```
  - ‘include_vars'不仅能够加载指定的变量文件,还能够一次性将指定目录下的所有变量文件中的变量加载,使用dir参数即可指定对应的目录
```yaml
---
- hosts: node2
  gather_facts: no
  remote_user: root
  tasks:
  - include_vars:
      dir: /testdir/ansible/test/
      name: trans_var
  - debug:
      msg: "{{trans_var}}"
# 上例中,使用dir参数指定了"/testdir/ansible/test/"目录,此目录中的所有变量文件都会被加载,但是在使用dir参数时,需要注意如下三点
# 第一:指定目录中的所有文件的文件后缀必须是 ‘.yaml' 、'.yml' 、'.json'中的一种,默认只有这三种后缀是合法后缀,如果目录中存在非合法后缀的文件,执行playbook时则会报错.
# 第二:如果此目录中的子目录中包含变量文件,子目录中的变量文件也会被递归的加载,而且子目录中的文件也必须遵守上述第一条规则.
# 第三:dir参数与file参数不能同时使用.
# 第一点与第二点都是默认设置,可以通过其他选项修改
# 当使用dir参数时,指定目录中的所有文件必须以 ‘.yaml' 、'.yml' 、'.json' 作为文件的后缀,如果想要手动指定合法的文件后缀名,则可以使用extensions参数指定哪些后缀是合法的文件后缀,extensions参数的值需要是一个列表
  tasks:
  - include_vars:
      dir: /testdir/ansible/test/
      extensions: [yaml,yml,json,varfile]
      name: trans_var
  - debug:
      msg: "{{trans_var}}"
# 上例中extensions参数的值为 “[yaml,yml,json,varfile]",这表示指定目录中的合法文件后缀名为yaml、yml、json和varfile.
# 当使用dir参数时,默认情况下会递归的加载指定目录及其子目录中的所有变量文件,如果想要控制递归的深度,则可以借助depth参数,示例如下
  tasks:
  - include_vars:
      dir: /testdir/ansible/test/
      depth: 1
      name: trans_var
  - debug:
      msg: "{{trans_var}}"
# 上例表示,加载"/testdir/ansible/test/"目录中的变量文件,但是其子目录中的变量文件将不会被加载,depth的值为1表示递归深度为1,默认值为0,表示递归到最底层的子目录.
# 在使用dir参数时,我们还可以借助正则表达式,匹配那些我们想要加载的变量文件,比如,我们只想加载指定目录中以"var_"开头的变量文件,则可以使用如下方法
  tasks:
  - include_vars:
      dir: /testdir/ansible/test/
      files_matching: "^var_.*"
      name: trans_var
  - debug:
      msg: "{{trans_var}}"
# 如上例所示,使用'files_matching'参数可以指定正则表达式,当指定目录中的文件名称符合正则时,则可以被加载
# 还可以明确指定,哪些变量文件不能被加载,使用'ignore_files'参数可以明确指定需要忽略的变量文件名称,'ignore_files'参数的值是需要是一个列表
  tasks:
  - include_vars:
      dir: /testdir/ansible/test/
      ignore_files: ["^var_.*",varintest.yaml]
      name: trans_var
  - debug:
      msg: "{{trans_var}}"
# 加载 /testdir/ansible/test/目录中的变量文件,但是所有以"var_"开头的变量文件和varintest.yaml变量文件将不会被加载, ‘files_matching'参数和'ignore_files'参数能够同时使用,当它们同时出现时,会先找出正则匹配到的文件,然后从中排除那些需要忽略的文件
```

3, 在2.4版本以后的ansible中,当执行了include_vars模块以后,include_vars模块会将载入的变量文件列表写入到自己的返回值中,这个返回值的关键字为'ansible_included_var_files',所以,如果我们想要知道本次任务引入了哪些变量文件,则可以使用如下方法
  - 如下示例:
```yaml
tasks:
  - include_vars:
      dir: /testdir/ansible/test/
    register: return_val
  - debug:
      msg: "{{return_val.ansible_included_var_files}}"
```
### 过滤器(二) json_query ###
1,ansible可以通过include_vars加载日志文件,debug格式化输出json/yaml内容,方便查看
  - json是yaml的子集,yaml是json的超集,yaml格式的数据和json格式的数据是可以互相转换的
  - 如下示例:
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  tasks:
  - include_vars:
      file: "/testdir/ansible/wsCdnLogList"
      name: testvar
  - debug:
      msg: "{{ testvar }}"
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => {
    "msg": {
        "logs": [
            {
                "domainName": "asia1.cdn.test.com",
                "files": [
                    {
                        "dateFrom": "2018-09-05-0000",
                        "dateTo": "2018-09-05-2359",
                        "fileMd5": "error",
                        "fileName": "2018-09-05-0000-2330_asia1.cdn.test.com.all.log.gz",
                        "fileSize": 254,
                        "logUrl": "http://log.testcd.com/log/zsy/asia1.cdn.test.com/2018-09-05-0000-2330_asia1.cdn.test.com.all.log.gz?wskey=XXXXX5a"
                    }
                ]
            },
            {
                "domainName": "image1.cdn.test.com",
                "files": [
                    {
                        "dateFrom": "2018-09-05-2200",
                        "dateTo": "2018-09-05-2259",
                        "fileMd5": "error",
                        "fileName": "2018-09-05-2200-2230_image1.cdn.test.com.cn.log.gz",
                        "fileSize": 10509,
                        "logUrl": "http://log.testcd.com/log/zsy/image1.cdn.test.com/2018-09-05-2200-2230_image1.cdn.test.com.cn.log.gz?wskey=XXXXX1c"
                    },
                    {
                        "dateFrom": "2018-09-05-2300",
                        "dateTo": "2018-09-05-2359",
                        "fileMd5": "error",
                        "fileName": "2018-09-05-2300-2330_image1.cdn.test.com.cn.log.gz",
                        "fileSize": 5637,
                        "logUrl": "http://log.testcd.com/log/zsy/image1.cdn.test.com/2018-09-05-2300-2330_image1.cdn.test.com.cn.log.gz?wskey=XXXXXfe"
                    }
                ]
            }
        ]
    }
}
# 变量文件的格式可以是yaml格式的,也可以是json格式的,上例就是将json格式的数据文件当做变量文件使用的
# 对于ansible来说,当我们把上例中的json数据文件当做变量文件引入时,就好像引入了一个我们定义好的yaml格式的变量文件一样,对于ansible来说是没有区别的
```
  - 当需要从上例中获取到logUrl时,可以使用如下方式获取:
```yaml
  tasks:
  - include_vars:
      file: "/testdir/ansible/wsCdnLogList"
      name: testvar
  - debug:
      msg: "{{ item.1.logUrl }}"
    with_subelements:
    - "{{testvar.logs}}"
    - files
```

2,上述例子中除with_subelements外,还可以使用json_query来获取
  - json_query的使用方式:
```yaml
# 假设testvarfile中内容如下
{
  "users": [
    {
      "name": "tom",
      "age": 18
    },
    {
      "name": "jerry",
      "age": 20
    }
  ]
}
# 如果要获取到所有user的name
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  tasks:
  - include_vars:
      file: "/testdir/ansible/testvarfile"
      name: testvar
  - debug:
      msg: "{{ testvar | json_query('users[*].name') }}"
# 这段数据当做变量赋值给了testvar变量,使用json_query过滤器对这个变量进行了处理,json_query('users[*].name')表示找到users列表中所有元素的name属性,输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [test70] => {
    "msg": [
        "tom",
        "jerry"
    ]
}
```
  - 更进一步
```yaml
# 假设testvarfile中内容如下
---
test:
  users:
  - name: tom
    age: 18
    hobby:
    - Skateboard
    - VideoGame
  - name: jerry
    age: 20
    hobby:
    - Music
# 要获取到所有的爱好
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  tasks:
  - include_vars:
      file: "/testdir/ansible/testvarfile1"
      name: testvar
  - debug:
      msg: "{{ testvar | json_query('test.users[*].hobby[*]') }}"
# 当数据结构中存在列表时,我们可以使用”列表名[*]”获取到列表下面的所有项,输出结果如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [me] => {
    "msg": [
        [
            "Skateboard",
            "VideoGame"
        ],
        [
            "Music"
        ]
    ]
}
# 想要根据条件获取到某个用户的某些信息
# method one
  tasks:
  - include_vars:
      file: "/testdir/ansible/testvarfile1"
      name: testvar
  - debug:
      # 此处使用了反引号,因为已经使用了'和",使用`用作区分
      msg: "{{ testvar | json_query('test.users[?name==`tom`].hobby[*]') }}"
# method two
  tasks:
  - include_vars:
      file: "/testdir/ansible/testvarfile1"
      name: testvar
  - debug:
      msg: "{{ testvar | json_query(querystring) }}"
    vars:
      querystring: "test.users[?name=='tom'].age"
# 在debug任务中使用vars关键字定义了一个只有当前debug任务能够使用的变量,从而避免了多层引号嵌套时所产生的冲突问题.

# 同时获取到用户的姓名、年龄两个属性的值,当需要同时获取多个属性值时,需要通过键值对的方式调用属性
  tasks:
  - include_vars:
      file: "/testdir/ansible/testvarfile1"
      name: testvar
  - debug:
      msg: "{{ testvar | json_query('test.users[*].{uname:name,uage:age}') }}"
# 输出内容如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [me] => {
    "msg": [
        {
            "uage": 18,
            "uname": "tom"
        },
        {
            "uage": 20,
            "uname": "jerry"
        }
    ]
}
```
  - 回到第一个示例,获取logUrl的方式如下:
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  vars_files:
  - /testdir/ansible/wsCdnLogList
  tasks:
  - debug:
      msg: "{{item}}"
    with_items: "{{ logs | json_query('[*].files[*].logUrl') }}"
```
### 过滤器(三) 其他过滤器 ###
1,其他一些常用的过滤器
```yaml
---
- hosts: me
  remote_user: liawne
  gather_facts: no
  tasks:
  ######################################################################
  #在调用shell模块时,如果引用某些变量时需要添加引号,则可以使用quote过滤器代替引号
  #示例如下,先看示例,后面会有注解
  - shell: "echo {{teststr | quote}} > /testdir/testfile"
    vars:
      teststr: "a\nb\nc"
  #上例中shell模块的写法与如下写法完全等效
  #shell: "echo '{{teststr}}' > /testdir/testfile"
  #没错,如你所见,quote过滤器能够代替引号
  #上例中,如果不对{{teststr}}添加引号,则会报错,因为teststr变量中包含"\n"转义符
  ######################################################################
  #ternary过滤器可以实现三元运算的效果 示例如下
  #如下示例表示如果name变量的值是John,那么对应的值则为Mr,否则则为Ms
  #简便的实现类似if else对变量赋值的效果
  - debug: 
      msg: "{{ (name == 'John') | ternary('Mr','Ms') }}"
    vars:
      name: "John"
  ######################################################################
  #basename过滤器可以获取到一个路径字符串中的文件名
  - debug:
      msg: "{{teststr | basename}}"
    vars:
      teststr: "/testdir/ansible/testfile"
  ######################################################################
  #获取到一个windows路径字符串中的文件名,2.0版本以后的ansible可用
  - debug:
      msg: "{{teststr | win_basename}}"
    vars:
      teststr: 'D:\study\zsythink'
  ######################################################################
  #dirname过滤器可以获取到一个路径字符串中的路径名
  - debug:
      msg: "{{teststr | dirname}}"
    vars:
      teststr: "/testdir/ansible/testfile"
  ######################################################################
  #获取到一个windows路径字符串中的文件名,2.0版本以后的ansible可用
  - debug:
      msg: "{{teststr | win_dirname}}"
    vars:
      teststr: 'D:\study\zsythink'
  ######################################################################
  #将一个windows路径字符串中的盘符和路径分开,2.0版本以后的ansible可用
  - debug:
      msg: "{{teststr | win_splitdrive}}"
    vars:
      teststr: 'D:\study\zsythink'
  #可以配合之前总结的过滤器一起使用,比如只获取到盘符,示例如下
  #msg: "{{teststr | win_splitdrive | first}}"
  #可以配合之前总结的过滤器一起使用,比如只获取到路径,示例如下
  #msg: "{{teststr | win_splitdrive | last}}"
  ######################################################################
  #realpath过滤器可以获取软链接文件所指向的真正文件
  - debug:
      msg: "{{ path | realpath }}"
    vars:
      path: "/testdir/ansible/testsoft"
  ######################################################################
  #relpath过滤器可以获取到path对于“指定路径”来说的“相对路径”
  - debug:
      msg: "{{ path | relpath('/testdir/testdir') }}"
    vars:
      path: "/testdir/ansible"
  ######################################################################
  #splitext过滤器可以将带有文件名后缀的路径从“.后缀”部分分开
  - debug:
      msg: "{{ path | splitext }}"
    vars:
      path: "/etc/nginx/conf.d/test.conf"
  #可以配置之前总结的过滤器,获取到文件后缀
  #msg: "{{ path | splitext | last}}"
  #可以配置之前总结的过滤器,获取到文件前缀名
  #msg: "{{ path | splitext | first | basename}}"
  ######################################################################
  #to_uuid过滤器能够为对应的字符串生成uuid
  - debug:
      msg: "{{ teststr | to_uuid }}"
    vars:
      teststr: "This is a test statement" 
  ######################################################################
  #bool过滤器可以根据字符串的内容返回bool值true或者false
  #字符串的内容为yes、1、True、true则返回布尔值true,字符串内容为其他内容则返回false
  - debug:
      msg: "{{ teststr | bool }}"
    vars:
      teststr: "1"
  #当和用户交互时,有可能需要用户从两个选项中选择一个,比如是否继续,
  #这时,将用户输入的字符串通过bool过滤器处理后得出布尔值,从而进行判断,比如如下用法
  #- debug:
  #    msg: "output when bool is true"
  #  when: some_string_user_input | bool
  ######################################################################
  #map过滤器可以从列表中获取到每个元素所共有的某个属性的值,并将这些值组成一个列表
  #当列表中嵌套了列表,不能越级获取属性的值,也就是说只能获取直接子元素的共有属性值.
  - vars:
      users:
      - name: tom
        age: 18
        hobby:
        - Skateboard
        - VideoGame
      - name: jerry
        age: 20
        hobby:
        - Music
    debug:
      msg: "{{ users | map(attribute='name') | list }}"
  #也可以组成一个字符串,用指定的字符隔开,比如分号
  #msg: "{{ users | map(attribute='name') | join(';') }}"
  ######################################################################
  #与python中的用法相同,两个日期类型相减能够算出两个日期间的时间差
  #下例中,我们使用to_datatime过滤器将字符串类型转换成了日期了类型,并且算出了时间差
  - debug:
      msg: '{{ ("2016-08-14 20:00:12"| to_datetime) - ("2012-12-25 19:00:00" | to_datetime) }}'
  #默认情况下,to_datatime转换的字符串的格式必须是“%Y-%m-%d %H:%M:%S”
  #如果对应的字符串不是这种格式,则需要在to_datetime中指定与字符串相同的时间格式,才能正确的转换为时间类型
  - debug:
      msg: '{{ ("20160814"| to_datetime("%Y%m%d")) - ("2012-12-25 19:00:00" | to_datetime) }}'
  #如下方法可以获取到两个日期之间一共相差多少秒
  - debug:
      msg: '{{ ( ("20160814"| to_datetime("%Y%m%d")) - ("20121225" | to_datetime("%Y%m%d")) ).total_seconds() }}'
  #如下方法可以获取到两个日期“时间位”相差多少秒,注意:日期位不会纳入对比计算范围
  #也就是说,下例中的2016-08-14和2012-12-25不会纳入计算范围
  #只是计算20:00:12与08:30:00相差多少秒
  #如果想要算出连带日期的秒数差则使用total_seconds()
  - debug:
      msg: '{{ ( ("2016-08-14 20:00:12"| to_datetime) - ("2012-12-25 08:30:00" | to_datetime) ).seconds }}'
  #如下方法可以获取到两个日期“日期位”相差多少天,注意:时间位不会纳入对比计算范围
  - debug:
      msg: '{{ ( ("2016-08-14 20:00:12"| to_datetime) - ("2012-12-25 08:30:00" | to_datetime) ).days }}'
  ######################################################################
  #使用base64编码方式对字符串进行编码
  - debug:
      msg: "{{ 'hello' | b64encode }}"
  #使用base64编码方式对字符串进行解码
  - debug:
      msg: "{{ 'aGVsbG8=' | b64decode }}"
  #######################################################################
  #使用sha1算法对字符串进行哈希
  - debug:
      msg: "{{ '123456' | hash('sha1') }}"
  #使用md5算法对字符串进行哈希
  - debug:
      msg: "{{ '123456' | hash('md5') }}"
  #获取到字符串的校验和,与md5哈希值一致
  - debug:
      msg: "{{ '123456' | checksum }}"
  #使用blowfish算法对字符串进行哈希,注:部分系统支持
  - debug:
      msg: "{{ '123456' | hash('blowfish') }}"
  #使用sha256算法对字符串进行哈希,哈希过程中会生成随机"盐",以便无法直接对比出原值
  - debug:
      msg: "{{ '123456' | password_hash('sha256') }}"
  #使用sha256算法对字符串进行哈希,并使用指定的字符串作为"盐"
  - debug:
      msg: "{{ '123456' | password_hash('sha256','mysalt') }}"
  #使用sha512算法对字符串进行哈希,哈希过程中会生成随机"盐",以便无法直接对比出原值
  - debug:
      msg: "{{ '123123' | password_hash('sha512') }}"
  #使用sha512算法对字符串进行哈希,并使用指定的字符串作为"盐"
  - debug:
      msg: "{{ '123123' | password_hash('sha512','ebzL.U5cjaHe55KK') }}"
  #如下方法可以幂等的为每个主机的密码生成对应哈希串
  #有了之前总结的过滤器用法作为基础,你一定已经看懂了
  - debug:
      msg: "{{ '123123' | password_hash('sha512', 65534|random(seed=inventory_hostname)|string) }}"
```

### lookup插件 ###
1,ansible中有很多种类的插件,比如之前总结的”tests”,也是插件的一种,ansible官网总结了各个插件的作用,并且将这些插件按照功能进行了分类
  - https://docs.ansible.com/ansible/latest/plugins/plugins.html

2,前文总结的”循环”在本质上也是一种插件,这种插件叫做”lookup插件”,先回忆一些”循环”的使用方法,以便能够更好的描述”循环”和”lookup插件”之间的关系
  - 列表示例如下:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "index is {{item.0}} , value is {{item.1}}"
    with_indexed_items: ['a','b','c']
# 等价于
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "index is {{item.0}} , value is {{item.1}}"
    loop: "{{ lookup('indexed_items',['a','b','c']) }}"
# 第一个示例使用”with_indexed_items关键字”处理列表
# 第二个示例使用”loop关键字”配合”lookup插件”处理列表
# 上例中,”lookup(‘indexed_items',[‘a','b','c'])” 这段代码就是在使用lookup插件,含义是,使用名为'indexed_items'的lookup插件处理[‘a','b','c']这个列表,'indexed_items'就是一个lookup插件
```
  - 字典示例如下:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      alice: female
      bob: male
  tasks:
  - debug:
      msg: "{{item.key}} is {{item.value}}"
    with_dict: "{{ users }}"
# 等价于
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      alice: female
      bob: male
  tasks:
  - debug:
      msg: "{{item.key}} is {{item.value}}"
    loop: "{{ lookup('dict',users) }}"
# 第一个示例使用”with_dict关键字”处理users字典变量
# 第二个示例使用”loop关键字”配合”lookup插件”处理users字典变量
# 上例中,”lookup(‘dict',users)”表示使用名为'dict'的lookup插件处理users字典变量,'dict'也是一个lookup插件
```

3,lookup插件的用法
  - lookup(‘插件名',被处理数据或参数)
  - 以”with_”开头的循环实际上就是”with_”和”lookup()”的组合,lookup插件可以作为循环的数据源

4,查看插件的帮助命令
  - 查看有哪些lookup插件可以使用: ansible-doc -t lookup -l(”-t”选项用于指定插件类型,”-l”选项表示列出列表)
  - 单独查看某个插件的使用方法,比如dict插件的使用方法: ansible-doc -t lookup dict
  - file插件可以获取到指定文件的文件内容（注:文件位于ansible主机中）,示例如下
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{ lookup('file','/testdir/testfile') }}"
# 如果想要获取多个文件中的内容,则可以传入多个文件路径,示例如下
      msg: "{{ lookup('file','/testdir/testfile','/testdir/testfile1') }}"
# file插件获得多个文件中的内容时,会将多个文件中的内容放置在一个字符串中,并用”逗号”隔开每个文件中的内容
# 想要获得一个字符串列表,将每个文件的内容当做列表中的一个独立的字符串
      msg: "{{ lookup('file','/testdir/testfile','/testdir/testfile1',wantlist=true) }}"
```

5,其他lookup插件用法示例
  - 示例如下:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  #file插件可以获取ansible主机中指定文件的内容
  - debug:
      msg: "{{ lookup('file','/testdir/testfile') }}"
  #env插件可以获取ansible主机中指定变量的值
  - debug:
      msg: "{{ lookup('env','PATH') }}"
  #first_found插件可以获取列表中第一个找到的文件
  #按照列表顺序在ansible主机中查找
  - debug:
      msg: "{{ lookup('first_found',looklist) }}"
    vars:
      looklist:
        - /testdir
        - /tmp/staging
  #当使用with_first_found时,可以在列表的最后添加- skip: true
  #表示如果列表中的所有文件都没有找到,则跳过当前任务,不会报错
  #当不确定有文件能够被匹配到时,推荐这种方式
  - debug:
      msg: "{{item}}"
    with_first_found:
      - /testdir1
      - /tmp/staging
      - skip: true
  #ini插件可以在ansible主机中的ini文件中查找对应key的值
  #如下示例表示从test.ini文件中的testA段落中查找testa1对应的值
  #测试文件/testdir/test.ini的内容如下(不包含注释符#号)
  #[testA]
  #testa1=Andy
  #testa2=Armand
  #
  #[testB]
  #testb1=Ben
  - debug:
      msg: "{{ lookup('ini','testa1 section=testA file=/testdir/test.ini') }}"
  #当未找到对应key时,默认返回空字符串,如果想要指定返回值,可以使用default选项,如下
  #msg: "{{ lookup('ini','test666 section=testA file=/testdir/test.ini default=notfound') }}"
  #可以使用正则表达式匹配对应的键名,需要设置re=true,表示开启正则支持,如下
  #msg: "{{ lookup('ini','testa[12] section=testA file=/testdir/test.ini re=true') }}"
  #ini插件除了可以从ini类型的文件中查找对应key,也可以从properties类型的文件中查找key
  #默认在操作的文件类型为ini,可以使用type指定properties类型,如下例所示
  #如下示例中,application.properties文件内容如下(不包含注释符#号)
  #http.port=8080
  #redis.no=0
  #imageCode = 1,2,3
  - debug:
      msg: "{{ lookup('ini','http.port type=properties file=/testdir/application.properties') }}"
  #dig插件可以获取指定域名的IP地址
  #此插件依赖dnspython库,可使用pip安装pip install dnspython
  #如果域名使用了CDN,可能返回多个地址
  - debug:
      msg: "{{ lookup('dig','www.baidu.com',wantlist=true) }}"
  #password插件可以生成随机的密码并保存在指定文件中
  - debug:
      msg: "{{ lookup('password','/tmp/testpasswdfile') }}"
  #以上插件还有一些参数我们没有涉及到,而且也还有很多插件没有总结,等到用到对应的插件时,再行介绍吧
  #你也可以访问官网的lookup插件列表页面,查看各个插件的用法
  #https://docs.ansible.com/ansible/latest/plugins/lookup.html
```
### 循环(八) ###
1,在新版本的ansible中,官方推荐的循环的使用方式不同,在2.6推荐的方式为loop加lookup和loop加filter
  - loop的简单使用方式:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{ item }}"
    loop:
    - teststr1
    - teststr2
```
  - loop加lookup插件替换with_X
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      alice: female
      bob: male
  tasks:
  - debug:
      msg: "{{item.key}} is {{item.value}}"
    loop: "{{ lookup('dict',users) }}"
```

2,在2.6版本开始,官方开始推荐使用"loop加filter"的方式来替代"loop加lookup"的方式
  - 示例如下:
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      alice: female
      bob: male
  tasks:
  - debug:
      msg: "{{item.key}} is {{item.value}}"
    loop: "{{ users | dict2items }}"
# users是一个字典格式的变量,它的结构是这样的
#     users:
#       alice: female
#       bob: male
# 当users字典被dict2items转换处理以后,会变成如下模样
#     users:
#     - key: alice
#       value: female
#     - key: bob
#       value: male
```

3,无论是"with_X"、"loop加lookup"还是"loop加filter",都是使用不同的方式,实现相同的功能而已

4,替换方式
  - with_list
```yaml

#loop可以替代with_list,当处理嵌套的列表时,列表不会被拉平
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    testlist:
    - a
    - [b,c]
    - d
  tasks:
  - debug:
      msg: "{{item}}"
    loop: "{{testlist}}"
```
  - with_flattened
```yaml
#flatten过滤器可以替代with_flattened,当处理多层嵌套的列表时,列表中所有的嵌套层级都会被拉平
#示例如下,flatten过滤器的用法在前文中已经总结过,此处不再赘述
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    testlist:
    - a
    - [b,c]
    - d
  tasks:
  - debug:
      msg: "{{item}}"
    loop: "{{testlist | flatten}}"
```
  - with_items
```yaml
#flatten过滤器（加参数）可以替代with_items,当处理多层嵌套的列表时,只有列表中的第一层会被拉平
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    testlist:
    - a
    - [b,c]
    - d
  tasks:
  - debug:
      msg: "{{item}}"
    loop: "{{testlist | flatten(levels=1)}}"
```
  - with_indexed_items
```yaml
#flatten过滤器（加参数）,再配合loop_control关键字,可以替代with_indexed_items
#当处理多层嵌套的列表时,只有列表中的第一层会被拉平,flatten过滤器的bug暂且忽略
#示例如下,之后会对示例进行解释
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    testlist:
    - a
    - [b,c]
    - d
  tasks:
  - debug:
      msg: "{{index}}--{{item}}"
    loop: "{{testlist | flatten(levels=1)}}"
    loop_control:
      index_var: index
# "loop_control"关键字可以用于控制循环的行为,比如在循环时获取到元素的索引.
# "index_var"是"loop_control"的一个设置选项,"index_var"的作用是让我们指定一个变量,"loop_control"会将列表元素的索引值存放到这个指定的变量中,比如如下配置
  loop_control:
    index_var: my_idx
# 上述设置表示,在遍历列表时,当前被遍历元素的索引会被放置到"my_idx"变量中,也就是说,当进行循环操作时,只要获取到"my_idx"变量的值,就能获取到当前元素的索引值
# loop_control还有其他的选项可以使用,当前暂时不列举
```
  - with_together
```yaml
#zip_longest过滤器配合list过滤器,可以替代with_together
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    testlist1: [ a, b ]
    testlist2: [ 1, 2, 3 ]
    testlist3: [ A, B, C, D ]
  tasks:
  - debug:
      msg: "{{ item.0 }} - {{ item.1 }} - {{item.2}}"
    with_together:
    - "{{testlist1}}"
    - "{{testlist2}}"
    - "{{testlist3}}"
  - debug:
      msg: "{{ item.0 }} - {{ item.1 }} - {{item.2}}"
    loop: "{{ testlist1 | zip_longest(testlist2,testlist3) | list }}"
```
  - with_nested/with_cartesian
```yaml
#product过滤器配合list过滤器,可以替代with_nested和with_cartesian
#如果你忘了with_nested和with_cartesian的用法,可以回顾前文
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    testlist1: [ a, b, c ]
    testlist2: [ 1, 2, 3, 4 ]
  tasks:
  - debug:
      msg: "{{ item.0 }}--{{ item.1 }}"
    loop: "{{ testlist1 | product(testlist2) | list }}"
```
  - with_sequence
```yaml
#range过滤器配合list过滤器可以代替with_sequence
#你可以先回顾一下with_sequence的用法,然后再测试如下示例
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{item}}"
    loop: "{{ range(0, 6, 2) | list }}"
# 需要格式化功能时:
  - debug:
      msg: "{{ 'number is %0.2f' | format(item) }}"
    loop: "{{ range(2, 7, 2) | list }}"
```
  - with_random_choice
```yaml
#使用random函数可以替代with_random_choice,由于random函数是随机取出列表中的一个值,并不涉及循环操作,所以并不用使用loop关键字.
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    testlist: [ a, b, c ]
  tasks:
  - debug:
      msg: "{{ testlist | random }}"
```
  - with_dict
```yaml
#除了上文总结的dict2items过滤器,dictsort过滤器也可以替代with_dict
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      d: daisy
      c: carol
      a: alice
      b: bob
      e: ella
  tasks:
  - debug:
      msg: "{{item.key}} -- {{item.value}}"
    loop: "{{ users | dict2items }}"
  - debug:
      msg: "{{item.0}} -- {{item.1}}"
    loop: "{{ users | dictsort }}"
```
  - with_subelements
```yaml
#subelements过滤器可以替代with_subelements
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
    - name: bob
      gender: male
      hobby:
        - Skateboard
        - VideoGame
    - name: alice
      gender: female
      hobby:
        - Music
  tasks:
  - debug:
      msg: "{{item.0.name}}'s hobby is {{item.1}}"
    with_subelements:
    - "{{users}}"
    - hobby
  - debug:
      msg: "{{item.0.name}}'s hobby is {{item.1}}"
    loop: "{{users | subelements('hobby')}}"
```

5,loop_control的其他参数使用
  - pause选项
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{item}}"
    loop:
    - 1
    - 2
    - 3
    loop_control:
      pause: 10
# 上例表示每次循环之间间隔10秒
```
  - label
```yaml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      alice:
        name: Alice Appleworth
        gender: female
        telephone: 123-456-7890
      bob:
        name: Bob Bananarama
        gender: male
        telephone: 987-654-3210
  tasks:
  - debug:
      msg: "{{item.key}}"
    loop: "{{users | dict2items}}"
# 此处输出内容太多,对于需要查找的内容较难找到
# label选项的作用,它可以在循环输出信息时,简化输出item的信息
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    users:
      alice:
        name: Alice Appleworth
        gender: female
        telephone: 123-456-7890
      bob:
        name: Bob Bananarama
        gender: male
        telephone: 987-654-3210
  tasks:
  - debug:
      msg: "{{item.key}}"
    loop: "{{users | dict2items}}"
    loop_control:
      label: "{{item.key}}"
```
### include ###
1,ansible中有类似编程语言中类似函数调用的功能,就是include,通过include,可以在一个playbook中包含另一个文件
  - include模块可以指定一个文件,这个文件中的内容是一个任务列表（一个或多个任务）,使用include模块引用对应的文件时,文件中的任务会在被引用处执行,就像写在被引用处一样
```yaml
# cat lamp.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: install_MysqlAndPhp.yml
  - yum:
      name: httpd
      state: present
 
# cat lnmp.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: install_MysqlAndPhp.yml
  - yum:
      name: nginx
      state: present
# cat install_MysqlAndPhp.yml
- yum:
    name: mysql
    state: present
- yum:
    name: php-fpm
    state: present
```

2,在handlers关键字中,也可以使用include,handlers也是一种任务,只是这种任务有相应的触发条件而已
  - 如下示例:
```yaml
# cat test_include.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - file:
     path: /opt/ttt
     state: touch
    notify: test include handlers
 
  handlers:
  - name: test include handlers
    include: include_handler.yml
 
# cat include_handler.yml
- debug:
    msg: "task1 of handlers"
- debug:
    msg: "task2 of handlers"
- debug:
    msg: "task3 of handlers"
```

3,"include"不仅能够引用任务列表,还能够引用playbook,比如,在一个playbook中引用另一个playbook
  - 示例如下:
```yaml
# cat lamp.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: install_MysqlAndPhp.yml
  - yum:
      name: httpd
      state: present
 
- include: lnmp.yml
# 在lamp.yml的结尾引入了lnmp.yml,当我们在执行lamp.yml时,会先执行lamp相关的任务,然后再执行lnmp.yml中的任务.
```

4,include还可以接受一些参数
  - 如下所示
```yaml
# cat test_include1.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: in.yml
     test_var1=hello
     test_var2=test
 
# cat in.yml
- debug:
    msg: "{{ test_var1 }}"
- debug:
    msg: "{{ test_var2 }}"
```
  - 还能够使用vars关键字,以key: value变量的方式传入参数变量
```yaml
  tasks:
  - include: in.yml
    vars:
     test_var1: hello
     test_var2: test
```
  - 通过vars关键字也能够传入结构稍微复杂的变量数据,以便在包含的文件中使用,示例如下:
```yaml
# cat test_include1.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: in.yml
    vars:
     users:
      bob:
        gender: male
      lucy:
        gender: female
         
# cat in.yml
- debug:
    msg: "{{ item.key}} is {{ item.value.gender }}"
  loop: "{{ users | dict2items }}"
```

5,可以针对某个include去打标签
  - 如下示例:
```yaml
# cat test_include1.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: in1.yml
    tags: t1
  - include: in2.yml
    tags: t2
 
# cat in1.yml
- debug:
    msg: "task1 in in1.yml"
- debug:
    msg: "task2 in in1.yml"
 
# cat in2.yml
- debug:
    msg: "task1 in in2.yml"
- debug:
    msg: "task2 in in2.yml"
```
  - 可以对include添加条件判断,还可以对include进行循环操作
```yaml
# cat test_include1.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: in3.yml
    when: 2 > 1
  - include: in3.yml
    loop:
    - 1
    - 2
    - 3
 
# cat in3.yml
- debug:
    msg: "task1 in in3.yml"
- debug:
    msg: "task2 in in3.yml"
# 循环的调用多个任务,可以使用上例中的方法,将需要循环调用的多个任务写入到一个yml文件中,然后使用include调用这个yml文件,再配合loop进行循环即可
```

6,loop_control中loop_var的使用
  - 如下示例:
```yaml
# cat A.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: B.yml
    loop:
    - 1
    - 2
 
# cat B.yml
- debug:
    msg: "{{item}}--task in B.yml"
  loop:
  - a
  - b
  - c
# B.yml中循环调用了debug模块,而在A.yml中,又循环的调用了B.yml,当出现这种"双层循环"的情况时,B文件中的item信息只打印B中的列表内容
```
  - 若要获取上例中外层item的值,可以使用loop_control中loop_var选项
```yaml
# cat A.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include: B.yml
    loop:
    - 1
    - 2
    loop_control:
      loop_var: outer_item
 
# cat B.yml
- debug:
    msg: "{{outer_item}}--{{item}}--task in B.yml"
  loop:
  - a
  - b
  - c
# 将loop_var选项的值设置为"outer_item",这表示,我们将外层循环的item值存放在了"outer_item"变量中,在B文件中的debug任务中,同时输出了"outer_item"变量和"item"变量的值
```
### include(二) ###
1,"include"的某些原始用法在之后的版本中可能会被弃用,在之后的版本中,会使用一些新的关键字代替这些原始用法

2,include_task模块
  - include模块可以用来包含一个任务列表,include_tasks模块的作用也是用来包含一个任务列表,在之后的版本中,如果我们想要包含一个任务列表,那么就可以使用"include_tasks"关键字代替"include"关键字,示例如下:
```yaml
# cat intest.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "test task1"
  - include_tasks: in.yml
  - debug:
      msg: "test task2"
 
# cat in.yml
- debug:
    msg: "task1 in in.yml"
- debug:
    msg: "task2 in in.yml"
# 当我们使用"include_tasks"时,"include_tasks"本身会被当做一个"task",这个"task"会把被include的文件的路径输出在控制台中
# "include"是透明的,"include_tasks"是可见的,"include_tasks"更像是一个任务,这个任务包含了其他的一些任务
```

3,在2.7版本之后,include_tasks模块加入了file和apply参数
  - file参数
```yaml
  - include_tasks:
      file: in.yml
  - include_tasks: in.yml
# 两种方式其实完全相同,只不过一个使用了"file"参数的方式,另一个使用了"free_form"的方式,虽然语法上不同,但是本质上没有区别
```
  - apply参数
```yaml
# cat intest.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "test task1"
  - include_tasks:
      file: in.yml
    tags: t1
  - debug:
      msg: "test task2"
# "include_tasks"这个任务本身被调用了,而"include_tasks"对应文件中的任务却没有被调用
# "include_tasks"与"include"并不相同,标签只会对"include_tasks"任务本身生效,而不会对其中包含的任务生效
# 如果想要tags对"include_tasks"中包含的所有任务都生效,需要使用到"include_tasks"模块的apply参数
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include_tasks:
      file: in.yml
      apply:
        tags:
        - t1
# 但如上写法并未按预期执行,连include_tasks都没有执行,需要完成预期内容,需要如下配置
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - include_tasks:
      file: in.yml
      apply:
        tags:
        - t1
    tags: always
# 在使用"include_tasks"时,不仅使用apply参数指定了tags,同时还使用tags关键字,对"include_tasks"本身添加了always标签
```

4,import_tasks
  - 如果想要包含引用一个任务列表,也可以使用"import_tasks"关键字
```yaml
# cat intest1.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "test task"
  - import_tasks: in.yml
 
# cat in.yml
- debug:
    msg: "task1 in in.yml"
- debug:
    msg: "task2 in in.yml"
# "import_tasks"模块并不会像"include_tasks"模块那样,在控制台中输出相关的任务信息,"import_tasks"是相对透明的
```
  - "import_tasks"是静态的,"include_tasks"是动态的
```yaml
# "静态"的意思就是被include的文件在playbook被加载时就展开了（是预处理的）.
# "动态"的意思就是被include的文件在playbook运行时才会被展开（是实时处理的）.
# 由于"include_tasks"是动态的,所以,被include的文件的文件名可以使用任何变量替换.
# 由于"import_tasks"是静态的,所以,被include的文件的文件名不能使用动态的变量替换.
# cat intest3.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  vars:
    file_name: in.yml
  tasks:
  - import_tasks: "{{file_name}}"
  - include_tasks: "{{file_name}}"
# 上述内容可以正常执行
# cat intest3.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - set_fact:
      file_name: in.yml
  - import_tasks: "{{file_name}}"
  - include_tasks: "{{file_name}}"
# 上述内容执行报错
# 当使用静态的import时,请确保文件名中使用到的变量被定义在vars中、vars_files中、或者extra-vars中,静态的import不支持其他方式传入的变量
```
  - 如果想要对包含的任务列表进行循环操作,则只能使用"include_tasks"关键字,不能使用"import_tasks"关键字,"import_tasks"并不支持循环操作,使用"loop"关键字或"with_items"关键字对include文件进行循环操作时,只能配合"include_tasks"才能正常运行
  - 使用when做条件判断执行include_tasks和import_tasks时,区别很大
```yaml
# 当对"include_tasks"使用when进行条件判断时,when对应的条件只会应用于"include_tasks"任务本身,当执行被包含的任务时,不会对这些被包含的任务重新进行条件判断.
# 当对"import_tasks"使用when进行条件判断时,when对应的条件会应用于被include的文件中的每一个任务,当执行被包含的任务时,会对每一个被包含的任务进行同样的条件判断.
# cat intest4.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - name: '----------set testvar to 0'
    set_fact:
      testnum: 0
  - debug:
      msg: '-----include_tasks-----enter the in1.yml-----'
  - include_tasks: in1.yml
    when: testnum == 0
 
  - name: '----------set testvar to 0'
    set_fact:
      testnum: 0
  - debug:
      msg: '-----import_tasks-----enter the in1.yml-----'
  - import_tasks: in1.yml
    when: testnum == 0
 
# cat in1.yml
- set_fact:
    testnum: 1
 
- debug:
    msg: "task1 in in1.yml"
# 输出内容如下:
PLAY [node2] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
 
TASK [----------set testvar to 0] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2]
 
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => {
    "msg": "-----include_tasks-----enter the in1.yml-----"
}
 
TASK [include_tasks] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
included: /testdir/ansible/in1.yml for node2
 
TASK [set_fact] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2]
 
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => {
    "msg": "task1 in in1.yml"
}
 
TASK [----------set testvar to 0] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2]
 
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2] => {
    "msg": "-----import_tasks-----enter the in1.yml-----"
}
 
TASK [set_fact] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ok: [node2]
 
TASK [debug] xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
skipping: [node2]
 
PLAY RECAP xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
node2                     : ok=8    changed=0    unreachable=0    failed=0
```
- tags的使用,与"include_tasks"不同,当为"import_tasks"添加标签时,tags是针对被包含文件中的所有任务生效的,与"include"关键字的效果相同.
- "include_tasks"与"import_tasks"都可以在handlers中使用,并没有什么不同 
  5,import_playbook
  - 使用"include"关键字除了能够引用任务列表,还能够引用整个playbook,在之后的版本中,如果想要引入整个playbook,则需要使用"import_playbook"模块代替"include"模块,因为在2.8版本以后,使用"include"关键字引用整个playbook的特性将会被弃用
  - 示例如下:
```yaml
# cat intest6.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "test task in intest6.yml"
 
- import_playbook: intest7.yml
 
# cat intest7.yml
---
- hosts: node2
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "test task in intest7.yml"
```
### jinja2模板(一) ###
1,对远端机器进行操作时,很多场景下都需要根据主机信息进行配置,这个时候可以使用template模块来完成相应的目的
  - 示例如下:
```yaml
# cat temptest.yml
---
- hosts: node2
 remote_user: root
 gather_facts: no
 tasks:
 - yum:
     name: redis
     state: present
 - template:
     src: /testdir/ansible/redis.conf
     dest: /etc/redis.conf
# cat /testdir/ansible/redis.conf,下列的ansible_host是host alias,需要注意
......
bind {{ansible_host}}
......
```
  - template还有其他的选项
```
# owner,group,mode(mode=0644,mode=u+x)
# force参数: 当远程主机的目标路径中已经存在同名文件,并且与最终生成的文件内容不同时,是否强制覆盖,可选值有yes和no,默认值为yes,表示覆盖,如果设置为no,则不会执行覆盖拷贝操作,远程主机中的文件保持不变
# backup参数: 当远程主机的目标路径中已经存在同名文件,并且与最终生成的文件内容不同时,是否对远程主机的文件进行备份,可选值有yes和no,当设置为yes时,会先备份远程主机中的文件,然后再将最终生成的文件拷贝到远程主机
```

2,上述例子中使用的template模块渲染,使用的都是jinja2模板引擎
```
  - \{\{      \}\}     :用来装载表达式,比如变量、运算表达式、比较表达式等.
  - \{\%      \%\}     :用来装载控制语句,比如 if 控制结构,for循环控制结构.
  - \{\#      \#\}     :用来装载注释,模板文件被渲染后,注释不会包含在最终生成的文件中.
```

3,双花括号的使用
  - 变量的引用,如下示例:
```bash
# ansible node2 -m template -e "testvar1=teststr" -a "src=test.j2 dest=/opt/test"
# cat test.j2
test jinja2 variable
test {{ testvar1 }} test

# 输出内容如下:
# cat test
test jinja2 variable
test teststr tesT
# "{{  }}"中包含的就是一个变量,当模板被渲染后,变量的值被替换到了最终的配置文件中
```
  - 包含表达式,示例如下:
```bash
## 比较表达式:
# cat test.j2
jinja2 test
{{ 1 == 1 }}
{{ 2 != 2 }}
{{ 2 > 1 }}
{{ 2 >= 1 }}
 
生成文件内容如下:
# cat test
jinja2 test
True
False
True
True

## 逻辑运算:
# cat test.j2
jinja2 test
{{ (2 > 1) or (1 > 2) }}
{{ (2 > 1) and (1 > 2) }}
 
{{ not true }}
{{ not True }}
 
# cat test
jinja2 test
True
False
 
False
False

## 算数运算:
# cat test.j2
jinja2 test
{{ 3 + 2 }}
{{ 3 - 4 }}
{{ 3 * 5 }}
{{ 2 ** 3 }}
{{ 7 / 5 }}
{{ 7 // 5 }}
{{ 17 % 5 }}
 
# cat test
jinja2 test
5
-1
15
8
1.4
1
2

## 成员运算:
# cat test.j2
jinja2 test
{{ 1 in [1,2,3,4] }}
{{ 1 not in [1,2,3,4] }}
 
# cat test
jinja2 test
True
False

## 一些基础的数据类型,都可以包含在"{{  }}"中,jinja2本身就是基于python的模板引擎,所以,python的基础数据类型都可以包含在"{{  }}"中
# cat test.j2
jinja2 test
### str
{{ 'testString' }}
{{ "testString" }}
### num
{{ 15 }}
{{ 18.8 }}
### list
{{ ['Aa','Bb','Cc','Dd'] }}
{{ ['Aa','Bb','Cc','Dd'].1 }}
{{ ['Aa','Bb','Cc','Dd'][1] }}
### tuple
{{ ('Aa','Bb','Cc','Dd') }}
{{ ('Aa','Bb','Cc','Dd').0 }}
{{ ('Aa','Bb','Cc','Dd')[0] }}
### dic
{{ {'name':'bob','age':18} }}
{{ {'name':'bob','age':18}.name }}
{{ {'name':'bob','age':18}['name'] }}
### Boolean
{{ True }}
{{ true }}
{{ False }}
{{ false }}
 
 
生成文件内容如下:
# cat test
jinja2 test
### str
testString
testString
### num
15
18.8
### list
['Aa', 'Bb', 'Cc', 'Dd']
Bb
Bb
### tuple
('Aa', 'Bb', 'Cc', 'Dd')
Aa
Aa
### dic
{'age': 18, 'name': 'bob'}
bob
bob
### Boolean
True
True
False
False
```
  - 过滤器也可以在双花括号中使用
```bash
# cat test.j2
jinja2 test
{{ 'abc' | upper }}
 
 
生成文件内容
# cat test
jinja2 test
ABC
```
  - jinja2的tests自然也能够在"双花括号"中使用
```bash
# cat test.j2
jinja2 test
{{ testvar1 is defined }}
{{ testvar1 is undefined }}
{{ '/opt' is exists }}
{{ '/opt' is file }}
{{ '/opt' is directory }}
 
# ansible node2 -m template -e "testvar1=1 testvar2=2" -a "src=test.j2 dest=/opt/test"
 
生成文件内容
# cat test
jinja2 test
True
False
True
False
True
```
  - lookup插件在双花括号中的使用
```bash
# cat /testdir/ansible/test.j2
jinja2 test
 
{{ lookup('file','/testdir/testfile') }}
 
{{ lookup('env','PATH') }}
 
test jinja2
 
 
# ansible主机中的testfile内容如下
# cat /testdir/testfile
testfile in ansible
These are for testing purposes only
 
# 生成文件内容如下
# cat test
jinja2 test
 
testfile in ansible
These are for testing purposes only
 
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
 
test jinja2
```

3,{# #}的使用
  - 同编程语言中注释方式的使用
```bash
# cat test.j2
jinja2 test
{#这是一行注释信息#}
jinja2 test
{#
这是多行注释信息,
模板被渲染以后,
最终的文件中不会包含这些信息
#}
jinja2 test
 
 
生成文件内容如下:
# cat test
jinja2 test
jinja2 test
jinja2 test
```

### jinja2模板(二) ###
1,结合if的使用
  - 通用结构
```yaml
{% if 条件一 %}
...
{% elif 条件N %}
...
{% else %}
...
{% endif %}
```
  - 三元运算
```bash
# cat test.j2
jinja2 test
{{ 'a' if 2>1 else 'b' }}
# 输出内容如下:
# cat /opt/test
jinja2 test
a
```

2,使用set
  - 如下示例:
```bash
# cat test.j2
jinja2 test
{% set teststr='abc' %}
{{ teststr }}
# 在jinja2中,使用set关键字定义变量,执行如下ad-hoc命令渲染模板
ansible node2 -m template -a "src=test.j2 dest=/opt/test"
# 输出内容如下:
# cat /opt/test
jinja2 test
abc
```

3,结合for的使用
  - 通用结构
```bash
{% for 迭代变量 in 可迭代对象 %}
{{ 迭代变量 }}
{% endfor %}
# 注: jinja2的控制语句大多都会遵循这个规则,即"XXX"控制语句需要使用"endXXX"作为结尾

## 默认换行方式:
# cat test.j2
jinja2 test
{% for i in [3,1,7,8,2] %}
{{ i }}
{% endfor %}
# ansible node2 -m template -a "src=test.j2 dest=/opt/test"
# cat /opt/test
jinja2 test
3
1
7
8
2

## 不换行方式:
# cat test.j2
jinja2 test
{% for i in [3,1,7,8,2] -%}
{{ i }}
{%- endfor %}
# 在for的结束控制符"%}"之前添加了减号"-"
# 在endfor的开始控制符"{%"之后添加到了减号"-"
# cat test
jinja2 test
31782

## 不换行,但有间隔方式
jinja2 test
{% for i in [3,1,7,8,2] -%}
{{ i }}{{ ' ' }}
{%- endfor %}
# 在循环每一项时,在每一项后面加入了一个空格字符串
# cat test
jinja2 test
3 1 7 8 2

## 不换行,但有间隔方式二
# cat test.j2
jinja2 test
{% for i in [3,1,7,8,2] -%}
{{ i~' ' }}
{%- endfor %}
# 在jinja2中,波浪符"~"就是字符串连接符,它会把所有的操作数转换为字符串,并且连接它们
```

4,结合for使用,循环字典
  - 如下示例:
```bash
# cat test.j2
jinja2 test
{% for key,val in {'name':'bob','age':18}.iteritems() %}
{{ key ~ ':' ~ val }}
{% endfor %}

# cat test
jinja2 test
age:18
name:bob
```

5,在使用for循环时,有一些内置的特殊变量可以使用
  - 当前循环操作为整个循环的第几次操作,则可以借助"loop.index"特殊变量
```jinja
# cat test.j2
jinja2 test
{% for i in [3,1,7,8,2] %}
{{ i ~ '----' ~ loop.index }}
{% endfor %}
# cat test
jinja2 test
3----1
1----2
7----3
8----4
2----5
```
  - 其他的一些循环变量
```bash
# loop.index   当前循环操作为整个循环的第几次循环,序号从1开始
# loop.index0   当前循环操作为整个循环的第几次循环,序号从0开始
# loop.revindex  当前循环操作距离整个循环结束还有几次,序号到1结束
# loop.revindex0 当前循环操作距离整个循环结束还有几次,序号到0结束
# loop.first    当操作可迭代对象中的第一个元素时,此变量的值为true
# loop.last    当操作可迭代对象中的最后一个元素时,此变量的值为true
# loop.length   可迭代对象的长度
# loop.depth   当使用递归的循环时,当前迭代所在的递归中的层级,层级序号从1开始
# loop.depth0   当使用递归的循环时,当前迭代所在的递归中的层级,层级序号从0开始
# loop.cycle()  这是一个辅助函数,通过这个函数我们可以在指定的一些值中进行轮询取值,具体参考之后的示例
```
  - 对一段内容循环的生成指定的次数,则可以借助range函数完成
```jinja
{% for i in range(3) %}
something
...
{% endfor %}
```

6,for的一些其他操作
  - 当for循环中没有使用if内联表达式时,也可以使用else块
```jinja
{% for u in userlist %}
  {{ u.name }}
{%else%}
  no one
{% endfor %}
# 只有userlist列表为空时,才会渲染else块后的内容
```
  - for支持递归操作
```jinja
{% set dictionary={ 'name':'bob','son':{ 'name':'tom','son':{ 'name':'jerry' } } }  %}
 
{% for key,value in dictionary.iteritems() recursive %}
  {% if key == 'name' %}
    {% set fathername=value %}
  {% endif %}
 
  {% if key == 'son' %}
    {{ fathername ~"'s son is "~ value.name}}
    {{ loop( value.iteritems() ) }}
  {% endif %}
{% endfor %}
# 使用了iteritems函数,在for循环的末尾,添加了recursive 修饰符,当for循环中有recursive时,表示这个循环是一个递归的循环,当需要在for循环中进行递归时,只要在需要进行递归的地方调用loop函数即可,上例中的"loop( value.iteritems() )"即为调用递归的部分,由于value也是一个字典,所以需要使用iteritems函数进行处理
bob's son is tom
tom's son is jerry
```
### jinja2模板(三) ###
1,jinja2模板文件中需要使用特殊字符
  - 最简单的方法就是直接在"双花括号"中使用引号将这类符号引起,当做纯粹的字符串进行处理
```jinja
# cat test.j2
{{  '{{' }}
{{  '}}' }}
{{ '{{ test string }}' }}
{{ '{% test string %}' }}
{{ '{# test string #}' }}
# cat test
{{
}}
{{ test string }}
{% test string %}
{# test string #}
```
  - 如果有较大的段落时,可以借助raw块,来实现需求
```jinja
# cat test.j2
{% raw %}
  {{ test }}
  {% test %}
  {# test #}
  {% if %}
  {% for %}
{% endraw %}
# "{% raw %}"与"{% endraw %}"之间的所有"{{  }}"、"{%  %}"或者"{#  #}"都不会被jinja2解析,上例模板被渲染后,raw块中的符号都会保持原样
# cat test
 
  {{ test }}
  {% test %}
  {# test #}
  {% if %}
  {% for %}
```

2,在调用模板引擎时,手动的指定一些符号,这些符号可以替换默认的区块
  - 如下示例:
```jinja
# cat test.j2
{% set test='abc' %}
 
(( test ))
 
{{ test }}
{{ test1 }}
{{ 'test' }}
{{ 'test1' }}
# 使用variable_start_string参数指定一个符号,这个符号用于替换"{{  }}"中的"{{“,同时,可以使用variable_end_string参数指定一个符号,这个符号用于替换"{{  }}"中的"}}"
# ansible node2 -m template -a "src=test.j2 dest=/opt/test variable_start_string='((' variable_end_string='))'"
# cat test
 
abc
 
{{ test }}
{{ test1 }}
{{ 'test' }}
{{ 'test1' }}

# 可以使用block_start_string参数指定一个符号, 这个符号用于替换"{%  %}"中的"{% ",可以使用block_end_string参数指定一个符号,这个符号用于替换"{%  %}"中的"%}"
```

3,jinja2中也有类似函数的东西,名字叫做宏
  - 如下示例:
```jinja
# 最简单的宏
# cat test.j2
{% macro testfunc() %}
  test string
  {% endmacro %}
   
   {{ testfunc() }}
# 指定参数的宏,包含默认值
{% macro testfunc(tv1=111) %}
  test string
    {{tv1}}
    {% endmacro %}
     
     {{ testfunc( ) }}
     {{ testfunc(666) }}
```
### ansible中的role ###
1,ansible官方定义的规范
  - role的标准目录结构
```bash
$ tree role
role
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
└── vars
    └── main.yml

# tasks目录:角色需要执行的主任务文件放置在此目录中,默认的主任务文件名为main.yml,当调用角色时,默认会执行main.yml文件中的任务,也可以将其他需要执行的任务文件通过include的方式包含在tasks/main.yml文件中.

# handlers目录:当角色需要调用handlers时,默认会在此目录中的main.yml文件中查找对应的handler

# defaults目录:角色会使用到的变量可以写入到此目录中的main.yml文件中,通常,defaults/main.yml文件中的变量都用于设置默认值,以便在你没有设置对应变量值时,变量有默认的值可以使用,定义在defaults/main.yml文件中的变量的优先级是最低的.

# vars目录:角色会使用到的变量可以写入到此目录中的main.yml文件中,看到这里你肯定会有疑问,vars/main.yml文件和defaults/main.yml文件的区别在哪里呢？区别就是,defaults/main.yml文件中的变量的优先级是最低的,而vars/main.yml文件中的变量的优先级非常高,如果你只是想提供一个默认的配置,那么你可以把对应的变量定义在defaults/main.yml中,如果你想要确保别人在调用角色时,使用的值就是你指定的值,则可以将变量定义在vars/main.yml中,因为定义在vars/main.yml文件中的变量的优先级非常高,所以其值比较难以覆盖.

# meta目录:如果你想要赋予这个角色一些元数据,则可以将元数据写入到meta/main.yml文件中,这些元数据用于描述角色的相关属性,比如 作者信息、角色主要作用等等,你也可以在meta/main.yml文件中定义这个角色依赖于哪些其他角色,或者改变角色的默认调用设定,在之后会有一些实际的示例,此处不用纠结.

# templates目录: 角色相关的模板文件可以放置在此目录中,当使用角色相关的模板时,如果没有指定路径,会默认从此目录中查找对应名称的模板文件.

# files目录:角色可能会用到的一些其他文件可以放置在此目录中,比如,当你定义nginx角色时,需要配置https,那么相关的证书文件即可放置在此目录中.

# 当然,上述目录并不全是必须的,也就是说,如果你的角色并没有相关的模板文件,那么角色目录中并不用包含templates目录,同理,其他目录也一样,一般情况下,都至少会有一个tasks目录.
```

2,调用role的方式
  - 如下示例:
```bash
# ls
testrole  test.yml

# cat test.yml
- hosts: node2
  roles:
    - testrole

# ansible-playbook test.yml

## 调用testrole角色时,test.yml会从同级目录中查找与testrole角色同名的目录
## 还有其他位置也可以被调用:
## 1,当前系统用户的家目录中的.ansible/roles目录,即 ~/.ansible/roles目录中
## 2,同级目录中的roles目录中
## 3,修改ansible的配置文件,编辑/etc/ansible/ansible.cfg配置文件,设置roles_path选项
roles_path    = /etc/ansible/roles:/opt:/testdir

## 或者在playbook中,直接接上role的绝对路径
```

3,一些role使用的问题
  - 在默认情况下,角色中的变量是全局可访问的,可以将变量的访问域变成角色所私有的,如果想要将变量变成角色私有的,则需要设置/etc/ansible/ansible.cfg文件,将private_role_vars的值设置为yes,默认情况下,"private_role_vars = yes"是被注释掉的,将前面的注释符去掉皆可
  - 默认情况下,无法多次调用同一个角色,也就是说,如下playbook只会调用一次testrole角色:
```bash
# cat test.yml
- hosts: node2
  roles:
    - role: testrole
      - role: testrole
# 如果想要多次调用同一个角色,有两种方法,如下:
# 方法一:设置角色的allow_duplicates属性 ,让其支持重复的调用.
# 方法二:调用角色时,传入的参数值不同.

# 方法一需要为角色设置allow_duplicates属性,而此属性需要设置在meta/main.yml文件中,所以我们需要在testrole中创建meta/main.yml文件,写入如下内容:
# cat testrole/meta/main.yml
allow_duplicates: true

# 方法二
# cat test.yml
# cat test.yml
- hosts: node2
  roles:
  - role: testrole
    vars:
      testvar: "zsythink"
  - role: testrole
    vars:
      testvar: "zsythink.net"
```

4,除了使用"-e"传入的变量的优先级,其他变量（包括主机变量）的优先级均低于vars/main.yml中变量的优先级

5,调试handler方法
  - 如下示例:
```bash
# cat testrole/tasks/main.yml
- debug:
    msg: "hello testrole!"
  changed_when: true
  notify: test_handler

# cat testrole/handlers/main.yml
- name: test_handler
  debug:
    msg: "this is a test handler"
```
### 常用技巧(一) ###
1,技巧一,在ansible中使用python字符串的一些特性
  - ansible基于python实现,当我们在ansible中处理字符串时,能够借助一些python的字符串特性,比如,在python中可以使用中括号(方括号)截取字符串中的一部分,在ansible中也可以利用这一特性
```bash
# cat test.yml
- hosts: me
  gather_facts: no
  vars:
    a: "vaA12345"
  tasks:
  - debug:
      msg: "{{a[2]}}"
# 使用a[2:5]获取到a字符串的第3到第5个字符（不包含第6个字符）
# 使用a[:5]获取到a字符串的第6个字符之前的所有字符（不包含第6个字符）
# 使用a[5:]获取到a字符串的第6个字符之后的所有字符（包含第6个字符）

## 之前成员运算符"in"和"not in",也是python的字符串运算符
- hosts: me
  gather_facts: no
  vars:
    a: "vaA12345"
  tasks:
  - debug:
      msg: "true"
    when: "'va' in a"
```
  - 运算符处理字符
```yaml
# 使用加号"+"连接两个字符串
- hosts: me
  gather_facts: no
  vars:
    a: "vaA12345"
    b: "vbB67890"
  tasks:
  - debug:
      msg: "{{a+b}}"

# 使用乘号"*"连续的重复输出字符串
- hosts: me
  gather_facts: no
  vars:
    a: "vaA12345"
  tasks:
  - debug:
      msg: "{{a*3}}"
```
  - 使用find查找字符串中字符位置
```yaml
- hosts: me
  gather_facts: no
  vars:
    a: "vaA12345"
  tasks:
  - debug:
      msg: "{{a.find('A1')}}"
# 输出如下:
TASK [debug] xxxxxxxxxxxxxxxxxxxxx
ok: [me] => {
    "msg": "2"
}

# 用作条件判断
- hosts: me
  gather_facts: no
  vars:
    a: "vaA12345"
  tasks:
  - debug:
      msg: "not found"
    when: a.find('A2') == -1
```

2,指定任务在某个节点上运行
  - 通过"delegate_to"关键字,可以指定某个任务在特定的主机上执行,这个特定的主机可以是目标主机中的某一个,也可以不是目标主机中的任何一个
```yaml
- hosts: node2,me
  gather_facts: no
  tasks:
  - file:
      path: "/tmp/ttt"
      state: touch
 
  - file:
      path: "/tmp/delegate"
      state: touch
    delegate_to: node3
 
  - file:
      path: "/tmp/ttt1"
      state: touch
```

3,让某个人物在ansible主机上执行,不在目标主机上执行
  - 有两种方式,如下示例:
```yaml
## delegate_to
- hosts: node2,node3
  gather_facts: no
  tasks:
  - file:
      path: "/tmp/inAnsible"
      state: touch
    delegate_to: local
 
  - file:
      path: "/tmp/test"
      state: touch

## connection: local
- hosts: node2,node3
  gather_facts: no
  tasks:
  - file:
      path: "/tmp/inAnsible"
      state: touch
    connection: local
 
  - file:
      path: "/tmp/test"
      state: touch
```

4,只跑一次的任务
  - 从网站上下载包,但目标主机共有5台
```yaml
- hosts: A,B,C,D,E
  gather_facts: no
  tasks:
  - get_url:
      url: "http://nexus.zsythink.net/repository/testraw/testfile/test.tar"
      dest: /tmp/
    # 结合本机执行一起使用
    connection: local
    run_once: true
 
  - copy:
      src: "/tmp/test.tar"
      dest: "/tmp"
```
### 常用技巧(二) ###
1,向列表中追加项
  - 如下内容:
```yaml
- hosts: me
  gather_facts: no
  vars:
   l1:
     - 1
     - 2
   l2:
     - a
     - b
     - 2
  tasks:
  - set_fact:
      l3: "{{ l1 + l2 }}"
  - debug:
      var: l3

# 直接列表相加
- hosts: me
  gather_facts: no
  vars:
   tlist:
     - 1
     - 2
  tasks:
  - set_fact:
      tlist: "{{ tlist + ['a'] }}"
  - debug:
      var: tlist

# jinja2的语法,完成上述追加元素的过程
- hosts: me
  gather_facts: no
  vars:
   tlist:
     - 1
     - 2
  tasks:
  - set_fact:
      tlist: "{% set tlist = tlist + ['a'] %}{{tlist}}"
  - debug:
      var: tlist

# 使用extend()追加
- hosts: me
  gather_facts: no
  vars:
   tlist:
     - 1
     - 2
  tasks:
  - set_fact:
      tlist: "{% set tlist = tlist + ['a'] %}{{tlist}}"
  - debug:
      var: tlist

# append追加
- hosts: me
  gather_facts: no
  vars:
   tlist:
     - 1
     - 2
  tasks:
  - set_fact:
      tlist: "{{ tlist.append('a') }}{{tlist}}"
  - debug:
      var: tlist
```

2, 在列表中插入项

  - 如下示例:
```yaml
# 使用insert()
- hosts: me
  gather_facts: no
  vars:
   tlist:
     - 11
     - 2
     - 11
  tasks:
  - set_fact:
      tlist: "{{ tlist.insert(1,'a') }}{{tlist}}"
  - debug:
      var: tlist
```

3, 在列表中删除项

  - 如下示例:
```yaml
# 使用pop
- hosts: me
  gather_facts: no
  vars:
   tlist:
     - 11
     - 2
     - 'a'
     - 'b'
     - 3
  tasks:
  - debug:
      msg: "{{tlist.pop(2)}}"

# 使用remove()
- hosts: me
  gather_facts: no
  vars:
   tlist:
     - 11
     - 'b'
     - 'a'
     - 'b'
     - 3
  tasks:
  - set_fact:
      tlist: "{{ tlist.remove('b') }}{{tlist}}"
  - debug:
      var: tlist
# for循环匹配删除
- hosts: me
  gather_facts: no
  vars:
    tlist:
    - 11
    - 'b'
    - 'a'
    - 'b'
    - 11
    - 'a11'
    - '1a'
    - '11'
  tasks:
  - set_fact:
      tlist: "{%for i in tlist%}{% if 11 in tlist%}{{tlist.remove(11)}}{%endif%}{%endfor%}{{tlist}}"
  - debug:
      var: tlist
```
