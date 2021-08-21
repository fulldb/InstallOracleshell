@[TOC](目录)

# 前言

使用 vagrant 的前提是要有 box 镜像盒子来初始化系统，网上有很多 box 可以下载，但是用自己的不是更香吗？自己动手，丰衣足食！

# 环境准备

## 软件准备

首先需要安装 vagrant + virtualbox + packer ，具体安装教程，请参考文章：[☀️ 福利向：⚡️万字图文⚡️ 带你 Vagrant 从入门到超神！❤️](https://www.modb.pro/db/88457)

## 下载系统镜像

下载 centos 8.3 安装包，下载地址：[精心整理Linux各版本安装包（包括Centos、Redhat、Oracle Linux），附下载链接🔗](https://www.modb.pro/db/83965)

![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-09def89e-96d8-42cc-aaa7-a04429917ac2.png)

**这里的校验码记录一下：** `aaf9d4b3071c16dbbda01dfe06085e5d0fdac76df323e3bbe87cce4318052247`

## 下载打包源码

下载打包模板源码：

```bash
git clone https://hub.fastgit.org/chef/bento.git
```
![](https://oss-emcsprod-public.modb.pro/image/editor/20210818-77809e73-6408-4a28-9fd9-8ee2297ccabd.png)

将系统镜像文件拷贝至 `bento/packer_templates/centos` 目录下：

![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-fa07a298-37a9-4db8-b575-8ff7c36d6874.png)

**<font color='green'>确认环境准备好之后，可以开始进行打包。</font>**

# 开始打包

## 自定义json文件

使用目录中的 `centos-8.3-x86_64.json` 文件，复制为 `centos83.json` ，进行自定义修改：

```json
{
  "builders": [
    {
      "boot_command": "{{ user `boot_command` }}",
      "boot_wait": "5s",
      "cpus": "{{ user `cpus` }}",
      "disk_size": "{{user `disk_size`}}",
      "guest_additions_path": "VBoxGuestAdditions_{{.Version}}.iso",
      "guest_additions_url": "{{ user `guest_additions_url` }}",
      "guest_os_type": "RedHat_64",
      "hard_drive_interface": "sata",
      "headless": "{{ user `headless` }}",
      "http_directory": "{{user `http_directory`}}",
      "iso_checksum": "{{user `iso_checksum`}}",
      "iso_url": "{{user `mirror`}}/{{user `mirror_directory`}}/{{user `iso_name`}}",
      "memory": "{{ user `memory` }}",
      "output_directory": "{{ user `build_directory` }}/packer-{{user `template`}}-virtualbox",
      "shutdown_command": "echo 'vagrant' | sudo -S /sbin/halt -h -p",
      "ssh_password": "vagrant",
      "ssh_port": 22,
      "ssh_timeout": "10000s",
      "ssh_username": "vagrant",
      "type": "virtualbox-iso",
      "virtualbox_version_file": ".vbox_version",
      "vm_name": "{{ user `template` }}"
    }
  ],
  "post-processors": [
    {
      "output": "{{ user `build_directory` }}/{{user `box_basename`}}.{{.Provider}}.box",
      "type": "vagrant"
    }
  ],
  "provisioners": [
    {
      "environment_vars": [
        "HOME_DIR=/home/vagrant",
        "http_proxy={{user `http_proxy`}}",
        "https_proxy={{user `https_proxy`}}",
        "no_proxy={{user `no_proxy`}}"
      ],
      "execute_command": "echo 'vagrant' | {{.Vars}} sudo -S -E sh -eux '{{.Path}}'",
      "expect_disconnect": true,
      "scripts": [
        "{{template_dir}}/scripts/update.sh",
        "{{template_dir}}/../_common/motd.sh",
        "{{template_dir}}/../_common/sshd.sh",
        "{{template_dir}}/scripts/networking.sh",
        "{{template_dir}}/../_common/vagrant.sh",
        "{{template_dir}}/../_common/virtualbox.sh",
        "{{template_dir}}/../_common/vmware.sh",
        "{{template_dir}}/../_common/parallels.sh",
        "{{template_dir}}/scripts/cleanup.sh",
        "{{template_dir}}/../_common/minimize.sh"
      ],
      "type": "shell"
    }
  ],
  "variables": {
    "box_basename": "centos8.3",
    "build_directory": "../../builds",
    "build_timestamp": "{{isotime \"20210820114500\"}}",
    "cpus": "2",
    "disk_size": "65536",
    "git_revision": "__unknown_git_revision__",
    "guest_additions_url": "",
    "headless": "",
    "http_directory": "{{template_dir}}/http",
    "http_proxy": "{{env `http_proxy`}}",
    "https_proxy": "{{env `https_proxy`}}",
    "hyperv_generation": "1",
    "hyperv_switch": "bento",
    "iso_checksum": "aaf9d4b3071c16dbbda01dfe06085e5d0fdac76df323e3bbe87cce4318052247",
    "iso_name": "CentOS-8.3.2011-x86_64-dvd1.iso",
    "ks_path": "8/ks.cfg",
    "memory": "2048",
    "mirror": "",
    "mirror_directory": "Volumes/DBA/voracle/bento/packer_templates/centos",
    "name": "centos8.3",
    "no_proxy": "{{env `no_proxy`}}",
    "template": "centos-8.3-x86_64",
    "boot_command": "<up><wait><tab> inst.text inst.ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/{{user `ks_path`}}<enter><wait>",
    "version": "TIMESTAMP"
  }
}
```

**<font color='red'>📢 注意：以下修改两个脚本，提前排坑。</font>**

## 修改 networking.sh 脚本

脚本位于 centos 中，`../centos/scripts/networking.sh`，由于无法访问 github ，因此 /etc/hosts 需要增加 github ip：

```bash
# modify by luciferliu for github raw.githubusercontent.com:443; Connection refused
echo '185.199.108.133 raw.githubusercontent.com' >>/etc/hosts;
echo '185.199.109.133 raw.githubusercontent.com' >>/etc/hosts;
echo '185.199.110.133 raw.githubusercontent.com' >>/etc/hosts;
echo '185.199.111.133 raw.githubusercontent.com' >>/etc/hosts;

echo '54.186.51.210 vault.centos.org' >>/etc/hosts;
echo '3.22.185.178 vault.centos.org' >>/etc/hosts;
echo '34.253.151.233 vault.centos.org' >>/etc/hosts;

ping raw.githubusercontent.com -c 5
ping vault.centos.org -c 5
```

![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-8435ff7f-84c4-4867-8137-22723d6d9b35.png)

## 修改 vagrant.sh 脚本

脚本位于 `bento/packer_templates/_common` 目录下，由于未关闭防火墙，443端口无法访问，因此一直报错，手动关闭防火墙：

```bash
pubkey_url="https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub";
mkdir -p $HOME_DIR/.ssh;

# modify by luciferliu ,443 port is close, stop firewalld.service
RELS=$(cat /etc/system-release)
OS_VER_PRI=$(echo "${RELS#*release}" | awk '{print $1}' | cut -f 1 -d '.')
if [ "${OS_VER_PRI}" -eq 6 ]; then
    service iptables stop
    if command -v curl >/dev/null 2>&1; then
        curl --insecure --location "$pubkey_url" > $HOME_DIR/.ssh/authorized_keys;
    elif command -v fetch >/dev/null 2>&1; then
        fetch -am -o $HOME_DIR/.ssh/authorized_keys "$pubkey_url";
    elif command -v wget >/dev/null 2>&1; then
        wget --no-check-certificate "$pubkey_url" -O $HOME_DIR/.ssh/authorized_keys;
    else
        echo "Cannot download vagrant public key";
        exit 1;
    fi
elif [ "${OS_VER_PRI}" -eq 7 ] || [ "${OS_VER_PRI}" -eq 8 ]; then
    systemctl stop firewalld.service
    if command -v wget >/dev/null 2>&1; then
        wget --no-check-certificate "$pubkey_url" -O $HOME_DIR/.ssh/authorized_keys;
    elif command -v curl >/dev/null 2>&1; then
        curl --insecure --location "$pubkey_url" > $HOME_DIR/.ssh/authorized_keys;
    elif command -v fetch >/dev/null 2>&1; then
        fetch -am -o $HOME_DIR/.ssh/authorized_keys "$pubkey_url";
    else
        echo "Cannot download vagrant public key";
        exit 1;
    fi
fi

```
![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-b25bf525-1c02-4116-8dd6-42c89f0a3972.png)

## 启动 packer 进行打包

```bash
packer build -only=virtualbox-iso centos83.json
```

![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-44916970-d52d-4494-8b93-d37b5ddcb319.png)

![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-e2362c41-8c40-4048-a493-a941dfe7978b.png)

显示如上，即已经打包成功，box 位置存放在：`../../builds/centos8.3.virtualbox.box` 。

![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-e2cafc07-e930-4bfe-ae96-fe944be3e342.png)

# 上传 box 镜像

不做演示，比较简单。

![](https://oss-emcsprod-public.modb.pro/image/editor/20210820-65ded28c-e8b4-4d97-9b6d-ae53d8518e0d.png)

**box镜像下载地址：[luciferliu/centos8.3](https://app.vagrantup.com/luciferliu/boxes/centos8.3)**

# 写在最后

为什么要打包 box 镜像盒子？ 以后可以使用 vagrant 直接初始化创建 linux 系统，不需要再一步步创建，为自动化奠定基础。

---
本次分享到此结束啦~

如果觉得文章对你有帮助，<font color='red'>**点赞、收藏、关注、评论**</font>，一键四连支持，你的支持就是我创作最大的动力。

技术交流可以 关注公众号：**Lucifer三思而后行**

![Lucifer三思而后行](https://img-blog.csdnimg.cn/20210702105616339.jpg)