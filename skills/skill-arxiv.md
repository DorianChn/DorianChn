# skill-arxiv：用Python搜索学术论文

## 这个skill能做什么
不用开浏览器，在终端里就能搜索 arXiv 上的学术论文，支持按关键词、作者、分类搜索，还能自动生成 BibTeX 引用格式。

## 使用场景
- 写论文/作业时需要找相关文献
- 跟踪某个领域的最新论文
- 查某个作者发了哪些论文
- 做文献综述时批量搜索
- 作业里引用论文，需要 BibTeX 格式

## 前置要求
```bash
# 不需要安装任何包！只用 Python 自带的库
# 确保 Python 3 已安装即可
python --version
```

## 快速开始

### 1. 搜索论文（关键词搜索）
```python
import urllib.request
import xml.etree.ElementTree as ET

# 搜索关键词（用 + 连接多个词）
query = "large+language+model+survey"
url = f"https://export.arxiv.org/api/query?search_query=all:{query}&max_results=5"

# 请求API
with urllib.request.urlopen(url) as resp:
    xml_data = resp.read().decode('utf-8')

# 解析XML
root = ET.fromstring(xml_data)
ns = {'a': 'http://www.w3.org/2005/Atom'}

# 打印结果
for i, entry in enumerate(root.findall('a:entry', ns)):
    title = entry.find('a:title', ns).text.strip().replace('\n', ' ')
    arxiv_id = entry.find('a:id', ns).text.split('/abs/')[-1]
    published = entry.find('a:published', ns).text[:10]
    authors = ', '.join(a.find('a:name', ns).text 
                        for a in entry.findall('a:author', ns))
    summary = entry.find('a:summary', ns).text.strip()[:150]
    
    print(f'#{i+1} [{arxiv_id}]')
    print(f'  标题: {title}')
    print(f'  作者: {authors}')
    print(f'  日期: {published}')
    print(f'  摘要: {summary}...')
    print(f'  链接: https://arxiv.org/abs/{arxiv_id}')
    print()
```

### 2. 更完整好用的版本
```python
"""
arxiv_search.py - 搜索arXiv学术论文
用法: python arxiv_search.py "关键词" --max 5
"""
import urllib.request
import xml.etree.ElementTree as ET
import sys


def search_arxiv(query, max_results=5, sort_by='relevance'):
    """搜索arXiv论文，返回论文列表"""
    # 构建URL
    query_encoded = query.replace(' ', '+')
    url = (f"https://export.arxiv.org/api/query?"
           f"search_query=all:{query_encoded}"
           f"&max_results={max_results}"
           f"&sortBy={sort_by}")
    
    # 请求
    with urllib.request.urlopen(url) as resp:
        xml_data = resp.read().decode('utf-8')
    
    # 解析
    root = ET.fromstring(xml_data)
    ns = {'a': 'http://www.w3.org/2005/Atom'}
    
    papers = []
    for entry in root.findall('a:entry', ns):
        title = entry.find('a:title', ns).text.strip().replace('\n', ' ')
        arxiv_id = entry.find('a:id', ns).text.split('/abs/')[-1]
        
        papers.append({
            'title': title,
            'id': arxiv_id,
            'authors': [a.find('a:name', ns).text 
                       for a in entry.findall('a:author', ns)],
            'published': entry.find('a:published', ns).text[:10],
            'summary': entry.find('a:summary', ns).text.strip()[:200],
            'url': f'https://arxiv.org/abs/{arxiv_id}',
            'pdf': f'https://arxiv.org/pdf/{arxiv_id}',
        })
    
    return papers


def print_papers(papers):
    """打印论文列表"""
    for i, p in enumerate(papers, 1):
        print(f'\n{'='*50}')
        print(f'#{i} [{p["id"]}]')
        print(f'标题: {p["title"]}')
        print(f'作者: {", ".join(p["authors"][:3])}'
              f'{", ..." if len(p["authors"]) > 3 else ""}')
        print(f'日期: {p["published"]}')
        print(f'摘要: {p["summary"][:150]}...')
        print(f'PDF: {p["pdf"]}')
    print(f'\n共找到 {len(papers)} 篇论文')


def generate_bibtex(paper):
    """生成 BibTeX 引用格式"""
    author_last = paper['authors'][0].split()[-1] if paper['authors'] else 'Unknown'
    bib_id = f"{author_last}{paper['published'][:4]}_{paper['id'].split('.')[-1]}"
    
    lines = [
        f'@article{{{bib_id},',
        f'  title     = {{{paper["title"]}}},',
        f'  author    = {{{" and ".join(paper["authors"])}}},',
        f'  year      = {{{paper["published"][:4]}}},',
        f'  eprint    = {{{paper["id"]}}},',
        f'  archivePrefix = {{arXiv}},',
        f'  url       = {{{paper["url"]}}}',
        '}',
    ]
    return '\n'.join(lines)


if __name__ == '__main__':
    # 从命令行参数读取关键词
    query = ' '.join(sys.argv[1:]) if len(sys.argv) > 1 else 'machine+learning'
    papers = search_arxiv(query, max_results=5)
    print_papers(papers)
    
    # 生成第一篇论文的 BibTeX
    if papers:
        print('\n--- BibTeX 引用 ---')
        print(generate_bibtex(papers[0]))
```

