---
layout: post
title:  "shell语言基础"
date:   2019-8-27
categories: 
excerpt: 
---

* content
{:toc}

# **shell语言**

"#!" 是一个约定的标记，它告诉系统这个脚本需要什么解释器来执行，即使用哪一种
Shell。一般都是用#!/bin/bash

## **变量**

### **普通变量**

命名只能使用英文字母，数字和下划线，首个字符不能以数字开头。

中间不能有空格，可以使用下划线（_）。

不能使用标点符号。

不能使用bash里的关键字（可用help命令查看保留关键字）。

变量名和等号之间不能有空格，使用变量时需要加”$”符号。花括号的意义是方便区分变量名称。

	#!/bin/bash
	a="hello world"
	echo "$ab"
	echo "${a}b"

	第一个输出会被认为空

	hello worldb

### **只读变量**

用readonly可以将变量定义为只读变量，只读变量的值不能被改变。

	#!/bin/bash
	a="hello world"
	readonly a

### **删除变量**

使用 unset 命令可以删除变量。

	unset a

### **字符串**

#### **引号**

单引号：

单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；

单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。

双引号：

双引号里可以有变量

双引号里可以出现转义字符

	#!/bin/bash
	a="world"
	b='hello $a'
	c="hello $a"
	echo $b
	echo $c
	
	
	
	hello $a
	hello world

#### **拼接字符串**

	#!/bin/bash
	a=" world"
	#用单引号拼接**
	b='hello '$a''
	c='hello $a'
	echo $b $c
	#用双引号拼接**
	b="hello "$a""
	c="hello $a"
	echo $b $c

	
	
	hello world hello $a
	hello world hello world

