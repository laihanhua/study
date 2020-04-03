### logging模块

- <https://blog.csdn.net/zyz511919766/article/details/25136485#>
- <https://zhuanlan.zhihu.com/p/21476174>

#### 关于树形结构
1. 日志打印是通过logger对象，logging.getLogger(name)获取，如果name不输入的话，则默认是root。
2. 除了root logger外，其他的logger都有父logger；子logger会继承父logger的属性，且输出到子logger时，会先经过父logger处理，所以父子logger都会打印日志（既会将消息分发给他的handler进行处理也会传递给所有的祖先Logger处理）
3. logger可以设置propagate=0,propagate属性的真与假继续向上传递日志或停止上传。默认情况下，日志是会层层上递的，所有日志最终都会被送到根记录器
4. 可以通过logger.getLogger('A.B')来指定子logger，这里B继承A。
