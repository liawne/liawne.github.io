---
title: bash学习
date: 2022-03-06 10:42:55
tags:
- bash
description: "通过advanced-bash-scripting学习bash的笔记内容"
---

# Learning BASH via abs

## 变量及操作符

### \#1. 特殊字符使用  
```
#   --> 注释(不包括#!)
    --> 变量替换使用 echo ${PATH#*:}
;   --> 命令分隔符,用于连接多个命令
;;  --> 用于终止一个case语句
.   --> 等同于source命令
    --> 可作为隐藏文件的文件名开头
    --> 相对路径的表示,当前路径和父目录路径
    --> 匹配单个字符
''  --> 忽略所有特殊字符,全部引用
""  --> 忽略除去`,$,\外所有的特殊字符,部分引用
,   --> 连接一串运算,只返回最后一个的结果
        let "t2 = ((a = 9, 15 / 3))"          ## set "a=9" and "t2 = 15 / 3"
    --> for file in /{,usr/}bin/*calc         ## 与{}一起使用,用作选择的一种,简略代码
1.1 变量替换中的大小写替换
    echo ${var,}   --> 第一个字符小写
    echo ${var,,}  --> 所有字符小写
    echo ${var^}   --> 第一个字符大写
    echo ${var^^}  --> 所有字符大写
\   --> 脱意符,除去特殊字符的含义,\X等效与'X'
`   --> 运算符,将命令的输出结果作为内容传递给变量
:   --> 空命令,no op,执行结果返回值为0
    --> 在if/then记过中作为占位符使用
        if condition;then :;else action;done
    --> 为需要二进制文件存在的场合提供占位符
        : ${username=`whoami`}   |  : ${1?"Usage: $0 ARGUMENT"}
    --> 作为切片使用,同python中的切片
        ${var:1} ${var:3:3}
    --> 同重定向一同使用,用于清空一个文件而不更改其权限,如果文件不存在,则创建该文件
        : > /tmp/testfile
    --> 可以用作注释符使用,但与#不同,:仍然会检查注释内容中命令是否正确
    --> 用作/etc/passwd文件和$PATH的分隔符
!   --> 更改命令返回值,反转test命令的含义
    --> ${!variable},获取变量$variable的变量值
    --> 调用历史命令,在脚本中无法使用
*   --> 通配符,用以指代在给定的目录下的任何文件
    --> 在正则表达式中,表示任意数量的(包括0)xxx
    --> 运算符,**便是乘方
?   --> 在双括号结构中,可以用做判断
        (( var0 = var1<98?9:21 ))
    --> 变量替换,当parameter不存在是,打印err_msg内容,返回值为1,两种形式的差别仅当parameter被声明且为空值时有差别
        ${parameter?err_msg}, ${parameter:?err_msg}
    --> 通配符,给定目录中的单字符文件名,在扩展正则表达式中代表单个字符
$   --> 变量替换
    --> 锚定符,代表行尾
${} --> 参数替换
$*  --> 位置参数
$@  --> 位置参数
$?  --> 命令返回值
$$  --> 进程ID
()  --> 小括号中的一组命令开启一个子shell,在脚本当中,其他部分无法读取括号中定义的变量
    --> 用于数组的初始化
        Array=(e1 e2 e3)
{}  --> 大括号展开
        echo {file1,file2}\ :{\ A," B",' C'}
    --> 扩展大括号展开
        echo {a..e}
        echo {1..10..2} 显示10以内的奇数，步长为2
    --> 代码块,内联组(inline group), anonymous function, the variate in the inline group \
        can be seen in the scripts anywhere
        { read line1
          read line2
        } < /tmp/passwd line1和line2可以直接读取passwd中的文件内容
        在{}中的代码块,一般情况下不会开启一个新的子shell
    --> {}可以用在非标准的for循环当中
        for ((n=1; n<=10; n++)); { echo -n "* $n *"; }
    --> many commands can be used in a {} section, just like a function, return the outcome
    --> 占位符, use with xargs; 'ls . | xargs -i -t cp ./{} $1'
{} \; --> 该结构用在find架构当中,不作为shell的关键字存在
        The ";" ends the -exec option of a find command sequence. It needs to be escaped to protect it from interpretation by the shell.
[]  --> test命令,[是test命令的内置指令(test的同义字),并不是test的链接
    --> 数组元素,用于展开各个序号的数组元素
    --> 正则表达式中使用,用于表示一个序列
        "[B-Pk-y]"
$[] --> 运算符,等效与(())
        echo $[$a+$b]

$[...] --> 整数扩展符, 可用于整数的计算; 'echo $[3*5]'
        
(())--> 运算符
        (( a = 23 )); (( a++ ))
<>  --> 重定向,包含>,&>,>&,>>,<,<>
    --> &> 把标准输出和标准错误都重定向
    --> >&2重定向到标准错误
    --> >> 追加
    --> [i]<>filename 打开文件filename读写,分配文件描述符i给文件,若i不存在,则默认使用stdin
    #!/bin/bash
    ## example 1-1
    lock_file=/tmp/$(basename $0).lock

    exec 300<>$lock_file 
    if ! flock -x -n 300; then
        echo "already running"
    else
        echo "starting..."
        sleep 30
    fi
    --> (command)> 进程替换, 暂且记下,后续有具体讲解
    --> << here document
    --> <<< here string
    --> askii 比较,对比文本字符顺序
\>  --> 单词边界,同时还有\<
        grep '\<the\>' textfile
[[]]--> test,比[]更灵活
|   --> 管道,将命令的输出作为输入传递给下一个命令
        echo ls -l | sh  ## 可以作为一种方式执行命令
    --> 管道运行在子shell当中,所以不能在管道中定义变量
        variable="initial_value";echo "new_value" | read variable;echo "variable = $variable"  # variable = initial_value
>|  --> 强制重定向,强制清空文件内容,即使已经设置了noclobber属性已经设置了
||  --> 或逻辑运算符,在test结构中使用
&   --> 在后台运行任务,在脚本中,命令甚至是循环结构可以运行在后台中
        for i in {1..10};do echo -n "$i"; done &
        脚本中包含后台运行的命令可能会导致脚本hang死,可以设置补救措施;
        a,在后台命令后面增加一个wait命令可以解决该问题
        b,在将结果重定向至/dev/null或者一个文件也可
        ls -l &; ehco "Done."; wait
&&  --> 逻辑和运算符,用于test结构中
-   --> 选项前缀, ls -l
    --> 从stdin重定向输入或从stdout重定向输入
        1, (cd /source/directory && tar cf - . ) | (cd /dest/directory && tar xpvf -)
        2, bunzip2 -c linux-2.6.16.tar.bz2 | tar xvf -
        3, grep Linux file1 | diff file2 -
    --> 前一个工作目录,由$OLDPWD变量保存
    --> 减号,用作逻辑运算符
--  --> 命令长格式选项前缀
        ls --all
    --> 与set结合使用,设置位置参数
        variable="one two three four five"
        set -- $variable;
        first_param=$1
        second_param=$2
        shift; shift
        echo "first parameter = $first_param"    # one
        echo "second parameter = $second_param"  # two
=   --> 赋值运算符;a=28
+   --> 逻辑运算符,相加
    --> 正则表达式,1各或者多个
    --> 特定命令的参数前缀,+为开启,-为关闭,如set命令
%   --> 逻辑运算符,余
~   --> 家目录,由$HOME保存
~+  --> 当前工作目录,由$PWD保存
~-  --> 上一个工作目录,由$OLDPWD保存
=~  --> 正则表达式中匹配
        variable="This is a fine mess."
        if [[ "$variable" =~ T.........fin*es* ]]
^   --> 行开头
    --> 参数替换中,大写替换
^^  --> 参数替换中,大写替换
' ' --> 空格,用作命令或者是变量的分隔
```
### \#2. 变量及参数

    1, 变量名是变量值的占位符,获取变量值的操作称之为变量替换
    2, 变量裸露的情况(没有前缀'$'符)
        --> 变量声明或者是赋值的时候
        --> 取消变量unset
        --> 被export
        --> 逻辑运算表达式(())中
    3, 变量的赋值可以在如下结构中
        --> '='中
        --> read结构
        --> 类似for var in xx的循环结构中
    4, 变量引用中
        --> 使用$a,不用""括起来,移除变量中的tab和newline
        --> 使用"$a",保存whitespace(空白)
    5, 使用$(...)替换`...`来进行变量赋值
    6, shell变量没有类型
    7, 特殊类型的变量
        --> 本地变量: 只在代码块或者是函数中可见
        --> 环境变量: 影响shell动作的变量,如PATH或者是PS1等
        --> 位置参数: $0,$1这些,$0表示脚本;$*和$@代表所有的位置参数
    8, 每次新建shell,在shell中会创建相应的环境变量
    9, 使用export命令来将本地变量变成环境变量
        --> 脚本中,变量只能export给子子进程,也即在脚本中声明的变量无法作用于脚本之外
    10,shift命令的作用是重新赋值位置参数,每执行一次,位置参数向左偏移一位
    	--> 默认可调用10个
    	--> $0不参与
    	--> 被重新赋值的变量是move,不是copy(只有两个位置参数,shift后,$2不存在)
    	--> shift可接受数字,表示每次偏移多少
    	--> 如下示例:   
            # 脚本名称不变,使用软链接给创建多个脚本名称,调用不同的名称,执行不同的功能
            ln -s /usr/local/bin/wh /usr/local/bin/wh-ripe
            ln -s /usr/local/bin/wh /usr/local/bin/wh-apnic
            ln -s /usr/local/bin/wh /usr/local/bin/wh-tucows
    
            E_NOARGS=75
    
            if [ -z "$1" ]; then
                echo "Usage: `basename $0` [domain-name]"
                exit $E_NOARGS
            fi
    
            case `basenam $0` in # Or: case ${0##*/} in
                "wh" 		) whois $1@whois.tucows.com;;
                "wh-ripe" 	) whois $1@whois.ripe.net;;
                "wh-apnic" 	) whois $1@whois.apnic.net;;
                "wh-cw" 	) whois $1@whois.cw.net;;
                * 			) echo "Usage: `basename $0` [domain-name]";;
            esac

