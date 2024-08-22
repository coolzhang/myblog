## MacOS上如何配置酷炫iTerm2

### 安装brew
```
# 中科大国内源
export HOMEBREW_CORE_GIT_REMOTE=https://mirrors.ustc.edu.cn/homebrew-core.git
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装常用软件
brew install wget
brew install python3
brew install mysql@5.7  # echo 'export PATH="/usr/local/opt/mysql@5.7/bin:$PATH"' >> ~/.zshrc
brew install lrzsz
``` 

### 卸载brew
```
wget https://raw.githubusercontent.com/Homebrew/install/master/uninstall -O uninstall.rb
ruby uninstall.rb
```

### 安装iTerm2
```
brew install --cask iterm2
```

### 配置iTerm2支持rz/sz
* 下载这两个脚本[iterm2-send-zmodem.sh](https://github.com/coolzhang/myblog/blob/master/misc/iterm2-send-zmodem.sh)、[iterm2-recv-zmodem.sh](https://github.com/coolzhang/myblog/blob/master/misc/iterm2-recv-zmodem.sh) 到目录/usr/local/bin目录下。
* 配置终端支持rz/sz，如下：

```
打开iTerm2的Preferences -> Profiles ->  Advanced -> Triggers，配置如下：
# 发送 sz
Regular expression: rz waiting to receive.\*\*B0100 (注意这里是这样)
Action: Run Silent Coprocess
Parameters: /usr/local/bin/iterm2-send-zmodem.sh
# 接收 rz
Regular expression:\*\*B00000000000000
Action: Run Silent Coprocess
Parameters: /usr/local/bin/iterm2-recv-zmodem.sh
#
# 参考文档：https://www.jianshu.com/p/d1bef4140e34
#
```

### 安装oh-my-zsh
```
curl -L https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh | sh
```

### 配置omz的语法高亮、命令补全、字体风格、venv提示
```
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

brew install svn && sudo mkdir -p /Library/Java/Extensions && sudo ln -s /usr/local/lib/libsvnjavahl-1.dylib /Library/Java/Extensions/libsvnjavahl-1.dylib
brew install homebrew/cask-fonts/font-roboto-mono-for-powerline
git clone https://github.com/Powerlevel9k/powerlevel9k.git ~/.oh-my-zsh/themes/powerlevel9k
exec $SHELL

# vim ~/.zshrc
ZSH_THEME="powerlevel9k/powerlevel9k"
POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(virtualenv context dir vcs)
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(status root_indicator background_jobs history time)
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)

# vim ~/.oh-my-zsh/plugins/virtualenv/virtualenv.plugin.zsh
注释掉 export VIRTUAL_ENV_DISABLE_PROMPT=1

echo "source ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${ZDOTDIR:-$HOME}/.zshrc
```

### 效果图
![iterm2_with_omz](https://github.com/coolzhang/myblog/blob/master/misc/iterm2_screen.png)

### 美化命令输出内容
```
brew install grc

# vim ~/.zshrc
alias colourify="grc -es --colour=auto"
alias diff='colourify diff'
alias make='colourify make'
alias gcc='colourify gcc'
alias g++='colourify g++'
alias as='colourify as'
alias gas='colourify gas'
alias ld='colourify ld'
alias netstat='colourify netstat'
alias ping='colourify ping'
alias traceroute='colourify /usr/sbin/traceroute'
alias head='colourify head'
alias tail='colourify tail'
alias dig='colourify dig'
alias mount='colourify mount'
alias ps='colourify ps'
alias mtr='colourify mtr'
alias df='colourify df'
```
