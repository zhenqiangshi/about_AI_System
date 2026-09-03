[[PaddleOCR]]
[[Docling & opendataloader-pdf]]
## 一、Opendataloder-PDF

**✅ OpenDataLoader PDF 使用 Hybrid 模式的完整命令**

### 1. 先安装 Hybrid 版本

```bash
pip install -U "opendataloader-pdf[hybrid]"
```

### 2. 启动 Hybrid 服务（必须先开）

```bash
# 启动 Hybrid 后端服务（推荐新开一个终端）
opendataloader-pdf-hybrid --port 5002
```

**可选参数**：
- `--force-ocr`：强制使用 OCR（适合扫描件/图片PDF）
- `--enrich-formula`：增强公式识别
- `--enrich-table`：增强表格识别

### 3. 使用 Hybrid 模式处理 PDF（主命令）

```bash
# 基础 Hybrid 命令（推荐）
opendataloader-pdf --hybrid docling-fast your_file.pdf --output ./output

# 更完整推荐命令（带 Hybrid 配置）
opendataloader-pdf \
  --hybrid docling-fast \
  --hybrid-url http://localhost:5002 \
  --output ./output \
  your_file.pdf
```

### 常用参数说明

| 参数                  | 作用                          | 推荐用法                     |
|-----------------------|-------------------------------|------------------------------|
| `--hybrid`            | 启用 Hybrid 模式              | `docling-fast`（推荐）       |
| `--hybrid-url`        | 指定后端服务地址              | `http://localhost:5002`      |
| `--output` / `-o`     | 输出文件夹                    | `./output`                   |
| `--force-ocr`         | 强制 OCR                      | 扫描件时使用                 |
| `--enrich-formula`    | 增强公式识别                  | 学术论文时使用               |
| `--lang`              | 语言                          | `zh` / `en` 等               |

### 处理文件夹示例

```bash
opendataloader-pdf 
--hybrid docling-fast ./pdf_folder/ 
--output ./output
```

**输出内容**：
- Markdown 文件（.md）
- JSON 结构化数据（带坐标）
- 可视化结果

---

**使用流程建议**：
1. 先运行 `opendataloader-pdf-hybrid --port 5002`
2. 再运行上面的 `opendataloader-pdf --hybrid ...` 命令


## 二、OCRmypdf

**✅ `ocrmypdf` 是一个功能非常强大的命令行工具，它的核心逻辑是：**读取输入的 PDF（哪怕是纯图片的扫描件），运行 OCR（光学字符识别）引擎识别出文字，然后将这些文字精准地“缝合”回原 PDF 的图层下方，生成一个可以自由复制、搜索的全新 PDF。**

下面为你详细拆解 `ocrmypdf` 的常用命令及其参数的含义。
https://ocrmypdf.readthedocs.io/en/latest/index.html
### 1. 最基础的命令（单语言识别）

Bash

```bash
ocrmypdf 
-l chi_sim 
--deskew 
--clean test.pdf 
./test-6-2.pdf 
--output-type pdf
```

**参数解释：**

- **`-l chi_sim`**：指定 OCR 识别的语言（Language）。`chi_sim` 代表**简体中文**（Simplified Chinese）。如果要识别英文，可以使用 `-l eng`。
    
- **`../test.pdf`**：输入的原始 PDF 文件路径（你的扫描件）。
    
- **`../output_searchable.pdf`**：输出的全新 PDF 文件路径（转换后支持搜索和复制的文件）。
    

### 2. 进阶常用参数（解决实际痛点）

在处理技术规格书、带表格的扫描件时，通常需要结合以下参数来提升识别质量和体验：

#### 🌐 多语言混合识别：`-l chi_sim+eng`

技术规格书里通常既有中文（如“产品技术规格书”），又有英文和型号（如 `M0546-LF`、`CE`）。你可以用 `+` 号把多种语言连起来：

Bash

