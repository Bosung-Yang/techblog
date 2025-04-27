# A Basics for your custom MCP

Model Context Protocal (MCP) is one of the hottest topic in AI.  
MCP의 목적은 LLM에서 사용할 데이터, 실행 함수, 프롬프트를 규격화하여 여러 AI 서비스를 통합하여 사용할 수 있는 환경을 조성하는 것이다.  
인기있는 주제인 만큼 튜토리얼이 많지만, 대부분의 튜토리얼이 MCP 서버에서 데이터를 가져와 연동하는 프로젝트 등에 목표를 두고 있기 때문에, 자신만의 MCP 서버 혹은 클라이언트를 생성하고자 하는 사람들에게는 바로 적용하기 어렵다.
따라서 본 포스트에서는 mcp의 기본 개념을 설명하고 custom mcp를 구축하고 접속하는 방법을 다루고자 한다.

# 구성
mcp는 기본적으로 서버-클라이언트로 구성되어 있으며, 3가지 컴포넌트로 구성됨.
- 호스트 : mcp 서버에서 데이터를 받아와 사용하려는 llm 어플리케이션
- 클라이언트 : 호스트내부에서 서버와 소통하는 부분
- 서버 : Context, tools, prompt를 클라이언트에 제공하는 서버

# 레이어 구성
프로토콜 레이어와 트랜스포트 레이어가 있지만, 본 글에서는 트랜스포트 레이어를 중심으로 다룸.
트랜스포트 레이어에서 실질적인 커뮤니케이션이 일어남. 
두가지 통신 메커니즘이 있음 : 
- STDIO (Standard IO) : 로컬 환경에서 사용
- HTTP with SSE : 클라이언트가 서버에 요청을 post하면 알맞은 응답을 서버거 해줌

# Resource
LLM이 사용할 데이터로 파일, 데이터베이스, API, 이미지 등의 데이터 타입을 갖고 있다.
리소스는 다음 format의 URI로 구성되어 있다. 
```
[protocol]://[host]/[path]
```
## 리소스 타입
리소스는 텍스트 데이터와 바이너리 데이터를 포함한 다양한 타입을 포함하고 있다.
- Text (UTF-8) : 소스코드, json, CSV 파일 등
- 바이너리 (base64) : 이미지, 피디에프, 비디오 등

## How to access to resource? (Resource Method)
MCP 서버에서 제공하는 리소스의 종류는 `resources/list` 메소드 혹은 엔드포인트에서 받아오면 된다. 
해당 메소드는 다음 형식의 json을 반환하는 것을 기본으로 한다.
```
{
uri : string; // Unique identifier for the resource
name : string; // Human-readable name
description : string;
mimeType : string; 
}
```
`resources/list`에서 리소스 목록을 가져왔으면 `resources/read` 메소드를 통해 읽을 수 있다.
`resources/read`는 다음과 같은 json을 반환한다
```
{
  contents: [
    {
      uri: string;        // The URI of the resource
      mimeType?: string;  // Optional MIME type

      // One of:
      text?: string;      // For text resources
      blob?: string;      // For binary resources (base64 encoded)
    }
  ]
}
```

# 프롬프트
재활용 가능한 프롬프트와 워크플로우를 만들기 용이함. 서버거 mcp를 제공하는 이유는 다음과 같음.
- 다양한 인자를 사용할 수 있음
- context에 리소스를 첨부하여 데이터에 직접적인 대응이 가능
- 워크 플로우에 대한 가이드를 제공할 수 있음
서버에서 제공하는 프롬프트 목록은 `prompt/list`에서 확인할 수 있으며, 검색한 프롬프트는 `prompt/get`을 통해 가져올 수 있음.

# Tools
LLM이 수행할 행동을 지시하는 부분으로 외부 시스템과 상호작용하고 계산을 수행할 수 있음.
`tools/list`에서 실행 가능한 tool 목록을 가져와 `tools/call` 메소드를 통해 실행할 수 있음

