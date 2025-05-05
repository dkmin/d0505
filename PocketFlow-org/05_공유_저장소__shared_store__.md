# Chapter 5: 공유 저장소 (Shared Store)

이전 챕터인 [플로우 (Flow)](04_플로우__flow__.md)에서는 [노드 (Node)](02_노드__node__.md)들을 [액션 (Action)](03_액션__action__.md)으로 연결하여 하나의 작업 흐름([그래프 (Graph)](01_그래프__graph__.md))을 만들고, 이를 [플로우 (Flow)](04_플로우__flow__.md) 객체를 통해 실행하는 방법을 배웠습니다.

그런데 [플로우 (Flow)](04_플로우__flow__.md) 내의 여러 [노드 (Node)](02_노드__node__.md)들이 서로 어떻게 데이터를 주고받을 수 있을까요? 예를 들어, '사용자 입력받기' [노드 (Node)](02_노드__node__.md)에서 받은 텍스트를 'LLM 호출하기' [노드 (Node)](02_노드__node__.md)로 전달하고, 'LLM 호출하기' [노드 (Node)](02_노드__node__.md)의 응답을 '응답 후처리하기' [노드 (Node)](02_노드__node__.md)로 전달해야 합니다. 각 [노드 (Node)](02_노드__node__.md)는 독립적인 작업을 수행하지만, 전체적인 작업 흐름을 위해서는 데이터 공유가 필수적입니다.

이럴 때 필요한 것이 바로 PocketFlow의 **공유 저장소 (Shared Store)** 입니다.

## 공유 저장소 (Shared Store)란 무엇일까요?

**공유 저장소 (Shared Store)** 는 이름 그대로 [플로우 (Flow)](04_플로우__flow__.md) 내의 **모든 [노드 (Node)](02_노드__node__.md)가 함께 사용하고 접근할 수 있는 데이터 저장 공간**입니다. 마치 모든 작업자가 동일한 정보를 보고 필요한 정보를 기록할 수 있는 **공유 게시판**과 같습니다.

PocketFlow에서 공유 저장소는 파이썬의 **딕셔너리(dictionary)** 형태로 구현됩니다. 키(key)와 값(value) 쌍으로 데이터를 저장하며, [플로우 (Flow)](04_플로우__flow__.md)가 시작될 때 이 딕셔너리가 생성되어 [플로우 (Flow)](04_플로우__flow__.md) 내의 모든 [노드 (Node)](02_노드__node__.md)에 전달됩니다.

이 공유 저장소를 사용하면:

*   한 [노드 (Node)](02_노드__node__.md)에서 생성한 데이터를 다른 [노드 (Node)](02_노드__node__.md)에서 쉽게 읽어 사용할 수 있습니다.
*   [플로우 (Flow)](04_플로우__flow__.md)의 현재 상태나 중간 결과를 저장하고 여러 [노드 (Node)](02_노드__node__.md)에서 참조할 수 있습니다.
*   복잡한 데이터 파이프라인에서 데이터 전달을 명확하고 유연하게 관리할 수 있습니다.

## 공유 저장소 사용하기: `shared` 매개변수

PocketFlow의 모든 [노드 (Node)](02_노드__node__.md) 클래스(`Node` 및 이를 상속하는 클래스)의 핵심 메서드인 `prep`, `exec_fallback`, `post`는 첫 번째 인자로 **`shared`** 라는 이름의 매개변수를 받습니다. 이 `shared`가 바로 현재 [플로우 (Flow)](04_플로우__flow__.md) 실행에 사용되는 공유 저장소 딕셔너리입니다.

