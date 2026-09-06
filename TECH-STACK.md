# RAG 智能知识库问答系统 — 技术栈梳理

> 面试准备用:每个选型都带"为什么",按「框架 → AI → 检索存储 → 异步工程化 → 前端部署 → 核心链路」的顺序讲解。

## 1. 核心框架

| 技术 | 版本 | 选型理由(面试话术) |
|------|------|------|
| Java / Spring Boot | 17 / 3.5.4 | 生态成熟,starter 全家桶减少样板代码 |
| Spring Web (MVC) | 内嵌 Tomcat | SSE 用 `Flux<ServerSentEvent>` 实现,不引入 WebFlux 全套,轻量 |
| Spring Data JPA + Hibernate | 6.6 | 元数据 CRUD 简单;复杂检索不在 MySQL,交给 ES |
| Lombok | 1.18.48 | 样板代码消除 |

## 2. AI 能力层

| 技术 | 用途 | 要点 |
|------|------|------|
| **LangChain4j** 1.19.0 | LLM 接入抽象 | 用 **策略模式**封装:`ChatModelStrategy` 接口 + `DeepSeek`/`Qwen` 实现,切换厂商只加一个类;OpenAI 兼容协议,国产模型全兼容 |
| DeepSeek(deepseek-chat) | 答案生成 | SSE 流式,按量计费便宜 |
| SiliconFlow BAAI/bge-m3 | 文本向量化 | 1024 维,中英双语效果好,有免费额度 |

## 3. 检索与存储

| 技术 | 版本 | 用途 |
|------|------|------|
| **Elasticsearch** | 8.17.0 | 官方 **Java API Client**(不是 spring-data-es,原因:需要 kNN query 的底层控制)。同一索引:`content` 走 text 倒排(BM25)、`embedding` 走 dense_vector(int8_hnsw, cosine) |
| **混合检索** | 自研 | BM25 + kNN 双路召回 → **RRF 倒数排名融合**(只关心排名,免去分数归一化)→ 应用层余弦相似度阈值过滤 |
| MySQL 8.0 | — | 文档元信息(状态机 PENDING→PARSING→SUCCESS/FAILED)、QA 记录统计 |
| Redis 7 | — | ① **语义缓存**(embedding 余弦匹配,List+LTRIM 限容量)② 每日 token 预算计数(INCR,跨实例共享) |

## 4. 异步与工程化

| 技术 | 用途 | 面试亮点 |
|------|------|---------|
| RabbitMQ 3.13 | 文档解析异步化 | 上传秒返回;消费者幂等(只处理 PENDING);失败不重抛死循环,落错误状态由用户重传 |
| Apache Tika 2.9.2 | 多格式文档解析 | PDF/Word/Markdown/txt 自动识别(注意:tika-core 需显式引入,standard-package 里是 provided) |
| Guava RateLimiter | QPS 限流(令牌桶) | 5 秒拿不到令牌直接拒绝,保护免费 API 额度 |
| SSE | 流式输出 | 五类事件:`refs`(出处)→ `token`(增量)→ `cacheHit` → `done` / `error` |

## 5. 前端 + 部署

- **前端**:单文件 `index.html`(原生 JS + EventSource),零构建 —— 项目定位是后端,前端只做演示
- **部署**:docker-compose 一键起 ES/MySQL/Redis/RabbitMQ;配置全部环境变量化(`ES_HOST`/`MYSQL_HOST`/API key 等),不硬编码

## 6. 核心链路(一张图讲完)

```
上传文档 ──► MySQL(PENDING)──► RabbitMQ ──► Tika 解析 ──► 分块(段落优先/400字/40重叠)
                                                              │
提问 ──► 限流/预算检查 ──► 语义缓存命中? ──否──► 混合检索(BM25+kNN→RRF)──► 阈值过滤
  ▲                              │是                                     │
  └────── SSE 流式输出 ◄── DeepSeek ◄── 拼装 Prompt(资料编号+问题) ◄────┘
          (refs 溯源)        完成后回写缓存 + QA 统计落库
```

## 面试时值得主动提的三个设计点

1. **为什么混合检索**:BM25 对专有名词/精确匹配强,向量检索懂语义改写;纯向量对新词/冷门文档弱 —— 两者互补(RAG 面试必考题)
2. **为什么阈值过滤在应用层**:ES 的 kNN 分数是 HNSW 距离的近似,和 BM25 分数不可比;应用层用 embedding 精确重算余弦,低于 0.5 宁可弃答(防幻觉)
3. **成本意识**:检索无结果不调大模型(省 token)、语义缓存"换个问法也命中"、每日 token 预算熔断 —— 生产 RAG 的真实痛点
