

[GITHUB](https://github.com/datalab-to/marker#marker)
**现有OCRmypdf进行一次扫描件转换+Marker进行二次OCR识别**
[[OCR]]

Marker converts documents to markdown, JSON, chunks, and HTML quickly and accurately.

- Converts PDF, image, PPTX, DOCX, XLSX, HTML, EPUB files in all languages
- Formats tables, forms, equations, inline math, links, references, and code blocks
- Extracts and saves images
- Removes headers/footers/other artifacts
- Extensible with your own formatting and logic
- Does structured extraction, given a JSON schema (beta)
- Optionally boost accuracy with LLMs (and your own prompt)
- Works on GPU, CPU, or MPS


```shell
(mkd) shizhenqiang@ubuntu-wsl:~/mkd/marker_test/test-6-2$ marker_single _page_9_Figure_7.jpeg --output_dir ./output_pic
Recognizing Layout: 100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.10s/it]
Running OCR Error Detection: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:00<00:00,  4.68it/s]
Detecting bboxes: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.07s/it]
Recognizing Text: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 5/5 [00:27<00:00,  5.60s/it]
Detecting bboxes: 0it [00:00, ?it/s]
2026-06-09 14:33:32,641 [INFO] marker: Saved markdown to ./output_pic/_page_9_Figure_7
2026-06-09 14:33:32,641 [INFO] marker: Total time: 32.499377727508545
(mkd) shizhenqiang@ubuntu-wsl:~/mkd/marker_test/test-6-2$ ls
_page_0_Picture_1.jpeg  _page_2_Picture_1.jpeg  _page_5_Picture_1.jpeg  _page_8_Picture_1.jpeg  _page_9_Picture_4.jpeg  test-6-2_meta.json
_page_0_Picture_2.jpeg  _page_3_Picture_1.jpeg  _page_6_Picture_1.jpeg  _page_9_Figure_7.jpeg   output_pic
_page_1_Picture_1.jpeg  _page_4_Picture_1.jpeg  _page_7_Picture_1.jpeg  _page_9_Picture_1.jpeg  test-6-2.md
```