# PBFT View Change 완전 플로우 다이어그램

## 파일 구조

```
engine.go         - PBFT 메인 엔진, ViewChange 시작/처리
view_change.go    - ViewChange 프로토콜 로직, 메시지 관리
```

---

## 1. 전체 시퀀스 다이어그램 (상세)

```mermaid
sequenceDiagram
    autonumber
    participant Timer as ViewChangeTimer
    participant E1 as node1 Engine
    participant VCM1 as node1 ViewChangeManager
    participant E2 as node2 Engine
    participant VCM2 as node2 ViewChangeManager
    participant E3 as node3 Engine
    participant VCM3 as node3 ViewChangeManager

    Note over Timer,VCM3: 🔴 View 0 - node0(리더) 죽음! 타임아웃 발생

    rect rgb(255, 200, 200)
        Note over Timer,VCM3: 단계 1: 타임아웃 → StartViewChange
        
        Timer->>E1: onTimeout() 호출
        E1->>E1: collectCheckpoints()
        E1->>E1: collectPreparedCertificates()
        E1->>VCM1: StartViewChange(newView=1, checkpoints, preparedSet)
        VCM1->>VCM1: inProgress = true
        VCM1->>VCM1: ViewChangeMsg 생성
        VCM1->>VCM1: viewChangeMsgs[1]["node1"] = msg 저장
        VCM1-->>E2: broadcast ViewChangeMsg
        VCM1-->>E3: broadcast ViewChangeMsg
        
        Note over E2,E3: node2, node3도 동일하게 타임아웃
        E2->>VCM2: StartViewChange(newView=1, ...)
        VCM2-->>E1: broadcast ViewChangeMsg
        VCM2-->>E3: broadcast ViewChangeMsg
        
        E3->>VCM3: StartViewChange(newView=1, ...)
        VCM3-->>E1: broadcast ViewChangeMsg
        VCM3-->>E2: broadcast ViewChangeMsg
    end

    rect rgb(200, 255, 200)
        Note over Timer,VCM3: 단계 2: HandleViewChange (투표 수집)
        
        E1->>E1: handleViewChange(msg from node2)
        E1->>VCM1: HandleViewChange(msg)
        VCM1->>VCM1: viewChangeMsgs[1]["node2"] = msg
        VCM1-->>E1: return false (2개, quorum 미달)
        
        E1->>E1: handleViewChange(msg from node3)
        E1->>VCM1: HandleViewChange(msg)
        VCM1->>VCM1: viewChangeMsgs[1]["node3"] = msg
        VCM1-->>E1: return true (3개, quorum 달성!)
        
        Note over E1: node1이 View 1 리더 (1 % 4 = 1)
    end

    rect rgb(200, 200, 255)
        Note over Timer,VCM3: 단계 3: CreateNewViewMsg (새 리더가 생성)
        
        E1->>E1: broadcastNewView(newView=1)
        E1->>VCM1: CreateNewViewMsg(newView=1)
        VCM1->>VCM1: quorum 확인 (3개 ✓)
        VCM1->>VCM1: vcMsgs 슬라이스로 변환
        VCM1->>VCM1: computePrePrepareSet(vcMsgs)
        Note over VCM1: maxCheckpoint=300 찾기<br/>maxPrepared=302 찾기<br/>블록 301,302 수집
        VCM1-->>E1: return NewViewMsg
        
        E1-->>E2: broadcast NewViewMsg
        E1-->>E3: broadcast NewViewMsg
        E1->>VCM1: HandleNewView(newViewMsg) 자기도 처리
    end

    rect rgb(255, 255, 200)
        Note over Timer,VCM3: 단계 4: HandleNewView (검증 및 수락)
        
        E2->>E2: handleNewView(msg)
        E2->>E2: 리더 검증 (node1 맞음 ✓)
        E2->>VCM2: HandleNewView(msg)
        VCM2->>VCM2: verifyNewViewMsg(msg)
        Note over VCM2: quorum 확인 ✓<br/>뷰 번호 일치 ✓<br/>PrePrepare 검증 ✓
        VCM2->>VCM2: currentView = 1
        VCM2->>VCM2: inProgress = false
        VCM2->>VCM2: 오래된 메시지 삭제
        VCM2->>E2: onViewChangeComplete(1) 콜백
        E2->>E2: e.view = 1
        VCM2-->>E2: return true
        
        E2->>E2: reprocessPrePrepare(블록301, view=1)
        E2->>E2: reprocessPrePrepare(블록302, view=1)
        E2-->>E1: broadcast Prepare for 301
        E2-->>E1: broadcast Prepare for 302
        
        Note over E3: node3도 동일하게 처리
    end

    Note over Timer,VCM3: 🟢 View 1 - node1이 새 리더, 정상 합의 재개
```