### \#3.  引用

	1, grep '[Ff]irst' *.txt;在''中的特殊情况,这里并不会去除[]的作用,而是作为被保护的内容传递给grep
	2, ""保护除过$,\,`的其他符号
	3, 可以引用\n
		quote=$'\116'
		echo -e '\'${quote,}
	4, $'...'可以将8进制或者16进制数转换成ascii字符保存到变量中
	    quote=$'\042'
	5, echo -e用于转换特殊含义字符

### \#4.  测试

    测试结构:
    1, if/then/elif/else结构,结合if后面的返回值确定执行顺序
        --> if结构不仅仅只能判断[]结构,后面可借命令
        --> if和then都是关键字参数,关键字开启代码块,在同一行开启一行新语句,就语句必须终结
    2, []结构,等效于test
    3, [[]]关键字结构,类比于test
    4, ((...))和let ...结构,运算结果为非零时,返回值为0;运算结果为零时,返回值为非零
    5, test是bash的内建命令,因此在一个脚本中使用test,使用的并不是/usr/bin/test二进制文件
        --> 在脚本中需要使用/usr/bin/test命令,需要写全路径
    6, 使用[[]]替换[]做测试结构可以防止很多脚本中的逻辑错误
        --> &&, ||, <, >可以用在[[]]中,但无法在[]中使用
        --> 8进制,10进制,16进制可在[[]]中进行比较,[]会报错
    7, (())也可用于测试,展开并对括号中的变量进行运算;当运算结果非零或为0或者'true';当运算结果为零时返回1或者'false'
    8, 文件测试,如下不同场景返回true
       --> -e 文件存在
       --> -f 是普通文件,不是目录或设备文件
       --> -s 大小不为零
       --> -d 是个目录
       --> -b 块设备
       --> -c 字符设备
       --> -p 管道文件
            # 示例内容:
            function show_input_type()
            {
            [ -p /dev/fd/0 ] && echo PIPE || echo STDIN
            }
            show_input_type "Input"             # STDIN
            echo "Input" | show_input_type      # PIPE
       --> -h 符号链接
       --> -L 符号链接
       --> -S socket文件
       --> -t 文件绑定一个终端设备
           [ -t 0 ]用来测试脚本执行是不是在终端上
       --> -r/w/x/g/u/k 检测文件是否具有相应的权限
       --> -O/G你是文件所属者(组)
       --> f1 -nt f2 文件f1比f2新
       --> f1 -ot f2 文件f1比f2旧
       --> f1 -ef f2 文件f1和f2是指向同一个文件的硬链接
       --> ! 非

### \#5.  操作符
    --> bash不明白浮点数的运算,会将浮点数当作字符处理;需要处理浮点数的运算,可以使用bc操作
    --> 逻辑运算符:
    1, && 和/与
        --> if [ $condition1 ] && [ $condition2 ] Same as: if [ $condition1 -a $condition2 ]
    2, ! 非
        --> if [ ! -f $file ]
    3, || 或
        --> if [ $condition1 ] || [ $condition2 ] Same as: if [ $condition1 -o $condition2 ]
    4, &&/||在判断结构中的使用
        --> if [[ $a -eq 24 && $b -eq 24 ]] works.
        --> if [ "$a" -eq 24 && "$b" -eq 47 ] error
    5, 进制换算
        --> 可以使用$((...))结构对各个进制数进行换算
            echo $((0x9abc))
        --> 在shell中,默认使用10进制,当把8进制或者16进制数使用let赋值存储在变量中,打印后变为10进制
            let 'hex = 0x32';echo $hex
        --> 不限制进制数格式,可以使用let进行换算,可以使用10 digits + 26 lowercase characters + 26 uppercase characters + @ + _
            echo $((36#zz)) $((2#10101010)) $((16#AF16)) $((53#1aA)) # 1295 170 44822 3375
    
    --> 双括号结构:
        同let,(())结构允许逻辑展开和运算
        1, (( a = 23 )) 赋值,等号左右两侧都要有空格
        2, (( --a ))和(( a-- ))都可以进行运算
        3, (( t = a<45?7:11 ))
            echo "if a < 45;then t = 7;else t = 11;fi" # a = 23
            echo "t = $t "                             # t = 7
    
    --> 运算符优先顺序:
        if [ "$v1" -gt "$v2" -o "$v1" -lt "$v2" -a -e "$filename" ]
        # Unclear what's going on here...,不能有超过三个的叠加存在(-a/-o)
        if [[ "$v1" -gt "$v2" ]] || [[ "$v1" -lt "$v2" ]] && [[ -e "$filename" ]]
        # Much better -- the condition tests are grouped in logical sections.

### \#6.  其他变量视角

	内部变量
	1, $BASHPID(返回当前shell),不同于$$(返回父shell),虽然大部分时候两个值相同
		echo $BASHPID;(echo $BASHPID)  ## output different
		echo $$;(echo $$)			   ## output same
	2, $DIRSTACK,在目录栈中的第一个值,被pushd和popd影响
		内置变量响应dirs命令,dirs会显示栈中所有的信息
	3, $EDITOR,默认被脚本引用的编辑器,一般是vi或者emacs,nano
	4, $EUID,有效UID
	5, $FUNCNAME,在函数中,显示当前函数的名字
	6, $GROUPS,当前用户所属用户组,是一个数组
		echo ${GROUPS[1]}
	7, $HOSTTYPE,检查当前系统的硬件类型
		echo $HOSTTYPE # i686
	8, $IFS,内部边界分隔符;这个变量决定bash怎样判断词组,字符串的边界
		--> 默认为空白符(空格,tab,新行)
		--> 这个值可以被更改
			 bash -c 'set w x y z; IFS=":-;"; echo "$*"'  ## w:x:y:z
		--> 在设置IFS时需要注意,IFS对待空格和其他字符不一样
			var=" a  b c  "使用for循环打印出来时,会变成三行,而不是和','一样,变为多行(超过三行)
			对于使用在awk中的FS也存在同样的问题 
	9, $PATH,二进制文件路径
	10,$PIPESTATUS,保存上一条命令的执行返回值
	11,$PPID,父进程的PID号
	12,$PROMPT_COMMAND,保存将要被执行的命令(是否为该含义待查)
	13,$PS1,主提示符,命令行界面显示
	14,$PS2,备提示符,在需要输入额外输入是显示,显示为'>'
	15,$PS3,select loop中使用
	16,$PS4,set -x后界面显示的提示符'+'
	17,$PWD,pwd命令显示结果
	18,$REPLY,当且仅当上一条read命令无变量时,保存read的变量值
	19,$SECONDS,当前脚本运行了多长时间
		while [ "$SECONDS" -le "$TIME_LIMIT" ]
	20,$SHELLOPTS,保存set -o的options项
	21,$TMOUT,设置为一个非零值之后,超时会自动登出
		--> 可以在脚本中设置超时,在一定时间内未输入,则退出
	    TMOUT=3
	    # Prompt times out at three seconds.
	    echo "What is your favorite song?"
	    echo "Quickly now, you only have $TMOUT seconds to answer!"
	    read song
	    if [ -z "$song" ]
	    then
	        song="(no answer)"
	        # Default response.
	    fi
	
	位置参数
	1, $0,$1,$2...,位置参数,从命令行传递给脚本,函数;或者使用set进行设置
	2, $#,位置参数或者命令行参数的数目
	3, $*,所有的位置参数,视为一个word,需要使用""外加
	4, $@,同$*,但每个参数单独使用""括起来,同样也需要使用""括起来
		--> 使用IFS和$@,$*时需要注意
		--> $@和$*仅在使用双引号引用的时候有差别
	5, $!,上一个跑在后台的job的进程号
	    --> 可用于任务控制
	    	{ sleep ${TIMEOUT}; eval 'kill -9 $!' &> /dev/null; }
	    --> 另一种方式
	        TIMEOUT=30
	        count=0
	                 possibly_hanging_job & {
	                while ((count < TIMEOUT )); do
	                    eval '[ ! -d "/proc/$!" ] && ((count = TIMEOUT))'
	                    # /proc is where information about running processes is found.
	                    # "-d" tests whether it exists (whether directory exists).
	                    # So, we're waiting for the job in question to show up.
	                    ((count++))
	                    sleep 1
	                done
	            eval '[ -d "/proc/$!" ] && kill -15 $!'
	            # If the hanging job is running, kill it.
	        }
	6, $_,映射为执行的上一个命令最后一项内容
		--> du > /dev/null;echo $_ 			# du
		--> ls -al > /dev/null;echo $_ 		# -al
	7, $?,命令,函数或者是脚本的执行状态返回值
	8, $$,脚本自己的pid号,常用于创建惟一的temp文件,相较于mktemp使用更简单
	
	变量归类
	1, 使用declare/typeset完成变量定义
	2, declare/typeset属于内建命令
	3, -r,设置只读变量
		--> declare -r var1=xx 等效于 readonly var1=xx
	4, -i,设置为整数变量
		--> 设置为整数变量,允许直接进行运算,不需要expr结构
	        n=6/3
	        echo "n = $n"
	        declare -i n
	
	        echo "n = $n"
	        # n = 6/3
	        # n = 2
	5, -a,设置为数组变量
	6, -f,显示函数
		--> declare -f后未接任何参数,显示所有的函数;可以用在ssh远程连接,传递函数使用
		--> declare -f func_name仅显示func_name函数内容
	7, -x,等效于export
	8, 使用declare声明一个变量会限制其scope
	9, 使用declare可以非常方便的辨别变量,尤其是在辨认数组时
		--> declare | grep HOME
		--> Colors=([0]="purple" [1]="reddish-orange" [2]="light green");declare |grep Colors
	
	变量操作
	--> bash允许大量字符串操作,部分属于变量替换操作,部分属于UNIX的expr功能
	1, 字符串长度
		--> ${#string},显示变量string的长度
		--> expr length $string,使用expr功能显示字符串长度
		--> expr "$string" : '.*',同样是显示变量string中字符串的长度值
	2, substring从string开头匹配的字符数,substring是正则表达式
		--> expr match "$string" '$substring'
			stringZ=abcABC123ABCabc
		--> expr "$string" : '$substring'
			echo `expr match "$stringZ" 'abc[A-Z]*.2'`   # 8
			echo `expr "$stringZ" : 'abc[A-Z]*.2'` 		 # 8
	3, substring在string中第一次匹配的下标号
		--> expr index $string $substring
			echo `expr index "$stringZ" 1c`
			# 'c' (in #3 position) matches before '1'.
	
	变量取出
		1, 切片用法
		--> ${string:position}  					# 从position处开始抽取string,此处的position和length都可以是变量
		--> ${string:position:length}   			# 从position处抽取length个string字符
			stringZ=abcABC123ABCabc
			echo ${stringZ:7} 						# 23ABCabc
			echo ${stringZ:7:3} 					# 23A
			echo ${stringZ:(-4)} 					# Cabc 区别于${stringZ:-4},这种形式等效于${string:-default}
		--> ${*:position} 							# 从position处开始取位置参数
		--> ${@:position} 							# 同上一条
		--> ${*:position:length} 					# 同string 的用法,换成位置参数
	   	--> expr substr $string $position $length   # expr用法,切片用法
		--> expr match "$string" '\($substring\)'	# 获取第一次匹配的substring内容,substring为正则表达式
		--> expr "$string" : '\($substring\)'		# 同上
			echo `expr match "$stringZ" '\(.[b-c]*[A-Z]..[0-9]\)'`	 # abcABC1
			echo `expr "$stringZ" : '\(.[b-c]*[A-Z]..[0-9]\)'`  	 # abcABC1
			echo `expr "$stringZ" : '\(.......\)'`  				 # abcABC1
		--> expr match "$string" '.*\($substring\)' # 获取尾部第一次匹配的substring内容,substring为正则表达式
		--> expr "$string" : '.*\($substring\)'     # 同上
			echo `expr match "$stringZ" '.*\([A-C][A-C][A-C][a-c]*\)'`  # ABCabc
			echo `expr "$stringZ" : '.*\(......\)'`						# ABCabc
	
	变量置换
	    file=/dir1/dir2/dir3/my.file.txt
	    --> ${file#*/}             # 拿掉第一条/及其左边的字串：dir1/dir2/dir3/my.file.txt
	    --> ${file##*/}            # 拿掉最后一条/及其左边的字串：my.file.txt
	    --> ${file#*.}             # 拿掉第一个.及其左边的字串：file.txt
	    --> ${file##*.}            # 拿掉最后一个.及其左边的字串：txt
	    --> ${file%/*}             # 拿掉最后一条/及其右边的字串：/dir1/dir2/dir3
	    --> ${file%%/*}            # 拿掉第一条/及其右边的字串：（空值）
	    --> ${file%.*}             # 拿掉最后一个.及其右边的字串：/dir1/dir2/dir3/my.file
	    --> ${file%%.*}            # 拿掉第一个.及其右边的字串：/dir1/dir2/dir3/my
	        mv $filename ${filename%$1}$2  # 可以用作重命名
	
	    --> ${file:0:5}            # 提取最左边的5个字节：/dir1
	    --> ${file:5:5}            # 提取第5个字节右边的连续5个字节：/dir2
	
	    --> ${file-my.file.txt}    # 假如$file没有设定，则使用my.file.txt作传回值。（空值及非空值时不作处理）
	    --> ${file:-my.file.txt}   # 假如$file没有设定或为空值，则使用my.file.txt作传回值。（非空值时不作处理）
	        ${username-`whoami`}   #  when username has not been set, return the value of result of whoami 
	        filename=${1:-DEFAULT_FILENAME}
	
	    --> ${file+my.file.txt}    # 假如$file设为空值或非空值，均使用my.file.txt作传回值。（没设定时不作处理）
	    --> ${file:+my.file.txt}   # 若$file为非空值，则使用my.file.txt作传回值。（没设定及空值时不作处理）
	    --> ${file=my.file.txt}    # 若$file没设定，则使用my.file.txt作传回值，同时将$file赋值为my.file.txt。\
	                                （空值及非空值时不作处理）
	    --> ${file:=my.file.txt}   # 若$file没设定或为空值，则使用my.file.txt作传回值，同时将$file赋值为my.file.txt。\
	                                （非空值时不作处理）
	    --> ${file?my.file.txt}    # 若$file没设定，则将my.file.txt输出至STDERR,用于变量报错设置（空值及非空值时不作处理）
	    --> ${file:?my.file.txt}   # 若$file没设定或为空值，则将my.file.txt输出至STDERR。（非空值时不作处理）
	    --> 要分清楚unset与null及non-null这三种赋值状态。一般而言，: 与null有关，若不带 : 的话，null不受影响，若带 : 则连null \
	        也受影响。
	
	    --> ${var,,}               # 更改var的大小写,将$var中的大写字符转换成小写
	    --> ${#var}                # get the length of the variate of var
	
	变量替换
	    stringZ=abcABC123ABCabc
	    --> echo ${stringZ/abc/xyz}     # xyzABC123ABCabc,将开头的abc替换成xyz
	    --> echo ${stringZ//abc/xyz}    # xyzABC123ABCxyz,将字符串中的所有abc替换成xyz
	        abc和xyz都可以使用变量替换
	    --> echo ${stringZ/abc}         # ABC123ABCabc,不包含replacement时,则是直接删除第一处匹配内容
	    --> echo ${stringZ//abc}        # ABC123ABC,同上,删除所有的匹配内容
	    --> echo ${stringZ/#abc/XYZ}    # XYZABC123ABCabc,匹配前端的第一个,进行替换
	    --> echo ${stringZ/%abc/XYZ}    # abcABC123ABCXYZ,匹配后端的最后一个,进行替换
	    --> echo ${!varprefix*}         # 匹配所有已声明已xyz开头的变量
	    --> echo ${!varprefix@}         # 匹配所有已声明已xyz开头的变量
	        abc23=something_else
	        b=${!abc*}
	        echo "b = $b"               # b = abc23
	        c=${!b}                     # Now, the more familiar type of indirect reference.
	        echo $c


    awk的使用,等效变量替换
        String=23skidoo1
        # 012345678 Bash 变量替换中bash的下标计算方式
        # 123456789 awk  变量替换中awk的下标计算方式
        --> echo | awk '{ print substr("'"${String}"'",3,4) }'  # skid
            前面使用空echo的作用是,所谓伪输入,不需要填写输入文件

### \#7. 循环和分支结构

```bash
1, for循环
    for arg in [list]; do command;done
    --> for循环和set结合使用,会很方便,以下是例子
