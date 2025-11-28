# PBFT View Change 학습 정리

## 개요

PBFT에서 리더(Primary)가 문제가 생겼을 때 리더를 교체하는 프로토콜에 대한 학습 내용을 정리한다.

---

## View Change 전체 흐름

```
1. 기존 리더(node0) 문제 발생 (죽음/악의적 행동/응답 없음)
2. 타임아웃 발생
3. 각 노드가 StartViewChange() 호출 → ViewChange 메시지 브로드캐스트
4. 각 노드가 HandleViewChange()로 투표 수집
5. quorum(2f+1) 달성
6. 새 리더 후보(node1)가 CreateNewViewMsg() 호출
7. 새 리더가 NewViewMsg 브로드캐스트
8. 각 노드가 HandleNewView()로 검증 후 수락
9. 새 뷰로 전환 완료, 정상 합의 재개
```

---

## 핵심 함수 정리

### StartViewChange

뷰 체인지를 시작하는 함수. 각 노드가 호출한다.

하는 일:
- ViewChangeMsg 생성 (내 체크포인트, Prepared 블록들 정보 포함)
- 내 메시지 저장
- 브로드캐스트

### HandleViewChange

다른 노드가 보낸 ViewChange 메시지를 처리하는 함수.

하는 일:
- 받은 메시지 저장 (맵에 nodeID를 키로)
- quorum 체크
- 2f+1개 모이면 true 반환

### CreateNewViewMsg

새 리더가 될 노드가 호출하는 함수.

하는 일:
- 수집한 ViewChange 메시지들 정리
- computePrePrepareSet으로 하다 만 블록들 계산
- NewViewMsg 생성

주의: 이 함수 호출한다고 바로 리더 되는 게 아니다. NewViewMsg를 보내고 다른 노드들이 수락해야 공식 리더가 된다.

### HandleNewView

새 리더가 보낸 NewViewMsg를 처리하는 함수.

하는 일:
- 메시지 검증 (ViewChangeMsgs가 2f+1개인지 등)
- currentView 업데이트
- inProgress = false로 변경
- 오래된 ViewChange 메시지 정리
- 완료 콜백 호출

### computePrePrepareSet

이전 뷰에서 하다 만 블록들을 찾는 함수.

하는 일:
- 체크포인트 중 가장 높은 것 찾기 (확정된 마지막 블록)
- PreparedSet 중 가장 높은 것 찾기 (작업 중이던 마지막 블록)
- 그 사이 블록들의 PrePrepare 정보 수집

---

## 주요 개념 정리

### View

리더가 누구인지 나타내는 번호.

```
View 0 → node0이 리더
View 1 → node1이 리더
View 2 → node2가 리더
...

계산: View % 노드수 = 리더 인덱스
```

### Checkpoint

블록이 확정된 지점을 나타내는 것. 주기적으로 생성된다.

```
블록 100 완료 → 체크포인트 생성 (seq=100)
블록 200 완료 → 체크포인트 생성 (seq=200)
...
```

View와 다른 개념이다. 체크포인트는 블록 번호를 의미한다.

### PreparedSet

Prepared 상태인 블록들의 목록.

```
블록 상태 단계: Idle → PrePrepared → Prepared → Committed

PreparedSet에는 Prepared 상태인 것만 들어간다.
PrePrepared나 Committed는 들어가지 않는다.
```

### quorum

합의에 필요한 최소 투표 수.

```
전체 노드 = 3f + 1 (f는 허용 가능한 비잔틴 노드 수)
quorum = 2f + 1

예: 노드 4개 → f=1 → quorum=3
```

### inProgress

뷰 체인지 진행 중인지 나타내는 플래그.

```
true: 뷰 체인지 중 → 정상 합의 멈춤
false: 뷰 체인지 끝남 → 정상 합의 재개 가능

StartViewChange()에서 true로 설정
HandleNewView()에서 false로 설정
```

---

## Go 문법 정리

### 제로 값 (Zero Value)

Go에서 변수를 선언만 하면 자동으로 초기화된다.

```go
var x int       // x = 0
var s string    // s = ""
var b bool      // b = false
var p *int      // p = nil
```

### 이중 맵 초기화

중첩 맵에서 안쪽 맵이 nil이면 먼저 초기화해야 한다.

```go
if m[key1] == nil {
    m[key1] = make(map[string]*SomeType)
}
m[key1][key2] = value
```

안 하면 panic 발생한다.

### 포인터 생성 두 가지 방법

```go
// 방법 1: 생성할 때 &
msg := &ViewChangeMsg{...}  // msg는 *ViewChangeMsg
someMap[key] = msg          // 그냥 대입

// 방법 2: 대입할 때 &
msg := ViewChangeMsg{...}   // msg는 ViewChangeMsg
someMap[key] = &msg         // & 붙여서 대입
```

둘 다 결과는 같다. 보통 방법 1을 많이 쓴다.

### for문에서 포인터 복사

```go
for _, item := range items {
    copy := item  // 먼저 복사
    someMap[key] = &copy  // 복사본의 주소 저장
}
```

직접 &item 하면 모든 항목이 같은 주소를 가리킬 수 있어서 위험하다.

### 맵에서 삭제하면서 순회

```go
for key := range m {
    if condition {
        delete(m, key)
    }
}
```

Go에서는 맵 순회 중 삭제해도 안전하다.

### 콜백 함수 패턴

함수를 변수에 저장하고 나중에 호출한다.

```go
type Manager struct {
    callback func(int)
}

func (m *Manager) SetCallback(f func(int)) {
    m.callback = f
}

func (m *Manager) DoSomething() {
    if m.callback != nil {
        m.callback(42)
    }
}
```

의존성을 줄이고 테스트하기 좋은 코드를 만들 수 있다.

### RLock vs Lock

```go
sync.RWMutex 메서드:
- Lock() / Unlock(): 쓰기 락. 혼자만 접근 가능.
- RLock() / RUnlock(): 읽기 락. 여러 고루틴 동시 접근 가능.

읽기만 하면 RLock, 수정하면 Lock 사용.
```

### time.AfterFunc

지정한 시간 후에 함수를 실행한다.

```go
timer := time.AfterFunc(10*time.Second, func() {
    fmt.Println("10초 지남!")
})

// 취소하려면
timer.Stop()
```

별도 고루틴에서 비동기로 실행된다.

### json.Marshal

구조체를 JSON 바이트로 변환한다.

```go
data, err := json.Marshal(someStruct)
// data = []byte(`{"field":"value",...}`)
```

네트워크로 전송할 때 사용한다.

---

## 데이터 구조

### viewChangeMsgs

```go
map[uint64]map[string]*ViewChangeMsg
    ↑         ↑          ↑
  뷰번호    노드ID      메시지

예:
viewChangeMsgs[1]["node0"] = node0이 보낸 "View 1로 바꾸자" 메시지
```

### preparedBlocks (computePrePrepareSet에서)

```go
map[uint64]*PrePrepareMsg
    ↑           ↑
  블록번호    블록정보

예:
preparedBlocks[301] = 블록301의 PrePrepare 정보
```

---

## 참고

- 같은 패키지(폴더) 내 파일들은 import 없이 서로 참조 가능하다.
- ViewChangeMsg, NewViewMsg 등은 messages.go에 정의되어 있다.