```python
from pocketflow import Node

class MyNode(Node):
    # prep 메서드는 공유 저장소(shared)를 받습니다.
    def prep(self, shared):
        # shared 딕셔너리에서 데이터를 읽어옵니다.
        data_from_previous_node = shared.get("some_key") 
        print(f"[MyNode] prep: shared에서 {data_from_previous_node} 읽음")
        # exec 메서드로 전달할 값을 반환합니다. (공유 저장소와는 별개)
        return data_from_previous_node

    # exec_fallback 메서드도 공유 저장소(shared)를 받습니다.
    def exec_fallback(self, prep_res, exc):
         print(f"[MyNode] exec_fallback: 에러 발생 시 shared 데이터 참조 가능")
         # shared["error_count"] = shared.get("error_count", 0) + 1 # 예시

    # post 메서드는 공유 저장소(shared)를 받습니다.
    # exec 결과(exec_res)를 받지만, 공유 저장소와는 다른 매개변수입니다.
    def post(self, shared, prep_res, exec_res):
        # exec 결과를 shared 딕셔너리에 저장합니다.
        shared["result_from_my_node"] = exec_res
        print(f"[MyNode] post: shared에 {exec_res} 저장")
        # 다음 노드로 이동할 액션을 반환합니다.
        return "next_step" 

```

위 코드에서 볼 수 있듯이, `prep`, `exec_fallback`, `post` 메서드 내에서는 `shared`라는 이름의 딕셔너리에 자유롭게 접근하여 데이터를 읽거나 쓸 수 있습니다.

[플로우 (Flow)](04_플로우__flow__.md)를 실행할 때, `.run()` 메서드의 인자로 초기 공유 저장소 딕셔너리를 전달합니다.

```python
from pocketflow import Flow
# ... (MyNode 등 노드 정의 생략)

my_node = MyNode()
# ... (노드 연결 생략)

# Flow 객체 생성 (시작 노드 지정)
my_flow = Flow(start=my_node)

# Flow 실행 시 공유 저장소 딕셔너리 전달
initial_shared_data = {"some_key": "Hello from Flow!"}
print("--- 플로우 실행 시작 ---")
# run 메서드에 shared 딕셔너리를 전달합니다.
my_flow.run(shared=initial_shared_data) 
print("--- 플로우 실행 종료 ---")
# 플로우 실행 후 shared 딕셔너리의 최종 상태 확인
print(f"플로우 실행 후 공유 저장소: {initial_shared_data}") 

```

이 코드를 실행하면, `MyNode`의 `prep` 메서드는 `initial_shared_data` 딕셔너리에서 `"some_key"` 값을 읽게 되고, `post` 메서드는 `exec_res` 값을 `"result_from_my_node"` 키로 다시 `initial_shared_data` 딕셔너리에 저장하게 됩니다. `.run()` 메서드 호출 후 `initial_shared_data` 딕셔너리를 출력해보면 플로우 실행 중에 노드들이 저장한 데이터가 남아있는 것을 확인할 수 있습니다.

## 실제 예시: 단어 카운터 플로우

`cookbook/pocketflow-communication` 예시는 공유 저장소를 활용하여 여러 노드가 상태를 공유하는 좋은 예시입니다. 이 예시는 사용자가 입력한 텍스트의 단어 수를 세고, 누적 통계를 공유 저장소에 저장하여 보여주는 간단한 단어 카운터입니다.

여기서 핵심적인 역할을 하는 세 개의 [노드 (Node)](02_노드__node__.md)를 살펴보고, 이들이 공유 저장소를 어떻게 사용하는지 확인해 봅시다.

```python
# cookbook/pocketflow-communication/nodes.py 파일 일부 발췌 및 한국어 주석 추가

from pocketflow import Node

# 사용자 입력을 받는 노드
class TextInput(Node):
    def prep(self, shared):
        """사용자 입력을 받습니다."""
        # 공유 저장소(shared)는 prep의 인자로 전달됩니다.
        return input("텍스트를 입력하세요 ('q' 입력 시 종료): ") 
    
    def post(self, shared, prep_res, exec_res):
        """텍스트를 shared에 저장하고 통계를 초기화/갱신합니다."""
        if prep_res == 'q':
            return "exit" # 'q' 입력 시 'exit' 액션 반환 (플로우 종료 목적)
        
        # shared 딕셔너리에 'text' 키로 입력 텍스트를 저장합니다.
        shared["text"] = prep_res
        
        # shared 딕셔너리에 'stats' 키가 없으면 초기화합니다.
        if "stats" not in shared:
            shared["stats"] = {
                "total_texts": 0, # 총 입력된 텍스트 수
                "total_words": 0  # 총 단어 수
            }
        # shared 딕셔너리의 통계 값을 갱신합니다.
        shared["stats"]["total_texts"] += 1
        
        # 'count' 액션을 반환하여 다음 노드(WordCounter)로 이동
        return "count" 

```

