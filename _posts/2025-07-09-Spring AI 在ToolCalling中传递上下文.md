---
title: Spring AI 在ToolCalling中传递上下文

author: Vi_error

description: Webflux底层的reactor线程模型使chatClient入口出的Threadlocal context无法传递到tool层，如何解决这个问题

categories:

- AI/LLM/Agent

tags:

- [Spring AI,AI,Agent,Reactor，ToolCalling]

---

## 背景

最近一直欠着一篇Spring AI Alibaba框架的调研+起步指南，那篇写到一半之后因为最近每天加班到10点钟实在无法继续完成。

今天算是跑通了全流程并且和之前的业务流程结合到了一起，虽然现在也是十点多了，但在小小的阶段点上，还是需要抓紧时间记录一下。不然下次什么时候才有时间呢。

下午一直在尝试解决一个问题：如何比较简洁在ToolCalling中传递上下文，并且保证Tool后的链路都能持续保有上下文。

首先解释一下为什么需要处理上下文。

如果是手写一个传统的服务，遵循 controller->service->dao这样的顺序同步编程，那么保存在线程中的用户信息将会链路中的每一环随时取得，如果我们打开业务代码，会看到非常多的处理都是依赖于Threadlocal中的信息去处理。

但是当我们使用Spring AI框架weblux模式编程的时候，传统的线程信息传递就被改变了，webflux底层依赖的reactor模型不会始终在同一个线程中执行任务，如果不做特殊的处理，在线程发生切换的时候，threadlocal中的信息也就会全部丢失。

## demo代码

给一些demo代码用于说明流程。

controller层

```Java
    @PostMapping("/ai/chat")
    @Operation(summary = "DashScope Flux Chat")
    public Flux<String> chatFlux(HttpServletResponse response, @Validated @RequestBody String prompt,
        @RequestHeader(value = "chatId", required = false,
            defaultValue = "spring-ai-alibaba-playground-chat") String chatId) {
        response.setCharacterEncoding("UTF-8");
        return chatService.chatWithTool(prompt);
    }
```

service层

```Java
public class ChatService {

    private final ChatClient chatClient;

    private final TimeTool timeTool;

    public ChatService(SimpleLoggerAdvisor simpleLoggerAdvisor,
        MessageChatMemoryAdvisor messageChatMemoryAdvisor, @Qualifier("dashscopeChatModel") ChatModel chatModel,
        @Qualifier("systemPromptTemplate") PromptTemplate systemPromptTemplate, TimeTool timeTool) {

        this.chatClient = ChatClient.builder(chatModel).defaultSystem(systemPromptTemplate.getTemplate())
            .defaultAdvisors(simpleLoggerAdvisor, messageChatMemoryAdvisor).build();

        this.timeTool = timeTool;
    }

    public Flux<String> chatWithTool(String prompt) {
        return Flux.defer(() -> {
            return chatClient.prompt(prompt).tools(timeTool).stream().content();
        });
    }

}
```

Tool

```Java
    @Tool(description = "获取当前UTC秒级时间戳")
    public Long currentTimestamp() {
        log.info("currentTimestamp LLM use this tool to query message, params: no params");
        return Instant.now().getEpochSecond();
    }
```

这些代码其实就完成了一套最简单的支持ToolCalling聊天机器人后端接口。

整个调用流程是这样的 ：

![Spring AI ToolCalling调用流程示意图](../posts_image/Spring%20AI%20ToolCalling调用流程示意图.png)

1. 用户发送一个chat请求给后端代码
2. 后端代码把请求中prompt加工后发给llm
3. llm判断是不是需要调用工具（ToolCalling）完成，如果需要的话，会生成对应tool的请求
4. 框架层根据LLM生成的请求调用tool，并把tool的返回还给LLM
5. LLM解析工具的返回，生成回答
6. web的用户接收服务返回

也就是说，从上面的service代码转到后面tool代码，中间会经过llm和框架层的处理，中间会发生线程切换，threadlocal信息会丢失。

## reactor如何实现线程切换

上面的问题肯定不是Spring AI特有的问题，在reactor上一定是一个早就有了的问题。Reactor提供了 ContextAPI解决线程切换时的上下文传递的问题。

