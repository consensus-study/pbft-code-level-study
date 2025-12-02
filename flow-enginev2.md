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
  │ ─── PrePrepare ───▶│              │              │
  │ ─── PrePrepare ───────────────────▶              │
  │ ─── PrePrepare ───────────────────────────────────▶
  │                    │              │              │
  │                    ▼              ▼              ▼
  │              ProcessProposal  ProcessProposal  ProcessProposal
  │              (블록 검증)       (블록 검증)       (블록 검증)
  │                    │              │              │
  │                    │              │              │
  ├─── Prepare ───────▶│              │              │  ┐
  ├─── Prepare ───────────────────────▶              │  │ Node0도
  ├─── Prepare ───────────────────────────────────────▶  ┘ 브로드캐스트
  │ ◀── Prepare ───────┤──────────────▶──────────────▶  ← Node1 브로드캐스트
  │ ◀── Prepare ───────────────────────┤─────────────▶  ← Node2 브로드캐스트
  │ ◀── Prepare ───────────────────────────────────────┤ ← Node3 브로드캐스트
  │                    │              │              │
  │  (모든 노드가 Prepare 3개 이상 수집 → 2f+1 충족!)   │
  │                    │              │              │
  ├─── Commit ────────▶│              │              │  ┐
  ├─── Commit ────────────────────────▶              │  │ Node0
  ├─── Commit ────────────────────────────────────────▶  ┘ 브로드캐스트
  │ ◀── Commit ────────┤──────────────▶──────────────▶  ← Node1 브로드캐스트
  │ ◀── Commit ────────────────────────┤─────────────▶  ← Node2 브로드캐스트
  │ ◀── Commit ────────────────────────────────────────┤ ← Node3 브로드캐스트
  │                    │              │              │
  │  (모든 노드가 Commit 3개 이상 수집 → 2f+1 충족!)    │
  │                    │              │              │
  ▼                    ▼              ▼              ▼
executeBlock()    executeBlock()  executeBlock()  executeBlock()
FinalizeBlock     FinalizeBlock   FinalizeBlock   FinalizeBlock
Commit            Commit          Commit          Commit
  │                    │              │              │
  ▼                    ▼              ▼              ▼
                    블록 1 완료!
```

**메시지 발신 정리:**

| 메시지 | 발신자 | 수신자 | 비고 |
|--------|--------|--------|------|
| PrePrepare | 리더(Node0)만 | 모든 노드 | 리더만 보냄 |
| Prepare | **모든 노드** | **모든 노드** | 각자 브로드캐스트 |
| Commit | **모든 노드** | **모든 노드** | 각자 브로드캐스트 |

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

### 8.1 EngineV2 구조체 상세

```go
// consensus/pbft/engine_v2.go

type EngineV2 struct {
    // 설정
    config  *Config
    chainID string
    
    // 상태
    view        uint64  // 현재 뷰 (리더 번호)
    sequenceNum uint64  // 현재 시퀀스 (= 블록 높이)
    lastAppHash []byte  // 마지막 앱 해시
    
    // 검증자
    validatorSet *types.ValidatorSet
    
    // 상태 저장소
    stateLog *StateLog  // 각 시퀀스별 상태 저장
    
    // 통신
    transport   Transport            // P2P 통신
    abciAdapter ABCIAdapterInterface // ABCI 어댑터
    
    // 채널
    msgChan     chan *Message     // 메시지 수신
    requestChan chan *RequestMsg  // 트랜잭션 수신
    
    // 뷰 체인지
    viewChangeManager *ViewChangeManager
    viewChangeTimer   *time.Timer
    
    // 블록 저장
    committedBlocks []*types.Block
    
    // 동기화
    mu      sync.RWMutex
    running bool
    done    chan struct{}
}
```

**각 필드 설명:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  필드              │  타입                    │  역할                    │
├─────────────────────────────────────────────────────────────────────────┤
│  view              │  uint64                  │  현재 리더 번호          │
│  sequenceNum       │  uint64                  │  다음 블록 높이          │
│  lastAppHash       │  []byte                  │  앱 상태 해시            │
│  validatorSet      │  *ValidatorSet           │  검증자 목록             │
│  stateLog          │  *StateLog               │  시퀀스별 합의 상태      │
│  transport         │  Transport               │  P2P 메시지 송수신       │
│  abciAdapter       │  ABCIAdapterInterface    │  Cosmos SDK 연결         │
│  msgChan           │  chan *Message           │  받은 메시지 큐          │
│  requestChan       │  chan *RequestMsg        │  받은 트랜잭션 큐        │
│  viewChangeManager │  *ViewChangeManager      │  뷰 체인지 관리          │
│  viewChangeTimer   │  *time.Timer             │  뷰 체인지 타이머        │
│  committedBlocks   │  []*Block                │  완료된 블록들           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 ABCIAdapterInterface

```go
// consensus/pbft/abci_adapter.go

type ABCIAdapterInterface interface {
    // 생명주기
    Close() error
    
    // 체인 초기화
    InitChain(ctx context.Context, chainID string, validators []*types.Validator, appState []byte) error
    
    // 블록 제안 (리더용)
    PrepareProposal(ctx context.Context, height int64, proposer []byte, txs [][]byte) ([][]byte, error)
    
    // 블록 검증 (팔로워용)
    ProcessProposal(ctx context.Context, height int64, proposer []byte, txs [][]byte, hash []byte) (bool, error)
    
    // 블록 실행
    FinalizeBlock(ctx context.Context, block *types.Block) (*ABCIExecutionResult, error)
    
    // 상태 확정
    Commit(ctx context.Context) (appHash []byte, retainHeight int64, err error)
    
    // 트랜잭션 검증
    CheckTx(ctx context.Context, tx []byte) (*abci.ResponseCheckTx, error)
    
    // 상태 조회/설정
    GetLastAppHash() []byte
    GetLastHeight() int64
    SetLastAppHash(hash []byte)
}
```

**메서드별 호출 시점:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  메서드            │  호출 시점                   │  호출자              │
├─────────────────────────────────────────────────────────────────────────┤
│  InitChain         │  노드 최초 시작 시            │  Node.Start()        │
│  PrepareProposal   │  블록 제안 시                │  proposeBlock()      │
│  ProcessProposal   │  PrePrepare 수신 시          │  handlePrePrepare()  │
│  FinalizeBlock     │  Commit 완료 후              │  executeBlock()      │
│  Commit            │  FinalizeBlock 후            │  executeBlock()      │
│  CheckTx           │  트랜잭션 수신 시 (선택)      │  SubmitRequest()     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.3 리더 선출 로직

```go
// 현재 노드가 리더인지 확인
func (e *EngineV2) isPrimary() bool {
    primaryIdx := int(e.view) % len(e.validatorSet.Validators)
    return e.validatorSet.Validators[primaryIdx].ID == e.config.NodeID
}

// 현재 리더 ID 반환
func (e *EngineV2) getPrimaryID() string {
    primaryIdx := int(e.view) % len(e.validatorSet.Validators)
    return e.validatorSet.Validators[primaryIdx].ID
}
```

**동작 예시:**

```
Validators = ["node0", "node1", "node2", "node3"]  (4개)

view = 0: primaryIdx = 0 % 4 = 0 → node0이 리더
view = 1: primaryIdx = 1 % 4 = 1 → node1이 리더
view = 2: primaryIdx = 2 % 4 = 2 → node2가 리더
view = 3: primaryIdx = 3 % 4 = 3 → node3이 리더
view = 4: primaryIdx = 4 % 4 = 0 → node0이 리더 (순환)
view = 5: primaryIdx = 5 % 4 = 1 → node1이 리더

┌─────────────────────────────────────────────────────────────────────────┐
│  뷰가 바뀌면 리더가 바뀜 (Round-Robin 방식)                               │
│  뷰 체인지는 리더가 응답 없을 때 발생                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.4 Quorum 계산

```go
// types/validator.go

func (vs *ValidatorSet) QuorumSize() int {
    n := len(vs.Validators)
    f := (n - 1) / 3  // 허용 가능한 Byzantine 노드 수
    return 2*f + 1    // 필요한 최소 투표 수
}

// f 계산: 3f + 1 <= n 을 만족하는 최대 f
// 4개 노드: 3f + 1 <= 4 → f <= 1 → f = 1
// 7개 노드: 3f + 1 <= 7 → f <= 2 → f = 2
```

**Quorum 테이블:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  노드 수 (n)  │  Byzantine (f)  │  Quorum (2f+1)  │  안전 조건          │
├─────────────────────────────────────────────────────────────────────────┤
│      4        │       1         │       3         │  1개 악의적 허용     │
│      5        │       1         │       3         │  1개 악의적 허용     │
│      6        │       1         │       3         │  1개 악의적 허용     │
│      7        │       2         │       5         │  2개 악의적 허용     │
│     10        │       3         │       7         │  3개 악의적 허용     │
│     13        │       4         │       9         │  4개 악의적 허용     │
└─────────────────────────────────────────────────────────────────────────┘

PBFT 공식: n >= 3f + 1 (최소 노드 수)
```

### 8.5 메인 루프 (run)

```go
// consensus/pbft/engine_v2.go

func (e *EngineV2) run() {
    for {
        select {
        case <-e.done:
            // 종료 신호
            return
            
        case msg := <-e.msgChan:
            // 다른 노드로부터 메시지 수신
            e.handleMessage(msg)
            
        case req := <-e.requestChan:
            // 클라이언트로부터 트랜잭션 수신
            if e.isPrimary() {
                // 리더만 블록 제안
                e.proposeBlock(req)
            }
            // 팔로워는 무시 (리더가 제안할 때까지 대기)
            
        case <-e.viewChangeTimer.C:
            // 타임아웃 발생 → 뷰 체인지 시작
            e.startViewChange()
        }
    }
}
```

**메인 루프 흐름도:**

```
                    ┌─────────────────┐
                    │   run() 시작    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
              ┌────▶│  select 대기    │◀────┐
              │     └────────┬────────┘     │
              │              │              │
              │    ┌─────────┼─────────┐    │
              │    │         │         │    │
              │    ▼         ▼         ▼    │
              │ ┌─────┐  ┌─────┐  ┌─────┐   │
              │ │done │  │msg  │  │req  │   │
              │ │Chan │  │Chan │  │Chan │   │
              │ └──┬──┘  └──┬──┘  └──┬──┘   │
              │    │        │        │      │
              │    ▼        ▼        ▼      │
              │ return   handle   propose   │
              │          Message  Block     │
              │            │        │       │
              └────────────┴────────┴───────┘
