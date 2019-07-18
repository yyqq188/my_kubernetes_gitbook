**启动ubuntu虚拟机的时候，希望直接执行安装k8s相关组件以及镜像的工作，这就要求在Vagrantfile中明确写出，先来说明下目录结构**

>  install_k8s_dir
>
> ​       k8s-libs/
>
> ​                  存放k8s的deb包以及镜像压缩包
>
> ​       install_shell.sh
>
> ​       Vagrantfile

**其中的k8s_libs包在我的[ftp服务器](ftp://39.106.163.125/pub/)中存放这，大家可以直接下载.**

---

下面开始就是install_shell.sh

我先说下思路

1. 要先创建root账户，vagrant的初始账户是vagrant
2. 执行安装，先安装docker ，kubectl，kubeadm，kubelet，再安装相关的images镜像
3. 在master节点执行kubeadm init命令，安装CNI插件的实现组件，在node节点执行join操作，并安装CNI插件的实现组件，形成k8s非高可用集群
4. 安装dashboard组件

**其中的1 和 2 是写在shell中，3 是要登录进去后手动执行**

#### 下面是install_shell.sh的一些内容

```shel
#! /bin/bash
# 创建root用户，设置密码并转入到root账户下
USER_COUNT=`cat /etc/passwd | grep '^zhangsan:' -c`
USER_NAME='root'
if [ $USER_COUNT -ne 1 ]
then
sudo useradd $USER_NAME -m 
echo "root" | sudo passwd $USER_NAME
else
echo 'user exits'
fi
sudo su root
#下面是执行安装deb包以及安装images
current_dir=`pwd`
libs=$current_dir/k8s_libs
echo "----------start install k8s base finished"
cd $libs
dpkg -i libltdl7_2.4.6-0.1_amd64.deb
dpkg -i docker.io_18.09.2-0ubuntu1~16.04.1_amd64.deb
dpkg -i kubectl_1.14.1-00_amd64.deb
dpkg -i kubernetes-cni_0.7.5-00_amd64.deb
dpkg -i socat_1.7.3.1-1_amd64.deb
dpkg -i ebtables_2.0.10.4-3.4ubuntu2.16.04.2_amd64.deb
dpkg -i conntrack_1%3a1.4.3-3_amd64.deb
dpkg -i kubelet_1.14.1-00_amd64.deb
dpkg -i cri-tools_1.12.0-00_amd64.deb
dpkg -i kubeadm_1.14.1-00_amd64.deb
echo "----------install k8s base finished"
echo "----------start k8s images install"
tar -zxvf images.tar.gz
images=$libs/image-tars/
for i in `ls $images`
 do
  docker load -i $libs/image-tars/$i
 done
echo "k8s images install finished"
rm -rf images.tar.gz
# 下面是安装dashboard的组件
....
```



#### 这里是Vagrantfile的内容，创建了3个节点的集群

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.6.0"

boxes = [
    {
        :name => "k8s-master",
        :eth1 => "192.168.205.16",
        :mem => "1024",
        :cpu => "2"
    },
    {
        :name => "k8s-slave-1",
        :eth1 => "192.168.205.17",
        :mem => "1024",
        :cpu => "2"
    },
    {
        :name => "k8s-slave-2",
        :eth1 => "192.168.205.18",
        :mem => "1024",
        :cpu => "2"
    }
]

Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu_1604"

  boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "vmware_fusion" do |v|
          v.vmx["memsize"] = opts[:mem]
          v.vmx["numvcpus"] = opts[:cpu]
        end

        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
          v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
        end

        config.vm.network :private_network, ip: opts[:eth1]
      end
  end
  #config.vm.synced_folder "/home/yhl/k8s_libs", "/home/vagrant/k8s_libs"
  config.vm.provision "shell", privileged: true, path: "./install_shell.sh" #重点看这里
end

```





#### 下一节就开始介绍镜像仓库的和gitlab的安装







​		

​                   

​        

​        