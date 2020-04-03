#### 大概步骤
1. 安装组件依赖	

```
$ xcode-select --install
$ brew update
$ brew install mercurial
``` 

2. 安装gvm,`bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)`

3. 如果本地之前没有安装过go,需要先安装`gvm install g1.4 -B`，因为g1.5后需要使用go编译器来编译

4. 安装`gvm install g1.9`,默认使用`gvm use g1.9 --default`

5. 使用前，先执行`source ~/.gvm/scripts/gvm`，之后使用默认设置的go版本

#### 指令
1. gvm list： 查看已经安装的版本
2. gvm install gx.xxx: 安装对应的go版本
3. gvm use gx.xxx [--default]: 切换到go目标版本

#### 参考文献
1. https://laucyun.com/ff3bc3db699464aa76756e41be780712.html#directory-034818229470128454
2. https://github.com/moovweb/gvm