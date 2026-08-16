# LangChain Docs (Tiếng Việt)

Dự án tài liệu MkDocs Material chứa bản dịch tiếng Việt của hai mục `oss/python/langchain` và `oss/python/langgraph` từ [docs.langchain.com](https://docs.langchain.com/oss/python/langchain/overview).

## Chạy thử local

```bash
pip install -r requirements.txt
mkdocs serve
```

Sau đó mở `http://127.0.0.1:8000`.

## Build tĩnh

```bash
mkdocs build
```

Output nằm ở thư mục `site/`.

## Cấu trúc

Nội dung nằm trong `docs/oss/python/langchain/` và `docs/oss/python/langgraph/`, mirror đúng cấu trúc URL của bản gốc (ví dụ `docs/oss/python/langchain/frontend/integrations/ai-elements.md` tương ứng `/oss/python/langchain/frontend/integrations/ai-elements` trên bản gốc).
