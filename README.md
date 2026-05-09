# 强脑科技产品综合知识问答系统

## 系统简介

本系统在原有集成架构上改造为**智能假肢与康复辅具产品综合知识问答**：结合 MySQL 结构化问答（高频产品 FAQ）与 RAG 非结构化知识库（以浙江省强脑科技有限公司及相关产品线为参考），根据问题自动选择检索与生成策略。

### 核心功能

- **多源知识集成**：结合MySQL结构化知识库和RAG非结构化知识库
- **智能问答**：根据问题类型自动选择最佳回答策略
- **流式输出**：支持WebSocket实时流式回答
- **对话历史管理**：记录和管理用户对话历史
- **产品知识源**：RAG 知识类别为 `product`（见 `rag_qa/data/product_data/`）
- **日常问候识别**：自动识别和回复日常问候语

## 系统架构

系统采用分层架构设计，主要包含以下组件：

1. **API层**：基于FastAPI构建的Web服务，提供HTTP和WebSocket接口
2. **业务逻辑层**：IntegratedQASystem核心类，协调各模块工作
3. **数据层**：
   - MySQL：存储结构化知识和对话历史
   - Redis：提供缓存服务
   - Milvus：向量数据库，存储文档嵌入
4. **检索层**：
   - BM25搜索：用于快速文本搜索
   - 向量检索：用于语义相似度搜索
5. **生成层**：调用DashScope API生成答案

## 技术栈

- **后端**：Python 3.10+, FastAPI
- **数据库**：MySQL, Redis, Milvus
- **AI模型**：
  - BERT：文本分类
  - BGE-M3：向量嵌入
  - Qwen-Plus：大语言模型
- **API**：DashScope (阿里云)

## 目录结构

```
integrated_qa_system/
├── base/                 # 基础模块
│   ├── __init__.py
│   ├── config.py         # 配置管理
│   └── logger.py         # 日志管理
├── mysql_qa/             # MySQL问答模块
│   ├── cache/            # 缓存相关
│   ├── data/             # 数据文件
│   ├── db/               # 数据库操作
│   ├── retrieval/        # 检索相关
│   └── utils/            # 工具函数
├── rag_qa/               # RAG问答模块
│   ├── classify_data/    # 分类数据
│   ├── core/             # 核心组件
│   ├── data/             # 知识库数据
│   ├── edu_document_loaders/  # 文档加载器
│   ├── edu_text_spliter/      # 文本分割器
│   └── models/           # 模型文件
├── logs/                 # 日志文件
├── static/               # 静态文件
├── api.py                # API接口
├── app.py                # 主应用
├── config.ini            # 配置文件
├── new_main.py           # 新主模块
└── old_main.py           # 旧主模块
```

## 安装与配置

### 前提条件

- Python 3.10 或更高版本
- MySQL 5.7 或更高版本
- Redis 6.0 或更高版本
- Milvus 2.0 或更高版本
- 阿里云DashScope API密钥

### 安装步骤

1. **克隆代码**

   ```bash
   git clone <repository-url>
   cd integrated_qa_system
   ```

2. **安装依赖**

   ```bash
   pip install -r requirements.txt
   ```

3. **配置数据库**

   - 启动MySQL服务
   - 创建数据库 `subjects_kg`
   - 导入 `mysql_qa/data/产品综合知识.csv` 数据（若曾导入过旧版 IT 题目，请先清空表 `jpkb`，并在 Redis 删除键 `qa_original_questions`、`qa_tokenized_questions` 以免 BM25 仍用旧缓存）

4. **配置Redis**

   - 启动Redis服务
   - 确保Redis密码与配置文件一致

5. **配置Milvus**

   - 启动Milvus服务
   - 创建数据库 `product_kb` 和集合 `product_rag`（与 `config.ini` 中 Milvus 配置一致；若从旧库迁移，请使用新库名并**重新灌入向量**）
   - 在 `rag_qa` 目录下执行建库索引（示例）：`python rag_main.py --data-processing`，将向 `product_data` 目录中的 Markdown 建索引并写入 Milvus

6. **配置API密钥**

   - 修改 `config.ini` 文件中的 `dashscope_api_key` 为你的实际API密钥

7. **（可选）查询分类 BERT**

   - RAG 路由使用 `rag_qa/core/bert_query_classifier`；更换语料后若需「专业咨询 / 通用知识」判定与假肢场景一致，请用 `rag_qa/classify_data/model_generic_5000.json` 按 `query_classifier.py` 中说明重新训练并覆盖该目录下权重。

## 运行系统

### 启动Web服务

```bash
python app.py
```

服务将在 `http://localhost:8000` 启动。

### 命令行交互

```bash
python new_main.py
```

## API接口

