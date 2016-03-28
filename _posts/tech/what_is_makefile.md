### 什么是 Makefile

Makefile 带来的好处是就是 “自动化编译”，通过执行 ```make``` 就是完成整个工程的自动编译。 make 命令执行的时候，需要 Makefile 文件，告诉 make 命令怎样进行编译和链接程序。

### Makefile 的规则和流程

Makefile 的规则大致如下：

		  target ... : prerequisites ...
            command
            ...
            ...
            
* target 就是一个目标文件， 可以是 object file， 也可以是执行文件， 还可以是一个标签（标签是一种伪目标）。
* prerequisites 就是生成那个 target 所需要的文件或者目标。
* command 就是 make 需要执行的命令。（可以是任意的 shell 命令）

一个简单实例：

	    edit : main.o kbd.o command.o display.o /
           insert.o search.o files.o utils.o
            cc -o edit main.o kbd.o command.o display.o /
                       insert.o search.o files.o utils.o

    main.o : main.c defs.h
            cc -c main.c
    kbd.o : kbd.c defs.h command.h
            cc -c kbd.c
    command.o : command.c defs.h command.h
            cc -c command.c
    display.o : display.c defs.h buffer.h
            cc -c display.c
    insert.o : insert.c defs.h buffer.h
            cc -c insert.c
    search.o : search.c defs.h buffer.h
            cc -c search.c
    files.o : files.c defs.h buffer.h command.h
            cc -c files.c
    utils.o : utils.c defs.h
            cc -c utils.c
    clean :
            rm edit main.o kbd.o command.o display.o /
               insert.o search.o files.o utils.o
               
在这个 Makefile 中 target 包含执行文件 edit 和中间目标文件 *.o， 依赖文件（perrequisites）就是冒号后面那些 .c 文件和 .h 文件。每个 .o 文件都有一组依赖的文件， 而这些 .o 文件又是执行文件 esit 的依赖文件。依赖关系的实质就是说明目标文件是由那些文件生成的。
在定义后依赖文件后，后续的一行定义了如何生成目标文件的系统命令， 一定要以 Tab 作为开头。 make 会表 tagets文件和 prerequisites 文件的修改日期， 如果 prerequisites 文件的日期要比 targets 文件要新，或者 target 不存在，那么 make 就会执行定义的命令生成目标。

clean 不是以文件，它是一个标签或者是说一个动作的名字， 冒号后面没有内容，说明它没有依赖文件， make 也就不会自动触发其定义的命令，如果要执行命令得指定 label 的名字， 即为 ``` make clean ```。

在输入 make 命令之后的流程如下：

1. make 会在当前目录寻找 Makefile 文件
2. 如果找到，他会找文件中的第一个目标文件，作为最终目标文件。
3. 如果edit文件不存在，或是edit所依赖的后面的 .o 文件的文件修改时间要比edit这个文件新，那么，他就会执行后面所定义的命令来生成edit这个文件。
4. 如果edit所依赖的.o文件也存在，那么make会在当前文件中找目标为.o文件的依赖性，如果找到则再根据那一个规则生成.o文件。（这有点像一个堆栈的过程）
5. 当然，你的C文件和H文件是存在的啦，于是make会生成 .o 文件，然后再用 .o 文件生命make的终极任务，也就是执行文件edit了。

### makefile 语法

#### 定义变量
	
	 objects = main.o kbd.o command.o display.o /
              insert.o search.o files.o utils.o
则：
	
	edit : main.o kbd.o command.o display.o /
           insert.o search.o files.o utils.o
    	cc -o edit main.o kbd.o command.o display.o /
       			   insert.o search.o files.o utils.o
 可以写成: 
 	   
	edit : $(objects)
    	cc -o edit $(objects)
#### 自动推导
只要make看到一个[.o]文件，它就会自动的把[.c]文件加在依赖关系中，如果make找到一个whatever.o，那么whatever.c， 就会是whatever.o的依赖文件。并且 cc -c whatever.c 也会被推导出来，于是，我们的makefile再也不用写得这么复杂。

    main.o : main.c defs.h
            cc -c main.c
可以写成:
	
	main.o : defs.h
#### 清空目标文件的规则

	clean:
		rm edit $(objects)
		
应该写成：

	.PHONY: clean
	clean:
		-rm edit $(objects)

.PHONY 意思是表示 clean 是“伪目标”， 在 rm 命令前面加一个减号的意思就是也许某些文件出现问题，但不要管，继续做后面的事。 另外 clean 应该放在 Makefile 的最后。

#### 引用其他的 Makefile

	include <filename>;
	
