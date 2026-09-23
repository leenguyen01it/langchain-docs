# Track: Nền tảng kỹ thuật phần mềm cho AI

Phần lớn thời gian của một AI Engineer **không** dành cho prompt hay model, mà cho những việc rất "phần mềm": xây API, xử lý lỗi, gọi song song, lưu dữ liệu, viết test, đóng gói và triển khai. Một prompt xuất sắc nằm trong một hệ thống chập chờn vẫn là một sản phẩm tồi.

Track này tập trung vào những kỹ năng phần mềm **đặc thù cho ứng dụng LLM**: gọi API chậm và tốn tiền, output không xác định trước, streaming từng token, tác vụ chạy lâu.

| Bài | Nội dung | Thời lượng |
|---|---|---|
| [Bài 1: Python hiện đại cho AI](01-python-hien-dai.md) | uv, type hints, Pydantic, cấu hình, async và gọi LLM song song | 1 tuần |
| [Bài 2: FastAPI và streaming](02-fastapi-streaming.md) | API cho ứng dụng LLM, Server-Sent Events, xử lý lỗi, giới hạn tốc độ | 1 tuần |
| [Bài 3: Database, cache và hàng đợi](03-du-lieu-hang-doi.md) | Lưu hội thoại và log chi phí, cache với Redis, tác vụ nền cho việc chạy lâu | 1 tuần |
| [Bài 4: Test, Docker và CI](04-test-docker-ci.md) | Test ứng dụng có LLM, đóng gói Docker, Docker Compose, GitHub Actions | 1 tuần |

**Yêu cầu trước:** biết Python cơ bản (hàm, class, module, list, dict), dùng được terminal và Git.

## Dự án xuyên suốt: `llm-service`

Cả track xây một dịch vụ nhỏ nhưng đầy đủ tính chất production: **API tóm tắt và phân loại phản hồi khách hàng**.

- Bài 1: thư viện Python gọi Claude, trả về kết quả có cấu trúc, xử lý hàng loạt song song.
- Bài 2: bọc thành API FastAPI, có endpoint streaming.
- Bài 3: lưu lịch sử, ghi log chi phí từng lệnh gọi, cache kết quả, xử lý file lớn bằng tác vụ nền.
- Bài 4: test tự động, đóng gói Docker, CI chạy mỗi lần push.

```text
llm-service/
├── pyproject.toml
├── .env
├── app/
│   ├── config.py        # cấu hình (Bài 1)
│   ├── schemas.py       # Pydantic models (Bài 1)
│   ├── llm.py           # gọi Claude (Bài 1)
│   ├── main.py          # FastAPI (Bài 2)
│   ├── db.py            # database (Bài 3)
│   ├── cache.py         # Redis cache (Bài 3)
│   └── worker.py        # tác vụ nền (Bài 3)
├── tests/               # (Bài 4)
├── Dockerfile           # (Bài 4)
└── compose.yaml         # (Bài 4)
```

Sau track này, bạn đã sẵn sàng cho [track Hiểu LLM](../hieu-llm/index.md), nơi đi sâu vào bản chất model và Claude API.

---

[Lộ trình AI Engineer](../index.md)