```
```
    set `uname -a`; for item in $(seq $#); do echo ${!item}; done
    for planet in "Mercury 36" "Venus 67" "Earth 93" "Mars 142" "Jupiter 483"; do
        set -- $planet
        echo "$1    $2,000,000 miles from the sun"
    done
```
        --> set中使用的--,避免难预测的bug,当后面的变量为空或者是以'-'开头
        --> [list]可以是一个变量,保存了多个值,用于for循环使用
        --> [list]也可以使用*通配符
        --> 无[list]项也可,循环使用的内容为位置参数
            for a; do echo -n "$a "; done       # 写入脚本后,执行脚本时,后接参数或者不接参数,得出结果不同
        --> [list]内容同样可为命令替换后的结果
            for name in $(awk 'BEGIN{FS=":"}{print $1}' < "$PASSWORD_FILE" )   # 系统上所有用户 
            for word in $(generate_list)                                       # 函数运行结果
        --> for loop结束后,在done后面可直接使用管道进行操作,例如排序等
            for file in "$( find $directory -type l )";do echo "$file"; done | sort  # 对循环执行后的结果进行排序
        --> C风格的for循环,需要用到(());
            LIMIT=10; for ((a=1; a <= LIMIT ; a++)); do echo $a; done
            for ((a=1, b=1; a <= LIMIT ; a++, b++)); do echo -n "$a-$b "; done
        --> 一般情况下,do和done分割for循环的结构,但在特定情况下,省略do和done
            for((n=1; n<=10; n++))
            {   # No do
                echo -n "* $n *"
            }   # No done!
            
            for n in 1 2 3
            { echo -n "$n "; }          # 在经典的for结构中,花括号中需要包含;,用于结尾
        --> E_NOARGS=65 --> The standards of variate naming, exit-because-no-arguments
        --> read command read reads a line every time, in the function 'while read i j', i stands for the first word, j stands for the rest of this line
        --> the difference between return and exit
            若在script里，用exit RV来指定其值，若没指定，在结束时以最后一道命令之RV为值。
            若在function里，则用return RV来代替exit RV即可。
            若在loop里，则用break
    
    2, while循环
        相对于for循环,while循环更适合用于不确定condition情况下进行的循环
        while [condition]; do command(s); done
        --> 在while循环中可能存在多个条件,但只有最后一个条件决定什么时候终止循环
            while echo "previous-variable = $previous"
                echo
                previous=$var1
                [ "$var1" != end ]; do ...
        --> 同for循环一样,while可以接收C风格的条件格式
            while (( a <= LIMIT ))
            do
                echo -n "$a"
                ((a += 1))      # let "a+=1"
            done
        --> while的条件可以直接接函数
            condition(){
                ((t++))
    
                if [ $t -lt 5 ]; then
                    return 0 # true
                else
                    return 1 # false
                fi
            }
            while condition
        --> 与read一起结合使用,得到while read结构
            cat file | while read line      # 同时是以管道作为输入内容
        --> while可以在done后使用'<'来作为内容输入
    3, until循环
        循环体在顶部,直到条件正确,才退出执行循环结构中的内容;与while循环相反
        until [ condition-is-true ]; do command(s); done
        until循环结构格式类同于for循环
        --> 条件为真才退出
            END_CONDITION=end
    		until [ "$var1" = "$END_CONDITION" ]
    		    # Tests condition here, at top of loop.
    		do
    		    echo "Input variable #1 "
    		    echo "($END_CONDITION to exit)"
    		    read var1
    		    echo "variable #1 = $var1"
    		    echo
    		done
    	--> until同样接受C风格的判断条件,使用双括号的格式(())
    		until (( var > LIMIT ))
    4, 嵌套循环
    	一个循环结构属于另一个循环结构的构成部分,称为嵌套循环
    5, 循环控制
    	影响循环行为的命令
    	break;continue
    	--> break的作用为终止当前循环
    	--> continue的作用为跳过当前这次的循环,在该分支中,跳过该分支后面需要执行的命令和操作
```
        while [ $a -le "$LIMIT" ]; do
            a=$(($a+1))
            if [ "$a" -eq 3 ] || [ "$a" -eq 11 ]; then 			# Excludes 3 and 11.
                continue										# Skip rest of this particular loop iteration.
            fi
            echo -n "$a"										# This will not execute for 3 and 11.
        done

        while [ "$a" -le "$LIMIT" ]; do
            a=$(($a+1))
            if [ "$a" -gt 2 ]; then
                break 											# Skip entire rest of loop.
            fi
            echo -n "$a"
        done
```
		--> break可以后接参数,单个的break表示终止当前循环;break N表示终止几层循环
		--> continue也可以接参数,单个的continue表示此次循环,continue N会终止当前层级的循环,开始下一次的循环,从N层开始
	6, 测试和分支
	    case和select结构不属于循环结构,但他们通过条件判断引导程序流向
	
	    --> case对标的是C/C++中的switch结构;case可以说是简化版的if/elif/elif/.../else结构,case可以用于设置程序接口
	        case "$variable" in
	        "$condition1")
	            command...
	            ;;
	        "$condition2")	
	            command...
	            ;;
	        esac
		
		--> 判断后接参数
	        E_PARAM=1
	        case "$1" in
	        "") echo "Usage: ${0##*/} <filename>"; exit $E_PARAM;;  # 提示信息,精简化
	        -*) FILENAME=./$1;;                                     # 如果后面所接参数包含破折号,将其替换为./$1,这样后面的命令嗯就不会把其当做$1
	        * ) FILENAME=$1;;
	        esac
```
    --> while和case一起使用:
        while [ $# -gt 0 ]; do
            case "$1" in
                -d|--debug)
                    DEBUG=1
                    ;;
                -c|--conf)
                    CONFFILE="$2"
                    shift
                    if [ ! -f $CONFFILE ]; then
                        echo "Error: Supplied file doesn't exist!"
                        exit $E_CONFFILE
                    fi
                    ;;
            esac
            shift
        done

    --> 做匹配函数:
                match_string (){
                    MATCH=0
                    E_NOMATCH=90
                    PARAMS=2
                    E_BAD_PARAMS=91

                    [ $# -eq $PARAMS ] || return $E_BAD_PARAMS

                    case "$1" in
                        "$2") return $MATCH;;
                        *   ) return $E_NOMATCH;;
                    esac
                }

    --> select继承自ksh,同样是一个可用于构建入口的工具
        select variable [in list]
        do
            command...
            break
        done