```

### 8.6 proposeBlock (블록 제안)

```go
func (e *EngineV2) proposeBlock(req *RequestMsg) {
    e.mu.Lock()
    defer e.mu.Unlock()
    
    // 1. 시퀀스 번호 증가
    e.sequenceNum++
    seqNum := e.sequenceNum
    
    // 2. 트랜잭션 수집
    txs := [][]byte{req.Operation}
    // 실제로는 여러 트랜잭션을 모아서 배치 처리
    
    // 3. ABCI PrepareProposal 호출
    //    앱이 트랜잭션 정렬/필터링
    ctx := context.Background()
    height := int64(seqNum)
    proposer := []byte(e.config.NodeID)
    
    preparedTxs, err := e.abciAdapter.PrepareProposal(ctx, height, proposer, txs)
    if err != nil {
        log.Printf("PrepareProposal failed: %v", err)
        return
    }
    
    // 4. 트랜잭션을 Transaction 타입으로 변환
    transactions := make([]*types.Transaction, len(preparedTxs))
    for i, tx := range preparedTxs {
        transactions[i] = &types.Transaction{
            ID:        fmt.Sprintf("tx-%d-%d", seqNum, i),
            Data:      tx,
            Timestamp: time.Now(),
        }
    }
    
    // 5. 블록 생성
    prevHash := e.lastAppHash
    if prevHash == nil {
        prevHash = []byte{}
    }
    
    block := types.NewBlock(seqNum, prevHash, e.config.NodeID, e.view, transactions)
    
    // 6. PrePrepare 메시지 생성
    prePrepareMsg := &PrePrepareMsg{
        View:        e.view,
        SequenceNum: seqNum,
        Digest:      block.Hash,
        Block:       block,
        PrimaryID:   e.config.NodeID,
    }
    
    // 7. StateLog에 저장
    state := e.stateLog.GetOrCreate(seqNum, e.view)
    state.SetPrePrepare(prePrepareMsg, block)
    state.TransitionToPrePrepared()
    
    // 8. 메시지 직렬화
    payload, err := json.Marshal(prePrepareMsg)
    if err != nil {
        log.Printf("Failed to marshal PrePrepare: %v", err)
        return
    }
    
    msg := &Message{
        Type:        PrePrepare,
        View:        e.view,
        SequenceNum: seqNum,
        Digest:      block.Hash,
        NodeID:      e.config.NodeID,
        Timestamp:   time.Now(),
        Payload:     payload,
    }
    
    // 9. 서명 (선택적)
    // msg.Signature = e.sign(msg)
    
    // 10. 브로드캐스트
    e.broadcast(msg)
    
    // 11. 자신의 Prepare도 전송
    e.sendPrepare(seqNum, block.Hash)
    
    // 12. 타이머 리셋
    e.resetViewChangeTimer()
    
    log.Printf("[%s] Proposed block %d with %d txs", e.config.NodeID, seqNum, len(transactions))
}
```

**proposeBlock 흐름도:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          proposeBlock()                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 1. seqNum++     │    │ 2. txs 수집     │    │ 3. PrepareProposal│
│                 │    │                 │    │    (ABCI 호출)    │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │ 4. Transaction 변환 │
                    └────────┬────────────┘
                             │
                             ▼
                    ┌─────────────────────┐
                    │ 5. Block 생성       │
                    │  NewBlock(...)      │
                    └────────┬────────────┘
                             │
                             ▼
                    ┌─────────────────────┐
                    │ 6. PrePrepareMsg    │
                    │    생성             │
                    └────────┬────────────┘
                             │
                             ▼
                    ┌─────────────────────┐
                    │ 7. StateLog 저장    │
                    │  state.SetPrePrepare│
                    └────────┬────────────┘
                             │
                             ▼
                    ┌─────────────────────┐
                    │ 8-9. 직렬화 & 서명  │
                    └────────┬────────────┘
                             │
                             ▼
                    ┌─────────────────────┐
                    │ 10. broadcast()     │
                    │  → 모든 노드에 전송  │
                    └────────┬────────────┘
                             │
                             ▼
                    ┌─────────────────────┐
                    │ 11. sendPrepare()   │
                    │  → 자신의 Prepare   │
                    └─────────────────────┘
```

### 8.7 handlePrePrepare (PrePrepare 처리)

```go
func (e *EngineV2) handlePrePrepare(msg *Message) {
    e.mu.Lock()
    defer e.mu.Unlock()
    
    // 1. 뷰 확인
    if msg.View != e.view {
        log.Printf("PrePrepare view mismatch: got %d, expected %d", msg.View, e.view)
        return
    }
    
    // 2. 리더 확인 (PrePrepare는 리더만 보낼 수 있음)
    if msg.NodeID != e.getPrimaryID() {
        log.Printf("PrePrepare from non-primary: %s", msg.NodeID)
        return
    }
    
    // 3. 이미 처리한 시퀀스인지 확인
    if msg.SequenceNum <= e.getLastCommittedSeqNum() {
        log.Printf("PrePrepare for already committed sequence: %d", msg.SequenceNum)
        return
    }
    
    // 4. Payload 디코딩
    var prePrepareMsg PrePrepareMsg
    if err := json.Unmarshal(msg.Payload, &prePrepareMsg); err != nil {
        log.Printf("Failed to unmarshal PrePrepare: %v", err)
        return
    }
    
    block := prePrepareMsg.Block
    
    // 5. 블록 해시 검증
    if !bytes.Equal(block.Hash, msg.Digest) {
        log.Printf("Block hash mismatch")
        return
    }
    
    // 6. ABCI ProcessProposal 호출 (블록 검증)
    ctx := context.Background()
    height := int64(msg.SequenceNum)
    proposer := []byte(msg.NodeID)
    
    txs := make([][]byte, len(block.Transactions))
    for i, tx := range block.Transactions {
        txs[i] = tx.Data
    }
    
    accepted, err := e.abciAdapter.ProcessProposal(ctx, height, proposer, txs, block.Hash)
    if err != nil {
        log.Printf("ProcessProposal failed: %v", err)
        return
    }
    
    // 7. 거부되면 종료
    if !accepted {
        log.Printf("Block rejected by ProcessProposal")
        return
    }
    
    // 8. StateLog에 저장
    state := e.stateLog.GetOrCreate(msg.SequenceNum, msg.View)
    state.SetPrePrepare(&prePrepareMsg, block)
    state.TransitionToPrePrepared()
    
    // 9. Prepare 메시지 브로드캐스트
    e.sendPrepare(msg.SequenceNum, block.Hash)
    
    // 10. 타이머 리셋
    e.resetViewChangeTimer()
    
    log.Printf("[%s] Accepted PrePrepare for block %d", e.config.NodeID, msg.SequenceNum)
}
```

**handlePrePrepare 검증 단계:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     handlePrePrepare() 검증                              │
└─────────────────────────────────────────────────────────────────────────┘

메시지 수신
    │
    ▼
┌─────────────────┐     실패
│ 1. 뷰 확인      │────────────▶ return (무시)
│ msg.View == e.view?           "뷰가 다름"
└────────┬────────┘
         │ 성공
         ▼
┌─────────────────┐     실패
│ 2. 리더 확인    │────────────▶ return (무시)
│ msg.NodeID == primary?        "리더가 아님"
└────────┬────────┘
         │ 성공
         ▼
┌─────────────────┐     실패
│ 3. 시퀀스 확인  │────────────▶ return (무시)
│ seqNum > lastCommitted?       "이미 커밋됨"
└────────┬────────┘
         │ 성공
         ▼
┌─────────────────┐     실패
│ 4. 디코딩       │────────────▶ return (무시)
│ json.Unmarshal()              "파싱 실패"
└────────┬────────┘
         │ 성공
         ▼
┌─────────────────┐     실패
│ 5. 해시 검증    │────────────▶ return (무시)
│ block.Hash == digest?         "해시 불일치"
└────────┬────────┘
         │ 성공
         ▼
┌─────────────────┐     REJECT
│ 6. ProcessProposal│───────────▶ return (무시)
│ ABCI 블록 검증    │           "앱이 거부"
└────────┬────────┘
         │ ACCEPT
         ▼
┌─────────────────┐
│ 7. 저장 & Prepare│
│    브로드캐스트  │
└─────────────────┘
```

### 8.8 handlePrepare (Prepare 처리)

```go
func (e *EngineV2) handlePrepare(msg *Message) {
    e.mu.Lock()
    defer e.mu.Unlock()
    
    // 1. 뷰 확인
    if msg.View != e.view {
        return
    }
    
    // 2. 상태 가져오기
    state := e.stateLog.Get(msg.SequenceNum, msg.View)
    if state == nil {
        // PrePrepare를 아직 못 받음 → 나중에 처리하도록 버퍼에 저장
        e.bufferMessage(msg)
        return
    }
    
    // 3. Digest 확인 (같은 블록에 대한 Prepare인지)
    if !bytes.Equal(msg.Digest, state.GetDigest()) {
        log.Printf("Prepare digest mismatch")
        return
    }
    
    // 4. Prepare 메시지 저장
    prepareMsg := &PrepareMsg{
        View:        msg.View,
        SequenceNum: msg.SequenceNum,
        Digest:      msg.Digest,
        NodeID:      msg.NodeID,
    }
    state.AddPrepare(prepareMsg)
    
    // 5. Quorum 확인 (2f+1개 이상 모았나?)
    quorum := e.validatorSet.QuorumSize()
    
    if state.PrepareCount() >= quorum && state.GetPhase() == PrePrepared {
        // 6. Prepared 상태로 전이
        state.TransitionToPrepared()
        
        // 7. Commit 메시지 브로드캐스트
        e.sendCommit(msg.SequenceNum, msg.Digest)
        
        log.Printf("[%s] Block %d prepared with %d prepares", 
            e.config.NodeID, msg.SequenceNum, state.PrepareCount())
    }
}
```

**Prepare 수집 과정:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Prepare 메시지 수집                                 │
└─────────────────────────────────────────────────────────────────────────┘

4개 노드, quorum = 3

시간 →

State.Prepares = []

Node1의 Prepare 도착 → State.Prepares = [node1]  (1개)
                       PrepareCount() = 1 < 3 (quorum)
                       → 아직 부족, 대기

Node2의 Prepare 도착 → State.Prepares = [node1, node2]  (2개)
                       PrepareCount() = 2 < 3 (quorum)
                       → 아직 부족, 대기

Node0의 Prepare 도착 → State.Prepares = [node1, node2, node0]  (3개)
                       PrepareCount() = 3 >= 3 (quorum)
                       → Quorum 충족! Commit 전송!

Node3의 Prepare 도착 → State.Prepares = [node1, node2, node0, node3]  (4개)
                       이미 Prepared 상태 → 무시 (이미 Commit 보냄)
```

### 8.9 handleCommit (Commit 처리)

```go
func (e *EngineV2) handleCommit(msg *Message) {
    e.mu.Lock()
    defer e.mu.Unlock()
    
    // 1. 뷰 확인
    if msg.View != e.view {
        return
    }
    
    // 2. 상태 가져오기
    state := e.stateLog.Get(msg.SequenceNum, msg.View)
    if state == nil {
        e.bufferMessage(msg)
        return
    }
    
    // 3. Digest 확인
    if !bytes.Equal(msg.Digest, state.GetDigest()) {
        return
    }
    
    // 4. Commit 메시지 저장
    commitMsg := &CommitMsg{
        View:        msg.View,
        SequenceNum: msg.SequenceNum,
        Digest:      msg.Digest,
        NodeID:      msg.NodeID,
    }
    state.AddCommit(commitMsg)
    
    // 5. Quorum 확인
    quorum := e.validatorSet.QuorumSize()
    
    if state.CommitCount() >= quorum && state.GetPhase() == Prepared {
        // 6. Committed 상태로 전이
        state.TransitionToCommitted()
        
        // 7. 블록 실행!
        e.executeBlock(state)
        
        log.Printf("[%s] Block %d committed with %d commits",
            e.config.NodeID, msg.SequenceNum, state.CommitCount())
    }
}
```

