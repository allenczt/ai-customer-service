## GraphRAG 上线/运行说明（面向客户）
## 适用场景：维护标准问答库（QA）+ 文档知识库（RAG/GraphRAG）+ 语音查询 + Memory 管理
## 建议运行环境：Windows PowerShell（命令前缀使用 .\rag_env\Scripts\python.exe）


## 如果只考虑上线，可以把前期的向量化在本地完成，只上传并定期更新向量知识库、QA数据库；向量模型、语音模型建议部署线上服务器。AI用API调用。

## =========================
## 模块 0：环境准备（首次）
## =========================
## 1) 激活虚拟环境（Windows PowerShell）
.\rag_env\Scripts\Activate.ps1

## 2) 安装依赖（如已安装可跳过）
## PDF 解析（中文 PDF 友好）+ OCR（扫描件 PDF）
# pip install pdfplumber pymupdf pdf2image easyocr
## 可选：复杂版式增强（表格/多栏）
# pip install unstructured[local-inference] layoutparser
## 其它核心依赖
# pip install torch torchvision torchaudio fastapi uvicorn sentence-transformers modelscope numpy

## 3) 下载本地模型（如已下载可跳过）
## Embedding 模型（用于检索）
# modelscope download --model BAAI/bge-large-zh-v1.5 --local_dir ./models/bge-large-zh-v1.5
## LLM 模型（用于生成回答/社区报告）
# modelscope download --model Qwen/Qwen3-1.7B --local_dir ./models/Qwen3-1.7B


## =========================
## 模块 1：标准问答库（QA）维护
## =========================
## 目标文件：qa_dataset.jsonl
## 输入支持：Word(.docx) / Excel(.xlsx/.xls)
## 行为说明：
## - 相同 question：保留“最后一次导入”的问答对（实现答案更新）
## - 支持删除指定 question

## 1.1 导入/更新 QA（Word / Excel）
python qa_extract.py -i "你的文件.docx" -o qa_dataset.jsonl
python qa_extract.py -i "新QA文件.xlsx" -o qa_dataset.jsonl

## 示例
python qa_extract.py -i "QA/吴忠市交通运输行业标准化智能问答库.docx" -o qa_dataset.jsonl
python qa_extract.py -i "QA_new/QA-0209整理.xlsx" -o qa_dataset.jsonl
python qa_extract.py -i "QA_new/客服工作QA-整理.xlsx" -o qa_dataset.jsonl

## 1.2 删除某个问题（按 question 精确匹配）
python qa_extract.py -o qa_dataset.jsonl -d "这里填要删除的完整问题文本"
## 1.3向量化保存，以后直接调用向量结果，不用每次都再向量化一次
.\rag_env\Scripts\python.exe qa_vectorize.py --qa .\qa_dataset.jsonl --out .\qa_cache\qa_cache.npz --embedding-model .\models\bge-large-zh-v1.5

## =========================
## 模块 2：文档知识库入库（RAG）
## =========================
## 目标文件：knowledge.db / knowledge_new.db
## 输入目录：documents / documents_new
## 输出目录：extracted_text / extracted_text_new（保存提取文本便于人工校对）
##
## 入库流程说明：
## - 支持 PDF/Word/文本
## - 文本型 PDF 优先直接解析；扫描件 PDF 会走 OCR
## - 解析后切分 chunks，生成 embedding，写入 SQLite

## 2.1 文档入库（首次/全量）
.\rag_env\Scripts\python.exe document_processor.py -i ".\documents" -o ".\knowledge.db" --text-output-dir ".\extracted_text"

## 2.2 新文档入库（增量库，推荐先写入 knowledge_new.db 再合并）
.\rag_env\Scripts\python.exe document_processor.py -i ".\documents_new" -o ".\knowledge_new.db" --text-output-dir ".\extracted_text_new"


## =========================
## 模块 3：DB 管理（合并/删除）
## =========================
## 场景：把 knowledge_new.db 的增量内容合并进 knowledge.db，或清理不需要的文档

## 3.1 安全合并（默认：自动跳过重复 chunk_id）
python db_manager.py merge --source knowledge_new.db

## 3.2 强制覆盖重复项（谨慎使用：会用新库内容覆盖旧库）
python db_manager.py merge --source knowledge_new.db --force-update

## 3.3 删除文档（先预览，确认后删除）
python db_manager.py delete --doc-name "bad_doc.pdf"
python db_manager.py delete --doc-name "bad_doc.pdf" --confirm

## 3.4 按业务域/权限批量清理（谨慎使用）
python db_manager.py delete --domain "通用监管" --confirm
python db_manager.py delete --permission "internal" --confirm


## =========================
## 模块 4：GraphRAG 构建与查询
## =========================
## 前置条件：knowledge_new.db（或 knowledge.db）已完成入库
## 产物：实体/社区/社区报告，用于更强的多跳推理与总结能力

## 4.1 构建 GraphRAG（顺序执行）
.\rag_env\Scripts\python.exe graph_builder.py --db .\knowledge_new.db
python community_detector.py
python report_generator.py --db ./knowledge_new.db --batch 5

## 4.2 社群可视化（可选）
python graphplot.py --db ./knowledge_new.db

## 4.3 查询
# ## 方式1：仅 GraphRAG（流水线 query）
# python run_graphrag_pipeline.py 危险品运输要注意什么
## 方式2：融合查询（标准 QA 优先，未命中再走 GraphRAG）
python combine_rag.py "危化品运输需要哪些资质？" --db-path ".\knowledge_new.db" --user-id "u001"

python combine_rag.py "电子证照显示已过期，要怎么处理？" --db-path ".\knowledge_new.db" --user-id "u001"

## 4.4 语音查询（先 ASR → 再查询）
## 下载语音识别模型（如已下载可跳过）
#modelscope download --model iic/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-pytorch --local_dir ./models/speech_paraformer
## 语音识别 + 查询
##需要安装语音基础依赖库，后期可以根据实际接口安装或者调用
python vioce_query.py --audio ".\test.wav" --db-path ".\knowledge_new.db" --user-id "u001" --log-dir ".\qa_memory"
python vioce_query.py "请问电子证照显示已过期，要怎么处理？？" --db-path ".\knowledge_new.db" --user-id "u001"
## 若新增/更新大量法律法规文档：建议重新执行 4.1~4.3 以更新图谱与社区报告


## =========================
## 模块 5：Memory 管理（持续积累）
## =========================
## 用途：把高频且正确的问答整理成高质量 QA，持续补充到 qa_dataset.jsonl

.\rag_env\Scripts\python.exe memory_manager.py --min-count 2

##如果用户表述不清，可以考虑澄清问答
python clarify_question.py


## =========================
## 模块 6：其它扩展能力
## =========================
## 6.1 搜索爬取补充知识（可选）
python web_search_crawler.py

## 6.2 OCR 效果优化（针对识别效果差的文件）
## 方案1：导出 OCR/解析结果 → 人工校对 → 再向量化
python extract_text_only.py
## 输出：./extracted_text/xxx.json（可用 VS Code/记事本修改 pages[].text）
python vectorize_text.py
## 方案2：使用 LLM OCR（如 GLM-OCR）
modelscope download --model ZhipuAI/GLM-OCR --local_dir ./models/GLM-OCR
python extract_text_with_llmocr.py

## 备注：后期如需更精细的分块/阈值等，可再调整参数与策略。