[Reactor官方关于ContextAPI的说明](https://projectreactor.io/docs/core/release/reference/advancedFeatures/context.html)

[Spring关于ContextAPI的说明](https://spring.io/blog/2023/03/28/context-propagation-with-project-reactor-1-the-basics#reactor-context)

Spring官方给出的线程切换例子：

```Java

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.util.context.Context;

import java.time.Duration;

public class Main {
    public static void main(String[] args) {
        Main main = new Main();
        main.handleRequest().doOnSuccess(v -> System.out.println("Request processed"))
            .doOnError(e -> System.err.println("Error: " + e)).block(Duration.ofSeconds(1));
    }

    Mono<Void> handleRequest() {
        String correlationId = correlationId();
        log("Assembling the chain", correlationId);

        return Mono.just("test-product").delayElement(Duration.ofMillis(1))
            .flatMap(product -> Flux.concat(addProduct(product), notifyShop(product).then()).then())
            .contextWrite(Context.of("CORRELATION_ID", correlationId));
    }

    static String correlationId() {
        return "Init Value";
    }

    Mono<Void> addProduct(String productName) {
        return Mono.deferContextual(ctx -> {
            log("Adding product: " + productName, ctx.get("CORRELATION_ID"));
            return Mono.empty(); 
        });
    }

    Mono<Boolean> notifyShop(String productName) {
        return Mono.deferContextual(ctx -> {
            log("Notifying shop about: " + productName, ctx.get("CORRELATION_ID"));
            return Mono.just(true);
        });
    }

    static void log(String message, String correlationId) {
        String threadName = Thread.currentThread().getName();
        String threadNameTail = threadName.substring(Math.max(0, threadName.length() - 10));
        System.out.printf("[%10s][%20s] %s%n", threadNameTail, correlationId, message);
    }
}
```

最后控制台的打印为：

```
[      main][          Init Value] Assembling the chain
[parallel-1][          Init Value] Adding product: test-product
[parallel-1][          Init Value] Notifying shop about: test-product
Request processed
```

这里有个反直觉的小细节，我们修改一下代码。

```Java
    Mono<Void> handleRequest() {
        String correlationId = correlationId();
        log("Assembling the chain", correlationId);

        return Mono.just("test-product").delayElement(Duration.ofMillis(1))
            .flatMap(product -> Flux.concat(addProduct(product), notifyShop(product).then()).then())
            .contextWrite(Context.of("CORRELATION_ID", "reset context"))
            .contextWrite(Context.of("CORRELATION_ID", correlationId));
    }

    static String correlationId() {
        return "Init Value";
    }
```

重新执行代码，输出为：

```
[      main][          Init Value] Assembling the chain
[parallel-1][       reset context] Adding product: test-product
[parallel-1][       reset context] Notifying shop about: test-product
Request processed
```

按照过去的经验，应该是写在语句最后的set生效，在reactor中是写在前面的生效。

## 在ToolCalling中传递上下文

ToolCalling给了更简单的上下文传递办法：ToolContext。

相对于上面Reactor的方式，使用ToolContext会提供一种更简单明了（符合传统编程习惯）的context传递方式。

[Spring AI官方示例](https://www.spring-doc.cn/spring-ai/1.0.0-SNAPSHOT/api_tools.html#_tool_context)

Service层，set Context

```
ChatModel chatModel = ...

String response = ChatClient.create(chatModel)
        .prompt("Tell me more about the customer with ID 42")
        .tools(new CustomerTools())
        .toolContext(Map.of("tenantId", "acme"))
        .call()
        .content();
System.out.println(response);
```

Tool层，使用Context

```
class CustomerTools {

    @Tool(description = "Retrieve customer information")
    Customer getCustomerInfo(Long id, ToolContext toolContext) {
        return customerRepository.findById(id, toolContext.get("tenantId"));
    }

}
```

这个代码的书写和使用难度显然就很低了，基本上不需要理解特殊的语法就能完全掌握。

但是这边还有一个新的问题，先看一下ToolContext的代码：

```
package org.springframework.ai.chat.model;

import java.util.Collections;
import java.util.List;
import java.util.Map;
import org.springframework.ai.chat.messages.Message;

public final class ToolContext {
    public static final String TOOL_CALL_HISTORY = "TOOL_CALL_HISTORY";
    private final Map<String, Object> context;

    public ToolContext(Map<String, Object> context) {
        this.context = Collections.unmodifiableMap(context);
    }

    public Map<String, Object> getContext() {
        return this.context;
    }

    public List<Message> getToolCallHistory() {
        return (List)this.context.get("TOOL_CALL_HISTORY");
    }
}
```

Spring AI定义ToolContext用来存储上下文和TOOL_CALL_HISTORY，它虽然有效的存储了所有我们需要的信息，但相对于Threadlocal的方式，仍然有两个问题：

1. 每个Tool方法的参数都要带上ToolContext作为变量，这个有点儿麻烦
2. 如果Tool方法中调用了其他方法，而其他方法仍然使用了Threadlocal的读取，就仍然还有问题

经历了若干不太成功的尝试之后，问题2的解法可以这样：

```Java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.ai.chat.model.ToolContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
public class ToolContextAspect {
    @Around("@annotation(org.springframework.ai.tool.annotation.Tool)")
    public Object aroundToolMethod(ProceedingJoinPoint pjp) throws Throwable {
        Object[] args = pjp.getArgs();
        MethodSignature signature = (MethodSignature)pjp.getSignature();
        Method method = signature.getMethod();
        Class<?>[] parameterTypes = method.getParameterTypes();

        ToolContext toolContext = null;
        for (int i = 0; i < parameterTypes.length; i++) {
            if (CommonConstants.THREAD_CONTEXT.equals(parameterTypes[i].getSimpleName())) {
                toolContext = (ToolContext)args[i];
                break;
            }
        }

        if (toolContext != null) {
            ThreadContextUtil.setContext((ThreadContext)toolContext.getContext().get(CommonConstants.THREAD_CONTEXT));
        }

        try {
            return pjp.proceed();
        } finally {
            if (toolContext != null) {
                ThreadContextUtil.clear();
            }
        }
    }
}
```

这样整个业务流程就跑起来了。仍然还有待解决的问题：

1. 灵活的Tool扩展机制，保证Tool能自动注册到chatClient的链路中
2. 对于一个产品，如何同时优雅地提供MCP Server和内部的ToolCalling实现

仍在解决中
