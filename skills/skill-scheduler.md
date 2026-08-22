# Python 定时任务（Task Scheduler）

## 这个skill能做什么
用Python让程序按指定时间自动运行——每天、每小时、每周定时执行任何Python函数，实现全自动工作流。

## 使用场景
- **日报自动生成**：每天早上9点自动抓取数据生成日报
- **数据备份**：每天凌晨自动备份数据库/文件
- **价格监控**：每小时自动检查商品价格（配合skill-price-monitor）
- **消息提醒**：定时发送微信/邮件提醒（喝水、打卡、交周报）
- **爬虫定时抓取**：每周定时抓取网页数据存库
- **文件清理**：定期删除临时文件、归档旧文件

## 前置要求
```bash
# 只需要一个库，Python 3.7+
pip install schedule
```

## 快速开始

### 1. 写一个最简单的定时任务
```python
import schedule
import time

def job():
    print("任务执行了！")

# 每10秒执行一次（测试用）
schedule.every(10).seconds.do(job)

while True:
    schedule.run_pending()   # 检查有没有到点的任务
    time.sleep(1)            # 每秒检查一次，别写太快
```

### 2. 运行
```bash
python timer.py
```
看到每10秒打印一次"任务执行了！"就成功了。

## 完整代码

创建一个 `daily_report.py` 文件（可直接运行）：

```python
"""
每日定时任务示例 - 每天早上9点自动生成日报
用法: python daily_report.py
"""
import schedule
import time
from datetime import datetime

# ========== 1. 定义要定时执行的任务函数 ==========

def generate_report():
    """生成日报：记录当前时间，写入文件"""
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    report = f"【日报】{now}\n今天完成了N件事：\n1. 工作事项A\n2. 学习事项B\n"
    # 写入文件（追加模式）
    with open("daily_report.txt", "a", encoding="utf-8") as f:
        f.write(report + "-" * 30 + "\n")
    print(f"[{now}] 日报已生成 → daily_report.txt")

def backup_files():
    """备份任务：把指定目录复制到备份目录"""
    import shutil, os
    src = "data"          # 要备份的文件夹
    dst = f"backup_{datetime.now().strftime('%Y%m%d')}"
    if os.path.exists(src):
        shutil.copytree(src, dst, dirs_exist_ok=True)
        print(f"备份完成 → {dst}")

# ========== 2. 设置定时规则（全部常用写法） ==========

# 按间隔
schedule.every(10).minutes.do(backup_files)          # 每10分钟
schedule.every().hour.do(generate_report)            # 每小时
schedule.every().day.at("09:00").do(generate_report) # 每天9点（最常用）

# 按星期
schedule.every().monday.at("09:30").do(generate_report)      # 每周一9:30
schedule.every().wednesday.at("13:15").do(backup_files)      # 每周三13:15

# 按日期
schedule.every().day.at("23:59").do(generate_report)         # 每天23:59

# ========== 3. 主循环（程序会一直运行） ==========

if __name__ == "__main__":
    print("⏰ 定时任务已启动，按 Ctrl+C 停止...")
    while True:
        schedule.run_pending()   # 执行所有到点的任务
        time.sleep(1)            # 睡1秒再检查，避免CPU空转
```

### 一次性任务（到期自动退出）
```python
import schedule, time
from datetime import datetime, timedelta

def job():
    print("倒计时结束，任务执行！")
    return schedule.CancelJob   # 执行完自动取消

# 10秒后执行一次，然后程序自动结束
schedule.every(10).seconds.do(job)
while schedule.jobs:            # 还有任务就继续跑
    schedule.run_pending()
    time.sleep(1)
print("所有任务完成，程序退出")
```

## 常见问题

**Q1: 任务到点了没执行？**
检查主循环有没有写。`schedule` 库必须在 `while True` 循环里不断调 `run_pending()`，只定义不循环 = 不执行。

**Q2: 程序关闭后任务就没了？**
对，`schedule` 是进程内调度，程序一停任务全停。要7x24小时运行：Windows 用任务计划程序，或部署到服务器/树莓派。

**Q3: 任务执行太慢，错过了下一个时间点？**
任务函数是串行执行的。慢任务之间要隔开时间，或用多线程：
```python
import threading
def job_thread():
    threading.Thread(target=job, daemon=True).start()
schedule.every(5).seconds.do(job_thread)
```

**Q4: 时间格式写错了？**
时间必须带引号：`schedule.every().day.at("09:00")`，且小时是24小时制，"09:00"不能写成"9:00"。

**Q5: 想设置"每2天"？**
`schedule.every(2).days.do(job)` 即可。

**Q6: 运行时报 `schedule` 模块找不到？**
`pip install schedule` 没装成功，重装：`python -m pip install schedule`。

## 进阶用法

**1. 加日志，出错可查**
```python
import logging
logging.basicConfig(level=logging.INFO,
    filename="scheduler.log", encoding="utf-8")
schedule.every().hour.do(generate_report)
```

**2. 任务失败自动重试 + 通知**
```python
def safe_run(func):
    def wrapper():
        try:
            func()
        except Exception as e:
            print(f"❌ 任务失败: {e}")   # 这里可以改成发微信/邮件
    return wrapper

schedule.every().day.at("09:00").do(safe_run(generate_report))
```

**3. 开机自启（Windows）**
把启动命令放进 `启动` 文件夹：`Win+R` → 输入 `shell:startup` → 创建 `run_timer.bat`：
```bat
@echo off
cd /d C:\你的项目目录
python daily_report.py
```

**4. 复杂调度用 APScheduler（支持持久化、跨进程）**
```bash
pip install apscheduler
```
```python
from apscheduler.schedulers.blocking import BlockingScheduler
sched = BlockingScheduler()
sched.add_job(generate_report, "cron", hour=9, minute=0)  # 每天9点
sched.add_job(backup_files, "interval", minutes=30)       # 每30分钟
sched.start()
```

## 参考资源
- schedule官方文档: https://schedule.readthedocs.io/
- APScheduler官方文档: https://apscheduler.readthedocs.io/
- Python时间处理教程: https://docs.python.org/zh-cn/3/library/datetime.html
- 本目录配套skill: skill-file-automation.md（定时+文件处理=全自动整理）