### 8.10 executeBlock (블록 실행)

```go
func (e *EngineV2) executeBlock(state *State) {
    block := state.Block
    
    // 1. ABCI FinalizeBlock 호출
    ctx := context.Background()
    result, err := e.abciAdapter.FinalizeBlock(ctx, block)
    if err != nil {
        log.Printf("FinalizeBlock failed: %v", err)
        return
    }
    
    // 2. 트랜잭션 결과 확인
    for i, txResult := range result.TxResults {
        if txResult.Code != 0 {
            // 트랜잭션 실패 (Code != 0은 에러)
            log.Printf("Tx %d failed: code=%d, log=%s", i, txResult.Code, txResult.Log)
        }
    }
    
    // 3. ABCI Commit 호출 (상태 확정)
    appHash, retainHeight, err := e.abciAdapter.Commit(ctx)
    if err != nil {
        log.Printf("Commit failed: %v", err)
        return
    }
    
    // 4. 상태 업데이트
    e.lastAppHash = result.AppHash
    state.MarkExecuted()
    e.committedBlocks = append(e.committedBlocks, block)
    
    // 5. 검증자 업데이트 처리
    if len(result.ValidatorUpdates) > 0 {
        e.handleValidatorUpdates(result.ValidatorUpdates)
    }
    
    // 6. 체크포인트 생성 (주기적)
    if state.SequenceNum % e.config.CheckpointInterval == 0 {
        e.createCheckpoint(state.SequenceNum)
    }
    
    // 7. 다음 블록 준비
    e.resetViewChangeTimer()
    
    log.Printf("[%s] Executed block %d, appHash=%x, retainHeight=%d",
        e.config.NodeID, state.SequenceNum, appHash, retainHeight)
}
```

**executeBlock 흐름도:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           executeBlock()                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │ 1. FinalizeBlock 호출   │
                    │    (블록 실행)           │
                    │                         │
                    │  abciAdapter.Finalize   │
                    │  Block(ctx, block)      │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ 2. 트랜잭션 결과 확인   │
                    │                         │
                    │  for tx in TxResults:   │
                    │    if tx.Code != 0:     │
                    │      log("failed")      │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ 3. Commit 호출          │
                    │    (상태 확정)           │
                    │                         │
                    │  abciAdapter.Commit()   │
                    │  → appHash, retainHeight│
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ 4. 상태 업데이트        │
                    │                         │
                    │  e.lastAppHash = ...    │
                    │  committedBlocks.append │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ 5. 검증자 업데이트      │
                    │                         │
                    │  handleValidatorUpdates │
                    │  (Power 변경 처리)      │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ 6. 체크포인트 생성      │
                    │    (100블록마다)        │
                    └─────────────────────────┘
```

### 8.11 ABCIAdapter 상세

```go
// consensus/pbft/abci_adapter.go

type ABCIAdapter struct {
    client      *abciclient.Client
    lastHeight  int64
    lastAppHash []byte
    maxTxBytes  int64
    chainID     string
    mu          sync.RWMutex
}

// 생성자
func NewABCIAdapter(abciAddress string, chainID string) (*ABCIAdapter, error) {
    config := &abciclient.Config{
        Address: abciAddress,
        Timeout: 10 * time.Second,
    }
    
    client, err := abciclient.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("failed to create ABCI client: %w", err)
    }
    
    return &ABCIAdapter{
        client:     client,
        maxTxBytes: 1024 * 1024,  // 1MB
        chainID:    chainID,
    }, nil
}
```

### 8.12 ABCIAdapter.FinalizeBlock 상세

```go
func (a *ABCIAdapter) FinalizeBlock(ctx context.Context, block *types.Block) (*ABCIExecutionResult, error) {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    // 1. 트랜잭션 추출 (Transaction 구조체 → raw bytes)
    txs := make([][]byte, len(block.Transactions))
    for i, tx := range block.Transactions {
        txs[i] = tx.Data
    }
    
    // 2. BlockData 생성 (PBFT 타입 → 중간 타입)
    blockData := &abciclient.BlockData{
        Height:       int64(block.Header.Height),
        Txs:          txs,
        Hash:         block.Hash,
        Time:         block.Header.Timestamp,
        ProposerAddr: []byte(block.Header.ProposerID),
    }
    
    // 3. ABCI Request 생성 (중간 타입 → ABCI 타입)
    req := abciclient.NewFinalizeBlockRequest(blockData)
    
    // 4. gRPC 호출
    resp, err := a.client.FinalizeBlock(ctx, req)
    if err != nil {
        return nil, fmt.Errorf("FinalizeBlock failed: %w", err)
    }
    
    // 5. Response → Result 변환
    result := abciclient.FinalizeBlockResponseToResult(resp)
    
    // 6. 상태 업데이트
    a.lastHeight = int64(block.Header.Height)
    a.lastAppHash = result.AppHash
    
    // 7. 결과 반환
    return &ABCIExecutionResult{
        TxResults:        convertTxResults(result.TxResults),
        ValidatorUpdates: result.ValidatorUpdates,
        AppHash:          result.AppHash,
        Events:           result.Events,
    }, nil
}
```

**타입 변환 과정:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FinalizeBlock 타입 변환                               │
└─────────────────────────────────────────────────────────────────────────┘

[PBFT 타입]                    [중간 타입]                 [ABCI 타입]
types.Block                    BlockData                   RequestFinalizeBlock
┌─────────────────┐           ┌─────────────────┐         ┌─────────────────┐
│ Header:         │           │                 │         │                 │
│   Height: 100   │ ────────▶ │ Height: 100     │ ──────▶ │ Height: 100     │
│   Timestamp     │           │ Time            │         │ Time            │
│   ProposerID    │           │ ProposerAddr    │         │ ProposerAddress │
│ Transactions:   │           │ Txs: [][]byte   │         │ Txs             │
│   []*Transaction│           │                 │         │                 │
│ Hash            │           │ Hash            │         │ Hash            │
└─────────────────┘           └─────────────────┘         └─────────────────┘

Transaction 변환:
┌─────────────────────────────────────────────────────────────────────────┐
│  types.Transaction          │        []byte                            │
│  ┌─────────────────┐        │        ┌─────────────────┐               │
│  │ ID: "tx-1"      │        │        │                 │               │
│  │ Data: [bytes]   │ ──────▶│ ──────▶│ [raw bytes]     │               │
│  │ Timestamp       │        │        │                 │               │
│  │ Signature       │        │        └─────────────────┘               │
│  └─────────────────┘        │                                          │
│                             │  Data 필드만 추출                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.13 ABCI Client 상세

```go
// abci/client.go

type Client struct {
    conn       *grpc.ClientConn
    client     abci.ABCIClient
    address    string
    timeout    time.Duration
    lastHeight int64
    lastAppHash []byte
    mu         sync.Mutex
}

// gRPC 연결 생성
func NewClient(config *Config) (*Client, error) {
    ctx, cancel := context.WithTimeout(context.Background(), config.Timeout)
    defer cancel()
    
    // gRPC 연결 옵션
    opts := []grpc.DialOption{
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),  // 연결될 때까지 블록
    }
    
    // 연결
    conn, err := grpc.DialContext(ctx, config.Address, opts...)
    if err != nil {
        return nil, fmt.Errorf("failed to connect: %w", err)
    }
    
    return &Client{
        conn:    conn,
        client:  abci.NewABCIClient(conn),
        address: config.Address,
        timeout: config.Timeout,
    }, nil
}

// FinalizeBlock gRPC 호출
func (c *Client) FinalizeBlock(ctx context.Context, req *abci.RequestFinalizeBlock) (*abci.ResponseFinalizeBlock, error) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    // 타임아웃 설정
    ctx, cancel := context.WithTimeout(ctx, c.timeout)
    defer cancel()
    
    // gRPC 호출
    resp, err := c.client.FinalizeBlock(ctx, req)
    if err != nil {
        return nil, fmt.Errorf("FinalizeBlock RPC failed: %w", err)
    }
    
    return resp, nil
}
```

### 8.14 상태 전이 (State)

```go
// consensus/pbft/state.go

type Phase int

const (
    PhaseInitial Phase = iota
    PhasePrePrepared
    PhasePrepared
    PhaseCommitted
    PhaseExecuted
)

type State struct {
    View        uint64
    SequenceNum uint64
    Phase       Phase
    
    PrePrepare *PrePrepareMsg
    Block      *types.Block
    Prepares   map[string]*PrepareMsg   // nodeID → Prepare
    Commits    map[string]*CommitMsg    // nodeID → Commit
    
    mu sync.RWMutex
}

func (s *State) TransitionToPrePrepared() {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.Phase == PhaseInitial {
        s.Phase = PhasePrePrepared
    }
}

func (s *State) TransitionToPrepared() {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.Phase == PhasePrePrepared {
        s.Phase = PhasePrepared
    }
}

func (s *State) TransitionToCommitted() {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.Phase == PhasePrepared {
        s.Phase = PhaseCommitted
    }
}

func (s *State) PrepareCount() int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return len(s.Prepares)
}

func (s *State) CommitCount() int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return len(s.Commits)
}
```

**상태 전이 다이어그램:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          상태 전이 다이어그램                            │
└─────────────────────────────────────────────────────────────────────────┘

   ┌─────────────┐
   │   Initial   │  ← State 생성 직후
   └──────┬──────┘
          │
          │ PrePrepare 수신 & 저장
          │ state.SetPrePrepare()
          ▼
   ┌─────────────┐
   │ PrePrepared │  ← 블록 데이터 저장됨
   └──────┬──────┘
          │
          │ Prepare 2f+1개 수집
          │ PrepareCount() >= quorum
          ▼
   ┌─────────────┐
   │  Prepared   │  ← Commit 보내도 됨
   └──────┬──────┘
          │
          │ Commit 2f+1개 수집
          │ CommitCount() >= quorum
          ▼
   ┌─────────────┐
   │  Committed  │  ← 블록 실행해도 됨
   └──────┬──────┘
          │
          │ executeBlock() 완료
          │ state.MarkExecuted()
          ▼
   ┌─────────────┐
   │  Executed   │  ← 완료!
   └─────────────┘
```

### 8.15 검증자 업데이트

```go
func (e *EngineV2) handleValidatorUpdates(updates []*abci.ValidatorUpdate) {
    for _, update := range updates {
        pubKeyData := update.PubKey.GetEd25519()
        
        if update.Power == 0 {
            // Power가 0이면 검증자 제거
            e.validatorSet.RemoveByPubKey(pubKeyData)
            log.Printf("Removed validator: %x", pubKeyData[:8])
        } else {
            // Power가 0보다 크면 추가/업데이트
            validator := &types.Validator{
                ID:        hex.EncodeToString(pubKeyData[:8]),
                PublicKey: pubKeyData,
                Power:     update.Power,
            }
            e.validatorSet.UpdateValidator(validator)
            log.Printf("Updated validator: %s, power: %d", validator.ID, update.Power)
        }
    }
    
    // Quorum 크기 재계산
    newQuorum := e.validatorSet.QuorumSize()
    e.viewChangeManager.UpdateQuorumSize(newQuorum)
    
    log.Printf("Validator set updated: %d validators, quorum: %d",
        len(e.validatorSet.Validators), newQuorum)
}
```

