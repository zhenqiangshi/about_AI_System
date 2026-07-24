MarkItDown 是微软开源的一款强大的 Python 工具，它能够将 PDF、Word、Excel、PPT、图像、音频等多种格式的文件统一转换为 Markdown 格式，非常适合用于 AI 知识库构建、文档管理和数据分析等场景。

以下是使用 MarkItDown 的详细保姆级教程：

### 一、 环境准备与安装

**1. 安装 Python 环境**  
MarkItDown 要求 Python 版本必须 **≥ 3.10**（推荐 3.10 - 3.12 之间）。

- 前往 Python 官网下载并安装，安装时务必勾选 **“Add Python.exe to PATH”**（配置环境变量）。
- 验证安装：在命令行（Windows 的 cmd 或 Mac 的终端）输入 `python --version`，确认版本号正确。

**2. 安装 MarkItDown**

- **基础安装**：在命令行输入 `pip install markitdown`。
- **全功能安装（推荐）**：为了支持所有文件格式（如音频、视频、PDF等），建议安装所有可选依赖：`pip install "markitdown[all]"`。
- **验证安装**：输入 `markitdown --help` 或 `pip show markitdown`，若显示帮助信息或版本号则说明安装成功。

**3. 安装 FFmpeg（视频/音频处理必备）**  
如果你需要转换 `.mp4` 等音视频文件，必须额外安装 FFmpeg。下载后将其 `bin` 目录下的 `ffmpeg.exe` 和 `ffprobe.exe` 复制到 Python 的 `Scripts` 目录中，并在命令行输入 `ffmpeg -version` 验证。

---

### 二、 核心使用方法

MarkItDown 提供了两种主要的使用方式：

#### 方法一：命令行使用（最简单快捷）

适合快速转换单个文件。

- **转换并在终端查看结果**：
    
    ```bash
    markitdown 我的报告.pdf
    ```
    
- **转换并保存到本地文件**（使用 `-o` 参数或 `>` 符号）：
    
    ```bash
    markitdown 我的报告.pdf -o 转换结果.md
    # 或者
    markitdown 我的报告.pdf > 转换结果.md
    ```
    
- **管道输入**（支持链式处理）：
    
    ```bash
    cat 数据.xlsx | markitdown
    ```
    

#### 方法二：Python API 调用（适合进阶与批量处理）

适合需要集成到自动化流程或批量处理文件的场景。

- **基础转换**：
    
    ```python
    from markitdown import MarkItDown
    
    md = MarkItDown()
    result = md.convert("test.xlsx")
    print(result.text_content)  # 输出 Markdown 文本
    ```
    
- **批量转换脚本示例**：
    
    ```python
    import os
    from markitdown import MarkItDown
    
    md = MarkItDown()
    folder = "待转换文件夹"
    for file in os.listdir(folder):
        if file.endswith((".pdf", ".docx", ".pptx")):
            input_path = os.path.join(folder, file)
            output_path = os.path.join(folder, os.path.splitext(file)[0] + ".md")
            result = md.convert(input_path)
            with open(output_path, "w", encoding="utf-8") as f:
                f.write(result.text_content)
            print(f"✅ 已转换：{file}")
    ```
    

---

### 三、 进阶高级用法

**1. 结合大模型（LLM）进行图像描述**  
MarkItDown 支持接入 OpenAI 等大模型，能够识别图片内容并生成文字描述。

```python
from markitdown import MarkItDown
from openai import OpenAI

client = OpenAI(api_key="你的API_KEY")
md = MarkItDown(llm_client=client, llm_model="gpt-4o")
result = md.convert("产品图.jpg")
print(result.text_content)
```

**2. 启用插件系统**  
支持通过插件扩展能力（如 YouTube 视频字幕提取等）。

```bash
# 查看已安装的插件
markitdown --list-plugins
# 启用插件进行转换
markitdown --use-plugins https://www.youtube.com/watch?v=xxx
```

---

### 四、 新手避坑指南

1. **中文乱码问题**：在 Python 代码中保存文件时，务必指定编码为 `encoding="utf-8"`。
2. **文件路径报错**：在命令行中，如果文件路径包含空格或特殊符号，请使用**英文双引号**将完整路径包裹起来。
3. **扫描版 PDF 转换失败**：纯图片构成的 PDF 可能需要接入 Azure Document Intelligence 或安装 `markitdown-ocr` 插件来进行文字识别。
4. **网络下载慢**：国内网络环境下，可以使用清华镜像源加速安装：`pip install markitdown[all] -i https://pypi.tuna.tsinghua.edu.cn/simple`。

