# PBFT View Change 타임아웃 상세 플로우

## 타임아웃이란?

```
타임아웃 = "리더가 일정 시간 동안 응답이 없으면 리더 교체"

기본 타임아웃: 10초
뷰 체인지 실패 시: 타임아웃 2배씩 증가 (exponential backoff)
```

---

## 1. 타임아웃 발생 조건

```mermaid
flowchart TD
    subgraph 정상상태["✅ 정상 상태"]
        A[리더가 블록 제안]
        B[노드가 PrePrepare 수신]
        C[resetViewChangeTimer 호출]
        D[타이머 리셋 - 다시 10초]
    end

    A --> B --> C --> D --> A

    subgraph 문제상황["❌ 문제 상황"]
        E[리더 죽음 / 악의적 행동 / 네트워크 문제]
        F[PrePrepare 안 옴]
        G[타이머 계속 진행...]
        H[10초 경과!]
        I[onTimeout 호출!]
    end

    E --> F --> G --> H --> I

    D -.->|리더 응답 없으면| F
```

---

## 2. 타이머 관련 코드 흐름

```mermaid
flowchart TD
    subgraph engine_start["Engine.Start()"]
        A1[엔진 시작]
        A2[resetViewChangeTimer 호출]
    end

    subgraph reset["resetViewChangeTimer()"]
        B1["기존 타이머 있으면 Stop()"]
        B2["time.AfterFunc(10초, startViewChange)"]
        B3[새 타이머 시작]
    end

    subgraph normal["정상 흐름"]
        C1[PrePrepare 수신]
        C2[handlePrePrepare 처리]
        C3[resetViewChangeTimer 호출]
        C4[타이머 리셋!]
    end

    subgraph timeout["타임아웃 흐름"]
        D1[10초 경과...]
        D2[startViewChange 자동 호출]
        D3[View Change 시작!]
    end

    A1 --> A2 --> B1 --> B2 --> B3

    B3 -->|리더 응답 있음| C1 --> C2 --> C3 --> C4 --> B1

    B3 -->|리더 응답 없음| D1 --> D2 --> D3
```

---

## 3. 단일 리더 죽음 시나리오

```mermaid
sequenceDiagram
    autonumber
    participant T as 타이머
    participant N0 as node0 (리더)
    participant N1 as node1
    participant N2 as node2
    participant N3 as node3

    Note over T,N3: View 0 - 정상 상태

    rect rgb(200, 255, 200)
        N0->>N1: PrePrepare (블록 100)
        N0->>N2: PrePrepare (블록 100)
        N0->>N3: PrePrepare (블록 100)
        N1->>T: resetViewChangeTimer()
        N2->>T: resetViewChangeTimer()
        N3->>T: resetViewChangeTimer()
        Note over T: 타이머 리셋 (0초부터 다시)
    end

    rect rgb(255, 200, 200)
        Note over N0: ❌ node0 죽음!
        Note over T: 타이머 계속 진행...
        T->>T: 1초... 2초... 3초...
        T->>T: 7초... 8초... 9초...
        T->>T: 10초 경과!
    end

    rect rgb(255, 255, 200)
        Note over T,N3: 타임아웃 발생!
        T->>N1: onTimeout() → startViewChange()
        T->>N2: onTimeout() → startViewChange()
        T->>N3: onTimeout() → startViewChange()
    end

    rect rgb(200, 200, 255)
        Note over N1,N3: View Change to View 1
        N1->>N2: ViewChangeMsg (View 1)
        N1->>N3: ViewChangeMsg (View 1)
        N2->>N1: ViewChangeMsg (View 1)
        N2->>N3: ViewChangeMsg (View 1)
        N3->>N1: ViewChangeMsg (View 1)
        N3->>N2: ViewChangeMsg (View 1)
        Note over N1,N3: quorum 달성! (3개)
    end

    rect rgb(200, 255, 200)
        Note over N1: View 1 리더 = 1 % 4 = node1
        N1->>N1: CreateNewViewMsg()
        N1->>N2: NewViewMsg
        N1->>N3: NewViewMsg
        Note over N1,N3: View 1 시작! 정상 재개
    end
```

---

## 4. 연속 리더 죽음 시나리오 (node0, node1 둘 다 죽음)

