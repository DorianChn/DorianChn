# GitHub 仓库管理（新手版）

## 这个skill能做什么
**用命令行管理 GitHub 仓库**——创建仓库、克隆代码、提交推送、处理分支、发布 Release，全程不用打开 GitHub 网页。

## 使用场景
- 把本地代码放到 GitHub 上备份
- 从 GitHub 下载别人的项目来学习
- 日常写代码：修改 → 提交 → 推送
- 给开源项目贡献代码（fork + PR）
- 发布软件版本（Release）

## 前置要求
```bash
# 1. 安装 Git（去 https://git-scm.com/ 下载）
git --version

# 2. 配置你的身份（第一次用 Git 必须做）
git config --global user.name "你的GitHub用户名"
git config --global user.email "你的邮箱@example.com"

# 3. 生成 SSH 密钥（免密码推送）
ssh-keygen -t ed25519 -C "你的邮箱@example.com" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
# 复制输出的内容，粘贴到 GitHub 设置 → SSH and GPG keys → New SSH key
```

## 快速开始

### 1. 克隆别人的仓库（下载代码）
```bash
# 克隆到当前目录
git clone https://github.com/用户名/仓库名.git

# 克隆到指定文件夹
git clone https://github.com/用户名/仓库名.git ./my-project

# 只下载最新版本（更快，适合大项目）
git clone --depth 1 https://github.com/用户名/仓库名.git
```

### 2. 把本地代码上传到 GitHub
```bash
# 先在 GitHub 上创建一个空仓库（不要勾选 README）
# 然后在本地项目目录执行：
git init
git add .
git commit -m "第一次提交"
git branch -M main
git remote add origin git@github.com:你的用户名/仓库名.git
git push -u origin main
```

### 3. 日常开发流程
```bash
# ① 查看当前状态
git status

# ② 添加修改的文件
git add 文件名.py        # 添加单个文件
git add .                # 添加所有修改

# ③ 提交到本地仓库
git commit -m "修复了某某bug"

# ④ 推送到 GitHub
git push

# ⑤ 拉取最新代码（别人推送的）
git pull
```

## 完整代码：交互式 Git 助手脚本

```python
#!/usr/bin/env python3
"""
git_helper.py - Git 命令行助手
功能：交互式选择常用 Git 操作，自动执行命令
适合：Git 新手，记不住命令的时候用
"""
import os
import subprocess
import sys


def run_cmd(cmd):
    """运行命令并显示输出"""
    print(f"\n$ {cmd}")
    result = subprocess.run(cmd, shell=True, text=True,
                            capture_output=True)
    if result.stdout:
        print(result.stdout)
    if result.returncode != 0:
        print(f"❌ 出错: {result.stderr}")
    return result.returncode == 0


def check_git():
    """检查 Git 是否已安装"""
    try:
        subprocess.run(["git", "--version"], capture_output=True)
        return True
    except FileNotFoundError:
        print("❌ 未安装 Git！请先安装：https://git-scm.com/")
        return False


def show_menu():
    """显示主菜单"""
    print("\n" + "=" * 50)
    print("  🐙 Git 助手 - 选择你要做的事")
    print("=" * 50)
    print("  1. 查看当前状态 (git status)")
    print("  2. 添加并提交 (git add + commit)")
    print("  3. 推送到远程 (git push)")
    print("  4. 拉取最新代码 (git pull)")
    print("  5. 查看提交历史 (git log)")
    print("  6. 创建新分支 (git branch)")
    print("  7. 切换分支 (git checkout)")
    print("  8. 合并分支 (git merge)")
    print("  9. 放弃修改 (git restore)")
    print("  10. 克隆仓库 (git clone)")
    print("  0. 退出")
    print("=" * 50)
    return input("请输入编号 (0-10): ").strip()


def main():
    if not check_git():
        return

    # 检查是否在 Git 仓库中
    in_repo = os.path.isdir(".git")

    while True:
        choice = show_menu()

        if choice == "0":
            print("👋 再见！")
            break

        elif choice == "1":
            # 查看状态
            run_cmd("git status")

        elif choice == "2":
            # 添加并提交
            if not in_repo:
                print("❌ 当前目录不是 Git 仓库")
                continue
            msg = input("提交信息: ").strip()
            if not msg:
                msg = "更新代码"
            run_cmd("git add .")
            run_cmd(f'git commit -m "{msg}"')

        elif choice == "3":
            # 推送
            if not in_repo:
                print("❌ 当前目录不是 Git 仓库")
                continue
            run_cmd("git push")

        elif choice == "4":
            # 拉取
            if not in_repo:
                print("❌ 当前目录不是 Git 仓库")
                continue
            run_cmd("git pull")

        elif choice == "5":
            # 查看历史
            run_cmd("git log --oneline --graph --all -10")

        elif choice == "6":
            # 创建分支
            name = input("新分支名: ").strip()
            if name:
                run_cmd(f"git branch {name}")

        elif choice == "7":
            # 切换分支
            # 先列出所有分支
            run_cmd("git branch -a")
            name = input("要切换到的分支名: ").strip()
            if name:
                run_cmd(f"git checkout {name}")

        elif choice == "8":
            # 合并分支
            name = input("要合并的分支名: ").strip()
            if name:
                run_cmd(f"git merge {name}")

        elif choice == "9":
            # 放弃修改
            print("  1. 放弃单个文件")
            print("  2. 放弃所有修改")
            sub = input("请选择 (1-2): ").strip()
            if sub == "1":
                path = input("文件路径: ").strip()
                if path:
                    run_cmd(f"git restore {path}")
            elif sub == "2":
                run_cmd("git restore .")

        elif choice == "10":
            # 克隆仓库
            url = input("仓库 URL: ").strip()
            if url:
                run_cmd(f"git clone {url}")

        else:
            print("❌ 无效输入，请输入 0-10")

        input("\n按回车继续...")
        print()


if __name__ == "__main__":
    # 切换脚本所在目录（让用户可以在任何目录运行）
    os.chdir(os.getcwd())
    main()
```

