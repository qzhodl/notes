logrotate 可以自动：
 - 按大小/天数切分
 - 自动压缩
 - 自动删除旧文件
 - 自动重新创建空日志文件

 1. 创建 logrotate 配置文件
 ```bash
 sudo vim /etc/logrotate.d/op-node
 ```bash

  ```bash
/var/log/myapp.log {
    size 100M
    rotate 7
    compress
    delaycompress
    missingok
    copytruncate
    notifempty
    su es es
}
 ```bash

 解释为：
 - 单个文件超过 100M 就 rotate
 - 保留 7 个旧文件
 - 自动 gzip 压缩
 - 第二天再压缩上一个
 - 文件不存在时不报错
 - 复制并清空原文件（适用于 tee）
 - 空文件不 rotate

2. 测试 logrotate 配置是否有效
```bash
sudo logrotate -d /etc/logrotate.d/op-node
```
3. 真正执行
```bash
sudo logrotate -f /etc/logrotate.d/op-node
```
4. logrotate 默认每天运行一次
```bash
systemctl status logrotate.timer
```
