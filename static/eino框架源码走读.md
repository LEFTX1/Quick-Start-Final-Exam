
# 快速开始
``` Go
  
func main() {  
    ctx := context.Background()  
  
    // 1. 创建对话模板  
    template := prompt.FromMessages(schema.FString,  
       // 系统消息模板  
       schema.SystemMessage("你是一个{role}。你需要用{style}的语气回答问题。你的目标是帮助程序员保持积极乐观的心态，提供技术建议的同时也要关注他们的心理健康。"),  
  
       // 插入需要的对话历史（新对话的话这里不填）  
       schema.MessagesPlaceholder("chat_history", true), // optional=true 允许历史为空  
  
       // 用户消息模板  
       schema.UserMessage("问题: {question}"),  
    )  
  
    // 2. 准备模板变量  
    variables := map[string]any{  
       "role":     "程序员鼓励师",  
       "style":    "积极、温暖且专业",  
       "question": "我的代码一直报错，感觉好沮丧，该怎么办？",  
       // 对话历史（这个例子里模拟两轮对话历史）  
       "chat_history": []*schema.Message{  
          schema.UserMessage("你好"),  
          schema.AssistantMessage("嘿！我是你的程序员鼓励师！记住，每个优秀的程序员都是从 Debug 中成长起来的。有什么我可以帮你的吗？", nil),  
          schema.UserMessage("我觉得自己写的代码太烂了"),  
          schema.AssistantMessage("每个程序员都经历过这个阶段！重要的是你在不断学习和进步。让我们一起看看代码，我相信通过重构和优化，它会变得更好。记住，Rome wasn't built in a day，代码质量是通过持续改进来提升的。", nil),  
       },  
    }  
  
    // 3. 使用模板生成消息  
    messages, err := template.Format(ctx, variables)  
    if err != nil {  
       log.Fatalf("模板格式化失败: %v", err)  
    }  
  
    // 4. 初始化 QwQ 模型 (通过 OpenAI 兼容接口)  
    config := &openai.ChatModelConfig{  
       APIKey:  "sk-e0d7dda93eb14ff5bcbf43f484fe6736",               // 使用您提供的 API Key       BaseURL: "https://dashscope.aliyuncs.com/compatible-mode/v1", // 设置为阿里云兼容接口  
       Model:   "qwq-plus",                                          // 可以显式指定模型，如果需要的话  
    }  
    chatModel, err := openai.NewChatModel(ctx, config)  
    if err != nil {  
       log.Fatalf("创建 ChatModel 失败: %v", err)  
    }  
  
    // 5. 运行 ChatModel (Generate 模式)  
    fmt.Println("--- Generate 模式 ---")  
    result, err := chatModel.Generate(ctx, messages)  
    if err != nil {  
       log.Printf("Generate 失败: %v", err) // 使用 Printf 而不是 Fatalf，以便继续执行 Stream    } else {  
       fmt.Printf("模型回复: %s\n", result.Content)  
       if result.ResponseMeta != nil /* && result.ResponseMeta.TokenUsage != nil */ { // 移除 TokenUsage 检查  
          // fmt.Printf("Token 使用: %+v\n", *result.ResponseMeta.TokenUsage) // 移除 TokenUsage 打印  
          fmt.Printf("响应元信息: %+v\n", *result.ResponseMeta) // 可以选择打印整个 MetaData       }  
    }  
  
    fmt.Println("--- Stream 模式 ---")  
    // 6. 运行 ChatModel (Stream 模式)  
    streamResult, err := chatModel.Stream(ctx, messages)  
    if err != nil {  
       log.Fatalf("Stream 失败: %v", err)  
    }  
    reportStream(streamResult)  
  
    fmt.Println("--- 执行完毕 ---")  
}  
  
// reportStream 处理并打印流式响应  
func reportStream(sr *schema.StreamReader[*schema.Message]) {  
    if sr == nil {  
       log.Println("StreamReader 为 nil，无法处理流式响应")  
       return  
    }  
    defer sr.Close()  
  
    fmt.Println("开始接收流式响应...")  
    fullResponse := ""  
    // var finalTokenUsage *schema.TokenUsage // 移除未使用的变量  
  
    i := 0  
    for {  
       chunk, err := sr.Recv()  
       if err == io.EOF { // 流式输出结束  
          fmt.Println("流式响应接收完毕。")  
          break  
       }  
       if err != nil {  
          log.Printf("接收流式响应失败: %v", err)  
          return // 出现错误则停止处理  
       }  
       if chunk != nil {  
          fmt.Printf("Chunk[%d]: %s\n", i, chunk.Content) // 修复：添加换行符 \n          fullResponse += chunk.Content  
  
          if chunk.ResponseMeta != nil /* && chunk.ResponseMeta.TokenUsage != nil */ { // 移除 TokenUsage 检查  
             // finalTokenUsage = chunk.ResponseMeta.TokenUsage // 移除 TokenUsage 赋值  
             // 通常最后一个 chunk 包含元信息, 但 TokenUsage 可能不直接可用  
             // 可以在这里记录或处理 chunk.ResponseMeta          }  
       }  
       i++  
    }  
  
    fmt.Println("\n--- 流式响应完整内容 ---")  
    fmt.Println(fullResponse)  
    /* if finalTokenUsage != nil { // 移除 TokenUsage 打印  
       fmt.Printf("Token 使用: %+v\n", *finalTokenUsage)  
    } */  
}
```

