# gateway-load-balacer-in-single-vpc-quickstart

GWLB in single VPC Quickstart CloudFormation Template

## 一、架构说明

### 1、GWLB简介

关于AWS GWLB，之前构建过一个[结合Transit Gateway的多VPC的GWLB方案](https://github.com/aobao32/gateway-load-balacer-centralized-solution-quickstart)。本次构建的quickstart则是一个更简化的单VPC方案，以满足特定场景下的要求。注意是特定场景下的需求，如果是全新设计，建议按照GWLB最佳实践的架构进行组网。

本方案构建一个单VPC的网络检测方案。

主要组件如下：

- Virtual Appliance：GWLB需要搭配Marketplaces上启动的第三方商用防火墙（NGFW），需要防火墙支持Geneve协议。本Quickstart中，使用EC2运行Geneve proxy程序模拟防火墙；
- GWLB Endpoint：VPC的流量入口，作为路由表的下一跳接受流量；
- GWLB：将流量分发给Target Group，目标组中每个EC2是接受Geneve协议封装后发送过来的原始流量，拆包检查；
- NAT Gateway：本架构中所有对外流量的统一出口，所有VPC访问外部互联网访问均汇集到此后统一出口；
- ELB：本架构中所有外部进入流量的统一入口；
- VPC：单个VPC运行所有组件。

### 2、单一VPC和独立VPC的架构选择

本CloudFormation包含两套模版，第一套模版在`single-vpc`目录中，即应用系统和GWLB共用同一个VPC。第二套模版在`separated-vpc`目录，应用系统和GWLB Endpoint在第一个VPC，GWLB和Virtual Appliance在第二个VPC。

根据场景需要二选一即可。二者架构图如下。

#### 如下为架构图1（对应目录`single-vpc`内的模版）：

![总体架构图和路由表说明](https://d51vuyprlknbq.cloudfront.net/GWLB/GWLB-in-single-VPC-Quickstart.png)

#### 如下为架构图2（对应目录`separated-vpc`内的模版）：

![总体架构图和路由表说明](https://d51vuyprlknbq.cloudfront.net/GWLB/GWLB-in-single-VPC-Quickstart-2.png)

## 二、启动模版使用说明

### 1、前提

确认如下：

- 本账号的EIP Limit大于5，新的NAT Gateway和堡垒机都将绑定EIP
- 中国区已经通过了ICP备案，可以测试从外网互联网发起的对ELB的80端口发起访问
- 本模版里边的IAM Role是按照AWS中国区编写的，如果在Global启动，请将模版`Nested-SSMRole.yml`其中的`aws-cn`替换为`aws`即可在Global区域使用

### 2、使用S3托管所有CloudFormation模版

- 下载所有模版下载到自己的AWS账户内的S3存储桶中，包含主模版和所有Nested嵌套模版
- 将所有模版文件设置为Public可访问，包含主模版和所有Nested嵌套模版
- 获取每一个模版文件可以从S3上公开访问的完整URL
- 编辑主模版 `CN-GWLB.yml` 文件，将其中嵌套调用各Nested模版的URL网址，修改为上一步生成的网址
- 最后，将修改过的主模版也上传到S3存储桶，并获取其Public访问地址

### 3、启动CloudFormation模版

进入Cloudformation界面，选择输入模版的S3地址，填入主模版 `CN-GWLB.yml` 的S3上公开的地址，然后启动继续配置模版。有三个参数必须设置，分别是：

- 模版名称
- EC2 Keypair
- 跳板机允许登陆的IP地址范围需要替换为当前上网地点的出口的公网IP（也可以后期修改）
- 其他参数不需要修改，默认参数即可启动

至此创建完成。

### 4、其他Post-configuration

本模版已经完成了GWLB和GWLB Endpoint的创建，并修改了对应子网的路由表，以确保出向去往互联网和入站流量放行。由此就不需要额外做Post-configuration了，实现了开箱即用。

注意：在中国区运行CloudFormation时候，因为CloudFormation中包含去Github下载代码，可能存在网络访问失败。如果遇到失败，着可以按照下文的说明排错，手工的从Github上下载代码，重新运行。

## 三、测试GWLB工作正常

### 1、内网经过GWLB向外访问

以下方法二选一：

#### （1）通过堡垒机的RDP远程桌面使用SSH登陆EC2访问

使用以下流程验证出向流量正常：

- 在CloudFormation模版界面找到堡垒机的EIP
- 使用RDP登陆到堡垒机上
- 从堡垒机上使用ssh或者putty登陆到内网的EC2
- 从内网EC2上执行ping命令访问外部互联网资源

#### （2）使用Session Manager登陆EC2访问

- 进入EC2控制台，选中位于内网的EC2
- 点击连接，点击Session Manager
- 从内网EC2上执行ping命令访问外部互联网资源

### 2、测试外网经过ELB和GWLB向内访问

从CloudFormation的输出界面找到ELB的外网Endpoint入口地址，将其复制下来，从VPC之外的的网络如办公室环境，使用浏览器或者curl访问。

反复刷新页面，如果交替显示`web server 1`和`web server 2`表示访问成功。

### 3、验证流量经过GWLB

通过EC2控制台上的连接按钮，使用Session Manager功能，连接到Virtual Appliance EC2上。然后执行如下命令即可：

```
tcpdump -nvv 'port 6081'
```

然后即可看到Linux Console上打出的转发的数据包，包括了在应用VPC的（10.1.x.x的源地址和目标地址），也可能看到GLWB封装后提交给Virtual Applianc虚拟设备的数据包（192.168.x.x）。

由此表示GLWB转发流量正常。

## 四、故障排除

### 1、Virtual Appliance上geneve-proxy未能正常部署/启动的排查

如果部署后遇到以下故障：


- （1）从控制台上无法使用Session Manager连接位于内网的WebApp1/WebApp2虚拟机
- （2）从跳板机上无法连接位于内网的WebApp虚拟机
- （3）外网ALB入口报告502/504错误，无法打开应用服务器
- （4）外网ALB的Target Group目标组里边显示的EC2状态不健康

如果遇到以上错误，直接原因是：Virtual Appliance上负责模拟防火墙转发流量的Geneve Proxy没有正常运行。

根本原因可能是：

- CloudFormation脚本中从github下载Geneve Proxy代码到Virtual Appliance的过程中，下载超时，未能正确下载代码，未能下载代码也就无法运行；因此需要重新下载代码；
- 代码已经从Github下载成功，也被正常运行，也过了一段时间python程序异常退出，需要手工重启python程序。

如果以上两个步骤的任意一个失效，则以上各种网络流量不正常。请按如下两个步骤检查和部署。

### 2、检查Virtual Appliance是否有Geneve proxy程序在运行并手工重启之

通过EC2控制台上的连接按钮，使用Session Manager功能，连接到Virtual Appliance EC2上。然后执行如下命令检查geneve python脚本是否在运行：

```
ps awuxf | grep python
```

如果返回结果是类似如下这样，表示geneve proxy运行正常。

```
[root@ip-192-168-102-23 ~]# ps awuxf | grep python
root       813  0.4  0.6 127304 13796 ?        S    05:14   0:07 /bin/python3 main.py
root      3276  0.0  0.0 112896   484 pts/0    S+   05:42   0:00                      \_ grep --color=auto python
[root@ip-192-168-102-23 ~]#
```

如果这个进程不存在，则表示可能程序异常退出。

此时需要先检查下`/root/geneve-proxy/`目录是否存在。如果目录已经存在，则执行如下命令重启：

```
cd /root/geneve-proxy/
nohup /bin/python3 main.py &
```

即可重新启动geneve proxy。网络流量也将转发正常。

如果`/root/geneve-proxy/`目录不存在，则表示在CloudFormation部署中，因为网络超时的关系，未能从github下载到geneve proxy源代码。那么请参考下边一个章节重新部署geneve proxy脚本。

### 3、如何手工部署Geneve Proxy

有些时候，自动部署的CloudFormation模版中从github下载Geneve Proxy可能因为跨境网络的关系访问失败，这时候本机上将不存在geneve proxy脚本，也自然无法自动启动脚本了。

此时需要手工的下载脚本。通过EC2控制台上的连接按钮，使用Session Manager功能，连接到Virtual Appliance EC2上。然后执行如下命令即可：

```
git clone https://github.com/sentialabs/geneve-proxy.git
mv geneve-proxy /root/
mv /root/geneve-proxy/config.yaml /root/geneve-proxy/config-orig.yaml
wget -O /root/geneve-proxy/config.yaml http://myworkshop.bitipcman.com/GWLB/GWLB-in-single-VPC-Quickstart-config.yaml
echo "cd /root/geneve-proxy/" >> /etc/rc.local
echo "nohup /bin/python3 main.py &" >> /etc/rc.local
chmod +x /etc/rc.local
reboot
```

以上命令部署完毕后会重启Virtual Appliance EC2。重启后，再次用Session manager登陆Virtual Appliance，通过如下命令检查geneve python脚本是否在运行：

```
ps awuxf | grep python
```

为了观察流量，可以使用如下命令查看GWLB发来的流量：

```
tcpdump -nvv 'port 6081'
```

手工配置Geneve Proxy也可以参考[这个](https://d5ubqqttnawof.cloudfront.net/video/GWLB-02-QuickstartDemo.mp4)视频演示（本视频有解说注意播放音量）。

本文完。