`TextInput` 노드는 `prep`에서 사용자 입력을 받고, `post`에서 그 결과를 `shared["text"]`에 저장합니다. 또한, `shared["stats"]` 딕셔너리를 초기화하거나 `total_texts` 값을 갱신합니다.

```python
# cookbook/pocketflow-communication/nodes.py 파일 일부 발췌 및 한국어 주석 추가

from pocketflow import Node
# ... (TextInput 노드 생략)

# 단어 수를 세는 노드
class WordCounter(Node):
    def prep(self, shared):
        """shared에서 텍스트를 가져옵니다."""
        # 이전 노드(TextInput)가 shared에 저장한 'text' 값을 읽어옵니다.
        return shared["text"] 
    
    def exec(self, text):
        """텍스트의 단어 수를 셉니다."""
        return len(text.split()) # 단어 수 계산 결과를 반환 (post로 전달)
    
    def post(self, shared, prep_res, exec_res):
        """shared의 단어 수 통계를 갱신합니다."""
        # shared 딕셔너리의 'stats' 값(딕셔너리)에 접근합니다.
        # exec 결과(exec_res)인 단어 수를 total_words에 더합니다.
        shared["stats"]["total_words"] += exec_res 
        
        # 'show' 액션을 반환하여 다음 노드(ShowStats)로 이동
        return "show"

```

`WordCounter` 노드는 `prep`에서 `shared["text"]` 값을 읽어 단어 수 계산에 사용하고, `post`에서 `exec` 결과를 `shared["stats"]["total_words"]`에 누적하여 저장합니다.

```python
# cookbook/pocketflow-communication/nodes.py 파일 일부 발췌 및 한국어 주석 추가

from pocketflow import Node
# ... (TextInput, WordCounter 노드 생략)

# 통계를 보여주는 노드
class ShowStats(Node):
    def prep(self, shared):
        """shared에서 통계 데이터를 가져옵니다."""
        # shared 딕셔너리의 'stats' 값을 읽어옵니다.
        return shared["stats"]
    
    def post(self, shared, prep_res, exec_res):
        """통계를 출력하고 플로우를 계속 진행할지 결정합니다."""
        # prep 결과(prep_res)인 통계 데이터를 사용합니다.
        stats = prep_res 
        print(f"\n통계:")
        print(f"- 처리된 텍스트 수: {stats['total_texts']}")
        print(f"- 총 단어 수: {stats['total_words']}")
        # 평균 단어 수 계산 및 출력
        print(f"- 텍스트당 평균 단어 수: {stats['total_words'] / stats['total_texts']:.1f}\n")
        
        # 'continue' 액션을 반환하여 플로우를 다시 시작 노드(TextInput)로 돌려보냄
        return "continue"

```

`ShowStats` 노드는 `prep`에서 `shared["stats"]` 값을 읽어오고, `post`에서는 이 데이터를 사용하여 계산된 통계를 출력합니다. 마지막으로 `"continue"` 액션을 반환하여 사용자가 새 텍스트를 입력할 수 있도록 플로우를 `TextInput` 노드로 되돌립니다.

이 세 노드가 공유 저장소(`shared`)라는 하나의 딕셔너리를 통해 데이터를 주고받으며, 누적 통계를 유지하는 것을 확인할 수 있습니다.

이 [플로우 (Flow)](04_플로우__flow__.md)는 다음과 같이 정의되고 실행됩니다.