---

## 2. 함수 호출 플로우차트 (engine.go ↔ view_change.go)

```mermaid
flowchart TD
    subgraph 타이머["⏰ 타임아웃 발생"]
        A[10초 경과<br/>리더 응답 없음]
    end

    subgraph engine_start["📁 engine.go - startViewChange"]
        B[startViewChange 호출]
        B1[newView = view + 1]
        B2[collectCheckpoints<br/>체크포인트 맵 → 슬라이스]
        B3[collectPreparedCertificates<br/>Prepared 상태 블록 수집]
    end

    subgraph vcm_start["📁 view_change.go - StartViewChange"]
        C[StartViewChange 호출]
        C1[newView <= currentView?]
        C2[inProgress = true]
        C3[ViewChangeMsg 생성]
        C4["viewChangeMsgs[newView][nodeID] = msg"]
        C5[broadcastFunc 호출]
    end

    A --> B
    B --> B1 --> B2 --> B3 --> C
    C --> C1
    C1 -->|Yes| C_END[return 무시]
    C1 -->|No| C2 --> C3 --> C4 --> C5

    subgraph broadcast["📡 네트워크 브로드캐스트"]
        D[ViewChangeMsg → 다른 노드들]
    end

    C5 --> D

    subgraph engine_handle["📁 engine.go - handleViewChange"]
        E[handleViewChange 호출]
        E1[json.Unmarshal 디코딩]
        E2[newView <= currentView?]
    end

    subgraph vcm_handle["📁 view_change.go - HandleViewChange"]
        F[HandleViewChange 호출]
        F1["viewChangeMsgs[newView] 맵 초기화"]
        F2["viewChangeMsgs[newView][nodeID] = msg"]
        F3["len >= quorumSize?"]
    end

    D --> E
    E --> E1 --> E2
    E2 -->|Yes| E_END[return 무시]
    E2 -->|No| F
    F --> F1 --> F2 --> F3
    F3 -->|No| F_WAIT[return false<br/>더 기다림]
    F3 -->|Yes| G

    subgraph engine_check["📁 engine.go - 리더 확인"]
        G[quorum 달성!]
        G1["newPrimaryIdx = newView % len(validators)"]
        G2[내가 새 리더?]
    end

    G --> G1 --> G2
    G2 -->|No| G_WAIT[NewView 메시지 대기]
    G2 -->|Yes| H

    subgraph engine_broadcast["📁 engine.go - broadcastNewView"]
        H[broadcastNewView 호출]
    end

    subgraph vcm_create["📁 view_change.go - CreateNewViewMsg"]
        I[CreateNewViewMsg 호출]
        I1["len(viewChangeMsgs) >= quorum?"]
        I2[맵 → 슬라이스 변환]
        I3[computePrePrepareSet 호출]
    end

    subgraph vcm_compute["📁 view_change.go - computePrePrepareSet"]
        J[computePrePrepareSet]
        J1[maxCheckpoint 찾기<br/>체크포인트 중 최대값]
        J2[maxPrepared 찾기<br/>Prepared 중 최대값]
        J3[preparedBlocks 맵 수집]
        J4["for seqNum = maxCheckpoint+1 to maxPrepared"]
        J5[prePrepares 슬라이스 생성]
    end

    H --> I
    I --> I1
    I1 -->|No| I_FAIL[return nil]
    I1 -->|Yes| I2 --> I3 --> J
    J --> J1 --> J2 --> J3 --> J4 --> J5

    subgraph vcm_newview["📁 view_change.go - NewViewMsg 생성"]
        K[NewViewMsg 생성]
        K1["View: newView"]
        K2["ViewChangeMsgs: 투표들"]
        K3["PrePrepareMsgs: 하다만 블록들"]
        K4["NewPrimaryID: 내 ID"]
    end

    J5 --> K
    K --> K1 --> K2 --> K3 --> K4

    subgraph broadcast2["📡 NewView 브로드캐스트"]
        L[NewViewMsg → 다른 노드들]
    end

    K4 --> L

    subgraph engine_handlenew["📁 engine.go - handleNewView"]
        M[handleNewView 호출]
        M1[json.Unmarshal 디코딩]
        M2[newView <= currentView?]
        M3["expectedPrimary = View % len"]
        M4[리더가 맞는지 검증]
    end

    L --> M
    M --> M1 --> M2
    M2 -->|Yes| M_END[return 무시]
    M2 -->|No| M3 --> M4
    M4 -->|틀림| M_REJECT[return 거부]
    M4 -->|맞음| N

    subgraph vcm_handlenew["📁 view_change.go - HandleNewView"]
        N[HandleNewView 호출]
        N1[verifyNewViewMsg 호출]
    end

    subgraph vcm_verify["📁 view_change.go - verifyNewViewMsg"]
        O[verifyNewViewMsg]
        O1["len(ViewChangeMsgs) >= quorum?"]
        O2[모든 메시지가 같은 View?]
        O3[PrePrepare 집합 검증]
        O4[Digest 충돌 확인]
        O5[누락된 블록 확인]
    end

    N --> N1 --> O
    O --> O1
    O1 -->|No| O_FAIL[return false]
    O1 -->|Yes| O2
    O2 -->|No| O_FAIL
    O2 -->|Yes| O3 --> O4 --> O5

    subgraph vcm_complete["📁 view_change.go - 완료 처리"]
        P[검증 통과!]
        P1["newViewMsgs[View] = msg"]
        P2[currentView = newView]
        P3[inProgress = false]
        P4[오래된 메시지 삭제]
        P5[onViewChangeComplete 콜백]
    end

    O5 --> P
    P --> P1 --> P2 --> P3 --> P4 --> P5

    subgraph engine_callback["📁 engine.go - onViewChangeComplete"]
        Q[onViewChangeComplete 콜백]
        Q1[e.view = newView]
        Q2[메트릭 업데이트]
        Q3[타이머 리셋]
    end

    P5 --> Q
    Q --> Q1 --> Q2 --> Q3

    subgraph engine_reprocess["📁 engine.go - reprocessPrePrepare"]
        R[reprocessPrePrepare 호출]
        R1[prePrepare.View = newView]
        R2[StateLog에 저장]
        R3[Prepare 메시지 브로드캐스트]
    end

    Q3 --> R
    R --> R1 --> R2 --> R3

    subgraph final["🟢 정상 합의 재개"]
        S[View 1 시작!]
        S1[새 리더가 블록 제안]
        S2[Prepare → Commit → Execute]
    end

    R3 --> S
    S --> S1 --> S2
```

