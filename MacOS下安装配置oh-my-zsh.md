# [oh-my-zsh](https://ohmyz.sh/)安装与配置

### 1. 安装homebrew  

```
$ xcode-select --install  
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"  
```

### 2. 安装zsh  

```
$ brew install zsh
```

### 3. 安装oh-my-zsh  

```
$ sh -c "$(wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```
### 4. 安装powerline fonts

```
$ git clone https://github.com/powerline/fonts.git --depth=1  
$ cd fonts  
$ ./install.sh  
$ cd ..  
$ rm -rf fonts  
```  
### 5. 配置.zshrc  

```
$ vim ~/.zshrc
ZSH_THEME="agnoster"
```

### 6. 配置iTerm2  

```
Profiles -> Text -> Font [Meslo LG ... for Powerline]
```

### 7. 成果   

![oh-my-zsh](https://github.com/coolzhang/myblog/blob/master/misc/ohmyz.png)