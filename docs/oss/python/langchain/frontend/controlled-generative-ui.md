# Controlled generative UI

> Render output của agent bằng các component do bạn tự viết: dùng component như tool, render tool-call, render state, và render reasoning

## Tổng quan

Controlled generative UI nằm ở đầu "do tác giả kiểm soát" (author-controlled) trong [phổ generative UI](generative-ui-overview.md). Bạn viết component, còn agent quyết định render component nào và truyền dữ liệu gì vào đó. Agent không bao giờ tự tạo ra markup: nó chỉ chọn từ một tập giao diện cố định mà bạn đã xây dựng và kiểm thử sẵn.

Pattern này cho độ dự đoán được (predictability) cao nhất trong số các cách tiếp cận generative UI. Vì mọi component đều xuất phát từ codebase của bạn, bạn kiểm soát chính xác branding, layout, khả năng tiếp cận (accessibility), và hành vi, đồng thời có thể đảm bảo bất cứ thứ gì agent hiển thị đều đã qua review của bạn. Đánh đổi là chi phí kỹ thuật: mỗi khả năng mới cần một component bạn viết sẵn từ trước. Thư viện component của bạn chính là ranh giới: agent chỉ có thể render những gì bạn đã cung cấp.

## Khi nào nên dùng cách tiếp cận này

Hãy dùng controlled generative UI cho những bề mặt (surface) có traffic cao, quan trọng với thương hiệu, nơi tập kết quả đầu ra đã biết trước và độ chính xác quan trọng hơn sự mới lạ: form, luồng xác nhận (confirmation flow), và bất kỳ bề mặt nào có yêu cầu nghiêm ngặt về branding hoặc accessibility. Khi bạn cần agent tự soạn (compose) các layout mà bạn không lường trước được, cho phần đuôi dài (long tail) của các tương tác phụ, hãy tiến thêm một bước trên phổ này sang [declarative generative UI](declarative-generative-ui.md).

Controlled generative UI bao gồm bốn kỹ thuật, bắt đầu từ việc agent chọn nguyên một giao diện cho tới việc frontend phản ứng với nội bộ (internals) của agent.

## Component như tool

Bạn expose các UI component cho agent theo cách bạn expose tool. Mỗi component có một tên, một mô tả, và một tập thuộc tính có kiểu (typed properties), agent chọn một component và cung cấp dữ liệu của nó như một phần trong response. Frontend ánh xạ (map) lựa chọn của agent sang implementation thật của bạn. Cách này giữ công việc của agent nhỏ gọn: chỉ quyết định giao diện nào đã được duyệt trước phù hợp với tình huống, còn code của bạn sở hữu toàn bộ phần giao diện trông ra sao và hành xử thế nào.

CopilotKit gọi pattern này là [components as tools](https://docs.copilotkit.ai/generative-ui/tool-based).

## Render tool-call

Khi agent gọi một tool, lượt gọi đó đi qua một vòng đời (lifecycle): đang chờ (pending), rồi hoàn tất (complete) hoặc thất bại (failed). Render tool-call biến mỗi giai đoạn thành UI được thiết kế riêng, ví dụ một card loading trong lúc tìm kiếm đang chạy, một card kết quả khi nó trả về, và một trạng thái lỗi nếu thất bại, thay vì hiện JSON thô. Điều này giúp hành động của agent dễ hiểu và tạo niềm tin cho user về những gì đang diễn ra.

Xem pattern [Tool calling](tool-calling.md).

## Render state

Agent expose state bền vững, có kiểu (durable, typed state), vượt ra ngoài danh sách message: todo, output pipeline, trích dẫn (citation), file sandbox, metric, và các object nghiệp vụ tuỳ chỉnh. Render state gắn (bind) component của bạn với state đó, nhờ vậy UI trở thành một view sống (live view) của công việc agent đang làm, thay vì chỉ là một bản transcript. Khi agent cập nhật state, giao diện cũng cập nhật theo.

Xem [typed agent state](overview.md) trong phần tổng quan frontend.
Để ánh xạ một payload response có kiểu duy nhất sang UI tuỳ chỉnh, xem [Structured output](structured-output.md).

## Reasoning

Các model có extended thinking tạo ra reasoning tách biệt với câu trả lời cuối cùng. Việc render reasoning cho user thấy agent đã đi đến kết quả bằng cách nào, giúp xây dựng lòng tin, hỗ trợ debug, và phục vụ việc kiểm toán (audit). Bạn kiểm soát cách thức và thời điểm reasoning xuất hiện, ví dụ trong một khối có thể thu gọn (collapsible), tách biệt với response.

Xem pattern [Reasoning tokens](reasoning-tokens.md).

## Xem thêm

* [Tổng quan Generative UI](generative-ui-overview.md)
* [Declarative generative UI](declarative-generative-ui.md)
* [Open-ended generative UI](open-ended-generative-ui.md)
