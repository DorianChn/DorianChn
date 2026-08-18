# Skill：Docker容器化部署

## 这个skill能做什么

用Docker把你的应用打包成"集装箱"，在任何电脑上都能一键运行，告别"我电脑上明明能跑"的尴尬。

## 使用场景

- **环境统一**：团队开发时，每个人环境都一样，不吵架
- **一键部署**：本地开发完，直接打包丢到服务器跑
- **微服务**：一个项目拆成多个服务（前端+后端+数据库），各自独立运行
- **学习实验**：想装个Redis/MySQL/PostgreSQL试试，不用污染本机

## 前置要求

```bash
# 1. 安装 Docker Desktop（Windows/Mac）
# 下载地址：https://www.docker.com/products/docker-desktop/
# 安装后重启电脑，打开 Docker Desktop

# 2. 验证安装
docker --version
# 输出示例：Docker version 24.0.7, build afdd53b

# 3. （可选）配置国内镜像加速
# 打开 Docker Desktop → Settings → Docker Engine
# 添加："registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
```

## 快速开始

### 5分钟跑一个Python Web应用

**第1步：创建项目文件**

```python
# app.py — 一个简单的Flask Web应用
from flask import Flask
import platform

app = Flask(__name__)

@app.route('/')
def hello():
    return f"""
    <h1>🐳 Hello Docker!</h1>
    <p>主机名: {platform.node()}</p>
    <p>Python版本: {platform.python_version()}</p>
    <p>系统: {platform.system()} {platform.release()}</p>
    """

# 启动服务，监听所有网络接口（0.0.0.0）
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**第2步：创建Dockerfile**

```dockerfile
# Dockerfile — 告诉Docker如何构建这个应用的镜像
# 语法：FROM 基础镜像:标签
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 复制依赖文件（先复制这个，利用Docker缓存加速构建）
COPY requirements.txt .
RUN pip install -r requirements.txt

# 复制源代码
COPY app.py .

# 声明容器运行时监听的端口（只是说明，实际映射靠 -p 参数）
EXPOSE 5000

# 容器启动时执行的命令
CMD ["python", "app.py"]
```

**第3步：创建依赖文件**

```txt
# requirements.txt
flask==3.0.0
```

**第4步：构建并运行**

```bash
# 构建镜像（构建上下文是当前目录 .）
# -t 给镜像打标签：名字:版本
docker build -t my-web-app:v1 .

# 运行容器
# -d 后台运行
# -p 宿主机端口:容器端口
# --name 给容器起个名字
docker run -d -p 5000:5000 --name my-app my-web-app:v1

# 打开浏览器访问
# http://localhost:5000

# 查看运行中的容器
docker ps

# 查看容器日志
docker logs my-app

# 停止容器
docker stop my-app

# 删除容器
docker rm my-app
```

## 完整代码

### 实战：Web应用 + MySQL数据库（docker-compose）

```yaml
# docker-compose.yml — 一键启动多个服务
version: '3.8'

services:
  # Web应用服务
  web:
    build: .                    # 从当前目录的Dockerfile构建
    ports:
      - "5000:5000"             # 映射端口
    depends_on:
      - db                      # 先启动数据库
    environment:
      - DB_HOST=db              # 数据库主机名（服务名）
      - DB_USER=myuser
      - DB_PASSWORD=mypass
      - DB_NAME=mydb

  # MySQL数据库服务
  db:
    image: mysql:8.0            # 直接使用官方镜像
    ports:
      - "3306:3306"             # 映射MySQL端口
    environment:
      - MYSQL_ROOT_PASSWORD=rootpass
      - MYSQL_USER=myuser
      - MYSQL_PASSWORD=mypass
      - MYSQL_DATABASE=mydb
    volumes:
      - mysql_data:/var/lib/mysql  # 数据持久化（重启不丢数据）

# 定义数据卷
volumes:
  mysql_data:
```

```python
# app_with_db.py — 带数据库的Web应用
from flask import Flask
import mysql.connector
import os

app = Flask(__name__)

# 从环境变量读取数据库配置（docker-compose里设置）
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'myuser'),
    'password': os.getenv('DB_PASSWORD', 'mypass'),
    'database': os.getenv('DB_NAME', 'mydb'),
}

def get_db():
    """连接数据库"""
    return mysql.connector.connect(**DB_CONFIG)

