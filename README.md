# CometBFT Consensus 완벽 가이드

> Tendermint 알고리즘의 모든 것 - 코드 레벨 심층 분석
> 
> **버전**: CometBFT v0.38+ / v1.0 기준 | **최종 업데이트**: 2025년

---

## 목차

1. [CometBFT 개요](#1-cometbft-개요)
   - [아키텍처](#아키텍처)
   - [코드 구조](#코드-구조)
2. [Tendermint 알고리즘](#2-tendermint-알고리즘)
   - [합의 속성](#합의-속성)
   - [Polka와 Lock](#polka와-lock-메커니즘)
3. [State 구조체](#3-state-구조체)
   - [RoundState](#roundstate---라운드-상태)
   - [VoteSet](#voteset---투표-집합)
4. [컨센서스 흐름](#4-컨센서스-흐름)
   - [receiveRoutine](#receiveroutine---메인-이벤트-루프)
   - [상태 전이](#상태-전이-함수들)
5. [ABCI 2.0 (ABCI++)](#5-abci-20-abci)
   - [PrepareProposal/ProcessProposal](#prepareproposal--processproposal)
   - [Vote Extensions](#vote-extensions-투표-확장)
   - [FinalizeBlock](#finalizeblock)
6. [WAL (Write-Ahead Log)](#6-wal-write-ahead-log)
7. [Proposer 선정](#7-proposer-선정-알고리즘)
8. [타임아웃 설정](#8-타임아웃-설정)
9. [Evidence 처리](#9-evidence-처리)
10. [블록 조각과 P2P](#10-블록-조각과-p2p)
11. [Mempool](#11-mempool)
12. [정리](#12-정리)

---

## 1. CometBFT 개요

**CometBFT**는 Tendermint Core의 공식 후속 프로젝트로, Byzantine Fault Tolerant 컨센서스 엔진입니다. Cosmos SDK와 함께 사용되어 수백 개의 블록체인을 지원합니다.

### 핵심 특징

- **즉시 최종성(Instant Finality)**: 블록이 커밋되면 되돌릴 수 없음
- **Byzantine Fault Tolerance**: 최대 f < n/3 악의적 노드 허용 (n 노드 중 f 악성)
- **Partial Synchrony**: GST(Global Stabilization Time) 이후 동기화 가정
- **ABCI 2.0**: PrepareProposal, ProcessProposal, Vote Extensions 지원

### 아키텍처

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CometBFT Node                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                      Consensus Engine                          │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │  │
│  │  │ State    │  │ Reactor  │  │ WAL      │  │ Evidence │       │  │
│  │  │ Machine  │  │ (P2P)    │  │          │  │ Pool     │       │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│         │                                                            │
│         │ ABCI 2.0 (gRPC/Socket)                                     │
│         ▼                                                            │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    Application (Cosmos SDK)                    │  │
│  │  PrepareProposal → ProcessProposal → ExtendVote → FinalizeBlock│  │
│  └────────────────────────────────────────────────────────────────┘  │
│         │                                                            │
│  ┌──────┴───────┐    ┌──────────────┐    ┌──────────────┐           │
│  │  Mempool     │    │  Block Store │    │  State Store │           │
│  │  (TxPool)    │    │  (Blocks)    │    │  (AppState)  │           │
│  └──────────────┘    └──────────────┘    └──────────────┘           │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    P2P Network Layer                           │  │
│  │        Gossip Protocol / Connection Management                 │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 코드 구조

**github.com/cometbft/cometbft**

```
cometbft/
├── // 핵심 컨센서스
├── consensus/
│   ├── state.go          // State 구조체, enterPropose, enterPrevote 등
│   ├── reactor.go        // P2P 메시지 처리
│   ├── wal.go            // Write-Ahead Log
│   └── types/
│       └── round_state.go // RoundState, HeightVoteSet
│
├── // ABCI 인터페이스
├── abci/
│   ├── types/
│   │   └── types.pb.go   // ABCI 메시지 정의
│   └── client/           // ABCI 클라이언트
│
├── // 상태 관리
├── state/
│   ├── state.go          // 블록체인 상태
│   ├── execution.go      // 블록 실행
│   └── store.go          // 상태 저장소
│
├── // 타입 정의
├── types/
│   ├── block.go          // Block, Header
│   ├── vote.go           // Vote, VoteSet
│   ├── proposal.go       // Proposal
│   ├── validator.go      // Validator, ValidatorSet
│   └── evidence.go       // Evidence (이중서명 증거)
│
├── // P2P 네트워킹
├── p2p/
│   ├── switch.go         // P2P Switch (피어 관리)
│   ├── conn/             // MConnection (멀티플렉스)
│   └── pex/              // Peer Exchange
│
├── // Mempool
├── mempool/
│   ├── clist_mempool.go  // 기본 Mempool 구현
│   └── reactor.go        // Mempool P2P
│
└── // Evidence
    evidence/
    ├── pool.go           // Evidence Pool
    └── reactor.go        // Evidence P2P
```

### 버전별 ABCI 비교

| v0.34 (Legacy) | v0.37 | v0.38+ / v1.0 (최신) |
|----------------|-------|------------------|
| BeginBlock | + PrepareProposal | PrepareProposal |
| DeliverTx × N | + ProcessProposal | ProcessProposal |
| EndBlock | BeginBlock/EndBlock | ExtendVote |
| Commit | | VerifyVoteExtension |
| | | **FinalizeBlock** |
| | | Commit |

---

## 2. Tendermint 알고리즘

Tendermint는 PBFT 계열의 BFT 컨센서스 알고리즘으로, DLS(Dwork-Lynch-Stockmeyer) 알고리즘에서 영감을 받았습니다.

### 합의 속성

| 속성 | 설명 | 보장 조건 |
|------|------|----------|
| **Safety** | 두 개의 다른 블록이 같은 높이에서 커밋되지 않음 | f < n/3 (항상) |
| **Liveness** | 결국 새 블록이 커밋됨 | f < n/3 + 부분 동기 (GST 이후) |
| **Validity** | 커밋된 블록은 유효한 트랜잭션만 포함 | 정직한 제안자 |

### 컨센서스 라운드 구조

```
Height H의 컨센서스:

┌─────────────────────────────────────────────────────────────────────┐
│                          Round 0                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   NewHeight ──→ Propose ──→ Prevote ──→ Precommit ──→ Commit       │
│       │           │           │            │            │          │
│       │      프로포저가    블록 검증 후   Prevote 결과  2/3+ 모이면   │
│       │      블록 제안    1차 투표     기반 2차 투표   블록 확정      │
│       │                                                             │
│       │         (실패 시)                                           │
│       │            ↓                                                │
│       └──────→ Round 1 ──→ Round 2 ──→ ...                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

타임아웃:
• ProposeTimeout: 제안 대기 (기본 3초)
• PrevoteTimeout: Prevote 수집 대기 (+2/3 Any 후)
• PrecommitTimeout: Precommit 수집 대기 (+2/3 Any 후)
```

### Polka와 Lock 메커니즘

#### 핵심 용어

- **Polka**: 특정 블록(또는 nil)에 대한 +2/3 Prevote 집합
- **PoLC (Proof of Lock Change)**: Lock을 변경하거나 해제할 수 있는 증거
- **Lock**: 검증자가 특정 블록에 "잠김" (다른 블록에 Prevote 불가)

#### Lock 규칙 (consensus/state.go)

```go
// Lock 규칙:
// 1. +2/3 Prevote를 본 블록에 Lock
// 2. Lock된 블록에만 Precommit 가능
// 3. 더 높은 라운드의 PoLC가 있어야 Lock 해제 가능

func (cs *State) enterPrecommit(height int64, round int32) {
    // Prevote 결과 확인
    blockID, ok := cs.Votes.Prevotes(round).TwoThirdsMajority()
    
    if !ok {
        // +2/3 없음 → nil Precommit
        cs.signAddVote(PrecommitType, nil)
        return
    }
    
    if blockID.IsNil() {
        // +2/3가 nil 선택 → Lock 해제
        cs.LockedRound = -1
        cs.LockedBlock = nil
        cs.signAddVote(PrecommitType, nil)
        return
    }
    
    // +2/3가 특정 블록 선택 → Lock & Precommit!
    if cs.ProposalBlock.HashesTo(blockID.Hash) {
        cs.LockedRound = round
        cs.LockedBlock = cs.ProposalBlock
        cs.signAddVote(PrecommitType, blockID.Hash)
    }
}
```

#### Lock이 Safety를 보장하는 이유

```
시나리오: Round 0에서 Block A에 Lock

Round 0:
  • 검증자 1,2,3이 Block A에 Prevote (+2/3)
  • 검증자 1,2,3이 Block A에 Lock
  • 네트워크 지연으로 Precommit 수집 실패

Round 1:
  • 새 제안자가 Block B 제안
  • 검증자 1,2,3: "나는 Block A에 Lock되어 있어!"
  • Block A에 대한 PoLC 없이는 Block B에 Prevote 불가
  
결과: 
  • Safety 보장: Block A와 Block B 동시 커밋 불가능
  • 정직한 노드가 2/3 이상이면, 결국 같은 블록에 합의
```

### 투표 규칙 상세

**consensus/state.go - defaultDoPrevote**

```go
func (cs *State) defaultDoPrevote(height int64, round int32) {
    // 규칙 1: Lock된 블록이 있으면 그 블록에 투표
    if cs.LockedBlock != nil {
        cs.signAddVote(PrevoteType, cs.LockedBlock.Hash())
        return
    }

    // 규칙 2: 제안된 블록이 없으면 nil 투표
    if cs.ProposalBlock == nil {
        cs.signAddVote(PrevoteType, nil)
        return
    }

    // 규칙 3: 블록 검증
    err := cs.blockExec.ValidateBlock(cs.state, cs.ProposalBlock)
    if err != nil {
        cs.signAddVote(PrevoteType, nil)
        return
    }

    // 규칙 4: 애플리케이션 검증 (ProcessProposal)
    isAppValid, err := cs.blockExec.ProcessProposal(cs.ProposalBlock, cs.state)
    if err != nil || !isAppValid {
        cs.signAddVote(PrevoteType, nil)
        return
    }

    // 모든 검증 통과! 블록에 투표
    cs.signAddVote(PrevoteType, cs.ProposalBlock.Hash())
}
```

---

## 3. State 구조체

### RoundState - 라운드 상태

**consensus/types/round_state.go**

```go
// RoundState는 컨센서스의 현재 상태를 나타냅니다
type RoundState struct {
    // 위치 정보
    Height    int64         // 현재 블록 높이
    Round     int32         // 현재 라운드 (0부터 시작)
    Step      RoundStepType // 현재 단계

    // 시간 정보
    StartTime     time.Time  // 라운드 시작 시간
    CommitTime    time.Time  // 커밋 시간

    // 검증자 정보
    Validators           *types.ValidatorSet
    Proposer             *types.Validator

    // 제안 정보
    Proposal             *types.Proposal
    ProposalBlock        *types.Block
    ProposalBlockParts   *types.PartSet

    // Lock 정보 (Safety 핵심!)
    LockedRound          int32
    LockedBlock          *types.Block
    LockedBlockParts     *types.PartSet

    // Valid 정보 (Precommit 없이 +2/3 Prevote 본 경우)
    ValidRound           int32
    ValidBlock           *types.Block
    ValidBlockParts      *types.PartSet

    // 투표 정보
    Votes                *HeightVoteSet

    // 이전 높이 커밋
    LastCommit           *types.ExtendedCommit
}

// RoundStepType 정의
const (
    RoundStepNewHeight     = 0x01 // 새 높이 대기
    RoundStepNewRound      = 0x02 // 새 라운드 시작
    RoundStepPropose       = 0x03 // 제안 단계
    RoundStepPrevote       = 0x04 // Prevote 단계
    RoundStepPrevoteWait   = 0x05 // +2/3 Any 대기
    RoundStepPrecommit     = 0x06 // Precommit 단계
    RoundStepPrecommitWait = 0x07 // +2/3 Any 대기
    RoundStepCommit        = 0x08 // 커밋 완료
)
```

### VoteSet - 투표 집합

**types/vote_set.go**

```go
// VoteSet은 특정 (Height, Round, Type)의 모든 투표를 관리
type VoteSet struct {
    chainID       string
    height        int64
    round         int32
    signedMsgType SignedMsgType  // Prevote or Precommit
    valSet        *ValidatorSet

    mtx           sync.Mutex
    votesBitArray *bits.BitArray  // 어떤 검증자가 투표했는지
    votes         []*Vote         // 인덱스 = 검증자 인덱스
    sum           int64           // 총 투표력

    // 블록별 투표력 추적
    votesByBlock  map[string]*blockVotes

    // +2/3 달성 여부
    maj23         *BlockID  // nil이 아니면 +2/3 달성
}

// +2/3 확인 메서드
func (voteSet *VoteSet) TwoThirdsMajority() (BlockID, bool) {
    if voteSet == nil {
        return BlockID{}, false
    }
    voteSet.mtx.Lock()
    defer voteSet.mtx.Unlock()
    
    if voteSet.maj23 != nil {
        return *voteSet.maj23, true
    }
    return BlockID{}, false
}

// +2/3 Any (nil 포함 아무거나) 확인
func (voteSet *VoteSet) HasTwoThirdsAny() bool {
    if voteSet == nil {
        return false
    }
    voteSet.mtx.Lock()
    defer voteSet.mtx.Unlock()
    
    return voteSet.sum > voteSet.valSet.TotalVotingPower()*2/3
}
```

### HeightVoteSet - 높이별 모든 투표

**consensus/types/height_vote_set.go**

```go
// HeightVoteSet은 한 높이의 모든 라운드 투표를 관리
type HeightVoteSet struct {
    chainID string
    height  int64
    valSet  *types.ValidatorSet

    mtx               sync.Mutex
    round             int32             // 알려진 최대 라운드
    roundVoteSets     map[int32]RoundVoteSet  // 라운드별 VoteSet
    peerCatchupRounds map[p2p.ID][]int32 // 피어별 캐치업 라운드
}

type RoundVoteSet struct {
    Prevotes   *types.VoteSet
    Precommits *types.VoteSet
}

// 특정 라운드의 Prevote 가져오기
func (hvs *HeightVoteSet) Prevotes(round int32) *types.VoteSet {
    hvs.mtx.Lock()
    defer hvs.mtx.Unlock()
    return hvs.getVoteSet(round, types.PrevoteType)
}
```

---

## 4. 컨센서스 흐름

### receiveRoutine - 메인 이벤트 루프

**consensus/state.go**

```go
// receiveRoutine은 컨센서스의 심장부입니다
func (cs *State) receiveRoutine(maxSteps int) {
    defer func() {
        if r := recover(); r != nil {
            cs.Logger.Error("CONSENSUS FAILURE!", "err", r)
        }
    }()

    for {
        rs := cs.RoundState
        var mi msgInfo

        select {
        // 1. 트랜잭션 사용 가능 알림
        case <-cs.txNotifier.TxsAvailable():
            cs.handleTxsAvailable()

        // 2. 피어로부터 메시지 (Proposal, Vote 등)
        case mi = <-cs.peerMsgQueue:
            cs.wal.Write(mi)      // WAL에 기록 (비동기)
            cs.handleMsg(mi)

        // 3. 내부 메시지 (자신의 투표)
        case mi = <-cs.internalMsgQueue:
            cs.wal.WriteSync(mi)  // WAL에 기록 (동기 - fsync!)
            cs.handleMsg(mi)

        // 4. 타임아웃 발생
        case ti := <-cs.timeoutTicker.Chan():
            cs.wal.Write(ti)
            cs.handleTimeout(ti, rs)

        // 5. 종료 신호
        case <-cs.Quit():
            return
        }
    }
}
```

### 상태 전이 함수들

#### enterNewRound - 새 라운드 시작

```go
func (cs *State) enterNewRound(height int64, round int32) {
    // 상태 검증
    if cs.Height != height || round < cs.Round || 
       (cs.Round == round && cs.Step != RoundStepNewHeight) {
        return
    }

    cs.Logger.Info("enterNewRound", 
        "height", height, 
        "round", round)

    // 라운드 상태 업데이트
    cs.Round = round
    cs.Step = RoundStepNewRound
    cs.Proposal = nil
    cs.ProposalBlock = nil
    cs.ProposalBlockParts = nil

    // 새 라운드의 제안자 결정
    cs.Validators.IncrementProposerPriority(round)
    cs.Proposer = cs.Validators.GetProposer()

    // 즉시 Propose 단계로
    cs.enterPropose(height, round)
}
```

#### enterPropose - 블록 제안

```go
func (cs *State) enterPropose(height int64, round int32) {
    // Propose 타임아웃 설정
    cs.scheduleTimeout(cs.config.Propose(round), height, round, RoundStepPropose)

    cs.Step = RoundStepPropose

    // 내가 이 라운드의 제안자인가?
    if cs.isProposer(cs.privValidatorPubKey.Address()) {
        cs.Logger.Info("enterPropose: Our turn to propose")
        cs.decideProposal(height, round)
    } else {
        cs.Logger.Info("enterPropose: Not our turn",
            "proposer", cs.Proposer.Address)
    }
}

func (cs *State) decideProposal(height int64, round int32) {
    var block *types.Block
    var blockParts *types.PartSet

    // ValidBlock이 있으면 그것 사용 (이전 라운드에서 +2/3 Prevote 받은 블록)
    if cs.ValidBlock != nil {
        block = cs.ValidBlock
        blockParts = cs.ValidBlockParts
    } else {
        // 새 블록 생성 (PrepareProposal 호출)
        block, blockParts = cs.createProposalBlock()
    }

    // Proposal 생성 및 서명
    proposal := types.NewProposal(height, round, cs.ValidRound, blockParts.Header())
    cs.privValidator.SignProposal(cs.state.ChainID, proposal)

    // 브로드캐스트
    cs.sendInternalMessage(msgInfo{&ProposalMessage{proposal}, ""})
    for i := 0; i < int(blockParts.Total()); i++ {
        part := blockParts.GetPart(i)
        cs.sendInternalMessage(msgInfo{&BlockPartMessage{height, round, part}, ""})
    }
}
```

#### enterPrecommit - 2차 투표

```go
func (cs *State) enterPrecommit(height int64, round int32) {
    cs.Step = RoundStepPrecommit

    // Prevote 결과 확인
    blockID, ok := cs.Votes.Prevotes(round).TwoThirdsMajority()

    if !ok {
        // +2/3 Prevote 없음 → nil Precommit
        cs.signAddVote(PrecommitType, nil)
        return
    }

    // *** Vote Extension 처리 (v0.38+) ***
    if cs.state.ConsensusParams.ABCI.VoteExtensionsEnabled(height) {
        // ExtendVote 호출
        extension, err := cs.blockExec.ExtendVote(
            cs.ProposalBlock, 
            cs.state,
        )
        if err == nil {
            vote.Extension = extension
        }
    }

    // Lock 업데이트 및 Precommit
    if !blockID.IsNil() && cs.ProposalBlock.HashesTo(blockID.Hash) {
        cs.LockedRound = round
        cs.LockedBlock = cs.ProposalBlock
        cs.signAddVote(PrecommitType, blockID.Hash)
    } else {
        cs.LockedRound = -1
        cs.LockedBlock = nil
        cs.signAddVote(PrecommitType, nil)
    }
}
```

#### finalizeCommit - 블록 확정

```go
func (cs *State) finalizeCommit(height int64) {
    block := cs.ProposalBlock
    blockParts := cs.ProposalBlockParts

    // 1. 블록 저장
    cs.blockStore.SaveBlock(block, blockParts, cs.Votes.Precommits(cs.CommitRound))

    // 2. WAL에 EndHeight 기록
    cs.wal.WriteSync(EndHeightMessage{height})

    // 3. 블록 실행! (FinalizeBlock 호출)
    stateCopy, err := cs.blockExec.ApplyVerifiedBlock(
        cs.state,
        types.BlockID{Hash: block.Hash(), PartSetHeader: blockParts.Header()},
        block,
    )

    // 4. 상태 업데이트
    cs.updateToState(stateCopy)

    // 5. Mempool 업데이트
    cs.mempool.Update(...)

    // 6. 다음 높이로
    cs.scheduleRound0(&cs.RoundState)
}
```

### 전체 흐름 다이어그램

```
                         ┌─────────────────────────────────────────────────────┐
                         │                    Height H                         │
                         └─────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Round R                                                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────┐    ┌─────────────────────────────────────────────────────────────┐     │
│  │NewRound │ →  │                      PROPOSE                                │     │
│  └─────────┘    │  • 제안자: PrepareProposal → 블록 생성                       │     │
│                 │  • 비제안자: 타임아웃 대기 (ProposeTimeout)                   │     │
│                 └──────────────────────────┬──────────────────────────────────┘     │
│                                            │ 블록 수신 또는 타임아웃                 │
│                                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           PREVOTE (1차 투표)                                 │   │
│  │  • ProcessProposal로 블록 검증                                               │   │
│  │  • 검증 성공: 블록 해시에 투표 / 실패: nil 투표                                │   │
│  │  • Lock된 블록 있으면 그 블록에 투표                                          │   │
│  └────────────────────────────────┬────────────────────────────────────────────┘   │
│                                   │ 투표 수집                                       │
│                                   ▼                                                │
│            ┌──────────────────────────────────────────────────────┐                 │
│            │            +2/3 Prevotes 수집 완료?                   │                 │
│            └────────────────────────┬─────────────────────────────┘                 │
│                    │                │                  │                            │
│         +2/3 Block A        +2/3 nil          +2/3 미달성                           │
│                    │                │                  │                            │
│                    ▼                ▼                  ▼                            │
│              Lock Block A     Unlock            PrevoteWait                         │
│                    │                │           (타임아웃)                           │
│                    └────────────────┴──────────────────┘                            │
│                                   │                                                 │
│                                   ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         PRECOMMIT (2차 투표)                                 │   │
│  │  • +2/3 Prevote 결과 기반 투표                                               │   │
│  │  • ExtendVote로 Vote Extension 추가 (v0.38+)                                 │   │
│  └────────────────────────────────┬────────────────────────────────────────────┘   │
│                                   │ 투표 수집                                       │
│                                   ▼                                                │
│            ┌──────────────────────────────────────────────────────┐                 │
│            │          +2/3 Precommits 수집 완료?                   │                 │
│            └────────────────────────┬─────────────────────────────┘                 │
│                    │                │                  │                            │
│         +2/3 Block A         +2/3 nil         +2/3 미달성                          │
│                    │                │                  │                            │
│                    ▼                ▼                  ▼                            │
│               COMMIT!          다음 라운드        PrecommitWait                      │
│                    │           (Round R+1)        (타임아웃)                         │
│                    │                                   │                            │
│                    │                                   ▼                            │
│                    │                             다음 라운드                         │
│                    │                             (Round R+1)                        │
│                    ▼                                                                │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  COMMIT                                                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  1. 블록 저장 (BlockStore)                                                          │
│  2. FinalizeBlock 호출 → 트랜잭션 실행                                              │
│  3. Commit 호출 → 상태 영구 저장                                                     │
│  4. Mempool 업데이트 (실행된 TX 제거)                                                │
│  5. Height H+1로 이동                                                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. ABCI 2.0 (ABCI++)

ABCI 2.0은 애플리케이션이 컨센서스에 더 깊이 관여할 수 있게 합니다.

### PrepareProposal / ProcessProposal

| | PrepareProposal | ProcessProposal |
|---|---|---|
| **호출 시점** | 제안자가 블록을 만들 때 | 제안받은 블록 검증 시 |
| **역할** | TX 재정렬/추가/제거<br>수수료 기반 정렬<br>배치 최적화 | 블록 유효성 검증<br>Accept/Reject 결정<br>즉시 실행 (옵션) |
| **결정론적?** | 아니오 (비결정적 허용) | 예 (반드시 결정론적) |

#### PrepareProposal 예시

**abci/types/types.pb.go**

```go
type RequestPrepareProposal struct {
    MaxTxBytes           int64                  // 최대 TX 바이트
    Txs                  [][]byte              // Mempool의 TX들
    LocalLastCommit      ExtendedCommitInfo    // 이전 커밋 + Vote Extensions
    Misbehavior          []Misbehavior         // 악성 행위 증거
    Height               int64
    Time                 time.Time
    NextValidatorsHash   []byte
    ProposerAddress      []byte
}

type ResponsePrepareProposal struct {
    Txs [][]byte  // 수정된 TX 목록 (재정렬, 추가, 제거 가능)
}

// 예시: 수수료 기반 정렬
func (app *MyApp) PrepareProposal(
    req *RequestPrepareProposal,
) *ResponsePrepareProposal {
    txs := req.Txs
    
    // 수수료로 정렬
    sort.Slice(txs, func(i, j int) bool {
        feeI := extractFee(txs[i])
        feeJ := extractFee(txs[j])
        return feeI > feeJ  // 높은 수수료 우선
    })

    // Vote Extension 데이터를 "특별 TX"로 주입 가능
    if len(req.LocalLastCommit.Votes) > 0 {
        oracleData := aggregateVoteExtensions(req.LocalLastCommit)
        txs = append([][]byte{oracleData}, txs...)
    }

    return &ResponsePrepareProposal{Txs: txs}
}
```

### Vote Extensions (투표 확장)

#### Vote Extensions란?

검증자가 Precommit 투표에 추가 데이터를 첨부할 수 있는 기능입니다.

**사용 사례**: 오라클 가격 피드, 암호화된 멤풀, 임계값 서명 등

#### Vote Extension 흐름

```
Height H:
  ┌───────────────────────────────────────────────────────────┐
  │  Precommit 단계                                           │
  │                                                           │
  │  Validator A: Precommit + VoteExtension(price: $100)      │
  │  Validator B: Precommit + VoteExtension(price: $101)      │
  │  Validator C: Precommit + VoteExtension(price: $99)       │
  │                                                           │
  │  → 블록 커밋 (Extensions도 함께 저장)                      │
  └───────────────────────────────────────────────────────────┘
                              │
                              ▼
Height H+1:
  ┌───────────────────────────────────────────────────────────┐
  │  PrepareProposal (제안자만)                                │
  │                                                           │
  │  LocalLastCommit에서 Extensions 접근:                      │
  │  • Validator A: $100                                      │
  │  • Validator B: $101                                      │
  │  • Validator C: $99                                       │
  │                                                           │
  │  중앙값 계산: $100 → 블록에 "특별 TX"로 포함                │
  └───────────────────────────────────────────────────────────┘
```

#### 코드 예시

```go
// ExtendVote: Precommit 시 Vote Extension 생성
type RequestExtendVote struct {
    Hash             []byte   // 블록 해시
    Height           int64
    Time             time.Time
    Txs              [][]byte
    ProposedLastCommit CommitInfo
    Misbehavior      []Misbehavior
    NextValidatorsHash []byte
    ProposerAddress  []byte
}

type ResponseExtendVote struct {
    VoteExtension []byte  // 투표에 첨부할 데이터
}

// 예시: 오라클 가격 피드
func (app *OracleApp) ExtendVote(req *RequestExtendVote) *ResponseExtendVote {
    // 외부 API에서 가격 조회 (비결정론적 OK)
    price := fetchPriceFromAPI("ETH/USD")
    
    ext := &OraclePriceExtension{
        Price:     price,
        Timestamp: time.Now(),
    }
    
    extBytes, _ := proto.Marshal(ext)
    return &ResponseExtendVote{VoteExtension: extBytes}
}

// VerifyVoteExtension: 다른 검증자의 Extension 검증
func (app *OracleApp) VerifyVoteExtension(
    req *RequestVerifyVoteExtension,
) *ResponseVerifyVoteExtension {
    var ext OraclePriceExtension
    if err := proto.Unmarshal(req.VoteExtension, &ext); err != nil {
        return &ResponseVerifyVoteExtension{
            Status: ResponseVerifyVoteExtension_REJECT,
        }
    }
    
    // 가격이 합리적인 범위인지 확인 (결정론적이어야 함)
    if ext.Price < 0 || ext.Price > 1000000 {
        return &ResponseVerifyVoteExtension{
            Status: ResponseVerifyVoteExtension_REJECT,
        }
    }
    
    return &ResponseVerifyVoteExtension{
        Status: ResponseVerifyVoteExtension_ACCEPT,
    }
}
```

**주의사항**
- 크기 제한: 너무 크면 레이턴시 증가
- VerifyVoteExtension은 결정론적: 모든 노드가 같은 결과
- 너무 많은 REJECT는 Liveness 저하
- Vote Extension은 블록 크기에 영향: 검증자당 추가 데이터 발생
- Extension 크기는 합리적으로 유지 (권장: 수 KB 이내)
- 네트워크 대역폭 고려 필수

### FinalizeBlock

v0.38+에서 `BeginBlock + DeliverTx[] + EndBlock`이 `FinalizeBlock` 하나로 통합되었습니다.

**통합 이유**
- API 단순화: 3번 호출 → 1번 호출
- 성능 개선: 네트워크 왕복 횟수 감소
- 원자성: 블록 실행이 단일 트랜잭션으로 처리
- 유연성: 애플리케이션이 TX 실행 순서를 더 잘 제어 가능

**FinalizeBlock 구조**

```go
type RequestFinalizeBlock struct {
    Txs                  [][]byte           // 실행할 트랜잭션들
    DecidedLastCommit    CommitInfo        // 이전 높이 커밋 정보
    Misbehavior          []Misbehavior      // 악성 행위 증거 (이중 서명 등)
    Hash                 []byte             // 블록 해시 (검증용)
    Height               int64              // 블록 높이
    Time                 time.Time          // 블록 타임스탬프
    NextValidatorsHash   []byte            // 다음 검증자 세트 해시
    ProposerAddress      []byte            // 제안자 주소
}

type ResponseFinalizeBlock struct {
    Events               []Event            // 블록 레벨 이벤트
    TxResults            []ExecTxResult     // 각 TX 실행 결과
    ValidatorUpdates     []ValidatorUpdate  // 검증자 세트 변경
    ConsensusParamUpdates *ConsensusParams  // 컨센서스 파라미터 변경
    AppHash              []byte             // 앱 상태 해시 (Merkle Root)
}

type ExecTxResult struct {
    Code      uint32   // 0 = success, 1+ = error code
    Data      []byte   // TX 실행 결과 데이터
    Log       string   // 로그 메시지
    Info      string   // 추가 정보
    GasWanted int64    // 요청한 가스
    GasUsed   int64    // 실제 사용한 가스
    Events    []Event  // TX 레벨 이벤트
    Codespace string   // 에러 코드 네임스페이스
}
```

**구현 예시**

```go
// 예시 구현
func (app *MyApp) FinalizeBlock(
    req *RequestFinalizeBlock,
) *ResponseFinalizeBlock {
    // 1. 블록 시작 처리 (기존 BeginBlock 로직)
    app.logger.Info("FinalizeBlock", "height", req.Height, "txs", len(req.Txs))
    
    // 블록 레벨 이벤트
    blockEvents := []Event{
        {
            Type: "begin_block",
            Attributes: []EventAttribute{
                {Key: "height", Value: fmt.Sprintf("%d", req.Height)},
                {Key: "proposer", Value: hex.EncodeToString(req.ProposerAddress)},
            },
        },
    }
    
    // 2. 각 트랜잭션 실행 (기존 DeliverTx 로직)
    txResults := make([]ExecTxResult, len(req.Txs))
    
    for i, txBytes := range req.Txs {
        // 트랜잭션 디코딩
        tx, err := app.decodeTx(txBytes)
        if err != nil {
            txResults[i] = ExecTxResult{
                Code: 1,
                Log:  fmt.Sprintf("failed to decode tx: %v", err),
            }
            continue
        }
        
        // 트랜잭션 실행 (결정론적이어야 함!)
        result := app.executeTx(tx)
        txResults[i] = result
        
        // 상태 업데이트
        if result.Code == 0 {
            app.state.ApplyTx(tx)
        }
    }
    
    // 3. Evidence 처리 (악성 검증자 슬래싱)
    for _, evidence := range req.Misbehavior {
        app.handleEvidence(evidence)
        
        blockEvents = append(blockEvents, Event{
            Type: "evidence",
            Attributes: []EventAttribute{
                {Key: "type", Value: evidence.Type.String()},
                {Key: "validator", Value: hex.EncodeToString(evidence.Validator.Address)},
                {Key: "height", Value: fmt.Sprintf("%d", evidence.Height)},
            },
        })
    }
    
    // 4. 블록 종료 처리 (기존 EndBlock 로직)
    // 보상 분배, 인플레이션 등
    app.distributeRewards(req.DecidedLastCommit)
    
    // 검증자 세트 업데이트 (옵션)
    var validatorUpdates []ValidatorUpdate
    if app.shouldUpdateValidators() {
        validatorUpdates = app.getValidatorUpdates()
    }
    
    // 5. 앱 해시 계산
    appHash := app.state.ComputeHash()
    
    return &ResponseFinalizeBlock{
        Events:           blockEvents,
        TxResults:        txResults,
        ValidatorUpdates: validatorUpdates,
        AppHash:          appHash,
    }
}
```

**기존 버전과의 비교**

v0.34 (Legacy):
```
BeginBlock(height=100)
  → 블록 레벨 이벤트, 보상 등
DeliverTx(tx1)
  → TX 실행
DeliverTx(tx2)
  → TX 실행
...
EndBlock(height=100)
  → 검증자 업데이트, 컨센서스 파라미터 변경
Commit()
  → AppHash 반환
```

v0.38+ (ABCI 2.0):
```
FinalizeBlock(height=100, txs=[tx1, tx2, ...])
  → 모든 로직을 한 번에 처리
  → AppHash 포함하여 반환
Commit()
  → 상태를 디스크에 영구 저장 (AppHash는 FinalizeBlock에서 이미 반환)
```

**장점**
1. **성능**: gRPC 호출 횟수 감소 (n+2 → 2)
2. **원자성**: 블록 실행이 단일 작업으로 처리
3. **단순성**: 애플리케이션 코드가 더 간결
4. **유연성**: TX 간 의존성 처리가 더 쉬움

---

## 6. WAL (Write-Ahead Log)

WAL은 크래시 복구를 위한 핵심 메커니즘입니다. 모든 컨센서스 메시지가 처리되기 전에 디스크에 기록됩니다.

**consensus/wal.go**

```go
// WAL 메시지 타입
type WALMessage interface{}

type msgInfo struct {
    Msg    Message
    PeerID p2p.ID
}

type timeoutInfo struct {
    Duration time.Duration
    Height   int64
    Round    int32
    Step     RoundStepType
}

type EndHeightMessage struct {
    Height int64
}

// WAL 인터페이스
type WAL interface {
    Write(WALMessage) error      // 비동기 쓰기
    WriteSync(WALMessage) error  // 동기 쓰기 (fsync)
    FlushAndSync() error         // 버퍼 플러시
    
    SearchForEndHeight(height int64, options *WALSearchOptions) (
        *WALReader, bool, error)  // 복구용 검색
}
```

### WAL 복구 과정

```
1. 노드 크래시 발생
   ┌─────────────────────────────────────────┐
   │  Height 100, Round 0, Step Precommit    │
   │  내 Precommit 투표 전송 직후 크래시       │
   └─────────────────────────────────────────┘

2. 노드 재시작
   ┌─────────────────────────────────────────┐
   │  WAL 파일 읽기:                          │
   │  - msgInfo (Proposal)                   │
   │  - msgInfo (Prevote from peer1)         │
   │  - msgInfo (내 Prevote) ← WriteSync     │
   │  - msgInfo (Precommit from peer2)       │
   │  - msgInfo (내 Precommit) ← WriteSync   │
   │                                         │
   │  EndHeightMessage 없음 = 아직 커밋 안됨  │
   └─────────────────────────────────────────┘

3. 상태 복구
   ┌─────────────────────────────────────────┐
   │  WAL 메시지들을 순서대로 재실행           │
   │  → 크래시 직전 상태로 복구                │
   │  → 컨센서스 계속 진행                    │
   └─────────────────────────────────────────┘
```

**Write vs WriteSync 차이점**

- **Write**: 버퍼에 쓰기 (빠름, 피어 메시지용)
  - 메모리 버퍼에만 기록
  - 나중에 일괄 fsync (배치 처리)
  - 성능 최적화를 위해 사용
  - Proposal, Vote 수신 등에 사용

- **WriteSync**: 디스크에 fsync (느림, 자신의 투표용)
  - 즉시 디스크에 flush
  - 운영체제 캐시를 거치지 않고 물리 디스크까지 기록
  - 크래시 시에도 데이터 보장
  - **자신의 투표는 반드시 WriteSync 사용**
  - 이중 투표 방지를 위한 필수 메커니즘

**왜 자신의 투표만 WriteSync?**

자신의 투표는 다시 생성할 수 없고, 만약 다른 투표를 재전송하면 이중 서명(equivocation)이 됩니다. 따라서 크래시 전에 디스크에 기록되어야 합니다.

---

## 7. Proposer 선정 알고리즘

CometBFT는 가중 라운드 로빈(Weighted Round Robin) 알고리즘을 사용합니다.

**types/validator_set.go**

```go
// ValidatorSet은 검증자 집합과 제안자 선정을 관리
type ValidatorSet struct {
    Validators []*Validator
    Proposer   *Validator
    
    totalVotingPower int64
}

type Validator struct {
    Address          Address
    PubKey           crypto.PubKey
    VotingPower      int64   // 투표력 (스테이킹량)
    ProposerPriority int64   // 현재 우선순위
}

// 제안자 선정: 가중 라운드 로빈
func (vals *ValidatorSet) IncrementProposerPriority(times int32) {
    for i := int32(0); i < times; i++ {
        // 1. 모든 검증자의 우선순위에 VotingPower 추가
        for _, val := range vals.Validators {
            val.ProposerPriority += val.VotingPower
        }
        
        // 2. 가장 높은 우선순위 검증자가 제안자
        vals.Proposer = vals.findProposer()
        
        // 3. 제안자의 우선순위에서 TotalVotingPower 차감
        vals.Proposer.ProposerPriority -= vals.TotalVotingPower()
    }
}

func (vals *ValidatorSet) findProposer() *Validator {
    var proposer *Validator
    for _, val := range vals.Validators {
        if proposer == nil || val.ProposerPriority > proposer.ProposerPriority {
            proposer = val
        }
    }
    return proposer
}
```

### 예시

```
3 검증자 (투표력: A=4, B=3, C=2, 총합=9)

초기 상태: Priority = [0, 0, 0]

Round 0:
  +VotingPower: [4, 3, 2]
  Proposer: A (최고 우선순위)
  -Total: [4-9, 3, 2] = [-5, 3, 2]

Round 1:
  +VotingPower: [-5+4, 3+3, 2+2] = [-1, 6, 4]
  Proposer: B (최고 우선순위)
  -Total: [-1, 6-9, 4] = [-1, -3, 4]

Round 2:
  +VotingPower: [-1+4, -3+3, 4+2] = [3, 0, 6]
  Proposer: C (최고 우선순위)
  -Total: [3, 0, 6-9] = [3, 0, -3]

Round 3:
  +VotingPower: [3+4, 0+3, -3+2] = [7, 3, -1]
  Proposer: A (최고 우선순위)
  ...

→ 투표력에 비례하여 제안 기회 분배!
→ 결정론적: 모든 노드가 같은 제안자 계산
```

---

## 8. 타임아웃 설정

**config/config.go - ConsensusConfig**

```go
type ConsensusConfig struct {
    // Propose 타임아웃 (블록 제안 대기)
    TimeoutPropose        time.Duration  // 기본: 3s
    TimeoutProposeDelta   time.Duration  // 기본: 500ms (라운드당 증가)
    
    // Prevote 타임아웃 (+2/3 Any 후 나머지 대기)
    TimeoutPrevote        time.Duration  // 기본: 1s
    TimeoutPrevoteDelta   time.Duration  // 기본: 500ms
    
    // Precommit 타임아웃 (+2/3 Any 후 나머지 대기)
    TimeoutPrecommit      time.Duration  // 기본: 1s
    TimeoutPrecommitDelta time.Duration  // 기본: 500ms
    
    // Commit 타임아웃 (다음 높이 전 대기)
    TimeoutCommit         time.Duration  // 기본: 1s
}

// 라운드에 따른 타임아웃 계산
func (cfg *ConsensusConfig) Propose(round int32) time.Duration {
    return cfg.TimeoutPropose + 
           cfg.TimeoutProposeDelta*time.Duration(round)
}
```

### 타임아웃 요약

| 타임아웃 | 기본값 | 용도 | 발생 후 동작 |
|---------|--------|------|-------------|
| TimeoutPropose | 3s | 제안 대기 | nil Prevote 후 다음 단계 |
| TimeoutPrevote | 1s | +2/3 Any 후 나머지 대기 | Precommit 단계로 |
| TimeoutPrecommit | 1s | +2/3 Any 후 나머지 대기 | 다음 라운드로 |
| TimeoutCommit | 1s | 커밋 후 다음 높이 전 대기 | 피어 동기화 시간 |

### 타임아웃 튜닝 팁

**빠른 블록 시간 (Low Latency)**
- 목표: 1-3초 블록 시간
- 설정:
  ```
  TimeoutPropose = 1s
  TimeoutPrevote = 500ms
  TimeoutPrecommit = 500ms
  TimeoutCommit = 500ms
  ```
- 적용 환경: 좋은 네트워크, 적은 검증자 (<50개)

**느린 애플리케이션 (Heavy Computation)**
- 목표: PrepareProposal/ProcessProposal 시간 확보
- 설정:
  ```
  TimeoutPropose = 10s  ← 증가
  TimeoutPrevote = 2s
  TimeoutPrecommit = 2s
  TimeoutCommit = 2s
  ```
- 적용 환경: 복잡한 TX 검증, 대용량 블록

**불안정한 네트워크 (High Latency)**
- 목표: 네트워크 지연 보상
- 설정:
  ```
  TimeoutPropose = 5s
  TimeoutProposeDelta = 1s   ← Delta 증가
  TimeoutPrevote = 2s
  TimeoutPrevoteDelta = 1s   ← Delta 증가
  TimeoutPrecommit = 2s
  TimeoutPrecommitDelta = 1s ← Delta 증가
  TimeoutCommit = 3s
  ```
- 적용 환경: 글로벌 분산 네트워크, 높은 레이턴시

**많은 검증자 (Large Validator Set)**
- 목표: 투표 수집 시간 확보
- 설정:
  ```
  TimeoutPropose = 5s
  TimeoutPrevote = 3s        ← 증가
  TimeoutPrecommit = 3s      ← 증가
  TimeoutCommit = 2s
  ```
- 적용 환경: 100+ 검증자

**실전 팁**
1. **모니터링 필수**: 각 단계별 실제 소요 시간 측정
2. **점진적 조정**: 작은 변경 후 효과 관찰
3. **라운드 진행 확인**: 라운드가 자주 증가하면 타임아웃 부족 신호
4. **네트워크 모니터링**: P2P 메시지 전파 시간 측정
5. **Delta 활용**: 라운드가 증가할수록 자동으로 타임아웃 증가

---

## 9. Evidence 처리

Evidence는 검증자의 악성 행위 (주로 이중 서명) 증거입니다.

**types/evidence.go**

```go
// Evidence 인터페이스
type Evidence interface {
    Height() int64          // 발생 높이
    Bytes() []byte         // 직렬화
    Hash() []byte          // 해시
    ValidateBasic() error  // 기본 검증
    String() string
}

// DuplicateVoteEvidence: 같은 (H,R)에서 다른 블록에 투표
type DuplicateVoteEvidence struct {
    VoteA            *Vote
    VoteB            *Vote
    TotalVotingPower int64
    ValidatorPower   int64
    Timestamp        time.Time
}

// LightClientAttackEvidence: 라이트 클라이언트 공격
type LightClientAttackEvidence struct {
    ConflictingBlock *LightBlock
    CommonHeight     int64
}
```

### 이중 투표 감지

```
Height 100, Round 0:

검증자 X가 두 개의 다른 Prevote 전송:

  ┌────────────────────────────────────────────────────────┐
  │  VoteA: Prevote for Block A                           │
  │    Height: 100, Round: 0                              │
  │    BlockID: 0xAAA...                                   │
  │    Signature: sig_A                                    │
  └────────────────────────────────────────────────────────┘
  
  ┌────────────────────────────────────────────────────────┐
  │  VoteB: Prevote for Block B (다른 블록!)              │
  │    Height: 100, Round: 0  (같은 H,R!)                 │
  │    BlockID: 0xBBB...                                   │
  │    Signature: sig_B                                    │
  └────────────────────────────────────────────────────────┘

감지 노드:
  → DuplicateVoteEvidence 생성
  → Evidence Pool에 추가
  → 다음 블록에 포함
  → 애플리케이션에서 슬래싱 처리
```

### Evidence Pool

**evidence/pool.go**

```go
// EvidencePool은 검증된 Evidence를 관리
type Pool struct {
    evidenceStore   dbm.DB
    evidenceList    *clist.CList      // 대기 중인 Evidence
    
    state           sm.State
    stateDB         sm.Store
    blockStore      *store.BlockStore
}

// Evidence 추가
func (evpool *Pool) AddEvidence(ev types.Evidence) error {
    // 1. 기본 검증
    if err := ev.ValidateBasic(); err != nil {
        return err
    }
    
    // 2. 이미 존재하는지 확인
    if evpool.isPending(ev) {
        return ErrEvidenceAlreadyStored
    }
    
    // 3. 유효 기간 확인 (MaxAgeNumBlocks, MaxAgeDuration)
    if !evpool.isWithinMaxAge(ev) {
        return ErrEvidenceTooOld
    }
    
    // 4. 검증자가 실제로 해당 높이에 존재했는지 확인
    if err := evpool.verify(ev); err != nil {
        return err
    }
    
    // 5. Pool에 추가
    evpool.evidenceList.PushBack(ev)
    return nil
}
```

---

## 10. 블록 조각과 P2P

### PartSet - 블록 조각화

**types/part_set.go**

```go
const (
    BlockPartSizeBytes = 65536  // 64KB
)

type PartSet struct {
    total uint32              // 총 조각 수
    hash  []byte              // Merkle Root
    
    mtx           sync.Mutex
    parts         []*Part
    partsBitArray *bits.BitArray  // 어떤 조각을 가지고 있는지
    count         uint32          // 현재 가진 조각 수
}

type Part struct {
    Index uint32        // 조각 인덱스
    Bytes []byte       // 조각 데이터
    Proof merkle.Proof // Merkle 증명
}

// 블록을 조각으로 분할
func NewPartSetFromData(data []byte, partSize uint32) *PartSet {
    total := (uint32(len(data)) + partSize - 1) / partSize
    parts := make([]*Part, total)
    partsBytes := make([][]byte, total)
    
    for i := uint32(0); i < total; i++ {
        start := i * partSize
        end := min(uint32(len(data)), (i+1)*partSize)
        
        part := &Part{Index: i, Bytes: data[start:end]}
        parts[i] = part
        partsBytes[i] = part.Bytes
    }
    
    // Merkle Root 및 각 조각의 Proof 생성
    root, proofs := merkle.ProofsFromByteSlices(partsBytes)
    for i := 0; i < int(total); i++ {
        parts[i].Proof = *proofs[i]
    }
    
    return &PartSet{total: total, hash: root, parts: parts}
}
```

### P2P 동시성 - MConnection

**p2p/conn/connection.go**

```go
// MConnection: 멀티플렉스 연결
type MConnection struct {
    conn        net.Conn
    bufConnReader *bufio.Reader
    bufConnWriter *bufio.Writer
    
    channels    []*Channel       // 채널별 큐
    channelsIdx map[byte]*Channel
    
    send        chan struct{}    // 전송 신호
    pong        chan struct{}    // Pong 응답
}

// 각 연결마다 2개 고루틴
func (c *MConnection) OnStart() error {
    go c.sendRoutine()  // 송신 고루틴
    go c.recvRoutine()  // 수신 고루틴
    return nil
}

// 송신 루틴
func (c *MConnection) sendRoutine() {
    for {
        select {
        case <-c.send:
            // 각 채널에서 패킷 가져와서 전송
            for _, ch := range c.channels {
                if ch.canSend() {
                    pkt := ch.nextPacket()
                    c.sendPacket(pkt)
                }
            }
        case <-c.quit:
            return
        }
    }
}

// 수신 루틴
func (c *MConnection) recvRoutine() {
    for {
        pkt, err := c.recvPacket()
        if err != nil {
            c.stopForError(err)
            return
        }
        
        // 채널에 따라 분배
        ch := c.channelsIdx[pkt.ChannelID]
        ch.recvPacket(pkt)
    }
}
```

### P2P 채널 구조

```
┌────────────────────────────────────────────────────────────────┐
│                        MConnection                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Channel 0x20 (StateChannel)                                   │
│  └─ 컨센서스 상태 메시지 (NewRoundStep, HasVote 등)             │
│                                                                │
│  Channel 0x21 (DataChannel)                                    │
│  └─ 블록 조각 (BlockPart) 전송                                 │
│                                                                │
│  Channel 0x22 (VoteChannel)                                    │
│  └─ 투표 메시지 (Prevote, Precommit)                           │
│                                                                │
│  Channel 0x23 (VoteSetBitsChannel)                             │
│  └─ 투표 비트맵 (어떤 투표를 가지고 있는지)                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘

100개 피어 연결 시:
• 200개 고루틴 (각 연결당 send/recv 2개)
• 각 채널은 독립적으로 병렬 전송
```

---

## 11. Mempool

**mempool/clist_mempool.go**

```go
// CListMempool: 기본 Mempool 구현 (연결 리스트 기반)
type CListMempool struct {
    config *config.MempoolConfig
    
    proxyAppConn proxy.AppConnMempool  // ABCI 연결
    
    txs          *clist.CList   // TX 연결 리스트 (순서 유지)
    txsMap       map[TxKey]*clist.CElement  // 빠른 검색용
    txsBytes     int64          // 총 TX 바이트
    
    cache        TxCache        // 이미 본 TX 캐시
    
    height       int64          // 현재 높이
    notifiedTxsAvailable bool   // TX 있음 알림 여부
}

// CheckTx: TX 검증 후 Mempool에 추가
func (mem *CListMempool) CheckTx(tx types.Tx) (*abciclient.ReqRes, error) {
    // 1. 크기 체크
    if len(tx) > mem.config.MaxTxBytes {
        return nil, ErrTxTooLarge
    }
    
    // 2. Mempool 용량 체크
    if mem.txsBytes+int64(len(tx)) > mem.config.MaxTxsBytes {
        return nil, ErrMempoolIsFull
    }
    
    // 3. 캐시 체크 (이미 본 TX?)
    if !mem.cache.Push(tx) {
        return nil, ErrTxInCache
    }
    
    // 4. ABCI CheckTx 호출
    reqRes, err := mem.proxyAppConn.CheckTxAsync(
        context.TODO(),
        &abci.RequestCheckTx{Tx: tx},
    )
    
    return reqRes, err
}

// ReapMaxBytesMaxGas: 블록에 포함할 TX 가져오기
func (mem *CListMempool) ReapMaxBytesMaxGas(
    maxBytes, maxGas int64,
) types.Txs {
    mem.updateMtx.RLock()
    defer mem.updateMtx.RUnlock()
    
    var (
        totalBytes int64
        totalGas   int64
        txs        []types.Tx
    )
    
    for e := mem.txs.Front(); e != nil; e = e.Next() {
        memTx := e.Value.(*mempoolTx)
        
        // 크기/가스 제한 체크
        if totalBytes+int64(len(memTx.tx)) > maxBytes {
            break
        }
        if maxGas > 0 && totalGas+memTx.gasWanted > maxGas {
            continue
        }
        
        totalBytes += int64(len(memTx.tx))
        totalGas += memTx.gasWanted
        txs = append(txs, memTx.tx)
    }
    
    return txs
}

// Update: 블록 실행 후 Mempool 업데이트
func (mem *CListMempool) Update(
    height int64,
    txs types.Txs,
    txResults []*abci.ExecTxResult,
    preCheck PreCheckFunc,
    postCheck PostCheckFunc,
) error {
    mem.updateMtx.Lock()
    defer mem.updateMtx.Unlock()
    
    // 실행된 TX 제거
    for i, tx := range txs {
        if e, ok := mem.txsMap[TxKey(tx)]; ok {
            mem.removeTx(tx, e, false)
        }
    }
    
    mem.height = height
    
    // 남은 TX 재검증 (옵션)
    if mem.config.Recheck {
        mem.recheckTxs()
    }
    
    return nil
}
```

### 우선순위 Mempool (v0.37+)

**mempool/v1/mempool.go**

```go
// TxPriority는 TX의 우선순위를 나타냅니다
type TxPriority struct {
    Priority int64   // 우선순위 (높을수록 먼저)
    Nonce    uint64  // 같은 발신자의 TX 순서
    Sender   string  // 발신자 주소
}

// 우선순위 큐 기반 Mempool
type PriorityMempool struct {
    txs *TxPriorityQueue  // 힙 기반 우선순위 큐
    // ...
}

// 애플리케이션에서 우선순위 설정 (CheckTx 응답)
type ResponseCheckTx struct {
    Code      uint32
    Data      []byte
    Log       string
    GasWanted int64
    GasUsed   int64
    Priority  int64   // ← 우선순위!
    Sender    string  // ← 발신자
}
```

---

## 12. 정리

### 핵심 컴포넌트 요약

| 컴포넌트 | 파일 위치 | 역할 | 핵심 포인트 |
|---------|----------|------|------------|
| **State** | consensus/state.go | 컨센서스 상태 머신 | enterPropose, enterPrevote, enterPrecommit |
| **VoteSet** | types/vote_set.go | 투표 수집 및 집계 | TwoThirdsMajority, HasTwoThirdsAny |
| **WAL** | consensus/wal.go | 크래시 복구 | Write vs WriteSync |
| **ABCI** | abci/types/ | 앱 인터페이스 | PrepareProposal, ProcessProposal, FinalizeBlock |
| **Evidence** | evidence/pool.go | 악성 행위 감지 | DuplicateVoteEvidence |
| **Mempool** | mempool/ | TX 대기열 | CheckTx, ReapMaxBytesMaxGas, Update |

### ABCI 2.0 메서드 호출 순서

1. **InitChain** - 제네시스 시 1회 호출. 초기 검증자 설정.
2. **PrepareProposal (제안자만)** - 블록 제안 전. TX 재정렬/추가/제거 가능. 비결정론적 OK.
3. **ProcessProposal (검증자)** - 제안받은 블록 검증. Accept/Reject. 결정론적 필수!
4. **ExtendVote (검증자)** - Precommit 시. Vote Extension 생성. 비결정론적 OK.
5. **VerifyVoteExtension (검증자)** - 다른 검증자의 Extension 검증. 결정론적 필수!
6. **FinalizeBlock** - 블록 확정 시. TX 실행. 결정론적 필수!
7. **Commit** - 상태 영구 저장. AppHash 반환.

### Safety vs Liveness 트레이드오프

**CometBFT는 Safety를 우선합니다.**

- **Safety**: 1/3 이상의 악성 노드가 있어도 포크 불가능
- **Liveness**: 1/3 이상의 악성 노드가 체인을 멈출 수 있음

포크보다 멈춤이 낫다는 철학입니다.

### 실전 개발 가이드

#### 1. ABCI 애플리케이션 개발 시 주의사항

**결정론적 실행 필수**
```go
// 나쁜 예: 비결정론적
func (app *MyApp) ProcessProposal(req *RequestProcessProposal) *ResponseProcessProposal {
    // 현재 시간 사용 - 노드마다 다를 수 있음
    if time.Now().Unix() > deadline {
        return &ResponseProcessProposal{Status: ResponseProcessProposal_REJECT}
    }
    // ...
}

// 좋은 예: 결정론적
func (app *MyApp) ProcessProposal(req *RequestProcessProposal) *ResponseProcessProposal {
    // 블록 타임스탬프 사용 - 모든 노드가 같음
    if req.Time.Unix() > deadline {
        return &ResponseProcessProposal{Status: ResponseProcessProposal_REJECT}
    }
    // ...
}
```

**상태 머신 복제**
- 같은 입력(블록)이면 같은 출력(AppHash)
- 난수, 시간, 외부 API 호출 금지
- 맵 순회 시 키 정렬 필수

#### 2. 디버깅 방법

**로그 레벨 설정**
```toml
# config.toml
[log]
level = "consensus:debug,state:info,*:error"
```

**유용한 로그 메시지**
- `enterNewRound`: 라운드 진행 추적
- `addVote`: 투표 수집 상황
- `finalizeCommit`: 블록 확정 확인
- `applyBlock`: 블록 실행 시간

**메트릭 모니터링**
```
# Prometheus 엔드포인트: http://localhost:26660/metrics

tendermint_consensus_height          # 현재 블록 높이
tendermint_consensus_rounds          # 라운드 횟수 (많으면 문제)
tendermint_consensus_validators      # 검증자 수
tendermint_mempool_size              # Mempool TX 수
tendermint_p2p_peers                 # 연결된 피어 수
```

#### 3. 성능 최적화

**Mempool 설정**
```toml
[mempool]
size = 5000                    # Mempool 크기
cache_size = 10000            # TX 캐시
max_tx_bytes = 1048576        # 1MB per TX
max_txs_bytes = 1073741824    # 1GB total
```

**합의 설정**
```toml
[consensus]
timeout_propose = 3s
timeout_propose_delta = 500ms
timeout_prevote = 1s
timeout_prevote_delta = 500ms
timeout_precommit = 1s
timeout_precommit_delta = 500ms
timeout_commit = 1s

# 빠른 블록 (네트워크 좋을 때)
# timeout_commit = 100ms
```

**P2P 설정**
```toml
[p2p]
max_num_inbound_peers = 40
max_num_outbound_peers = 10
send_rate = 5120000           # 5MB/s
recv_rate = 5120000           # 5MB/s
```

#### 4. 일반적인 문제 해결

**문제: 라운드가 계속 증가**
```
원인: 타임아웃 부족, 네트워크 지연, 느린 애플리케이션
해결: 
  1. 타임아웃 증가
  2. PrepareProposal/ProcessProposal 최적화
  3. 네트워크 상태 점검
```

**문제: 블록 시간이 불규칙**
```
원인: 제안자가 자주 바뀜, 일부 검증자 오프라인
해결:
  1. 검증자 모니터링 강화
  2. TimeoutCommit 조정
  3. 피어 연결 상태 확인
```

**문제: Mempool이 가득 참**
```
원인: 블록 크기 부족, TX 처리 속도 느림
해결:
  1. ConsensusParams.Block.MaxBytes 증가
  2. 애플리케이션 TX 처리 최적화
  3. Mempool 크기 증가
```

**문제: 상태 동기화 실패**
```
원인: 블록 저장소 손상, AppHash 불일치
해결:
  1. unsafe-reset-all로 초기화 (주의!)
  2. State sync 사용
  3. 스냅샷에서 복구
```

#### 5. 보안 권장사항

**키 관리**
- 검증자 키는 하드웨어 보안 모듈(HSM) 사용
- priv_validator_key.json 암호화 저장
- 정기적인 키 로테이션

**네트워크 보안**
- 센트리 노드 아키텍처 사용
- 검증자는 공개 IP 노출 금지
- VPN 또는 프라이빗 네트워크 사용

**모니터링**
- 이중 서명 감지 시스템
- 투표 참여율 모니터링
- 블록 생성 시간 알람

#### 6. 업그레이드 전략

**소프트 포크 (후방 호환)**
```go
// 애플리케이션 버전 체크
func (app *MyApp) ProcessProposal(req *RequestProcessProposal) *ResponseProcessProposal {
    if req.Height >= UPGRADE_HEIGHT {
        // 새 로직
        return app.processProposalV2(req)
    }
    // 기존 로직
    return app.processProposalV1(req)
}
```

**하드 포크 (비호환 변경)**
1. 거버넌스 제안 통과
2. 업그레이드 높이 합의
3. 모든 검증자 동시 업그레이드
4. 높이 도달 시 자동 전환

### 핵심 원칙 요약

1. **결정론적 실행**: 같은 블록 = 같은 AppHash
2. **Safety First**: 포크보다는 멈춤
3. **2/3 다수결**: 비잔틴 허용 한계
4. **즉시 최종성**: 커밋 = 되돌릴 수 없음
5. **WAL 활용**: 크래시 복구 보장

### 다음 단계 학습

- **Cosmos SDK**: IBC, Staking, Governance 모듈
- **IBC**: 크로스체인 통신 프로토콜
- **CosmWasm**: 스마트 컨트랙트 프레임워크
- **Ignite CLI**: 블록체인 개발 도구

---

## 참고 자료

- [CometBFT 공식 문서](https://docs.cometbft.com)
- [CometBFT GitHub](https://github.com/cometbft/cometbft)
- [Tendermint 논문](https://arxiv.org/abs/1807.04938)
- [ABCI 2.0 스펙](https://docs.cometbft.com/main/spec/abci/)

---

**마지막 업데이트**: 2025년 | **기준 버전**: CometBFT v0.38+ / v1.0

이 문서는 Cosmos 생태계 학습을 위해 작성되었습니다.
