# 机器学习入门（Python 版）

## 这个 skill 能做什么
用 Python 的 scikit-learn 做机器学习——训练模型预测数据，比如预测房价、识别垃圾邮件、分类客户。不需要数学基础，复制代码就能跑。

## 使用场景
- **学生**：课程作业、入门 AI 实践
- **打工人**：预测销售额、客户分类、异常检测
- **开发者**：给项目加 AI 功能、数据挖掘

## 前置要求
```bash
# 安装三个库，一行搞定
pip install scikit-learn pandas numpy matplotlib
```

## 快速开始

### 5分钟上手：预测鸢尾花种类
```python
# 1. 加载数据和模型
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier

# 2. 准备数据
iris = load_iris()
X, y = iris.data, iris.target  # 特征和标签

# 3. 训练模型
model = RandomForestClassifier()
model.fit(X, y)

# 4. 预测新数据
new_flower = [[5.1, 3.5, 1.4, 0.2]]  # 花萼长宽、花瓣长宽
pred = model.predict(new_flower)
print(f"预测种类: {iris.target_names[pred[0]]}")  # setosa
```

## 完整代码

### 完整案例：房价预测（含数据分割 + 评估 + 可视化）

```python
"""
房价预测 - 完整机器学习流程
数据集：波士顿房价（sklearn内置）
"""
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.datasets import load_diabetes
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error, r2_score

# ========== 1. 加载数据 ==========
data = load_diabetes()
df = pd.DataFrame(data.data, columns=data.feature_names)
df['target'] = data.target

print("数据形状:", df.shape)
print(df.head())

# ========== 2. 分割数据 ==========
# 训练集80% + 测试集20%
X = data.data
y = data.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# ========== 3. 训练模型 ==========
model = RandomForestRegressor(
    n_estimators=100,  # 100棵树
    max_depth=10,      # 最大深度
    random_state=42
)
model.fit(X_train, y_train)

# ========== 4. 预测和评估 ==========
y_pred = model.predict(X_test)

# 评估指标
mse = mean_squared_error(y_test, y_pred)
r2 = r2_score(y_test, y_pred)
print(f"\n均方误差(MSE): {mse:.2f}")
print(f"R² 分数: {r2:.2f}")  # 越接近1越好

# ========== 5. 可视化 ==========
plt.figure(figsize=(8, 5))
plt.scatter(y_test, y_pred, alpha=0.5)
plt.plot([y.min(), y.max()], [y.min(), y.max()], 'r--')
plt.xlabel("真实值")
plt.ylabel("预测值")
plt.title("房价预测结果")
plt.tight_layout()
plt.savefig("prediction_result.png")
print("图表已保存: prediction_result.png")
```

### 完整案例：垃圾邮件分类

```python
"""
垃圾邮件分类 - 文本分类入门
"""
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, classification_report

# ========== 1. 准备数据 ==========
emails = [
    "免费领取大奖，点击链接！",        # 垃圾
    "明天开会，记得带材料",            # 正常
    "恭喜中奖，速来领取",              # 垃圾
    "项目进度怎么样？",                 # 正常
    "限时优惠，错过等一年",            # 垃圾
    "今晚一起吃饭吗？",                 # 正常
    "您的账户异常，立即验证",          # 垃圾
    "周报记得提交",                    # 正常
]
labels = [1, 0, 1, 0, 1, 0, 1, 0]  # 1=垃圾, 0=正常

# ========== 2. 文本转数字特征 ==========
vectorizer = CountVectorizer()
X = vectorizer.fit_transform(emails)

# ========== 3. 分割数据 ==========
X_train, X_test, y_train, y_test = train_test_split(
    X, labels, test_size=0.25, random_state=42
)

# ========== 4. 训练模型 ==========
model = MultinomialNB()
model.fit(X_train, y_train)

# ========== 5. 预测和评估 ==========
y_pred = model.predict(X_test)
print(f"准确率: {accuracy_score(y_test, y_pred):.2f}")
print(classification_report(y_test, y_pred, target_names=['正常', '垃圾']))

# ========== 6. 预测新邮件 ==========
new_emails = ["恭喜您获得iPhone大奖", "下午三点开会"]
new_X = vectorizer.transform(new_emails)
predictions = model.predict(new_X)
for email, label in zip(new_emails, predictions):
    print(f"'{email}' → {'垃圾邮件' if label else '正常邮件'}")
```

## 常见问题

### Q1: 什么是训练集和测试集？
**A**: 训练集是给模型"学习"的数据，测试集是"考试"的数据。用测试集评估模型在未知数据上的表现，防止过拟合（只会背答案不会做题）。

### Q2: 模型准确率低怎么办？
6个方向，从易到难：
1. 数据量不够 → 多收集数据
2. 特征太少 → 添加更多特征
3. 数据要标准化 → 加 `StandardScaler`
4. 模型太简单 → 换 RandomForest
5. 调参 → 调整 `n_estimators`、`max_depth`
6. 数据问题 → 检查是否有异常值、缺失值

### Q3: 怎么选模型？
| 问题类型 | 首选模型 | 适用场景 |
|---------|---------|---------|
| 二分类（是/否） | LogisticRegression | 垃圾邮件、疾病诊断 |
| 多分类（多种类别） | RandomForestClassifier | 手写数字、图片分类 |
| 回归（预测数值） | RandomForestRegressor | 房价、销量预测 |
| 聚类（自动分组） | KMeans | 客户分群、图像分割 |

### Q4: 安装报错怎么办？
- **报错 `pip` 不是命令** → 先装 Python（python.org）
- **报错 `Microsoft Visual C++`** → 装 [VC++ Redistributable](https://aka.ms/vs/17/release/vc_redist.x64.exe)
- **报错 `No module named sklearn`** → 运行 `pip install scikit-learn`

### Q5: 什么是过拟合？
模型把训练数据背得太熟，换新数据就不行了。表现为：训练集准确率99%，测试集准确率60%。解决方法：减少模型复杂度、加正则化、增加数据量。

## 进阶用法

### 调参优化
```python
from sklearn.model_selection import GridSearchCV

# 自动搜索最佳参数
param_grid = {
    'n_estimators': [50, 100, 200],
    'max_depth': [5, 10, 20, None]
}
grid = GridSearchCV(RandomForestRegressor(), param_grid, cv=5)
grid.fit(X_train, y_train)
print("最佳参数:", grid.best_params_)
```

### 保存和加载模型
```python
import joblib

# 保存
joblib.dump(model, 'my_model.pkl')

# 加载
model = joblib.load('my_model.pkl')
```

### 数据标准化
```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)  # 注意：用训练集的scaler
```

## 参考资源
- [scikit-learn 官方教程](https://scikit-learn.org/stable/tutorial/basic/tutorial.html)（英文）
- [scikit-learn 中文文档](https://sklearn.apachecn.org/)（中文）
- [Kaggle 入门竞赛](https://www.kaggle.com/competitions)（实战练习）
- [机器学习速查表](https://scikit-learn.org/stable/tutorial/machine_learning_map/)（算法选择指南）