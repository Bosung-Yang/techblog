# A Basics for your custom MCP.md

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



