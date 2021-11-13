# alibaba-cloud-linux_3.2104_v20210910
#### 故障描述
在新购的阿里云ECS服务器上安装k8s单机集群，在对```Redis```单节点进行压力测试的时候，系统负载发生异常偏高情况。
说明：实际上只是压测的时候能100%复现问题，其他在容器启动或者运行过程中，也会出现系统负载异常偏高，只不过相比压测的时候少点。

#### 系统环境

```Alibaba Cloud Linux 3```镜像发布记录：https://help.aliyun.com/document_detail/212634.htm?spm=a2c4g.11186623.0.0.17e82bf7MmbbFz#concept-2070819

新购服务器```4核32G```预安装```Alibaba Cloud Linux  3.2104 64位```，内核版本如下。
```bash
root@master01: ~ 22:48:56
# uname -a
Linux master01 5.10.23-5.al8.x86_64 #1 SMP Fri Apr 23 16:56:08 CST 2021 x86_64 x86_64 x86_64 GNU/Linux
```
安装阿里云主机插件
```bash
# 有的主机可以选择不预选安装，这里为了保证测试环境一致，如果没有安装就手动安装一下。
ARGUS_VERSION=3.5.3 /bin/bash -c "$(curl -s https://cms-agent-cn-hangzhou.oss-cn-hangzhou-internal.aliyuncs.com/Argus/agent_install_ecs-1.2.sh)"
```
#### 安装 k8s 实验环境
