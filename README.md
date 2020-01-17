# chdfs-fuse
## 简介
chdfs-fuse能让您在Linux系统中把Tencent CHDFS挂载到本地文件系统中，您能够便捷的通过本地文件系统操作CHDFS上的内容，实现数据的共享。
## 功能
chdfs-fuse基于fuse底层接口实现，主要功能包括：
- 支持POSIX文件系统的大部分功能，包括文件读写，目录操作，权限设置等
- MD5校验保证数据完整性

## 安装
### 安装依赖
为保障最优性能，建议linux kernel 5.0.0+。

Ubuntu 14.04+:
```bash
sudo apt-get install fuse
```
CentOS 6.5+:
```bash
sudo yum install fuse
```
### 二进制文件
为方便用户使用，提供了二进制可执行程序chdfs-fuse。[下载地址](https://github.com/tencentyun/chdfs-fuse/releases)


## 运行
### 挂载
- 普通模式
```bash
chmod +x ./bin/chdfs-fuse
#创建本地挂载目录
mkdir /mnt/chdfstest
nohup ./bin/chdfs-fuse /mnt/chdfstest/ --config=./conf/config.toml &
```
- 调试模式（查看fuse接口调用）
 ```bash
nohup ./bin/chdfs-fuse -debug /mnt/chdfstest/ --config=./conf/config.toml &
 ```
- 允许其他用户访问
 ```bash
nohup ./bin/chdfs-fuse -debug -allow_other /mnt/chdfstest/ --config=./conf/config.toml &
 ```
- 同步模式
```bash
nohup ./bin/chdfs-fuse -debug -o sync /mnt/chdfstest/ --config=./conf/config.toml &
```
 如果遇到类似“/mnt/chdfstest: Transport endpoint is not connected”等字样，通常是由于强杀进程导致无法重新挂载，建议先umount再mount：
```bash
 umount /mnt/chdfstest
 nohup ./bin/chdfs-fuse /mnt/chdfstest/ -config=./conf/config.toml &
```
### 取消挂载
```bash
umount /mnt/chdfstest/
```
如果遇到类似“umount: /mnt/chdfstest/: target is busy.”等字样，通常是由于挂载点正在被使用，导致无法直接卸载，建议先查看在使用的进程：
```bash
fuser -mv /mnt/chdfstest/
```
再杀死占用的进程：
```bash
fuser -kv /mnt/chdfstest/
```
再次查看（仅剩kernel占用为止）：
```bash
fuser -mv /mnt/chdfstest/
```
再使用卸载命令：
```bash
umount /mnt/chdfstest/
```
注意：
- 可以使用fuser -kv /mnt/chdfstest/进行kill进程。
- 也可以使用kill命令杀掉对应的进程 。
- 强制kill进程可能会导致数据丢失，请确保数据得到有效备份后，再进行相关操作。
### 配置样例
```text
[proxy]
url="http://f4mxxxxxxxx-xxxx.chdfs.ap-beijing.myqcloud.com"

[client]
mount-point="f4mxxxxxxxx-xxxx"
renew-session-lease-time-sec=10

[cache]
update-sts-time-sec=30
cos-client-timeout-sec=5
inode-attr-expired-time-sec=30
[cache.read]
block-expired-time-sec=10
max-block-num=256
read-ahead-block-num=15
max-cos-load-qps=1024
load-thread-num=128
select-thread-num=64
[cache.write]
max-mem-table-range-num=32
max-mem-table-size-mb=64
max-cos-flush-qps=256
flush-thread-num=128
commit-queue-len=100
max-commit-heap-size=500
auto-merge=true

[log]
level="info"

[log.file]
filename="./log/chdfs.log"
log-rotate=true
max-size=2000
max-days=7
max-backups=100
```
### 配置介绍
|名称|默认值|描述|
|-|-|-|
|proxy.url|-|远程挂载地址，例：http://f4mxxxxxxxx-xxxx.chdfs.ap-beijing.myqcloud.com|
|security.ssl-ca-path|-|CA路径，例：/etc/ssl/certs/ca-bundle.crt|
|client.renew-session-lease-time-sec|10|会话续租时间（s）|
|client.mount-point|-|远程挂载点，例：f4mxxxxxxxx-xxxx|
|client.user|当前用户名|用户名|
|client.group|当前组名|组名|
|client.force-sync|false|强制sync，不依赖“-o sync”|
|cache.update-sts-time-sec|30|数据读写临时密钥刷新时间（s）|
|cache.cos-client-timeout-sec|5|数据上传/下载超时时间（s）|
|cache.inode-attr-expired-time-sec|30|inode属性缓存有效时间（s）|
|cache.read.block-expired-time-sec|10|【读操作】单Fd数据读缓存有效时间（s）（block粒度）|
|cache.read.max-block-num|256|【读操作】单Fd数据读缓存block最大数量|
|cache.read.read-ahead-block-num|15|【读操作】单Fd预读block数量（read-ahead-block-num < max-block-num）|
|cache.read.max-cos-load-qps|1024|【读操作】多Fd数据下载最大QPS（QPS * 1M < 网卡带宽）|
|cache.read.load-thread-num|128|【读操作】多Fd数据下载worker数量|
|cache.read.select-thread-num|64|【读操作】多Fd元数据查询队列长度|
|cache.write.max-mem-table-range-num|32|【写操作】单Fd当前数据写缓存range最大数量|
|cache.write.mem-table-size-mb|64|【写操作】单Fd当前数据写缓存最大容量（MB）|
|cache.write.max-cos-flush-qps|256|【写操作】多Fd数据上传最大QPS（QPS * 4M < 网卡带宽）|
|cache.write.flush-thread-num|128|【写操作】多Fd数据上传worker数量|
|cache.write.commit-queue-len|100|【写操作】单Fd元数据提交队列长度|
|cache.write.max-commit-heap-size|500|【写操作】单Fd元数据提交最大容量（无需设置）|
|cache.write.auto-merge|true|【写操作】单Fd写时合并文件碎片|
|log.level|info|日志级别|
|log.file.filename|default.log|日志文件名|
|log.file.log-rotate|true|日志分割|
|log.file.max-size|-|单个日志文件最大容量（MB）|
|log.file.max-days|-|单个日志文件保存最长时间（天）|
|log.file.max-backups|-|历史日志文件最多文件数量|

## 局限性
- 暂不支持链接操作和扩展属性设置，已实现fuse底层接口如下：
  - StatFS
  - LookUpInode
  - GetInodeAttr
  - SetInodeAttr
  - MkDir
  - CreateFile
  - Rename
  - RmDir
  - Unlink
  - OpenDir
  - ReadDir
  - ReleaseDirFd
  - OpenFile
  - ReadFile
  - WriteFile
  - SyncFile
  - FlushFile
  - ReleaseFileFd
- 不支持单文件并发写
- 数据读操作不是强一致，即时读取的数据存在有效时间，下次读取时命中会返回

## 内存回收策略
chdfs-fuse用go语言实现，runtime在释放内存时默认行为是MADV_FREE。读写文件完成后，若观察到chdfs-fuse占用的内存没有下降，表示此时gc的内存空间没有返回给操作系统，但不影响其他进程运行，若其他进程抢占内存空间，chdfs-fuse占用的内存会迅速下降。
当然，释放内存也可以强制采用MADV_DONTNEED行为，读写文件完成后，gc的内存空间会返回给操作系统。
```bash
GODEBUG=madvdontneed=1 ./bin/chdfs-fuse /mnt/chdfstest/ --config=./conf/config.toml
```
- MADV_DONTNEED
表示应用程序不希望很快访问此地址范围。
- MADV_FREE
表示应用程序不需要此地址范围中包含的信息，但可以立即重用这些页面。