---

## 3. 데이터 흐름 다이어그램

```mermaid
flowchart LR
    subgraph engine["engine.go"]
        E_CP[checkpoints 맵<br/>seqNum → hash]
        E_SL[stateLog<br/>StateLog]
        E_VCM[viewChangeManager<br/>*ViewChangeManager]
    end

    subgraph view_change["view_change.go"]
        V_MSGS["viewChangeMsgs<br/>map[view]map[nodeID]*ViewChangeMsg"]
        V_NEW["newViewMsgs<br/>map[view]*NewViewMsg"]
        V_VIEW[currentView<br/>uint64]
        V_PROG[inProgress<br/>bool]
    end

    subgraph messages["메시지 구조"]
        MSG_VC["ViewChangeMsg<br/>- NewView<br/>- LastSeqNum<br/>- Checkpoints[]<br/>- PreparedSet[]<br/>- NodeID"]
        MSG_NV["NewViewMsg<br/>- View<br/>- ViewChangeMsgs[]<br/>- PrePrepareMsgs[]<br/>- NewPrimaryID"]
    end

    E_CP -->|collectCheckpoints| MSG_VC
    E_SL -->|collectPreparedCertificates| MSG_VC
    MSG_VC -->|저장| V_MSGS
    V_MSGS -->|CreateNewViewMsg| MSG_NV
    MSG_NV -->|HandleNewView| V_NEW
    MSG_NV -->|검증 후| V_VIEW
```

