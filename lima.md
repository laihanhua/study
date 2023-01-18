## lima && nerdctl 替换 docker && docker desktop

intel mac机器，按照[教程安装lima](https://mdnice.com/writing/d6e0cdeac23a4bf29673b0bbd936737f)后，创建如下文件夹mkdir ~/.lima/bin，并在该文件夹下创建文件：

docker
```js
#!/usr/bin/env bash
# ~/.lima/bin/docker
lima nerdctl $@
```

docker-compose
```js
#!/usr/bin/env bash
# ~/.lima/bin/docker-compose
lima nerdctl compose $@
```

设置权限：chmod +x ~/.lima/bin/docker ~/.lima/bin/docker-compose

并在~/.zshrc或者~/.bashrc设置路径：export PATH=$HOME/.lima/bin:$PATH

完成后就可以按照以前docker的指令继续使用容器。

ps：可能存在部分指令不兼容的情况
