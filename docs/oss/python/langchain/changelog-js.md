# Changelog (JavaScript/TypeScript)

> Nhật ký cập nhật và cải tiến cho các package JavaScript/TypeScript của chúng tôi

!!! note "Ghi chú"
    Trang này thuộc mục `oss/javascript` của tài liệu gốc (không phải `oss/python`), được đưa vào đây để mirror đầy đủ trang `changelog-js` trong danh sách 76 trang nguồn của mục `langchain`.

!!! info "Đăng ký nhận tin"
    Changelog này có kèm [RSS feed](https://docs.langchain.com/oss/javascript/releases/changelog/rss.xml), có thể tích hợp với [Slack](https://slack.com/help/articles/218688467-Add-RSS-feeds-to-Slack), [email](https://zapier.com/apps/email/integrations/rss/1441/send-new-rss-feed-entries-via-email), các Discord bot như [Readybot](https://readybot.io/) hoặc [RSS Feeds to Discord Bot](https://rss.app/en/bots/rssfeeds-discord-bot), và các công cụ đăng ký khác.

### Mar 24, 2026 · `deepagents`

## `deepagents` v1.9.0-alpha.0

Bản phát hành alpha của `deepagents` v1.9.0.

* **[Async subagents](https://docs.langchain.com/oss/javascript/deepagents/async-subagents)**: Deep Agents có thể khởi chạy các tác vụ nền không chặn (non-blocking), để người dùng tiếp tục tương tác với agent trong khi subagent làm việc song song. Cần [LangSmith Deployment](https://docs.langchain.com/langsmith/deployment) cho sub-agent.

* **[Backend](https://docs.langchain.com/oss/javascript/deepagents/backends) protocol v2**: Chúng tôi giới thiệu một backend protocol v2 mới (`BackendProtocolV2`) với các thay đổi tương thích ngược cho backend interface của Deep Agents. Các thay đổi chính:
    * **Kiểu kết quả có cấu trúc**: Mọi method giờ trả về các object `Result` có cấu trúc (ví dụ `ReadResult`, `LsResult`, `GrepResult`, `GlobResult`) với xử lý lỗi nhất quán qua field `error`, thay vì trả giá trị thô hoặc throw exception.
    * **Hỗ trợ file đa phương tiện (multi-modal)**: `read()` trả về một `ReadResult` với field `.content` thay vì chuỗi thuần. Với file nhị phân (ảnh, PDF, audio, video), nội dung `Uint8Array` thô đầy đủ được trả về qua `readRaw()`, giúp agent làm việc với file đa phương tiện một cách tự nhiên (native).
    * **Đơn giản hoá tên method**: `lsInfo` → `ls`, `grepRaw` → `grep`, `globInfo` → `glob`.
    * **Tương thích ngược**: Các backend v1 hiện có có thể được chuyển đổi (adapt) sang interface v2 bằng `adaptBackendProtocol`. Các interface v1 (`BackendProtocolV1`, `SandboxBackendProtocolV1`) đã deprecated nhưng vẫn giữ lại để tương thích.

### Jan 14, 2026 · `langgraph`

## v1.1.0

### `@langchain/langgraph`

Giới thiệu **StateSchema**, một cách gọn gàng, không phụ thuộc thư viện cụ thể (library-agnostic) để định nghĩa graph state, hoạt động với bất kỳ thư viện validation nào tuân thủ [Standard Schema](https://github.com/standard-schema/standard-schema).

### Hỗ trợ Standard JSON Schema

LangGraph giờ hỗ trợ [Standard JSON Schema](https://standardschema.dev/json-schema), một đặc tả mở được triển khai bởi Zod 4, Valibot, ArkType, và các thư viện schema khác. Điều này có nghĩa bạn có thể dùng thư viện validation ưa thích mà không bị khoá (lock-in):

```typescript
import { z } from "zod"; // hoặc valibot, arktype, v.v.
import { StateSchema, ReducedValue, MessagesValue } from "@langchain/langgraph";

const AgentState = new StateSchema({
  messages: MessagesValue,
  currentStep: z.string(),
  count: z.number().default(0),
  history: new ReducedValue(
    z.array(z.string()).default(() => []),
    {
      inputSchema: z.string(),
      reducer: (current, next) => [...current, next],
    }
  ),
});

// Kiểu state và update type-safe
type State = typeof AgentState.State;
type Update = typeof AgentState.Update;

const graph = new StateGraph(AgentState)
  .addNode("agent", (state) => ({ count: state.count + 1 }))
  .addEdge(START, "agent")
  .addEdge("agent", END)
  .compile();
```

### Các primitive giá trị state mới

* **ReducedValue**: Định nghĩa field với reducer tuỳ chỉnh để tích luỹ giá trị. Hỗ trợ schema input và output riêng biệt cho reducer input type-safe.
* **UntrackedValue**: Định nghĩa state tạm thời tồn tại trong lúc thực thi nhưng không bao giờ được checkpoint, hữu ích cho kết nối database, cache, hoặc cấu hình chỉ dùng lúc runtime.
* **MessagesValue**: Một `ReducedValue` dựng sẵn cho chat message với reducer message chuẩn.

### Export type helper

Các type utility export mới để định kiểu cho các hàm nằm ngoài graph builder:

* `GraphNode<Schema, Nodes?, Config?>` - Định kiểu cho các hàm node với suy luận (inference) đầy đủ
* `ConditionalEdgeRouter<Schema, Nodes?>` - Định kiểu cho conditional edge router

```typescript
// Định kiểu cho hàm node độc lập
const myNode: GraphNode<typeof AgentState> = (state, config) => {
  return { count: state.count + 1 };
};

// Dùng trực tiếp type helper của schema
const processState = (state: typeof AgentState.State) => {
  console.log(state.count);
};
```

API `Annotation` hiện có và API dựa trên zod vẫn hoạt động không đổi, `StateSchema` là một lựa chọn bổ sung cho những ai thích định nghĩa theo hướng schema-first.

**Tìm hiểu thêm về StateSchema**: xem tài liệu đầy đủ về cách định nghĩa graph state với StateSchema, ReducedValue, và UntrackedValue tại [đây](https://docs.langchain.com/oss/javascript/langgraph/graph-api#schema).

**Tìm hiểu về type utility**: dùng GraphNode và ConditionalEdgeRouter để định kiểu cho các hàm nằm ngoài graph builder, xem [đây](https://docs.langchain.com/oss/javascript/langgraph/graph-api#type-utilities).

### Dec 12, 2025 · nhiều package

## v1.2.0

### `langchain`

* [Structured output](https://docs.langchain.com/oss/javascript/langchain/structured-output): Thêm khả năng đặt thủ công chế độ `strict` khi dùng `providerStrategy` cho structured output.

### `@langchain/openai`

* **Tool có sẵn mới từ provider:** Hỗ trợ file search, web search, code interpreter, tạo ảnh, computer use, shell, và tool MCP connector được thực thi ở phía server bởi provider. Xem [Server-side tool use](https://docs.langchain.com/oss/javascript/langchain/tools#server-side-tool-use) và tích hợp chat [OpenAI](https://docs.langchain.com/oss/javascript/integrations/chat/openai).
* **Kiểm duyệt nội dung:** Tuỳ chọn `moderateContent` mới trên `ChatOpenAI` để phát hiện và xử lý nội dung không an toàn.
* Ưu tiên dùng Responses API cho model GPT-5.2 Pro.

## v1.3.0

### `@langchain/anthropic`

* **Tool có sẵn mới từ provider:** Hỗ trợ text editor, web fetch, computer use, tool search, và tool MCP toolset được thực thi ở phía server bởi provider. Xem [Server-side tool use](https://docs.langchain.com/oss/javascript/langchain/tools#server-side-tool-use) và tích hợp chat [Anthropic](https://docs.langchain.com/oss/javascript/integrations/chat/anthropic).
* Export type `ChatAnthropicInput` để tăng độ an toàn kiểu (type safety).

## v1.1.0

### `@langchain/ollama`

* **Structured output native:** Thêm hỗ trợ structured output native qua `withStructuredOutput`.
* Hỗ trợ cấu hình `baseUrl` tuỳ chỉnh.

## v1.0.0

### `@langchain/community`

* Document loader Jira cập nhật dùng API v3.
* LanceDB: Thêm hỗ trợ `similaritySearch()` và `similaritySearchWithScore()`.
* Hỗ trợ hybrid search cho Elasticsearch.
* `GoogleCalendarDeleteTool` mới.
* Nhiều sửa lỗi cho LlamaCppEmbeddings, PrismaVectorStore, IBM WatsonX, và các cải thiện bảo mật.

### Các package khác

* **@langchain/xai:** Hỗ trợ Native Live Search.
* **@langchain/tavily:** Thêm endpoint research của Tavily.
* **@langchain/mongodb:** LLM cache MongoDB mới.
* **@langchain/mcp-adapters:** Thêm tuỳ chọn `onConnectionError`.
* **@langchain/google-common:** Hỗ trợ method `jsonSchema` trong `withStructuredOutput`.
* **@langchain/core:** Sửa lỗi bảo mật, cải thiện lồng subgraph trong sơ đồ Mermaid, dùng UUID7 cho run ID.

### Nov 25, 2025 · `langchain`

## v1.1.0

* [Model profiles](https://docs.langchain.com/oss/javascript/langchain/models#model-profiles): Chat model giờ expose các tính năng và khả năng được hỗ trợ qua getter `.profile`. Dữ liệu này lấy từ [models.dev](https://models.dev), một dự án mã nguồn mở cung cấp dữ liệu khả năng của model.
* [Model retry middleware](https://docs.langchain.com/oss/javascript/langchain/middleware/built-in#model-retry): Middleware mới để tự động retry các lần gọi model thất bại với exponential backoff có thể cấu hình, cải thiện độ tin cậy của agent.
* [Content moderation middleware](https://docs.langchain.com/oss/javascript/langchain/middleware/built-in#provider-specific-middleware): Middleware kiểm duyệt nội dung của OpenAI để phát hiện và xử lý nội dung không an toàn trong tương tác agent. Hỗ trợ kiểm tra input người dùng, output model, và kết quả tool.
* [Summarization middleware](https://docs.langchain.com/oss/javascript/langchain/middleware/built-in#summarization): Cập nhật để hỗ trợ các điểm kích hoạt linh hoạt, dùng model profile cho việc tóm tắt theo ngữ cảnh.
* [Structured output](https://docs.langchain.com/oss/javascript/langchain/structured-output): Hỗ trợ `ProviderStrategy` (structured output native) giờ có thể được suy luận từ model profile.
* [`SystemMessage` cho `createAgent`](https://docs.langchain.com/oss/javascript/langchain/middleware/custom#dynamic-prompt): Hỗ trợ truyền trực tiếp instance `SystemMessage` vào tham số `systemPrompt` của `createAgent`, cùng method `concat` mới để mở rộng system message. Mở khoá các tính năng nâng cao như cache control và content block có cấu trúc.
* [Dynamic system prompt middleware](https://docs.langchain.com/oss/javascript/langchain/short-term-memory): Giá trị trả về từ `dynamicSystemPromptMiddleware` giờ hoàn toàn mang tính cộng thêm (additive). Khi trả về một [`SystemMessage`](https://reference.langchain.com/javascript/langchain-core/messages/SystemMessage) hoặc `string`, chúng được gộp với system message hiện có thay vì thay thế, giúp dễ kết hợp nhiều middleware cùng chỉnh sửa prompt hơn.
* **Cải thiện tương thích:** Sửa lỗi xử lý lỗi validation của Zod v4 trong structured output và tool schema, đảm bảo thông báo lỗi chi tiết hiển thị đúng.

### Oct 20, 2025 · `langchain`, `langgraph`

## v1.0.0

### `langchain`

* [Release notes](https://docs.langchain.com/oss/javascript/releases/langchain-v1)
* [Hướng dẫn migration](https://docs.langchain.com/oss/javascript/migrate/langchain-v1)

### `langgraph`

* [Release notes](https://docs.langchain.com/oss/javascript/releases/langgraph-v1)
* [Hướng dẫn migration](https://docs.langchain.com/oss/javascript/migrate/langgraph-v1)

!!! info "Phản hồi"
    Nếu bạn gặp vấn đề hoặc có phản hồi, vui lòng [mở một issue](https://github.com/langchain-ai/docs/issues/new?template=01-langchain.yml) để chúng tôi cải thiện. Để xem tài liệu v0.x, [truy cập nội dung lưu trữ](https://github.com/langchain-ai/langchainjs/tree/v0.3/docs/core_docs/docs).

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/javascript/releases/changelog.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