保存为 `git_helper.py`，然后运行：
```bash
python git_helper.py
```

## 常见问题

**Q: `git push` 报错 `failed to push some refs`？**
A: 远程仓库有比你新的提交，需要先 `git pull` 拉取合并，再推送。

**Q: 不小心提交了不想提交的文件怎么办？**
```bash
# 撤回最后一次提交（保留修改）
git reset --soft HEAD~1

# 彻底撤回（删除修改，慎用！）
git reset --hard HEAD~1
```

**Q: 怎么把多个提交合并成一个？**
```bash
# 合并最近 3 个提交
git rebase -i HEAD~3
# 把 pick 改成 squash 或 s
```

**Q: 如何忽略某些文件？**
A: 在项目根目录创建 `.gitignore` 文件：
```
# Python
__pycache__/
*.pyc
.env
venv/

# 系统
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
```

**Q: `git push` 要输入密码很烦？**
A: 用 SSH 方式克隆（`git@github.com:...` 格式），配置好 SSH 密钥后就不用输密码了。

**Q: 误删了文件怎么恢复？**
```bash
# 恢复已删除但还没提交的文件
git restore 文件名

# 从历史版本恢复
git checkout HEAD~1 -- 文件名
```

## 进阶用法

### 1. 给开源项目贡献代码（Fork + PR）
```bash
# ① 在 GitHub 网页上点击 Fork（复制别人的仓库到你名下）
# ② 克隆你的 Fork
git clone git@github.com:你的用户名/原仓库名.git
cd 原仓库名

# ③ 添加上游仓库（原作者的）
git remote add upstream git@github.com:原作者/原仓库名.git

# ④ 创建新分支写代码
git checkout -b my-feature

# ⑤ 写代码... 然后提交推送
git add .
git commit -m "添加了某某功能"
git push origin my-feature

# ⑥ 去 GitHub 你的仓库页面，点击 Pull Request 按钮
```

### 2. 发布 Release（版本发布）
```bash
# 打标签
git tag v1.0.0 -m "第一个正式版本"

# 推送标签到 GitHub
git push origin v1.0.0

# 推送到 GitHub 后，仓库页面会自动生成 Release
# 也可以手动上传安装包
```

### 3. 同时维护多个版本
```bash
# 从 v1.0 分支创建修复分支
git checkout -b hotfix v1.0

# 修复bug后推送到两个分支
git push origin hotfix
git checkout main
git merge hotfix
git push origin main
```

## 参考资源
- [Git 官方文档（中文）](https://git-scm.com/book/zh/v2)
- [GitHub 入门指南](https://guides.github.com/)
- [Learn Git Branching（交互式学习）](https://learngitbranching.js.org/?locale=zh_CN)
- [Git Flight Rules（常见问题速查）](https://github.com/k88hudson/git-flight-rules/blob/master/README_zh-CN.md)