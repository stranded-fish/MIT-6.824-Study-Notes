# lab1 tmp

1. 问题描述：

plugin.Open("wc"): plugin was built with a different version of package runtime

If you change anything in the mr/ directory, you will probably have to re-build any MapReduce plugins you use, with something like go build -race -buildmode=plugin ../mrapps/wc.go

1. gob error encoding body
https://blog.csdn.net/qq_33781658/article/details/88110758

1. 包权限（首字母大写 = public， 首字母小写 private）

rpc 包中必须要大写，才能正常传递、解析

1. sockName := coordinatorSock() 相关


**bash 相关**

1. wait 命令 -n 踩雷

https://www.cnpython.com/qa/54331

https://computingforgeeks.com/how-to-install-bash-5-on-centos-linux/

