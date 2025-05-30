---
title: code with copilot-mcp server demo

author: Vi_error

description:  copilot开发初体验

categories:

  - 随手记

tags:

  - [ copilot,MCP ]

---


最近从Java转Go开发，需要完成一个Mcp Adapter，实现Rest和MCP两种请求的互转。

我之前完全没有go语言开发的技术基础，MCP也是刚接触的。在有限的时间里完成这个工作需要把精力优先放在协议本身上，因此go语法这块，只能depends on LLM了。

自从AI大火，之前也尝试过AI协助编程，但是因为在Java上写出了肌肉记忆，经常觉得打字不如自己写的快，一直没有良好实践。所以也打算借这个机会完全尝试一下。

## 任务拆分

1. MCP协议本身的调研
2. 官方SDK和非官方可用SDK的实现情况
3. Adapter的实现思路调研
4. 确定代码逻辑和SDK，基于copilot完成代码开发

前三步都属于技术调研，开发前的准备，前后大概花了一周，看了MCP官方文档，理解从MCP host、client到server的实际链路，各模型对MCP的支持情况，大厂AIInfra相关产品MCP应用的支持、SDK的可用情况、Higress+Nacos的adapter实现，基本确定了自己实现一个adapter需要完成哪些代码逻辑，测试代码实现情况需要用到哪些工具。

以下是调研结果：

1. go目前还没有官方的SDK，但已经有个使用比较官方的开源库：github.com/mark3labs/mcp-go；
2. Higress的实现简洁易懂，它实现了rest服务支持MCP，但是rest不能支持上下文，MCP的sessionId在这里也就是无效的；所以这里要分情况：
   1. 如果是rest服务转成MCP，那么上下文是没有价值的，即使每次都init client对结果影响都不大，但是有速度有影响；
   2. 如果是MCP服务转成Rest请求可以访问，那么需要处理一下MCP上下文中sessionId和rest请求中session（其他会话标识也可以）的映射关系；
3. 完成Adapter功能开发需要完成：
   1. rest 请求转MCP client请求；
   2. MCP client请求转rest请求，这个不是趋势，将存量rest转MCP过渡是有价值的，继续开发rest强转mcp client是逆潮流的；
4. 需要开发一个MCP Server、一个Rest Server，一个MCP Client和一个curl请求；

## 开发过程

接下来就是和copilot一起开发的过程了，根据程序员定律，所有开发工作从环境配置开始。

以下是部分提示词

> 我是Macbook用户，M2芯片，请指导我使用brew完成go环境安装。完成环境安装后，我将使用goland开始工程开发
> 请使用github.com/mark3labs/mcp-go作为依赖，参照以下代码（代码是mark3labs工程的示例）完成一个MCP Server的开发，MCP Server中包含一个tool，该Tool的作用是向传入的人名问好。（我创建了一个main.go文件并且将go.mod作为上下文提供给了copilot）
> 请使用github.com/mark3labs/mcp-go作为依赖，实现一个MCP client，该client可以调用刚刚完成MCP Server。（这里同样创建了一个main.go文件提供给copilot）
> 请使用github.com/mark3labs/mcp-go作为依赖，创建一个adapter，该adapter是一个restful接口，接口参数是一个String类型的人名，当接口被请求时，它会作为一个MCP client调用MCP Server完成一次请求，并将MCP server的response返回给接口调用者

LLM基本可以根据这些提示词返回可用的代码，但是完成整个功能的开发还是要求你有一些背景知识，比如：

- 你需要知道go工程的组织形式，比如gopath，goroot配置，go mod的基本命令，这些也是基于LLM快速学习和配置的，但并不体现在代码生成的过程中；
- 完成整个过程的测试需要启动多个服务，要有Server 端口配置，go服务启动等等基础知识；
- 依赖可能会报错，需要手动解决下，当然报错也可以直接丢给模型，模型会给一些解决方案，但不一定对；
- 代码可能需要调整，生成错误的时候模型是不能发现异常的，需要你自己根据官方的demo fix；
- 到生成adapter的时候，因为关键代码已经都给出来了，程序员本能会让你觉得你自己能复制粘贴的更快；

最终跑完流程的时候，对于go的语法我还是基本一无所知的。

## 总结

完成这个功能开发整体没用太多时间，可能跟我自己用Java完成这个功能的耗时差不多。

也就是说，copilot可以完全抹平简单任务上对于开发语言的要求。

以后写代码的核心能力要求可能转变为：

- 任务理解、拆分和信息搜集能力
- 关联性问题解决时的逻辑性和分析能力
- 计算机科学的背景知识：在复杂问题、性能问题中能够分辨问题解决方向的能力

写这个文章记录一下工作风格改变的触发点。

## 一些代码，但并不重要

MCP Server