---

## 4. 상태 전이 다이어그램

```mermaid
stateDiagram-v2
    [*] --> 정상합의: Engine.Start()

    state 정상합의 {
        [*] --> 블록제안
        블록제안 --> PrePrepare: 리더가 제안
        PrePrepare --> Prepare: 2f+1 Prepare
        Prepare --> Commit: 2f+1 Commit
        Commit --> 실행: executeBlock
        실행 --> 블록제안: 다음 블록
    }

    정상합의 --> 타임아웃: 10초 리더 응답 없음
    
    state 타임아웃 {
        [*] --> startViewChange
        startViewChange --> collectCheckpoints
        collectCheckpoints --> collectPreparedCertificates
        collectPreparedCertificates --> VCM_StartViewChange
    }

    타임아웃 --> ViewChange진행중: inProgress = true

    state ViewChange진행중 {
        [*] --> ViewChangeMsg브로드캐스트
        ViewChangeMsg브로드캐스트 --> HandleViewChange
        HandleViewChange --> 투표수집중
        
        state 투표수집중 {
            [*] --> 대기
            대기 --> 메시지도착: ViewChangeMsg 수신
            메시지도착 --> quorum확인
            quorum확인 --> 대기: 부족
            quorum확인 --> 리더확인: 달성!
        }
        
        리더확인 --> CreateNewViewMsg: 내가 리더
        리더확인 --> NewView대기: 리더 아님
        CreateNewViewMsg --> NewViewMsg브로드캐스트
    }

    ViewChange진행중 --> HandleNewView: NewViewMsg 수신

    state HandleNewView {
        [*] --> 리더검증
        리더검증 --> verifyNewViewMsg: View%N 확인
        verifyNewViewMsg --> quorum검증
        quorum검증 --> View일치검증
        View일치검증 --> PrePrepare검증
        PrePrepare검증 --> 검증완료
    }

    HandleNewView --> 뷰전환완료: 검증 통과

    state 뷰전환완료 {
        [*] --> currentView업데이트
        currentView업데이트 --> inProgress_false: inProgress = false
        inProgress_false --> 콜백호출: onViewChangeComplete
        콜백호출 --> reprocessPrePrepare
        reprocessPrePrepare --> 하다만블록재처리
    }

    뷰전환완료 --> 정상합의: 새 뷰에서 재개
```

---

## 5. 노드별 상세 타임라인

```mermaid
gantt
    title View Change 타임라인 (node1 기준)
    dateFormat X
    axisFormat %s초

    section 정상 상태
    View 0 정상 합의     :a1, 0, 10

    section 타임아웃
    리더 응답 대기       :a2, 10, 20
    타임아웃 발생!       :crit, a3, 20, 21

    section engine.go
    startViewChange()    :b1, 21, 23
    collectCheckpoints() :b2, 23, 24
    collectPreparedCert():b3, 24, 25

    section view_change.go (Start)
    StartViewChange()    :c1, 25, 27
    ViewChangeMsg 생성   :c2, 27, 28
    브로드캐스트         :c3, 28, 30

    section 투표 수집
    node2 메시지 수신    :d1, 30, 32
    HandleViewChange()   :d2, 32, 33
    node3 메시지 수신    :d3, 33, 35
    HandleViewChange()   :d4, 35, 36
    quorum 달성!         :crit, d5, 36, 37

    section view_change.go (Create)
    CreateNewViewMsg()   :e1, 37, 39
    computePrePrepareSet():e2, 39, 41
    NewViewMsg 생성      :e3, 41, 42

    section engine.go (Broadcast)
    broadcastNewView()   :f1, 42, 44
    NewViewMsg 전송      :f2, 44, 46

    section view_change.go (Handle)
    HandleNewView()      :g1, 46, 48
    verifyNewViewMsg()   :g2, 48, 50
    currentView = 1      :g3, 50, 51
    콜백 호출            :g4, 51, 52

    section engine.go (Callback)
    onViewChangeComplete():h1, 52, 54
    e.view = 1           :h2, 54, 55
    reprocessPrePrepare():h3, 55, 57

    section 새 View
    View 1 시작!         :milestone, 57, 58
    정상 합의 재개       :i1, 58, 70
```

---

## 6. 검증 로직 상세 (verifyNewViewMsg)