### 1. 非流式查询

- **URL**: `/api/query`
- **方法**: POST
- **请求体**:
  ```json
  {
    "query": "你的问题",
    "source_filter": "可选的知识过滤",
    "session_id": "可选的会话ID"
  }
  ```
- **响应**:
  ```json
  {
    "answer": "回答内容",
    "is_streaming": false,
    "session_id": "会话ID",
    "processing_time": 0.123
  }
  ```

### 2. 流式查询（WebSocket）

- **URL**: `/api/stream`
- **方法**: WebSocket
- **消息格式**:
  ```json
  {
    "query": "你的问题",
    "source_filter": "可选的知识过滤",
    "session_id": "可选的会话ID"
  }
  ```
- **响应格式**:
  - 开始: `{"type": "start", "session_id": "会话ID"}`
  - 令牌: `{"type": "token", "token": "回答片段", "session_id": "会话ID"}`
  - 结束: `{"type": "end", "session_id": "会话ID", "is_complete": true, "processing_time": 1.234}`
  - 错误: `{"type": "error", "error": "错误信息"}`

### 3. 会话管理

- **创建会话**: POST `/api/create_session`
- **获取历史**: GET `/api/history/{session_id}`
- **清除历史**: DELETE `/api/history/{session_id}`

### 4. 其他接口

- **健康检查**: GET `/health`
- **获取支持的知识**: GET `/api/sources`

## 系统工作流程

1. **接收查询**：系统接收用户的问题
2. **问候语检测**：检查是否为日常问候语
3. **BM25搜索**：在MySQL中搜索相关答案
4. **判断是否需要RAG**：根据搜索结果的置信度判断
5. **RAG处理**：如果需要，使用RAG系统处理
6. **生成答案**：调用LLM生成最终答案
7. **流式输出**：通过WebSocket实时返回答案
8. **更新历史**：将对话记录保存到数据库

## 配置说明

配置文件 `config.ini` 包含以下主要配置项：

- **mysql**：MySQL数据库连接信息
- **redis**：Redis缓存连接信息
- **milvus**：Milvus向量数据库配置
- **llm**：大语言模型配置
- **retrieval**：检索参数配置
- **app**：应用相关配置

## 示例使用

### 命令行示例

```bash
$ python new_main.py

欢迎使用集成问答系统！
会话ID: 12345678-1234-1234-1234-1234567890ab
支持的知识类别：['product']
输入查询进行问答，输入 'exit' 退出。

输入查询: 仿生手靠什么识别动作
请输入知识类别 (product) (直接回车默认不过滤): product

答案: （示例）主要通过前臂残肢表面采集的肌电等生物电信号，经算法识别运动意图后驱动手指动作。

最近对话历史:
1. 问: 仿生手靠什么识别动作
   答: （示例）肌电等生物电信号经算法映射为手指动作。
```

### API示例

#### 非流式查询

```bash
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"浙江强脑主营方向是什么\", \"source_filter\": \"product\"}"
```

#### WebSocket示例

```javascript
const socket = new WebSocket('ws://localhost:8000/api/stream');

socket.onopen = () => {
  socket.send(JSON.stringify({
    "query": "接受腔是什么",
    "source_filter": "product"
  }));
};

socket.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'token') {
    console.log('Received token:', data.token);
  } else if (data.type === 'end') {
    console.log('Conversation ended');
  }
};
```

## 故障排除

### 常见问题

1. **无法连接数据库**
   - 检查MySQL和Redis服务是否运行
   - 验证配置文件中的连接信息

2. **API调用失败**
   - 检查DashScope API密钥是否正确
   - 确保网络连接正常

3. **RAG检索无结果**
   - 检查Milvus服务是否运行
   - 验证知识库数据是否正确导入

4. **系统响应缓慢**
   - 检查服务器资源使用情况
   - 优化数据库查询和检索参数

## 日志管理

系统日志存储在 `logs/app.log` 文件中，包含系统运行状态、错误信息和查询记录。

## 性能优化

1. **缓存策略**：使用Redis缓存热门查询结果
2. **索引优化**：为MySQL查询添加适当的索引
3. **检索参数调优**：根据实际情况调整检索参数
4. **并发处理**：使用异步处理提高并发能力

## 未来规划

- **多语言支持**：增加对多语言的支持
- **知识图谱集成**：整合知识图谱提高回答质量
- **个性化推荐**：根据用户历史提供个性化回答
- **模型微调**：针对特定领域微调语言模型
- **前端界面**：开发更友好的前端界面

## 贡献

欢迎提交Issue和Pull Request来改进系统。

## 许可证

本项目采用MIT许可证。

## 联系我们

如有问题或建议，请联系：
- 客服电话：12345678
- 邮箱：contact@example.com