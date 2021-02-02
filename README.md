## 一、sudo漏洞说明

监测到sudo堆溢出漏洞（CVE-2021-3156），成功利用此漏洞，任何没有特权的用户都可以在易受攻击的主机上获得root特权，需要将sudo版本更新至1.8.23-10及以上版本。



## 二、环境说明

|    主机名     |  操作系统版本   |      ip      | gcc版本 | sudo版本 |       备注        |
| :-----------: | :-------------: | :----------: | :-----: | -------- | :---------------: |
| ansible-tower | Centos 7.6.1810 | 172.16.7.100 |    /    | /        | ansible管理服务器 |
|      157      | Centos 7.6.1810 | 172.16.7.150 |  4.8.5  | 1.8.23   |    被管服务器     |
|      158      | Centos 7.6.1810 | 172.16.7.158 |    /    | 1.8.23   |    被管服务器     |

## 三、漏洞修复方式

> - yum方式
> - 源码方式

yum方式需先更新yum源，然后直接执行`yum install sudo`即可；源码方式需下载对应的源文件然后编译安装，本文重点介绍源码方式。

安装包下载

目前最新的稳定版本为1.9.5，下载地址为：https://www.sudo.ws/dist/sudo-1.9.5p2.tar.gz

## 四、yaml文件说明

```bash
---
- hosts: "{{ hostlist }}"
  gather_facts: no
  tasks:
  - name: gcc check
    shell:
      gcc -v
    register: gcc
    ignore_errors: true

  - name: install gcc
    yum:
      name=gcc
      state=present
    when: gcc.rc != 0

  - name: Unarchive sudo 
    unarchive:
      src: /tmp/sudo-1.9.5p2.tar.gz 
      dest: /root
      mode: 0755
      owner: root
      group: root

  - name: install sudo
    shell: |
      cd /root/sudo-1.9.5p2/
      ./configure --prefix=/usr --libexecdir=/usr/lib --with-secure-path --with-all-insults --with-env-editor --docdir=/usr/share/doc/sudo-1.9.5p2 --with-passprompt="[sudo] password for 
%p: "
      make && make install && ln -sfv libsudo_util.so.0.0.0 /usr/lib/sudo/libsudo_util.so.0
```

![image-20210201170924497](https://i.loli.net/2021/02/02/wP9gndbBTrFkxAM.png)

- 'gcc check'：检查gcc是否安装，如果未安装则忽略报错让安装进程继续；
- 'install gcc'：安装gcc，当gcc安装的检查结果不为0即未安装gcc时进行gcc的安装；
- 'Unarchive sudo'：解压安装包并上传到目标服务器/root目录；
- 'install sudo'：进行sudo源码安装；



## 五、执行过程及验证

```bash
[root@ansible-tower ansible]# ansible-playbook sudo.yaml -e hostlist=all
[root@ansible-tower ansible]# ansible -m shell -a "sudo --version|grep 'Sudoers audit plugin version'" all
```

![image-20210201171722107](https://i.loli.net/2021/02/02/LKvtlVnDipH14BU.png)

完成两台服务器sudo升级，版本为1.9.5