**검증자 업데이트 예시:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        검증자 업데이트 예시                              │
└─────────────────────────────────────────────────────────────────────────┘

현재 상태:
  Validators = [node0(10), node1(10), node2(10), node3(10)]
  Quorum = 3

업데이트 수신:
  updates = [
    {PubKey: node4_key, Power: 10},  // 새 검증자 추가
    {PubKey: node1_key, Power: 0},   // node1 제거
    {PubKey: node2_key, Power: 20},  // node2 파워 증가
  ]

처리 후:
  Validators = [node0(10), node2(20), node3(10), node4(10)]
  Quorum = 3  (4개 노드, 동일)

┌─────────────────────────────────────────────────────────────────────────┐
│  Power의 의미:                                                          │
│  - 투표력 (Voting Power)                                                │
│  - 지분 증명에서는 스테이킹 양에 비례                                     │
│  - Power가 높으면 리더 선출 확률 증가 (가중치 기반일 경우)                 │
│  - Power = 0은 검증자 제거를 의미                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.16 View Change (리더 교체)

```go
// 뷰 체인지 시작
func (e *EngineV2) startViewChange() {
    e.mu.Lock()
    defer e.mu.Unlock()
    
    newView := e.view + 1
    
    // ViewChange 메시지 생성
    viewChangeMsg := &ViewChangeMsg{
        NewView:     newView,
        LastSeqNum:  e.getLastCommittedSeqNum(),
        Checkpoints: e.getStableCheckpoints(),
        PreparedSet: e.getPreparedCertificates(),
        NodeID:      e.config.NodeID,
    }
    
    // ViewChangeManager에 등록
    e.viewChangeManager.AddViewChange(viewChangeMsg)
    
    // 브로드캐스트
    e.broadcastViewChange(viewChangeMsg)
    
    log.Printf("[%s] Started view change to view %d", e.config.NodeID, newView)
}

// ViewChange 메시지 처리
func (e *EngineV2) handleViewChange(msg *Message) {
    var viewChangeMsg ViewChangeMsg
    json.Unmarshal(msg.Payload, &viewChangeMsg)
    
    // ViewChangeManager에 추가
    e.viewChangeManager.AddViewChange(&viewChangeMsg)
    
    // 새 리더인 경우, 2f+1개 모이면 NewView 브로드캐스트
    if e.isNewPrimary(viewChangeMsg.NewView) {
        if e.viewChangeManager.HasQuorum(viewChangeMsg.NewView) {
            e.broadcastNewView(viewChangeMsg.NewView)
        }
    }
}

// NewView 메시지 처리
func (e *EngineV2) handleNewView(msg *Message) {
    var newViewMsg NewViewMsg
    json.Unmarshal(msg.Payload, &newViewMsg)
    
    // 검증
    if !e.validateNewView(&newViewMsg) {
        return
    }
    
    // 뷰 업데이트
    e.view = newViewMsg.View
    
    // 타이머 리셋
    e.resetViewChangeTimer()
    
    // 미처리 블록 재처리
    for _, prePrepare := range newViewMsg.PrePrepareMsgs {
        e.reprocessPrePrepare(&prePrepare)
    }
    
    log.Printf("[%s] Entered new view %d", e.config.NodeID, e.view)
}
```

**View Change 흐름:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         View Change 흐름                                 │
└─────────────────────────────────────────────────────────────────────────┘

정상 상황 (view = 0, 리더 = node0):
  node0 → PrePrepare → node1, node2, node3 → Prepare → Commit → 완료

리더 장애 상황:
  node0이 응답 없음...
  
  ┌─────────────────────────────────────────────────────────────────────┐
  │  viewChangeTimer 만료!                                              │
  │                                                                     │
  │  node1: "10초 지났는데 리더가 안 보내네? → ViewChange 시작!"          │
  │  node2: "나도! → ViewChange 시작!"                                   │
  │  node3: "나도! → ViewChange 시작!"                                   │
  └─────────────────────────────────────────────────────────────────────┘
  
  ViewChange 메시지 교환 (view = 0 → 1):
  
  node1 ─── ViewChange(newView=1) ───▶ node2, node3
  node2 ─── ViewChange(newView=1) ───▶ node1, node3
  node3 ─── ViewChange(newView=1) ───▶ node1, node2
  
  새 리더 (node1)가 2f+1개 ViewChange 수집:
  
  node1: "ViewChange 3개 모았다! → NewView 브로드캐스트!"
  
  node1 ─── NewView(view=1) ───▶ node2, node3
  
  새 뷰 시작:
  
  view = 1, 리더 = node1 (1 % 4 = 1)
  node1이 새 블록 제안 시작!
```

### 8.17 GRPCTransport 상세 (transport/grpc.go)

```go
// transport/grpc.go

// GRPCTransport implements gRPC-based P2P communication for PBFT.
type GRPCTransport struct {
    mu sync.RWMutex

    nodeID   string           // 이 노드의 ID
    address  string           // 리슨 주소 (예: "0.0.0.0:26656")
    server   *grpc.Server     // gRPC 서버
    listener net.Listener     // TCP 리스너

    // Peer connections (다른 노드들과의 연결)
    peers map[string]*peerConn  // nodeID → peerConn

    // Message handler callback (Engine으로 메시지 전달)
    msgHandler func(*pbft.Message)

    // Running state
    running bool
    done    chan struct{}

    // Embed for forward compatibility
    pbftv1.UnimplementedPBFTServiceServer
}

// peerConn represents a connection to a peer node.
type peerConn struct {
    id     string                      // 피어 노드 ID
    addr   string                      // 피어 주소
    conn   *grpc.ClientConn            // gRPC 연결
    client pbftv1.PBFTServiceClient    // gRPC 클라이언트
}
```

**GRPCTransport 구조:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         GRPCTransport 구조                               │
└─────────────────────────────────────────────────────────────────────────┘

                    GRPCTransport (node0)
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│   nodeID: "node0"                                                     │
│   address: "0.0.0.0:26656"                                            │
│                                                                       │
│   ┌─────────────────────────────────────────────────────────────┐    │
│   │                    gRPC Server                               │    │
│   │  (다른 노드로부터 메시지 수신)                                │    │
│   │                                                              │    │
│   │  BroadcastMessage() ← 브로드캐스트 메시지 수신               │    │
│   │  SendMessage()      ← 직접 메시지 수신                       │    │
│   │  MessageStream()    ← 스트림 메시지 수신                     │    │
│   └─────────────────────────────────────────────────────────────┘    │
│                              │                                        │
│                              │ msgHandler(msg)                        │
│                              ▼                                        │
│                    ┌─────────────────┐                                │
│                    │  Engine.msgChan │                                │
│                    └─────────────────┘                                │
│                                                                       │
│   peers map:                                                          │
│   ┌─────────────────────────────────────────────────────────────┐    │
│   │  "node1" → peerConn{conn, client} ──▶ node1:26656           │    │
│   │  "node2" → peerConn{conn, client} ──▶ node2:26656           │    │
│   │  "node3" → peerConn{conn, client} ──▶ node3:26656           │    │
│   └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### 8.18 GRPCTransport 생성 및 시작

```go
// 생성자
func NewGRPCTransport(nodeID, address string) (*GRPCTransport, error) {
    return &GRPCTransport{
        nodeID:  nodeID,
        address: address,
        peers:   make(map[string]*peerConn),
        done:    make(chan struct{}),
    }, nil
}

// Start starts the gRPC server.
func (t *GRPCTransport) Start() error {
    // 1. TCP 리스너 생성
    listener, err := net.Listen("tcp", t.address)
    if err != nil {
        return fmt.Errorf("failed to listen on %s: %w", t.address, err)
    }
    t.listener = listener

    // 2. gRPC 서버 생성 (64MB 메시지 크기 제한)
    t.server = grpc.NewServer(
        grpc.MaxRecvMsgSize(64 * 1024 * 1024), // 64MB
        grpc.MaxSendMsgSize(64 * 1024 * 1024),
    )
    
    // 3. PBFT 서비스 등록
    pbftv1.RegisterPBFTServiceServer(t.server, t)

    t.mu.Lock()
    t.running = true
    t.mu.Unlock()

    // 4. 고루틴으로 서버 실행 (블로킹 방지)
    go func() {
        if err := t.server.Serve(listener); err != nil {
            t.mu.RLock()
            running := t.running
            t.mu.RUnlock()
            if running {
                fmt.Printf("[GRPCTransport] Server error: %v\n", err)
            }
        }
    }()

    fmt.Printf("[GRPCTransport] Started on %s\n", t.address)
    return nil
}
```

**Start() 흐름:**

```
Start() 호출
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  1. net.Listen("tcp", "0.0.0.0:26656")                                  │
│     → TCP 포트 열기                                                      │
│     → "이 포트로 연결 기다림"                                             │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  2. grpc.NewServer(MaxRecvMsgSize, MaxSendMsgSize)                      │
│     → gRPC 서버 인스턴스 생성                                            │
│     → 64MB 메시지 제한 (큰 블록도 전송 가능)                              │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  3. pbftv1.RegisterPBFTServiceServer(server, t)                         │
│     → PBFT 서비스 핸들러 등록                                            │
│     → t가 PBFTServiceServer 인터페이스 구현                              │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  4. go server.Serve(listener)                                           │
│     → 고루틴으로 실행 (메인 스레드 블로킹 방지)                           │
│     → 연결 대기 시작                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.19 피어 연결 (AddPeer)

```go
// AddPeer connects to a remote peer.
func (t *GRPCTransport) AddPeer(nodeID, address string) error {
    // 1. 타임아웃 설정 (10초)
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // 2. gRPC 연결 (클라이언트로서)
    conn, err := grpc.DialContext(
        ctx,
        address,
        grpc.WithTransportCredentials(insecure.NewCredentials()),  // TLS 없이
        grpc.WithBlock(),  // 연결될 때까지 블로킹
    )
    if err != nil {
        return fmt.Errorf("failed to connect to peer %s at %s: %w", nodeID, address, err)
    }

    // 3. PBFT 클라이언트 생성
    client := pbftv1.NewPBFTServiceClient(conn)

    // 4. peers 맵에 저장
    t.mu.Lock()
    t.peers[nodeID] = &peerConn{
        id:     nodeID,
        addr:   address,
        conn:   conn,
        client: client,
    }
    t.mu.Unlock()

    fmt.Printf("[GRPCTransport] Connected to peer %s at %s\n", nodeID, address)
    return nil
}
```

**피어 연결 다이어그램:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         피어 연결 과정                                   │
└─────────────────────────────────────────────────────────────────────────┘

node0이 node1에 연결:

    node0                                           node1
      │                                               │
      │  grpc.DialContext("node1:26656")              │
      │ ─────────────────────────────────────────────▶│
      │                                               │
      │              TCP 연결 수립                      │
      │ ◀─────────────────────────────────────────────│
      │                                               │
      │  NewPBFTServiceClient(conn)                   │
      │                                               │
      │                                               │
      ▼                                               │
