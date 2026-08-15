# Xây dựng SQL agent

## Tổng quan

Trong tutorial này, bạn sẽ học cách xây dựng một agent có thể trả lời câu hỏi về một database SQL bằng [agent](agents.md) của LangChain.

Ở mức tổng quan, agent sẽ:

1. Lấy danh sách các table và schema có sẵn từ database
2. Quyết định table nào liên quan tới câu hỏi
3. Lấy schema của các table liên quan
4. Sinh một query dựa trên câu hỏi và thông tin từ schema
5. Kiểm tra lại query để tìm lỗi thường gặp bằng LLM
6. Thực thi query và trả về kết quả
7. Sửa lỗi do database engine báo cho tới khi query thành công
8. Xây dựng phản hồi dựa trên kết quả

!!! warning "Cảnh báo"
    Xây dựng hệ thống hỏi đáp (Q&A) cho database SQL đòi hỏi thực thi các câu SQL query do model sinh ra. Có rủi ro cố hữu khi làm điều này. Hãy đảm bảo quyền kết nối database của bạn luôn được giới hạn ở phạm vi hẹp nhất có thể cho nhu cầu của agent. Điều này giúp giảm thiểu, dù không loại bỏ hoàn toàn, rủi ro khi xây dựng hệ thống dựa trên model.

### Khái niệm

Tutorial này bao gồm các khái niệm sau:

* [Tools](tools.md) để đọc từ database SQL
* [Agent](agents.md) của LangChain
* Quy trình [Human-in-the-loop](human-in-the-loop.md)

## Thiết lập

### 1. Cài đặt dependency

```bash
pip install langchain langgraph
```

### 2. Thiết lập LangSmith

Thiết lập [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-sql-agent) để kiểm tra những gì đang xảy ra bên trong chain hoặc agent của bạn. Sau đó thiết lập các biến môi trường sau:

```shell
export LANGSMITH_TRACING="true"
export LANGSMITH_API_KEY="..."
```

## Xây dựng SQL agent của bạn

### 1. Chọn một LLM

