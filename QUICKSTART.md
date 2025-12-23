# SmartVoyage 快速开始指南

## 5分钟快速部署

### 步骤1: 环境准备
```bash
# 1. 确保已安装 Python 3.8+
python --version

# 2. 安装依赖
pip install -r requirements.txt

# 3. 配置 MySQL 数据库
# 启动 MySQL 服务
# 创建数据库
mysql -u root -p -e "CREATE DATABASE travel_rag;"

# 4. 导入数据
mysql -u root -p travel_rag < sql/create_table.sql
mysql -u root -p travel_rag < sql/insert_data.sql
```

### 步骤2: 配置 API 密钥
编辑 `config.py` 文件：
```python
# 修改此行，填入您的 DashScope API 密钥
self.api_key = 'sk-your-actual-api-key-here'
```

### 步骤3: 启动代理服务器
打开三个终端窗口，分别执行：

**终端1 - 天气代理**:
```bash
python a2a_server/weather_server.py
# 监听端口: 5005
```

**终端2 - 票务查询代理**:
```bash
python a2a_server/ticket_server.py
# 监听端口: 5006
```

**终端3 - 票务预订代理**:
```bash
python a2a_server/order_server.py
# 监听端口: 5007
```

### 步骤4: 启动应用程序
#### 选项A: Web 界面 (推荐)
```bash
streamlit run app.py
```
访问: http://localhost:8501

#### 选项B: 命令行界面
```bash
python main.py
```

## 快速测试

### 测试用例
在 Web 界面或命令行中输入以下问题测试系统：

1. **天气查询**: "今天北京天气怎么样？"
2. **机票查询**: "查询明天北京到上海的航班"
3. **火车票查询**: "帮我查一下后天北京到南京的高铁票"
4. **景点推荐**: "推荐几个杭州的旅游景点"
5. **演唱会票**: "周杰伦北京演唱会的票还有吗？"

### 预期响应
- 系统会识别意图并调用相应代理
- 返回格式化的中文响应
- Web 界面右侧显示 Agent Card 信息

## 配置文件说明

### `config.py` 关键配置
```python
# 大模型配置 (必须修改)
self.base_url = 'https://dashscope.aliyuncs.com/compatible-mode/v1'
self.api_key = 'sk-'  # ← 替换为您的 API 密钥
self.model_name = 'qwen-plus'

# 数据库配置 (根据实际情况修改)
self.host = 'localhost'      # MySQL 主机
self.user = 'root'           # MySQL 用户名
self.password = 'root'       # MySQL 密码
self.database = 'travel_rag' # 数据库名
```

## 常见问题解决

### Q1: 代理服务器启动失败
**症状**: `Address already in use` 错误
**解决**:
```bash
# 查看占用端口的进程
netstat -ano | findstr :5005
# 终止进程或修改 config.py 中的端口号
```

### Q2: 数据库连接失败
**症状**: `Can't connect to MySQL server` 错误
**解决**:
1. 确保 MySQL 服务已启动
2. 检查 `config.py` 中的数据库配置
3. 确认用户权限

### Q3: API 调用失败
**症状**: `Invalid API Key` 错误
**解决**:
1. 确认 `config.py` 中的 API 密钥正确
2. 检查阿里云 DashScope 账户余额
3. 确认网络连接正常

### Q4: 依赖安装失败
**解决**:
```bash
# 使用国内镜像源
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 或使用 conda 环境
conda create -n smartvoyage python=3.9
conda activate smartvoyage
pip install -r requirements.txt
```

## 开发命令速查

### 启动所有服务 (简化版)
创建 `start_all.bat` (Windows) 或 `start_all.sh` (Linux/Mac):

**Windows (start_all.bat)**:
```batch
@echo off
start cmd /k "python a2a_server/weather_server.py"
timeout /t 2
start cmd /k "python a2a_server/ticket_server.py"
timeout /t 2
start cmd /k "python a2a_server/order_server.py"
timeout /t 2
echo 所有代理服务器已启动
echo 启动 Web 界面...
streamlit run app.py
```

**Linux/Mac (start_all.sh)**:
```bash
#!/bin/bash
# 启动代理服务器
python a2a_server/weather_server.py &
sleep 2
python a2a_server/ticket_server.py &
sleep 2
python a2a_server/order_server.py &
sleep 2

echo "所有代理服务器已启动"
echo "启动 Web 界面..."
streamlit run app.py
```

### 查看日志
```bash
# 实时查看日志
tail -f logs/app.log

# 查看错误
grep -i "error\|exception" logs/app.log

# 查看特定代理日志
grep "WeatherQueryAssistant" logs/app.log
```

## 下一步

1. **探索功能**: 尝试各种旅行相关查询
2. **查看代码**: 阅读 `app.py` 和 `main.py` 了解实现
3. **定制开发**: 根据需要修改提示词或添加新功能
4. **部署上线**: 考虑 Docker 容器化部署

---

**提示**: 首次使用建议从 Web 界面开始，体验更完整的功能和界面。