## 解决复制粘贴时全注释问题

>在`vim`中，当复制带有#号开头的行时，会自动注释掉后面的内容。如果想要关闭这个功能，可以执行命令`:set nopaste`

```shell
:set paste  ## 进入粘贴模式，临时使用
```


```shell
$ tail -2 /etc/vimrc  ## 直接修改全局vim 配置，立即生效
set paste
set number
```

## .vimrc

[http://xstarcd.github.io/wiki/vim/vim_indent.html](http://xstarcd.github.io/wiki/vim/vim_indent.html)
[https://segmentfault.com/a/1190000021133524](https://segmentfault.com/a/1190000021133524)


### 1 Redhat

```shell

######## v1 
syntax on   # 语法高亮
set scrolloff=7 # 让光标始终处在屏幕中间，正常距离是7行
set ai  # 回车后自动缩进到合适位置
set ts=4  # 正常情况下一个tab键是4个空格
set et    # 将所有tab键的制表符换成空格
set encoding=utf8  # utf8 字符集
autocmd FileType yaml setlocal sw=2 ts=2 et ai  # 当文件是yaml格式时将vim相关规则定为2空格对齐、tab是2空格、tab自动转空格、自动缩进
nnoremap <C -a> <Home>    # 快捷键curl + a = HOME键
nnoremap <C -e> <End>     # 快捷键curl + e = END键
nnoremap ; :              # 按; = ：

######## v2
syntax on
set scrolloff=7
set ai
set ts=4
set et
set encoding=utf8
autocmd FileType yaml setlocal sw=2 ts=2 et ai
nnoremap <C -a> <Home>
nnoremap <C -e> <End>
nnoremap ; :

######## v3
syntax on
set ai
set ts=2
set sw=2
set et

########## v4 
syntax on
set scrolloff=7
set ai
set ts=4
set et
set encoding=utf8
autocmd FileType yaml setlocal sw=2 ts=2 et ai
nnoremap <C-a> <Home>
nnoremap <C-e> <End>
nnoremap <F2> :set nu! nu?<CR>
nnoremap ; :


# 自动缩进
:set ai  # 自动缩进，在按o之后会自动缩进并与上一行保持对齐

############ v5
vim ~/.vimrc

-------------------------------------------------------------

syntax on
set scrolloff=7
set mouse=a  # 鼠标定位
set ai
set ts=4
set et
set encoding=utf8
autocmd FileType yaml setlocal sw=2 ts=2 et ai
nnoremap <C-a> <Home>
nnoremap <C-e> <End>
nnoremap <F2> :set nu! nu?<CR>
nnoremap ; :
```

### 2 Ubuntu

```shell
set nu
set nowrap
set encoding=utf-8
set fenc=utf-8
set mouse=v
set expandtab
set hlsearch
set textwidth=79
set fileformat=unix
set cindent                 #  以C/C++的模式缩进
set scrolloff=5            # 设定光标离窗口上下边界 5 行时窗口自动滚动
set shiftwidth=4          # 设定 << 和 >> 命令移动时的宽度为 4
set softtabstop=4       # 使得按退格键时可以一次删掉 4 个空格,不足 4 个时删掉所有剩下的空格）
set tabstop=4            # 设定 tab 长度为 4
syntax on                   #  自动语法高亮
set autoindent # 自动对齐
set smartindent # 智能对齐
```

## 插入时间

```shell
# 光标选中要插入的位置
:r!date
# 会另起一行插入当前时间
```


## 同时查看两个文件

```shell
vim -O register{,1}.yaml
```


## 美化bash

https://linux.cn/article-8118-1.html
