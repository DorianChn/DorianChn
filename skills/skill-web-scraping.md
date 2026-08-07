# 网页爬虫入门（Python 版）

## 这个 skill 能做什么
用 Python 自动抓取网页上的公开数据——比如抓取新闻标题、商品价格、电影榜单，保存到 Excel 或 JSON 文件，不用手动复制粘贴。

## 使用场景
- 看到某个网站的数据想批量下载（比如电影排行榜、招聘信息）
- 监控商品价格变化，降价时自动通知
- 采集公开数据集做分析或练手项目
- 爬取学习资料、文档、博客文章

## 前置要求
```bash
# 安装两个库就够了
pip install requests beautifulsoup4
```

## 快速开始

### 1. 抓取网页标题（5行代码）
```python
import requests
from bs4 import BeautifulSoup

# 请求网页（加个请求头伪装成浏览器）
headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"}
resp = requests.get("https://example.com", headers=headers, timeout=10)

# 解析 HTML
soup = BeautifulSoup(resp.text, "html.parser")

# 提取标题
print("页面标题:", soup.title.text.strip())
```

### 2. 抓取所有链接
```python
# 接上面的 soup
for a in soup.find_all("a"):
    href = a.get("href")
    text = a.text.strip()
    if href and text:
        print(f"{text} -> {href}")
```

## 完整代码：豆瓣电影 Top250 爬虫

```python
"""
douban_top250.py - 豆瓣电影 Top250 爬虫
抓取电影排名、名称、评分、评价人数，保存到 JSON 文件
"""
import requests
import time
import json
from bs4 import BeautifulSoup

# ========== 配置 ==========
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "Accept-Language": "zh-CN,zh;q=0.9",
}
OUTPUT_FILE = "douban_top250.json"


def fetch_page(start):
    """抓取一页（25条），start 从0开始"""
    url = f"https://movie.douban.com/top250?start={start}"
    resp = requests.get(url, headers=HEADERS, timeout=10)
    resp.encoding = "utf-8"
    soup = BeautifulSoup(resp.text, "html.parser")

    movies = []
    for item in soup.select(".item"):
        # 排名
        rank = item.select_one(".pic em").text
        # 电影名
        title = item.select_one(".title").text
        # 评分
        rating = item.select_one(".rating_num").text
        # 评价人数
        quote = item.select_one(".inq")
        comment = quote.text if quote else ""

        movies.append({
            "rank": int(rank),
            "title": title,
            "rating": float(rating),
            "comment": comment,
        })

    return movies


def crawl_top250():
    """抓取全部250部电影"""
    all_movies = []
    for start in range(0, 250, 25):
        print(f"正在抓取第 {start+1}-{start+25} 部...")
        movies = fetch_page(start)
        all_movies.extend(movies)
        time.sleep(1)  # 礼貌延时，别搞崩人家服务器

    # 按排名排序
    all_movies.sort(key=lambda x: x["rank"])

    # 保存到 JSON
    with open(OUTPUT_FILE, "w", encoding="utf-8") as f:
        json.dump(all_movies, f, ensure_ascii=False, indent=2)

    print(f"\n✅ 抓取完成！共 {len(all_movies)} 部电影")
    print(f"📁 已保存到 {OUTPUT_FILE}")
    return all_movies


def show_stats(movies):
    """显示统计信息"""
    avg_rating = sum(m["rating"] for m in movies) / len(movies)
    print(f"\n📊 统计信息：")
    print(f"   平均评分：{avg_rating:.2f}")
    print(f"   最高分：{max(m['rating'] for m in movies)}")
    print(f"   最低分：{min(m['rating'] for m in movies)}")
    print(f"   前3名：{', '.join(m['title'] for m in movies[:3])}")


if __name__ == "__main__":
    movies = crawl_top250()
    show_stats(movies)
```

## 常见问题

**Q: 报错 `ConnectionError` 或超时？**
A: 国内访问某些网站可能不稳定，可以：
- 加 `timeout=15` 延长等待时间
- 或者换个网站练习（比如 httpbin.org）

**Q: 抓取到的数据是乱的？**
A: 检查网页编码，有些网站不是 UTF-8：
```python
resp.encoding = resp.apparent_encoding  # 自动检测编码
```

**Q: 被网站封了 IP 怎么办？**
A: 降低请求频率，每两次请求之间加 `time.sleep(2)`。如果还不行，加请求头伪装：
```python
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "Referer": "https://www.google.com/",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
}
```

**Q: 有些网站内容是 JavaScript 动态加载的，抓不到？**
A: 这种需要 `selenium` 或 `playwright` 模拟浏览器，见进阶用法。

**Q: 爬虫违法吗？**
A: 爬取**公开数据**（不登录就能看的内容）一般没问题。但要注意：
- 遵守网站的 `robots.txt`（例：https://www.douban.com/robots.txt）
- 不要频繁请求（别搞崩人家服务器）
- 不要爬取需要登录或个人隐私的内容

## 进阶用法

### 1. 动态页面爬虫（用 Playwright）
```python
# 安装：pip install playwright && playwright install chromium
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://example.com")
    # 等待页面加载完
    page.wait_for_selector(".content")
    # 获取内容
    html = page.content()
    browser.close()
```

### 2. 爬取多个页面（并发）
```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch_one(url):
    resp = requests.get(url, headers=HEADERS, timeout=10)
    return resp.text

urls = [f"https://example.com/page/{i}" for i in range(1, 11)]
with ThreadPoolExecutor(max_workers=5) as pool:
    futures = {pool.submit(fetch_one, url): url for url in urls}
    for future in as_completed(futures):
        print(f"完成：{futures[future]}")
```

### 3. 爬取结果保存到 Excel
```python
import openpyxl

def save_to_excel(movies, filename="movies.xlsx"):
    wb = openpyxl.Workbook()
    ws = wb.active
    ws.title = "电影排行榜"
    ws.append(["排名", "电影名", "评分", "评论"])
    for m in movies:
        ws.append([m["rank"], m["title"], m["rating"], m["comment"]])
    wb.save(filename)
    print(f"已保存到 {filename}")

# 用法：save_to_excel(movies)
```

## 参考资源
- [BeautifulSoup 官方文档](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
- [Requests 文档](https://requests.readthedocs.io/)
- [Playwright Python 文档](https://playwright.dev/python/)
- [robots.txt 说明](https://developers.google.com/search/docs/crawling-indexing/robots/intro)