┌─────────────────┐                                   │
│ peers["node1"]  │                                   │
│ = peerConn{     │                                   │
│   id: "node1"   │                                   │
│   addr: "..."   │                                   │
│   conn: conn    │ ──── gRPC 연결 ────────────────────│
│   client: client│                                   │
│ }               │                                   │
└─────────────────┘                                   │

4개 노드 연결 상태 (node0 기준):

node0.peers = {
    "node1": peerConn ──▶ node1:26656
    "node2": peerConn ──▶ node2:26656
    "node3": peerConn ──▶ node3:26656
}

node0은:
- 서버로서: node1, node2, node3의 연결을 받음 (수신)
- 클라이언트로서: node1, node2, node3에 연결 (송신)
```

### 8.20 브로드캐스트 (Broadcast)

```go
// Broadcast sends a message to all connected peers.
func (t *GRPCTransport) Broadcast(msg *pbft.Message) error {
    // 1. 피어 목록 복사 (락 최소화)
    t.mu.RLock()
    peers := make([]*peerConn, 0, len(t.peers))
    for _, peer := range t.peers {
        peers = append(peers, peer)
    }
    t.mu.RUnlock()

    // 2. PBFT Message → Protobuf 변환
    protoMsg := messageToProto(msg)

    // 3. 모든 피어에게 병렬 전송
    var wg sync.WaitGroup
    var errMu sync.Mutex
    var lastErr error

    for _, peer := range peers {
        wg.Add(1)
        go func(p *peerConn) {
            defer wg.Done()
            
            // 타임아웃 5초
            ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()

            // gRPC 호출
            _, err := p.client.BroadcastMessage(ctx, &pbftv1.BroadcastMessageRequest{
                Message: protoMsg,
            })
            if err != nil {
                errMu.Lock()
                lastErr = err
                errMu.Unlock()
                fmt.Printf("[GRPCTransport] Broadcast to %s failed: %v\n", p.id, err)
            }
        }(peer)
    }
    
    // 4. 모든 전송 완료 대기
    wg.Wait()

    return lastErr
}
```

**Broadcast 병렬 전송:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Broadcast 병렬 전송                                │
└─────────────────────────────────────────────────────────────────────────┘

node0.Broadcast(PrePrepareMsg)

         │
         │  peers 복사
         ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  peers = [node1, node2, node3]                                  │
    └─────────────────────────────────────────────────────────────────┘
         │
         │  병렬 전송 (고루틴)
         │
    ┌────┴────┬────────────┬────────────┐
    │         │            │            │
    ▼         ▼            ▼            │
┌──────┐  ┌──────┐    ┌──────┐         │
│ go   │  │ go   │    │ go   │         │
│ send │  │ send │    │ send │         │
│ to   │  │ to   │    │ to   │         │
│node1 │  │node2 │    │node3 │         │
└──┬───┘  └──┬───┘    └──┬───┘         │
   │         │           │             │
   │ gRPC    │ gRPC      │ gRPC        │
   ▼         ▼           ▼             │
 node1     node2       node3           │
                                       │
                                       │
   └─────────┴───────────┴─────────────┘
                    │
                    │  wg.Wait() - 모두 완료 대기
                    ▼
               return lastErr


시간 비교:

순차 전송 (안 좋음):              병렬 전송 (좋음):
  node1 ─── 100ms                  node1 ┐
  node2 ─── 100ms                  node2 ├── 100ms (동시)
  node3 ─── 100ms                  node3 ┘
  ─────────────────                ─────────────
  총 300ms                         총 100ms
```

### 8.21 메시지 수신 (BroadcastMessage 서버 핸들러)

```go
// BroadcastMessage handles incoming broadcast messages from peers.
func (t *GRPCTransport) BroadcastMessage(ctx context.Context, req *pbftv1.BroadcastMessageRequest) (*pbftv1.BroadcastMessageResponse, error) {
    // 1. 핸들러가 설정되어 있고, 메시지가 있으면
    if t.msgHandler != nil && req.Message != nil {
        // 2. Protobuf → PBFT Message 변환
        msg := protoToMessage(req.Message)
        
        // 3. Engine으로 전달!
        t.msgHandler(msg)  // → engine.handleIncomingMessage()
    }
    
    return &pbftv1.BroadcastMessageResponse{Success: true}, nil
}
```

**메시지 수신 흐름:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       메시지 수신 흐름                                   │
└─────────────────────────────────────────────────────────────────────────┘

node1이 node0에게 Prepare 전송:

node1                               node0
  │                                   │
  │  BroadcastMessage(PrepareMsg)     │
  │ ─────────────────────────────────▶│
  │                                   │
  │                         ┌─────────▼─────────┐
  │                         │  BroadcastMessage │
  │                         │  (서버 핸들러)     │
  │                         └─────────┬─────────┘
  │                                   │
  │                                   │ protoToMessage()
  │                                   │ (Protobuf → PBFT)
  │                                   │
  │                                   ▼
  │                         ┌─────────────────────┐
  │                         │  msgHandler(msg)    │
  │                         │                     │
  │                         │  = engine.handle    │
  │                         │    IncomingMessage  │
  │                         └─────────┬───────────┘
  │                                   │
  │                                   ▼
  │                         ┌─────────────────────┐
  │                         │  engine.msgChan     │
  │                         │      <- msg         │
  │                         └─────────┬───────────┘
  │                                   │
  │                                   ▼
  │                         ┌─────────────────────┐
  │                         │  handlePrepare(msg) │
  │                         └─────────────────────┘
  │                                   │
  │       Success: true               │
  │ ◀─────────────────────────────────│
```

### 8.22 타입 변환 함수

```go
// messageToProto converts a PBFT Message to protobuf format.
// 전송 시 사용 (PBFT → Protobuf)
func messageToProto(msg *pbft.Message) *pbftv1.PBFTMessage {
    return &pbftv1.PBFTMessage{
        Type:        convertMessageType(msg.Type),      // enum 변환
        View:        msg.View,
        SequenceNum: msg.SequenceNum,
        Digest:      msg.Digest,
        NodeId:      msg.NodeID,
        Timestamp:   timestamppb.New(msg.Timestamp),    // time.Time → Timestamp
        Signature:   msg.Signature,
        Payload:     msg.Payload,
    }
}

// protoToMessage converts a protobuf message to PBFT Message format.
// 수신 시 사용 (Protobuf → PBFT)
func protoToMessage(proto *pbftv1.PBFTMessage) *pbft.Message {
    var ts time.Time
    if proto.Timestamp != nil {
        ts = proto.Timestamp.AsTime()  // Timestamp → time.Time
    }

    return &pbft.Message{
        Type:        convertProtoMessageType(proto.Type),  // enum 변환
        View:        proto.View,
        SequenceNum: proto.SequenceNum,
        Digest:      proto.Digest,
        NodeID:      proto.NodeId,
        Timestamp:   ts,
        Signature:   proto.Signature,
        Payload:     proto.Payload,
    }
}

// MessageType 변환 (PBFT → Protobuf)
func convertMessageType(mt pbft.MessageType) pbftv1.MessageType {
    switch mt {
    case pbft.PrePrepare:
        return pbftv1.MessageType_MESSAGE_TYPE_PRE_PREPARE
    case pbft.Prepare:
        return pbftv1.MessageType_MESSAGE_TYPE_PREPARE
    case pbft.Commit:
        return pbftv1.MessageType_MESSAGE_TYPE_COMMIT
    case pbft.ViewChange:
        return pbftv1.MessageType_MESSAGE_TYPE_VIEW_CHANGE
    case pbft.NewView:
        return pbftv1.MessageType_MESSAGE_TYPE_NEW_VIEW
    case pbft.CheckpointMsgType:
        return pbftv1.MessageType_MESSAGE_TYPE_CHECKPOINT
    default:
        return pbftv1.MessageType_MESSAGE_TYPE_UNSPECIFIED
    }
}
```

**타입 변환 다이어그램:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         타입 변환 흐름                                   │
└─────────────────────────────────────────────────────────────────────────┘

전송 시 (node0 → node1):

  pbft.Message                 pbftv1.PBFTMessage              네트워크
┌─────────────────┐          ┌─────────────────┐          ┌─────────────┐
│ Type: PrePrepare│          │ Type: PRE_PREPARE│          │             │
│ View: 0         │ ───────▶ │ View: 0         │ ───────▶ │   bytes     │
│ SequenceNum: 1  │ message  │ SequenceNum: 1  │ protobuf │ (직렬화)    │
│ Timestamp: time │ ToProto  │ Timestamp: pb.T │ marshal  │             │
│ Payload: []byte │          │ Payload: []byte │          │             │
└─────────────────┘          └─────────────────┘          └─────────────┘


수신 시 (node1에서):

     네트워크               pbftv1.PBFTMessage              pbft.Message
┌─────────────┐          ┌─────────────────┐          ┌─────────────────┐
│             │          │ Type: PRE_PREPARE│          │ Type: PrePrepare│
│   bytes     │ ───────▶ │ View: 0         │ ───────▶ │ View: 0         │
│ (직렬화)    │ protobuf │ SequenceNum: 1  │ proto    │ SequenceNum: 1  │
│             │ unmarshal│ Timestamp: pb.T │ ToMessage│ Timestamp: time │
│             │          │ Payload: []byte │          │ Payload: []byte │
└─────────────┘          └─────────────────┘          └─────────────────┘
```

### 8.23 TransportInterface

```go
// TransportInterface defines the interface for PBFT transport layer.
// This allows for different implementations (gRPC, TCP, mock).
type TransportInterface interface {
    Start() error
    Stop()
    AddPeer(nodeID, address string) error
    RemovePeer(nodeID string)
    Broadcast(msg *pbft.Message) error
    Send(nodeID string, msg *pbft.Message) error
    SetMessageHandler(handler func(*pbft.Message))
    GetPeers() []string
    PeerCount() int
}

// Ensure GRPCTransport implements TransportInterface
var _ TransportInterface = (*GRPCTransport)(nil)
```

**인터페이스 설계 이점:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   TransportInterface 장점                                │
└─────────────────────────────────────────────────────────────────────────┘

인터페이스를 사용하면 다양한 구현 가능:

                    TransportInterface
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  GRPCTransport  │ │  TCPTransport   │ │  MockTransport  │
│  (프로덕션)      │ │  (경량)          │ │  (테스트)        │
└─────────────────┘ └─────────────────┘ └─────────────────┘

Engine은 TransportInterface만 알면 됨:

type EngineV2 struct {
    transport TransportInterface  // 어떤 구현이든 OK
    ...
}

// 프로덕션
engine.transport = NewGRPCTransport(...)

