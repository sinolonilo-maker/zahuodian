# 🔧 Git 与 GitHub SSH 完整配置与踩坑修复手册

这份手册记录了在 Windows 下从零配置 Git，并通过 SSH 方式将本地仓库成功推送到 GitHub 的完整流程，以及对应的踩坑修复经验。

---

## 1. 基础配置：告诉 Git 你是谁

在安装完 Git 后，第一步是配置你的用户名和邮箱。这相当于在你的每一次代码提交（Commit）上签个名。

**操作指令（在命令行/PowerShell 中执行）：**
```bash
# 替换为你的 GitHub 用户名
git config --global user.name "Your Name"

# 替换为你的 GitHub 注册邮箱（或者是 GitHub 提供的 noreply 隐私邮箱）
git config --global user.email "your_email@example.com"
```

**检查是否配置成功：**
```bash
git config --list
```
*如果你能在这个列表里看到你的 `user.name` 和 `user.email`，说明配置成功。*

---

## 2. 核心报错现象与原因分析

在尝试用 `git push -u origin main` 将代码推送到 GitHub 时，大家最常遇到以下报错：

```text
Host key verification failed.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

### 🔴 为什么会报错？
哪怕你已经在本地生成了 SSH 密钥对（`id_rsa` 和 `id_rsa.pub`），并且已经把公钥（`id_rsa.pub`）粘贴到了 GitHub 网站的 `Settings -> SSH and GPG keys` 中，你**依然可能会遇到这个报错**。

**原因在于：Git 认生。**
当你第一次通过 SSH 协议（`git@github.com`）连接 GitHub 的服务器时，你的电脑系统（具体说是 SSH 客户端）并不认识这台服务器。出于安全考虑，系统会跳出一个提示，问你：*“我没见过这个服务器的指纹（Host Key），你确定要连吗？”*

平时我们在命令行手动输入 `ssh -T git@github.com` 时，系统会提示 `Are you sure you want to continue connecting (yes/no/[fingerprint])?`，你输入 `yes` 敲回车，系统就会把 GitHub 加入到“信任名单”（`~/.ssh/known_hosts` 文件）中，以后就能顺利通行了。

**踩坑点：** 但是，如果你的推送命令是通过自动化脚本、第三方工具或者在某些未跳出交互界面的控制台中触发的，**你根本没有机会输入这句 `yes`**，于是连接直接被拒绝，导致连接失败。

---

## 3. 终极修复方案：自动把 GitHub 加入信任名单

为了跳过这个交互式的 `yes/no` 卡点，强制让 SSH 信任未知的受信任主机，我们需要修改运行时的环境变量。

### 🛠️ 修复指令（一招解决）

在尝试推送（Push）之前，在 PowerShell（终端）中运行以下命令，临时配置 SSH 的严格主机密钥检查策略：

```powershell
$env:GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=accept-new"
git push -u origin main
```

### 💡 这行命令是什么意思？
1. `$env:GIT_SSH_COMMAND`：这是告诉 Git：“等会你底层调用 SSH 去连接时，不要用默认设置，用我指定的参数。”
2. `-o StrictHostKeyChecking=accept-new`：这是 SSH 的一个高级参数，意思是**“如果这个服务器我以前没见过（或者密钥更新了），自动接受并保存它的新公钥（指纹），不要弹出来问我”**。

一旦执行成功，你的终端会提示：
`Warning: Permanently added 'github.com' (ED25519) to the list of known hosts.`
（意思是：已经将 github.com 永久添加到了已知的信任主机列表中。）

伴随这个提示，你的代码就会顺利推送到 GitHub，大功告成！

---

## 4. 补充知识：常规的完整推送流程

如果你创建了一个新项目，完整的 Git 初始化和推送流程如下：

```bash
# 1. 在当前文件夹初始化 Git
git init

# 2. 把当前目录下的所有文件添加到暂存区
git add .
# （或者只添加指定文件：git add 你的文件.md）

# 3. 提交并写备注
git commit -m "第一次提交：初始化项目"

# 4. 确保当前分支叫 main（GitHub 默认主分支从 master 改为了 main）
git branch -M main

# 5. 绑定远端仓库地址（只需要绑定一次）
git remote add origin git@github.com:你的名字/你的仓库名.git

# 6. 【核心修复步骤】配置 SSH 自动信任并推送
$env:GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=accept-new"
git push -u origin main
```

*(备注：执行过一次 `accept-new` 或者手动输过 `yes` 之后，`github.com` 就已经被写入你电脑的 `~/.ssh/known_hosts` 文件里了。以后只要你还在用这台电脑，推送就只需要简单的 `git push`，不再需要这一长串环境变量了。)*
