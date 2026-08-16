# Python文件批量处理自动化

> 用Python批量处理文件、自动整理文件夹、给重复劳动说拜拜

## 这个skill能做什么

用Python脚本自动处理文件：批量重命名、按类型分类、查找重复文件、监控文件夹变化。

## 使用场景

- 下载文件夹乱成一团 → 一键按文件类型归类
- 照片全是 `IMG_001.jpg` → 批量重命名成 `2024-01-01_北京.jpg`
- 磁盘空间不够 → 找出重复文件清理
- 每天要处理同类文件 → 设置文件夹监控自动处理

## 前置要求

```bash
# 只需要Python，标准库就够了！
# 如果需要监控文件夹，额外装一个
pip install watchdog
```

## 快速开始

```bash
# 1. 把下面的代码保存为 organize.py
# 2. 放到要整理的文件夹里
# 3. 运行
python organize.py
```

## 完整代码

### 1️⃣ 按文件类型自动归类（最常用）

```python
"""
功能：把文件夹里的文件按类型自动归类
使用：python organize.py
效果：所有 .jpg 进"图片"文件夹，.mp3 进"音乐"文件夹...
"""
import os
import shutil
from pathlib import Path

# 文件类型分类规则
# 键是目标文件夹名，值是文件后缀列表
FILE_TYPES = {
    "图片": [".jpg", ".jpeg", ".png", ".gif", ".bmp", ".webp"],
    "文档": [".pdf", ".doc", ".docx", ".xlsx", ".pptx", ".txt"],
    "视频": [".mp4", ".avi", ".mkv", ".mov", ".flv"],
    "音乐": [".mp3", ".wav", ".flac", ".aac"],
    "压缩包": [".zip", ".rar", ".7z", ".tar", ".gz"],
    "代码": [".py", ".js", ".html", ".css", ".java", ".cpp"],
    "安装包": [".exe", ".msi", ".dmg", ".apk"],
}

def organize_folder(folder_path):
    """按文件类型整理文件夹"""
    folder = Path(folder_path)

    # 确保文件夹存在
    if not folder.exists():
        print(f"❌ 文件夹不存在: {folder_path}")
        return

    # 统计信息
    moved = 0
    errors = 0

    for file in folder.iterdir():
        if file.is_file():  # 只处理文件，不处理文件夹
            # 获取文件后缀（小写）
            ext = file.suffix.lower()

            # 找对应的分类
            target_dir = None
            for dir_name, extensions in FILE_TYPES.items():
                if ext in extensions:
                    target_dir = dir_name
                    break

            # 没匹配到的归到"其他"
            if target_dir is None:
                target_dir = "其他"

            # 创建目标文件夹
            target_path = folder / target_dir
            target_path.mkdir(exist_ok=True)

            # 移动文件（处理重名）
            dest = target_path / file.name
            if dest.exists():
                # 重名时加数字后缀
                name = file.stem  # 文件名（不含后缀）
                count = 1
                while dest.exists():
                    dest = target_path / f"{name}_{count}{ext}"
                    count += 1

            try:
                shutil.move(str(file), str(dest))
                print(f"  ✅ {file.name} → {target_dir}/")
                moved += 1
            except Exception as e:
                print(f"  ❌ {file.name} 移动失败: {e}")
                errors += 1

    print(f"\n📊 整理完成！移动了 {moved} 个文件，{errors} 个失败")

# ----- 使用 -----
if __name__ == "__main__":
    organize_folder(".")  # 整理当前文件夹
    # organize_folder("D:/下载")  # 也可以指定路径
```

### 2️⃣ 批量重命名文件

```python
"""
功能：批量重命名文件，支持多种命名规则
使用：python rename.py
"""
import os
from pathlib import Path
import re

def batch_rename(folder_path, pattern="序号", prefix="", start=1):
    """
    批量重命名文件

    参数:
        folder_path: 文件夹路径
        pattern: 命名模式 - "序号" / "日期" / "原文件名"
        prefix: 文件名前缀
        start: 起始序号
    """
    folder = Path(folder_path)
    files = [f for f in folder.iterdir() if f.is_file()]

    print(f"📁 找到 {len(files)} 个文件，开始重命名...")

    for i, file in enumerate(files):
        ext = file.suffix  # 后缀（如 .jpg）
        old_name = file.stem  # 原名（不含后缀）

        # 生成新文件名
        if pattern == "序号":
            new_name = f"{prefix}{start + i:03d}{ext}"
        elif pattern == "原文件名":
            new_name = f"{prefix}{old_name}{ext}"
        else:
            new_name = f"{prefix}{old_name}{ext}"

        new_path = file.parent / new_name

        # 处理重名
        count = 1
        while new_path.exists() and new_path != file:
            new_name = f"{prefix}{start + i:03d}_{count}{ext}"
            new_path = file.parent / new_name
            count += 1

        file.rename(new_path)
        print(f"  {file.name} → {new_name}")

    print(f"✅ 重命名完成！")

def rename_by_pattern(folder_path, find_str, replace_str):
    """替换文件名中的指定文字"""
    folder = Path(folder_path)
    count = 0

    for file in folder.iterdir():
        if file.is_file() and find_str in file.stem:
            new_name = file.stem.replace(find_str, replace_str) + file.suffix
            file.rename(file.parent / new_name)
            print(f"  {file.name} → {new_name}")
            count += 1

    print(f"✅ 替换完成，修改了 {count} 个文件")

# ----- 使用示例 -----
if __name__ == "__main__":
    # 例1：照片按序号重命名
    batch_rename(".", pattern="序号", prefix="照片_", start=1)

    # 例2：把文件名中的"下载"替换成"素材"
    # rename_by_pattern(".", "下载", "素材")
```