// 테스트
engine.transport = NewMockTransport(...)  // 네트워크 없이 테스트 가능
```

### 8.24 전체 네트워크 토폴로지

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    4개 노드 네트워크 토폴로지                             │
└─────────────────────────────────────────────────────────────────────────┘

모든 노드가 서로 연결 (Full Mesh):

         ┌─────────────────────────────────────────┐
         │                                         │
         │    node0:26656 ◀─────────────────────┐  │
         │        │                              │  │
         │        │  ┌───────────────────────┐   │  │
         │        │  │                       │   │  │
         │        ▼  ▼                       │   │  │
         │    node1:26656 ◀───────────┐      │   │  │
         │        │                    │      │   │  │
         │        │  ┌─────────────┐   │      │   │  │
         │        │  │             │   │      │   │  │
         │        ▼  ▼             │   │      │   │  │
         │    node2:26656 ◀────┐   │   │      │   │  │
         │        │             │   │   │      │   │  │
         │        │             │   │   │      │   │  │
         │        ▼             │   │   │      │   │  │
         │    node3:26656 ──────┴───┴───┴──────┴───┘  │
         │                                         │
         └─────────────────────────────────────────┘

각 노드의 peers 맵:

node0.peers = {node1, node2, node3}  → 3개 연결
node1.peers = {node0, node2, node3}  → 3개 연결
node2.peers = {node0, node1, node3}  → 3개 연결
node3.peers = {node0, node1, node2}  → 3개 연결

총 연결 수: 4 × 3 = 12 (양방향이므로 실제 TCP 연결은 6개)

메시지 흐름 예시 (node0이 PrePrepare 브로드캐스트):

node0.Broadcast(PrePrepare)
    │
    ├──▶ node0.peers["node1"].client.BroadcastMessage() ──▶ node1
    ├──▶ node0.peers["node2"].client.BroadcastMessage() ──▶ node2
    └──▶ node0.peers["node3"].client.BroadcastMessage() ──▶ node3
```

---

## 9. Mempool (트랜잭션 풀) 상세

### 9.1 Mempool 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MEMPOOL                                         │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         txStore (map)                                │    │
│  │                    [txHash] -> *Tx                                   │    │
│  │   ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐          │    │
│  │   │tx1 │ │tx2 │ │tx3 │ │tx4 │ │tx5 │ │tx6 │ │tx7 │ │... │          │    │
│  │   └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      senderIndex (map)                               │    │
│  │                  [sender] -> []*Tx (nonce 정렬)                      │    │
│  │   ┌──────────────────┐  ┌──────────────────┐                        │    │
│  │   │ sender_A:        │  │ sender_B:        │                        │    │
│  │   │  [tx1, tx3, tx5] │  │  [tx2, tx4]      │                        │    │
│  │   └──────────────────┘  └──────────────────┘                        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      priorityQueue (정렬)                            │    │
│  │              (GasPrice 기준 우선순위 정렬)                            │    │
│  │                                                                      │    │
│  │   높은 우선순위 ◄─────────────────────────────► 낮은 우선순위        │    │
│  │   [tx7:100] [tx3:80] [tx1:50] [tx5:30] [tx2:20] [tx4:10]            │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Mempool 역할:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Mempool = 대기실                                                        │
│                                                                         │
│  - 사용자가 보낸 트랜잭션을 임시 보관                                     │
│  - 블록에 포함될 때까지 대기                                              │
│  - 우선순위(가스 가격) 기준 정렬                                          │
│  - 중복/무효 트랜잭션 필터링                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Tx 구조체 (mempool/tx.go)

```go
// mempool/tx.go

// Tx represents a transaction in the mempool.
type Tx struct {
    // 트랜잭션 식별자
    Hash []byte // SHA256 해시
    ID   string // 해시의 hex 문자열

    // 트랜잭션 데이터
    Data []byte // 원본 트랜잭션 바이트

    // 메타데이터
    Sender    string    // 발신자 주소
    Nonce     uint64    // 발신자별 순차 번호
    GasPrice  uint64    // 가스 가격 (우선순위 결정용)
    GasLimit  uint64    // 가스 한도
    Timestamp time.Time // 멤풀 진입 시간

    // 상태
    Height    int64     // CheckTx 시점의 블록 높이
    CheckedAt time.Time // 검증 시간
}

// NewTx creates a new transaction from raw bytes.
func NewTx(data []byte) *Tx {
    hash := sha256.Sum256(data)
    return &Tx{
        Hash:      hash[:],
        ID:        hex.EncodeToString(hash[:]),
        Data:      data,
        Timestamp: time.Now(),
        CheckedAt: time.Now(),
    }
}

// NewTxWithMeta creates a new transaction with metadata.
func NewTxWithMeta(data []byte, sender string, nonce, gasPrice, gasLimit uint64) *Tx {
    tx := NewTx(data)
    tx.Sender = sender
    tx.Nonce = nonce
    tx.GasPrice = gasPrice
    tx.GasLimit = gasLimit
    return tx
}

// Priority returns the priority score for ordering.
// Higher gas price = higher priority.
func (tx *Tx) Priority() uint64 {
    return tx.GasPrice
}
```

**Tx 필드 설명:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  필드          │  타입       │  설명                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  Hash          │  []byte     │  SHA256(Data), 32바이트                  │
│  ID            │  string     │  Hash의 hex 문자열, 조회용 키            │
│  Data          │  []byte     │  원본 트랜잭션 바이트                    │
│  Sender        │  string     │  발신자 주소 (예: "cosmos1abc...")       │
│  Nonce         │  uint64     │  발신자별 순차 번호 (리플레이 방지)       │
│  GasPrice      │  uint64     │  가스당 가격 (우선순위 = 높을수록 먼저)   │
│  GasLimit      │  uint64     │  최대 가스 사용량                        │
│  Timestamp     │  time.Time  │  멤풀 진입 시간 (TTL 계산용)             │
│  Height        │  int64      │  검증 시점 블록 높이                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Mempool 구조체

```go
// mempool/mempool.go

type Mempool struct {
    mu sync.RWMutex

    // 설정
    config *Config

    // 트랜잭션 저장소
    txStore map[string]*Tx // txHash -> Tx

    // 발신자별 인덱스 (nonce 순서 유지)
    senderIndex map[string][]*Tx // sender -> []*Tx (nonce 정렬)

    // 현재 상태
    txCount   int   // 현재 트랜잭션 수
    txBytes   int64 // 현재 총 바이트
    height    int64 // 현재 블록 높이
    isRunning bool

    // 발신자별 마지막 nonce 추적
    senderNonce map[string]uint64 // sender -> lastNonce

    // 최근 제거된 트랜잭션 캐시 (중복 방지)
    recentlyRemoved map[string]time.Time

    // 콜백 (ABCI CheckTx)
    checkTxCallback CheckTxCallback

    // 브로드캐스트 채널
    newTxCh chan *Tx

    // 종료
    ctx    context.Context
    cancel context.CancelFunc

    // 메트릭
    metrics *MempoolMetrics
}
```

**저장소 구조:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          저장소 구조 비교                                │
└─────────────────────────────────────────────────────────────────────────┘

txStore (빠른 조회용):
┌──────────────────────────────────────────────────────────┐
│  Key (txID)              │  Value (*Tx)                  │
├──────────────────────────────────────────────────────────┤
│  "abc123..."             │  &Tx{ID: "abc123", ...}       │
│  "def456..."             │  &Tx{ID: "def456", ...}       │
│  "ghi789..."             │  &Tx{ID: "ghi789", ...}       │
└──────────────────────────────────────────────────────────┘
용도: O(1) 조회, 중복 체크


senderIndex (발신자별 관리):
┌──────────────────────────────────────────────────────────┐
│  Key (sender)            │  Value ([]*Tx)                │
├──────────────────────────────────────────────────────────┤
│  "cosmos1alice..."       │  [tx1(nonce:1), tx3(nonce:2)] │
│  "cosmos1bob..."         │  [tx2(nonce:5), tx4(nonce:6)] │
└──────────────────────────────────────────────────────────┘
용도: 같은 발신자의 트랜잭션 관리, nonce 순서 유지


senderNonce (nonce 추적):
┌──────────────────────────────────────────────────────────┐
│  Key (sender)            │  Value (lastNonce)            │
├──────────────────────────────────────────────────────────┤
│  "cosmos1alice..."       │  2                            │
│  "cosmos1bob..."         │  6                            │
└──────────────────────────────────────────────────────────┘
용도: 다음 nonce 검증, 리플레이 공격 방지
```

### 9.4 Config (설정)

```go
// mempool/mempool.go

type Config struct {
    // 크기 제한
    MaxTxs      int   // 최대 트랜잭션 수 (기본: 5000)
    MaxBytes    int64 // 최대 바이트 (기본: 1GB)
    MaxTxBytes  int   // 단일 트랜잭션 최대 바이트 (기본: 1MB)
    MaxBatchTxs int   // 한 번에 가져올 최대 트랜잭션 수 (기본: 500)

    // TTL (Time To Live)
    TTL time.Duration // 트랜잭션 만료 시간 (기본: 10분)

    // 재검사
    RecheckEnabled bool          // 블록 후 재검사 활성화
    RecheckTimeout time.Duration // 재검사 타임아웃

    // 캐시
    CacheSize int // 최근 제거된 tx 캐시 크기

    // 최소 가스 가격
    MinGasPrice uint64
}