## 1创建对话模版
``` Go
// 1. 创建对话模板  
template := prompt.FromMessages(schema.FString,  
    // 系统消息模板  
    schema.SystemMessage("你是一个{role}。你需要用{style}的语气回答问题。你的目标是帮助程序员保持积极乐观的心态，提供技术建议的同时也要关注他们的心理健康。"),  
  
    // 插入需要的对话历史（新对话的话这里不填）  
    schema.MessagesPlaceholder("chat_history", true), // optional=true 允许历史为空  
  
    // 用户消息模板  
    schema.UserMessage("问题: {question}"),  
)

----------------------------------------
// FromMessages 从 给定的模板 和 格式类型 创建一个新的 DefaultChatTemplate  
// 例如:  
//  
//  template := prompt.FromMessages(schema.FString, &schema.Message{Content: "Hello, {name}!"}, &schema.Message{Content: "how are you?"})  
//   在 chain 或 graph 中使用  
//  chain := compose.NewChain[map[string]any, []*schema.Message]()  
//  chain.AppendChatTemplate(template)  
func FromMessages(
formatType schema.FormatType, 
templates ...schema.MessagesTemplate) *DefaultChatTemplate {  
    return &DefaultChatTemplate{  
       templates:  templates,  // 设置模板列表  
       formatType: formatType, // 设置格式化类型  
    }  
}

type FormatType uint8  
  
const (  
    // FString Python 风格的格式化字符串，由 pyfmt 库支持，实现了 PEP-3101 规范  
    FString FormatType = 0  
    // GoTemplate Go 标准库的模板系统  
    GoTemplate FormatType = 1  
    // Jinja2 类似 Python Jinja2 模板的实现，由 gonja 库提供  
    Jinja2 FormatType = 2  
)
// DefaultChatTemplate 是默认的聊天模板实现  
// 用于将多个消息模板组合成一个完整的模板  
type DefaultChatTemplate struct {  
    // templates 是聊天模板的模板列表  
    // 每个元素都实现了 MessagesTemplate 接口  
    templates []schema.MessagesTemplate  
    // formatType 是聊天模板的格式化类型  
    // 支持 FString、GoTemplate 和 Jinja2 三种格式  
    formatType schema.FormatType  
}

// MessagesTemplate 是消息模板的接口  
// 用于将模板渲染为消息列表  
// 例如：  
//  
//  chatTemplate := prompt.FromMessages(  
//     schema.SystemMessage("you are eino helper"),  
//     schema.MessagesPlaceholder("history", false), // <= 这将使用 params 中 "history" 的值  
//  )  
//  msgs, err := chatTemplate.Format(ctx, params)  
type MessagesTemplate interface {  
    // Format 使用指定的格式类型和参数将模板渲染为消息列表  
    Format(ctx context.Context, 
    vs map[string]any, 
    formatType FormatType) ([]*Message, error)  
}

  
// Message 是 Eino 框架中最核心的消息结构，用于表示各种来源的消息  
// 如用户输入、系统指令、模型回复、工具输出等  
type Message struct {  
    // Role 表示消息的角色（来源），如 "user", "assistant", "system", "tool"    Role RoleType `json:"role"`  
    // Content 消息的主要文本内容  
    Content string `json:"content"`  
  
    // MultiContent 多模态内容数组  
    // 如果 MultiContent 不为空，则使用它而不是 Content  
    // 如果 MultiContent 为空，则使用 Content  
    MultiContent []ChatMessagePart `json:"multi_content,omitempty"`  
  
    // Name 消息的名称，可选  
    Name string `json:"name,omitempty"`  
  
    // ToolCalls 仅用于 AssistantMessage（模型生成的消息）  
    // 包含模型要求调用的工具列表  
    ToolCalls []ToolCall `json:"tool_calls,omitempty"`  
  
    // ToolCallID 仅用于 ToolMessage（工具调用结果消息）  
    // 指示这条消息是哪个工具调用的结果  
    ToolCallID string `json:"tool_call_id,omitempty"`  
  
    // ResponseMeta 包含响应相关的元数据，如结束原因、token 使用情况等  
    ResponseMeta *ResponseMeta `json:"response_meta,omitempty"`  
  
    // Extra 用于存储模型实现的自定义信息  
    Extra map[string]any `json:"extra,omitempty"`  
}

  
// messagesPlaceholder 是一个占位符，用于在模板中引用变量  
type messagesPlaceholder struct {  
    // key 是参数映射中的键名  
    key string  
    // optional 表示该占位符是否可选（如果为 false 且找不到键，会返回错误）  
    optional bool  
}

  
// MessagesPlaceholder 可以将参数中的消息列表渲染到模板中的占位符位置  
// 例如：  
//  
//  placeholder := MessagesPlaceholder("history", false)  
//  params := map[string]any{  
//     "history": []*schema.Message{{Role: "user", Content: "what is eino?"}, {Role: "assistant", Content: "eino is a great freamwork to build llm apps"}},  
//     "query": "how to use eino?",  
//  }  
//  chatTemplate := chatTpl := prompt.FromMessages(  
//     schema.SystemMessage("you are eino helper"),  
//     schema.MessagesPlaceholder("history", false), // <= 这将使用 params 中 "history" 的值  
//  )  
//  msgs, err := chatTemplate.Format(ctx, params)  
func MessagesPlaceholder(key string, optional bool) MessagesTemplate {  
    return &messagesPlaceholder{  
       key:      key,  
       optional: optional,  
    }  
}
```

### 什么是schema？
**规范 (Specification/Standard)**:

- “规范” 强调了 “schema” 作为一套必须遵守的规则或标准。
    
- 它适用于描述 API 的整体结构（可以说 “API 规范” 包含了请求和响应的格式规范）以及函数调用的定义（函数的 “参数规范”）。
    
- 这个翻译带有更强的约束性意味。