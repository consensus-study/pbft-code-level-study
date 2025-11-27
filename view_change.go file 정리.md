# PBFT View Change 정리

## View Change란

PBFT에서 리더(Primary)가 응답하지 않거나 악의적인 행동을 할 때 리더를 교체하는 프로토콜이다. 정상적인 상황에서는 View 0의 리더인 Node0이 블록을 제안하고 합의를 진행한다. 하지만 리더에 문제가 생기면 타임아웃이 발생하고, View Change를 통해 새로운 리더로 교체된다.

리더는 View 번호에 따라 결정된다. View 번호를 노드 수로 나눈 나머지가 리더의 인덱스가 된다. 예를 들어 노드가 4개라면 View 0에서는 Node0, View 1에서는 Node1이 리더가 된다.

---

## ViewChangeManager 구조체

```go
type ViewChangeManager struct {
    mu sync.RWMutex
    currentView    uint64
    viewChangeMsgs map[uint64]map[string]*ViewChangeMsg
    newViewMsgs    map[uint64]*NewViewMsg
    inProgress     bool
    quorumSize     int
    nodeID         string
    broadcastFunc        func(*Message)
    onViewChangeComplete func(newView uint64)
}
```

### 주요 필드 설명

`currentView`는 현재 뷰 번호를 저장한다. 뷰 체인지가 완료되면 이 값이 증가한다.

`viewChangeMsgs`는 이중 맵 구조로 되어 있다. 첫 번째 키는 뷰 번호이고, 두 번째 키는 노드 ID이다. 각 노드가 보낸 ViewChange 메시지를 저장하며, 같은 노드가 여러 번 보내도 하나만 저장된다.

`inProgress`는 현재 뷰 체인지가 진행 중인지를 나타낸다.

`quorumSize`는 합의에 필요한 최소 투표 수로, 보통 2f+1이다.

`broadcastFunc`와 `onViewChangeComplete`는 콜백 함수다. ViewChangeManager가 직접 네트워크 코드나 다른 모듈에 의존하지 않고, 필요한 동작을 외부에서 주입받아 실행한다.

---

## 콜백 함수 패턴

Go에서는 함수를 변수에 저장할 수 있다. ViewChangeManager는 이 특성을 활용해서 브로드캐스트 로직이나 완료 후 처리 로직을 외부에서 주입받는다.

```go
vcm := NewViewChangeManager("node0", 3)

vcm.SetBroadcastFunc(func(msg *Message) {
    // 실제 네트워크 전송 코드
    network.Broadcast(msg)
})

vcm.SetOnViewChangeComplete(func(newView uint64) {
    // 뷰 체인지 완료 후 처리
    resumeConsensus()
})
```

이렇게 하면 ViewChangeManager는 네트워크 패키지에 의존하지 않아도 된다. 테스트할 때도 가짜 함수를 넣어서 쉽게 테스트할 수 있다.

---

## StartViewChange 함수

뷰 체인지를 시작하는 함수다.

```go
func (vcm *ViewChangeManager) StartViewChange(newView uint64, lastSeqNum uint64, 
    checkpoints []Checkpoint, preparedSet []PreparedCert)
```

먼저 새로운 뷰 번호가 현재 뷰보다 큰지 확인한다. 같거나 작으면 이미 처리된 것이므로 무시한다.

그 다음 ViewChangeMsg를 생성한다. 이 메시지에는 새 뷰 번호, 마지막 시퀀스, 체크포인트 정보, 그리고 현재 Prepared 상태인 블록 정보가 담긴다.

생성한 메시지는 자신의 맵에 저장하고, broadcastFunc를 통해 다른 노드들에게 전송한다.

---

## HandleViewChange 함수

다른 노드가 보낸 ViewChange 메시지를 처리한다.

```go
func (vcm *ViewChangeManager) HandleViewChange(msg *ViewChangeMsg) bool
```

받은 메시지를 맵에 저장한다. 이때 이중 맵의 안쪽 맵이 없으면 먼저 생성해야 한다.

```go
if vcm.viewChangeMsgs[msg.NewView] == nil {
    vcm.viewChangeMsgs[msg.NewView] = make(map[string]*ViewChangeMsg)
}
vcm.viewChangeMsgs[msg.NewView][msg.NodeID] = msg
```

Go에서 중첩 맵을 사용할 때는 안쪽 맵을 먼저 초기화해야 한다. 그렇지 않으면 nil 맵에 접근하게 되어 panic이 발생한다.

메시지를 저장한 후 해당 뷰에 대한 메시지가 quorumSize 이상이면 true를 반환한다.

---

## CreateNewViewMsg 함수

새 리더가 호출하는 함수다. 충분한 ViewChange 메시지를 받은 후 NewView 메시지를 생성한다.