--- 
이 포스트에서는 sse를 통해 통신하는 mcp 서버-클라이언트를 구현한다. stdio는 실제 서비스에서는 사용할 확률이 적고 대부분 테스트 단계에서 사용되기 때문임.
# Implementation Example for Servers
클라이언트는 서버로부터 데이터를 받아오기만 하기 때문에 자연스러운 흐름을 위해 서버측 구현부터 살펴본다.

## import libraries
```
import uvicorn
from starlette.applications import Starlette
from starlette.requests import Request
from starlette.routing import Route, Mount

from mcp.server.fastmcp import FastMCP
from mcp.shared.exceptions import McpError
from mcp.types import ErrorData, INTERNAL_ERROR, INVALID_PARAMS
from mcp.server.sse import SseServerTransport

from fastmcp.prompts.base import UserMessage, AssistantMessage
```
주요 라이브러리는 다음과 같음
- Uvicorn : 서버 앱을 호스팅하기 위하여
- Starlette : SSE를 Handle하여 HTTP로 통신을 할 수 있도록 중간다리역활
- MCP and FastMCP : mcp 서버 및 구성요소 구현을 위하여


## 구성 요소 선언
```
mcp = FastMCP('Demo')
# Add an addition tool
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

@mcp.tool()
def read_wikipedia_article(url: str) -> str:
    """
    Fetch a Wikipedia article at the provided URL, parse its main content,
    convert it to Markdown, and return the resulting text.

    Usage:
        read_wikipedia_article("https://en.wikipedia.org/wiki/Python_(programming_language)")
    """
    try:
        if not url.startswith("http"):
            raise ValueError("URL must start with http or https.")

        response = requests.get(url, timeout=10)
        if response.status_code != 200:
            raise McpError(
                ErrorData(
                    INTERNAL_ERROR,
                    f"Failed to retrieve the article. HTTP status code: {response.status_code}"
                )
            )

        soup = BeautifulSoup(response.text, "html.parser")
        content_div = soup.find("div", {"id": "mw-content-text"})
        if not content_div:
            raise McpError(
                ErrorData(
                    INVALID_PARAMS,
                    "Could not find the main content on the provided Wikipedia URL."
                )
            )

        markdown_text = str(content_div)
        return markdown_text

    except ValueError as e:
        raise McpError(ErrorData(INVALID_PARAMS, str(e))) from e
    except RequestException as e:
        raise McpError(ErrorData(INTERNAL_ERROR, f"Request error: {str(e)}")) from e
    except Exception as e:
        raise McpError(ErrorData(INTERNAL_ERROR, f"Unexpected error: {str(e)}")) from e

@mcp.prompt()
def ask_review(code_snippet: str) -> str:
    """Generates a standard code review request."""
    return f"Please review the following code snippet for potential bugs and style issues:\n```python\n{code_snippet}\n```"

@mcp.prompt()
def debug_session_start(error_message: str) -> list[UserMessage]:
    """Initiates a debugging help session."""
    return [
        UserMessage(f"I encountered an error:\n{error_message}"),
        AssistantMessage("Okay, I can help with that. Can you provide the full traceback and tell me what you were trying to do?")
    ]

@mcp.resource("config://app-version")
def get_app_version() -> str:
    """Returns the application version."""
    return "v2.1.0"

@mcp.resource("db://users/{user_id}/email")
async def get_user_email(user_id: str) -> str:
    """Retrieves the email address for a given user ID."""
    # Replace with actual database lookup
    emails = {"123": "alice@example.com", "456": "bob@example.com"}
    return emails.get(user_id, "not_found@example.com")

@mcp.resource("data://product-categories")
def get_categories() -> list[str]:
    """Returns a list of available product categories."""
    return ["Electronics", "Books", "Home Goods"]

# Add a dynamic greeting resource
@mcp.resource("greeting://{name}")
def get_greeting(name: str) -> str:
    """Get a personalized greeting"""
    return f"Hello, {name}!"
```
먼저 FastMCP를 이용해서 서버를 init함.  
그 이후 리소스, 프롬프트, 툴은 mcp 데코레이터를 통해 선언이 가능함.