```python
# cookbook/pocketflow-communication/flow.py 파일 일부 발췌 및 한국어 주석 추가

from pocketflow import Flow
from .nodes import TextInput, WordCounter, ShowStats, EndNode # 위에서 정의한 노드들 import

# 노드 인스턴스 생성
text_input_node = TextInput()
word_counter_node = WordCounter()
show_stats_node = ShowStats()
end_node = EndNode()

# 노드 연결 정의 (액션 사용)
text_input_node - "count" >> word_counter_node # TextInput -> WordCounter (액션: "count")
word_counter_node - "show" >> show_stats_node # WordCounter -> ShowStats (액션: "show")
show_stats_node - "continue" >> text_input_node # ShowStats -> TextInput (액션: "continue") - 루프 형성!
text_input_node - "exit" >> end_node           # TextInput -> EndNode (액션: "exit")

# Flow 객체 생성 및 시작 노드 지정
# main.py 파일에서 이 flow 객체를 import하여 사용합니다.
# 예: from flow import word_counter_flow
word_counter_flow = Flow(start=text_input_node)

# main.py 등 실행 파일에서 다음과 같이 run 메서드를 호출합니다.
# initial_shared = {} # 빈 딕셔너리로 시작
# word_counter_flow.run(shared=initial_shared) 
# print(f"최종 shared 상태: {initial_shared}") # 플로우 실행 후 상태 확인

```

`Flow(start=text_input_node)`로 `text_input_node`를 시작 노드로 지정하고, `flow.run(shared={})`와 같이 초기 빈 딕셔너리를 `shared` 인자로 넘겨주면 플로우가 실행됩니다. 이 딕셔너리가 바로 [플로우 (Flow)](04_플로우__flow__.md) 실행 전체에 걸쳐 유지되고 각 노드에 전달되는 공유 저장소입니다.

## 공유 저장소는 어떻게 전달될까요? (내부 동작)

[플로우 (Flow)](04_플로우__flow__.md)의 `.run()` 메서드가 호출되면 내부적으로 `_run()` 메서드가 실행되고, 이 메서드가 핵심 오케스트레이션 로직인 `_orch()` 메서드를 호출합니다. 이 `_orch()` 메서드가 바로 공유 저장소(`shared`)를 계속해서 다음 [노드 (Node)](02_노드__node__.md)로 전달하는 역할을 합니다.

```python
# pocketflow/__init__.py 파일 일부 발췌 (Flow 클래스의 _orch 메서드)
class Flow(BaseNode):
    # ... (초기화 및 유틸리티 메서드 생략)

    def _orch(self, shared, params=None):
        # 시작 노드부터 시작
        curr = copy.copy(self.start_node) 
        last_action = None # 이전 노드가 반환한 액션

        # 현재 실행할 노드가 있는 동안 반복
        while curr: 
            curr.set_params(params or {**self.params})
            
            # !!! 현재 노드 실행 시 shared 딕셔너리를 전달 !!!
            last_action = curr._run(shared) 
            
            # 다음 노드를 찾을 때도 shared 상태는 계속 유지됩니다.
            next_node = self.get_next_node(curr, last_action)
            
            # 다음 노드로 이동
            # 다음 노드의 _run 메서드에도 동일한 shared 딕셔너리가 전달됩니다.
            curr = copy.copy(next_node) 
            
        return last_action

    # ... (다른 메서드 생략)
```

위 `_orch` 메서드를 보면 `curr._run(shared)`와 같이 현재 [노드 (Node)](02_노드__node__.md)의 `_run` 메서드를 호출할 때 `shared` 딕셔너리를 인자로 전달하는 것을 볼 수 있습니다.

[노드 (Node)](02_노드__node__.md)의 `_run` 메서드는 이 전달받은 `shared` 딕셔너리를 자신의 `prep`, `_exec`, `post` 메서드로 다시 전달합니다.

```python
# pocketflow/__init__.py 파일 일부 발췌 (BaseNode 클래스의 _run 메서드)
# Node 클래스는 BaseNode를 상속받으므로 동일하게 동작합니다.
class BaseNode:
    # ... (다른 메서드 생략)

    def _run(self, shared):
        # !!! 전달받은 shared 딕셔너리를 prep 메서드로 전달 !!!
        prep_res = self.prep(shared)
        
        # _exec 메서드는 prep 결과를 받지만, shared는 직접 받지 않습니다.
        exec_res = self._exec(prep_res)
        
        # !!! 전달받은 shared 딕셔너리를 post 메서드로 전달 !!!
        return self.post(shared, prep_res, exec_res)

    # ... (다른 메서드 생략)
```

