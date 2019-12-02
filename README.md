# chdfs-fuse
## 简介
chdfs-fuse能让您在Linux系统中把Tencent CHDFS挂载到本地文件系统中，您能够便捷的通过本地文件系统操作CHDFS上的内容，实现数据的共享。
## 功能
chdfs-fuse基于fuse底层接口实现，主要功能包括：
- 支持POSIX文件系统的大部分功能，包括文件读写，目录操作，权限设置等
- MD5校验保证数据完整性

## 安装
### 安装依赖
Ubuntu 14.04+:
```bash
sudo apt-get install fuse
```
CentOS 6.5+:
```bash
sudo yum install fuse
```
### 二进制文件
为方便用户使用，提供了二进制可执行程序chdfs-fuse。


## 运行
### 挂载
- 普通模式
```bash
chmod +x ./chdfs-fuse
#创建本地挂载目录
mkdir /mnt/chdfstest
./chdfs-fuse /mnt/chdfstest/ --config=./conf/config.toml
```
- 调试模式（查看fuse接口调用）
 ```bash
./chdfs-fuse -debug /mnt/chdfstest/ --config=./conf/config.toml
 ```
 
### 取消挂载
```bash
fusermount -u /mnt/chdfstest/
```
### 配置样例
```text
#[security]
#ssl-ca-path="/etc/ssl/certs/ca-bundle.crt"

[proxy]
url="https://f4mxxxxxxxx-xxxx.chdfs.ap-beijing.myqcloud.com:443"

[client]
mount-point="f4mxxxxxxxx-xxxx"
renew-session-lease-time-sec=10

[cache]
latch-num=10
update-sts-time-sec=30
cos-client-timeout-sec=5
[cache.read]
block-expired-time-sec=10
max-block-num=256
read-ahead-block-num=63
[cache.write]
max-mem-table-range-num=32
max-mem-table-size-mb=64
max-cos-flush-qps=32
flush-thread-num=64
flush-queue-len=64
commit-queue-len=50
max-commit-heap-size=500

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
|proxy.url|-|远程挂载地址，例：https://f4mxxxxxxxx-xxxx.chdfs.ap-beijing.myqcloud.com:443|
|security.ssl-ca-path|-|CA路径，例：/etc/ssl/certs/ca-bundle.crt|
|client.renew-session-lease-time-sec|10|会话续租时间（s）|
|client.mount-point|-|远程挂载点，例：f4mxxxxxxxx-xxxx|
|client.uid|当前用户Uid|用户Uid|
|client.gid|当前用户Gid|用户Gid|
|cache.latch-num|10|Fd哈希槽数量|
|cache.update-sts-time-sec|30|数据读写临时密钥刷新时间（s）|
|cache.cos-client-timeout-sec|5|数据上传/下载超时时间（s）|
|cache.read.block-expired-time-sec|10|【读操作】单Fd数据读缓存有效时间（s）（block粒度）|
|cache.read.max-block-num|256|【读操作】单Fd数据读缓存block最大数量|
|cache.read.read-ahead-block-num|63|【读操作】单Fd预读block数量|
|cache.write.max-mem-table-range-num|32|【写操作】单Fd当前数据写缓存range最大数量|
|cache.write.mem-table-size-mb|64|【写操作】单Fd当前数据写缓存最大容量（MB）|
|cache.write.max-cos-flush-qps|32|【写操作】多Fd数据上传最大QPS（QPS * BlockSize < 网卡带宽）|
|cache.write.flush-thread-num|64|【写操作】多Fd数据上传worker数量|
|cache.write.flush-queue-len|64|【写操作】多Fd数据上传队列长度|
|cache.write.commit-queue-len|50|【写操作】单Fd元数据提交队列长度|
|cache.write.max-commit-heap-size|500|【写操作】单Fd元数据提交最大容量（无需设置）|
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
GODEBUG=madvdontneed=1 ./chdfs-fuse /mnt/chdfstest/ --config=./conf/config.toml
```
- MADV_DONTNEED
表示应用程序不希望很快访问此地址范围。
- MADV_FREE
表示应用程序不需要此地址范围中包含的信息，但可以立即重用这些页面。