```bash
ocrmypdf -l chi_sim+eng ../test.pdf ../output.pdf
```

> **作用：** 同时启用中文和英文 OCR 引擎，大幅提升型号、代码、英文单词的识别准确率。

#### 📐 自动纠正倾斜：`--rotate-pages`

有时候扫描件是用手机拍的，或者扫描仪放歪了，页面是斜的或者倒过来的。

Bash

```bash
ocrmypdf -l chi_sim+eng --rotate-pages ../test.pdf ../output.pdf
```

> **作用：** 自动检测页面的文字方向，并在生成新 PDF 时把页面**旋转回正**，非常智能。

#### 🧼 图像预处理（去噪、去黑边）：`--deskew` 和 `--clean`

扫描件经常带有复印留下的黑边、歪斜的表格线、或者订书钉的阴影。

Bash

```bash
ocrmypdf -l chi_sim+eng --deskew --clean ../test.pdf ../output.pdf
```

> **作用：** > * `--deskew`：微调文字的倾斜度（让每一行字变水平）。
> 
> - `--clean`：在 OCR 识别前对图片进行降噪、擦除边缘黑边，让输出的文档更干净。
>     

#### 🚀 强力覆盖模式：`--redo-ocr`

如果你的 PDF 原本已经被人用很烂的软件做过一次 OCR，里面有一堆乱码，你想用 Tesseract 重新识别：

Bash

```bash
ocrmypdf -l chi_sim+eng --redo-ocr ../test.pdf ../output.pdf
```

> **作用：** 强行擦除 PDF 原有的错误文本层，彻底重新运行 OCR。

### 🛠️ 推荐给你的“全能通杀”命令

针对你手头那份包含中英文、产品型号、可能存在排版倾斜的技术规格书扫描件，推荐直接使用下面这个组合命令：

Bash

```bash
ocrmypdf -l chi_sim+eng --rotate-pages --deskew --clean ../test.pdf ../searchable_ok.pdf
```

### ⚠️ 运行前的必备前提（Ubuntu/WSL 环境）

在使用 `-l chi_sim` 之前，你的 WSL 系统必须**安装了 Tesseract** 引擎及其对应的中文语言包，否则会报错提示找不到语言文件。安装命令如下：

Bash

```bash
sudo apt update
# 安装 ocrmypdf 以及 Tesseract 的简体中文、繁体中文和英文包
sudo apt install ocrmypdf tesseract-ocr-chi-sim tesseract-ocr-chi-tra tesseract-ocr-eng
```



---

### 第一步：libredwg

```bash
sudo apt update
sudo apt install -y git build-essential autoconf libtool
git clone https://git.savannah.gnu.org/git/libredwg.git
cd libredwg
sh ./autogen.sh
./configure
make
sudo make install
```

---
🚀 `dwgread` 核心高级用法指南

既然你想用它，这里分享几个 `dwgread` 最强力的分析命令，它能把复杂的二进制 DWG 图纸直接转换成我们能看得懂的文本结构：

1. 将 DWG 转换为纯文本 JSON 格式（强推，最适合提取数据）

这是 `dwgread` 最具价值的功能。它能把图纸里所有的图层、线段、圆、文字全部解构成一个标准的 JSON 文件，你可以用 Python、Java 或任何语言轻松解析这个 JSON。

Bash

```bash
dwgread -O JSON -o house_plan.json input.dwg
```

2. 打印图纸的简要统计信息（快速了解图纸）

如果你只想看这张图纸包含多少个实体（文本、线、块），可以用 `-v`（Verbose 模式，可选 1-9 的严重级别）：

Bash

```bash
dwgread -v3 input.dwg
```

它会在屏幕上打印出该 DWG 文件的版本（如 AC1027 代表 AutoCAD 2013）、对象总数以及是否有损坏。

3. 转换为 DXF 格式（等同于 dwg2dxf）

