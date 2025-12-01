## 목차

1. [전체 아키텍처](#1-전체-아키텍처)
2. [파일 구조와 역할](#2-파일-구조와-역할)
3. [계층별 상세 설명](#3-계층별-상세-설명)
4. [데이터 흐름](#4-데이터-흐름)
5. [PBFT 합의 과정](#5-pbft-합의-과정)
6. [노드 시작부터 블록 실행까지](#6-노드-시작부터-블록-실행까지)
7. [메시지 타입과 처리](#7-메시지-타입과-처리)
8. [핵심 코드 분석](#8-핵심-코드-분석)
9. [요약 정리](#9-요약-정리)

---

## 1. 전체 아키텍처

### 1.1 시스템 구성도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           PBFT-Cosmos 시스템                             │
└─────────────────────────────────────────────────────────────────────────┘

                              사용자 요청
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              Node (노드)                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         EngineV2 (PBFT 엔진)                     │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  PrePrepare → Prepare → Commit (합의 로직)               │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                              │                                          │
│  ┌───────────────────────────▼─────────────────────────────────────┐   │
│  │                       ABCIAdapter (어댑터)                       │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  PBFT 타입 ←→ ABCI 타입 변환                             │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                              │                                          │
│  ┌───────────────────────────▼─────────────────────────────────────┐   │
│  │                       ABCI Client (클라이언트)                   │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  gRPC로 Cosmos SDK 앱과 통신                             │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                              │                                          │
│  ┌───────────────────────────▼─────────────────────────────────────┐   │
│  │                     GRPCTransport (P2P 통신)                     │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  다른 PBFT 노드들과 메시지 송수신                         │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ gRPC
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Cosmos SDK 앱                                  │
│  (상태 관리, 트랜잭션 실행, 검증)                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 비유로 이해하기

```
회사 조직으로 비유:

┌─────────────────┐
│      Node       │  = 회사 전체
│    (회사)       │
└────────┬────────┘
         │
┌────────▼────────┐
│    EngineV2     │  = 사장님 (결정하는 사람)
│    (사장님)      │    "이 블록 실행할까?"
└────────┬────────┘
         │
┌────────▼────────┐
│   ABCIAdapter   │  = 비서 (통역/문서 정리)
│    (비서)        │    "PBFT 형식 → ABCI 형식 변환"
└────────┬────────┘
         │
┌────────▼────────┐
│   ABCI Client   │  = 배달부 (전달하는 사람)
│    (배달부)      │    "Cosmos SDK에 요청 전달"
└────────┬────────┘
         │
┌────────▼────────┐
│  GRPCTransport  │  = 전화/메신저 (소통 도구)
│   (전화기)       │    "다른 노드와 연락"
└─────────────────┘
```

---

## 2. 파일 구조와 역할

### 2.1 전체 파일 구조

```
pbft-cosmos/
│
├── node/                          # 노드 관련
│   ├── node.go                    # 노드 메인 구조체
│   └── config.go                  # 노드 설정
│
├── consensus/pbft/                # PBFT 합의 엔진
│   ├── engine_v2.go               # ABCI 2.0 호환 엔진
│   ├── abci_adapter.go            # ABCI 어댑터
│   ├── message.go                 # 메시지 타입
│   ├── state.go                   # 상태 관리
│   └── view_change.go             # 뷰 체인지
│
├── abci/                          # ABCI 클라이언트
│   ├── client.go                  # gRPC 클라이언트
│   └── types.go                   # 타입 변환 헬퍼
│
├── transport/                     # 네트워크 통신
│   └── grpc.go                    # gRPC P2P 통신
│
├── api/pbft/v1/                   # Protobuf 생성 코드
│   ├── types.pb.go                # 메시지 타입
│   └── service.pb.go              # gRPC 서비스
│
└── types/                         # 공통 타입
    ├── block.go                   # 블록 타입
    ├── transaction.go             # 트랜잭션 타입
    └── validator.go               # 검증자 타입
```

### 2.2 각 파일의 역할

| 파일 | 역할 | 비유 |
|------|------|------|
| `node/node.go` | 모든 컴포넌트 조립, 시작/종료 | 회사 본사 |
| `node/config.go` | 설정 값 관리 | 회사 정관 |
| `engine_v2.go` | PBFT 합의 로직 실행 | 사장님 |
| `abci_adapter.go` | 타입 변환, 중간 계층 | 비서 |
| `abci/client.go` | gRPC 통신 | 배달부 |
| `abci/types.go` | 데이터 변환 함수 | 번역 사전 |
| `transport/grpc.go` | 노드 간 P2P 통신 | 전화기 |
| `api/pbft/v1/*.go` | Protobuf 메시지 정의 | 표준 양식 |

---

## 3. 계층별 상세 설명

### 3.1 Node 계층 (최상위)

```go
// node/node.go

type Node struct {
    config    *Config              // 설정
    engine    *pbft.Engine         // PBFT 엔진
    transport *transport.GRPCTransport  // P2P 통신
    metrics   *metrics.Metrics     // 메트릭
    running   bool                 // 실행 상태
}
```

**Node가 하는 일:**

```
1. 시작할 때:
   ┌─────────────────────────────────────────┐
   │  Transport 시작 (P2P 서버 열기)          │
   │          ↓                              │
   │  Peer 연결 (다른 노드들과 연결)           │
   │          ↓                              │
   │  Metrics 서버 시작 (선택)                │
   │          ↓                              │
   │  Engine 시작 (합의 시작)                 │
   └─────────────────────────────────────────┘

2. 실행 중:
   - 트랜잭션 제출 (SubmitTx)
   - 상태 조회 (GetHeight, GetView, IsPrimary)
   
3. 종료할 때:
   - Engine 정지
   - Transport 정지
   - Metrics 서버 정지
```

### 3.2 Config 계층 (설정)

```go
// node/config.go

type Config struct {
    // 노드 식별
    NodeID  string    // "node0", "node1" 등
    ChainID string    // "pbft-chain"
    
    // 네트워크 주소
    ListenAddr string // "0.0.0.0:26656" (P2P)
    ABCIAddr   string // "localhost:26658" (Cosmos SDK)
    
    // 피어 목록
    Peers []string    // ["node1@localhost:26657", ...]
    
    // 검증자 목록
    Validators []*pbft.ValidatorInfo
    
    // 타이밍
    RequestTimeout     time.Duration  // 5초
    ViewChangeTimeout  time.Duration  // 10초
    CheckpointInterval uint64         // 100블록마다
    WindowSize         uint64         // 200
}
```

**설정 값 설명:**

```
NodeID = "node0"
  → 이 노드의 이름/ID

ChainID = "pbft-chain"  
  → 블록체인 네트워크 이름

ListenAddr = "0.0.0.0:26656"
  → 다른 PBFT 노드가 연결할 주소

ABCIAddr = "localhost:26658"
  → Cosmos SDK 앱 주소

Peers = ["node1@localhost:26657", "node2@localhost:26658"]
  → 연결할 다른 노드들 (형식: "노드ID@주소")

Validators = [{ID: "node0", Power: 10}, ...]
  → 검증자 목록과 투표력

ViewChangeTimeout = 10초
  → 리더가 응답 없으면 10초 후 리더 교체
```

### 3.3 EngineV2 계층 (PBFT 합의)

```go
// consensus/pbft/engine_v2.go

type EngineV2 struct {
    config       *Config
    view         uint64              // 현재 뷰 (리더 번호)
    sequenceNum  uint64              // 현재 블록 높이
    
    validatorSet *types.ValidatorSet // 검증자 목록
    stateLog     *StateLog           // 상태 저장소
    
    transport    Transport           // P2P 통신
    abciAdapter  ABCIAdapterInterface // ABCI 어댑터
    
    msgChan      chan *Message       // 메시지 수신 채널
    requestChan  chan *RequestMsg    // 요청 수신 채널
    
    viewChangeManager *ViewChangeManager // 뷰 체인지 관리
}
```

**EngineV2가 하는 일:**

```
                    ┌─────────────────────────────────────┐
                    │           EngineV2                  │
                    └─────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
         ▼                          ▼                          ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  proposeBlock   │      │ handlePrePrepare│      │  executeBlock   │
│  (블록 제안)     │      │ (PrePrepare처리) │      │  (블록 실행)     │
│                 │      │                 │      │                 │
│  리더만 호출     │      │  팔로워가 처리   │      │  합의 완료 후    │
└─────────────────┘      └─────────────────┘      └─────────────────┘
         │                          │                          │
         ▼                          ▼                          ▼
   PrepareProposal            ProcessProposal            FinalizeBlock
   (앱에 트랜잭션 정리 요청)    (앱에 블록 검증 요청)       (앱에 블록 실행 요청)
```

### 3.4 ABCIAdapter 계층 (변환)

```go
// consensus/pbft/abci_adapter.go

type ABCIAdapter struct {
    client      *abciclient.Client  // ABCI 클라이언트
    lastHeight  int64               // 마지막 높이
    lastAppHash []byte              // 마지막 앱 해시
    maxTxBytes  int64               // 최대 트랜잭션 크기
    chainID     string              // 체인 ID
}
```

**ABCIAdapter가 하는 일:**

```
입력 (PBFT 타입)              변환               출력 (ABCI 타입)
─────────────────────────────────────────────────────────────────

types.Block          →    BlockData         →    RequestFinalizeBlock
  │                         │                        │
  ├─ Header.Height          ├─ Height                ├─ Height
  ├─ Transactions[]         ├─ Txs                   ├─ Txs
  ├─ Hash                   ├─ Hash                  ├─ Hash
  └─ Header.Timestamp       └─ Time                  └─ Time


메서드별 변환:

┌─────────────────────┬─────────────────────┬─────────────────────┐
│   EngineV2 호출     │   ABCIAdapter       │   Client 호출       │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ PrepareProposal     │ PBFT→ABCI 변환      │ client.Prepare...   │
│ (height, txs)       │ NewPrepareProposal  │ (Request)           │
│                     │ Request() 사용       │                     │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ ProcessProposal     │ PBFT→ABCI 변환      │ client.Process...   │
│ (height, txs, hash) │ NewProcessProposal  │ (Request)           │
│                     │ Request() 사용       │                     │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ FinalizeBlock       │ types.Block →       │ client.Finalize...  │
│ (block)             │ BlockData →         │ (Request)           │
│                     │ Request 변환        │                     │
└─────────────────────┴─────────────────────┴─────────────────────┘
```

### 3.5 ABCI Client 계층 (통신)

```go
// abci/client.go

type Client struct {
    conn    *grpc.ClientConn   // gRPC 연결
    client  abci.ABCIClient    // ABCI gRPC 클라이언트
    address string             // Cosmos SDK 앱 주소
    timeout time.Duration      // 타임아웃
}
```

**Client가 하는 일:**

```
ABCI 메서드 목록:

┌─────────────────────┬─────────────────────────────────────────┐
│      메서드          │              용도                       │
├─────────────────────┼─────────────────────────────────────────┤
│ InitChain           │ 체인 시작 시 초기화                      │
│ CheckTx             │ 트랜잭션 사전 검증 (멤풀 진입 전)         │
│ PrepareProposal     │ 블록 제안 준비 (트랜잭션 정렬)           │
│ ProcessProposal     │ 블록 제안 검증 (ACCEPT/REJECT)          │
│ FinalizeBlock       │ 블록 실행 (상태 변경)                    │
│ Commit              │ 상태 확정 (디스크 저장)                  │
│ Query               │ 상태 조회 (읽기 전용)                    │
└─────────────────────┴─────────────────────────────────────────┘

gRPC 통신 흐름:

┌─────────────┐     Request      ┌─────────────┐
│   Client    │ ───────────────▶ │ Cosmos SDK  │
│             │     (gRPC)       │     앱      │
│             │ ◀─────────────── │             │
└─────────────┘     Response     └─────────────┘
```

### 3.6 GRPCTransport 계층 (P2P)

```go
// transport/grpc.go (구현 필요)

type GRPCTransport struct {
    nodeID     string
    address    string
    server     *grpc.Server
    peers      map[string]*peerConn
    msgHandler func(*Message)
}
```

**Transport가 하는 일:**

```
노드 간 P2P 통신:

   Node0                    Node1                    Node2
     │                        │                        │
     │    PrePrepare          │                        │
     │ ──────────────────────▶│                        │
     │ ──────────────────────────────────────────────▶│
     │                        │                        │
     │        Prepare         │        Prepare         │
     │ ◀──────────────────────│                        │
     │ ◀──────────────────────────────────────────────│
     │                        │                        │
     │        Commit          │        Commit          │
     │ ──────────────────────▶│                        │
     │ ──────────────────────────────────────────────▶│
     │ ◀──────────────────────│                        │
     │ ◀──────────────────────────────────────────────│


PBFTService gRPC 메서드:

┌─────────────────────┬─────────────────────────────────────────┐
│      메서드          │              용도                       │
├─────────────────────┼─────────────────────────────────────────┤
│ BroadcastMessage    │ 모든 피어에게 메시지 전송                │
│ SendMessage         │ 특정 노드에게 메시지 전송                │
│ MessageStream       │ 양방향 스트림 (실시간 통신)              │
│ SyncState           │ 상태 동기화                             │
│ GetCheckpoint       │ 체크포인트 조회                         │
│ GetStatus           │ 노드 상태 조회                          │
└─────────────────────┴─────────────────────────────────────────┘
```

---

## 4. 데이터 흐름

### 4.1 타입 변환 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         타입 변환 흐름                                   │
└─────────────────────────────────────────────────────────────────────────┘

[PBFT 내부 타입]              [변환 중간]              [ABCI 타입]
─────────────────────────────────────────────────────────────────────────

types.Block                                        abci.RequestFinalizeBlock
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ Header:         │      │ BlockData:      │      │                 │
│   Height: 100   │  →   │   Height: 100   │  →   │ Height: 100     │
│   Timestamp     │      │   Time          │      │ Time            │
│   ProposerID    │      │   ProposerAddr  │      │ ProposerAddress │
│ Transactions[]  │      │   Txs           │      │ Txs             │
│ Hash            │      │   Hash          │      │ Hash            │
└─────────────────┘      └─────────────────┘      └─────────────────┘

코드로 보면:

// 1단계: PBFT Block에서 데이터 추출
txs := make([][]byte, len(block.Transactions))
for i, tx := range block.Transactions {
    txs[i] = tx.Data  // Transaction 구조체에서 raw bytes만 추출
}

// 2단계: 중간 타입으로 변환
blockData := &abciclient.BlockData{
    Height:       int64(block.Header.Height),
    Txs:          txs,
    Hash:         block.Hash,
    Time:         block.Header.Timestamp,
    ProposerAddr: []byte(block.Header.ProposerID),
}

// 3단계: ABCI Request 생성
req := abciclient.NewFinalizeBlockRequest(blockData)
// → abci.RequestFinalizeBlock 반환

// 4단계: gRPC로 전송
resp, err := a.client.FinalizeBlock(ctx, req)
```

### 4.2 메시지 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         메시지 전송 흐름                                 │
└─────────────────────────────────────────────────────────────────────────┘

1. Engine에서 메시지 생성:
   ┌─────────────────────────────────────────────────────────────────┐
   │  prePrepareMsg := NewPrePrepareMsg(view, seqNum, block, nodeID) │
   │  payload, _ := json.Marshal(prePrepareMsg)                      │
   │  msg := NewMessage(PrePrepare, view, seqNum, block.Hash, nodeID)│
   │  msg.Payload = payload                                          │
   └─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
2. Transport로 브로드캐스트:
   ┌─────────────────────────────────────────────────────────────────┐
   │  e.broadcast(msg)                                               │
   │    └─▶ e.transport.Broadcast(msg)                               │
   └─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
3. gRPC로 전송 (Protobuf 변환):
   ┌─────────────────────────────────────────────────────────────────┐
   │  pbft.Message (Go 구조체)                                       │
   │       │                                                         │
   │       ▼ Protobuf 직렬화                                         │
   │  []byte (바이너리)                                              │
   │       │                                                         │
   │       ▼ 네트워크 전송                                            │
   │  다른 노드 도착                                                  │
   │       │                                                         │
   │       ▼ Protobuf 역직렬화                                       │
   │  pbft.Message (Go 구조체)                                       │
   └─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
4. 수신 노드에서 처리:
   ┌─────────────────────────────────────────────────────────────────┐
   │  transport.SetMessageHandler(engine.handleIncomingMessage)      │
   │       │                                                         │
   │       ▼                                                         │
   │  e.msgChan <- msg  // 채널로 전달                                │
   │       │                                                         │
   │       ▼                                                         │
   │  e.handleMessage(msg)  // 메시지 타입별 처리                     │
   └─────────────────────────────────────────────────────────────────┘
```

---

## 5. PBFT 합의 과정

### 5.1 전체 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PBFT 합의 전체 흐름                              │
└─────────────────────────────────────────────────────────────────────────┘

[Phase 0: Request]
    사용자가 트랜잭션 제출
         │
         ▼
    ┌─────────────────┐
    │  node.SubmitTx  │  "송금 트랜잭션 제출"
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ engine.request  │  requestChan에 추가
    │    Chan <- req  │
    └────────┬────────┘
             │
             ▼
[Phase 1: Pre-Prepare] (리더만)
    ┌─────────────────────────────────────────────────────────────────┐
    │                       proposeBlock()                            │
    │                                                                 │
    │  1. abciAdapter.PrepareProposal() → 트랜잭션 정렬               │
    │  2. 블록 생성                                                   │
    │  3. PrePrepare 메시지 브로드캐스트                              │
    └─────────────────────────────────────────────────────────────────┘
             │
             │ PrePrepare 메시지
             ▼
[Phase 2: Prepare] (모든 노드)
    ┌─────────────────────────────────────────────────────────────────┐
    │                     handlePrePrepare()                          │
    │                                                                 │
    │  1. 리더인지 확인                                               │
    │  2. abciAdapter.ProcessProposal() → 블록 검증                   │
    │  3. ACCEPT이면 Prepare 메시지 브로드캐스트                       │
    └─────────────────────────────────────────────────────────────────┘
             │
             │ Prepare 메시지 (2f+1개 수집)
             ▼
[Phase 3: Commit] (모든 노드)
    ┌─────────────────────────────────────────────────────────────────┐
    │                      handlePrepare()                            │
    │                                                                 │
    │  1. Prepare 메시지 저장                                         │
    │  2. 2f+1개 이상이면 Commit 메시지 브로드캐스트                    │
    └─────────────────────────────────────────────────────────────────┘
             │
             │ Commit 메시지 (2f+1개 수집)
             ▼
[Phase 4: Execute]
    ┌─────────────────────────────────────────────────────────────────┐
    │                       handleCommit()                            │
    │                                                                 │
    │  1. Commit 메시지 저장                                          │
    │  2. 2f+1개 이상이면 executeBlock() 호출                          │
    └─────────────────────────────────────────────────────────────────┘
             │
             ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                       executeBlock()                            │
    │                                                                 │
    │  1. abciAdapter.FinalizeBlock() → 블록 실행                     │
    │  2. abciAdapter.Commit() → 상태 확정                            │
    │  3. handleValidatorUpdates() → 검증자 변경 처리                 │
    │  4. createCheckpoint() → 체크포인트 생성                        │
    └─────────────────────────────────────────────────────────────────┘
             │
             ▼
    블록 완료! 다음 블록으로...
```

### 5.2 4개 노드 시나리오

```
┌─────────────────────────────────────────────────────────────────────────┐
│              4개 노드 PBFT 합의 과정 (f=1, 2f+1=3)                       │
└─────────────────────────────────────────────────────────────────────────┘

시간 →

Node0          Node1          Node2          Node3
(리더)
  │
  │ ─── PrePrepare ──────────────────────────────────────────▶
  │                    │              │              │
  │                    ▼              ▼              ▼
  │              ProcessProposal  ProcessProposal  ProcessProposal
  │              (블록 검증)       (블록 검증)       (블록 검증)
  │                    │              │              │
  │ ◀─── Prepare ──────┘              │              │
  │ ◀─── Prepare ─────────────────────┘              │
  │ ◀─── Prepare ────────────────────────────────────┘
  │
  │  3개 수집 (2f+1 충족!)
  │
  │ ─── Commit ──────────────────────────────────────────────▶
  │ ◀─── Commit ───────┐              │              │
  │ ◀─── Commit ──────────────────────┘              │
  │ ◀─── Commit ─────────────────────────────────────┘
  │
  │  3개 수집 (2f+1 충족!)
  │
  ▼
executeBlock()         executeBlock()  executeBlock()  executeBlock()
FinalizeBlock          FinalizeBlock   FinalizeBlock   FinalizeBlock
Commit                 Commit          Commit          Commit

  │
  ▼
블록 1 완료!
```

---

## 6. 노드 시작부터 블록 실행까지

### 6.1 노드 시작

```go
// 1. Config 생성
config := &node.Config{
    NodeID:     "node0",
    ChainID:    "pbft-chain",
    ListenAddr: "0.0.0.0:26656",
    ABCIAddr:   "localhost:26658",
    Peers:      []string{"node1@localhost:26657", "node2@localhost:26658"},
    Validators: validators,
}

// 2. Node 생성
n, err := node.NewNodeWithABCI(config)
```

```
내부 동작:

NewNodeWithABCI(config)
         │
         ├─▶ transport.NewGRPCTransport()    // P2P 통신 준비
         │
         ├─▶ pbft.NewValidatorSet()          // 검증자 목록 생성
         │
         ├─▶ pbft.NewABCIAdapter()           // ABCI 어댑터 생성
         │         │
         │         └─▶ abciclient.NewClient()  // gRPC 연결
         │
         └─▶ pbft.NewEngineV2()              // PBFT 엔진 생성
                   │
                   └─▶ ViewChangeManager 초기화
                   └─▶ StateLog 초기화
                   └─▶ 채널 생성 (msgChan, requestChan)
```

### 6.2 노드 실행

```go
// 3. Node 시작
err := n.Start(ctx)
```

```
Start(ctx) 내부 동작:

┌─────────────────────────────────────────────────────────────────────────┐
│  1. Transport 시작                                                      │
│     └─▶ gRPC 서버 시작 (26656 포트)                                     │
│         "다른 노드의 연결을 기다림"                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  2. Peer 연결                                                           │
│     for peer in config.Peers:                                           │
│         transport.AddPeer(peerID, peerAddr)                             │
│         "node1@localhost:26657에 연결..."                               │
│         "node2@localhost:26658에 연결..."                               │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  3. Metrics 서버 시작 (선택)                                             │
│     go n.startMetricsServer()                                           │
│         "http://localhost:26660/metrics"                                │
│         "http://localhost:26660/health"                                 │
│         "http://localhost:26660/status"                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  4. Engine 시작                                                         │
│     e.Start()                                                           │
│         │                                                               │
│         ├─▶ go e.run()              // 메인 루프 시작                   │
│         │                                                               │
│         └─▶ e.resetViewChangeTimer() // 뷰체인지 타이머 시작             │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  5. 메인 루프 (run)                                                      │
│     for {                                                               │
│         select {                                                        │
│         case <-ctx.Done():                                              │
│             return  // 종료                                             │
│         case msg := <-msgChan:                                          │
│             handleMessage(msg)  // 메시지 처리                           │
│         case req := <-requestChan:                                      │
│             if isPrimary() {                                            │
│                 proposeBlock(req)  // 블록 제안 (리더만)                  │
│             }                                                           │
│         }                                                               │
│     }                                                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 트랜잭션 제출

```go
// 4. 트랜잭션 제출
err := n.SubmitTx([]byte("send 100 to Bob"), "client1")
```

```
SubmitTx() 흐름:

┌─────────────────┐
│  n.SubmitTx()   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  engine.SubmitRequest(operation, clientID)                              │
│                                                                         │
│  req := &RequestMsg{                                                    │
│      Operation: []byte("send 100 to Bob"),                              │
│      Timestamp: time.Now(),                                             │
│      ClientID:  "client1",                                              │
│  }                                                                      │
│                                                                         │
│  e.requestChan <- req  // 채널에 추가                                    │
└─────────────────────────────────────────────────────────────────────────┘
         │
         │  run() 루프에서 수신
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  case req := <-requestChan:                                             │
│      if e.isPrimary() {                                                 │
│          e.proposeBlock(req)  // 리더면 블록 제안!                        │
│      }                                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.4 블록 제안 (리더)

```go
// proposeBlock() 상세

func (e *EngineV2) proposeBlock(req *RequestMsg) {
    // 1. 시퀀스 번호 증가
    e.sequenceNum++
    
    // 2. 트랜잭션 수집
    txs := [][]byte{req.Operation}
    
    // 3. ABCI PrepareProposal 호출
    preparedTxs, err := e.abciAdapter.PrepareProposal(ctx, height, proposer, txs)
    
    // 4. 블록 생성
    block := types.NewBlock(seqNum, prevHash, nodeID, view, transactions)
    
    // 5. PrePrepare 메시지 생성
    prePrepareMsg := NewPrePrepareMsg(view, seqNum, block, nodeID)
    
    // 6. StateLog에 저장
    state.SetPrePrepare(prePrepareMsg, block)
    
    // 7. 브로드캐스트
    e.broadcast(msg)
}
```

```
PrepareProposal 호출 흐름:

e.abciAdapter.PrepareProposal(ctx, height, proposer, txs)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ABCIAdapter.PrepareProposal()                                          │
│                                                                         │
│  // 1. Request 생성                                                     │
│  req := abciclient.NewPrepareProposalRequest(txs, maxBytes, height, ...)│
│                                                                         │
│  // 2. ABCI 클라이언트 호출                                              │
│  resp, err := a.client.PrepareProposal(ctx, req)                        │
│                                                                         │
│  // 3. 결과 반환                                                        │
│  return resp.Txs, nil                                                   │
└─────────────────────────────────────────────────────────────────────────┘
         │
         │ gRPC
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Cosmos SDK 앱                                                          │
│                                                                         │
│  "트랜잭션 정렬하고 필터링해서 돌려줄게"                                   │
│  - 중복 제거                                                            │
│  - 우선순위 정렬                                                         │
│  - 크기 제한 적용                                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.5 블록 검증 (팔로워)

```go
// handlePrePrepare() 상세

func (e *EngineV2) handlePrePrepare(msg *Message) {
    // 1. 리더 확인
    if msg.NodeID != e.getPrimaryID() {
        return  // 리더가 아니면 무시
    }
    
    // 2. 뷰 확인
    if msg.View != currentView {
        return  // 현재 뷰가 아니면 무시
    }
    
    // 3. 메시지 디코딩
    var prePrepareMsg PrePrepareMsg
    json.Unmarshal(msg.Payload, &prePrepareMsg)
    
    // 4. ABCI ProcessProposal 호출
    accepted, err := e.abciAdapter.ProcessProposal(ctx, height, proposer, txs, hash)
    
    // 5. 거부되면 종료
    if !accepted {
        return
    }
    
    // 6. StateLog에 저장
    state.SetPrePrepare(&prePrepareMsg, block)
    
    // 7. Prepare 메시지 브로드캐스트
    e.broadcast(prepareMsg)
}
```

### 6.6 블록 실행

```go
// executeBlock() 상세

func (e *EngineV2) executeBlock(state *State) {
    // 1. FinalizeBlock 호출
    result, err := e.abciAdapter.FinalizeBlock(ctx, state.Block)
    
    // 2. 트랜잭션 결과 확인
    for i, txResult := range result.TxResults {
        if txResult.Code != 0 {
            log.Printf("Tx %d failed: %s", i, txResult.Log)
        }
    }
    
    // 3. Commit 호출
    appHash, retainHeight, err := e.abciAdapter.Commit(ctx)
    
    // 4. 상태 업데이트
    e.lastAppHash = result.AppHash
    state.MarkExecuted()
    e.committedBlocks = append(e.committedBlocks, state.Block)
    
    // 5. 검증자 업데이트 처리
    if len(result.ValidatorUpdates) > 0 {
        e.handleValidatorUpdates(result.ValidatorUpdates)
    }
    
    // 6. 체크포인트 생성
    if state.SequenceNum % checkpointInterval == 0 {
        e.createCheckpoint(state.SequenceNum)
    }
}
```

```
FinalizeBlock 호출 흐름:

e.abciAdapter.FinalizeBlock(ctx, state.Block)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ABCIAdapter.FinalizeBlock()                                            │
│                                                                         │
│  // 1. 트랜잭션 추출                                                    │
│  txs := make([][]byte, len(block.Transactions))                         │
│  for i, tx := range block.Transactions {                                │
│      txs[i] = tx.Data                                                   │
│  }                                                                      │
│                                                                         │
│  // 2. BlockData 생성                                                   │
│  blockData := &abciclient.BlockData{                                    │
│      Height: int64(block.Header.Height),                                │
│      Txs:    txs,                                                       │
│      Hash:   block.Hash,                                                │
│      Time:   block.Header.Timestamp,                                    │
│      ProposerAddr: []byte(block.Header.ProposerID),                     │
│  }                                                                      │
│                                                                         │
│  // 3. Request 생성                                                     │
│  req := abciclient.NewFinalizeBlockRequest(blockData)                   │
│                                                                         │
│  // 4. ABCI 클라이언트 호출                                              │
│  resp, err := a.client.FinalizeBlock(ctx, req)                          │
│                                                                         │
│  // 5. 결과 변환                                                        │
│  result := abciclient.FinalizeBlockResponseToResult(resp)               │
│                                                                         │
│  return &ABCIExecutionResult{                                           │
│      TxResults:        result.TxResults,                                │
│      ValidatorUpdates: result.ValidatorUpdates,                         │
│      AppHash:          result.AppHash,                                  │
│  }, nil                                                                 │
└─────────────────────────────────────────────────────────────────────────┘
         │
         │ gRPC
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Cosmos SDK 앱                                                          │
│                                                                         │
│  내부적으로:                                                             │
│  1. BeginBlock 로직 실행                                                │
│  2. 각 트랜잭션 실행 (상태 변경)                                          │
│  3. EndBlock 로직 실행                                                  │
│  4. 결과 반환 (TxResults, ValidatorUpdates, AppHash)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 메시지 타입과 처리

### 7.1 메시지 타입 (api/pbft/v1/types.pb.go)

```go
type MessageType int32

const (
    MessageType_MESSAGE_TYPE_UNSPECIFIED MessageType = 0
    MessageType_MESSAGE_TYPE_PRE_PREPARE MessageType = 1
    MessageType_MESSAGE_TYPE_PREPARE     MessageType = 2
    MessageType_MESSAGE_TYPE_COMMIT      MessageType = 3
    MessageType_MESSAGE_TYPE_VIEW_CHANGE MessageType = 4
    MessageType_MESSAGE_TYPE_NEW_VIEW    MessageType = 5
    MessageType_MESSAGE_TYPE_CHECKPOINT  MessageType = 6
)
```

### 7.2 각 메시지 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          PBFTMessage (공통)                             │
├─────────────────────────────────────────────────────────────────────────┤
│  Type        MessageType    // 메시지 종류                              │
│  View        uint64         // 뷰 번호                                  │
│  SequenceNum uint64         // 시퀀스 번호 (블록 높이)                   │
│  Digest      []byte         // 블록 해시                                │
│  NodeId      string         // 발신 노드 ID                             │
│  Timestamp   Timestamp      // 타임스탬프                               │
│  Signature   []byte         // 서명                                    │
│  Payload     []byte         // 실제 메시지 데이터 (JSON 인코딩)          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         PrePrepareMsg                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  View        uint64    // 뷰 번호                                       │
│  SequenceNum uint64    // 시퀀스 번호                                   │
│  Digest      []byte    // 블록 해시                                     │
│  BlockData   []byte    // 블록 데이터 (직렬화)                           │
│  PrimaryId   string    // 리더 ID                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         PrepareMsg / CommitMsg                          │
├─────────────────────────────────────────────────────────────────────────┤
│  View        uint64    // 뷰 번호                                       │
│  SequenceNum uint64    // 시퀀스 번호                                   │
│  Digest      []byte    // 블록 해시                                     │
│  NodeId      string    // 발신 노드 ID                                  │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         ViewChangeMsg                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  NewView     uint64          // 새 뷰 번호                              │
│  LastSeqNum  uint64          // 마지막 시퀀스                            │
│  Checkpoints []Checkpoint    // 체크포인트 목록                          │
│  PreparedSet []PreparedCert  // 준비된 블록 인증서                       │
│  NodeId      string          // 발신 노드 ID                            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                          NewViewMsg                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  View           uint64           // 새 뷰 번호                          │
│  ViewChangeMsgs []ViewChangeMsg  // 수집된 뷰체인지 메시지                │
│  PrePrepareMsgs []PrePrepareMsg  // 재처리할 PrePrepare                  │
│  NewPrimaryId   string           // 새 리더 ID                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.3 메시지 처리 흐름

```go
func (e *EngineV2) handleMessage(msg *Message) {
    switch msg.Type {
    case PrePrepare:
        e.handlePrePrepare(msg)
    case Prepare:
        e.handlePrepare(msg)
    case Commit:
        e.handleCommit(msg)
    case ViewChange:
        e.handleViewChange(msg)
    case NewView:
        e.handleNewView(msg)
    }
}
```

```
메시지 수신 → 처리 흐름:

네트워크에서 메시지 도착
         │
         ▼
transport.msgHandler(msg)  // Transport가 Engine에 전달
         │
         ▼
e.msgChan <- msg           // 채널에 추가
         │
         ▼
run() 루프에서 수신
         │
         ▼
case msg := <-msgChan:
    handleMessage(msg)
         │
         ▼
switch msg.Type:
    ┌─────────────────────────────────────────────────────────────────┐
    │  PrePrepare  →  handlePrePrepare()  →  ProcessProposal 호출    │
    │  Prepare     →  handlePrepare()     →  Commit 전송             │
    │  Commit      →  handleCommit()      →  executeBlock 호출       │
    │  ViewChange  →  handleViewChange()  →  뷰 체인지 처리          │
    │  NewView     →  handleNewView()     →  새 뷰 시작              │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 8. 핵심 코드 분석

### 8.1 리더 선출

```go
func (e *EngineV2) isPrimary() bool {
    primaryIdx := int(e.view) % len(e.validatorSet.Validators)
    return e.validatorSet.Validators[primaryIdx].ID == e.config.NodeID
}
```

```
view = 0: node0이 리더 (0 % 4 = 0)
view = 1: node1이 리더 (1 % 4 = 1)
view = 2: node2가 리더 (2 % 4 = 2)
view = 3: node3이 리더 (3 % 4 = 3)
view = 4: node0이 리더 (4 % 4 = 0)  ← 다시 순환
```

### 8.2 Quorum 계산

```go
func (vs *ValidatorSet) QuorumSize() int {
    n := len(vs.Validators)
    f := (n - 1) / 3  // 허용 가능한 Byzantine 노드 수
    return 2*f + 1    // 필요한 최소 투표 수
}
```

```
4개 노드: f = 1, quorum = 3
7개 노드: f = 2, quorum = 5
10개 노드: f = 3, quorum = 7
```

### 8.3 상태 전이

```go
// handlePrepare에서
if state.IsPrepared(quorum) && state.GetPhase() == PrePrepared {
    state.TransitionToPrepared()  // PrePrepared → Prepared
    e.broadcast(commitMsg)
}

// handleCommit에서
if state.IsCommitted(quorum) && state.GetPhase() == Prepared {
    state.TransitionToCommitted()  // Prepared → Committed
    e.executeBlock(state)
}
```

```
상태 전이 다이어그램:

   ┌─────────────┐   PrePrepare 수신   ┌─────────────┐
   │   Initial   │ ─────────────────▶  │ PrePrepared │
   └─────────────┘                     └──────┬──────┘
                                              │
                                              │ 2f+1 Prepare 수집
                                              ▼
                                       ┌─────────────┐
                                       │  Prepared   │
                                       └──────┬──────┘
                                              │
                                              │ 2f+1 Commit 수집
                                              ▼
                                       ┌─────────────┐
                                       │  Committed  │
                                       └──────┬──────┘
                                              │
                                              │ executeBlock()
                                              ▼
                                       ┌─────────────┐
                                       │  Executed   │
                                       └─────────────┘
```

### 8.4 검증자 업데이트

```go
func (e *EngineV2) handleValidatorUpdates(updates []abci.ValidatorUpdate) {
    for _, update := range updates {
        if update.Power == 0 {
            // 검증자 제거
            e.validatorSet.RemoveByPubKey(update.PubKey.Data)
        } else {
            // 검증자 추가/업데이트
            e.validatorSet.UpdateValidator(&types.Validator{
                ID:        string(update.PubKey.Data),
                PublicKey: update.PubKey.Data,
                Power:     update.Power,
            })
        }
    }
    
    // Quorum 크기 재계산
    e.viewChangeManager.UpdateQuorumSize(e.validatorSet.QuorumSize())
}
```

---

## 9. 요약 정리

### 9.1 계층 구조 요약

```
┌─────────────────────────────────────────────────────────────────────────┐
│  계층          │  파일                  │  역할                         │
├─────────────────────────────────────────────────────────────────────────┤
│  Node          │  node/node.go          │  모든 컴포넌트 조립, 시작/종료  │
│  Config        │  node/config.go        │  설정 값 관리                  │
│  EngineV2      │  pbft/engine_v2.go     │  PBFT 합의 로직                │
│  ABCIAdapter   │  pbft/abci_adapter.go  │  타입 변환, 중간 계층          │
│  ABCI Client   │  abci/client.go        │  gRPC 통신 (Cosmos SDK)        │
│  Types         │  abci/types.go         │  헬퍼 함수, 데이터 변환        │
│  Transport     │  transport/grpc.go     │  P2P 통신 (노드 간)            │
│  Protobuf      │  api/pbft/v1/*.go      │  메시지 정의                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 ABCI 메서드 매핑

```
┌─────────────────────────────────────────────────────────────────────────┐
│  기존 (Application)      │  새 코드 (ABCI)        │  호출 시점           │
├─────────────────────────────────────────────────────────────────────────┤
│  GetPendingTransactions  │  PrepareProposal       │  블록 제안 시 (리더)  │
│  ValidateBlock           │  ProcessProposal       │  PrePrepare 수신 시  │
│  ExecuteBlock            │  FinalizeBlock         │  Commit 완료 후      │
│  Commit                  │  Commit                │  블록 확정 시        │
│  (없음)                  │  CheckTx               │  멤풀 진입 전        │
│  (없음)                  │  InitChain             │  체인 시작 시        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 전체 데이터 흐름

```
사용자 요청 (트랜잭션)
       │
       ▼
┌─────────────────┐
│     Node        │  SubmitTx()
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   EngineV2      │  proposeBlock() / handlePrePrepare() / executeBlock()
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ABCIAdapter    │  PrepareProposal() / ProcessProposal() / FinalizeBlock()
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ABCI Client    │  gRPC 호출
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Cosmos SDK 앱  │  트랜잭션 실행, 상태 변경
└─────────────────┘
```

### 9.4 핵심 포인트

```
1. PBFT 합의 로직은 변경 없음
   - PrePrepare → Prepare → Commit 동일
   - 2f+1 투표 로직 동일
   - View Change 동일

2. 변경된 것은 앱과 통신하는 방식
   - 기존: 직접 함수 호출 (app.XXX())
   - 새것: gRPC로 네트워크 요청 (abciAdapter.XXX())

3. 계층 분리
   - EngineV2: 합의만 담당
   - ABCIAdapter: 변환만 담당
   - Client: 통신만 담당
   
4. 장점
   - 표준 ABCI 인터페이스 사용 → 어떤 Cosmos SDK 앱이든 연결 가능
   - 계층 분리 → 테스트 용이 (Mock 사용 가능)
   - 코드 가독성 향상
```

---

## 끝

이 문서가 PBFT-Cosmos 코드를 이해하는 데 도움이 되길 바랍니다!

질문이 있으면 언제든 물어보세요. 🚀
