# 单文件Web应用

> 一个HTML文件搞定一个交互式小工具，无需服务器、无需安装、双击就能用

## 这个skill能做什么

用**一个HTML文件**创建完整的交互式Web应用：包含界面、样式、交互逻辑和本地数据存储，双击就能运行。

## 使用场景

- 做一个**个人记账本**，记录每天花销
- 做一个**任务看板**，管理待办事项
- 做一个**数据仪表盘**，实时展示数据
- 做一个**学习计时器**，番茄工作法
- 总之，任何不需要后端的**小工具**都可以用一个HTML搞定

## 前置要求

- 一个浏览器（Chrome/Edge推荐）
- 一个文本编辑器（VS Code / 记事本都可以）
- 不需要安装任何东西

## 快速开始

1. 复制下面的完整代码，保存为 `myapp.html`
2. 双击 `myapp.html` 在浏览器打开
3. 开始使用！

## 完整代码

### 示例1：待办事项管理（70行）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>我的待办</title>
<style>
  /* ===== 全局样式 ===== */
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: 'Segoe UI', system-ui, sans-serif;
    background: #0b0e14;      /* 深色背景，护眼 */
    color: #cdd4e0;
    max-width: 500px;
    margin: 40px auto;
    padding: 0 16px;
  }
  h1 { font-size: 24px; margin-bottom: 20px; color: #fff; }
  .input-row {
    display: flex; gap: 8px; margin-bottom: 20px;
  }
  .input-row input {
    flex: 1; padding: 10px; border-radius: 8px;
    border: 1px solid #1e2533; background: #131821; color: #fff;
    font-size: 14px;
  }
  .input-row input:focus { outline: none; border-color: #4a9eff; }
  .input-row button {
    padding: 10px 20px; border-radius: 8px; border: none;
    background: #4a9eff; color: #fff; font-size: 14px;
    cursor: pointer;
  }
  .input-row button:hover { background: #3a8eef; }
  /* ===== 待办列表 ===== */
  .todo-item {
    display: flex; align-items: center; gap: 10px;
    padding: 12px; background: #131821; border-radius: 8px;
    margin-bottom: 8px; border: 1px solid #1e2533;
  }
  .todo-item.done .todo-text {
    text-decoration: line-through; color: #666;
  }
  .todo-text { flex: 1; }
  .todo-item button {
    background: none; border: none; color: #e74c3c;
    cursor: pointer; font-size: 16px;
  }
  /* ===== 统计栏 ===== */
  .stats {
    text-align: center; margin-top: 20px;
    color: #888; font-size: 13px;
  }
  .stats span { color: #4a9eff; }
</style>
</head>
<body>

<h1>📋 我的待办</h1>

<div class="input-row">
  <input id="todoInput" placeholder="输入要做的事..." autofocus>
  <button onclick="addTodo()">添加</button>
</div>

<div id="todoList"></div>

<div class="stats">
  共 <span id="totalCount">0</span> 项 · 
  已完成 <span id="doneCount">0</span> 项
</div>

<script>
// ===== 数据管理（localStorage 持久化） =====
let todos = [];

// 加载保存的数据（try-catch 防止文件协议下 localStorage 报错）
try {
  const saved = localStorage.getItem('todos');
  if (saved) todos = JSON.parse(saved);
} catch(e) { /* 浏览器限制时使用空数据 */ }

// 保存数据
function saveTodos() {
  try { localStorage.setItem('todos', JSON.stringify(todos)); } catch(e) {}
}

// ===== 渲染列表 =====
function renderTodos() {
  const list = document.getElementById('todoList');
  list.innerHTML = todos.map((todo, i) => `
    <div class="todo-item ${todo.done ? 'done' : ''}">
      <input type="checkbox" ${todo.done ? 'checked' : ''}
             onchange="toggleTodo(${i})">
      <span class="todo-text">${todo.text}</span>
      <button onclick="deleteTodo(${i})">✕</button>
    </div>
  `).join('');

  // 更新统计
  document.getElementById('totalCount').textContent = todos.length;
  document.getElementById('doneCount').textContent =
    todos.filter(t => t.done).length;
}

// ===== 操作函数 =====
function addTodo() {
  const input = document.getElementById('todoInput');
  const text = input.value.trim();
  if (!text) return alert('请输入待办内容');
  todos.push({ text, done: false });
  input.value = '';          // 清空输入框
  input.focus();             // 保持焦点
  saveTodos();
  renderTodos();
}

function toggleTodo(index) {
  todos[index].done = !todos[index].done;
  saveTodos();
  renderTodos();
}

function deleteTodo(index) {
  todos.splice(index, 1);    // 删除该项
  saveTodos();
  renderTodos();
}

// ===== 按回车键添加 =====
document.getElementById('todoInput')
  .addEventListener('keydown', e => {
    if (e.key === 'Enter') addTodo();
  });

// ===== 启动 =====
renderTodos();
</script>
</body>
</html>
```

### 示例2：番茄钟计时器（单独文件，可替换上面的代码）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>🍅 番茄钟</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: 'Segoe UI', system-ui, sans-serif;
    background: #0b0e14; color: #cdd4e0;
    display: flex; justify-content: center; align-items: center;
    min-height: 100vh;
  }
  .card {
    background: #131821; border-radius: 16px; padding: 40px;
    border: 1px solid #1e2533; text-align: center; width: 340px;
  }
  .timer {
    font-size: 72px; font-weight: 700; color: #fff;
    font-variant-numeric: tabular-nums;  /* 等宽数字，不抖动 */
    margin: 20px 0; letter-spacing: 4px;
  }
  .status {
    font-size: 14px; color: #4a9eff; margin-bottom: 8px;
  }
  .btn-group { display: flex; gap: 8px; justify-content: center; }
  .btn-group button {
    padding: 10px 24px; border-radius: 8px; border: none;
    font-size: 14px; cursor: pointer; transition: 0.2s;
  }
  .btn-start { background: #4a9eff; color: #fff; }
  .btn-start:hover { background: #3a8eef; }
  .btn-start.running { background: #e74c3c; }
  .btn-start.running:hover { background: #c0392b; }
  .btn-reset { background: #1e2533; color: #888; }
  .btn-reset:hover { background: #2a3140; color: #fff; }
  .sessions { margin-top: 20px; font-size: 13px; color: #666; }
</style>
</head>
<body>
<div class="card">
  <div class="status" id="status">🍅 专注时间</div>
  <div class="timer" id="timer">25:00</div>
  <div class="btn-group">
    <button class="btn-start" id="startBtn" onclick="toggleTimer()">开始</button>
    <button class="btn-reset" onclick="resetTimer()">重置</button>
  </div>
  <div class="sessions">今日完成 <span id="sessionCount">0</span> 个番茄</div>
</div>

<script>
// ===== 番茄钟配置 =====
const WORK_TIME = 25 * 60;    // 25分钟专注
const BREAK_TIME = 5 * 60;    // 5分钟休息

let timeLeft = WORK_TIME;     // 剩余秒数
let isRunning = false;        // 是否在运行
let isWork = true;            // 专注模式还是休息模式
let timer = null;             // setInterval 句柄

// 加载今日完成数
let sessions = 0;
try {
  const today = new Date().toDateString();
  const saved = JSON.parse(localStorage.getItem('pomodoro') || '{}');
  if (saved.date === today) sessions = saved.count || 0;
} catch(e) {}

// 更新显示
function updateDisplay() {
  const m = String(Math.floor(timeLeft / 60)).padStart(2, '0');
  const s = String(timeLeft % 60).padStart(2, '0');
  document.getElementById('timer').textContent = `${m}:${s}`;
  document.title = `${m}:${s} - 番茄钟`;
}

// 切换 开始/暂停
function toggleTimer() {
  const btn = document.getElementById('startBtn');
  if (isRunning) {
    // 暂停
    clearInterval(timer);
    isRunning = false;
    btn.textContent = '继续';
    btn.classList.remove('running');
  } else {
    // 开始
    isRunning = true;
    btn.textContent = '暂停';
    btn.classList.add('running');
    timer = setInterval(() => {
      timeLeft--;
      updateDisplay();
      if (timeLeft <= 0) {
        clearInterval(timer);
        isRunning = false;
        btn.textContent = '开始';
        btn.classList.remove('running');
        // 切换模式
        isWork = !isWork;
        timeLeft = isWork ? WORK_TIME : BREAK_TIME;
        updateDisplay();
        document.getElementById('status').textContent =
          isWork ? '🍅 专注时间' : '☕ 休息时间';
        // 专注完成时记录
        if (isWork) {
          sessions++;
          document.getElementById('sessionCount').textContent = sessions;
          try {
            localStorage.setItem('pomodoro', JSON.stringify({
              date: new Date().toDateString(), count: sessions
            }));
          } catch(e) {}
          new Notification('🍅 番茄钟', {  // 桌面通知
            body: '专注时间结束！休息一下吧~'
          });
        }
      }
    }, 1000);
  }
}

// 重置
function resetTimer() {
  clearInterval(timer);
  isRunning = false;
  isWork = true;
  timeLeft = WORK_TIME;
  document.getElementById('startBtn').textContent = '开始';
  document.getElementById('startBtn').classList.remove('running');
  document.getElementById('status').textContent = '🍅 专注时间';
  updateDisplay();
}

// 请求桌面通知权限
if ('Notification' in window && Notification.permission === 'default') {
  Notification.requestPermission();
}

// 启动
updateDisplay();
document.getElementById('sessionCount').textContent = sessions;
</script>
</body>
</html>
```

## 常见问题

**Q: 双击HTML文件后，浏览器打开但页面是空白的？**
A: 检查文件是否保存为 `.html` 后缀，不是 `.txt`。在文件资源管理器里勾选"文件扩展名"确认。

**Q: 数据刷新后就不见了？**
A: 本示例用 `localStorage` 存数据。清除浏览器缓存或使用隐私模式会丢失数据。正常使用不会丢。

**Q: 为什么用 `try-catch` 包着 localStorage？**
A: 某些浏览器在 `file://` 协议下禁止 `localStorage`，不加 try-catch 整个脚本会报错停止。

**Q: 如何让页面更好看？**
A: 搜 "CSS 颜色搭配" 找配色方案，或搜 "CSS 卡片布局" 学排版技巧。

**Q: 如何添加更多功能？**
A: 把新的 HTML 元素加到 `<body>`，CSS 样式加到 `<style>`，JS 逻辑加到 `<script>` 就行。

## 进阶用法

### 添加 Chart.js 图表

在 `<head>` 中引入 Chart.js 的 CDN：
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
```
然后在 `<script>` 中创建图表：
```javascript
const ctx = document.getElementById('myChart').getContext('2d');
new Chart(ctx, {
  type: 'bar',
  data: {
    labels: ['周一', '周二', '周三'],
    datasets: [{ label: '花费', data: [120, 85, 200] }]
  }
});
```

### 添加实时数据（定时刷新）

```javascript
async function fetchData() {
  try {
    const res = await fetch('https://api.example.com/data');
    const data = await res.json();
    updateUI(data);
  } catch(e) {
    console.log('网络错误，使用缓存数据');
  }
}
// 每30秒刷新一次
fetchData();
setInterval(fetchData, 30000);
```

### 打包为桌面应用（可选）

用 [Nativefier](https://github.com/nativefier/nativefier) 或 [PWA Builder](https://www.pwabuilder.com/) 把 HTML 变成 .exe 安装包。

## 参考资源

- [MDN Web 文档](https://developer.mozilla.org/zh-CN/docs/Web) - 最权威的Web开发教程
- [Chart.js 官方文档](https://www.chartjs.org/docs/) - 图表库用法
- [CSS Tricks](https://css-tricks.com/) - CSS技巧大全
- 推荐学习路径：HTML → CSS → JavaScript → 本项目