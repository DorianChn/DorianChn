# 价格监控工具（Product Price Monitor）

## 这个skill能做什么
用Python自动监控商品价格，当价格降到目标价时自动提醒你，同时记录价格历史变化趋势。

## 使用场景
- **购物比价**：盯住想买的商品，等降价再入手
- **机票酒店**：监控航班和酒店价格，逢低买入
- **库存监控**：缺货商品补货时立刻通知你
- **价格趋势**：分析商品历史价格，判断最佳购买时机

## 前置要求
```bash
# 安装依赖
pip install requests beautifulsoup4 lxml
```

## 快速开始

### 1. 配置要监控的商品
在 `config.json` 中填写商品信息：
```json
{
  "products": [
    {
      "name": "商品名称",
      "url": "商品详情页URL",
      "target_price": 1000,
      "selector": ".price"  // CSS选择器，定位价格元素
    }
  ],
  "check_interval": 3600,     // 检查间隔（秒），默认1小时
  "alert_method": "console"   // 提醒方式：console/email/webhook
}
```

### 2. 运行监控脚本
```bash
python price_monitor.py
```

### 3. 查看价格历史
```bash
# 查看保存的价格记录
cat price_history.csv
```

## 完整代码

创建一个 `price_monitor.py` 文件：

```python
"""
价格监控工具 - 自动跟踪商品价格，降价提醒
用法：
  1. 创建 config.json 配置文件
  2. 运行: python price_monitor.py
  3. 程序每小时检查一次价格，降价时自动通知
"""

import requests
from bs4 import BeautifulSoup
import json
import csv
import time
import os
from datetime import datetime
import smtplib
from email.mime.text import MIMEText

# ============ 配置区 ============

# 默认配置（也可以从 config.json 读取）
DEFAULT_CONFIG = {
    "products": [
        {
            "name": "示例商品",
            "url": "https://example.com/product",
            "target_price": 1000,       # 目标价（低于此价提醒）
            "selector": ".price",        # 页面中价格元素的CSS选择器
            "currency": "¥"              # 货币符号
        }
    ],
    "check_interval": 3600,             # 检查间隔（秒）
    "alert_method": "console",          # console | email | webhook
    "email": {
        "smtp_server": "smtp.qq.com",
        "smtp_port": 587,
        "sender": "your@qq.com",
        "password": "your_smtp_code",
        "receiver": "your@qq.com"
    },
    "webhook_url": ""                   # 企业微信/钉钉机器人URL
}

# ============ 核心功能 ============

def load_config(config_file="config.json"):
    """加载配置文件"""
    if os.path.exists(config_file):
        with open(config_file, "r", encoding="utf-8") as f:
            return json.load(f)
    # 没有配置文件，创建默认配置
    with open(config_file, "w", encoding="utf-8") as f:
        json.dump(DEFAULT_CONFIG, f, ensure_ascii=False, indent=2)
    print(f"[INFO] 已创建默认配置文件: {config_file}")
    print("[INFO] 请修改配置后重新运行")
    return DEFAULT_CONFIG


def fetch_price(url, selector):
    """
    从网页抓取价格
    返回: (价格数值, 价格文本, 是否成功)
    """
    headers = {
        "User-Agent": (
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
            "AppleWebKit/537.36 (KHTML, like Gecko) "
            "Chrome/120.0.0.0 Safari/537.36"
        ),
        "Accept-Language": "zh-CN,zh;q=0.9",
    }
    try:
        resp = requests.get(url, headers=headers, timeout=15)
        resp.raise_for_status()
        resp.encoding = resp.apparent_encoding  # 自动检测编码

        soup = BeautifulSoup(resp.text, "lxml")
        # 如果选择器以 # 开头，按 ID 查找
        # 如果以 . 开头，按 class 查找
        if selector.startswith("#"):
            elem = soup.find(id=selector[1:])
        elif selector.startswith("."):
            elem = soup.find(class_=selector[1:])
        else:
            elem = soup.select_one(selector)

        if elem is None:
            return None, "未找到价格元素", False

        price_text = elem.get_text(strip=True)
        # 提取数字（支持格式: ¥1,299.00 / 1299 / 1,299.00元）
        price_num = None
        import re
        # 匹配数字（含小数点和逗号）
        numbers = re.findall(r"[\d,]+\.?\d*", price_text.replace(",", ""))
        if numbers:
            # 取第一个数字（通常是价格）
            price_num = float(numbers[0].replace(",", ""))

        return price_num, price_text, True

    except requests.RequestException as e:
        return None, f"请求失败: {e}", False
    except Exception as e:
        return None, f"解析失败: {e}", False


def save_price_history(product_name, price, currency, csv_file="price_history.csv"):
    """保存价格记录到CSV"""
    file_exists = os.path.exists(csv_file)
    with open(csv_file, "a", newline="", encoding="utf-8-sig") as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(["时间", "商品", "价格", "货币"])
        writer.writerow([
            datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            product_name,
            price,
            currency
        ])


def send_alert_console(product_name, current_price, target_price, url):
    """控制台提醒"""
    print(f"\n{'='*50}")
    print(f"🔔 降价提醒！")
    print(f"商品: {product_name}")
    print(f"当前价: {current_price}")
    print(f"目标价: {target_price}")
    print(f"链接: {url}")
    print(f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'='*50}\n")


def send_alert_email(product_name, current_price, target_price, url, config):
    """邮件提醒"""
    email_cfg = config.get("email", {})
    if not email_cfg.get("sender"):
        print("[WARN] 邮件配置不完整，跳过邮件提醒")
        return

    subject = f"🔔 降价提醒: {product_name}"
    body = f"""
商品: {product_name}
当前价格: {current_price}
目标价格: {target_price}
购买链接: {url}
检查时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

快去下手吧！
    """

    msg = MIMEText(body, "plain", "utf-8")
    msg["Subject"] = subject
    msg["From"] = email_cfg["sender"]
    msg["To"] = email_cfg["receiver"]

    try:
        server = smtplib.SMTP(email_cfg["smtp_server"], email_cfg["smtp_port"])
        server.starttls()
        server.login(email_cfg["sender"], email_cfg["password"])
        server.send_message(msg)
        server.quit()
        print(f"[OK] 邮件提醒已发送")
    except Exception as e:
        print(f"[ERR] 邮件发送失败: {e}")


def send_alert_webhook(product_name, current_price, target_price, url, webhook_url):
    """企业微信/钉钉机器人提醒"""
    if not webhook_url:
        return
    data = {
        "msgtype": "text",
        "text": {
            "content": (
                f"🔔 降价提醒\n"
                f"商品: {product_name}\n"
                f"当前价: {current_price}\n"
                f"目标价: {target_price}\n"
                f"链接: {url}"
            )
        }
    }
    try:
        requests.post(webhook_url, json=data, timeout=5)
        print(f"[OK] Webhook 提醒已发送")
    except Exception as e:
        print(f"[ERR] Webhook 发送失败: {e}")


def send_alert(product_name, current_price, target_price, url, config):
    """发送提醒（根据配置选择方式）"""
    method = config.get("alert_method", "console")
    if method == "console":
        send_alert_console(product_name, current_price, target_price, url)
    elif method == "email":
        send_alert_email(product_name, current_price, target_price, url, config)
    elif method == "webhook":
        send_alert_webhook(
            product_name, current_price, target_price, url,
            config.get("webhook_url", "")
        )


# ============ 主流程 ============

def check_prices(config):
    """检查所有商品价格"""
    print(f"\n[{datetime.now().strftime('%H:%M:%S')}] 开始检查价格...")
    for product in config["products"]:
        name = product["name"]
        url = product["url"]
        target = product["target_price"]
        selector = product.get("selector", ".price")
        currency = product.get("currency", "¥")

        print(f"  📦 检查: {name}")
        price_num, price_text, success = fetch_price(url, selector)

        if not success:
            print(f"  ❌ {price_text}")
            continue

        print(f"  💰 当前价: {currency}{price_num} ({price_text})")
        save_price_history(name, price_num, currency)

        if price_num is not None and price_num <= target:
            print(f"  🎉 低于目标价 {currency}{target}！发送提醒...")
            send_alert(name, f"{currency}{price_num}", f"{currency}{target}", url, config)
        else:
            print(f"  ✅ 尚未达到目标价 {currency}{target}")

    print(f"[{datetime.now().strftime('%H:%M:%S')}] 检查完成\n")


def show_history(csv_file="price_history.csv", n=10):
    """显示最近N条价格记录"""
    if not os.path.exists(csv_file):
        print("[INFO] 暂无价格记录")
        return
    with open(csv_file, "r", encoding="utf-8-sig") as f:
        reader = csv.reader(f)
        rows = list(reader)
    if len(rows) <= 1:
        print("[INFO] 暂无价格记录")
        return
    print(f"\n📊 最近{n}条价格记录:")
    print(f"{'时间':<20} {'商品':<15} {'价格':<10}")
    print("-" * 50)
    for row in rows[-n:]:
        print(f"{row[0]:<20} {row[1]:<15} {row[2]:<10}")


# ============ 入口 ============

def main():
    """主函数：单次检查或持续监控"""
    config = load_config()

    # 先检查一次
    check_prices(config)
    show_history()

    # 如果设置了间隔，持续监控
    interval = config.get("check_interval", 0)
    if interval > 0:
        print(f"⏰ 将持续监控，每 {interval} 秒检查一次")
        print("按 Ctrl+C 停止\n")
        try:
            while True:
                time.sleep(interval)
                check_prices(config)
        except KeyboardInterrupt:
            print("\n[INFO] 监控已停止")
    else:
        print("[INFO] 单次检查完成（设置 check_interval > 0 可开启持续监控）")


if __name__ == "__main__":
    main()
```

