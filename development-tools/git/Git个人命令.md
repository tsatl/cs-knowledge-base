# Git SSH 配置

## 1. 生成 SSH 密钥

```powershell
ssh-keygen -t ed25519 -C "你的邮箱"
```

默认生成：

```text
~/.ssh/id_ed25519       # 私钥，禁止泄露
~/.ssh/id_ed25519.pub   # 公钥，可上传 GitHub
```

------

## 2. 查看公钥

PowerShell：

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

复制完整输出内容。

------

## 3. 添加到 GitHub

进入：

```text
GitHub
→ Settings
→ SSH and GPG keys
→ New SSH key
```

配置：

```text
Title：自定义，如 Windows-PC
Key type：Authentication Key
Key：粘贴 id_ed25519.pub 内容
```

------

## 4. 测试 SSH 连接

```powershell
ssh -T git@github.com
```

首次连接输入：

```text
yes
```

成功时会看到：

```text
You've successfully authenticated
```

------

## 5. 配置 Git 用户信息

```powershell
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

查看配置：

```powershell
git config --global user.name
git config --global user.email
```

查看全部 Git 配置：

```powershell
git config --list
```

------

## 6. 使用 SSH 仓库地址

克隆仓库：

```powershell
git clone git@github.com:用户名/仓库名.git
```

已有仓库切换为 SSH：

```powershell
git remote set-url origin git@github.com:用户名/仓库名.git
```

查看远程仓库：

```powershell
git remote -v
```

------

## 速记

```text
ssh-keygen
    ↓
生成公钥和私钥
    ↓
上传公钥到 GitHub
    ↓
ssh -T 测试
    ↓
配置 Git user.name / user.email
    ↓
使用 git@github.com:... SSH 地址
```

注意：

```text
id_ed25519.pub → 公钥，可以上传
id_ed25519     → 私钥，不能泄露
```





# Git 常用项目推送命令





