- `LangChain` runnables와 chains를 REAT API로 배포할 수 있도록 지원한다
- [FastAPI](https://fastapi.tiangolo.com/)와 통합되어 있고, 데이터 유효성 검사를 위한 [pydantic](https://docs.pydantic.dev/latest/)을 사용한다

### 특징
- 입출력 스키마는 랭체인 객체로부터 추론되고 모든 API 호출에 대해 적용되며, 풍부한 에러 메세지와 함께 제공된다
- JSONSchema, Swagger를 통한 API 문서 페이지
- `/invoke/`, `/batch/`, `/stream/` 엔드 포인트: 단일 서버에서 많은 동시 요청을 지원
- `/stream_log/` 엔드 포인트: 체인과 에이전트의 모든(혹은 일부) 중간 단계를 스트리밍
- `/playground/` 엔드 포인트: Playground 페이지 제공
- [[랭스미스(LangSmith)]]에 내장된 추적(tracing) 기능

### 한계
- 서버에서 발생하는 이벤트에 대한 클라이언트 콜백 미지원
- Pydantic V2를 사용할 경우 OpenAPI 문서가 생성되지 않음