## 使用示例

### 示例1：监控京东商品价格

```python
# 先找到商品页面的价格元素CSS选择器
# 打开京东商品页 → 右键价格 → 检查 → 复制CSS选择器

# config.json 配置：
{
  "products": [
    {
      "name": "iPhone 15",
      "url": "https://item.jd.com/100063251896.html",
      "target_price": 4999,
      "selector": ".price"
    }
  ],
  "check_interval": 3600,
  "alert_method": "console"
}
```

### 示例2：监控多个商品

```python
# config.json 配置多个商品：
{
  "products": [
    {
      "name": "iPhone 15",
      "url": "https://item.jd.com/100063251896.html",
      "target_price": 4999,
      "selector": ".price"
    },
    {
      "name": "MacBook Air",
      "url": "https://item.jd.com/100062356808.html",
      "target_price": 7999,
      "selector": ".price"
    },
    {
      "name": "AirPods Pro",
      "url": "https://item.jd.com/100048370566.html",
      "target_price": 1499,
      "selector": ".price"
    }
  ],
  "check_interval": 7200,     # 每2小时检查一次
  "alert_method": "console"
}
```

### 示例3：邮件提醒配置（QQ邮箱）

```python
# 1. 登录QQ邮箱 → 设置 → 账户 → 生成授权码
# 2. 在 config.json 中添加：
{
  "alert_method": "email",
  "email": {
    "smtp_server": "smtp.qq.com",
    "smtp_port": 587,
    "sender": "yourname@qq.com",
    "password": "你的SMTP授权码",
    "receiver": "yourname@qq.com"
  }
}
```