## 完整代码

上面②的代码就是完整可运行的版本。把它保存为 `arxiv_search.py`，然后：

```bash
# 搜索论文（默认5篇）
python arxiv_search.py "transformer attention"

# 搜索最新论文（按日期排序）
# 修改代码中 sort_by='submittedDate'

# 搜索特定作者
python arxiv_search.py "author:lecun"
```

## 常见问题

**Q: 报错 `urllib.error.HTTPError: HTTP Error 503`**
A: arXiv API 有频率限制，两次请求之间至少间隔3秒。加一行 `time.sleep(3)`。

**Q: 中文搜索不行？**
A: arXiv 论文基本都是英文，用英文关键词搜索。

**Q: 怎么搜特定分类的论文？**
```python
# 搜索 AI 分类
url = "https://export.arxiv.org/api/query?search_query=cat:cs.AI&max_results=5"
```
常用分类：
- `cs.AI` - 人工智能
- `cs.LG` - 机器学习
- `cs.CL` - 自然语言处理
- `cs.CV` - 计算机视觉
- `stat.ML` - 统计机器学习

**Q: 怎么读论文全文？**
A: 复制 PDF 链接到浏览器打开，或者用 `web_extract` 工具：
```bash
# 在 Hermes 中直接提取 PDF 内容
web_extract(urls=["https://arxiv.org/pdf/2402.03300"])
```

**Q: 搜索不到结果？**
A: 试试更简单的关键词，不用特殊符号。

## 进阶用法

### 1. 搜索特定论文并查看引用量
```python
# 先用 arXiv 找到论文 ID
# 用 Semantic Scholar API 查引用量
import json

paper_id = "1706.03762"  # Attention Is All You Need
url = f"https://api.semanticscholar.org/graph/v1/paper/arXiv:{paper_id}?fields=title,citationCount"
with urllib.request.urlopen(url) as resp:
    data = json.loads(resp.read())
    print(f"论文: {data['title']}")
    print(f"被引用次数: {data['citationCount']}")
```

### 2. 批量搜索并保存到文件
```python
import json

keywords = ["transformer", "reinforcement learning", "diffusion model"]
all_papers = []

for kw in keywords:
    papers = search_arxiv(kw, max_results=3)
    all_papers.extend(papers)

with open('papers.json', 'w', encoding='utf-8') as f:
    json.dump(all_papers, f, ensure_ascii=False, indent=2)

print(f"共收集 {len(all_papers)} 篇论文，已保存到 papers.json")
```

## 参考资源
- [arXiv API 官方文档](https://info.arxiv.org/help/api/index.html)
- [arXiv 分类列表](https://arxiv.org/category_taxonomy)
- [Semantic Scholar API](https://api.semanticscholar.org/)