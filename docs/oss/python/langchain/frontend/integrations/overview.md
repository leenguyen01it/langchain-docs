# Tổng quan

> Kết nối `useStream` với bất kỳ thư viện component UI React hoặc framework generative UI nào

[`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) không phụ thuộc vào UI (UI-agnostic). Nó trả về state reactive thuần tuý gồm messages, tool calls, cờ loading, values, và thread metadata mà bạn kết nối (wire) với bất kỳ lớp hiển thị nào bạn chọn. Các trang này minh hoạ cách các thư viện khác nhau tích hợp với frontend của LangChain, mỗi thư viện có một triết lý riêng khi xây dựng AI chat và generative UI.

## Tích hợp

**CopilotKit** ([xem chi tiết](copilotkit.md))
Runtime chat AI đầy đủ với hỗ trợ generative UI có cấu trúc. Thêm một endpoint CopilotKit tuỳ chỉnh vào LangGraph deployment của bạn, sau đó render cây component động trong React.

**AI Elements** ([xem chi tiết](ai-elements.md))
Các component dựa trên shadcn/ui có thể kết hợp (composable) cho AI chat. Chỉ cần thả `Conversation`, `Message`, `Tool`, và `Reasoning` vào và kết nối trực tiếp với `stream.messages`.

**assistant-ui** ([xem chi tiết](assistant-ui.md))
Framework React headless với lớp runtime đầy đủ. Cầu nối `useStream` với `AssistantRuntimeProvider` qua adapter `useExternalStoreRuntime`.

**OpenUI** ([xem chi tiết](openui.md))
Thư viện generative UI cho phép agent tạo ra các dashboard tương tác, hoàn chỉnh, bằng một DSL component khai báo (declarative). Được thiết kế riêng cho các UI dạng report, giàu dữ liệu.

## Chọn thư viện nào

Mỗi thư viện phù hợp với một mô hình tích hợp hơi khác nhau. Lựa chọn phụ thuộc vào loại UI bạn đang xây dựng:

| | CopilotKit | AI Elements | assistant-ui | OpenUI |
| --- | --- | --- | --- | --- |
| **Phù hợp nhất cho** | Runtime chat đầy đủ cộng với generative UI có cấu trúc | Chat với các loại message phong phú | Chat đầy đủ tính năng với thiết lập tối thiểu | Dashboard và report được tạo tự động |
| **Phong cách UI** | Chat shell của CopilotKit cộng với renderer message tuỳ chỉnh | Component shadcn/ui có thể kết hợp | Slot headless cộng với theme mặc định | Thư viện component dựng sẵn với DSL khai báo |
| **Tuỳ chỉnh** | Endpoint backend tuỳ chỉnh, agent context, và renderer | Chỉnh sửa trực tiếp file mã nguồn | Ghi đè slot component | Theme qua CSS custom properties |
| **Trải nghiệm streaming** | Chat stream do runtime quản lý với payload assistant có cấu trúc | Render tăng dần (progressive) ở cấp component | Quản lý thread có sẵn | Hoisting: shell hiện ngay lập tức, dữ liệu điền vào sau |
| **Tool call** | Qua runtime của CopilotKit và renderer tuỳ chỉnh | `Tool` / `ToolHeader` / `ToolOutput` | Tuỳ chỉnh qua slot message | Inline ngay trong UI được tạo ra |
| **Định dạng agent** | Phản hồi assistant có cấu trúc cộng với Markdown tuỳ chọn | Bất kỳ `stream.messages` nào | Bất kỳ `stream.messages` nào | Agent xuất text theo openui-lang |

Cả bốn thư viện đều hoạt động tốt với LangChain agent, và ba thư viện sau còn kết nối trực tiếp với [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream). CopilotKit đặc biệt hữu ích khi bạn muốn một lớp runtime phong phú hơn và một endpoint riêng có thể chạy song song với một deployment [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview).
