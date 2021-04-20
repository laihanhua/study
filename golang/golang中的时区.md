## golang中的时区对比

### 对比结果
#### 应用和数据库在同一时区
类型 | 系统时区 | goalng 设置时区 和 goalng输出 | mysql插入和select
---|---|---|---|
timestamp | CST | parseTime=true&loc=UTC:                 2019-04-02 13:18:17 +0000 UTC <br>parseTime=true&loc=Asia/Shanghai:        2019-04-02 13:18:17 +0800 CST<br>parseTime=true&loc=Europe/London:        2019-04-02 13:18:17 +0100 BST<br>parseTime=true&loc=Local:                2019-04-02 13:18:17 +0800 CST | 2019-04-02 13:18:17(CST): 2019-04-02 13:18:17
timestamp | BST |parseTime=true&loc=UTC:                  2019-04-02 06:18:17 +0000 UTC<br>parseTime=true&loc=Asia/Shanghai:        2019-04-02 06:18:17 +0800 CST<br>parseTime=true&loc=Europe/London:        2019-04-02 06:18:17 +0100 BST<br>parseTime=true&loc=Local:                2019-04-02 06:18:17 +0100 BST |2019-04-02 13:18:17(CST):2019-04-02 06:18:17

### 一句话总结
timestamp,date,datetime中，只有timestamp包括时区信息；
插入时，不指定时区的话，timestamp就使用mysql的时区
数据持久化后，timestamp,date,datetime都相当于字符串的操作
取出来时，timestamp等于入库时候的时区时间+程序的时区，date,datetime不受影响(因为包含时区信息)