```

​	
```

```
		--> select结构提示用户输入给定的选项之一,默认情况下使用环境变量PS3作为提示符,但这个可以被改变
```
		PS3='Choose your favorite vegetable: ' 	 # Sets the prompt string.
											  	 # Otherwise it defaults to #? .

		select vegetable in "beans" "carrots" "potatoes" "onions" "rutabagas"
		do
			echo
			echo "Your favorite veggie is $vegetable."
			echo "Yuck!"
			echo
			break 								 # What happens if there is no 'break' here?
		done
```

		--> 如果结构中list不存在,select会使用传递给脚本或包含select结构的函数的位置参数$@;可类比for variable [in list]
```
        PS3='Choose your favorite vegetable: '
		choice_of(){
		select vegetable							# [in list] omitted, so 'select' uses arguments passed to function.
		do
			echo
			echo "Your favorite veggie is $vegetable."
			echo "Yuck!"
			echo
			break
		done
		}
		choice_of beans rice carrots radishes rutabaga spinach
		------------------------------------------------------------------------------------------------------
```
### \#8. 命令替换

	命令替换重新单个甚至多个命令的输出结果,逐字的将输出内容传递给另一个上下文
	--> 命令的输出结果可以是传递给其他命令的参数,可以是设置变量,甚至生成for循环需要使用的内容参数
	--> 命令替换有两种形式:`commands`,$(commands),两种方式等效
	--> 命令替换生成一个子shell
	--> 命令替换可能把输出结果分片
		COMMAND `echo a b` 		 # 2 args: a and b
		COMMAND "`echo a b`"	 # 1 arg: "a b"
	    note: 有时命令替换会出现不期望的结果
	    mkdir 'dir with trailing newline
	    '
	    cd "`pwd`"      # error inform
	    cd "$PWD"       # works fine
	--> 使用echo输出命令替换厚的未括变量,echo会将换行符去除
	--> 命令替换允许使用重定向或者cat来获取文件内容作为变量内容
	    echo ` <$0`     # 输出脚本内容
	--> 不要将一个长文本内容作为值赋给一个变量，也不要将二进制文件内容作为变量的值 
	--> 没有缓冲溢出的情况出现，这是翻译性语言的特性，相较编译语言提供更多的保护
	--> 变量声明甚至可以通过一个循环结构来赋值
```
    variable1=`for i in 1 2 3 4 5
    do
    echo -n "$i"
    done`
```
    --> 命令替换使用$()替换掉反引号的使用
        允许这种形式：content=$(<$File2)
    --> $()的形式允许多重嵌套

### \#9. 运算展开

    算数展开提供了一个强大的工具，用于在脚本中进行算数运算；常用的有反引号、双括号、let
    
    变种：
    --> 使用反引号的算数运算，常常和expr结合使用
        z=`expr $z + 3`
    --> 算数展开使用双括号和let，反引号的结构已经被(())，$(())和let替换
        z=$(($z+3))或者z=$((z+3))也可以
        (( n += 1 ))是正确的；(( $n += 1 ))是错误的
        let z=z+3是正确的；let "z += 3"也可以



## linux命令

### \#1. 内部命令和内部指令
​    内建指令是指包含在bash工具集内部的命令；内建命令的作用一方面是为了提升性能，常用于需要fork新进程的命令，另一方面是出于某些命令需要直接访问shell内部

    --> 父进程获取到子进程的pid后，可以传递参数给子进程，反过来则不行；这个需要注意，出现这种问题时，一般难以排查
```
    PIDS=$(pidof sh $0) # Process IDs of the various instances of this script.
    P_array=( $PIDS )
    echo $PIDS

    let "instances = ${#P_array[*]} - 1" 
    echo "$instances instance(s) of this script running."
    echo "[Hit Ctl-C to exit.]"; 
    echo

    sleep 1
    sh $0
    exit 0
```
    --> 一般来说，bash内建指令不会自动fork新的进程，外部命令或者管道过滤时会fork新进程
    --> 内建指令可能和系统命令有同样的名字，但bash会使用内建命令，echo和/bin/echo并不一样
        echo "This line uses the \"echo\" builtin."
        /bin/echo "This line uses the /bin/echo system command."
    --> 关键字是预留字、符号，或者是操作符；关键字对shell来说具有特殊的含义，是shell的语句块的构成部分；关键字属于bash的硬编码部分，不同于内建指令，关键字是命令的子单元

### \#2. IO命令

    echo：打印表达式或者变量到标准输出
    --> 和-e使用，用来打印脱意符
    --> 默认情况下，每个echo会打印一个终端换行符，当与-n一起使用时，会将换行符省略掉
    --> echo `command`的形式会删除所有由command生成的换行符
        变量$IFS默认情况下降'\n'作为分隔符之一，bash将换行符后面的内容作为参数传递给echo，echo将这些参数打印出来，使用空格分隔
    
    printf：格式化打印，增强型的echo，是一个C语言中printf()函数的限制型变体，部分内容与原函数使用不同
    printf format-string... parameter...
    --> 格式化输出
```
declare -r PI=3.14159265358979
printf "Pi to 2 decimal places = %1.2f" $PI
printf "Pi to 9 decimal places = %1.9f" $PI
```
    --> 格式化输出错误内容很实用
```
# 注意$'strings...'的格式，在此处与%s的使用
error()
{
    printf "$@" >&2 # Formats positional params passed, and sends them to stderr.
    echo
    exit $E_BADDIR
}
cd $var || error $"Can't cd to %s." "$var"
```
    read：通过标准输入读取变量值，动态的通过键盘获取值，与-a选项一起使用时可获取数组变量
    --> read通常情况下，'\'会去除换行的含义，当与-r参数一同使用时，'\'按照原意进行输出
    --> -s：不显示输入内容到屏幕上
    --> -n：设置仅接收多少字符，-n同样能接受特殊按键，但需要清楚特殊按键对应的字符，但不能获取到回车的字符
            arrowup='\[A'
            arrowdown='\[B'
            arrowrt='\[C'
            arrowleft='\[D'
            insert='\[2'
            delete='\[3'
    --> -p：在接收输入内容前，打印后续内容到屏幕上，作为提示用
    --> -t：用在设置超时时间的场景下
    --> -u：获取目标文件的文件描述符
    --> read命令同样可以通过重定向到标准输入的文件来读取变量，如果文件内容超过一行，则只有第一行内容会被用于变量读取；
    --> 当read后接的参数多余一个时，默认会以空格（或者连续空格）作为分隔符来读取变量，此行为可通过更改环境变量$IFS来改变
```
read var1 < /tmp/file1
read var1 var2 < /tmp/file1
while read line; do echo $line; done < /tmp/file1
```

### \#3. 文件系统命令

    cd：切换路径命令
    --> 使用-P参数，忽略链接文件
    --> cd -,切换到上一个目录，$OLDPWD变量保存的内容
    --> 使用两个斜杠时，cd命令会出现我们不期望的情况
```
# 以下的问题在命令行和脚本中都存在，需要注意
$ cd //
$ pwd
//
```
    pwd：显示当前工作目录路径
    -->使用该命令的效果同$PWD完全相同
    
    pushd,popd,dirs：这个命令集合是一个工作目录书签工具，用于在工作目录中有序的来回切换；后进先出的堆栈方式处理工作目录组，允许对这个堆栈进行各种不同的操作
    --> pushd dir-name把目录dir-name放入到到堆栈中（栈顶），同时切换当前目录到dir-name中去
    --> popd 将目录栈顶的目录从栈中移除，同时将工作目录切换至此时的栈顶目录中去
    --> dirs 列出栈中的目录列表，popd和pushd会引用到dirs
    
    在脚本中需要切换多个目录工作时，使用这个命令集可以方便的进行管理，$DIRSTACK数组包含有目录的列表栈内容

### \#4. 变量操作命令

    let：let命令执行对变量的算数运算操作，在很多种情况下，let相当于简化版的expr命令
```
let a=11; let a=a+5
let "a <<= 3"
let "a += 5"
let a++(++a)
let "t = a<7?7:11"
```
    --> 使用let命令，在特定情况下，命令返回值和通常情况不同
```
$ var=0
$ echo $?
0
$ let var++
$ echo $?
1
```
    eval：
    eval arg1 [arg2] ... [argN]
    结合一个表达式或者一列表达式中的参数，是这些参数联合；所有在表达式中的变量都会被展开，得出的字符串被转换为命令
```
$ command_string="ps ax"
$ eval "$command_string"
```
    --> 每次调用eval都会强制重新评估其参数
```
$ a="$b"
$ b="$c"
$ c=d
$ echo $a
$ eval echo $a
$ eval eval echo $a
```
```
params=$#
param=1
while [ "$param" -le "$params" ]
do
    echo -n "Command-line parameter "
    echo -n \$$param
    echo -n " = "
    eval echo \$$param
    (( param ++ ))
done
```
    --> 使用eval命令有一定的风险，如果存在替换的方案，尽量使用替换方案来实现目的；像是eval $COMMANDS，在命令的返回结果中可能存在危险的内容如rm -rf *等
    set：
    set命令可用于更改脚本内部的变量值或者脚本选项，用法之一是可以设置option flags来更改脚本执行的动作；另一个用法是可以是将命令的输出结果设置为位置参数。  
```
set `uname -a`
```
    --> 当单独使用set命令时，终端显示所有的环境变量以及已经设置的变量
    --> set后接--,表示将一个变量的内容设置为位置参数,当--后没有任何参数时,表示取消所有的位置参数
```
variable="one two three four five"
set -- $variable
first_param=$1
second_param=$2
shift; shift

remaining_params="$*"
echo "first parameter = $first_param"                   # one
echo "second parameter = $second_param"                 # two
echo "remaining parameters = $remaining_params"         # three four five

set --
first_param=$1
second_param=$2
echo "first parameter = $first_param"                   # (null value)
echo "second parameter = $second_param"                 # (null value)
```
    unset:
    unset命令删除一个shell变量,将变量的值设置为null,改命令不影响位置参数
    --> 大多数情况下,使用unset设置过的变量和undeclare设置过的变量是等效的;但对于${parameter:-default}还是有区分的
    
    export:
    export命令将变量的值声明至所有由脚本生成的子shell或者是shell,令其都可使用;在开机启动脚本中使用是export一个重要使用场景,有初始化环境的作用,让后启用的脚本能够继承环境变量
    --> 父进程是没有办法获取到子进程的变量的
    --> 大部分情况下,export var=xxx和var=xxx;export var是等效的,但在某些情况下有差别
```
bash$ export var=(a b); echo ${var[0]}
(a b)
bash$ var=(a b); export var; echo ${var[0]}
a
```

    declare/typeset:
    这两个命令用来设定或者限制变量的属性
    readonly:
    等效于declare -r,将一个变量设置为只读,或者实际效用为设置成一个常量,这个命令类似于C中的const
    
    getopsts:
    这个强大的命令解析命令行参数,传递给脚本使用,这个命令类似于C中的外部命令getopt,getopt库函数;命令允许传递和连接多个选项,并且为脚本联合多个参数,如下所示:
```
scriptname -abc -e /usr/local
```
    --> getopts使用两个默认的变量:
        $OPTIND(OPTion INDex)是参数指针
        $OPTARG(OPTion ARGument)是参数指定一个选项
        选项名声明时后接冒号表明这个选项有一个指定的参数
    --> getopts结构一般会同while循环使用,每次处理一个选项和参数,然后变量$OPTIND指针指向下一个值
        命令行传递给脚本的参数前面必须接'-',存在'-'的前缀,让getopts识别命令行参数为选项;实际上,没有'-'时,getopts不会去处理这些参数,直接当作缺失选项处理
        getopts模板与while循环有些许差别
        getopts结构是外部命令getopt命令的替换
```
while getopts ":abcde:fg" Option
# Initial declaration.
# a, b, c, d, e, f, and g are the options (flags) expected.