``` go
package main

import (
 "context"
 "errors"
 "fmt"
 "log"
 "time"

 "github.com/mark3labs/mcp-go/mcp"
 "github.com/mark3labs/mcp-go/server"
)

func main() {
 // Create MCP server
 s := server.NewMCPServer(
  "Demo one",
  "1.0.0",
 )

 // Add greetTool
 greetTool := mcp.NewTool("hello_world",
  mcp.WithDescription("Say hello to someone"),
  mcp.WithString("name",
   mcp.Required(),
   mcp.Description("Name of the person to greet"),
  ),
 )

 // Add greetTool handler
 s.AddTool(greetTool, helloHandler)

 timeTool := mcp.NewTool("get_time",
  mcp.WithDescription("get_time"))
 s.AddTool(timeTool, timeHandler)

 // start sse server
 sseServer := server.NewSSEServer(s, server.WithBaseURL("http://localhost:20181/sse"))
 log.Printf("SSE server listening on :20181")
 if err := sseServer.Start(":20181"); err != nil {
  log.Fatalf("Server error: %v", err)
 }
 // Start the stdio server
 //if err := server.ServeStdio(s); err != nil {
 // fmt.Printf("Server error: %v\n", err)
 //}
}

func helloHandler(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
 name, ok := request.Params.Arguments["name"].(string)
 if !ok {
  return nil, errors.New("name must be a string")
 }

 return mcp.NewToolResultText(fmt.Sprintf("Hello, %s!, This is from your go mcp server", name)), nil
}
func timeHandler(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
 return mcp.NewToolResultText(time.Now().Format(time.DateTime) + "========"), nil
}
```

adpter

``` go
package main

import (
 "context"
 "encoding/json"
 "errors"
 "fmt"
 "github.com/mark3labs/mcp-go/client"
 "github.com/mark3labs/mcp-go/mcp"
 "log"
 "net/http"
 "os"
 "os/signal"
 "syscall"
 "time"
)

func main() {
 mux := http.NewServeMux()

 mux.HandleFunc("/mcp2rest", func(w http.ResponseWriter, r *http.Request) {
  fmt.Println("api query occurs ")

  if r.Method != http.MethodGet {
   w.WriteHeader(http.StatusMethodNotAllowed)
   return
  }
  w.Header().Set("Content-Type", "application/json")
  result, err := getTimeFromMCP()
  if err != nil {
   log.Printf("Error calling getTimeFromMCP: %v", err)
   w.WriteHeader(http.StatusInternalServerError)
   return
  }

  response := struct {
   Message *mcp.CallToolResult
  }{
   Message: result,
  }

  if err := json.NewEncoder(w).Encode(response); err != nil {
   log.Printf("Error encoding response: %v", err)
   w.WriteHeader(http.StatusInternalServerError)
  }
 })

 server := &http.Server{
  Addr:    ":8082",
  Handler: mux,
 }

 stopChan := make(chan os.Signal, 1)
 signal.Notify(stopChan, syscall.SIGINT, os.Interrupt)

 go func() {
  log.Println("mcp to rest adapter start")
  if err := server.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
   log.Fatalf("Server error: %v", err)
  }
 }()

 <-stopChan
 log.Println("Shutting down server...")
 if err := server.Shutdown(context.Background()); err != nil {
  log.Printf("Error shutting down server: %v\n", err)
 }

 log.Println("mcp to rest adapter end")
}

func getTimeFromMCP() (*mcp.CallToolResult, error) {
 client, err := client.NewSSEMCPClient("http://localhost:20181/sse")

 if err != nil {
  log.Printf("Failed to create client: %v", err)
 }
 defer client.Close()
 ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
 defer cancel()

 // start the client
 if err := client.Start(ctx); err != nil {
  log.Printf("Failed to start client: %v", err)
 }
 log.Printf("Client started")
 // Initialize
 initRequest := mcp.InitializeRequest{}
 initRequest.Params.ProtocolVersion = mcp.LATEST_PROTOCOL_VERSION
 initRequest.Params.ClientInfo = mcp.Implementation{
  Name: "get time",
 }
 result, err := client.Initialize(ctx, initRequest)
 if err != nil {
  log.Printf("Failed to initialize: %v", err)
 }

 if result.ServerInfo.Name != "Demo one" {
  log.Printf("Expected server name 'Demo one', got '%s'\n",
   result.ServerInfo.Name,
  )
 }
 log.Printf("Initialized: %v", result)
 // Test Ping
 if err := client.Ping(ctx); err != nil {
  log.Printf("Ping failed: %v", err)
 }

 log.Printf("Ping succeeded")

 // Test ListTools
 toolsRequest := mcp.ListToolsRequest{}
 tools, err := client.ListTools(ctx, toolsRequest)
 if err != nil {
  log.Printf("ListTools failed: %v", err)
 }
 log.Printf("Tools: %v", tools)
 for index, value := range tools.Tools {
  log.Printf("Tool index is :%v, name is : %v", index, value.Name)
 }

 request := mcp.CallToolRequest{}
 request.Params.Name = "get_time"
 response, err := client.CallTool(ctx, request)
 if err != nil {
  log.Printf("CallTool failed: %v", err)
 }

 if len(response.Content) != 1 {
  log.Printf("Expected 1 content item, got %d", len(response.Content))
 }
 log.Printf("CallTool succeeded,response: %v", response)

 return response, nil
}

```
