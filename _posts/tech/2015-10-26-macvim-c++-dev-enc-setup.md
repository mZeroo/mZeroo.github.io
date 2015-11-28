---
layout: post
title: MacVim C++ 开发环境设置
category: 技术
tags: 工具
keywords: MacVim, C++
description: 
---  

## - 安装软件和插件列表


* [macvim](https://github.com/b4winckler/macvim) : vim for mac
* [Vundle](https://github.com/VundleVim/Vundle.vim): vim 插件安装管理插件
* [YouCompleteMe](https://github.com/Valloric/YouCompleteMe): 代码补全插件
* [syntastic](https://github.com/scrooloose/syntastic#introduction): 语法分析和错误提示插件，可实时提示语法错误
* [NERD_tree](https://github.com/scrooloose/nerdtree): 目录树插件
* [command-t](https://github.com/wincent/Command-T): 快速导航定位文件插件
* [vim-gitgutter](https://github.com/airblade/vim-gitgutter): vim git 插件，看文件修改情况
* [Ag](https://github.com/ggreer/the_silver_searcher) : a better grep, ag.vim 插件依赖于它
* [ag.vim](https://github.com/rking/ag.vim): 文本查找插件
* [ctags](http://ctags.sourceforge.net/) 遍历源代码文件生成tags文件
* [taglist](https://github.com/vim-scripts/taglist.vim): 显示函数列表
* [a.vim](https://github.com/vim-scripts/a.vim): .h 和 .cpp 切换
* 自动删除行为空格


## - 具体安装
#### 1) install macvim

mac 下更好的 vim ：[Git Repo](https://github.com/b4winckler/macvim)

#### 2) install Vundle

类似于 python pip，vim 插件安装管理工具： [Git Repo](https://github.com/VundleVim/Vundle.vim)

#### 3) install YouCompleteMe

代码补全插件： [Git Repo](https://github.com/Valloric/YouCompleteMe)

在有了 Vundle 之后， 安装 YouCompleteMe， 只需要在 .vimrc 添加 ```Bundle 'Valloric/YouCompleteMe'``` ， 然后打开 macvim，输入 :BundleInstall 即安装完成。 添加 C-family 语言支持， 在终端执行如下命令:

	# Compiling YCM with semantic support for C-family languages:
	cd ~/.vim/bundle/YouCompleteMe
	./install.py --clang-completer #(you may need to install CMake first)

重新打开 macvim 就具备提示功能了， 如果 macvim 提示 : no .ycm_extra_conf.py file detected，可以将 ~/.vim/bundle/YouCompleteMe//third_party/ycmd/cpp/ycm/.ycm_extra_conf.py 拷贝到 Home 目录下， 同时将 ```echo | clang -std=c++11 -stdlib=libc++ -v -E -x c++ -``` path 添加到 -isystem 中， 如：


		
    ➜  learn_makefile  echo | clang -std=c++11 -stdlib=libc++ -v -E -x c++ -
    Apple LLVM version 6.0 (clang-600.0.57) (based on LLVM 3.5svn)
    Target: x86_64-apple-darwin14.0.0
    Thread model: posix
     "/Library/Developer/CommandLineTools/usr/bin/clang" -cc1 -triple x86_64-apple-macosx10.10.0 -E -disable-free -disable-llvm-verifier -main-file-name - -mrelocation-model pic -pic-level 2 -mdisable-fp-elim -masm-verbose -munwind-tables -target-cpu core2 -target-linker-version 241.9 -v -resource-dir /Library/Developer/CommandLineTools/usr/bin/../lib/clang/6.0 -stdlib=libc++ -std=c++11 -fdeprecated-macro -fdebug-compilation-dir /Users/xiemeng/learn_makefile -ferror-limit 19 -fmessage-length 176 -stack-protector 1 -mstackrealign -fblocks -fobjc-runtime=macosx-10.10.0 -fencode-extended-block-signature -fcxx-exceptions -fexceptions -fdiagnostics-show-option -fcolor-diagnostics -vectorize-slp -o - -x c++ -
    clang -cc1 version 6.0 based upon LLVM 3.5svn default target x86_64-apple-darwin14.0.0
    ignoring nonexistent directory "/usr/include/c++/v1"
    #include "..." search starts here:
    #include <...> search starts here:
     /Library/Developer/CommandLineTools/usr/bin/../include/c++/v1
     /usr/local/include
     /Library/Developer/CommandLineTools/usr/bin/../lib/clang/6.0/include
     /Library/Developer/CommandLineTools/usr/include
     /usr/include
     /System/Library/Frameworks (framework directory)
     /Library/Frameworks (framework directory)
    End of search list.
    # 1 "<stdin>"
    # 1 "<built-in>" 1
    # 1 "<built-in>" 3
    # 188 "<built-in>" 3
    # 1 "<command line>" 1
    # 1 "<built-in>" 2
    # 1 "<stdin>" 2

将路径添加到  ~/.ycm_extra_conf.py 中：  

	'-isystem',
    '/usr/include',
    '-isystem',
    '/usr/local/include',
    '-isystem',
    '/Library/Developer/CommandLineTools/usr/include',
    '-isystem',
    '/Library/Developer/CommandLineTools/usr/bin/../include/c++/v1',
    '-isystem',
    '/Library/Developer/CommandLineTools/usr/bin/../lib/clang/7.0/include',
    '-isystem',
    '/Library/Developer/CommandLineTools/usr/include',
    '-isystem',
    '/System/Library/Frameworks',
    '-isystem',
    '/Library/Frameworks'


YouCompleteMe 就安装完成了， 这时候打开 macvim 编辑 cpp 文件就可以有自动提示了。 


#### 4) install syntastic
语法检查插件： [Git Repo](https://github.com/scrooloose/syntastic#introduction)
添加  scrooloose/syntastic 到 .vimrc 进行安装即可。


#### 5) install NERD_tree

树形目录插件： [Git Repo](https://github.com/scrooloose/nerdtree)

将 Bundle 'scrooloose/nerdtree' 添加到 .vimrc 进行安装。添加 map <C-n> :NERDTreeToggle<CR> 到 .vimrc, 打开 macvim 输入 Ctrl + n 即可。如果想要在目录中忽略某些文件， 设置如下命令:
	
	let NERDTreeIgnore = ['\.pyc$', '\.o$']


#### 6) install command-t

提供快速导航定位文件功能的插件： [Git repo](https://github.com/wincent/Command-T)
将 Bundle 'wincent/command-t' 添加到 .vimrc 进行安装。还需要执行如下命令：

	cd ~/.vim/bundle/command-t
	rake make

之后打开 macvim 输入 ```\t```， 就可以进行文件查找了。如果遇到 command-t vim "Could not load C extension"， 注意 ruby 版本的提示， 将本地 ruby 使用 rvm 调整为要求的版本， 并执行重新执行上述命令即可（记得 clean）。

如果在搜索文件的时候需要忽略默写类型的文件， 比如 *.o 文件， 在 .vimrc 添加如下配置：

	set wildignore=*.swp,*.bak,*.pyc,*.class,*.jar,*.gif,*.png,*.jpg,*.o


#### 7) install vim-gitgutter

实时显示 git diff 的插件： [Git Repo](https://github.com/airblade/vim-gitgutter)


#### 8) install ag.vim

[Ag](https://github.com/ggreer/the_silver_searcher): 更好的 grep。
ag.vim 就是它的一个 wrapper，[Git Repo](https://github.com/rking/ag.vim) 安装方法同上。

#### 9) install ctags

brew install ctags

#### 10) install taglist

	" add to ~/.vimrc
	Bundle 'vim-scripts/taglist.vim'
	let Tlist_Show_One_File=1     "不同时显示多个文件的tag，只显示当前文件的 
	let Tlist_Exit_OnlyWindow=1   "如果taglist窗口是最后一个窗口，则退出vim  
	let Tlist_Use_Right_Window = 1         "在右侧窗口中显示taglist窗口

安装后，打开 vim， 输入 :Tlist 即可

#### 11）自动删除行尾空格


	fun! StripTrailingWhitespace()
    	"Don't strip on these filetypes
    	if &ft =~ 'markdown'
        	return
    	endif
    	%s/\s\+$//e
	endfun
	autocmd BufWritePre * call StripTrailingWhitespace()

添加到 .vimrc 就可以删除烦人的行尾空格了

## - 参考

[1] [http://blog.jobbole.com/58978/](http://blog.jobbole.com/58978/)