filename可以是当前操作系统Shell的文件模式（可以保含路径和通配符）。在include前面可以有一些空字符，但是绝不能是[Tab]键开始。include和<filename>;可以用一个或多个空格隔开。如果文件都没有指定绝对路径或是相对路径的话，make会在当前目录下首先寻找，如果当前目录下没有找到，那么，make还会在下面的几个目录下找：

    1、如果make执行时，有“-I”或“--include-dir”参数，那么make就会在这个参数所指定的目录下去寻找。
    2、如果目录<prefix>;/include（一般是：/usr/local/bin或/usr/include）存在的话，make也会去找。

如果有文件没有找到的话，make会生成一条警告信息，但不会马上出现致命错误。它会继续载入其它的文件，一旦完成makefile的读取，make会再 重试这些没有找到，或是不能读取的文件，如果还是不行，make才会出现一条致命信息。如果你想让make不理那些无法读取的文件，而继续执行，你可以在 include前加一个减号“-”。如：````-include <filename>;```，其表示，无论include过程中出现什么错误，都不要报错继续执行。和其它版本make兼容的相关命令是sinclude，其作用和这一个是一样的。

#### 通配符

make支持三各通配符：“*”，“?”和“[...]”, 这是和Unix的B-Shell是相同的。

    print: *.c
         lpr -p $?
         touch print
*.c 说明目标文件依赖所有 .c 文件。 ```objects = *.o``` 中的 *.o 并不会展开， objects 的值就是 *.o， 如果需要 objects 为展开值则应写成：

	objects := $(wildcard *.o)
	
#### 文件搜寻 VPATH

在一些大的工程中，有大量的源文件，我们通常的做法是把这许多的源文件分类，并存放在不同的目录中。所以，当make需要去找寻文件的依赖关系时，你可以在文件前加上路径，但最好的方法是把一个路径告诉make，让make在自动去找。Makefile 文件中的特殊变量 “VPATH” 就可以完成这个功能，```  VPATH = src:../headers``` 指定了两个目录 src 和 ../headers， make 会按照这个顺序进行搜索。另一种方式是使用 vpath 关键字， 可以俩徐使用 vpath 语句来制定不同的搜索策略， vpath 语句中出现了相同的 <pattern> ，则 make 会根据语句的顺序来决定搜索的顺序。 

#### 伪目标

伪目标不会生成相应的目标文件，一般情况下也没有依赖。伪目标为第一个目标的时候也可以作为“默认目标”，比如需要生成多个目标文件的时候，可以使用伪目标的特性：

	 
	 all : prog1 prog2 prog3
    .PHONY : all

    prog1 : prog1.o utils.o
            cc -o prog1 prog1.o utils.o

    prog2 : prog2.o
            cc -o prog2 prog2.o

    prog3 : prog3.o sort.o utils.o
            cc -o prog3 prog3.o sort.o utils.o
            


#### 多目标

Makefile 的目标可以不止一个， 其可以支持过个目标， 有可能我们的多个目标依赖于同一个文件， 如下例:

	    bigoutput littleoutput : text.g
            generate text.g -$(subst output,,$@) >; $@

等价于：

	bigoutput : text.g
            generate text.g -big >; bigoutput
    littleoutput : text.g
            generate text.g -little >; littleoutput
            
其中 $@ 为自动化变量， 指代目标文件， subst 为函数，截取字符串的意思。

#### 静态模式

静态模式可以更加容易的定义多目标的规则，可以让我们的规则变得更加有弹性和灵活。大致语法如下:

	    <targets ...>;: <target-pattern>;: <prereq-patterns ...>;
            <commands>;
            ...

	
	
* targets 定义一系列的目标文件， 可以使用通配符。
* target-parttern 指定了 targets 和目标集模式。
* prereq-patterns 伪目标依赖的模式，它对 target-parrtern 形成了模式进行一次依赖目标的定义。

见下例：

    objects = foo.o bar.o

    all: $(objects)

    $(objects): %.o: %.c
            $(CC) -c $(CFLAGS) $< -o $@
等价于：


    foo.o : foo.c
            $(CC) -c $(CFLAGS) foo.c -o foo.o
    bar.o : bar.c
            $(CC) -c $(CFLAGS) bar.c -o bar.o

#### 自动生成依赖

在大型工程中一个 C 文件包含哪些头文件会经常发生变化， 这个时候我们就需要修改 Makefile。为了避免这样的问题 C/C++ 编译器支持了 -M 选项， 自动寻找原文件中包含的头文件， 并生成一个依赖关系。如:

     cc -M main.c
  
 输出：main.o : main.c defs.h 
 
 这个功能可以和 Makefile 结合起来， 为每个 “name.c” 文件都生成一个 “name.d” 的 Makefile 文件， 文件中就存放对应的文件的依赖关系。