# The : after option 'e' shows it will have an argument passed with it.
do
    case $Option in
    a ) # Do something with variable 'a'.
    b ) # Do something with variable 'b'.
    ...
    e) # Do something with 'e', and also with $OPTARG, 
       # which is the associated argument passed with option 'e'.
    ...
    g ) # Do something with variable 'g'.
    esac
done
shift $(($OPTIND - 1))      # Move argument pointer to next.
                            # All this is not nearly as complicated as it looks <grin>.
```

```
NO_ARGS=0
E_OPTERROR=85
if [ $# -eq "$NO_ARGS" ]				# Script invoked with no command-line args?
then
	echo "Usage: `basename $0` options (-mnopqrs)"
	exit $E_OPTERROR
 										# Exit and explain usage.
										# Usage: scriptname -options
										# Note: dash (-) necessary
fi

while getopts ":mnopq:rs" Option
do
case $Option in
	m)      echo "Scenario #1: option -m- [OPTIND=${OPTIND}]";;
	n | o ) echo "Scenario #2: option -$Option- [OPTIND=${OPTIND}]";;
	p)      echo "Scenario #3: option -p- [OPTIND=${OPTIND}]";;
	q)      echo "Scenario #4: option -q- with argument \"$OPTARG\" [OPTIND=${OPTIND}]";;
	r | s ) echo "Scenario #5: option -$Option-";;
	*)      echo "Unimplemented option chosen.";; 		# Default.
esac
done

shift $(($OPTIND - 1))
exit $?
```

### \#5. 脚本行为命令

    source/. (点命令)
    这个命令,当在脚本外使用时,作用为调用一个脚本;
    在脚本内部使用,source file-name的作用为加载file-name的内容;
    --> source一个文件,将该文件的代码加载进入当前脚本(作用内容于C里面的#include作用)
    --> 最终结果是,一个被source过的文件,代码就像在当前脚本物理上包含了被source过的文件的代码内容;当多个脚本使用同一个数据文件时,这种方式很有用
    --> 如果被source的文件本身是一个可执行脚本,当source后,脚本会执行,然后将控制权交回调用其的脚本;一个可执行的source文件可以使用return来实现目的
    --> 参数(可选)可以传递给被source的文件作为位置参数
```
source $filename $args1 $args2
```
    --> 脚本甚至可以在执行时source自己,随然可能没有什么实际的应用场景
```
#!/bin/bash
MAXPASSCNT=100
echo -n "$pass_count "
let "pass_count += 1"

while [ "$pass_count" -le $MAXPASSCNT ]; do
    . $0
done

echo
exit 0
```

    exit
    无条件的终止一个脚本;exit命令可以选择性的后接一个整数参数,用作脚本的exit返回状态
    --> 最简单的方式是直接使用exit 0, 指明此次运行成功
    --> 如果一个脚本以未接参数的exit作为结尾,那么脚本的返回值则为最后一条命令的返回值,等效于exit $?
    --> 一个exit命令也可以用于终止一个子shell
    
    exec
    这个shell内建命令替换当前进程为一个指定的命令
    --> 通常情况下,当shell遇到一个命令时,会fork一个子进程来执行该命令,当使用exec命令时,shell不再fork子进程,并且exec调用的命令替换了shell
    --> 当exec在脚本中使用时,当exec调用的命令结束时,会强制退出脚本
```
#!/bin/bash
exec echo "Exiting \"$0\" at line $LINENO."     # Exit from script here.
                                                # $LINENO is an internal Bash variable set to the line number it's on.
# The following lines never execute.
echo "This echo fails to echo."
exit 99
```
```
#!/bin/bash
# self-exec.sh
# Note: Set permissions on this script to 555 or 755,
#		then call it with ./self-exec.sh or sh ./self-exec.sh.

echo
echo "This line appears ONCE in the script, yet it keeps echoing."
echo "The PID of this instance of the script is still $$."

# Demonstrates that a subshell is not forked off.

echo "==================== Hit Ctl-C to exit ===================="
sleep 1
exec $0 							# Spawns another instance of this same script
									#+ that replaces the previous one.

echo "This line will never echo!"	# Why not?

exit 99								# Will not exit here!
									# Exit code will not be 99!
```
	--> exec同时还具有重新声明文件描述符的功能,例如:exec <zzz-file用文件zzz-file内容替换标准输入
	--> 在find命令中使用的-exec选项和shell内建命令exec不同,需要注意
	
	shopt
	这个命令允许在运行的过程中更改shell选项,通常出现在bash启动文件中,同样在脚本中可以使用,需要version 2以上版本的bash
```
shopt -s cdspell
# Allows minor misspelling of directory names with 'cd'
# Option -s sets, -u unsets.

cd /hpme 	# Oops! Mistyped '/home'.
pwd 		# /home
			# The shell corrected the misspelling.