### 3️⃣ 查找重复文件

```python
"""
功能：查找文件夹中的重复文件（基于文件大小+MD5）
使用：python find_duplicates.py
"""
import os
from pathlib import Path
import hashlib
from collections import defaultdict

def get_file_hash(file_path):
    """计算文件的MD5值"""
    hasher = hashlib.md5()
    # 大文件分块读取，不占内存
    with open(file_path, "rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            hasher.update(chunk)
    return hasher.hexdigest()

def find_duplicates(folder_path):
    """查找重复文件"""
    folder = Path(folder_path)
    size_map = defaultdict(list)  # 按大小分组

    print("🔍 正在扫描文件...")
    # 第一轮：按文件大小分组
    for file in folder.rglob("*"):  # rglob 递归所有子文件夹
        if file.is_file():
            size_map[file.stat().st_size].append(file)

    # 第二轮：相同大小的文件比较MD5
    duplicates = []
    total = sum(1 for files in size_map.values() if len(files) > 1)
    checked = 0

    for size, files in size_map.items():
        if len(files) < 2:
            continue

        hash_map = defaultdict(list)
        for file in files:
            file_hash = get_file_hash(file)
            hash_map[file_hash].append(file)

        # 找出MD5相同的文件
        for file_hash, same_files in hash_map.items():
            if len(same_files) > 1:
                duplicates.append(same_files)
                checked += 1

        # 进度提示
        if checked % 10 == 0:
            print(f"  已检查 {checked} 组...")

    # 输出结果
    if not duplicates:
        print("✅ 没有找到重复文件！")
        return

    print(f"\n📋 找到 {len(duplicates)} 组重复文件，共可清理 {sum(len(g)-1 for g in duplicates)} 个文件\n")

    for i, group in enumerate(duplicates, 1):
        print(f"第 {i} 组重复（{group[0].stat().st_size / 1024:.1f} KB）：")
        for j, file in enumerate(group):
            marker = " ⭐ 保留" if j == 0 else " ❌ 可删"
            print(f"  {marker}: {file}")
        print()

    # 计算总浪费空间
    wasted = sum((len(g) - 1) * g[0].stat().st_size for g in duplicates)
    print(f"💾 共浪费 {wasted / 1024 / 1024:.1f} MB 空间")

# ----- 使用 -----
if __name__ == "__main__":
    find_duplicates(".")  # 查找当前目录
    # find_duplicates("D:/下载")  # 指定文件夹
```

### 4️⃣ 监控文件夹变化（进阶）

```python
"""
功能：监控文件夹，新文件出现时自动处理
使用：需要先安装 pip install watchdog
"""
import time
from pathlib import Path

try:
    from watchdog.observers import Observer
    from watchdog.events import FileSystemEventHandler
except ImportError:
    print("请先安装: pip install watchdog")
    exit(1)

class AutoOrganizeHandler(FileSystemEventHandler):
    """当新文件出现时自动归类"""

    def on_created(self, event):
        """文件创建时触发"""
        if event.is_directory:
            return
        file_path = Path(event.src_path)
        time.sleep(1)  # 等文件写完成
        print(f"📥 检测到新文件: {file_path.name}")

        # 这里可以调用上面的 organize 函数
        # 或者自定义处理逻辑
        # 例如：如果是图片，自动压缩
        # 如果是PDF，自动提取文字
        # 这里是演示，只打印信息
        print(f"  文件大小: {file_path.stat().st_size / 1024:.1f} KB")

def watch_folder(folder_path):
    """开始监控文件夹"""
    folder = Path(folder_path)
    if not folder.exists():
        print(f"❌ 文件夹不存在: {folder_path}")
        return

    print(f"👀 正在监控文件夹: {folder_path}")
    print("按 Ctrl+C 停止监控\n")

    event_handler = AutoOrganizeHandler()
    observer = Observer()
    observer.schedule(event_handler, str(folder), recursive=False)
    observer.start()

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
        print("\n👋 监控已停止")

    observer.join()

# ----- 使用 -----
if __name__ == "__main__":
    watch_folder(".")  # 监控当前文件夹
```

## 常见问题

**Q: 运行报错 `Permission denied`**
A: 文件被其他程序占用了，关掉Word/Excel等再试

**Q: 中文文件名乱码**
A: 在代码开头加一行：`# -*- coding: utf-8 -*-`

**Q: 文件太多跑得慢**
A: 先用 `os.listdir()` 看文件数量，超1000个可以分批处理

**Q: 误操作怎么办？**
A: 所有脚本都是**移动**不是删除，去目标文件夹就能找回

**Q: 可以处理子文件夹里的文件吗？**
A: 代码里用的是 `iterdir()` 只处理当前层。要递归用 `rglob("*")` 替换 `iterdir()` 即可

## 进阶用法

- **定时清理**：配合Windows任务计划程序，每天凌晨自动整理下载文件夹
- **图片压缩**：结合 `Pillow` 库，自动压缩图片到指定大小
- **日志分析**：监控日志文件夹，新日志出现自动提取错误信息
- **备份脚本**：每天自动备份重要文件到D盘

## 参考资源

- [Python官方文档 - os模块](https://docs.python.org/3/library/os.html)
- [Python官方文档 - pathlib模块](https://docs.python.org/3/library/pathlib.html)
- [watchdog文档](https://python-watchdog.readthedocs.io/)

---

> 💡 **安全提示**：运行前建议先在小文件夹测试，确认效果后再处理重要文件