```mermaid
flowchart TD
    A[verifyNewViewMsg 시작] --> B{ViewChangeMsgs<br/>2f+1개 이상?}
    
    B -->|No| FAIL1[❌ return false<br/>quorum 부족]
    B -->|Yes| C

    C[모든 ViewChangeMsg 순회] --> D{모든 메시지가<br/>같은 View?}
    
    D -->|No| FAIL2[❌ return false<br/>View 불일치]
    D -->|Yes| E

    E[maxCheckpoint 계산] --> F[각 ViewChangeMsg의<br/>Checkpoints에서 최대값]
    
    F --> G[Prepared 블록 수집] --> H[preparedBlocks 맵 생성<br/>seqNum → digest]

    H --> I{같은 seqNum에<br/>다른 digest?}
    
    I -->|Yes| FAIL3[❌ return false<br/>충돌 발견 - 비잔틴!]
    I -->|No| J

    J[NewView.PrePrepareMsgs 검증] --> K{체크포인트 이하<br/>시퀀스 포함?}
    
    K -->|Yes| FAIL4[❌ return false<br/>이미 커밋된 블록]
    K -->|No| L

    L{Prepared 블록과<br/>digest 일치?}
    
    L -->|No| FAIL5[❌ return false<br/>digest 불일치]
    L -->|Yes| M

    M[누락 검사] --> N{"maxCheckpoint+1 ~<br/>maxPrepared 사이<br/>모든 Prepared 블록<br/>포함?"}
    
    N -->|No| FAIL6[❌ return false<br/>블록 누락]
    N -->|Yes| SUCCESS[✅ return true<br/>검증 통과!]

    style FAIL1 fill:#ffcccc
    style FAIL2 fill:#ffcccc
    style FAIL3 fill:#ffcccc
    style FAIL4 fill:#ffcccc
    style FAIL5 fill:#ffcccc
    style FAIL6 fill:#ffcccc
    style SUCCESS fill:#ccffcc
```

---

## 7. Checkpoint와 View Change 관계

```mermaid
flowchart TD
    subgraph 블록진행["블록 진행 상황"]
        B1[블록 1~100 ✓]
        B2[블록 101~200 ✓]
        B3[블록 201~300 ✓]
        B4[블록 301 - Prepared]
        B5[블록 302 - Prepared]
        B6[블록 303 - PrePrepared]
    end

    subgraph 체크포인트["체크포인트 생성"]
        CP1[Checkpoint 100<br/>hash: abc...]
        CP2[Checkpoint 200<br/>hash: def...]
        CP3[Checkpoint 300<br/>hash: ghi...]
    end

    subgraph ViewChange시["View Change 시"]
        VC1[collectCheckpoints]
        VC2["Checkpoints: [100, 200, 300]"]
        VC3[collectPreparedCertificates]
        VC4["PreparedSet: [301, 302]"]
    end

    subgraph 계산["computePrePrepareSet"]
        CALC1[maxCheckpoint = 300]
        CALC2[maxPrepared = 302]
        CALC3["결과: [301, 302]"]
    end

    B1 --> CP1
    B2 --> CP2
    B3 --> CP3
    
    CP1 & CP2 & CP3 --> VC1
    VC1 --> VC2
    
    B4 & B5 --> VC3
    VC3 --> VC4
    
    VC2 --> CALC1
    VC4 --> CALC2
    CALC1 & CALC2 --> CALC3

    subgraph 결과["NewView 결과"]
        R1[블록 301 재합의]
        R2[블록 302 재합의]
        R3[블록 303은 포함 안됨<br/>PrePrepared 상태라서]
    end

    CALC3 --> R1 & R2
```

---

## 8. 에러 케이스 및 처리