虽然有专门的 `dwg2dxf` 工具，但 `dwgread` 本身也支持直接输出 DXF：

Bash

```bash
dwgread -O DXF -o output.dxf input.dwg
```
### 第二步：安装 Python 依赖

```bash
pip3 install ezdxf pandas openpyxl
```

---

### 第三步：完整脚本（推荐最终版）

创建一个文件 `dwg_reader.py`：

```bash
nano dwg_reader.py
```

**粘贴以下完整代码**：

```python
import os
import sys
from datetime import datetime
import ezdxf
from ezdxf.addons import odafc   # 推荐方式，自动调用 ODA

def read_dwg_file(dwg_path, output_dir="./output"):
    """完整读取 DWG 文件（自动转换 + 提取信息）"""
    
    if not os.path.exists(dwg_path):
        print("❌ 文件不存在！")
        return
    
    os.makedirs(output_dir, exist_ok=True)
    print(f"[{datetime.now().strftime('%H:%M:%S')}] 开始处理: {dwg_path}")
    
    try:
        # ========== 1. 直接读取 DWG（ezdxf + ODA 自动转换）==========
        print("正在读取 DWG 文件（自动转 DXF）...")
        doc = odafc.readfile(dwg_path, version="ACAD2018")   # 支持 R2010 ~ R2025
        
        msp = doc.modelspace()
        
        # ========== 2. 基本信息 ==========
        print("\n" + "="*70)
        print("📊 DWG 文件信息总结")
        print("="*70)
        print(f"DWG 版本          : {doc.dxfversion}")
        print(f"图层数量          : {len(doc.layers)}")
        print(f"实体总数量        : {len(msp)}")
        print(f"处理时间          : {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        
        # ========== 3. 图层列表 ==========
        print("\n📋 图层列表（前20个）:")
        for i, layer in enumerate(list(doc.layers)[:20]):
            print(f"  {i+1:2d}. {layer.dxf.name:<25} 颜色: {layer.dxf.color}")
        if len(doc.layers) > 20:
            print(f"  ... 共 {len(doc.layers)} 个图层")
        
        # ========== 4. 文字提取 ==========
        print("\n🔤 提取文字内容（前30条）:")
        texts = []
        for entity in msp.query("TEXT MTEXT"):
            try:
                text = entity.dxf.text.strip() if hasattr(entity.dxf, 'text') else str(entity.text).strip()
                if text:
                    texts.append((text, entity.dxf.layer))
            except:
                continue
        
        for i, (txt, layer) in enumerate(texts[:30]):
            preview = txt[:75] + "..." if len(txt) > 75 else txt
            print(f"  {i+1:2d}. [{layer}] {preview}")
        
        if len(texts) > 30:
            print(f"\n   ... 共找到 {len(texts)} 条文字")
        
        # ========== 5. 保存 DXF（可选）==========
        dxf_path = os.path.join(output_dir, os.path.basename(dwg_path).replace(".dwg", ".dxf"))
        doc.saveas(dxf_path)
        print(f"\n✅ DXF 文件已保存: {dxf_path}")
        
        print("\n🎉 处理完成！")
        
    except Exception as e:
        print(f"❌ 处理失败: {e}")
        print("提示：请确保 ODA File Converter 已正确安装")


# ====================== 使用示例 ======================
if __name__ == "__main__":
    if len(sys.argv) > 1:
        dwg_file = sys.argv[1]
    else:
        dwg_file = "你的文件.dwg"          # ←←← 修改这里
    
    read_dwg_file(dwg_file, "./dwg_output")
```

---

### 运行方式

```bash
# 单文件运行
python3 dwg_reader.py "test.dwg"

# 或直接修改脚本里的文件名后运行
python3 dwg_reader.py
```

---

**这个方案优点**：
- 自动处理 DWG → DXF
- 代码简洁稳定
- 可轻松扩展（块属性、表格、标注、坐标等）