```mermaid
sequenceDiagram
    autonumber
    participant T as 타이머
    participant N0 as node0 (View 0 리더)
    participant N1 as node1 (View 1 리더)
    participant N2 as node2 (View 2 리더)
    participant N3 as node3

    Note over T,N3: View 0 - node0이 리더

    rect rgb(255, 200, 200)
        Note over N0: ❌ node0 죽음!
        Note over N1: ❌ node1도 죽음!
        T->>T: 10초 경과...
    end

    rect rgb(255, 255, 200)
        Note over T,N3: 1차 타임아웃!
        T->>N2: startViewChange()
        T->>N3: startViewChange()
        Note over N2,N3: View 1로 View Change 시도
    end

    rect rgb(200, 200, 255)
        N2->>N3: ViewChangeMsg (View 1)
        N3->>N2: ViewChangeMsg (View 1)
        Note over N2,N3: 2개만... quorum 미달!<br/>(node0, node1 없음)
    end

    rect rgb(255, 200, 200)
        Note over N2,N3: quorum 달성 못 함<br/>but 일단 계속 진행...
        Note over N1: View 1 리더 = node1<br/>근데 node1 죽어있음!
        Note over N2,N3: NewView 안 옴...
        T->>T: 20초 대기... (10초 × 2)
    end

    rect rgb(255, 255, 200)
        Note over T,N3: 2차 타임아웃! (20초)
        T->>N2: startViewChange()
        T->>N3: startViewChange()
        Note over N2,N3: View 2로 View Change 시도
    end

    rect rgb(200, 200, 255)
        N2->>N3: ViewChangeMsg (View 2)
        N3->>N2: ViewChangeMsg (View 2)
        Note over N2,N3: 여전히 2개...<br/>quorum 미달!
    end

    Note over N2,N3: 🤔 문제: quorum(3개) 필요한데<br/>살아있는 노드가 2개뿐!<br/>→ 합의 불가능 상태

    rect rgb(255, 200, 200)
        Note over T,N3: 시스템 정지 상태
        Note over N2,N3: f=1 시스템에서<br/>2개 노드 죽으면 합의 불가
    end
```

---

## 5. 연속 리더 죽음 + 복구 시나리오 (node0만 죽고, node1은 나중에 응답)

```mermaid
sequenceDiagram
    autonumber
    participant T as 타이머
    participant N0 as node0
    participant N1 as node1
    participant N2 as node2
    participant N3 as node3

    Note over T,N3: View 0 - node0이 리더

    rect rgb(255, 200, 200)
        Note over N0: ❌ node0 죽음!
        T->>T: 10초 경과...
    end

    rect rgb(255, 255, 200)
        Note over T,N3: 1차 타임아웃!
        T->>N1: startViewChange()
        T->>N2: startViewChange()
        T->>N3: startViewChange()
    end

    rect rgb(200, 200, 255)
        N1->>N2: ViewChangeMsg (View 1)
        N1->>N3: ViewChangeMsg (View 1)
        N2->>N1: ViewChangeMsg (View 1)
        N2->>N3: ViewChangeMsg (View 1)
        N3->>N1: ViewChangeMsg (View 1)
        N3->>N2: ViewChangeMsg (View 1)
        Note over N1,N3: quorum 달성! (3개)
    end

    rect rgb(255, 255, 200)
        Note over N1: View 1 리더 = node1
        Note over N1: 근데 node1이 네트워크 지연 중...
        Note over N2,N3: NewView 안 옴...
        T->>T: 20초 대기...
    end

    rect rgb(255, 200, 200)
        Note over T,N3: 2차 타임아웃! (20초)
        T->>N2: startViewChange()
        T->>N3: startViewChange()
        Note over N2,N3: View 2로 View Change 시도
    end

    rect rgb(200, 255, 200)
        Note over N1: node1 네트워크 복구!
        N1->>N1: 뒤늦게 View 1 NewView 생성
        N1->>N2: NewViewMsg (View 1)
        N1->>N3: NewViewMsg (View 1)
    end

    rect rgb(255, 255, 200)
        Note over N2,N3: View 1 NewView 수신
        N2->>N2: newViewMsg.View(1) <= currentView?
        Note over N2: 이미 View 2 투표 중...<br/>View 1은 무시!
    end

    rect rgb(200, 200, 255)
        N1->>N2: ViewChangeMsg (View 2)
        N1->>N3: ViewChangeMsg (View 2)
        Note over N1,N3: View 2 quorum 달성! (3개)
    end

    rect rgb(200, 255, 200)
        Note over N2: View 2 리더 = 2 % 4 = node2
        N2->>N1: NewViewMsg (View 2)
        N2->>N3: NewViewMsg (View 2)
        Note over N1,N3: View 2 시작! 정상 재개
    end
```

---

## 6. 타임아웃 증가 (Exponential Backoff) 상세

