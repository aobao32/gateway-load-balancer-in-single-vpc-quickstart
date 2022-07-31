# gateway-load-balacer-in-single-vpc-quickstart

GWLB in single VPC Quickstart CloudFormation Template

## 一、架构说明

本方案构建一个单VPC的网络检测方案，可参考[之前多VPC的GWLB方案](https://github.com/aobao32/gateway-load-balacer-centralized-solution-quickstart)。

主要组件如下：

- Virtual Appliance：GWLB需要搭配Marketplaces上启动的第三方商用防火墙（NGFW），需要防火墙支持Geneve协议。本Quickstart中，使用EC2运行Geneve proxy程序模拟防火墙；
- GWLB Endpoint：VPC的流量入口，作为路由表的下一跳接受流量；
- GWLB：将流量分发给Target Group，目标组中每个EC2是接受Geneve协议封装后发送过来的原始流量，拆包检查；
- NAT Gateway：本架构中所有对外流量的统一出口，所有VPC访问外部互联网访问均汇集到此后统一出口；
- ELB：本架构中所有外部进入流量的统一入口；
- VPC：单个VPC运行所有组件。

总体架构图和路由表说明如下。

![总体架构图和路由表说明](https://d51vuyprlknbq.cloudfront.net/GWLB/GWLB-in-single-VPC-Quickstart.png)

## 二、启动模版使用说明

### 1、前提

确认如下：

- 本账号的EIP Limit大于5，新的NAT Gateway和堡垒机都将绑定EIP
- 中国区已经通过了ICP备案，可以测试从外网互联网发起的对ELB的80端口发起访问
- 本模版里边的IAM Role是按照AWS中国区编写的，如果在Global启动，请将模版`Nested-SSMRole.yml`其中的`aws-cn`替换为`aws`即可在Global区域使用。

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

本模版已经完成了GWLB和GWLB Endpoint的创建，并修改了对应子网的路由表，以确保出向去往互联网和入站

## 三、测试内网经过GWLB向外访问

以下方法二选一：

### 1、通过堡垒机使用SSH登陆EC2访问

使用以下流程验证出向流量正常：

- 在CloudFormation模版界面找到堡垒机的EIP
- 使用RDP登陆到堡垒机上
- 从堡垒机上使用ssh或者putty登陆到内网的EC2
- 从内网EC2上执行ping命令访问外部互联网资源

### 2、使用Session Manager登陆EC2访问

- 进入EC2控制台，选中位于内网的EC2
- 点击连接，点击Session Manager
- 从内网EC2上执行ping命令访问外部互联网资源

## 四、测试外网经过ELB和GWLB向内访问

从CloudFormation的输出界面找到ELB的外网Endpoint入口地址，将其复制下来，从VPC之外的的网络如办公室环境，使用浏览器或者curl访问。

反复刷新页面，如果交替显示`web server 1`和`web server 2`表示访问成功。

## 五、故障排除

### 1、从控制台上无法使用Session Manager连接位于内网的WebApp虚拟机

可能原因：EC2 Session Manager正常运行的前提是EC2具有正常的出向流量，这个出向流量是经过GWLB的。如果Virtual Appliance上负责模拟防火墙转发流量的Geneve Proxy失效，可能是CloudFormation脚本中从github下载代码时候超时，也可能是python程序异常退出，需要手工重启python程序；

再重新调整的Virtual Appliance后，位于内网的EC2可能无法感知到Session Manager所需要的出向连接已经恢复。此时重启下Session Manager要连接的EC2即可正常使用Session Manager连接。

### 2、从跳板机上无法连接位于内网的WebApp虚拟机

可能原因：跳板机上访问内网的Ping和SSH流量也需要经过GWLB的。如果Virtual Appliance上负责模拟防火墙转发流量的Geneve Proxy失效，可能是CloudFormation脚本中从github下载代码时候超时，也可能是python程序异常退出，需要手工重启python程序。

### 3、外网访问ELB失败

可能原因：外网ELB的流量先来到NAT Gateway所在的子网，然后通过目标组的健康检查，再发送流量到目标组中的EC2。ELB健康检查的流量和转发到目标组的流量是需要经过GWLB的。如果Virtual Appliance上负责模拟防火墙转发流量的Geneve Proxy失效，可能是CloudFormation脚本中从github下载代码时候超时，也可能是python程序异常退出，需要手工重启python程序。

### 4、验证 Geneve Proxy 工作正常

在模版启动后，在EC2中可以看到带有标签名为`Virtual Appliance1`和`VirtualAppliance2`的两个EC2。这两个节点运行Amazon Linux 2，且已经从github上下载了geneve proxy到 `/root/geneve-proxy` 目录。

本CloudFormation通过EC2的`User Data`进行了如下配置：

- 从Github上下载geneve-proxy
- 通过/etc/rc.local随机自动启动

如果以上两个步骤的任意一个失效，则以上各种网络流量不正常。

### 5、如何手工部署Geneve Proxy

在需要重新配置Geneve Proxy的时候，请参考[这个](https://d5ubqqttnawof.cloudfront.net/video/GWLB-02-QuickstartDemo.mp4)视频（本视频有解说注意播放音量）。
