
## 前言


上篇文章介绍了使用Semantic Kernel Chat Completion Agent实现的版本。


[使用C\#构建一个论文总结AI Agent](https://github.com)


今天来介绍一下使用Microsoft.Extensions.AI的版本。


## Microsoft.Extensions.AI介绍


Microsoft.Extensions.AI 是微软为 .NET 生态系统推出的一组核心库，旨在为开发者提供统一的 C\# 抽象层，简化与 AI 服务的集成。它通过与 .NET 生态系统的深度协作（包括与 Semantic Kernel 团队的合作），为开发者提供了一种标准化的方式来与各种 AI 服务（如大型语言模型、嵌入生成、工具调用等）进行交互。


GitHub地址：[https://github.com/dotnet/extensions/tree/main/src/Libraries/Microsoft.Extensions.AI](https://github.com):[milou云加速器官网](https://jiechuangmoxing.com)


## 实践


新建一个C\#控制台项目。


安装包：


![image-20250104145553743](https://img2024.cnblogs.com/blog/3288240/202501/3288240-20250104152225232-1002405359.png)


创建插件：



```
internal sealed class PaperAssistantPlugin
{
    public PaperAssistantPlugin()
    {
        var envVars = DotEnv.Read();
        ApiKeyCredential apiKeyCredential = new ApiKeyCredential(envVars["PaperSummaryApiKey"]);

        OpenAIClientOptions openAIClientOptions = new OpenAIClientOptions();
        openAIClientOptions.Endpoint = new Uri($"{envVars["PaperSummaryEndpoint"]}");

        IChatClient openaiClient =
        new OpenAIClient(apiKeyCredential, openAIClientOptions)
            .AsChatClient(envVars["PaperSummaryModelId"]);

        Client = new ChatClientBuilder(openaiClient)
                     .UseFunctionInvocation()
                     .Build();
    }

    internal IChatClient Client { get; set; }

    [Description("读取指定路径的PDF文档内容")]
    [return: Description("PDF文档内容")]
    public string ExtractPDFContent(string filePath)
    {
        Console.WriteLine($"执行函数ExtractPDFContent，参数{filePath}");

        StringBuilder text = new StringBuilder();
        // 读取PDF内容
        using (PdfDocument document = PdfDocument.Open(filePath))
        {
            foreach (var page in document.GetPages())
            {
                text.Append(page.Text);
            }
        }
        return text.ToString();
    }

    [Description("根据文件路径与笔记内容创建一个md格式的文件")]
    public void SaveMDNotes([Description("保存笔记的路径")] string filePath, [Description("笔记的md格式内容")] string mdContent)
    {
        try
        {
            Console.WriteLine($"执行函数SaveMDNotes，参数1：{filePath},参数2：{mdContent}");

            // 检查文件是否存在，如果不存在则创建
            if (!File.Exists(filePath))
            {
                // 创建文件并写入内容
                File.WriteAllText(filePath, mdContent);
            }
            else
            {
                // 如果文件已存在，覆盖写入内容
                File.WriteAllText(filePath, mdContent);
            }
        }
        catch (Exception ex)
        {
            // 处理异常
            Console.WriteLine($"An error occurred: {ex.Message}");
        }
    }

    [Description("总结论文内容生成一个md格式的笔记，并将笔记保存到指定路径")]
    public async void GeneratePaperSummary(string filePath1, string filePath2)
    {
        Console.WriteLine($"执行函数GeneratePaperSummary，参数1：{filePath1},参数2：{filePath2}");

        StringBuilder text = new StringBuilder();
        // 读取PDF内容
        using (PdfDocument document = PdfDocument.Open(filePath1))
        {
            foreach (var page in document.GetPages())
            {
                text.Append(page.Text);
            }
        }

        // 生成md格式的笔记
        string skPrompt = """
                            请使用md格式总结论文的摘要、前言、文献综述、主要论点、研究方法、结果和结论。
                            论文标题为《[论文标题]》，作者为[作者姓名]，发表于[发表年份]。请确保总结包含以下内容：
                            论文摘要
                            论文前言
                            论文文献综诉
                            主要研究问题和背景
                            使用的研究方法和技术
                            主要结果和发现
                            论文的结论和未来研究方向
                            """;
        List history = [];
        history.Add(new ChatMessage(ChatRole.System, skPrompt));
        history.Add(new ChatMessage(ChatRole.User, text.ToString()));

        var result = await Client.CompleteAsync(history);

        try
        {
            // 检查文件是否存在，如果不存在则创建
            if (!File.Exists(filePath2))
            {
                // 创建文件并写入内容
                File.WriteAllText(filePath2, result.ToString());
                Console.WriteLine($"生成笔记成功，笔记路径：{filePath2}");
            }
            else
            {
                // 如果文件已存在，覆盖写入内容
                File.WriteAllText(filePath2, result.ToString());
            }
        }
        catch (Exception ex)
        {
            // 处理异常
            Console.WriteLine($"An error occurred: {ex.Message}");
        }
    }
}

```

创建好了插件之后，我们需要创建一个IChatClient，由于国内大模型提供商大部分都已经兼容了OpenAI格式，所以安装Microsoft.Extensions.AI.OpenAI就可以用了。


在Microsoft.Extensions.AI.OpenAI中使用国内大语言模型的方式如下所示：



```
 var envVars = DotEnv.Read();

 ApiKeyCredential apiKeyCredential = new ApiKeyCredential(envVars["ToolUseApiKey"]);

 OpenAIClientOptions openAIClientOptions = new OpenAIClientOptions();
 openAIClientOptions.Endpoint = new Uri($"{envVars["ToolUseEndpoint"]}");

 IChatClient openaiClient =
 new OpenAIClient(apiKeyCredential, openAIClientOptions)
 .AsChatClient(envVars["ToolUseModelId"]);

 IChatClient client = new ChatClientBuilder(openaiClient)
 .UseFunctionInvocation()
 .Build();

```

在ChatOptions中导入工具：



```
 ChatOptions chatOptions = new()
 {
     Tools = [AIFunctionFactory.Create(paperAssistantPlugin.ExtractPDFContent),
              AIFunctionFactory.Create(paperAssistantPlugin.SaveMDNotes),
              AIFunctionFactory.Create(paperAssistantPlugin.GeneratePaperSummary)]
 };

```

总结论文并将笔记保存至指定路径：


![image-20250104150522404](https://img2024.cnblogs.com/blog/3288240/202501/3288240-20250104152225277-1969989285.png)


![image-20250104150643237](https://img2024.cnblogs.com/blog/3288240/202501/3288240-20250104152225331-433151228.png)


提问论文相关问题：


![image-20250104150933860](https://img2024.cnblogs.com/blog/3288240/202501/3288240-20250104152225273-1687851613.png)


将笔记保存至指定路径：


![image-20250104151253266](https://img2024.cnblogs.com/blog/3288240/202501/3288240-20250104152225321-1502104433.png)


![image-20250104151334522](https://img2024.cnblogs.com/blog/3288240/202501/3288240-20250104152225342-518768520.png)


## 总结


只是一个非常简单的示例，希望对大家使用Microsoft.Extensions.AI实现自己的应用有所帮助。代码已上传至GitHub，地址：[https://github.com/Ming\-jiayou/PaperAssistant。](https://github.com)