#### **获取字符串长度**

	#!/bin/bash
	a="hello world"
	b=${#a}
	echo $b

		
	11

#### **提取子字符串**

	#!/bin/bash
	a="hello world"
	b=${a:1:8}
	echo $b

		
	ello wor

#### **查找子字符串**

脚本中 ` 是反引号，在键盘ESC的下面第一个。

	#!/bin/bash
	a="hello world"
	b=`expr index "${a}" w`
	echo $b

	7

## **数组**

格式：a=(value1 ... valuen)

读取数组：

	#!/bin/bash
	a=("hello" 1 2 3)
	echo ${a[0]} ${a[1]}

	hello 1

获取数组中的所有元素：

	#!/bin/bash
	a=("hello" 1 2 3)
	echo ${a[*]}
	echo ${a[@]}


	hello 1 2 3
	hello 1 2 3

获取数组的长度：

	#!/bin/bash
	a=("hello" 1 2 3)
	echo ${#a[*]}
	echo ${#a[@]}

	4
	4

## **运算符**

bash不支持数字运算，所以一般用别的命令进行操作，expr是一款表达式计算工具，使用它能完成表达式的求值操作。

### **算术运算符**

	#!/bin/bash
	a=10
	b=20
	val=`expr $a + $b`
	echo "a + b : $val"
	val=`expr $a - $b`
	echo "a - b : $val"
	val=`expr $a * $b`
	echo "a * b : $val"
	val=`expr $b / $a`
	echo "b / a : $val"
	val=`expr $b % $a`
	echo "b % a : $val"

	
	a + b : 30
	a - b : -10
	a * b : 200
	b / a : 2
	b % a : 0

注意表达式和运算符之间要有空格，用*的时候前面一定要加转义符号。

### **关系运算符**

	| -eq | 检测两个数是否相等，相等返回 true。                   | [ $a -eq $b ] 返回 false。 |
	|------|-------------------------------------------------------|------------------------------|
	| -ne | 检测两个数是否不相等，不相等返回 true。               | [ $a -ne $b ] 返回 true。  |
	| -gt | 检测左边的数是否大于右边的，如果是，则返回 true。     | [ $a -gt $b ] 返回 false。 |
	| -lt | 检测左边的数是否小于右边的，如果是，则返回 true。     | [ $a -lt $b ] 返回 true。  |
	| -ge | 检测左边的数是否大于等于右边的，如果是，则返回 true。 | [ $a -ge $b ] 返回 false。 |
	| -le | 检测左边的数是否小于等于右边的，如果是，则返回 true。 | [ $a -le $b ] 返回 true。  |

### **布尔运算符**

	| !   | 非运算，表达式为 true 则返回 false，否则返回 true。 | [ ! false ] 返回 true。                    |
	|-----|-----------------------------------------------------|--------------------------------------------|
	| -o | 或运算，有一个表达式为 true 则返回 true。           | [ $a -lt 20 -o $b -gt 100 ] 返回 true。  |
	| -a | 与运算，两个表达式都为 true 才返回 true。           | [ $a -lt 20 -a $b -gt 100 ] 返回 false。 |

### **逻辑运算符**

	| &&   | 逻辑的 AND | [[ $a -lt 100 && $b -gt 100 ]] 返回 false  |
	|------|------------|----------------------------------------------|
	| || | 逻辑的 OR  | [[ $a -lt 100 || $b -gt 100 ]] 返回 true |

### **字符串运算符**

	| =   | 检测两个字符串是否相等，相等返回 true。   | [ $a = $b ] 返回 false。 |
	|-----|-------------------------------------------|----------------------------|
	| !=  | 检测两个字符串是否相等，不相等返回 true。 | [ $a != $b ] 返回 true。 |
	| -z | 检测字符串长度是否为0，为0返回 true。     | [ -z $a ] 返回 false。    |
	| -n | 检测字符串长度是否为0，不为0返回 true。   | [ -n "$a" ] 返回 true。   |
	| $  | 检测字符串是否为空，不为空返回 true。     | [ $a ] 返回 true。        |

### **文件测试运算符**

	| -b file | 检测文件是否是块设备文件，如果是，则返回 true。                             | [ -b $file ] 返回 false。 |
	|----------|-----------------------------------------------------------------------------|----------------------------|
	| -c file | 检测文件是否是字符设备文件，如果是，则返回 true。                           | [ -c $file ] 返回 false。 |
	| -d file | 检测文件是否是目录，如果是，则返回 true。                                   | [ -d $file ] 返回 false。 |
	| -f file | 检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。 | [ -f $file ] 返回 true。  |
	| -g file | 检测文件是否设置了 SGID 位，如果是，则返回 true。                           | [ -g $file ] 返回 false。 |
	| -k file | 检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。                 | [ -k $file ] 返回 false。 |
	| -p file | 检测文件是否是有名管道，如果是，则返回 true。                               | [ -p $file ] 返回 false。 |
	| -u file | 检测文件是否设置了 SUID 位，如果是，则返回 true。                           | [ -u $file ] 返回 false。 |
	| -r file | 检测文件是否可读，如果是，则返回 true。                                     | [ -r $file ] 返回 true。  |
	| -w file | 检测文件是否可写，如果是，则返回 true。                                     | [ -w $file ] 返回 true。  |
	| -x file | 检测文件是否可执行，如果是，则返回 true。                                   | [ -x $file ] 返回 true。  |
	| -s file | 检测文件是否为空（文件大小是否大于0），不为空返回 true。                    | [ -s $file ] 返回 true。  |
	| -e file | 检测文件（包括目录）是否存在，如果是，则返回 true。                         | [ -e $file ] 返回 true。  |

## **函数**

格式：

	[ function ] funname [()]

	{

		action;

		[return int;]

	}

	
	funWithParam(){
	echo "第一个参数为 $1 !"
	echo "第二个参数为 $2 !"
	echo "第十个参数为 $10 !"
	echo "第十个参数为 ${10} !"
	echo "第十一个参数为 ${11} !"
	echo "参数总数有 $# 个!"
	echo "作为一个字符串输出所有参数 $* !"
	}
	funWithParam 1 2 3 4 5 6 7 8 9 34 73

		
		
	第一个参数为 1 !
	第二个参数为 2 !
	第十个参数为 10 !
	第十个参数为 34 !
	第十一个参数为 73 !
	参数总数有 11 个!
	作为一个字符串输出所有参数 1 2 3 4 5 6 7 8 9 34 73 !

注意第十个参数以后必须要用${10}表示。

带返回值的：

	#!/bin/bash
	funWithParam(){
	return `expr 1 + 1`
	}
	funWithParam
	echo $?
	echo $?

	2  
	0

$?为上一条命令的执行结果，所以紧跟函数调用后的$?就是返回值，中间不能穿插别的代码，否则$?将不会再是返回值。

## **条件/循环语句**

### **if语句**

	#!/bin/bash
	a=10
	b=20
	if [ $a == $b ]
	then
		echo "a = b"
	elif [ $a -gt $b ]
		then
	echo "a > b"
	else
		echo "a < b"
	fi

	
	a < b

注意必须以fi结尾。

### **for语句**

	#!/bin/bash
	a=(1 2 3 4 5 6)
	for i in ${a[*]}
	do
		echo $i
	done
	for i in 1 2 3 4 5 6
	do
		echo $i
	done


	1
	2
	3
	4
	5
	6
	1
	2
	3
	4
	5
	6

### **while语句**

	#!/bin/bash
	a=5
	while [ $a -gt 1 ]
	do
		echo $a
		let "a--"
	done

	5  
	4  
	3  
	2

### **until 循环**

	#!/bin/bash
	a=5
	until [ ! $a -lt 10 ]
	do
		echo $a
		a=`expr $a + 1`
	done

	5
	6
	7
	8
	9

### **case语句**

格式：

格式：
case 变量 in 

值1 )

    执行动作1
	
    ;;
	
值2 )
	
    执行动作2
	
    ;;
	
值3 )
	
    执行动作3
	
    ;;
	
....
	
* )

    如果变量的值都不是以上的值，则执行此程序
	
    ;;
	
esac


	#!/bin/bash
	a=2
	case $a in
		1)  echo '你选择了 1'
		;;
		2)  echo '你选择了 2'
		;;
		3)  echo '你选择了 3'
		;;
		4)  echo '你选择了 4'
		;;
		*)  echo '你没有输入 1 到 4 之间的数字'
		;;
	esac

		
	你选择了 2


## **其他命令**

### **echo**

作为字符串的输出语句。

	#!/bin/bash
	a=10
	echo "It is a test"
	echo "\"It is a test\""
	echo -e "OK! \n" # -e 开启转义
	echo "OK! \n"
	echo -e "OK! \c" # -e 开启转义 \c 不换行
	echo "OK! \c"
	echo "$a"
	echo `date`

	
	
	It is a test
	"It is a test"
	OK! 

	OK! \n
	OK! OK! \c
	10
	Tue Aug 27 11:51:54 UTC 2019


### **printf**

	#!/bin/bash
	a=10
	b=20
	printf "Hello, Shell\n"
	printf "%-10d %-10d\n" $a $b #左对齐
	printf "%10d %10d\n" $a $b #右对齐

	
	
	Hello, Shell
	10         20        
			10         20


### **test**

test 命令用于检查某个条件是否成立，它可以进行数值、字符和文件三个方面的测试。

	num1=100
	num2=100
	if test $[num1] -eq $[num2]
	then
		echo '两个数相等！'
	else
		echo '两个数不相等！'
	fi


## **输入/输出重定向**

	| command > file   | 将输出重定向到 file。                              |
	|-------------------|----------------------------------------------------|
	| command < file   | 将输入重定向到 file。                              |
	| command >> file | 将输出以追加的方式重定向到 file。                  |
	| n > file         | 将文件描述符为 n 的文件重定向到 file。             |
	| n >> file       | 将文件描述符为 n 的文件以追加的方式重定向到 file。 |
	| n >& m           | 将输出文件 m 和 n 合并。                           |
	| n <& m           | 将输入文件 m 和 n 合并。                           |
	| << tag          | 将开始标记 tag 和结束标记 tag 之间的内容作为输入。 |

文件描述符 0 通常是标准输入（STDIN），1 是标准输出（STDOUT），2是标准错误输出（STDERR）

	#!/bin/bash
	echo `ls`
	cat text.sh
	ls script.sh 1> text.sh
	cat text.sh

	script.sh
	cat: text.sh: No such file or directory
	script.sh


相当于把script.sh重定位给了text.sh

经常看到很多shell脚本里面会有一句 2> /dev/null。/dev/null是一个特殊的设备文件，这个文件接收到任何数据都会被丢弃。将命令的输出重定向到它，会起到"禁止输出"的效果。

## **参考链接**

https://www.runoob.com/linux/linux-shell.html
