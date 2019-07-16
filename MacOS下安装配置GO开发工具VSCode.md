MacOS下安装配置GO开发工具-Visual Studio Code

### 1. 下载&安装Go  

> https://golang.google.cn/     # 默认安装在/usr/local/go目录下

### 2. 配置Go环境变量
$ mkdir -p $HOME/go/{bin,pkg,src}  
$ vim ~/.bash_profile   
> PATH=$PATH:/usr/local/go/bin:$HOME/go/bin  
> GOROOT=/usr/local/go  
> GOPATH=$HOME/go  
> export GOROOT GOPATH PATH  

### 3. 下载&安装vscode

> https://code.visualstudio.com/download

### 4. 配置相关插件  

#### 4-1. 创建必要目录  

> $ mkdir -p $GOPATH/src/{github.com/golang,golang.org/x}  

#### 4-2. 手动按照Go工具包
> $ cd $GOPATH/src/golang.org/x  
> $ git clone https://github.com/golang/tools.git tools   
> $ cp -r tools $GOPATH/src/github.com/golang/  

#### 4-3. 按照vscode提示安装所有相关工具

>Installing 11 tools at /Users/zhanghai/go/bin  
  gocode  
  gopkgs  
  go-outline  
  go-symbols  
  guru  
  gorename  
  dlv  
  gocode-gomod  
  godef  
  goreturns  
  golint  

>Installing github.com/mdempsky/gocode SUCCEEDED  
Installing github.com/uudashr/gopkgs/cmd/gopkgs SUCCEEDED  
Installing github.com/ramya-rao-a/go-outline SUCCEEDED  
Installing github.com/acroca/go-symbols SUCCEEDED  
Installing golang.org/x/tools/cmd/guru SUCCEEDED  
Installing golang.org/x/tools/cmd/gorename SUCCEEDED  
Installing github.com/go-delve/delve/cmd/dlv SUCCEEDED  
Installing github.com/stamblerre/gocode SUCCEEDED  
Installing github.com/rogpeppe/godef SUCCEEDED  
Installing github.com/sqs/goreturns SUCCEEDED  
Installing golang.org/x/lint/golint FAILED  

>1 tools failed to install.  

其中golint安装失败，需要手动安装，如下：  
> $ go get -u -v github.com/golang/lint/golint  
> $ cp -r github.com/golang/lint golang.org/x/  
> $ go install golang.org/x/lint/golint