```mermaid
flowchart TD
    subgraph round1["1차 시도"]
        A1[View 0 → View 1]
        A2[타임아웃: 10초]
        A3[실패!]
    end

    subgraph round2["2차 시도"]
        B1[View 1 → View 2]
        B2["타임아웃: 20초 (10 × 2)"]
        B3[실패!]
    end

    subgraph round3["3차 시도"]
        C1[View 2 → View 3]
        C2["타임아웃: 40초 (20 × 2)"]
        C3[실패!]
    end

    subgraph round4["4차 시도"]
        D1[View 3 → View 4]
        D2["타임아웃: 80초 (40 × 2)"]
        D3[성공!]
    end

    A1 --> A2 --> A3 --> B1
    B1 --> B2 --> B3 --> C1
    C1 --> C2 --> C3 --> D1
    D1 --> D2 --> D3

    subgraph 코드["코드"]
        E1["e.viewChangeTimer = time.AfterFunc(<br/>e.config.ViewChangeTimeout * 2,<br/>startViewChange)"]
    end
```

```mermaid
gantt
    title 타임아웃 증가 타임라인
    dateFormat s
    axisFormat %S초

    section View 0
    정상 상태           :a1, 0, 10
    
    section View 1 시도
    타임아웃 대기 (10초) :crit, b1, 10, 20
    View Change 시도    :b2, 20, 22
    NewView 대기        :b3, 22, 25
    실패!               :crit, b4, 25, 26

    section View 2 시도
    타임아웃 대기 (20초) :crit, c1, 26, 46
    View Change 시도    :c2, 46, 48
    NewView 대기        :c3, 48, 51
    실패!               :crit, c4, 51, 52

    section View 3 시도
    타임아웃 대기 (40초) :crit, d1, 52, 92
    View Change 시도    :d2, 92, 94
    NewView 수신        :d3, 94, 96
    성공!               :done, d4, 96, 100
```

---

## 7. startViewChange 상세 흐름

```mermaid
flowchart TD
    subgraph trigger["트리거"]
        A[time.AfterFunc 타임아웃]
        A --> B[startViewChange 호출]
    end

    subgraph prepare["준비 단계"]
        B --> C["newView = e.view + 1"]
        C --> D["lastSeqNum = e.sequenceNum"]
    end

    subgraph collect["데이터 수집"]
        D --> E[collectCheckpoints]
        E --> E1["checkpoints 맵 순회"]
        E1 --> E2["[]Checkpoint 생성"]
        
        E2 --> F[collectPreparedCertificates]
        F --> F1["stateLog 윈도우 순회"]
        F1 --> F2["IsPrepared && !Executed 찾기"]
        F2 --> F3["[]PreparedCert 생성"]
    end

    subgraph vcm["ViewChangeManager 호출"]
        F3 --> G["viewChangeManager.StartViewChange(<br/>newView, lastSeqNum,<br/>checkpoints, preparedSet)"]
    end

    subgraph timer["타이머 재설정"]
        G --> H["타이머 재시작"]
        H --> I["time.AfterFunc(<br/>ViewChangeTimeout × 2,<br/>startViewChange)"]
        I --> J[다음 타임아웃 대비]
    end

    subgraph next["다음 단계"]
        J --> K{성공?}
        K -->|Yes| L[정상 재개]
        K -->|No| M[다시 타임아웃]
        M --> A
    end
```

---

## 8. resetViewChangeTimer 상세 흐름

```mermaid
flowchart TD
    subgraph when["언제 호출되나?"]
        A1[Engine.Start 시]
        A2[handlePrePrepare 후]
        A3[onViewChangeComplete 후]
    end

    subgraph func["resetViewChangeTimer()"]
        B1{"e.viewChangeTimer<br/>!= nil?"}
        B1 -->|Yes| B2["viewChangeTimer.Stop()"]
        B1 -->|No| B3[스킵]
        B2 --> B4
        B3 --> B4
        B4["time.AfterFunc(<br/>ViewChangeTimeout,<br/>startViewChange)"]
        B4 --> B5[새 타이머 저장]
    end

    subgraph result["결과"]
        C1[10초 타이머 시작]
        C2[10초 안에 리더 응답 오면 리셋]
        C3[10초 경과하면 startViewChange]
    end

    A1 --> B1
    A2 --> B1
    A3 --> B1
    B5 --> C1
    C1 --> C2
    C1 --> C3
```

---

## 9. 전체 타임아웃 상태 머신

