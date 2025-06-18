---
title: mongodb在腾讯云部署和使用
subtitle: 在AI领域很多场景需要识别图文中的文字
date: 2024-4-14 13:30:00
categories: other
index_img: /img/mongodb-logo.png
# banner_img: /img/read-code-index.png
---

## 安装

1. openCloudOS 9, 安装时选择 redHat,选择后会自动显示`redHat/centos x64`
2. 解压后，bin 文件夹下之前有`mongo.conf`，`mongodb 8.09`版本没有，可以自己在 bin 文件夹下创建`mongo.conf`,启动时`./mongod --config mongodb.conf`
3. `/www/server/mongodb/bin/mongod: error while loading shared libraries: libcrypto.so.1.1: cannot open shared object file: No such file or directory`
   1. sudo dnf install -y compat-openssl11
   2. find / -name "libcrypto.so.1.1" 2>/dev/null
      1. 若输出类似 /usr/lib64/libcrypto.so.1.1，则成功

## 启动 mongodb

1. 安装好 mongodb
2. 创建 db 目录和 log 目录，log 目录里面有 mongo.log，是 logpath
3. 使用`mongo.conf`文件进行配置参数

   ```
    # 设置数据文件的存放目录
    storage:
      dbPath: ../data

    # 设置日志文件的存放目录及其日志文件名
    systemLog:
      destination: file
      path: ../logs/mongodb.log
      logRotate: reopen
      logAppend: true
      verbosity: 0

    # 设置端口号（默认的端口号是 27017）
    net:
      port: 27099
      bindIp: 0.0.0.0

    # 设置为以守护进程的方式运行，即在后台运行
    # fork: true

    # 防止通过HTTP进行访问，3版本以后就没有这个参数了
    # nohttpinterface: true

    # 启用身份验证,在未创建校验用户之前，需要关闭
    security:
      authorization: enabled

    setParameter:
      enableLocalhostAuthBypass: 0

    #设置为以守护进程的方式运行
    processManagement:
      fork: true

   ```

4. 创建身份账户

   ```
    use('admin');

    db.createUser({
      user: "xxx",
      pwd: "xxx",
      roles: [
        { role: "userAdminAnyDatabase", db: "admin" },
        { role: "readWriteAnyDatabase", db: "admin" }
      ]
    })
   ```

5. 启动 mongodb: `cd /www/server/mongodb/bin && ./mongod --config mongodb.conf`
6. 查看后台进程和关闭：
   - ps aux | grep mongod 和 kill pid
   - sudo systemctl stop mongod(使用 systemctl（仅限于使用 systemd 的 Linux 发行版)
   - sudo service mongod stop(使用 service 命令（仅限于使用 init 的 Linux 发行版)
     sudo service /www/server/mongodb/bin/mongod stop

## 参考

> mac: https://juejin.cn/post/7052585815037673479

> centos: https://juejin.cn/post/7163071747414032391

> 远程连接+身份认证：https://blog.csdn.net/jianleking/article/details/79715097

> 内置角色：https://www.mongodb.com/docs/manual/reference/built-in-roles/

```

```