```

	caller
	在一个函数中放置一个caller命令,会打印出是在第几行调用这个函数,不再函数中使用没有作用
```
#!/bin/bash
function1 ()
{
	caller 0	# Tell me about it.
}
function1
```
	--> caller命令在被source后,同样能够打印出在被source文件中的位置,类似于一个函数,这个被称作为子程序
	--> caller在debug中比较有用

### \#6 常用命令

	true
	一个返回成功(0)的退出状态码命令,不做任何其他动作
	常用与无限循环结构中: while true
	
	false
	一个返回失败(非0)退出状态码命令,不做任何其他动作
	常用场景同样为无限循环结构: while false
	
	type[cmd]
	类似于which的外部命令,type命名用于鉴定cmd命令;
	--> 不同于which,type是一个内建命令
	--> type使用-a参数时,用于鉴定关键字和内建命令,同样定位命令在系统上的唯一名称
```
bash$ type '['
[ is a shell builtin
bash$ type -a '['
[ is a shell builtin
[ is /usr/bin/[
bash$ type type
type is a shell builtin
```
	--> 在测试一个命令是否存在时,type命令非常有用,可以在判断结构中使用
```
bash$ type bogus_command &>/dev/null
bash$ echo $?
1
```
	hash [cmds]
	记录指定命令的path名--在shell的哈希表中--因此shell或者脚本不需要逐次的查找$PATH变量目录来调用这些命令
	--> 当hash命令后不接参数时,则仅仅将已经hash的命令列出来
	--> 使用-r参数是清空hash表操作
	
	bind
	内建命令bind显示或者修改readline关键子绑定
	readline:The readline library is what Bash uses for reading input in an interactive shell.
	
	help
	获取简短的shell内建命令帮助信息,这个命令whatis的副本,但是是一个内建命令
	help命令在bash v4后显示的信息详尽很多

### \#7. job控制命令

	该部分内容的部分命令需要后接任务鉴定符作为参数
	
	jobs
	列出当前在后台运行的所有job,给出job数字,没有ps有用
	--> job和process的概念很容易混淆;一些特定的内建命令,像kill,disown,wait后接job号或者进程号作为参数;但fg,bg和jobs命令则仅接收任务号(job号)作为参数
```
bash$ sleep 100 &
[1] 1384
bash $ jobs
[1]+ Running	 sleep 100 &
```
	上面的命令及其输出中,数字1是任务号(由当前shell获取的job号),1384是进程号(由系统获取),杀掉job/process,使用kill %1或者kill 1384
	
	disown
	移除shell中table内正在运行的job
```
$ jobs
[1]   Running                 sleep 1000 &
[2]   Running                 sleep 1000 &
[3]-  Running                 sleep 1000 &
[4]+  Running                 sleep 1000 &
$ disown
$ jobs
[1]   Running                 sleep 1000 &
[2]-  Running                 sleep 1000 &
[3]+  Running                 sleep 1000 &
```

	fg,bg
	fg命令将一个运行在后台的任务切换至前台. bg命令重新启动一个挂起的任务,并在后台执行
	--> 如果没有指定任务号,则默认fg和bg会作用与当前正在running状态的任务
	
	wait
	暂时挂起脚本的执行过程,直到所有的后台运行任务已经结束,或者是作为参数的任务号/进程号已经终止,返回wait后接命令的返回状态码
	--> 使用wait的场景通常为:指定的后台任务已完成,再继续执行脚本后续内容
```
#!/bin/bash
ROOT_UID=0											# Only users with $UID 0 have root privileges.
E_NOTROOT=65
E_NOPARAMS=66
if [ "$UID" -ne "$ROOT_UID" ]; then
	echo "Must be root to run this script."			# "Run along kid, it's past your bedtime."
	exit $E_NOTROOT
fi

if [ -z "$1" ]; then
	echo "Usage: `basename $0` find-string"
	exit $E_NOPARAMS
fi

echo "Updating 'locate' database..."
echo "This may take a while."
updatedb /usr &										# Must be run as root.

wait
# Don't run the rest of the script until 'updatedb' finished.
# You want the the database updated before looking up the file name.

locate $1
# Without the 'wait' command, in the worse case scenario,
#+ the script would exit while 'updatedb' was still running,
#+ leaving it as an orphan process.
exit 0
```
	--> wait也可以后接一个任务识别号作为参数,例如: wait %1或者wait $PPID
	--> 在一个脚本中,使用&符号让命令在后台执行可能会导致脚本hang死直到按下enter键,这种情况主要出现在需要输出到标准输出的命令,这种情况的出现比较让人烦;在这种命令后接wait
```
#!/bin/bash
# test.sh		  

ls -l &
echo "Done."
wait
bash$ ./test.sh
Done.
 [bozo@localhost test-scripts]$ total 1
 -rwxr-xr-x    1 bozo     bozo           34 Oct 11 15:09 test.sh
```
	将命令输出写入到文件或者/dev/null中也可以解决这个问题
	
	suspend
	改命令与ctrl+Z有相似的效果,但是这个命令是挂起shell(shell的父进程应该在合适的时候恢复该shell的运行)
	
	logout
	退出一个登录shell,后面可借参数n,指定登出shell的状态返回码
	
	times
	显示进程执行用时数据,区别于time命令;得出的信息价值有限,对于profile和基础shell脚本并不通用
	
	kill
	强制终止一个进程,通过传递合适的终止信号
```
#!/bin/bash
# self-destruct.sh
kill $$				# Script kills its own process here.
					# Recall that "$$" is the script's PID.

echo "This line will not echo."
# Instead, the shell sends a "Terminated" message to stdout.

exit 0				# Normal exit? No!

# After this script terminates prematurely,
#+ what exit status does it return?
#
# sh self-destruct.sh
# echo $?
# 143
#
# 143 = 128 + 15
#					TERM signal
```

	--> kill -l列出所有支持的信号列表(包含在文件/usr/include/asm/signal.h)
	--> kill -9是一个确认杀死,通常使用在单独使用kill命令无法杀死的场景下,有时,kill -15也能生效
	--> 僵尸进程是一个子进程已经终止,但父进程还没有被kill,无法被一个loged-on用户杀死--你无法杀死一个已经死的物件--但是init迟早会清理这些进程
	
	killall
	killall命令通过名字杀死一个运行进程,而不是通过pid
	--> 如果同时存在多个命令运行实例,则执行killall命令会将所有的这些实例全部杀死
	--> 此处的killall是指在/usr/bin/下的命令,而不是/etc/rc.d/init.d下的killall脚本
	
	command
	command命令禁用接在其后的命令的alias和function
	--> 这个命令是三个影响脚本命令执行的指令之一,其他两个分别是builtin和enable
	
	builtin
	使用命令builtin BUILTIN_COMMAND将命令BUILTIN_COMMAND当作一个shell内建命令来执行;临时禁用与调用命令同名的functions或者是系统外部命令
	
	enable
	改命令能够启用或者禁用一个shell内建命令
	例如: 命令enable -n kill禁用了shell内建命令kill,当bash继续调用kill时,调用的是外部命令kill
	--> enable的-a选项列出所有的shell内建命令,指示这些命令是否被enable
	--> -f filename选项让enable加载一个内建命令作为一个共享库模块从一个正确编译的二进制文件

### \#8. 任务识别符

|内容|含义|
|:-|:-|
|%N |任务号
|%S |引用以S开头的job |
|%?S|引用包含S的job |
|%%|当前job(最后一个在前台停止的任务或者是最后一个在后台启动的任务) |
|%+|当前job(最后一个在前台停止的任务或者是最后一个在后台启动的任务)|
|%-|上一个job|
|$!|上一个后台任务|

### \#9. 外部过滤器,程序及命令

标准的UNIX命令让shell脚本更加灵活,shell脚本的能力通过多个系统命令和shell指令的整合来实现

### \#10. 基础命令

    ls
    基础文件罗列命令.很容易理解这个命令的低调;
    用-R参数是递归的罗列出当前目录下的内容
    用-S参数是按照文件的大小来排序
    用-t参数是按照文件的修改时间来排序
    用-v参数是文件名逆序罗列(依照数值顺序)
    用-b参数是显示脱意字符
    用-i参数是显示inode信息
    --> ls返回一个非零返回值,当目标文件不存在时
    
    cat/tac
    cat是concatenate的缩写,获取一个文件的内容,显示到标准输出,当与重定向符(>或者>>)一同使用时,一般用来连接文件
    --> -n参数是在文件的前面加上行号
    --> -b参数是只显示非空行内容
    --> -v参数是显示不可打印字符
    --> -s参数是将多行空行显示为一行空行
    
    --> 在一个管道中,直接使用重定向会比cat的效率更高
```
cat filename | tr a-z A-Z
tr a-z A-Z < filename
```
    --> tac是cat的反向命令,反向输出一个文件的内容(最后一行变为第一行,依次向上)
    
    rev
    将文件的每一行内容反向输出到标准输出,注意这个通tac功能是不一样的
    
    cp
    文件复制命令
    cp file1 file2 --> 将file1复制到文件file2,如果file2存在,则覆盖文件file2内容
    --> 尤其注意-a(archive)参数的使用,可以用于将一个目录完整复制
    --> -u(update)参数避免覆盖
    --> -r/-R 递归的复制
    
    move
    文件移动命令
    命令等效与cp和rm的结合体,用来将多个文件移动到一个文件夹中,或者重命名文件夹;
    --> 当mv用在一个非交互的脚本中时,mv会使用-f(force)参数来忽略用户的输入内容
    --> 当把一个目录mv为另一个已经存在目录名称时,该目录会变成已存在目录的子目录
    
    rm
    删除文件命令
    --> -f(force)参数用于强制删除文件,即使该文件为只读,在脚本中使用,用来绕开用户输入非常有用
    --> 当一个文件以'-'开头时,使用rm删除会失败;rm将以'-'的内容作为参数使用
        a.解决方式之一是在要删除文件的前面加上'--'(选项结束标识符)
        b.另一种方式是,在文件名前加上./,表示是在当前目录下的文件
```
rm -- -badname
rm ./-badname
```
    --> 当使用-r参数时,表示从当前指定目录递归删除
        a.使用路径名称中包含变量的时候,尤其小心,当变量不存在时,有可能就变成了rm -rf /
        b.使用rm -rf *时需要注意;若命令执行时,当前工作路径不对,改命令结束后,效果将等同如rm -rf /
    
    rmdir
    删除文件夹命令
    当使用这个命令删除文件夹时,目录必须为空(包括.文件--隐藏文件)
    --> 特定场景下使用这个命令来替代rm -rf dirname,防止误删错误文件夹
    
    mkdir
    创建文件夹命令
    创建一个新的文件夹,如: mkdir -p project/programs/December; -p参数会自动创建任何必需的父目录
    
    chmod
    更改已存在文件或者文件夹的属性命令
    --> 数字方式 chmod 755 /dirname
    --> 字符方式 chmod u+rwx /filename
    
    chattr
    更改文件属性命令,这个命令与chmod的功能类似,但是使用不同的选项和语法方式,(仅在ext文件系统上使用?)
    --> chattr的一个特殊的参数'i'(immutable),当一个文件具有了i属性,表示这个文件是不可更改的,不能被修改,删除,链接(root也不行)
    --> 当一个文件具有了's'(secure)属性时,文件被删除后,之前文件占用的块将被二进制0填充
    --> 当一个文件具有了'u'(undelete)属性时,文件被删除后,文件内容仍能够获取到(状态为undeleted)
    --> 当一个文件具有了'c'(compress)属性时,当文件被写时,自动压缩到磁盘文件,当文件被读时,自动从磁盘解压读取
    --> chattr包含的文件属性不同使用ls -l显示出来
    
    ln
    为存在的文件创建链接,链接是一个文件的映射,文件的别名
    --> ln命令允许被链接的文件包含不止一个映射名
    --> ln命令还可以用作别名来使用,在脚本中充作别名(abs-p38)
    --> ln只创建文件的映射,指向文件的指针大小只有几个字节大小
    --> ln命令最常与-s选项一同使用(符号链接或者称为软链接);软链接的优势之一是可以跨越文件系统创建
    	ln -s oldfile newfile将oldfile链接一个新的文件名称newfile
    --> 更改文件被链接文件的内容时,软链接和硬链接同时能够体现更改的文件内容;不同点是
    	a.删除或者重命名被链接文件后,硬链接不受影响,源文件的存储块内容并没有发生变化;但对于指向源文件名称的软链接来说,旧文件名称已经不存在,软链接将失效
    	b.软链接的优点是可以跨越文件系统进行链接,并且,不同于硬链接,软链接还可以指向文件夹,硬链接则不行
    --> 链接的存在可以让脚本(或其他任何可执行的文件)通过不同的名称来调用(eg:/sbin/iptables -> xtables-multi),通过名称来限定脚本执行哪部分功能
```
#!/bin/bash
# hello.sh
ln -s hello.sh goodbye
HELLO_CALL=65
GOODBYE_CALL=66
if [ $0 = "./goodbye" ]; then
	echo "Good-bye!"			# Some other goodbye-type commands, as appropriate.
	exit $GOODBYE_CALL
fi

echo "Hello!"
exit $HELLO_CALL
```

	man/info
	获取帮助文档,info文档一般描述信息会比man文档更详尽
	--> man文档的书写可以通过脚本来优化(abs-709)

### \#11. 高级命令

	find 
	命令形式: -exec COMMAND \;
	为每一行find匹配的内容执行COMMAND命令,命令序列是以';'结尾
	--> 此处使用的-exec与shell自带命令exec别混淆了
	--> 注意此处不是命令分隔使用到的';',find命令序列中';'是脱意的,为了确保shell将';'按照字面意思传递给find
	--> 如果COMMAND中包含有{},则find将find匹配的路径或者文件名通过{}来替换;
```
find ~/ -name 'core*' -exec rm {} \;
find /etc -exec grep '[0-9][0-9]*[.][0-9][0-9]*[.][0-9][0-9]*[.][0-9][0-9]*' {} \;
find /home -type f -atime +5 -exec rm {} \;
```
	时间匹配: find可以通过文件的时间戳进行查找匹配
		mtime = last modification time of the target file
		ctime = last status change time (via 'chmod' or otherwise)
		atime = last access time
		(-mtime -1表示前一天被修改过的文件)
	文件匹配: find可以通过文件类型进行查找匹配
		f = regular file
		d = directory
		l = symbolic link, etc.
```
## 删除当前目录下包含特殊字符的文件
find . -name '*[+{;"\\=?~()<>&*|$ ]*' -maxdepth 0 -exec rm -f '{}' \;
## 通过inum删除文件
inum=`ls -i | grep "$1" | awk '{print $1}'`
find . -inum $inum -exec rm {} \;
```

    xargs
    一个用来传递参数给命令的过滤器,同样也是一个集合命令的工具
    --> xargs将数据流打散成足够小的块,用来给命令或者进程处理,可以将这个命令想成更加强大的后向引用
    --> 在有些场景下,命令替换报too many arguements时,用xargs却可以使用
    --> 一般场景下,xargs通过管道或者标准输入读取数据,但也可以通过一个文件的内容来获取
    --> 默认传递给xargs的命令是echo,这表示当输入管道给到xargs时,换行符和一些其他空白字符会被跳过
```
bash$ ls -l | xargs
total 0 -rw-rw-r-- 1 bozo bozo 0 Jan 29 23:58 file1 -rw-rw-r-- 1 bozo bozo 0 Jan...
```
    --> 命令ls | xargs -p -l gzip 将当前目录下的每个文件用gzip打包,每次一个,每进行一次会提示一次
    --> xargs依次序处理传递给其的参数,一次一项
    --> -n NN形式;限制每次传递给命令的参数数目,ls | xargs -n 3 echo -- 每次打印三个名字
    --> 另外一个有用的参数是-0,通常和find -print0或者grep -lZ一同使用;这个场景下允许处理的参数中包含空白字符或者引用
```
find / -type f -print0 | xargs -0 grep -liwZ GUI | xargs -0 rm -f
grep -rliwZ GUI / | xargs -0 rm -f
```
    --> -P选项允许并行执行命令,在多核机器中能够提高执行速度
```
ls *gif | xargs -t -n1 -P2 gif2png
# Converts all the gif images in current directory to png.
# Options:
# =======
# -t  Print command to stderr.
# -n1 At most 1 argument per command line.
# -P2 Run up to 2 processes simultaneously.
```
	--> 在find中使用,一对花括号的作用是作为占位符使用的
```
ls . | xargs -i -t cp ./{} $1

# -t is "verbose" (output command-line to stderr) option.
# -i is "replace strings" option.
# {} is a placeholder for output text.
# This is similar to the use of a curly-bracket pair in "find."
```

	expr
	通用表达式求值运算命令
	连接并对给出的命令选项和参数进行求值操作(各个参数之间必须用空格隔开)
	--> 可进行的操作包括有: 算数运算,比较,字符或者逻辑运算
```
expr 3 + 5
expr 5 % 3
expr 1 / 0
expr 5 \* 3                                     # 乘法运算,在expr表达式中,乘号需要脱意
y=`expr $y + 1`                                 # 变量自增,等效于 let y=y+1 and y=$(($y+1))
z=`expr substr $string $position $length`       # 获取变量string中position位置length长度的字符
```
    --> ':'运算符可以用来替代match;命令 b=`expr $a : [0-9]*`等效于 b=`expr match $a [0-9]*`

### \#12. 时间/日期命令

    date
    打印当前时间和日期信息到标准输出,这个命令的格式化输出和解析选项很有用
    --> -u选项显示UTC时间
    --> date可以用来计算不同时间之间的时间间隔
    --> date有大量的输出选项可供选择;比如%N是给出当前时间的纳秒格式,这个形式可以用来获取随机数
```
date +%N | sed -e 's/000$//' -e 's/^0//'
date --date='6 days ago'
date --date='1 year ago'
```

    zdump
    时区dump:打印指定时区的时间
```
bash$ zdump EST
EST Tue Sep 18 22:09:22 2001 EST
```

    time
    显示执行一个命令的准确时间
    --> time不同于命令times,注意
```
bash$ time ls -l /
real    0m0.004s
user    0m0.004s
sys     0m0.001s
```

    touch
    更新文件的访问/修改时间为系统/指定时间,同时具有新建一个文件的功能
    --> 当文件zzz不存在时,touch zzz会创建一个大小为0的文件zzz;这种方式用来存储日期/时间戳很有用
    --> touch命令等效于:>> newfile或者>> newfile(普通文件)
    --> 可以结合cp -a一起使用,使用touch更新不想被覆盖的文件时间戳,再用cp -u
    
    at
    at任务控制命令在指定的时间执行一系列指定的命令;命令类似于cron,at命令很适合只执行一次的命令
    --> 使用-f选项或者<输入重定向,at充文件中读取命令;文件应该是一个可执行的shell脚本,同时是非交互式脚本
    --> 需要执行较多内容时,可以结合run-parts命令来实现
```
bash$ at 2:30 am Friday < at-jobs.list
job 2 at 2000-10-27 02:30
```

    batch
    batch命令类似于at命令,但要求在系统负载低于0.8时执行;类似于at,可以接-f选项
    
    cal
    打印简化版的日历到标准输出,可接不同选项,显示不同年份和月份的日历
    
    sleep
    这个是shell形势下的wait循环;命令暂停指定秒数的时间,什么都不做
    --> 对于后台运行的命令或者定时进程很有用,定时检查事件
    --> sleep命令默认情况下是使用秒,但也可以用分钟,小时或者天来定时
    --> watch命令比sleep命令更适合周期性检查命令执行情况
    
    usleep
    类似与sleep,但默认时间为微秒,用在对时间更加精准或查询更加频繁的场景下
    
    hwclock/clock
    命令hwclock获取或者调整机器的硬件时间,某些选项需要root权限才能使用
    --> 在rhel6中,启动时,/etc/rc.d/rc.sysinit启动文件通过hwclock获取硬件时间来设置系统时间
    --> clock是hwclock的同义词

### \#13. 文本处理命令

    sort
    文件归类工具,通常用在管道后过滤使用
    --> 命令用来正序,逆序或者指定位置关键字/字符位置来处理文本流或者文件
    --> 使用-m选项,合入预处理过的文件
    --> 可以查看info文档来获取sort的使用场景
    
    tsort
    拓扑排序,读取以空格分隔的字符串对，并根据输入模式进行排序;通常情况下tsort命令排序的结果与sort排序结果有较大差别
    
    uniq
    该命令移除文件中重复出现的行,通常会结合sort和管道一同使用
```
cat list-1 list-2 list-3 | sort | uniq > final.list
```
    --> 使用-c选项,在输出结果中显示重复出现的次数
    --> sort INPUTFILE | uniq -c | sort -nr　命令打印出在INPUTFILE中出现的频次信息,使用场景为分析log文件或者字典列表等
```
sed -e 's/\.//g' -e 's/\,//g' -e 's/ //g' "$1" | tr 'A-Z' 'a-z' | sort | uniq -c | sort -nr
```

    expand/unexpand
    命令expand用来将tab展开为空格,通常与管道结合使用
    命令unexpand将空格转换为tab,是expand的反转命令
    
    cut
    展开文件指定区域的命令.命令类似于在awk结构中的print $N,但相比较之下,限制更多.在脚本中使用cut命令会比使用awk更简单.cut重要的选项-f和-d
```
# 使用cut来获取挂载文件系统列表
cut -d ' ' -f1,2 /etc/mtab
# 使用cut列出OS和内核版本
uname -a | cut -d" " -f1,3,11,12
# 使用cut来解析email信息头部
bash$ grep '^Subject:' read-messages | cut -c10-80
Re: Linux suitable for mission-critical apps?
MAKE MILLIONS WORKING AT HOME!!!
Spam complaint
Re: Spam complaint
```
    --> 甚至可以指定换行符作为分隔符
```
bash$ cut -d'
' -f3,7,19 testfile
This is line 3 of testfile.
This is line 7 of testfile.
This is line 19 of testfile.
```
    --> cut -d ' ' -f2,3 filename的结果等效于awk -F'[ ]' '{ print $2, $3 }' filename
    
    paste
    用于将多个文件合成为一个文件,将合成的结果输出到标准输出
    
    join
    这是一个特殊形式的paste命令;这个强大的程序允许按照一定的关联关系来合并两个文件内容,这实际上已经是一个简化版的关系数据库形式了
    --> join命令只对两个文件进行操作，但只粘贴那些带有公共标记字段（通常是数字标签）的行，并将结果写入stdout
    --> 要加入的文件应根据标记字段进行排序，以使匹配正常工作
```
File: 1.data
100200300Shoes
Laces
Socks

File: 2.data
100200300$40.00
$1.00
$2.00

bash$ join 1.data 2.data
File: 1.data 2.data
100 Shoes $40.00
200 Laces $1.00
300 Socks $2.00
# 标识字段只出现一次
```

    head
    打印一个文件的启示内容到标准输出,默认行数是10,但允许设置不同数字
    --> head有多个不同的参数可以使用,如-c(字符数),-n(行数)
    
    tail
    打印一个文件的尾部到标准输出,默认行数是10,但该数字可以通过接-n选项更改;通常用来追踪一个文件内容的变化
    --> tail -f等效于tailf,动态实时的显示文件内容
    --> 显示某个文件的指定行内容,其中一种方式: head -n 8 database.txt | tail -n 1 l
    --> 使用新的 tail -n $LINES filename 代替旧有形式tail -$LINES filename
    
    grep
    一个多功能的搜索工具,可以使用正则表达式;命令形式grep pattern [file...]
    --> -i选项,忽略大小写
    --> -w选项,仅匹配整个单词
    --> -l选项,仅显示匹配相关内容的文件名
    --> -r选项,递归搜索
    --> -n选项,显示匹配到的内容所在行,并将行数显示在匹配内容的前面
    --> -v选项,不显示匹配到的内容
    --> -c选项,显示匹配到的次数
    --> --color选项,将匹配到的内容上色显示
    --> -o选项,仅打印匹配到的内容,不显示一整行
    --> -q选项,不打印任何内容,在判断结构中使用很方便
    --> 当仅搜索一个文件时,在输出结果中要将文件名一同打印,可以后接/dev/null作为第二个文件名
```
bash$ grep Linux osinfo.txt /dev/null
```
    --> grep匹配到相关内容后,返回值为0,这样可以结合判断结构来使用
    --> 使用sed命令可以仿真grep的行为: sed -n /"$1"/p $file
    --> 怎样让grep匹配两个不同的匹配模式,可以使用方式grep pattern1 | grep pattern2来实现
    --> egrep等效于grep -E,允许使用扩展正则表达式来进行匹配,形式更加灵活
    --> fgrep等效于grep -F,逐字匹配(不使用正则表达式),速度会更快一些
    --> agrep(近似匹配/模糊匹配),扩展grep的能力,使其能够进行模糊匹配
    --> 匹配压缩文件内容时,使用zgrep/zegrep/zfgrep,这些命令同样能用在普通文件,虽然速度慢一些,但对于包含多种文件类型(普通文件/压缩文件)的场景下操作则更方便
    --> 匹配bzip文件时使用bzgrep命令
    
    look
    look命令工作形式类似于grep,但是是基于字典(一个已排序的单词列表)进行查询;默认情况下,look在/usr/dict/words下进行匹配,但可以指定进行匹配的字典
```
file=words.database                  # Data file from which to read words to test.
while [ "$word" != end ]; do         # Last word in data file.
    read word
    look $word > /dev/null
    lookup=$?
    if [ "$lookup" -eq 0 ]; then
        echo "\"$word\" is valid."
    else
        echo "\"$word\" is invalid."
    fi
done <"$file"
```

    sed/awk
    脚本语言,非常适合用来处理文本文件和标准输出的内容
    sed
    非交互式的流编辑器,在batch(不需要用户介入的情况下运行一组命令)模式下允许使用很多ex命令,在脚本中使用很广泛
    awk
    可编程式文件内容提取及格式化程序,适合用来操作或展开格式化的文本文件的列,语法格式类似于C语言
    
    wc
    计数程序
    --> -w:单词数;-l:行数;-c:字节数;-m:字符数;-L:给出最长行的长度
    
    tr
    字符转化过滤器
    --> 视情况决定是否需要使用引用或者括号,引号引用可以防止shell将属于tr的特殊字符先行处理了,括号必须被引起来,防止直接被展开
```
tr "A-Z" "*" <filename 
tr A-Z \* <filename
```
    --> 接上-d选项,用于删除一系列的指定字符
    --> --squeeze-repeats(-s)选项用来删除连续的字符(保留第一个),这个选项用来移除多余的空白字符很有用
    --> -c(补足)选项将未匹配的内容用指定的字符替代
```
bash$ echo "acfdeb123" | tr -c b-d +
+c+d+b++++
bash$ echo "abcd2ef1" | tr '[:alpha:]' -
----2--1
```

    fold
    将输入的行折叠成指定宽度的过滤器
    --> 使用-s选项用来以空格作为隔断符,防止直接截断字符
```
b=`ls /usr/local/bin`
$ echo $b | fold - -s -w 40
cnpm compile cops-cli gitbook pacvim
ptyping qr-filetransfer runenpass sle
study s-tui t termtosvg tget tiv
# 等效于echo $b | fmt -w $WIDTH
```

    fmt
    简单的格式化工具,用于将长行分割为多行
    
    column
    列格式化工具,将列类型的文本输出转换成格式友好的打印形式,在合适的位置插入tab
    --> 对于文件中存在多列内容,但未对齐的格式,使用column -t处理会很方便
    
    colrm
    列删除处理器,该命令删除文件中的列内容并直接写该文件
    --> colrm 2 4 <filename 删除文件中的第二到第四列内容
    --> 上列命令使用时,若文件中包含有tab之类的不可打印字符,则可能导致无法预估的结果,考虑使用expand/unexpand命令处理之后再管道传输给colrm
    
    nl
    打印文件内容并附上行号
    --> 打印的是非空行内容
    --> nl的操作非常类似于cat -b,同样是不打印非空行
    
    cpio
    这个特定的归档复制命令已经很少用了,基本已经被tar/gzip取代;但在某些场景下,还是能够使用
    --> 指定块大小进行复制,速度会比tar更快
    find "$source" -depth | cpio -admvp "$destination"
    
    rpm2cpio
    这个命令从rpm包中解压一个cpio文件出来
    
    gzip
    结合-c选项使用时,将gzip的输出写入到标准输出,结合管道使用非常有用
    
    zcat/bzcat
    可以用来查看gzip/bzip文件中的内容,相当与bzip/gzip文件下的cat
    
    readlink
    获取符号链接指向的文件
    
    strings
    用来获取二进制文件或其他数据文件中的可打印字符
    
    diff
    逐行打印两个对比文件的不同内容
    --> 使用--side-by-side选项,每个文件内容显示一列,相较原来的形式更方便对比查看
    --> 使用-c和-u选项可以让输出结果更适合查看
    --> 可以在判断结构中使用diff命令,当比较的两个文件同一的时候,命令返回值为0,当比较的两个文件不一样时,返回值非零
    
    patch
    一个非常灵活的版本记录命令,结合diff使用比较常见,可以给出一个由diff命令创建的差异文件
    
    diff3
    diff命令的升级版命令,可以一次比较三个文件,正常执行后命令返回值为0,但改命令不会打印文件差异内容出来
    
    split/csplit
    用于将一个文件切片为小块的工具,使用场景一般为文件太大,需要切片后邮件发送或者拷贝到移动磁盘
    csplit命令通过文件内容来进行分片
    
    sum,cksum,md5sum,sha1sum
    以上命令是用于创建校验和的工具,校验和是通过对一个文件内容进行数学计算得出的字符串,用于检查文件的完整性
    
    openssl
    可以用于加密使用,结合tar一同使用,可以方便的加密相关目录和文件
    
    shred
    用随机字符多次重写文件,可用在安全要求高的场景下
    
    mktemp
    创建一个拥有唯一名称的临时文件,不使用其他参数是,在/tmp目录下创建一个大小为0的文件
    --> tempfile=`mktemp $PREFIX.XXXXXX` 指定创建的新文件包含有多少个随机字符
    
    ptx
    ptx[targetfile]命令输出目标文件的排列索引（交叉引用列表）。如有必要，可以在管道中进一步过滤和格式化。
    
    ipcalc
    用于换算和查看ip相关的内容
    
    traceroute
    通过发送的包追踪路由
    
    sx,rx
    命令集通过xmodem协议与远端服务器传输文件
    
    sz,rz
    命令集通过zmodem协议与远端服务器传输文件,zmodem的速度相对xmodem更快
    
    ssh
    --> 在循环中使用ssh时,可能出现不可预期的情况,可以后接-f或者-n选项来避免
    
    tput
    --> 初始化终端和/或从terminfo数据中获取有关它的信息。各种选项允许某些终端操作：tput clear等于clear；tput reset等于reset。
    --> tput可以用来对终端进行操作,更改字符显示方式等
    
    reset
    重置终端参数并且清屏
    
    script
    记录键盘敲击记录,执行该命令后,会在当前目录下生成一个文件,用于记录后面敲击的键盘记录
    
    factor
    后接一个数字,将该数字的各个因数打印出来
    
    bc
    bash无法进行浮点数计算,且缺少一些重要的运算功能,bc可以满足部分需求
    --> bc可以用在脚本中,用来对变量进行计算获值variable=$(echo "OPTIONS; OPERATIONS" | bc)
    --> 另外一种形式是结合here document的方式来作为输入
```
<< EOF
18.33 * 19.78
EOF
`
```

    awk
    另外一种进行浮点数运算的方式是使用awk命令
```
AWKSCRIPT=' { printf( "%3.7f\n", sqrt($1*$1 + $2*$2) ) } '
#           command(s) / parameters passed to awk

# Now, pipe the parameters to awk.
echo -n "Hypotenuse of $1 and $2 = "
echo $1 $2 | awk "$AWKSCRIPT"
```

    jot,seq
    生成一个整数序列,用户可以自定义步长和分隔符
    --> seq -s : 5 指定分隔符为:,默认情况下是换行符
    --> jot和seq都可以用在for循环中
    
    run-parts
    run-parts命令会执行目标目录下的所有脚本,默认依照ascii字母顺序执行,当然,脚本需要有执行权限
    
    yes
    yes默认的动作为返回y及换行符到标准输出,需要使用ctrl+C来终止
    --> 使用yes string,则后面会不断重复出现string
    --> 使用场景: yes | rm -r dirname
    
    tee
    类似与重定向,但与重定向不同
                               (redirection)
                              |----> to file
                              |
    ==========================|====================
    command ---> command ---> |tee ---> command ---> ---> output of pipe
    ===============================================
    
    mkfifo
    命令创建一个命名管道,一个临时的first-in-first-out用于在不同的进程之间传递数据,一般情况下,一个进程向FIFO中写数据,另一个进程则从FIFO中读取数据
```
(cut -d' ' -f1 | tr "a-z" "A-Z") >pipe2 <pipe1 &
ls -l | tr -s ' ' | cut -d' ' -f3,9- | tee pipe1 | cut -d' ' -f2 | paste - pipe2
rm -f pipe1
rm -f pipe2
```

### \#14. 其他命令
    groups
    显示当前用户属于哪些组
    
    lid
    命令将显示给出的用户名所属的groups列表
    
    logname
    显示登录当前终端登录系统的用户名称,即使su之后,改命令仍显示为源用户
    
    ac
    显示用户登录的时长
    
    newgrp
    更改当前用户的组id而不需要登出系统
    
    tty
    显示当前终端所在的文件名称
    
    stty
    显示或者是更改终端的设置,改命令较复杂,在脚本中使用时,可以控制终端的行为以及其输出内容的形式
    
    setterm
    设置指定的终端属性,这条命令会更改输出到标准输出结果的显示情况
    
    getty/agetty
    使用getty/agetty来初始化终端进程,这些命令不能在脚本中使用,脚本中可使用的是stty
    
    lastcomm
    显示上一条执行命令的一些相关信息,内容是存储在/var/accounut/pacct文件中的
    
    strace(system trace)
    用于诊断和debug系统调用的工具,该命令和ltrace可用来查找一个程序运行失败的原因
    
    itrace(library trace)
    用于诊断和debug库调用的工具
    
    nc(netcat)
    nc工具集是一个用来连接和监听TCP/UDP端口的工具
    
    dmesg
    打印所有的系统启动日志信息到标准输出
    
    size
    size /path/to/binary 会给出一个二进制文件的各个段大小,对于程序员来说,需要使用
    
    logger
    追加用户自定义的内容到/var/log/messages中去,普通用户也可以使用
    --> 该命令可以用来在脚本中增加debug信息到日志中
    
    logrotate
    该工具用于管理系统的日志文件,轮转,压缩,删除或者email发送等,一般是结合cron一起使用来实现日志管理的
    
    fuser
    获取当前正在访问给定文件,文件集,或者目录的进程(通过进程号显示)
    --> 同-k选项一同使用时,杀掉这些进程,在插拔可移动设备的场景下使用较多
    --> 同-n选项一同使用时,获取到当前在访问指定端口的进程,与nmap一同搭配使用非常有用
```
# nmap localhost 
PORT    STATE   SERVICE
25/tcp  open    smtp

# fuser -un tcp 25
25/tcp: 2095(root)
```

    nmap
    network mapper and port scanner,网络映射和端口扫描,查看指定主机开放的端口
    
    sync
    执行命令后,强制当前环境下,将buffer中的数据写入到磁盘中
    
    losetup
    创建或者配置loop设备
    
    mkswap
    创建一个swap分区或者文件,swap区域启动需要使用swapon命令
    
    swapon/swapoff
    启用/关闭swap设备
    
    dumpe2fs
    打印显示详细的文件系统相关信息,必须由root用户调用
    
    hdparm
    显示/更改磁盘参数,必须由root用户使用,错误使用很危险
    
    badblocks
    检查一个存储设备的坏块,在一个刚格式化后的设备或者备份文件时使用
    
    lsusb,usbmodules
    显示所有的usb设备信息
    
    lspci
    列出当前使用到的pci总线
    
    chroot
    顾名思义,更改当前的根目录位置
    
    lockfile
    属于procmail包的一部分,他创建一个lock 文件,用于作为一个标记文件存在,控制文件,设备或者资源的可获取性
    
    flock
    命令给文件设置一个锁信息的公告,当命令执行完成之后,其他命令或者进程才能操作刚才的文件
```
flock $0 cat $0 > lockfile__$0
# Set a lock on the script the above line appears in,
#+ while listing the script to stdout.
```
    --> 与lockfile命令不同的是,flock命令不会自动创建一个lock文件
    
    mknod
    创建一个块设备或者字符设备(当在系统上新增了一个新的设备之后可能有必要这样做),MAKEDEV工具比mknod更容易使用,且具备相应的功能
    
    MAKEDEV
    用于创建设备文件的命令,必须由root用户来执行,文件保存在/dev下,是高级版的mknod
    
    tmpwatch
    自动删除有一段时间没有被访问过的文件,通常结合cron一同使用,用于删除log文件
    
    dump/restore
    dump命令是设计用于备份文件系统的命令,命令读取磁盘分区的裸数据并写一个二进制文件
    
    ulimit
    设置一个更高的系统使用极值
    
    quota
    显示用户或者组的磁盘配额信息
    
    setquota
    设置用户或者组的磁盘配额
    
    depmod
    生成模块依赖文件,通常会被启动脚本调用
    
    ldd
    显示一个可执行文件需要使用到的共享库依赖
    
    watch
    以特定间隔运行一个命令

## 特殊内容

### \#1. here document

    << 可以结合vi一同使用,如下所示:
```
# Insert 2 lines in file, then save.
#--------Begin here document-----------#
vi $TARGETFILE <<x23LimitStringx23
i
This is line 1 of the example file.
This is line 2 of the example file.
^[
ZZ
x23LimitStringx23
#----------End here document-----------#
```
    可以使用vi +n的形式指定文件打开后在第几行