func DefaultConfig() *Config {
    return &Config{
        MaxTxs:         5000,
        MaxBytes:       1024 * 1024 * 1024, // 1GB
        MaxTxBytes:     1024 * 1024,        // 1MB
        MaxBatchTxs:    500,
        TTL:            10 * time.Minute,
        RecheckEnabled: true,
        RecheckTimeout: 5 * time.Second,
        CacheSize:      10000,
        MinGasPrice:    0,
    }
}
```

**설정값 의미:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  설정              │  기본값      │  의미                               │
├─────────────────────────────────────────────────────────────────────────┤
│  MaxTxs            │  5000        │  최대 5000개 트랜잭션 보관           │
│  MaxBytes          │  1GB         │  총 크기 1GB 제한                   │
│  MaxTxBytes        │  1MB         │  단일 트랜잭션 1MB 제한             │
│  MaxBatchTxs       │  500         │  블록당 최대 500개 트랜잭션         │
│  TTL               │  10분        │  10분 지나면 자동 삭제              │
│  RecheckEnabled    │  true        │  블록 후 남은 tx 재검증             │
│  MinGasPrice       │  0           │  최소 가스 가격 (스팸 방지)         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.5 트랜잭션 추가 (AddTx)

```go
// AddTxWithMeta adds a transaction with metadata.
func (mp *Mempool) AddTxWithMeta(txBytes []byte, sender string, nonce, gasPrice, gasLimit uint64) error {
    mp.mu.Lock()
    defer mp.mu.Unlock()

    if !mp.isRunning {
        return ErrMempoolNotRunning
    }

    // 1. 크기 체크
    if len(txBytes) > mp.config.MaxTxBytes {
        return fmt.Errorf("%w: size %d > max %d", ErrTxTooLarge, len(txBytes), mp.config.MaxTxBytes)
    }

    // 2. 트랜잭션 생성
    tx := NewTxWithMeta(txBytes, sender, nonce, gasPrice, gasLimit)

    // 3. 중복 체크
    if _, exists := mp.txStore[tx.ID]; exists {
        return ErrTxAlreadyExists
    }

    // 4. 최근 제거된 트랜잭션 체크
    if _, removed := mp.recentlyRemoved[tx.ID]; removed {
        return ErrTxAlreadyExists
    }

    // 5. 최소 가스 가격 체크
    if gasPrice < mp.config.MinGasPrice {
        return fmt.Errorf("%w: price %d < min %d", ErrInsufficientGas, gasPrice, mp.config.MinGasPrice)
    }

    // 6. Nonce 체크
    if sender != "" {
        if err := mp.checkNonce(sender, nonce); err != nil {
            return err
        }
    }

    // 7. CheckTx 콜백 호출 (ABCI 검증)
    if mp.checkTxCallback != nil {
        if err := mp.checkTxCallback(tx); err != nil {
            return fmt.Errorf("%w: %v", ErrInvalidTx, err)
        }
    }

    // 8. 용량 체크 및 필요시 퇴출
    if err := mp.ensureCapacity(tx); err != nil {
        return err
    }

    // 9. 저장
    mp.addTxLocked(tx)

    // 10. 새 트랜잭션 알림
    select {
    case mp.newTxCh <- tx:
    default:
    }

    return nil
}
```

**AddTx 흐름도:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          트랜잭션 추가 흐름                              │
└─────────────────────────────────────────────────────────────────────────┘

  Client                   Mempool                    ABCI App
    │                         │                          │
    │   1. AddTx(txBytes)     │                          │
    │ ───────────────────────►│                          │
    │                         │                          │
    │                         │ 2. 크기 체크              │
    │                         │    (> 1MB? → 거부)        │
    │                         │                          │
    │                         │ 3. 중복 체크              │
    │                         │    (txStore에 존재?)      │
    │                         │                          │
    │                         │ 4. 가스 가격 체크         │
    │                         │    (< MinGasPrice?)      │
    │                         │                          │
    │                         │ 5. Nonce 체크            │
    │                         │    (너무 낮거나 갭?)     │
    │                         │                          │
    │                         │ 6. CheckTx 콜백          │
    │                         │ ────────────────────────►│
    │                         │                          │ 앱에서 검증
    │                         │◄────────────────────────│ (잔액, 서명 등)
    │                         │                          │
    │                         │ 7. 용량 체크             │
    │                         │    (가득 찼으면 퇴출)    │
    │                         │                          │
    │                         │ 8. 저장                  │
    │                         │    - txStore[id] = tx    │
    │                         │    - senderIndex 갱신    │
    │                         │                          │
    │  9. nil (성공)          │                          │
    │◄────────────────────────│                          │
```

### 9.6 용량 관리 (퇴출 로직)

```go
// ensureCapacity ensures there's room for the new transaction.
func (mp *Mempool) ensureCapacity(newTx *Tx) error {
    // 트랜잭션 수 체크
    for mp.txCount >= mp.config.MaxTxs {
        if err := mp.evictLowestPriority(newTx.GasPrice); err != nil {
            return ErrMempoolFull
        }
    }

    // 바이트 체크
    for mp.txBytes+int64(newTx.Size()) > mp.config.MaxBytes {
        if err := mp.evictLowestPriority(newTx.GasPrice); err != nil {
            return ErrMempoolFull
        }
    }

    return nil
}

// evictLowestPriority removes the lowest priority transaction.
func (mp *Mempool) evictLowestPriority(minPrice uint64) error {
    var lowestTx *Tx
    var lowestPrice uint64 = ^uint64(0) // Max uint64

    // 가장 낮은 가스 가격 찾기
    for _, tx := range mp.txStore {
        if tx.GasPrice < lowestPrice {
            lowestPrice = tx.GasPrice
            lowestTx = tx
        }
    }

    if lowestTx == nil {
        return errors.New("no transaction to evict")
    }

    // 새 트랜잭션보다 낮은 우선순위만 퇴출
    if lowestPrice >= minPrice {
        return errors.New("cannot evict higher priority transaction")
    }

    mp.removeTxLocked(lowestTx.ID, true)
    return nil
}
```

**퇴출 로직 예시:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           퇴출 (Eviction) 예시                           │
└─────────────────────────────────────────────────────────────────────────┘

현재 멤풀 (MaxTxs = 5, 가득 참):
┌─────────────────────────────────────────────────────────────────────────┐
│  tx1 (GasPrice: 100)                                                    │
│  tx2 (GasPrice: 50)                                                     │
│  tx3 (GasPrice: 30)   ← 가장 낮은 우선순위                              │
│  tx4 (GasPrice: 80)                                                     │
│  tx5 (GasPrice: 60)                                                     │
└─────────────────────────────────────────────────────────────────────────┘

새 트랜잭션 도착: tx6 (GasPrice: 70)

1. 멤풀 가득 참 (5 >= MaxTxs)
2. 가장 낮은 우선순위 찾기 → tx3 (GasPrice: 30)
3. tx6.GasPrice(70) > tx3.GasPrice(30) → 퇴출 가능!
4. tx3 제거, tx6 추가

결과:
┌─────────────────────────────────────────────────────────────────────────┐
│  tx1 (GasPrice: 100)                                                    │
│  tx2 (GasPrice: 50)                                                     │
│  tx6 (GasPrice: 70)   ← 새로 추가됨                                     │
│  tx4 (GasPrice: 80)                                                     │
│  tx5 (GasPrice: 60)                                                     │
└─────────────────────────────────────────────────────────────────────────┘

만약 tx7 (GasPrice: 20) 이 오면?
→ tx7.GasPrice(20) < 최소(50) → 퇴출 불가 → ErrMempoolFull
```

### 9.7 트랜잭션 조회 (ReapMaxTxs)

```go
// ReapMaxTxs returns up to max transactions for block proposal.
// Transactions are sorted by priority (gas price).
func (mp *Mempool) ReapMaxTxs(max int) []*Tx {
    mp.mu.RLock()
    defer mp.mu.RUnlock()

    if max <= 0 || max > mp.config.MaxBatchTxs {
        max = mp.config.MaxBatchTxs
    }

    if mp.txCount == 0 {
        return nil
    }

    // 모든 트랜잭션을 슬라이스로 복사
    txs := make([]*Tx, 0, mp.txCount)
    for _, tx := range mp.txStore {
        txs = append(txs, tx)
    }

    // 우선순위 정렬 (GasPrice 내림차순)
    sort.Slice(txs, func(i, j int) bool {
        return txs[i].GasPrice > txs[j].GasPrice
    })

    // 상위 max개 반환
    if len(txs) > max {
        txs = txs[:max]
    }

    return txs
}
```

**ReapMaxTxs 사용 시점:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      블록 제안 시 트랜잭션 선택                          │
└─────────────────────────────────────────────────────────────────────────┘

  Primary Node              Mempool
       │                       │
       │  proposeBlock()       │
       │                       │
       │  ReapMaxTxs(500)      │
       │ ─────────────────────►│
       │                       │
       │                       │ 1. 모든 tx 수집
       │                       │
       │                       │ 2. GasPrice 내림차순 정렬
       │                       │    [100, 80, 70, 60, 50, ...]
       │                       │
       │                       │ 3. 상위 500개 선택
       │                       │
       │  []*Tx (정렬됨)       │
       │◄───────────────────── │
       │                       │
       │  블록 생성            │
       │                       │


정렬 예시:

정렬 전 (txStore 순서 = 무작위):
  [tx3:30, tx1:100, tx5:60, tx2:50, tx4:80]

정렬 후 (GasPrice 내림차순):
  [tx1:100, tx4:80, tx5:60, tx2:50, tx3:30]
      ↑
   우선순위 높음 (먼저 블록에 포함)
```

### 9.8 블록 커밋 후 처리 (Update)

```go
// Update is called after a block is committed.
func (mp *Mempool) Update(height int64, committedTxs [][]byte) error {
    mp.mu.Lock()
    defer mp.mu.Unlock()

    mp.height = height

    // 1. 커밋된 트랜잭션 제거
    for _, txBytes := range committedTxs {
        tx := NewTx(txBytes)
        mp.removeTxLocked(tx.ID, true)  // 캐시에 추가
    }

    // 2. Recheck (선택적)
    if mp.config.RecheckEnabled {
        mp.recheckTxsLocked()
    }

    return nil
}

// recheckTxsLocked rechecks all remaining transactions.
func (mp *Mempool) recheckTxsLocked() {
    if mp.checkTxCallback == nil {
        return
    }

    toRemove := make([]string, 0)

    for id, tx := range mp.txStore {
        // 재검증
        if err := mp.checkTxCallback(tx); err != nil {
            toRemove = append(toRemove, id)
        }
    }

    // 무효 트랜잭션 제거
    for _, id := range toRemove {
        mp.removeTxLocked(id, false)
    }
}
```

**Update 흐름:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         블록 커밋 후 처리                                │
└─────────────────────────────────────────────────────────────────────────┘

  Consensus Engine            Mempool                    ABCI App
       │                         │                          │
       │  executeBlock() 완료    │                          │
       │                         │                          │
       │  Update(height=100,     │                          │
       │     committedTxs)       │                          │
       │ ───────────────────────►│                          │
       │                         │                          │
       │                         │ 1. 커밋된 tx 제거        │
       │                         │    - txStore에서 삭제    │
       │                         │    - senderIndex 갱신    │
       │                         │    - 캐시에 추가         │
       │                         │                          │
       │                         │ 2. 높이 업데이트         │
       │                         │    height = 100          │
       │                         │                          │
       │                         │ 3. Recheck               │
       │                         │    (남은 tx 재검증)      │
       │                         │ ────────────────────────►│
       │                         │                          │ CheckTx
       │                         │◄────────────────────────│
       │                         │                          │
       │                         │ 4. 무효 tx 제거          │
       │                         │                          │
       │  완료                   │                          │
       │◄─────────────────────── │                          │


왜 Recheck가 필요한가?

블록 100에서:
- tx_A: Alice → Bob 100원 (성공, 커밋됨)
- tx_B: Alice → Charlie 50원 (멤풀에 대기 중)

블록 커밋 후:
- Alice 잔액이 100원 → 0원으로 변경
- tx_B는 이제 무효 (잔액 부족)
- Recheck로 tx_B 발견 → 제거
```

### 9.9 백그라운드 작업 (만료 처리)

```go
// expireLoop periodically removes expired transactions.
func (mp *Mempool) expireLoop() {
    ticker := time.NewTicker(mp.config.TTL / 2)  // 5분마다 체크
    defer ticker.Stop()

    for {
        select {
        case <-mp.ctx.Done():
            return
        case <-ticker.C:
            mp.expireTxs()
        }
    }
}