@app.route('/')
def index():
    return '<h1>🐳 App is running!</h1><p><a href="/init">初始化数据库</a></p>'

@app.route('/init')
def init_db():
    """初始化数据库表"""
    try:
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS visitors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                visit_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        conn.commit()
        cursor.close()
        conn.close()
        return '✅ 数据库表创建成功！<a href="/visit">去看看</a>'
    except Exception as e:
        return f'❌ 数据库连接失败: {e}'

@app.route('/visit')
def visit():
    """记录访问并展示"""
    try:
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute('INSERT INTO visitors (visit_time) VALUES (NOW())')
        conn.commit()
        cursor.execute('SELECT COUNT(*) FROM visitors')
        count = cursor.fetchone()[0]
        cursor.close()
        conn.close()
        return f'<h1>👋 你是第 {count} 位访客！</h1>'
    except Exception as e:
        return f'❌ 查询失败: {e}'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装MySQL客户端依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    default-libmysqlclient-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app_with_db.py .
# 如果是app_with_db.py，需要重命名为app.py
# 或者修改CMD命令
CMD ["python", "app_with_db.py"]
```

```txt
# requirements.txt
flask==3.0.0
mysql-connector-python==8.2.0
```

**运行方式：**

```bash
# 一键启动所有服务（-d 后台运行）
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f

# 停止所有服务
docker-compose down

# 停止并删除数据卷（清空数据库）
docker-compose down -v
```

### 常用Docker命令速查表

```bash
# 🖼️ 镜像操作
docker images                        # 列出所有镜像
docker pull nginx:alpine             # 下载镜像
docker rmi 镜像ID                     # 删除镜像
docker build -t 名字:标签 .           # 构建镜像

# 📦 容器操作
docker ps                            # 运行中的容器
docker ps -a                         # 所有容器（含已停止的）
docker start 容器名                   # 启动已停止的容器
docker stop 容器名                    # 停止容器
docker rm 容器名                      # 删除容器
docker logs -f 容器名                 # 实时查看日志
docker exec -it 容器名 bash           # 进入容器内部（exit退出）

# 🧹 清理
docker system prune                  # 清理未使用的资源
docker system prune -a               # 清理所有未使用的镜像
```

## 常见问题

### Q1: 端口被占用怎么办？

```bash
# 错误信息：port is already allocated
# 解决方法：换个端口映射
docker run -d -p 5001:5000 --name my-app my-web-app:v1
# 访问 http://localhost:5001
```

### Q2: 容器退出（Exited）是什么原因？

```bash
# 查看退出原因
docker logs 容器名

# 常见原因：代码报错、端口被占用、依赖没装
# 先用交互模式运行看报错
docker run -it --rm my-web-app:v1 bash
# 手动运行 python app.py 看报什么错
```

### Q3: 容器里的数据怎么保存？

```bash
# 不加数据卷的话，容器删除后数据就没了
# 方案：使用数据卷（volume）
docker run -v mydata:/app/data my-image
# mydata 是Docker管理的数据卷，删除容器也不会丢

# 方案2：挂载本机目录（开发调试方便）
docker run -v C:/mycode:/app my-image
# 本机改代码，容器里自动生效
```

### Q4: Docker Desktop占用空间太大？

```bash
# 清理未使用的镜像、容器、网络
docker system prune -a --volumes

# 在Docker Desktop设置里限制资源：
# Settings → Resources → Advanced
# 建议：CPU 4核，内存 4GB，Swap 2GB
```

### Q5: 国内下载镜像太慢？

```bash
# 配置镜像加速器
# Docker Desktop → Settings → Docker Engine
# 编辑json，添加：
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
# 点击 Apply & Restart
```

## 进阶用法

### 多阶段构建（减小镜像体积）

```dockerfile
# 第一阶段：编译
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# 第二阶段：运行（只复制编译好的文件）
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
# 镜像体积从 1GB 降到 200MB！
```

### 使用.dockerignore（加速构建）

```dockerfile
# .dockerignore — 放在项目根目录
.git
__pycache__
*.pyc
.env
venv
.DS_Store
```

## 参考资源

- [Docker官方文档](https://docs.docker.com/) — 最权威的教程
- [Docker菜鸟教程](https://www.runoob.com/docker/docker-tutorial.html) — 中文入门
- [Play with Docker](https://labs.play-with-docker.com/) — 在线练习，不用安装
- [Docker Hub](https://hub.docker.com/) — 找现成的镜像