## Handleing SSE and Deploying Server app
적어도 나에게는 sse를 이용한 통신이 생소했고, mcp 서버에서 데이터를 어떻게 제공해야하는지에 대한 정보를 찾는데 많은 시간을 보냈었음.
한가지 쉽고 유용한 방법은 sse를 라우팅하여 http로 서버-클라이언트간 통신이 가능하게 하는 것임.
```

sse = SseServerTransport("/messages/")

async def handle_sse(request: Request) -> None:
    _server = mcp._mcp_server
    async with sse.connect_sse(
        request.scope,
        request.receive,
        request._send,
    ) as (reader, writer):
        await _server.run(reader, writer, _server.create_initialization_options())

app = Starlette(
    debug=True,
    routes=[
        Route("/sse", endpoint=handle_sse),
        Mount("/messages/", app=sse.handle_post_message),
    ],
)

if __name__ == "__main__":
    uvicorn.run(app, host="localhost", port=8001)
```
MCP에서 제공하는 `SseServerTransport`으로 통신을 할 인터페이스를 생성하고, `handle_sse` 메소드를 통해서 서버 동작을 처리함.
그 이후 starlette를 통해 sse를 routing해서 http에 마운트하여 일반 사용자가 http를 통해 sse메소드를 받아볼 수 있도록 구현한다.
이렇게 생성된 서버는 uvicorn을 통해 배포된다.

# Implementation examples for MCP Client
## import libaries
```
import asyncio
import sys
import traceback
from urllib.parse import urlparse

from mcp import ClientSession
from mcp.client.sse import sse_client
```

## MCP 클라이언트 
```
def print_items(name: str, result: any) -> None:print(f"\nAvailable {name}:")
    items = getattr(result, name)
    if items:
        for item in items:
            print(" *", item)
    else:
        print("No items available")


async def main(server_url: str, article_url: str = None):
    """Connect to the MCP server, list its capabilities, and optionally call a tool.

    Args:
        server_url: Full URL to SSE endpoint (e.g. http://localhost:8000/sse)
        article_url: (Optional) Wikipedia URL to fetch an article
    """
    if urlparse(server_url).scheme not in ("http", "https"):
        print("Error: Server URL must start with http:// or https://")
        sys.exit(1)

    try:
        async with sse_client(server_url) as streams:
            async with ClientSession(streams[0], streams[1]) as session:
                await session.initialize()
                print("Connected to MCP server at", server_url)
                print_items("tools", await session.list_tools())
                print_items("resources", await session.list_resources())
                #print(session.list_resource_templates)
                print_items("resourceTemplates", await session.list_resource_templates())
                print_items("prompts", await session.list_prompts())

                """
                if article_url:
                    print("\nCalling read_wikipedia_article tool...")
                    try:
                        # Use the documented call_tool method to invoke the tool
                        response = await session.call_tool(
                            "read_wikipedia_article", arguments={"url": article_url}
                        )
                        print("\n=== Wikipedia Article Markdown Content ===\n")
                        print(response)
                    except Exception as tool_exc:
                        print("Error calling read_wikipedia_article tool:")
                        traceback.print_exception(
                            type(tool_exc), tool_exc, tool_exc.__traceback__
                        )
                """
    except Exception as e:
        print(f"Error connecting to server: {e}")
        traceback.print_exception(type(e), e, e.__traceback__)
        sys.exit(1)
```
클라이언트는 `sse_client`를 이용하여 stream을 생성하고 `ClientSession`을 생성한다.  
그 다음 사용가능한 리소스, 툴, 프롬프트 목록은 각각 `list_tools, list_resources, list_prompts`로 확인 할 수 있다.