```mermaid
stateDiagram-v2
    [*] --> 정상합의: Engine.Start()

    state 정상합의 {
        [*] --> 타이머작동
        타이머작동 --> PrePrepare수신: 리더 응답
        PrePrepare수신 --> 타이머리셋: resetViewChangeTimer()
        타이머리셋 --> 타이머작동
    }

    정상합의 --> 타임아웃발생: 10초 경과

    state 타임아웃발생 {
        [*] --> startViewChange호출
        startViewChange호출 --> 데이터수집
        데이터수집 --> ViewChange브로드캐스트
        ViewChange브로드캐스트 --> 타이머재설정
        타이머재설정 --> 대기: timeout × 2
    }

    타임아웃발생 --> ViewChange진행

    state ViewChange진행 {
        [*] --> 투표수집
        투표수집 --> quorum확인
        quorum확인 --> 투표수집: 부족
        quorum확인 --> 리더확인: 달성
        리더확인 --> NewView생성: 내가 리더
        리더확인 --> NewView대기: 리더 아님
    }

    ViewChange진행 --> 타임아웃발생: NewView 안 옴 (timeout × 2)
    ViewChange진행 --> NewView처리: NewView 수신

    state NewView처리 {
        [*] --> 검증
        검증 --> 수락: 통과
        검증 --> 거부: 실패
        수락 --> 뷰업데이트
        뷰업데이트 --> 콜백호출
        콜백호출 --> 타이머리셋2
    }

    NewView처리 --> 정상합의: 성공
    거부 --> 타임아웃발생: 다시 대기
```

---

## 10. 노드별 타이머 상태 (독립적)

```mermaid
flowchart LR
    subgraph node1["node1"]
        T1[타이머 10초]
        T1 --> |PrePrepare| R1[리셋]
        R1 --> T1
        T1 --> |타임아웃| S1[startViewChange]
    end

    subgraph node2["node2"]
        T2[타이머 10초]
        T2 --> |PrePrepare| R2[리셋]
        R2 --> T2
        T2 --> |타임아웃| S2[startViewChange]
    end

    subgraph node3["node3"]
        T3[타이머 10초]
        T3 --> |PrePrepare| R3[리셋]
        R3 --> T3
        T3 --> |타임아웃| S3[startViewChange]
    end

    subgraph note["참고"]
        N1[각 노드의 타이머는 독립적]
        N2[네트워크 지연으로 약간 다를 수 있음]
        N3[하지만 거의 비슷한 시점에 타임아웃]
    end
```

---

## 11. 실제 시간 흐름 예시

```
시간(초)  node0(리더)    node1         node2         node3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  0      블록 제안 ──────────────────────────────────────────►
  1                     타이머 리셋    타이머 리셋    타이머 리셋
  2      블록 제안 ──────────────────────────────────────────►
  3                     타이머 리셋    타이머 리셋    타이머 리셋
  4      ❌ 죽음!
  5                     타이머: 1초    타이머: 1초    타이머: 1초
  6                     타이머: 2초    타이머: 2초    타이머: 2초
  ...
  13                    타이머: 9초    타이머: 9초    타이머: 9초
  14                    ⏰ 10초!       ⏰ 10초!       ⏰ 10초!
  15                    startVC()     startVC()     startVC()
  16                    ──ViewChangeMsg 교환──────────────────
  17                    quorum!       quorum!       quorum!
  18                    CreateNewView
  19                    ──NewViewMsg 브로드캐스트────────────►
  20                    HandleNewView HandleNewView HandleNewView
  21                    View 1 시작!  View 1 시작!  View 1 시작!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 12. 타임아웃 관련 설정

```mermaid
flowchart TD
    subgraph config["Config 설정"]
        A["ViewChangeTimeout: 10초<br/>(기본값)"]
        B["RequestTimeout: 5초"]
    end

    subgraph usage["사용처"]
        C["resetViewChangeTimer()<br/>→ 10초 타이머"]
        D["startViewChange() 내부<br/>→ 10초 × 2 = 20초"]
        E["2차 실패 시<br/>→ 20초 × 2 = 40초"]
    end

    subgraph customize["커스터마이즈"]
        F["DefaultConfig에서 수정 가능"]
        G["네트워크 상황에 따라 조절"]
    end

    A --> C
    A --> D
    D --> E
    F --> A
    G --> A
```

---

## 요약

```
타임아웃 핵심 포인트:

1. 기본 타임아웃: 10초
2. 리더가 PrePrepare 보내면 타이머 리셋
3. 타임아웃 발생 시 startViewChange() 자동 호출
4. View Change 실패 시 타임아웃 2배로 증가 (exponential backoff)
5. 각 노드의 타이머는 독립적
6. 여러 리더가 연속으로 죽으면 계속 View 증가
7. quorum 불가능하면 시스템 정지 (f+1개 이상 살아있어야 함)
```
