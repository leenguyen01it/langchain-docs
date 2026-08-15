# Open-ended generative UI

> Render UI được tạo ra bên ngoài ứng dụng của bạn, chẳng hạn như MCP Apps chạy trong sandbox, ở đầu mở nhất của phổ generative UI

## Tổng quan

Open-ended generative UI nằm ở đầu "do agent tạo ra" của [phổ generative UI](generative-ui-overview.md). Giao diện được tạo ra bên ngoài ứng dụng của bạn, ví dụ bởi một MCP server, và frontend của bạn render nó bên trong một sandbox. Không phải bạn cũng không phải agent viết ra các component: một bên thứ ba cung cấp chúng, và app của bạn chỉ đóng vai trò host.

Cách tiếp cận này mang lại phạm vi biểu đạt rộng nhất: agent sở hữu canvas. Một khả năng (capability) có thể xuất hiện cùng với giao diện đã được xây dựng sẵn của riêng nó, nên bạn có thể hiển thị các công cụ tương tác mà bạn chưa từng tự triển khai, mà không cần viết code frontend ở phía bạn. Cách này phù hợp cho các visualization một lần và các câu trả lời tuỳ biến, nơi một kết quả bất ngờ và đủ tốt còn giá trị hơn một kết quả có thể đoán trước. Đây cũng là cách tiếp cận thử nghiệm nhất, ít xác định (deterministic) nhất, chậm hơn và tốn kém hơn khi chạy, và UI không đáng tin cậy là loại khó đảm bảo tính nhất quán, khả năng tiếp cận (accessibility), và an toàn nhất, nên nó phải được cách ly khỏi phần còn lại của ứng dụng bạn.

## Khi nào nên dùng cách tiếp cận này

Dùng open-ended generative UI khi bạn muốn hiển thị các khả năng và giao diện tồn tại bên ngoài ứng dụng của bạn và tiến hoá độc lập với nó, chẳng hạn các công cụ được publish bởi một hệ sinh thái MCP server. Khi bạn cần đảm bảo về branding, khả năng tiếp cận, hoặc layout, hãy quay lại dọc theo phổ này về hướng [declarative](declarative-generative-ui.md) hoặc [controlled](controlled-generative-ui.md) generative UI, nơi ứng dụng của bạn sở hữu các component.

## MCP Apps

[Model Context Protocol](https://modelcontextprotocol.io) cho phép agent kết nối tới các server bên ngoài cung cấp tool và resource. MCP Apps mở rộng ý tưởng đó sang cả giao diện: một MCP server cung cấp UI tương tác, và frontend render nó, thường trong một iframe, trực tiếp trong cuộc hội thoại. Server sở hữu component, dữ liệu, và tương tác, còn ứng dụng của bạn chỉ cung cấp khung (frame) và kết nối tới agent.

CopilotKit tài liệu hoá pattern này với tên [MCP Apps](https://docs.copilotkit.ai/generative-ui/mcp-apps).

## Sandbox và an toàn

Vì giao diện đến từ bên thứ ba, hãy coi nó là không đáng tin cậy. Render nó trong một ngữ cảnh cách ly, chẳng hạn một iframe sandbox, và giới hạn những gì nó có thể truy cập để một app hoạt động sai hoặc độc hại không thể chạm tới phần còn lại của trang hoặc dữ liệu người dùng của bạn. Sandbox là thứ khiến đầu mở của phổ này khả dụng trong production: nó giới hạn (contain) phạm vi biểu đạt thay vì thu hẹp nó.

## Xem thêm

* [Tổng quan Generative UI](generative-ui-overview.md)
* [Controlled generative UI](controlled-generative-ui.md)
* [Declarative generative UI](declarative-generative-ui.md)
