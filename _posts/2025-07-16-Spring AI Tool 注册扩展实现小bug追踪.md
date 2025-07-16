---
title: Spring AI Tool 注册扩展实现小bug追踪

author: Vi_error

description: 一个小bug的处理过程。Tool可以为chatClient提供更多可以使用的能力，遵从Java设计模式的惯性，我们应该也有一套支持Tool扩展但是不用修改chatClient实现的方法。

categories:

- AI/LLM/Agent

tags:

- [Spring AI,AI,Agent,Memory]

---

在写Tool层实现的时候，其实没觉得这是一个问题，写一个AbstractClass，让所有的tool 都extends AbstractClass，然后扫描bean一次性加载进去，常规套路。

但是很意外的这个套路失灵了。

代码是这样的（省略部分）：

```JAVA
public class AgentChatService {


    private final List<AbstractTool> tools;

    public Flux<String> chatWithTool(String prompt) {
        return Flux.defer(() -> chatClient.prompt(prompt).tools(tools).stream().content());
    }
}
```

报错信息一直是这样的(隐去了部分的className)：

```
No @Tool annotated methods found in [Tool@7d081b40, Tool@17618df2, Tool@27e1be76, TimeTool@3d574310].Did you mean to pass a ToolCallback or ToolCallbackProvider? If so, you have to use .toolCallbacks() instead of .tool()

```

也就是切换成这种方式，`@Tool`的注解框架突然扫不到了。

我下意识的怀疑toolList是空对象，但很快意识到不对，因为报错信息已经明确提示了具体的类。

Debug跟进去看一下，对象注入一切正常，现在有两个怀疑：

- AbstractToolList泛化有影响
- CGLIB代理有影响

断点进入框架代码：

```JAVA
        public ChatClient.ChatClientRequestSpec tools(Object... toolObjects) {
            Assert.notNull(toolObjects, "toolObjects cannot be null");
            Assert.noNullElements(toolObjects, "toolObjects cannot contain null elements");
            this.toolCallbacks.addAll(Arrays.asList(ToolCallbacks.from(toolObjects)));
            return this;
        }

```

我传入了一个什么来着？一个List。

改一下：

```JAVA

public class AgentChatService {


    private final List<AbstractTool> tools;

    public Flux<String> chatWithTool(String prompt) {
        return Flux.defer(() -> chatClient.prompt(prompt).tools(tools.toArray()).stream().content());
    }
}
```

重新测试一下，一切正常了。

细节是魔鬼。