## 常见问题

### Q: 抓不到价格？
**原因：** 很多网站是动态加载的（JavaScript渲染），requests拿不到完整HTML。
**解决：**
1. 先查看页面源代码（Ctrl+U），看价格是否在HTML中
2. 如果不在，尝试找API接口（F12 → Network → 搜索价格关键词）
3. 或者用 Selenium 替代 requests（更复杂一点）

### Q: 被网站封IP？
**原因：** 请求太频繁。
**解决：**
1. 加长检查间隔（建议至少1小时）
2. 使用代理IP
3. 添加随机延时

### Q: 价格格式解析失败？
**原因：** 不同网站价格格式不同（¥1,299 / $1299.00 / 1299元）。
**解决：** 脚本已支持常见格式，如果失败可以手动修改 `fetch_price()` 中的正则表达式。

### Q: 如何找到CSS选择器？
**方法：** 在浏览器中 → 右键价格 → 检查 → 右键价格元素 → Copy → Copy selector

### Q: 价格历史数据怎么看？
**方法：** 运行 `python price_monitor.py` 后会自动显示，或者直接打开 `price_history.csv` 用Excel查看。

## 进阶用法

### 1. 用企业微信机器人提醒
```python
# 创建企业微信群机器人 → 复制Webhook URL
# 在 config.json 中设置：
{
  "alert_method": "webhook",
  "webhook_url": "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx"
}
```

### 2. 生成价格趋势图
```python
# 安装依赖
pip install matplotlib

# 绘制价格趋势
python -c "
import pandas as pd
import matplotlib.pyplot as plt
df = pd.read_csv('price_history.csv')
df['时间'] = pd.to_datetime(df['时间'])
for product in df['商品'].unique():
    data = df[df['商品'] == product]
    plt.plot(data['时间'], data['价格'], label=product, marker='o')
plt.legend()
plt.title('价格趋势')
plt.xticks(rotation=45)
plt.tight_layout()
plt.savefig('price_trend.png')
print('图表已保存到 price_trend.png')
"
```

### 3. 定时自动运行（Windows任务计划程序）
```bash
# 创建 run_monitor.bat：
@echo off
cd /d "C:\你的项目路径"
python price_monitor.py

# 然后：
# 1. 按 Win+R → 输入 taskschd.msc
# 2. 创建任务 → 触发器设为每天/每小时
# 3. 操作 → 启动程序 → 选择 run_monitor.bat
```

## 参考资源
- [BeautifulSoup 官方文档](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
- [Requests 库文档](https://requests.readthedocs.io/)
- [CSS选择器速查表](https://www.w3schools.com/cssref/css_selectors.asp)
- QQ邮箱SMTP设置: 设置 → 账户 → POP3/SMTP服务 → 生成授权码
- 企业微信机器人: 群设置 → 群机器人 → 添加