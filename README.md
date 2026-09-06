# RAG 智能知识库问答系统

Java 后端面试项目:企业级智能知识库。上传文档 → 自动解析入库 → 自然语言提问 → 检索相关资料 → 大模型生成**带原文出处**的答案。

> 完整项目规划与面试 Q&A 模拟见 `../RAG项目规划.md`

## 架构(两条链路)

```
【入库链路】上传文档 → MQ 异步 → Tika 解析 → 分块(overlap) → Embedding 向量化 → ES 索引(状态机管理)
【问答链路】提问 → 限流 → 语义缓存(Redis) → 混合检索(BM25 + kNN, RRF 融合) → 相似度过滤
            → Prompt 拼装 → 大模型 SSE 流式回答 → 引用溯源 + token 统计 + 缓存回写
```

## 技术栈

| 组件 | 选型 | 说明 |
|---|---|---|
| 框架 | Spring Boot 3.5 + JDK 17 | |
| AI 框架 | LangChain4j 1.19 | 策略模式封装,支持 DeepSeek / 通义千问切换 |
| 大模型 | DeepSeek(OpenAI 兼容) | 默认 `deepseek-chat` |
| Embedding | BGE-M3(SiliconFlow 托管) | 1024 维,有免费额度 |
| 向量检索 | Elasticsearch 8.17 | dense_vector + kNN + BM25 混合检索 |
| 缓存 | Redis | 语义缓存(embedding 余弦相似度) |
| 异步 | RabbitMQ | 文档解析任务队列 |
| 数据库 | MySQL 8 | 文档元信息、问答统计 |
| 流式输出 | SSE | 打字机效果 |
| 文档解析 | Apache Tika | PDF / Word / Markdown / txt |

## 快速开始

### 1. 申请 API Key(两个都要)

- **DeepSeek**:https://platform.deepseek.com → 充值少量金额即可(几块钱够演示很久)
- **SiliconFlow**(embedding,免费):https://siliconflow.cn → 注册后在 API 密钥页创建

### 2. 启动中间件

```bash
docker compose up -d
# Elasticsearch :9200 | Redis :6379 | RabbitMQ :5672(管理台 :15672) | MySQL :3306
```

### 3. 配置密钥并启动

环境变量方式(推荐):

```bash
export CHAT_API_KEY=sk-你的deepseek密钥
export EMBEDDING_API_KEY=sk-你的siliconflow密钥
mvn spring-boot:run
```

或者直接修改 `src/main/resources/application.yml` 中的 `rag.llm.*.api-key`。

### 4. 打开页面

访问 http://localhost:8080,上传 `docs/sample.md`,等状态变为「已入库」后提问:

- 「缓存穿透怎么解决?」(→ 命中 kNN + BM25)
- 「为什么会有缓存穿透、击穿、雪崩?换个说法问同样的问题」(→ 第二次命中语义缓存)
- 「Spring 事务什么时候失效?」(→ 答案附引用出处)

## 项目结构

```
src/main/java/com/interview/rag/
├── RagApplication.java
├── config/          # 配置与模型策略(策略模式)
│   ├── ChatModelStrategy.java / DeepSeek... / Qwen... / ChatModelManager
│   ├── LlmConfig / ElasticsearchConfig / RabbitConfig / WebConfig
│   └── *Properties(rag.* 配置绑定)
├── controller/      # DocumentController(上传/删除) ChatController(SSE) StatsController
├── domain/          # DocumentEntity / QaRecord / ParseStatus 状态机
├── repository/      # JPA
├── mq/              # 解析任务生产者/消费者
├── service/
│   ├── DocumentParseService   # Tika 解析 + 入库(消费者线程,幂等)
│   ├── ChunkingService        # 段落/句子级分块 + overlap
│   ├── EmbeddingService       # 文本向量化
│   ├── VectorIndexService     # ES 索引 + BM25/kNN 双路检索
│   ├── HybridSearchService    # RRF 融合 + 余弦相似度阈值过滤
│   ├── ChatService            # RAG 编排 + SSE 流式输出
│   ├── SemanticCacheService   # Redis 语义缓存
│   └── RateLimitService       # 令牌桶限流 + 每日 token 预算
├── model/           # Chunk / DocFragment / Reference / StatsResponse
├── util/            # VectorUtil(余弦相似度)
└── exception/       # 统一异常处理
```

## 面试要点速览

- **模型可插拔**:策略模式 + OpenAI 兼容协议,配置一个 `provider` 切换厂商
- **为什么混合检索**:BM25 懂关键词(专有名词/精确匹配),向量懂语义;RRF 只比排名不比分数,天然免归一化
- **相似度阈值过滤在应用层做**:精确控制余弦相似度语义,防止答非所问(防幻觉)
- **异步解析**:MQ 削峰 + 状态机幂等 + 失败落库不重投
- **成本控制**:语义缓存(相似问题 0 token)、检索无结果不调模型、每日 token 预算、QPS 限流
- **SSE vs WebSocket**:单向推送场景,HTTP 原生、代理友好

## 常见问题

- **ES 起不来**:Windows 需 WSL2 + Docker Desktop;ES 首次启动较慢,等 30 秒左右
- **提示维度不匹配**:更换 embedding 模型时,同步改 `rag.llm.embedding.dimension`,并删除旧索引重启
- **免费额度打爆**:默认 QPS=5,可在 `rag.rate-limit.qps` 调整
- **中文 BM25 检索效果一般**:标准分词器对中文按单字切分,生产可装 IK 分词插件,演示不影响(向量路兜底)
- **依赖下载慢**:Maven 建议配置阿里云镜像