핵심은 computePrePrepareSet 호출이다. 이전 뷰에서 진행 중이던 블록들을 찾아서 새 뷰에서 다시 처리할 목록을 만든다.

---

## computePrePrepareSet 함수

이전 뷰에서 하다 만 작업을 찾는 함수다.

각 노드가 ViewChange 메시지에 자신의 체크포인트와 Prepared 상태 블록 정보를 담아서 보낸다. 이 함수는 그 정보들을 모아서 분석한다.

먼저 모든 체크포인트 중 가장 높은 시퀀스 번호를 찾는다. 체크포인트까지는 확실히 완료된 것이다.

그 다음 Prepared 상태인 블록 중 가장 높은 시퀀스 번호를 찾는다. 이것이 이전 리더가 마지막으로 작업하던 블록이다.

체크포인트와 마지막 Prepared 사이의 블록들이 "하다 만 작업"이다. 새 리더는 이 블록들의 PrePrepare를 다시 브로드캐스트해서 합의를 처음부터 진행한다.

예를 들어 체크포인트가 5이고 마지막 Prepared가 8이면, 블록 6, 7, 8에 대한 PrePrepare 메시지 목록을 반환한다.

---

## HandleNewView 함수

새 리더가 보낸 NewView 메시지를 처리한다.

먼저 메시지를 검증한다. ViewChange 메시지가 quorumSize 이상인지, 모든 메시지가 같은 뷰에 대한 것인지 확인한다.

검증에 통과하면 현재 뷰를 업데이트하고, 진행 중 플래그를 해제하고, 오래된 ViewChange 메시지들을 정리한다.

마지막으로 onViewChangeComplete 콜백을 호출해서 뷰 체인지가 완료되었음을 알린다.

---

## ViewChangeTimer 구조체

리더 타임아웃을 관리하는 타이머다.

```go
type ViewChangeTimer struct {
    mu        sync.Mutex
    timeout   time.Duration
    timer     *time.Timer
    onTimeout func()
}
```

Start 함수를 호출하면 타이머가 시작된다. timeout 시간이 지나면 onTimeout 함수가 자동으로 호출된다.

블록이 정상적으로 도착하면 Stop으로 타이머를 취소한다. 타임아웃이 발생하면 onTimeout에서 StartViewChange를 호출해서 뷰 체인지를 시작한다.

time.AfterFunc는 지정한 시간 후에 함수를 실행하는 Go 표준 라이브러리 함수다. 별도의 고루틴에서 실행된다.

---

## View Change 전체 흐름

1. 타이머가 만료되면 onTimeout이 호출되고 StartViewChange가 실행된다.

2. 각 노드가 ViewChange 메시지를 브로드캐스트한다.

3. HandleViewChange에서 메시지를 수집한다. quorumSize 이상 모이면 true를 반환한다.

4. 새 리더가 CreateNewViewMsg를 호출해서 NewView 메시지를 생성한다.

5. 새 리더가 NewView 메시지를 브로드캐스트한다.

6. 각 노드가 HandleNewView에서 메시지를 검증하고 뷰를 업데이트한다.

7. onViewChangeComplete 콜백이 호출되고 정상 합의가 재개된다.

---

## Go 문법 정리

### 이중 맵 초기화

```go
if vcm.viewChangeMsgs[newView] == nil {
    vcm.viewChangeMsgs[newView] = make(map[string]*ViewChangeMsg)
}
vcm.viewChangeMsgs[newView][nodeID] = msg
```

중첩 맵에서 안쪽 맵이 nil이면 먼저 make로 초기화해야 한다. 그렇지 않으면 panic이 발생한다.

### 함수 타입 변수

```go
broadcastFunc func(*Message)
```

Go에서 함수도 변수에 저장할 수 있다. 이를 활용하면 의존성을 줄이고 테스트하기 쉬운 코드를 작성할 수 있다.

### for range 맵 순회

```go
for v := range vcm.viewChangeMsgs {
    delete(vcm.viewChangeMsgs, v)
}
```

맵을 range로 순회할 때 키만 필요하면 값 변수를 생략할 수 있다. 순회 중에 delete로 항목을 삭제하는 것도 Go에서는 안전하다.

### 포인터 역참조

```go
vcMsgs = append(vcMsgs, *msg)
```

msg가 포인터일 때 *msg는 포인터가 가리키는 실제 값이다. 이렇게 하면 값을 복사해서 저장하므로 원본이 바뀌어도 영향받지 않는다.

### time.AfterFunc

```go
vct.timer = time.AfterFunc(vct.timeout, vct.onTimeout)
```

지정한 시간 후에 함수를 실행한다. 별도의 고루틴에서 비동기로 실행되며, 반환된 Timer로 취소할 수 있다.
