# Git的基础使用教学
贡献者：何文轩
## 使用Git的原因及简单介绍
>1. 版本管理
可以跟踪代码历史，每次修改都有记录，方便回溯和恢复。
可以随时回退到之前的版本，防止误操作导致数据丢失。
> 2. 团队协作
多人可以同时开发，不用担心代码冲突。
通过分支（Branch），不同任务可以独立开发，最后合并到主分支。
> 3. 代码备份
代码托管在 GitHub、Gitee、GitLab 等平台，不怕本地数据丢失。
任何地方都可以拉取代码，随时恢复开发环境。
> 4. 提高开发效率
多人协作更流畅，自动合并代码，减少手动修改的麻烦。
> 5. 适用于开源和企业项目
GitHub、Gitee 上的开源项目都用 Git，方便贡献代码。
企业团队普遍采用 Git 管理代码，提高协作效率。
简单来说：Git 能帮助你更安全、更高效地管理代码，并支持多人协作。

## 一、下载并安装Git
### Windows：
#### 1. 官网下载：[Git下载链接](https://git-scm.com/downloads/win)
#### 2. 安装记得换路径，不要安装到C盘。其他一路默认就可，也可以打开自动更新，添加快捷图标等。
#### 3. 安装完成后按Win+R输入cmd回车打开Windows PowerShell，输入：
```bash
git --version
```
输出版本号，代表安装成功。

## 二、配置Git
### 1. 在任意路径右键->查看更多选项->Open Git Bash Here，打开git命令终端。
### 2. 输入以下命令配置身份信息（建议使用真实信息）：
```bash
git config --global user.name "你的GitHub用户名"
git config --global user.email "你的GitHub邮箱"
```
检查配置是否成功：
```bash
git config --list
```

## 三、创建本地仓库
### 1. 在项目目录下按第二点所说流程打开git中断，或直接输入
```bash
cd 你的项目目录
```
### 2. 输入以下命令，会在文件里创建一个.git文件，出现代表创建成功
```bash
git init
```

## 四、创建GitHub仓库
### 1. 注册账号并登录GitHub
### 2. 点击 “New Repository” ，配置仓库信息后创建一个新仓库
### 3. 复制仓库的HTTPS或SSH地址，稍后会用到

## 五、连接本地仓库和远程仓库
有两种方式，分别是ssh和https，但是综合比较，ssh比https好很多，推荐使用。  
> - HTTPS 方式在每次 git push 或 git pull 时，都可能要求输入 GitHub 账户的用户名和密码（如果未配置凭据存储）。
SSH 方式则使用 SSH Key 进行身份认证，只需要配置一次，以后就能自动认证，无需手动输入密码。
> - HTTPS 认证 需要使用 GitHub 个人访问令牌（PAT），而 PAT 具有一定的权限，一旦泄露，可能带来安全隐患。
SSH 认证 通过 公私钥加密，私钥存放在本地，公钥添加到 GitHub，即使公钥泄露也无法被滥用。
只要私钥安全（不要泄露 id_rsa），就能保证 SSH 认证的安全性  
> - SSH 方式 使用 Git 协议，默认端口 22，在某些环境下比 HTTPS 方式（走 443 端口）速度更快，尤其是 大文件传输 时表现更好。   
> - HTTPS 方式 在访问私有仓库时，每次都需要身份验证，而 SSH 方式一旦设置好，访问私有仓库更方便。
在某些内网或防火墙环境下，HTTPS 可能被拦截，而 SSH 可能不受影响，可以更稳定地访问 GitHub 或其他 Git 服务器。(校园网环境https基本传不出去doge)

### HTTPS方法
```bash
git remote add origin 你的GitHub仓库地址（https地址）
git remote add origin 
git branch -M main  # 将默认分支改为 main（可选，好习惯）
```
### SSH方法
#### 1. 首先要生成一个SSH Key
```bash
ssh-keygen -t rsa -C "你的GitHub邮箱"
```
一直按回车，直到生成完成   
#### 2. 添加SSH Key
打开公钥文件，粘贴公钥并保存
```bash
cat ~/.ssh/id_rsa.pub
```
#### 3. 把公钥添加到github中   
    1. 点击右上角头像 → Settings
    2. 左侧栏点击 SSH and GPG keys
    3. 点击 New SSH key
    4. 粘贴公钥内容
