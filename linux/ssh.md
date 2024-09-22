---
title: "SSH密钥登陆"
date: 2024-09-08
draft: true
tags: ["Linux", "configuration"]
categories: ["Learning"]
---

## 阿里云

尝试使用windows客户端登录阿里云服务器，参考了[10分钟手把手教你通过SSH，使用密钥/账号远程登录Linux服务器（Windows/macOS）](https://www.bilibili.com/video/BV1cL411w7RB/?share_source=copy_web&vd_source=0d697b39bd28b6cf6923ec4eb9bd05f4)

### 客户端生成密钥

进入到`C:\Users\用户名\.ssh`目录下，若没有公钥和私钥文件，则进行生成。

执行`ssh-keygen`命令，之后一路回车，然后在该目录下会出现`ida_rsa`和`ida_rsa.pub`两个文件，分别存放私钥和公钥。

### 部署公钥

使用密码登录服务器，在用户目录下进入到`.ssh`文件夹，创建或编辑`authorized_keys`文件，将上一步获得的公钥内容复制并追加到该文件中。

## 中科大vlab

[Vlab实验中心](https://vlab.ustc.edu.cn/)，参考[帮助文档](https://vlab.ustc.edu.cn/docs/login/ssh/)

### 下载私钥

进入到管理界面，下载私钥`vlab-vm9164.pem`。

### 本地配置

将该私钥文件放置到本地`C:\Users\用户名\.ssh`文件夹之下，并重命名为`vlab.pem`。

然后即可以在 cmd 中使用`ssh -i ~/.ssh/vlab.pem ubuntu@vlab.ustc.edu.cn`来登录到远程主机。

### 配置文件

如果需要使用vscode来进行操作，则可以使用SSH配置文件。进入到`C:\Users\用户名\.ssh`文件夹之下，创建或编辑`config`文件。然后最好使用 vscode 打开该文件并编辑，追加以下内容：

```text
Host vlab
    HostName vlab.ustc.edu.cn
    User ubuntu
    IdentityFile ~/.ssh/vlab.pem
```

之后在vscode中就可以方便地进行连接了。