// expireTxs removes expired transactions.
func (mp *Mempool) expireTxs() {
    mp.mu.Lock()
    defer mp.mu.Unlock()

    now := time.Now()
    toRemove := make([]string, 0)

    for id, tx := range mp.txStore {
        if now.Sub(tx.Timestamp) > mp.config.TTL {  // 10분 경과
            toRemove = append(toRemove, id)
        }
    }

    for _, id := range toRemove {
        mp.removeTxLocked(id, true)
    }
}
```

**TTL 만료 예시:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           TTL 만료 처리                                  │
└─────────────────────────────────────────────────────────────────────────┘

시간 흐름:

00:00  tx1 추가 (Timestamp: 00:00)
00:03  tx2 추가 (Timestamp: 00:03)
00:05  expireLoop 실행 → 만료된 tx 없음
00:07  tx3 추가 (Timestamp: 00:07)
00:10  expireLoop 실행
       - tx1: 00:10 - 00:00 = 10분 → TTL 초과 → 제거!
       - tx2: 00:10 - 00:03 = 7분 → OK
       - tx3: 00:10 - 00:07 = 3분 → OK
00:13  tx2 만료 예정...


왜 TTL이 필요한가?

1. 오래된 트랜잭션 정리
   - 네트워크 혼잡으로 블록에 못 들어간 tx
   - 낮은 가스 가격으로 계속 밀리는 tx

2. 메모리 관리
   - 무한정 쌓이는 것 방지

3. 사용자 경험
   - 너무 오래된 tx는 의미 없음
   - 사용자가 다시 제출하도록 유도
```

### 9.10 Reactor (네트워크 연동)

```go
// mempool/reactor.go

// Reactor connects the mempool to the network layer.
type Reactor struct {
    mu sync.RWMutex

    config  *ReactorConfig
    mempool *Mempool

    // 네트워크 브로드캐스터
    broadcaster Broadcaster

    // 브로드캐스트 큐
    broadcastQueue chan *Tx

    // 상태
    isRunning bool
    ctx       context.Context
    cancel    context.CancelFunc
}

// Broadcaster defines the interface for broadcasting transactions.
type Broadcaster interface {
    BroadcastTx(tx []byte) error
    SendTx(peerID string, tx []byte) error
}
```

**Reactor 역할:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              REACTOR                                     │
│                                                                          │
│   ┌───────────┐         ┌───────────┐         ┌───────────┐            │
│   │   Client  │         │  Reactor  │         │   Peers   │            │
│   └─────┬─────┘         └─────┬─────┘         └─────┬─────┘            │
│         │                     │                     │                   │
│         │  SubmitTx           │                     │                   │
│         │────────────────────►│                     │                   │
│         │                     │                     │                   │
│         │                     │  AddTx to Mempool   │                   │
│         │                     │────────┐            │                   │
│         │                     │        │            │                   │
│         │                     │◄───────┘            │                   │
│         │                     │                     │                   │
│         │                     │  BroadcastTx        │                   │
│         │                     │────────────────────►│  다른 노드들에게  │
│         │                     │                     │  전파             │
│         │                     │                     │                   │
│         │                     │◄────────────────────│                   │
│         │                     │  ReceiveTx          │  다른 노드에서    │
│         │                     │                     │  수신             │
│         │                     │  AddTx to Mempool   │                   │
│         │                     │────────┐            │                   │
│         │                     │        │            │                   │
│         │                     │◄───────┘            │                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.11 SubmitTx (클라이언트 → 네트워크)

```go
// SubmitTxWithMeta submits a transaction with metadata.
func (r *Reactor) SubmitTxWithMeta(txBytes []byte, sender string, nonce, gasPrice, gasLimit uint64) error {
    // 1. 멤풀에 추가
    if err := r.mempool.AddTxWithMeta(txBytes, sender, nonce, gasPrice, gasLimit); err != nil {
        return err
    }

    // 2. 브로드캐스트 큐에 추가
    if r.config.BroadcastEnabled {
        tx := NewTxWithMeta(txBytes, sender, nonce, gasPrice, gasLimit)
        select {
        case r.broadcastQueue <- tx:
        default:
            // 큐가 가득 차면 무시 (이미 멤풀에는 추가됨)
        }
    }

    return nil
}
```

### 9.12 브로드캐스트 루프

```go
// broadcastLoop handles broadcasting transactions to peers.
func (r *Reactor) broadcastLoop() {
    var batch []*Tx
    ticker := time.NewTicker(r.config.BroadcastDelay)  // 10ms
    defer ticker.Stop()

    for {
        select {
        case <-r.ctx.Done():
            return

        case tx := <-r.broadcastQueue:
            batch = append(batch, tx)

            // 배치가 가득 차면 즉시 전송
            if len(batch) >= r.config.MaxBroadcastBatch {
                r.broadcastBatch(batch)
                batch = nil
            }

        case <-ticker.C:
            // 주기적으로 배치 전송
            if len(batch) > 0 {
                r.broadcastBatch(batch)
                batch = nil
            }
        }
    }
}
```

**배치 브로드캐스트:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        배치 브로드캐스트                                  │
└─────────────────────────────────────────────────────────────────────────┘

개별 전송 (비효율):
  tx1 → 네트워크 → 대기
  tx2 → 네트워크 → 대기
  tx3 → 네트워크 → 대기
  ...
  총 100번의 네트워크 호출


배치 전송 (효율적):
  [tx1, tx2, tx3, ..., tx100] → 네트워크
  10ms 동안 모아서 한 번에 전송
  또는 100개 모이면 즉시 전송


시간 흐름:

00:00.000  tx1 도착 → batch = [tx1]
00:00.003  tx2 도착 → batch = [tx1, tx2]
00:00.007  tx3 도착 → batch = [tx1, tx2, tx3]
00:00.010  ticker 발동 → broadcastBatch([tx1, tx2, tx3])
                        batch = []
00:00.015  tx4 도착 → batch = [tx4]
...
```

### 9.13 Mempool과 PBFT 통합

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Mempool과 PBFT 통합 흐름                             │
└─────────────────────────────────────────────────────────────────────────┘

1. 트랜잭션 수신:
   Client → Reactor.SubmitTx() → Mempool.AddTx() → CheckTx (ABCI)
                              → Reactor.BroadcastTx() → 다른 노드들

2. 블록 제안 (리더):
   EngineV2.proposeBlock()
       │
       ├─▶ Mempool.ReapMaxTxs(500)  ← 상위 500개 트랜잭션 선택
       │
       ├─▶ ABCIAdapter.PrepareProposal()  ← 앱이 최종 정렬
       │
       └─▶ 블록 생성 & PrePrepare 브로드캐스트

3. 블록 실행:
   EngineV2.executeBlock()
       │
       ├─▶ ABCIAdapter.FinalizeBlock()  ← 블록 실행
       │
       ├─▶ ABCIAdapter.Commit()  ← 상태 확정
       │
       └─▶ Mempool.Update(height, committedTxs)  ← 커밋된 tx 제거


전체 흐름 다이어그램:

  Client          Mempool         EngineV2        ABCIAdapter      Cosmos App
    │                │               │                │               │
    │  SubmitTx      │               │                │               │
    │───────────────►│               │                │               │
    │                │               │                │               │
    │                │  CheckTx      │                │               │
    │                │───────────────┼───────────────►│───────────────►│
    │                │◄──────────────┼────────────────│◄───────────────│
    │                │               │                │               │
    │                │               │  (리더만)      │               │
    │                │  ReapMaxTxs   │                │               │
    │                │◄──────────────│                │               │
    │                │───────────────►│               │               │
    │                │               │                │               │
    │                │               │ PrepareProposal│               │
    │                │               │───────────────►│───────────────►│
    │                │               │◄───────────────│◄───────────────│
    │                │               │                │               │
    │                │               │  PBFT 합의...  │               │
    │                │               │                │               │
    │                │               │ FinalizeBlock  │               │
    │                │               │───────────────►│───────────────►│
    │                │               │◄───────────────│◄───────────────│
    │                │               │                │               │
    │                │  Update       │                │               │
    │                │◄──────────────│                │               │
    │                │ (커밋된 tx 제거)               │               │
```

---

## 10. 요약 정리

### 10.1 계층 구조 요약

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
│  Mempool       │  mempool/mempool.go    │  트랜잭션 대기열 관리          │
│  Reactor       │  mempool/reactor.go    │  트랜잭션 네트워크 전파        │
│  Protobuf      │  api/pbft/v1/*.go      │  메시지 정의                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 ABCI 메서드 매핑

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

### 10.3 전체 데이터 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         전체 데이터 흐름                                 │
└─────────────────────────────────────────────────────────────────────────┘

사용자 요청 (트랜잭션)
       │
       ▼
┌─────────────────┐
│    Reactor      │  SubmitTx() - 네트워크 전파
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Mempool      │  AddTx() - 검증 후 저장
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Node        │  SubmitTx() - 엔진에 전달
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
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Mempool      │  Update() - 커밋된 tx 제거
└─────────────────┘
```

### 10.4 Mempool 요약

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Mempool 요약                                    │
└─────────────────────────────────────────────────────────────────────────┘

역할:
  - 트랜잭션 임시 저장 (블록에 포함될 때까지)
  - 우선순위 관리 (GasPrice 기준)
  - 중복/무효 트랜잭션 필터링
  - 만료 트랜잭션 정리

저장소:
  - txStore: 빠른 조회 (O(1))
  - senderIndex: 발신자별 관리 (nonce 순서)
  - senderNonce: 리플레이 공격 방지

주요 메서드:
  - AddTx(): 트랜잭션 추가 (검증 포함)
  - ReapMaxTxs(): 블록용 트랜잭션 선택
  - Update(): 블록 커밋 후 정리

용량 관리:
  - MaxTxs: 최대 5000개
  - MaxBytes: 최대 1GB
  - TTL: 10분 후 만료
  - 퇴출: 낮은 GasPrice 우선 제거

Reactor:
  - 네트워크 전파 담당
  - 배치 브로드캐스트 (효율성)
  - 다른 노드에서 수신
```

### 10.5 GRPCTransport 요약

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       GRPCTransport 요약                                 │
└─────────────────────────────────────────────────────────────────────────┘

역할:
  - 노드 간 P2P 통신
  - PBFT 메시지 송수신

구조:
  - 서버: 다른 노드의 연결 수신
  - 클라이언트: 다른 노드에 연결

주요 메서드:
  - Start(): gRPC 서버 시작
  - AddPeer(): 피어 연결
  - Broadcast(): 모든 피어에 전송 (병렬)
  - Send(): 특정 피어에 전송

타입 변환:
  - messageToProto(): PBFT → Protobuf (전송용)
  - protoToMessage(): Protobuf → PBFT (수신용)

메시지 수신:
  - BroadcastMessage() 핸들러
  - msgHandler 콜백 → Engine.msgChan
```

### 10.6 핵심 포인트

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           핵심 포인트                                    │
└─────────────────────────────────────────────────────────────────────────┘

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
   - Mempool: 트랜잭션 관리만 담당
   - Transport: P2P 통신만 담당
   
4. 장점
   - 표준 ABCI 인터페이스 사용 → 어떤 Cosmos SDK 앱이든 연결 가능
   - 계층 분리 → 테스트 용이 (Mock 사용 가능)
   - 코드 가독성 향상
   - 각 컴포넌트 독립적 개발/테스트 가능

5. 메시지 흐름 (PBFT)
   - PrePrepare: 리더 → 모든 노드
   - Prepare: 모든 노드 → 모든 노드 (브로드캐스트)
   - Commit: 모든 노드 → 모든 노드 (브로드캐스트)

6. 트랜잭션 흐름
   - 클라이언트 → Reactor → Mempool → 검증 → 저장
   - Mempool → Engine → PrepareProposal → 블록 생성
   - 블록 커밋 → Mempool.Update() → 커밋된 tx 제거
```

---
