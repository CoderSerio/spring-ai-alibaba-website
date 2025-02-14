---
title: 检索增强生成RAG（Retrieval-Augmented Generation）
keywords: [Spring AI, RAG, 模型上下文协议, 智能体应用]
description: "Spring AI 智能体如何使用RAG"
---
#  RAG 简介

## 一、 什么是RAG（检索增强生成）
RAG（Retrieval Augmented Generation，检索增强生成）是一种结合信息检索和文本生成的技术范式。

## 🌟 核心设计理念
RAG技术就像给AI装上了「实时百科大脑」，通过**先查资料后回答**的机制，让AI摆脱传统模型的"知识遗忘"困境。

## 🛠️ 四大核心步骤

### 1. 文档切割 → 建立智能档案库
- **核心任务**: 将海量文档转化为易检索的知识碎片
- **实现方式**:
   - 就像把厚重词典拆解成单词卡片
   - 采用智能分块算法保持语义连贯性
   - 给每个知识碎片打标签（如"技术规格"、"操作指南"）

> 📌 关键价值：优质的知识切割如同图书馆分类系统，决定了后续检索效率

### 2. 向量编码 → 构建语义地图
- **核心转换**:
   - 用AI模型将文字转化为数学向量
   - 使语义相近的内容产生相似数学特征
- **数据存储**:
   - 所有向量存入专用数据库
   - 建立快速检索索引（类似图书馆书目检索系统）

🎯 示例效果："续航时间"和"电池容量"会被编码为相似向量

### 3. 相似检索 → 智能资料猎人
**应答触发流程**：
1. 将用户问题转为"问题向量"
2. 通过多维度匹配策略搜索知识库：
   - 语义相似度
   - 关键词匹配度
   - 时效性权重
3. 输出指定个数最相关文档片段

### 4. 生成增强 → 专业报告撰写
**应答构建过程**：
1. 将检索结果作为指定参考资料
2. AI生成时自动关联相关知识片段。
3. 输出形式可以包含：
   - 自然语言回答
   - 附参考资料溯源路径

📝 输出示例：
> "根据《产品手册v2.3》第5章内容：该设备续航时间为..."


## 二、Spring AI 标准接口实现 RAG 

### 2.1 核心实现代码

#### 配置类
```java
@Configuration
public class RagConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder.defaultSystem("你将作为一名机器人产品的专家，对于用户的使用需求作出解答")
                .build();
    }

    @Bean
    VectorStore vectorStore(EmbeddingModel embeddingModel) {
        SimpleVectorStore simpleVectorStore = SimpleVectorStore.builder(embeddingModel)
                .build();

        // 生成一个机器人产品说明书的文档
        List<Document> documents = List.of(
                new Document("产品说明书:产品名称：智能机器人\n" +
                        "产品描述：智能机器人是一个智能设备，能够自动完成各种任务。\n" +
                        "功能：\n" +
                        "1. 自动导航：机器人能够自动导航到指定位置。\n" +
                        "2. 自动抓取：机器人能够自动抓取物品。\n" +
                        "3. 自动放置：机器人能够自动放置物品。\n"));

        simpleVectorStore.add(documents);
        return simpleVectorStore;
    }


}

```
通过这个配置类，完成以下内容：

1 、配置ChatClient作为Bean，其中设置系统默认角色为机器人产品专家， 负责处理用户查询并生成回答
向量存储配置。

2、初始化SimpleVectorStore，加载机器人产品说明书文档，将文档转换为向量形式存储。
>SimpleVectorStore是将向量保存在内存ConcurrentHashmap中，Spring AI提供了多种存储方式，如Redis、MongoDB等，可以根据实际情况选择适合的存储方式。


#### 检索增强服务
```java
@RestController
@RequestMapping("/ai")
public class RagController {

    @Autowired
    private ChatClient chatClient;

    @Autowired
    private VectorStore vectorStore;


    @PostMapping(value = "/chat", produces = "text/plain; charset=UTF-8")
    public String generation(String userInput) {
        // 发起聊天请求并处理响应
        return chatClient.prompt()
                .user(userInput)
                .advisors(new QuestionAnswerAdvisor(vectorStore))
                .call()
                .content();
    }
}
```
通过添加QuestionAnswerAdvisor并提供对应的向量存储，可以将之前放入的文档作为参考资料，并生成增强回答。

### 2.2 运行程序
启动Spring Boot应用程序，并访问`/ai/chat`接口，传入用户问题，即可获取增强回答。如下：
```bash
POST http://localhost:8080/spring-ai/ai/chat?userInput=机器人有哪些功能？

HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8

根据您提供的智能机器人产品说明书，该机器人的主要功能包括：

1. 自动导航：机器人可以自动导航到指定的位置。
2. 自动抓取：机器人能够自动抓取物品。
3. 自动放置：机器人能够自动放置物品。

如果您需要更详细的信息或者关于其他功能的问题，请提供具体的需求，我会尽力帮助您。

```
这样测试结果，可以清晰地看到AI生成的回答，并参考了机器人产品说明书中的相关信息。