```mermaid
flowchart TD
    subgraph 정상케이스["✅ 정상 케이스"]
        N1[타임아웃] --> N2[ViewChange 투표]
        N2 --> N3[quorum 달성]
        N3 --> N4[새 리더 NewView]
        N4 --> N5[검증 통과]
        N5 --> N6[정상 재개]
    end

    subgraph 에러1["❌ 에러: 오래된 View"]
        E1_1["newView <= currentView"]
        E1_2[StartViewChange: return 무시]
        E1_3[HandleViewChange: return 무시]
    end

    subgraph 에러2["❌ 에러: quorum 미달"]
        E2_1[투표 2개만 도착]
        E2_2[HandleViewChange: return false]
        E2_3[계속 대기...]
        E2_4[타임아웃 → 다시 시도]
    end

    subgraph 에러3["❌ 에러: 잘못된 리더"]
        E3_1[NewView 수신]
        E3_2["expectedPrimary ≠ msg.NewPrimaryID"]
        E3_3[handleNewView: return 거부]
    end

    subgraph 에러4["❌ 에러: 검증 실패"]
        E4_1[verifyNewViewMsg 실패]
        E4_2[quorum 부족]
        E4_3[View 불일치]
        E4_4[digest 충돌]
        E4_5[HandleNewView: return false]
    end

    subgraph 에러5["❌ 에러: 비잔틴 노드"]
        E5_1[같은 seqNum에 다른 블록]
        E5_2["preparedBlocks 충돌 감지"]
        E5_3[verifyNewViewMsg: return false]
        E5_4[악의적 노드 무시]
    end
```

---

## 9. 콜백 연결 구조

```mermaid
flowchart TD
    subgraph engine_init["engine.go - NewEngine"]
        A1[ViewChangeManager 생성]
        A2["SetBroadcastFunc(engine.broadcast)"]
        A3["SetOnViewChangeComplete(engine.onViewChangeComplete)"]
    end

    subgraph view_change_store["view_change.go - 저장"]
        B1[broadcastFunc = f]
        B2[onViewChangeComplete = f]
    end

    subgraph view_change_use["view_change.go - 사용"]
        C1["StartViewChange에서<br/>broadcastFunc(msg) 호출"]
        C2["HandleNewView에서<br/>onViewChangeComplete(view) 호출"]
    end

    subgraph engine_impl["engine.go - 실제 함수"]
        D1["broadcast(msg)<br/>transport.Broadcast(msg)"]
        D2["onViewChangeComplete(view)<br/>e.view = view"]
    end

    A1 --> A2 --> B1
    A1 --> A3 --> B2
    B1 --> C1 --> D1
    B2 --> C2 --> D2
```

---

## 10. 전체 요약 플로우

```mermaid
flowchart TD
    START[🔴 리더 죽음] --> TIMEOUT[⏰ 10초 타임아웃]
    
    TIMEOUT --> |engine.go| SC[startViewChange]
    SC --> COLLECT[체크포인트 + PreparedSet 수집]
    
    COLLECT --> |view_change.go| SVC[StartViewChange]
    SVC --> VCMSG[ViewChangeMsg 브로드캐스트]
    
    VCMSG --> |engine.go| HVC[handleViewChange]
    HVC --> |view_change.go| HVC2[HandleViewChange]
    HVC2 --> QUORUM{quorum<br/>달성?}
    
    QUORUM -->|No| VCMSG
    QUORUM -->|Yes| LEADER{내가<br/>새 리더?}
    
    LEADER -->|No| WAIT[NewView 대기]
    LEADER -->|Yes| |engine.go| BNV[broadcastNewView]
    
    BNV --> |view_change.go| CNVM[CreateNewViewMsg]
    CNVM --> COMPUTE[computePrePrepareSet]
    COMPUTE --> NVMSG[NewViewMsg 브로드캐스트]
    
    NVMSG --> |engine.go| HNV[handleNewView]
    WAIT --> HNV
    
    HNV --> |view_change.go| HNV2[HandleNewView]
    HNV2 --> VERIFY[verifyNewViewMsg]
    
    VERIFY --> VALID{검증<br/>통과?}
    VALID -->|No| REJECT[거부]
    VALID -->|Yes| UPDATE[currentView 업데이트]
    
    UPDATE --> CALLBACK[onViewChangeComplete 콜백]
    CALLBACK --> |engine.go| EVIEW[e.view = newView]
    
    EVIEW --> REPROCESS[reprocessPrePrepare<br/>하다 만 블록 재처리]
    
    REPROCESS --> END[🟢 정상 합의 재개!]

    style START fill:#ffcccc
    style END fill:#ccffcc
    style TIMEOUT fill:#ffffcc
    style QUORUM fill:#ccccff
    style LEADER fill:#ccccff
    style VALID fill:#ccccff
```