Chọn một model hỗ trợ [tool-calling](https://docs.langchain.com/oss/python/integrations/providers/overview):

=== "OpenAI"

    👉 Đọc [tài liệu tích hợp OpenAI chat model](https://docs.langchain.com/oss/python/integrations/chat/openai/)

    ```bash
    pip install -U "langchain[openai]"
    ```

    **`init_chat_model`**

    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["OPENAI_API_KEY"] = "sk-..."

    model = init_chat_model("gpt-5.5")
    ```

    **Model Class**

    ```python
    import os
    from langchain_openai import ChatOpenAI

    os.environ["OPENAI_API_KEY"] = "sk-..."

    model = ChatOpenAI(model="gpt-5.5")
    ```

=== "Anthropic"

    👉 Đọc [tài liệu tích hợp Anthropic chat model](https://docs.langchain.com/oss/python/integrations/chat/anthropic/)

    ```bash
    pip install -U "langchain[anthropic]"
    ```

    **`init_chat_model`**

    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["ANTHROPIC_API_KEY"] = "sk-..."

    model = init_chat_model("claude-sonnet-4-6")
    ```

    **Model Class**

    ```python
    import os
    from langchain_anthropic import ChatAnthropic

    os.environ["ANTHROPIC_API_KEY"] = "sk-..."

    model = ChatAnthropic(model="claude-sonnet-4-6")
    ```

=== "Azure"

    👉 Đọc [tài liệu tích hợp Azure chat model](https://docs.langchain.com/oss/python/integrations/chat/azure_chat_openai/)

    ```bash
    pip install -U "langchain[openai]"
    ```

    **`init_chat_model`**

    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["AZURE_OPENAI_API_KEY"] = "..."
    os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
    os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

    model = init_chat_model(
        "azure_openai:gpt-5.5",
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
    )
    ```

    **Model Class**

    ```python
    import os
    from langchain_openai import AzureChatOpenAI

    os.environ["AZURE_OPENAI_API_KEY"] = "..."
    os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
    os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

    model = AzureChatOpenAI(
        model="gpt-5.5",
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"]
    )
    ```

=== "Google Gemini"

    👉 Đọc [tài liệu tích hợp Google GenAI chat model](https://docs.langchain.com/oss/python/integrations/chat/google_generative_ai/)

    ```bash
    pip install -U "langchain[google-genai]"
    ```

    **`init_chat_model`**

    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["GOOGLE_API_KEY"] = "..."

    model = init_chat_model("google_genai:gemini-2.5-flash-lite")
    ```

    **Model Class**

    ```python
    import os
    from langchain_google_genai import ChatGoogleGenerativeAI

    os.environ["GOOGLE_API_KEY"] = "..."

    model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
    ```

=== "AWS Bedrock"

    👉 Đọc [tài liệu tích hợp AWS Bedrock chat model](https://docs.langchain.com/oss/python/integrations/chat/bedrock/)

    ```bash
    pip install -U "langchain[aws]"
    ```

    **`init_chat_model`**

    ```python
    from langchain.chat_models import init_chat_model

    # Làm theo các bước sau để cấu hình credential:
    # https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html

    model = init_chat_model(
        "us.anthropic.claude-sonnet-4-6",
        model_provider="bedrock_converse",
    )
    ```

    **Model Class**

    ```python
    from langchain_aws import ChatBedrock

    model = ChatBedrock(model="us.anthropic.claude-sonnet-4-6")
    ```

=== "HuggingFace"

    👉 Đọc [tài liệu tích hợp HuggingFace chat model](https://docs.langchain.com/oss/python/integrations/chat/huggingface/)

    ```bash
    pip install -U "langchain[huggingface]"
    ```

    **`init_chat_model`**

    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

    model = init_chat_model(
        "microsoft/Phi-3-mini-4k-instruct",
        model_provider="huggingface",
        temperature=0.7,
        max_tokens=1024,
    )
    ```

    **Model Class**

    ```python
    import os
    from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint

    os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

    llm = HuggingFaceEndpoint(
        repo_id="microsoft/Phi-3-mini-4k-instruct",
        temperature=0.7,
        max_length=1024,
    )
    model = ChatHuggingFace(llm=llm)
    ```

=== "OpenRouter"

    👉 Đọc [tài liệu tích hợp OpenRouter chat model](https://docs.langchain.com/oss/python/integrations/chat/openrouter/)

    ```bash
    pip install -U "langchain-openrouter"
    ```

    **`init_chat_model`**

    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["OPENROUTER_API_KEY"] = "sk-..."

    model = init_chat_model(
        "auto",
        model_provider="openrouter",
    )
    ```

    **Model Class**

    ```python
    import os
    from langchain_openrouter import ChatOpenRouter

    os.environ["OPENROUTER_API_KEY"] = "sk-..."

    model = ChatOpenRouter(model="auto")
    ```

Output hiển thị trong các ví dụ bên dưới dùng OpenAI.

### 2. Cấu hình database

Bạn sẽ tạo một [database SQLite](https://www.sqlitetutorial.net/sqlite-sample-database/) cho tutorial này. SQLite là một database nhẹ, dễ thiết lập và sử dụng. Chúng ta sẽ load database `chinook`, một database mẫu mô phỏng một cửa hàng media kỹ thuật số.

Để tiện lợi, chúng tôi đã host sẵn database (`Chinook.db`) trên một GCS bucket công khai.

```python
import pathlib
import requests

url = "https://storage.googleapis.com/benchmarks-artifacts/chinook/Chinook.db"
local_path = pathlib.Path("Chinook.db")

if local_path.exists():
    print(f"{local_path} already exists, skipping download.")
else:
    response = requests.get(url, timeout=60)
    if response.status_code == 200:
        local_path.write_bytes(response.content)
        print(f"File downloaded and saved as {local_path}")
    else:
        print(f"Failed to download the file. Status code: {response.status_code}")
```

Chúng ta sẽ dùng module `sqlite3` có sẵn của Python để tương tác với database:

```python
import sqlite3

con = sqlite3.connect("Chinook.db")
cursor = con.cursor()

cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
tables = [row[0] for row in cursor.fetchall() if not row[0].startswith("sqlite_")]

print("Dialect: sqlite")
print(f"Available tables: {tables}")

cursor.execute("SELECT * FROM Artist LIMIT 5;")
print(f"Sample output: {cursor.fetchall()}")
con.close()
```

```
Dialect: sqlite
Available tables: ['Album', 'Artist', 'Customer', 'Employee', 'Genre', 'Invoice', 'InvoiceLine', 'MediaType', 'Playlist', 'PlaylistTrack', 'Track']
Sample output: [(1, 'AC/DC'), (2, 'Accept'), (3, 'Aerosmith'), (4, 'Alanis Morissette'), (5, 'Alice In Chains')]
```

### 3. Thêm tool để tương tác với database

!!! warning "Cảnh báo"
    Các database tool dưới đây chỉ là wrapper tối giản cho mục đích minh hoạ. Chúng không nhằm để bảo mật hoặc dùng trong production. Hãy dùng quyền database phạm vi hẹp và thêm validation riêng cho ứng dụng trước khi thực thi SQL do model sinh ra.

Chúng ta có thể triển khai [tool](tools.md) database dưới dạng wrapper mỏng dùng decorator `@tool` từ `langchain.tools`:

```python
import sqlite3
from langchain.tools import tool

# Dưới đây là các tool tối giản cho mục đích minh hoạ.
# Chúng không nhằm để bảo mật hoặc dùng trong production.

@tool
def sql_db_list_tables() -> str:
    """Input is an empty string, output is a comma-separated list of tables in the database."""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
        tables = [row[0] for row in cursor.fetchall() if not row[0].startswith("sqlite_")]
        return ", ".join(tables)
    finally:
        con.close()

@tool
def sql_db_schema(table_names: str) -> str:
    """Input to this tool is a comma-separated list of tables, output is the schema and sample rows for those tables.
    Be sure that the tables actually exist by calling sql_db_list_tables first!
    Example Input: table1, table2, table3"""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
        valid_tables = {row[0] for row in cursor.fetchall() if not row[0].startswith("sqlite_")}
        results = []
        for table in table_names.split(","):
            table = table.strip()
            if table not in valid_tables:
                results.append(f"Error: table_names {{{table!r}}} not found in database")
                continue
            cursor.execute("SELECT sql FROM sqlite_master WHERE type='table' AND name=?;", (table,))
            schema_row = cursor.fetchone()
            if schema_row:
                results.append(schema_row[0])
                try:
                    quoted_table = '"' + table.replace('"', '""') + '"'
                    cursor.execute(f"SELECT * FROM {quoted_table} LIMIT 3;")
                    rows = cursor.fetchall()
                    if rows:
                        col_names = [description[0] for description in cursor.description]
                        results.append(
                            f"/*\n3 rows from {table} table:\n"
                            + "\t".join(col_names)
                            + "\n"
                            + "\n".join("\t".join(str(x) for x in row) for row in rows)
                            + "\n*/"
                        )
                except Exception as e:
                    results.append(f"Error fetching sample rows: {e}")
        return "\n\n".join(results)
    finally:
        con.close()

@tool
def sql_db_query(query: str) -> str:
    """Input to this tool is a detailed and correct SQL query, output is a result from the database.
    If the query is not correct, an error message will be returned.
    If an error is returned, rewrite the query, check the query, and try again.
    If you encounter an issue with Unknown column 'xxxx' in 'field list', use sql_db_schema to query the correct table fields."""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute(query)
        res = cursor.fetchall()
        return str(res)
    except Exception as e:
        return f"Error: {e}"
    finally:
        con.close()

@tool
def sql_db_query_checker(query: str) -> str:
    """Use this tool to double check if your query is correct before executing it.
    Always use this tool before executing a query with sql_db_query!"""
    trigger_prompt = """{query}
Double check the sqlite query above for common mistakes, including:
- Using NOT IN with NULL values
- Using UNION when UNION ALL should have been used
- Using BETWEEN for exclusive ranges
- Data type mismatch in predicates
- Properly quoting identifiers
- Using the correct number of arguments for functions
- Casting to the correct data type
- Using the proper columns for joins

If there are any of the above mistakes, rewrite the query. If there are no mistakes, just reproduce the original query.

Output the final SQL query only.

SQL Query: """.format(query=query)

    response = model.invoke(trigger_prompt)
    return response.text.strip()

tools = [sql_db_list_tables, sql_db_schema, sql_db_query, sql_db_query_checker]

# Dùng tên biến vòng lặp khác để không che khuất decorator `tool`.
for t in tools:
    print(f"{t.name}: {t.description}\n")
```

```
sql_db_query: Input to this tool is a detailed and correct SQL query, output is a result from the database.
    If the query is not correct, an error message will be returned.
    If an error is returned, rewrite the query, check the query, and try again.
    If you encounter an issue with Unknown column 'xxxx' in 'field list', use sql_db_schema to query the correct table fields.

sql_db_schema: Input to this tool is a comma-separated list of tables, output is the schema and sample rows for those tables.
    Be sure that the tables actually exist by calling sql_db_list_tables first!
    Example Input: table1, table2, table3

sql_db_list_tables: Input is an empty string, output is a comma-separated list of tables in the database.

sql_db_query_checker: Use this tool to double check if your query is correct before executing it.
    Always use this tool before executing a query with sql_db_query!
```

### 4. Tạo agent

Dùng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) để xây dựng một [ReAct agent](https://arxiv.org/pdf/2210.03629) với lượng code tối thiểu. Agent sẽ diễn giải request và sinh một lệnh SQL, sau đó tool sẽ thực thi lệnh đó. Nếu lệnh có lỗi, message lỗi được trả về cho model. Model sau đó có thể xem xét request gốc cùng message lỗi mới và sinh một lệnh mới. Việc này có thể tiếp tục cho tới khi LLM sinh lệnh thành công hoặc đạt số lần thử tối đa. Pattern cung cấp phản hồi cho model (message lỗi trong trường hợp này) rất mạnh mẽ.

Khởi tạo agent với một system prompt mô tả rõ để tuỳ chỉnh hành vi:

```python
system_prompt = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

You MUST double check your query before executing it. If you get an error while
executing a query, rewrite the query and try again.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the
database.

To start you should ALWAYS look at the tables in the database to see what you
can query. Do NOT skip this step.

Then you should query the schema of the most relevant tables.
""".format(
    dialect="sqlite",
    top_k=5,
)
```

Bây giờ, tạo một agent với model, tool, và prompt:

```python
from langchain.agents import create_agent


agent = create_agent(
    model,
    tools,
    system_prompt=system_prompt,
)
```

### 5. Chạy agent

Chạy agent với một câu hỏi mẫu và quan sát hành vi của nó:

```python
question = "Which genre on average has the longest tracks?"

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": question}]},
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
        for delta in item.output_deltas:
            print(delta, end="", flush=True)
        print(f"\nTool result: {item.output}")

final_state = stream.output
```

```
================================ Human Message =================================

Which genre on average has the longest tracks?
================================== Ai Message ==================================
Tool Calls:
  sql_db_list_tables (call_BQsWg8P65apHc8BTJ1NPDvnM)
 Call ID: call_BQsWg8P65apHc8BTJ1NPDvnM
  Args:
================================= Tool Message =================================
Name: sql_db_list_tables

Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track
================================== Ai Message ==================================
Tool Calls:
  sql_db_schema (call_i89tjKECFSeERbuACYm4w0cU)
 Call ID: call_i89tjKECFSeERbuACYm4w0cU
  Args:
    table_names: Track, Genre
================================= Tool Message =================================
Name: sql_db_schema


CREATE TABLE "Genre" (
	"GenreId" INTEGER NOT NULL,
	"Name" NVARCHAR(120),
	PRIMARY KEY ("GenreId")
)

/*
3 rows from Genre table:
GenreId	Name
1	Rock
2	Jazz
3	Metal
*/


CREATE TABLE "Track" (
	"TrackId" INTEGER NOT NULL,
	"Name" NVARCHAR(200) NOT NULL,
	"AlbumId" INTEGER,
	"MediaTypeId" INTEGER NOT NULL,
	"GenreId" INTEGER,
	"Composer" NVARCHAR(220),
	"Milliseconds" INTEGER NOT NULL,
	"Bytes" INTEGER,
	"UnitPrice" NUMERIC(10, 2) NOT NULL,
	PRIMARY KEY ("TrackId"),
	FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"),
	FOREIGN KEY("GenreId") REFERENCES "Genre" ("GenreId"),
	FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)

/*
3 rows from Track table:
TrackId	Name	AlbumId	MediaTypeId	GenreId	Composer	Milliseconds	Bytes	UnitPrice
1	For Those About To Rock (We Salute You)	1	1	1	Angus Young, Malcolm Young, Brian Johnson	343719	11170334	0.99
2	Balls to the Wall	2	2	1	U. Dirkschneider, W. Hoffmann, H. Frank, P. Baltes, S. Kaufmann, G. Hoffmann	342562	5510424	0.99
3	Fast As a Shark	3	2	1	F. Baltes, S. Kaufman, U. Dirkscneider & W. Hoffman	230619	3990994	0.99
*/
================================== Ai Message ==================================
Tool Calls:
  sql_db_query_checker (call_G64yYm6R6UauiVPCXJZMA49b)
 Call ID: call_G64yYm6R6UauiVPCXJZMA49b
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AverageLength FROM Track INNER JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AverageLength DESC LIMIT 5;
================================= Tool Message =================================
Name: sql_db_query_checker

SELECT Genre.Name, AVG(Track.Milliseconds) AS AverageLength FROM Track INNER JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AverageLength DESC LIMIT 5;
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_AnO3SrhD0ODJBxh6dHMwvHwZ)
 Call ID: call_AnO3SrhD0ODJBxh6dHMwvHwZ
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AverageLength FROM Track INNER JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AverageLength DESC LIMIT 5;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385), ('Science Fiction', 2625549.076923077), ('Drama', 2575283.78125), ('TV Shows', 2145041.0215053763), ('Comedy', 1585263.705882353)]
================================== Ai Message ==================================

On average, the genre with the longest tracks is "Sci Fi & Fantasy" with an average track length of approximately 2,911,783 milliseconds. This is followed by "Science Fiction," "Drama," "TV Shows," and "Comedy."
```

Agent đã viết đúng một query, kiểm tra lại query, và chạy nó để đưa ra phản hồi cuối cùng.

!!! note "Ghi chú"
    Bạn có thể kiểm tra mọi khía cạnh của lần chạy trên, bao gồm các bước đã thực hiện, tool nào được gọi, prompt nào LLM đã thấy, và nhiều hơn nữa trong [LangSmith trace](https://smith.langchain.com/public/cd2ce887-388a-4bb1-a29d-48208ce50d15/r).

### 6. (Tuỳ chọn) Dùng Studio

[Studio](https://docs.langchain.com/langsmith/studio) cung cấp một vòng lặp "phía client" cùng với memory nên bạn có thể chạy dưới dạng giao diện chat và truy vấn database. Bạn có thể hỏi những câu như "Tell me the scheme of the database" hoặc "Show me the invoices for the 5 top customers". Bạn sẽ thấy lệnh SQL được sinh ra và output kết quả. Chi tiết cách bắt đầu ở bên dưới.

**Chạy agent của bạn trong Studio**

Ngoài các package đã đề cập ở trên, bạn cần:

```shell
pip install -U langgraph-cli[inmem]>=0.4.0
```

Trong thư mục bạn sẽ chạy, bạn cần một file `langgraph.json` với nội dung sau:

```json
{
  "dependencies": ["."],
  "graphs": {
      "agent": "./sql_agent.py:agent",
      "graph": "./sql_agent_langgraph.py:graph"
  },
  "env": ".env"
}
```

Tạo một file `sql_agent.py` và thêm nội dung sau:

```python
# sql_agent.py for studio
import pathlib
import sqlite3

import requests
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.tools import tool

# Khởi tạo một LLM
model = init_chat_model("gpt-5.5")

# Lấy database, lưu về local
url = "https://storage.googleapis.com/benchmarks-artifacts/chinook/Chinook.db"
local_path = pathlib.Path("Chinook.db")

if local_path.exists():
    print(f"{local_path} already exists, skipping download.")
else:
    response = requests.get(url, timeout=60)
    if response.status_code == 200:
        local_path.write_bytes(response.content)
        print(f"File downloaded and saved as {local_path}")
    else:
        print(f"Failed to download the file. Status code: {response.status_code}")

# Dưới đây là các tool tối giản cho mục đích minh hoạ.

@tool
def sql_db_list_tables() -> str:
    """Input is an empty string, output is a comma-separated list of tables in the database."""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
        tables = [row[0] for row in cursor.fetchall() if not row[0].startswith("sqlite_")]
        return ", ".join(tables)
    finally:
        con.close()

@tool
def sql_db_schema(table_names: str) -> str:
    """Input to this tool is a comma-separated list of tables, output is the schema and sample rows for those tables.
    Be sure that the tables actually exist by calling sql_db_list_tables first!
    Example Input: table1, table2, table3"""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
        valid_tables = {row[0] for row in cursor.fetchall() if not row[0].startswith("sqlite_")}
        results = []
        for table in table_names.split(","):
            table = table.strip()
            if table not in valid_tables:
                results.append(f"Error: table_names {{{table!r}}} not found in database")
                continue
            cursor.execute("SELECT sql FROM sqlite_master WHERE type='table' AND name=?;", (table,))
            schema_row = cursor.fetchone()
            if schema_row:
                results.append(schema_row[0])
                try:
                    quoted_table = '"' + table.replace('"', '""') + '"'
                    cursor.execute(f"SELECT * FROM {quoted_table} LIMIT 3;")
                    rows = cursor.fetchall()
                    if rows:
                        col_names = [description[0] for description in cursor.description]
                        results.append(
                            f"/*\n3 rows from {table} table:\n"
                            + "\t".join(col_names)
                            + "\n"
                            + "\n".join("\t".join(str(x) for x in row) for row in rows)
                            + "\n*/"
                        )
                except Exception as e:
                    results.append(f"Error fetching sample rows: {e}")
        return "\n\n".join(results)
    finally:
        con.close()

@tool
def sql_db_query(query: str) -> str:
    """Input to this tool is a detailed and correct SQL query, output is a result from the database.
    If the query is not correct, an error message will be returned.
    If an error is returned, rewrite the query, check the query, and try again.
    If you encounter an issue with Unknown column 'xxxx' in 'field list', use sql_db_schema to query the correct table fields."""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute(query)
        res = cursor.fetchall()
        return str(res)
    except Exception as e:
        return f"Error: {e}"
    finally:
        con.close()

@tool
def sql_db_query_checker(query: str) -> str:
    """Use this tool to double check if your query is correct before executing it.
    Always use this tool before executing a query with sql_db_query!"""
    trigger_prompt = """{query}
Double check the sqlite query above for common mistakes, including:
- Using NOT IN with NULL values
- Using UNION when UNION ALL should have been used
- Using BETWEEN for exclusive ranges
- Data type mismatch in predicates
- Properly quoting identifiers
- Using the correct number of arguments for functions
- Casting to the correct data type
- Using the proper columns for joins

If there are any of the above mistakes, rewrite the query. If there are no mistakes, just reproduce the original query.

Output the final SQL query only.

SQL Query: """.format(query=query)

    response = model.invoke(trigger_prompt)
    return response.text.strip()

tools = [sql_db_list_tables, sql_db_schema, sql_db_query, sql_db_query_checker]

# Dùng tên biến vòng lặp khác để không che khuất decorator `tool`.
for t in tools:
    print(f"{t.name}: {t.description}\n")

# Dùng create_agent
system_prompt = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

You MUST double check your query before executing it. If you get an error while
executing a query, rewrite the query and try again.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the
database.

To start you should ALWAYS look at the tables in the database to see what you
can query. Do NOT skip this step.

Then you should query the schema of the most relevant tables.
""".format(
    dialect="sqlite",
    top_k=5,
)

agent = create_agent(
    model,
    tools,
    system_prompt=system_prompt,
)
```

### 7. Triển khai human-in-the-loop review

Sẽ là thận trọng nếu kiểm tra các SQL query của agent trước khi chúng được thực thi, để tránh hành động ngoài ý muốn hoặc kém hiệu quả.

Agent LangChain hỗ trợ sẵn [middleware human-in-the-loop](human-in-the-loop.md) để thêm bước giám sát vào tool call của agent. Hãy cấu hình agent để dừng lại chờ human review khi gọi tool `sql_db_query`:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware # [!code highlight]
from langgraph.checkpoint.memory import InMemorySaver # [!code highlight]


agent = create_agent(
    model,
    tools,
    system_prompt=system_prompt,
    middleware=[ # [!code highlight]
        HumanInTheLoopMiddleware( # [!code highlight]
            interrupt_on={"sql_db_query": True}, # [!code highlight]
            description_prefix="Tool execution pending approval", # [!code highlight]
        ), # [!code highlight]
    ], # [!code highlight]
    checkpointer=InMemorySaver(), # [!code highlight]
)
```

!!! note "Ghi chú"
    Chúng ta đã thêm một [checkpointer](short-term-memory.md) vào agent để cho phép việc thực thi được tạm dừng và tiếp tục. Xem [hướng dẫn human-in-the-loop](human-in-the-loop.md) để biết chi tiết về điều này cũng như các cấu hình middleware khả dụng.

Khi chạy agent, nó sẽ tạm dừng để chờ review trước khi thực thi tool `sql_db_query`:

```python
question = "Which genre on average has the longest tracks?"
config = {"configurable": {"thread_id": "1"}} # [!code highlight]

stream = agent.stream_events( # [!code highlight]
    {"messages": [{"role": "user", "content": question}]},
    config, # [!code highlight]
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
if stream.interrupted: # [!code highlight]
    print("INTERRUPTED:") # [!code highlight]
    interrupt = stream.interrupts[0] # [!code highlight]
    for request in interrupt.value["action_requests"]: # [!code highlight]
        print(request["description"]) # [!code highlight]
```

```
...

INTERRUPTED:
Tool execution pending approval

Tool: sql_db_query
Args: {'query': 'SELECT g.Name AS Genre, AVG(t.Milliseconds) AS AvgTrackLength FROM Track t JOIN Genre g ON t.GenreId = g.GenreId GROUP BY g.Name ORDER BY AvgTrackLength DESC LIMIT 1;'}
```

Chúng ta có thể tiếp tục thực thi, trong trường hợp này là chấp nhận query, bằng [Command](https://docs.langchain.com/oss/python/langgraph/use-graph-api#combine-control-flow-and-state-updates-with-command):

```python
from langgraph.types import Command # [!code highlight]

stream = agent.stream_events( # [!code highlight]
    Command(resume={"decisions": [{"type": "approve"}]}), # [!code highlight]
    config,
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
if stream.interrupted:
    print("INTERRUPTED:")
    interrupt = stream.interrupts[0]
    for request in interrupt.value["action_requests"]:
        print(request["description"])
```

```
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_7oz86Epg7lYRqi9rQHbZPS1U)
 Call ID: call_7oz86Epg7lYRqi9rQHbZPS1U
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgDuration FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AvgDuration DESC LIMIT 5;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385), ('Science Fiction', 2625549.076923077), ('Drama', 2575283.78125), ('TV Shows', 2145041.0215053763), ('Comedy', 1585263.705882353)]
================================== Ai Message ==================================

The genre with the longest average track length is "Sci Fi & Fantasy" with an average duration of about 2,911,783 milliseconds, followed by "Science Fiction" and "Drama."
```

Xem [hướng dẫn human-in-the-loop](human-in-the-loop.md) để biết chi tiết.

## Bước tiếp theo

Để tuỳ biến sâu hơn, xem [tutorial này](https://docs.langchain.com/oss/python/langgraph/sql-agent) về việc triển khai SQL agent trực tiếp bằng các primitive của LangGraph.
