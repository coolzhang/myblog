# 背景  
配置基于vim的开发环境  

# 准备  

## 依赖的软件(vim或插件)
python 3.6、ncurses-devel、cmake3、devtoolset-8 ([安装](https://www.softwarecollections.org/en/scls/rhscl/devtoolset-8/))   

## [vim](https://github.com/vim/vim.git)版本  
* vim 8 ([需要基于python3.6+进行编译安装](https://blog.csdn.net/yang5915/article/details/109704633))  

## 安装vim插件管理器[vim-plug](https://github.com/junegunn/vim-plug)   
```
# 安装vim-plug
shell> curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
         https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim  
# vim配置
shell> vim ~/.vimrc  
" Specify a directory for plugins
call plug#begin('~/.vim/plugged')

Plug 'fatih/vim-go', { 'do': ':GoUpdateBinaries' }
Plug 'ycm-core/YouCompleteMe', { 'do': './install.py' }

" Initialize plugin system
call plug#end()
# 安装相关插件
shell> vim +PlugInstall
# 额外操作（默认gopls安装到/root/go/bin下）
shell> mkdir -p /root/.vim/plugged/YouCompleteMe/third_party/ycmd/third_party/go
shell> ln -s /root/go/bin /root/.vim/plugged/YouCompleteMe/third_party/ycmd/third_party/go/bin
```
# Vim配置  

```
#
# ~/.vimrc
#
" YCM settings
let g:ycm_key_invoke_completion = '<c-z>'
let g:ycm_semantic_triggers =  {
                        \ 'c,cpp,python,java,go,erlang,perl': ['re!\w{2}'],
                        \ 'cs,lua,javascript': ['re!\w{2}'],
                        \ }
let g:ycm_filetype_whitelist = {
                        \ "c":1,
                        \ "cpp":1,
                        \ "objc":1,
                        \ "go": 1,
                        \ "sh":1,
                        \ "zsh":1,
                        \ "py":1,
                        \ }
" set mapleader
let mapleader = ","

au Filetype go nnoremap <leader>v :vsp <CR>:exe "GoDef" <CR>
au Filetype go nnoremap <leader>s :sp <CR>:exe "GoDef"<CR>
au Filetype go nnoremap <leader>t :tab split <CR>:exe "GoDef"<CR>
au Filetype go nnoremap <leader>r : <CR>:exe "GoRun" <CR>
au Filetype go nnoremap <leader>d : <CR>:exe "GoDoc" <CR>
```
# 参考
* [Golang开发环境搭建-Vim篇](https://www.cnblogs.com/bonelee/p/6500613.html)  
* [Go development environment for Vim](https://blog.gopheracademy.com/vimgo-development-environment/)
* [Vim as a Go IDE using LSP and vim-go](https://octetz.com/docs/2019/2019-04-24-vim-as-a-go-ide/)