#### 4. 测试是否成功
```bash
ssh -T git@github.com
```
如果成功会看到如下输出：
```
Hi 用户名! You've successfully authenticated, but GitHub does not provide shell access.
```
#### 5. 链接本地仓库和Github仓库
```bash
git remote add origin 你的GitHub仓库地址（ssh地址）
```
### 检查连接情况
输入以下命令即可看到当前仓库链接的所有远程仓库，对，一个本地仓库可以链接多个远程仓库的。
```bash
git remote -v
```
输出github仓库地址即完成连接

## 六、 提交代码
```bash
git add . #添加内容进暂存区
git commit -m "提交信息" #把暂存区内容存到本地仓库
git push origin main #把本地仓库更改推送到远程仓库main分支
```

## 七、回退版本
### 回退到上一个版本
- 保留代码修改（常用），保留代码和暂存区
```bash
git reset --soft HEAD^
```
- 丢弃修改（常用），丢弃暂存区修改，保留代码
```bash
git reset --mixed HEAD^
```
- 彻底删除（危险⚠️），回退commit+暂存+修改全清除
```bash
git reset --hard HEAD^
```
### 回退到指定版本
#### 1. 查看commit记录
```bash
git log
```
会看到如下输出：
```bash
commit a1b2c3d4e5f6g7h8... #复制这段
Author: xxx
Date: ...
Message: 修复登录问题
```
#### 2. 回退
```bash
git reset --soft a1b2c3d
```
或
```bash
git reset --mixed a1b2c3d
```
或
```bash
git reset --hard a1b2c3d
```
### 撤销回退
#### 1. 查看reflog
```bash
git reflog
```
会列出所有的操作记录
```bash
a1b2c3d HEAD@{0}: reset: moving to HEAD^
e5f6g7h HEAD@{1}: commit: 修复登录问题
```
#### 2. 滚到指定操作记录
```bash
git reset --hard e5f6g7h
```

## 八、 新建分支
| 优点 | 说明 |
| --- | :---: |
|✅ 避免冲突|每人开发自己的功能，互不干扰。|
|✅ 更清晰 | 分支命名可以反映开发内容 |
|✅ 易于管理 | 可以单独审核每个人的代码 |
|✅ 回退方便	| 如果某个人的功能不稳定，可以只回滚或暂停该分支 |
具体操作为：
```bash
git switch -c feature/mywork #创建并切换到新分支
```
查看是否成功：
```bash
git branch -vv
```
如果成功会有如下输出：
```
* feature/mywork  123abc [origin/feature/mywork] commit message
```
## 九、多人协作开发
基本规定：
- main ：主分支，保证是稳定版本
- dev ：开发分支，用于开发新的功能
- feature/xxxx ：功能分支，用于开发新的功能
- hotfix/xxxx ：bug修复分支，用于修复bug

### 1. 克隆主仓库
```bash
git clone 仓库地址
```
### 2. 切换到dev分支
```bash
git checkout dev #切换到dev分支
git pull origin dev #拉取远程dev分支到本地dev分支
```
### 3. 创建自己的功能分支
```bash
git switch -c feature/xxxx #创建并切换到新分支
```
### 4. 开发自己的功能，并提交代码到feature/xxxx分支,推送到远端分支
### 5. 合并到dev分支（由管理员/审核人执行）  
可以在远程仓库直接操作或在本地合并：
```bash
git checkout dev #切换到dev分支
git pull origin dev #拉取远程dev分支到本地dev分支
git merge feature/xxxx #合并feature/xxxx分支到dev分支
git push origin dev #把dev分支推送到远程dev分支
```
### 6. 删除已完成分支
```bash
git branch -d feature/xxxx #删除本地分支
git push origin --delete feature/xxxx #删除远端分支
```

## 附加
### VsCode推送代码
图形化界面，易于操作，原理同终端。
1. 把要保留更改的文件添加到暂存区
2. 写提交信息，把暂存区内容加入本地仓库
3. 点击发布，把本地仓库修改推送到远程仓库
### 标准格式
之前教的虽然够用，但其实每次推送提交的信息格式都不是很标准，用命令行操作过于复杂，可以使用VsCode的“git-commit-pligin”插件实现，具体过程不在赘述，请自行查阅插件说明

## 结束总结
以上内容只是Git的最基础的使用，也只是Git的冰山一角，Git还有很多很多功能没介绍到，大家可以自己去发现，之后发现好用的功能会不定时更新此文档   
