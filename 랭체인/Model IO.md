- 언어 모델 애플리케이션에서 가장 중요한 것은 언어 모델
- 랭체인은 어떠한 언어과도 인터페이스 할 수 있는[^1] 구성 요소를 제공한다
![[Model IO.png]]

[^1]: 두 시스템, 기술 또는 소프트웨어간 상호 작용할 수 있다

### Models
#### LLMs
- 랭체인에서 LLMs는 순수한 텍스트 완성 모델
- 문자열 프롬프트를 입력받아 결과를 문자로 출력한다

#### Chat Models
- LLMs을 활용하고, 대화를 할 수 있도록 조정된다
- 단일 문자열 대신 채팅 메시지 목록을 입력으로 받아 AI 메시지를 출력한다

#### 고려 사항
- LLMs와 Chat Models의 프롬프팅 전략은 상이할 수 있다
	- 모델에 맞는 적절한 프롬프팅 전략을 사용해야 한다
- 다른 모델은 다른 프롬프팅 전략을 사용해야 한다
	- Anthropic 모델은 XML을 사용하는 것이 좋고, OpenAi는 JSON을 사용한는 것이 좋다
### Messages
-  [[Model IO#Chat Models|Chat Model]]은 메시지 리스트를 입력 받아서 하나의 메시지를 반환한다
- Message의 프로퍼티
	-  `role`: 메시지의 생성 주체
	- `content`: 메시지의 내용
		- content는 문자열(String)과 딕셔너리 리스트(A List of dictionaries)가 될 수 있다
	- `additional_kwargs`: 메시지에 대한 추가 정보
#### HumanMessage
- 사용자가 생성하는 메시지
#### AlMessage
- 모델이 생성하는 메시지
- `additional_kwargs` 프로퍼티를 가진다
- OpenAI인 경우 `function_call` 프로퍼티를 가진다
#### SystemMessage
- 시스템이 생성하는 메시지
- 모든 모델이 제공하지는 않는다
- 모델이 어떻게 동작하는지 알려준다
#### FunctionMessage
- 함수 호출에 대한 결과를 나타내는 메시지
- 결과를 제공하는 함수에 대한 이름을 전달하는 `name` 프로퍼티를 제공한다
#### ToolMessage
- 도구 호출에 대한 결과를 나타내는 메시지
- OpenAI의 `function`과 `tool` 메시지 타입을 구분하기
### Prompts
- 때로는 사용자가 작성한 내용이 모델에 곧바로 전달되지 않는다
- 어떠한 방법으로 변형이 되어서 문자나 메시지 리스트로 모델에 전달될 수 있다
- 사용자의 입력을 받아서 문자나 메시지 리스트로 변형하는 것을 `Prompt Templates`라고 한다
#### PromptValue
- ChatModel과 LLMs는 서로 다른 입력 유형을 전달 받는다
- `PromptValue`는 LLMs를 위한 문자열(string)과 ChatModels를 위한 메시지 사이를 변환해준다
#### PromptTemplate
- 프롬프트 템플릿의 예시
- 템플릿 문자열로 이루어져있다
- 사용자의 입력을 포맷팅 되어 최종 문자열을 생성한다
#### MessagePromptTemplate
- 프롬프트 템플릿의 예시
- 특정 역할을 의미하는 템플릿 메시지와 `PromptTemplate`로 구성되어있다
- 사용자의 입력을 포매팅하여 메시지의 `content`가 되는 최종 문자열을 생성한다
-  HumanMessagePromptTemplate, AIMessagePromptTemplate, SystemMessagePromptTemplate가 있다
#### MessagePlaceHolder
- 메시지 리스트가 프롬프트에 대한 입력일 경우 사용한다
- `variable_name`인수로 매개변수화 된다
#### ChatPromptTemplate
- 프롬프트 템플릿의 예시
- [[Model IO#Prompts#MessagePlaceHolder|MessagePlaceHolder]], [[Model IO#Prompts#MessagePromptTemplate|MessagePromptTemplate]]의 리스트로 이루어져있다
- 사용자의 입력을 포매팅하여 최종 메시지 리스트를 생성한다
### Output Parsers
- 모델의 출력은 보통 문자열이나 메시지다
- 아웃풋 파서는 모델의 출력을 받아 사용하기 쉬운 형태로 변환한다
#### StrOutputParser
- 언어 모델의 출력을 문자열로 변환하는 간단한 형태의 파서
- LLMs인 경우 단순 문자열로 반환, ChatModels인 경우 메시지의 `.content`로 반환한다

#### OpenAI Functions Parser
- OpenAI의 함수 호출을 위한 파서

#### Agent OutputParser
- [[에이전트(Agent)]]는 언어 모델을 사용하여 수행해야 할 단계를 결졍하는 시스템
- Agent OutputParser는 모델의 출력을 수행해야 할 작업을 나타낼 수 있는 스키마로 변환한다