이러한 방식으로, PocketFlow [플로우 (Flow)](04_플로우__flow__.md)가 실행되는 동안 시작 시점에 제공된 **동일한 딕셔너리 객체**가 각 [노드 (Node)](02_노드__node__.md)의 핵심 메서드들로 계속 전달됩니다. 따라서 한 [노드 (Node)](02_노드__node__.md)가 `shared` 딕셔너리에 저장하거나 수정한 내용은 다음에 실행될 다른 [노드 (Node)](02_诺德 (Node).md)에서 즉시 확인할 수 있게 됩니다.

이 과정을 간단한 순서도로 표현하면 다음과 같습니다.

```mermaid
sequenceDiagram
    participant Flow as 플로우 (_orch)
    participant CurrentNode as 현재 노드
    participant SharedStore as 공유 저장소 (shared)

    Flow->>CurrentNode: _run(shared) 호출 (shared 전달)
    CurrentNode->>CurrentNode: prep(shared) 실행 (shared 사용)
    CurrentNode->>CurrentNode: _exec(prep 결과) 실행
    CurrentNode->>CurrentNode: post(shared, prep 결과, exec 결과) 실행 (shared 사용/수정)
    CurrentNode-->>Flow: 실행 결과 및 액션 반환
    Flow->>Flow: get_next_node(...) 호출 (shared는 다음 노드 호출 시에도 동일하게 전달될 준비)
    Note right of Flow: 다음 노드를 찾고,<br/>동일한 shared를 전달하여<br/>다시 _run 호출
    ... loop continues ...
```

보시다시피 공유 저장소(`shared`)는 [플로우 (Flow)](04_플로우__flow__.md) 실행 스택을 따라 각 [노드 (Node)](02_노드__node__.md)로 전달되며, [노드 (Node)](02_노드__node__.md)들은 이 딕셔너리를 통해 데이터를 교환합니다.

## 요약

이번 챕터에서는 PocketFlow [플로우 (Flow)](04_플로우__flow__.md) 내에서 [노드 (Node)](02_노드__node__.md)들이 데이터를 공유하는 핵심 메커니즘인 **공유 저장소 (Shared Store)** 에 대해 알아보았습니다.

*   **공유 저장소**는 [플로우 (Flow)](04_플로우__flow__.md) 내의 모든 [노드 (Node)](02_노드__node__.md)가 함께 접근할 수 있는 딕셔너리 형태의 데이터 공간입니다.
*   [노드 (Node)](02_노드__node__.md) 클래스의 `prep`, `exec_fallback`, `post` 메서드는 **`shared`** 라는 매개변수를 통해 이 공유 저장소에 접근할 수 있습니다.
*   `Flow` 객체의 `.run()` 메서드에 `shared` 딕셔너리를 인자로 전달하여 [플로우 (Flow)](04_플로우__flow__.md) 실행을 시작하며, 이 딕셔너리가 [플로우 (Flow)](04_플로우__flow__.md) 실행 내내 유지됩니다.
*   공유 저장소를 통해 각 [노드 (Node)](02_노드__node__.md)는 이전 [노드 (Node)](02_노드__node__.md)가 저장한 데이터를 읽고, 자신의 작업 결과를 저장하여 다음 [노드 (Node)](02_노드__node__.md)가 사용할 수 있도록 합니다.

공유 저장소는 PocketFlow에서 데이터 흐름을 관리하는 가장 기본적인 방법이며, 복잡한 워크플로우에서 상태를 유지하고 정보를 전달하는 데 필수적입니다.

다음 챕터에서는 여러 개의 입력을 한 번에 처리하는 **[배치 처리 (Batch Processing)](06_배치_처리__batch_processing__.md)** 에 대해 알아보겠습니다.

[Next Chapter: 배치 처리 (Batch Processing)](06_배치_처리__batch_processing__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)