# gateway-load-balacer-in-single-vpc-quickstart
GWLB in single VPC Quickstart CloudFormation Template

## 一、架构说明

本方案构建一个单VPC的网络检测方案，可参考[之前多VPC的GWLB方案](https://github.com/aobao32/gateway-load-balacer-centralized-solution-quickstart)。

主要组件如下：

- NGFW：GWLB需要搭配Marketplaces启动的第三方商用防火墙，需要防火墙参加支持Geneve协议，本Quickstart中，使用Geneve-proxy模拟防火墙
- GWLB Endpoint：VPC的流量入口，作为路由表的下一跳接受流量
- GWLB：将流量分发给Target Group，目标组中每个EC2是接受Geneve协议封装后发送过来的原始流量，拆包检查
- NAT Gateway：本架构中所有对外流量的统一出口，所有VPC访问外部互联网访问均汇集到此后统一出口
- ELB：本架构中所有外部进入流量的统一入口
- VPC1：单个VPC运行所有组件

总体架构图和路由表说明如下。

![总体架构图和路由表说明](https://d51vuyprlknbq.cloudfront.net/GWLB/GWLB-in-single-VPC-Quickstart.png)

## 二、启动模版使用说明

使用方法如下：

- 下载所有模版到S3存储桶内，包含主模版和所有Nested嵌套模版
- 将文件设置为Public可访问，包含主模版和所有Nested嵌套模版，获取他们公开后的URL
- 替换主模版 `CN-GWLB.yml` 文件中，各Nested嵌套模版的URL网址为自己的存储桶网址
- 进入Cloudformation界面，选择输入模版的S3地址，填入主模版 `CN-GWLB.yml` 的S3上公开的地址
- 填写Cloudformation参数，除模版名称和EC2 Keypair的名字需要指定外，其余可以默认，完成创建

## 三、模版启动后的Post-configuration

### 1、开启 Geneve Proxy

本Quickstart使用的是Geneve Proxy模拟防火墙。在模版启动后，在VPC1中可以看到带有标签名为 `AccessVPC-VirtualAppliance1` 和 `AccessVPC-VirtualAppliance2` 的两个节点，这两个节点运行Amazon Linux 2，且已经下载了geneve proxy到 `/root/geneve-proxy` 目录。

配置方法是：

- 编辑 `/root/geneve-proxy/config.yml` 配置文件，编辑 `blocked_transport_protocols` 调整要拦截的访问，编辑 `allowed_transport_protocols` 允许访问，编辑完成保存退出
- 执行命令 `nohup python3 main.py &` 启动程序并排入后台
- 执行命令 `tcpdump -nvv 'port 6081'` 查看流量

以上配置过程可参考[这个](https://d5ubqqttnawof.cloudfront.net/video/GWLB-02-QuickstartDemo.mp4)视频（本视频有解说注意播放音量）。

### 2、补充GWLB Endpoint所需要的路由表

本步骤待更新。

## 四、内网向外网的访问

在名为VPC1PrivateSubnet的子网内，创建一个EC2。然后从本EC2发器ping外网的测试。

## 五、外网向内网的测试

在上一步创建的EC2上，部署web服务器。

创建一个Internet facing的ALB，并绑定到VPC1NATSubnet上。在这个ALB的Target选择为IP类型，然后输入步骤四的EC2的内网IP地址，通过健康检查。

然后从ALB的外网地址测试访问。
