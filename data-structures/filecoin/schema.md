# Filecoin Data Structures

Schemas are grouped by their serialized blocks. Other than those types listed in "Basic Types" and "Crypto Types", each grouping of schema types in a code block represents a data structure that is serialized into a single IPLD block with its own Link (CID).

Advanced Data Layouts (ADLs) are shown in their expanded form here, as the data appears on-block. Their logical forms for programmatic purposes are `Map` for the HAMT and `List` for the AMT.

There are some data structures that are repeats of the same forms, primarily the AMT and HAMTs that share the same data types. They are not de-duplicated here for clarity to demonstrate the different purposes of those data structures.

For more information about the IPLD Schema language, see the [specificaiton](https://specs.ipld.io/schemas/).

## Basic Types

```ipldsch
# Address is a `https://github.com/filecoin-project/go-address` in its binary byte format.
# It may be used as a hamt key, and will be often visualized or presented to users
# in its more readable string form.
type Address bytes

# Go big.Int
# Prefer presenting to users either as a number or a string view of the decimal number
# for readability.
type BigInt bytes

# An indicator of which RPC method on an actor a message should trigger execution of.
type MethodNum int

# The 'f0...' subset of Addresses used for the actual indexes of actors in a state root.
type ActorID int

# the height of a block in the chain. Should fit in an Int64
type ChainEpoch int

type TokenAmount BigInt

type PaddedPieceSize int

type PeerID bytes

type SectorSize int

type SectorNumber int

type PartitionNumber int

type BitField bytes

type StoragePower BigInt

type DealID int

type DealWeight BigInt

type Multiaddr bytes

type RegisteredSealProof int

type TransactionID # TxnID
```

## Crypto Types

```ipldsch
type Signature union {
  SignatureBLS 0
  SignatureSecp256k1 1
} representation byteprefix
```

## Genesis Block

```ipldsch
type Genesis struct {
  Datetime String
  Network String
  Token String
  TokenAmounts TokenAmounts
  Message String
}

type TokenAmounts struct {
  TotalSupply String
  Miners String
  ProtocolLabs {String:String}
}
```

## Chain

```ipldsch
type BlockHeader struct {
  Miner Address
  Ticket nullable Ticket
  ElectionProof nullable ElectionProof
  BeaconEntries [BeaconEntry]
  WinPoStProof [PoStProof]
  Parents [&Any] # TODO: can we &BlockHeader this or is it a union with Genesis?
  ParentWeight Bytes
  Height ChainEpoch
  ParentStateRoot &StateRoot
  ParentMessageReceipts &MessageReceiptAMT
  Messages &TxMeta
  BLSAggregate nullable Signature
  Timestamp Int
  BlockSig nullable Signature
  ForkSignaling Int
  ParentBaseFee TokenAmount
} representation tuple

type Ticket struct {
  VRFProof Bytes
} representation tuple

type ElectionProof struct {
  WinCount Int
  VRFProof Bytes
} representation tuple

type BeaconEntry struct {
  Round Int
  Data Bytes
} representation tuple

type PoStProof struct {
  PoStProof Int
  ProofBytes Bytes
} representation tuple
```

**AMT**: This is an ADL representing `type MessageReceiptList [MessageReceipt]`.

```ipldsch
type ExitCode int

```ipldsch
type MessageReceiptAMT struct {
  height Int
  count Int
  node MessageReceiptAMTNode
} representation tuple

# This can also be the root of a block
type MessageReceiptAMTNode struct {
  bitmap Bytes
  children [&MessageReceiptAMTNode]
  values [&Message]
} representation tuple

type MessageReceipt struct {
  ExitCode ExitCode
  Return   Bytes
  GasUsed  Int
} representation tuple
```

```ipldsch
type StateRoot struct {
  Version Int
  Actors &ActorsHAMT
  Info &StateInfo
} representation tuple
```

Note that `StateInfo` is an [empty struct](https://github.com/filecoin-project/lotus/blob/f225b7b928de73e20dd8b8d4d075a56fbf3b8a25/chain/types/state.go#L26), which encodes as an empty CBOR array (`0x80`).

```ipldsch
type StateInfo struct {
} representation tuple
```

## Messages

```ipldsch
type TxMeta struct {
  BlsMessages &MessageLinkAMT
  SecpkMessages &SignedMessageLinkAMT
} representation tuple

```ipldsch
type Message struct {
  Version Int
  To Address
  From Address
  Nonce Int
  Value BigInt
  GasLimit Int
  GasFeeCap BigInt
  GasPremium BigInt
  Method MethodNum
  Params Bytes
} representation tuple

type SignedMessage struct {
  Message Message
  Signature Signature
} representation tuple
```

**AMT**: This is an ADL representing `type MessagesList [&Message]`.

```ipldsch
type MessageLinkAMT struct {
  height Int
  count Int
  node MessageLinkAMTNode
} representation tuple

# This can also be the root of a block
type MessageLinkAMTNode struct {
  bitmap Bytes
  children [&MessageLinkAMTNode]
  values [&Message]
} representation tuple
```

**AMT**: This is an ADL representing `type SignedMessagesList [&SignedMessage]`.

Note this is identical in layout to `MessagesList` and contains links at the leaves, but is listed here for completeness.

```ipldsch
type SignedMessageLinkAMT struct {
  height Int
  count Int
  node SignedMessageLinkAMTNode
} representation tuple

# This can also be the root of a block
type SignedMessageLinkAMTNode struct {
  bitmap Bytes
  children [&SignedMessageLinkAMTNode]
  values [&SignedMessage]
} representation tuple
```

## Actors

**HAMT**: This is an ADL representing `type ActorsMap {AddressString:Actors}`, where `AddressString` is the string form of the raw bytes of the `Address`.

```ipldsch
type ActorsHAMT struct {
  map Bytes
  data [ ActorsHAMTElement ]
} representation tuple

type ActorsHAMTElement union {
  | ActorsHAMTLink "0"
  | ActorsHAMTBucket "1"
} representation keyed

type ActorsHAMTLink &ActorsHAMT

type ActorsHAMTBucket [ ActorsHAMTBucketEntry ]

type ActorsHAMTBucketEntry struct {
  key Bytes # Address bytes
  value Actor # inline
} representation tuple

type Actor struct {
  code &Any # An inline CID encoded as raw+identity
  head &Any # An implicit union of each actor type, keyed by `code` here
  nonce CallSeqNum
  balance Bytes
} representation tuple
```

### InitActor

```ipldsch
type InitV0State struct {
  AddressMap &ActorIDHAMT
  NextID ActorID
  NetworkName String
} representation tuple
```

**HAMT**: This is an ADL representing `type ActorIDHAMT {AddressString:ActorID}`, where `AddressString` is the string form of the raw bytes of the `Address`.

```ipldsch
type ActorIDHAMT struct {
  map Bytes
  data [ ActorIDHAMTElement ]
} representation tuple

type ActorIDHAMTElement union {
  | ActorIDHAMTLink "0"
  | ActorIDHAMTBucket "1"
} representation keyed

type ActorIDHAMTLink &ActorIDHAMT

type ActorIDHAMTBucket [ ActorIDHAMTBucketEntry ]

type ActorIDHAMTBucketEntry struct {
  key Address
  value ActorID
} representation tuple
```

### CronActor

```ipldsch
type CronV0State struct {
  Entries [CronV0Entry]
} representation tuple

type CronV0Entry struct {
  Receiver Address
  MethodNum MethodNum
} representation tuple
```

### RewardActor

**v0**

```ipldsch
type Spacetime bytes

type RewardV0State struct {
  CumsumBaseline Spacetime
  CumsumRealized Spacetime
  EffectiveNetworkTime ChainEpoch
  EffectiveBaselinePower StoragePower
  ThisEpochReward TokenAmount
  ThisEpochRewardSmoothed nullable V0FilterEstimate
  ThisEpochBaselinePower StoragePower
  Epoch ChainEpoch
  TotalMined TokenAmount
} representation tuple

type V0FilterEstimate struct {
  PositionEstimate BigInt
  VelocityEstimate BigInt
} representation tuple
```

**v2**

```ipldsch
type RewardV2State struct {
  CumsumBaseline Spacetime
  CumsumRealized Spacetime
  EffectiveNetworkTime ChainEpoch
  EffectiveBaselinePower StoragePower
  ThisEpochReward TokenAmount
  ThisEpochRewardSmoothed V0FilterEstimate
  ThisEpochBaselinePower StoragePower
  Epoch ChainEpoch
  TotalStoragePowerReward TokenAmount
  SimpleTotal BigInt
  BaselineTotal BigInt
} representation tuple
```

### AccountActor

```ipldsch
type AccountV0State struct {
  Address Address
} representation tuple
```

### StorageMarketActor

```ipldsch
type MarketV0State struct {
  Proposals &DealProposalAMT # AMT[DealID]DealProposal
  States &DealStateAMT # AMT[DealID]DealState
  PendingProposals &MarketV0DealProposalHAMT # HAMT[DealCid]DealProposal
  EscrowTable &BalanceTableHAMT # HAMT[Address]TokenAmount
  LockedTable &BalanceTableHAMT # HAMT[Address]TokenAmount
  NextID DealID
  DealOpsByEpoch &DealOpsByEpochHAMT # SetMultimap: HAMT[ChainEpoch]Set[DealID]
  LastCron ChainEpoch
  TotalClientLockedCollateral TokenAmount
  TotalProviderLockedCollateral TokenAmount
  TotalClientStorageFee TokenAmount
} representation tuple
```

**AMT**: This is an ADL representing `type DealProposalList [DealProposal]`, indexed by `DealID`.

```ipldsch
type DealProposalAMT struct {
  height Int
  count Int
  node DealProposalAMTNode
} representation tuple

type DealProposalAMTNode struct {
  bitmap Bytes
  children [&DealProposalAMTNode]
  values [MarketV0DealProposal] # inline
} representation tuple

type MarketV0DealProposal struct {
  PieceCID &Any # CommP: A CID with fil-commitment-unsealed + sha2_256-trunc254-padded
  PieceSize PaddedPieceSize
  VerifiedDeal Bool
  Client Address
  Provider Address
  Label String
  StartEpoch ChainEpoch
  EndEpoch ChainEpoch
  StoragePricePerEpoch TokenAmount
  ProviderCollateral TokenAmount
  ClientCollateral TokenAmount
} representation tuple
```

**AMT**: This is an ADL representing `type DealProposalList [DealState]`, indexed by `DealID`.

```ipldsch
type DealStateAMT struct {
  height Int
  count Int
  node DealStateAMTNode
} representation tuple

type DealStateAMTNode struct {
  bitmap Bytes
  children [&DealStateAMTNode]
  values [MarketV0DealState] # inline
} representation tuple

type MarketV0DealState struct {
  SectorStartEpoch ChainEpoch
  LastUpdatedEpoch ChainEpoch
  SlashEpoch ChainEpoch
} representation tuple
```

**HAMT**: This is an ADL representing `type ActorsMap {DealCidBytes:Actors}`.

```ipldsch
type DealCidBytes bytes

type MarketV0DealProposalHAMT struct {
  map Bytes
  data [ MarketV0DealProposalHAMTElement ]
} representation tuple

type MarketV0DealProposalHAMTElement union {
  | MarketV0DealProposalHAMTLink "0"
  | Bucket "1"
} representation keyed

type MarketV0DealProposalHAMTLink &MarketV0DealProposalHAMT

type MarketV0DealProposalHAMTBucket [ MarketV0DealProposalHAMTBucketEntry ]

type MarketV0DealProposalHAMTBucketEntry struct {
  key DealCidBytes
  value MarketV0DealProposal
} representation tuple

type MarketV0DealProposal struct {
  PieceCID &Any # A CID with fil-commitment-unsealed + sha2_256-trunc254-padded
  PieceSize PaddedPieceSize
  VerifiedDeal Bool
  Client Address
  Provider Address
  Label String
  StartEpoch ChainEpoch
  EndEpoch ChainEpoch
  StoragePricePerEpoch TokenAmount
  ProviderCollateral TokenAmount
  ClientCollateral TokenAmount
} representation tuple
```

**HAMT**: This is an ADL representing `type BalanceTable {AddressString:TokenAmount}`, where `AddressString` is the string form of the raw bytes of the `Address`.

Note that this HAMT block form is indistinguishable from `ActorIDHAMT` and `DealIDHAMT` which are also `{String:Int}`.

```ipldsch
type BalanceTableHAMT struct {
  map Bytes
  data [ BalanceTableHAMTElement ]
} representation tuple

type BalanceTableHAMTElement union {
  | BalanceTableHAMTLink "0"
  | BalanceTableHAMTBucket "1"
} representation keyed

type BalanceTableHAMTLink &BalanceTableHAMT

type BalanceTableHAMTBucket [ BalanceTableHAMTBucketEntry ]

type BalanceTableHAMTBucketEntry struct {
  key Bytes # Address
  value TokenAmount
} representation tuple
```

**SetMultimap (HAMT+HAMT)**: This is an ADL representing a Set within a Map `type DealOpsByEpoch {ChainEpochString:{DealIDString:Null}}`, where `ChainEpochString` is a text form of the `ChainEpoch` integer and `DealIDString` is the bytes of a uvarint form of `DealID`.

```ipldsch
# HAMT/map root structure

type DealOpsByEpochHAMT struct {
  map Bytes
  data [ DealOpsByEpochHAMTElement ]
} representation tuple

type DealOpsByEpochHAMTElement union {
  | DealOpsByEpochHAMTLink "0"
  | DealOpsByEpochHAMTBucket "1"
} representation keyed

type DealOpsByEpochHAMTLink &DealOpsByEpochHAMTLink

type DealOpsByEpochHAMTBucket [ DealOpsByEpochHAMTBucketEntry ]

type DealOpsByEpochHAMTBucketEntry struct {
  key Bytes
  value &DealOpsByEpochAMT
} representation tuple

# HAMT/set leaf structure (map of nulls)

type DealOpsByEpochHAMTSet struct {
  map Bytes
  data [ DealOpsByEpochHAMTSetElement ]
} representation tuple

type DealOpsByEpochHAMTSetElement union {
  | DealOpsByEpochHAMTSet_Link "0"
  | DealOpsByEpochHAMTSetBucket "1"
} representation keyed

type DealOpsByEpochHAMTSet_Link &DealOpsByEpochHAMTSet

type DealOpsByEpochHAMTSetBucket [ DealOpsByEpochHAMTSetBucketEntry ]

type DealOpsByEpochHAMTSetBucketEntry struct {
  key Bytes # The bytes of the string form of ChainEpoch
  value Null
} representation tuple
```

### StorageMarketActor

```ipldsch
type MinerV0State struct {
  Info  &MinerV0Info
  PreCommitDeposits TokenAmount
  LockedFunds TokenAmount
  VestingFunds &MinerV0VestingFunds
  InitialPledge TokenAmount
  PreCommittedSectors &MinerV0SectorPreCommitOnChainInfoHAMT # HAMT[SectorNumber]SectorPreCommitOnChainInfo
  PreCommittedSectorsExpiry &BitFieldQueueAMT # AMT[ChainEpoch]BitField
  AllocatedSectors &BitField
  Sectors &MinerV0SectorOnChainInfoAMT # AMT[SectorNumber]SectorOnChainInfo
  ProvingPeriodStart ChainEpoch
  CurrentDeadline Int
  Deadlines &MinerV0Deadlines
  EarlyTerminations BitField
} representation tuple
```

**v2**

```ipldsch
type MinerV2State struct {
  Info  &MinerV2Info
  PreCommitDeposits TokenAmount
  LockedFunds TokenAmount
  VestingFunds &MinerV0VestingFunds
  FeeDebt TokenAmount
  InitialPledge TokenAmount
  PreCommittedSectors &MinerV0SectorPreCommitOnChainInfoHAMT # HAMT[SectorNumber]SectorPreCommitOnChainInfo
  PreCommittedSectorsExpiry &BitFieldQueueAMT # AMT[ChainEpoch]BitField
  AllocatedSectors &BitField
  Sectors &MinerV0SectorOnChainInfoAMT # AMT[SectorNumber]SectorOnChainInfo
  ProvingPeriodStart ChainEpoch
  CurrentDeadline Int
  Deadlines &MinerV2Deadlines
  EarlyTerminations BitField
} representation tuple
```

```ipldsch
type MinerV0Info struct {
  Owner Address
  Worker Address
  ControlAddresses nullable [Address]
  PendingWorkerKey nullable MinerV0WorkerChangeKey
  PeerId PeerID
  Multiaddrs nullable [Multiaddr]
  SealProofType Int
  SectorSize SectorSize
  WindowPoStPartitionSectors Int
} representation tuple

type MinerV0WorkerChangeKey struct {
  NewWorker Address
  EffectiveAt ChainEpoch
} representation tuple
```

**v2**

```ipldsch
type MinerV2Info struct {
  Owner Address
  Worker Address
  ControlAddresses nullable [Address]
  PendingWorkerKey nullable MinerV0WorkerChangeKey
  PeerId PeerID
  Multiaddrs nullable [Multiaddr]
  SealProofType Int
  SectorSize SectorSize
  WindowPoStPartitionSectors Int
  ConsensusFaultElapsed ChainEpoch
  PendingOwnerAddress nullable Address
} representation tuple
```

```ipldsch
type MinerV0VestingFunds struct {
  Funds [MinerV0VestingFund]
} representation tuple

type MinerV0VestingFund struct {
  Epoch ChainEpoch
  Amount TokenAmount
} representation tuple
```

```ipldsch
type MinerV0Deadlines struct {
  Due MinerV0DeadlineLinkList
} representation tuple

# Must be 48 CIDs
type MinerV0DeadlineLinkList struct {
  deadline1 &MinerV0Deadline
  deadline2 &MinerV0Deadline
  deadline3 &MinerV0Deadline
  deadline4 &MinerV0Deadline
  deadline5 &MinerV0Deadline
  deadline6 &MinerV0Deadline
  deadline7 &MinerV0Deadline
  deadline8 &MinerV0Deadline
  deadline9 &MinerV0Deadline
  deadline10 &MinerV0Deadline
  deadline11 &MinerV0Deadline
  deadline12 &MinerV0Deadline
  deadline13 &MinerV0Deadline
  deadline14 &MinerV0Deadline
  deadline15 &MinerV0Deadline
  deadline16 &MinerV0Deadline
  deadline17 &MinerV0Deadline
  deadline18 &MinerV0Deadline
  deadline19 &MinerV0Deadline
  deadline20 &MinerV0Deadline
  deadline21 &MinerV0Deadline
  deadline22 &MinerV0Deadline
  deadline23 &MinerV0Deadline
  deadline24 &MinerV0Deadline
  deadline25 &MinerV0Deadline
  deadline26 &MinerV0Deadline
  deadline27 &MinerV0Deadline
  deadline28 &MinerV0Deadline
  deadline29 &MinerV0Deadline
  deadline30 &MinerV0Deadline
  deadline31 &MinerV0Deadline
  deadline32 &MinerV0Deadline
  deadline33 &MinerV0Deadline
  deadline34 &MinerV0Deadline
  deadline35 &MinerV0Deadline
  deadline36 &MinerV0Deadline
  deadline37 &MinerV0Deadline
  deadline38 &MinerV0Deadline
  deadline39 &MinerV0Deadline
  deadline40 &MinerV0Deadline
  deadline41 &MinerV0Deadline
  deadline42 &MinerV0Deadline
  deadline43 &MinerV0Deadline
  deadline44 &MinerV0Deadline
  deadline45 &MinerV0Deadline
  deadline46 &MinerV0Deadline
  deadline47 &MinerV0Deadline
  deadline48 &MinerV0Deadline
} representation tuple
```

**v2** (same form as `MinerV0Deadlines` but the eventual link to `MinerV2Partition` is different.)

```ipldsch
type MinerV2Deadlines struct {
  Due MinerV2DeadlineLinkList
} representation tuple

# Must be 48 CIDs
type MinerV2DeadlineLinkList struct {
  deadline1 &MinerV2Deadline
  deadline2 &MinerV2Deadline
  deadline3 &MinerV2Deadline
  deadline4 &MinerV2Deadline
  deadline5 &MinerV2Deadline
  deadline6 &MinerV2Deadline
  deadline7 &MinerV2Deadline
  deadline8 &MinerV2Deadline
  deadline9 &MinerV2Deadline
  deadline10 &MinerV2Deadline
  deadline11 &MinerV2Deadline
  deadline12 &MinerV2Deadline
  deadline13 &MinerV2Deadline
  deadline14 &MinerV2Deadline
  deadline15 &MinerV2Deadline
  deadline16 &MinerV2Deadline
  deadline17 &MinerV2Deadline
  deadline18 &MinerV2Deadline
  deadline19 &MinerV2Deadline
  deadline20 &MinerV2Deadline
  deadline21 &MinerV2Deadline
  deadline22 &MinerV2Deadline
  deadline23 &MinerV2Deadline
  deadline24 &MinerV2Deadline
  deadline25 &MinerV2Deadline
  deadline26 &MinerV2Deadline
  deadline27 &MinerV2Deadline
  deadline28 &MinerV2Deadline
  deadline29 &MinerV2Deadline
  deadline30 &MinerV2Deadline
  deadline31 &MinerV2Deadline
  deadline32 &MinerV2Deadline
  deadline33 &MinerV2Deadline
  deadline34 &MinerV2Deadline
  deadline35 &MinerV2Deadline
  deadline36 &MinerV2Deadline
  deadline37 &MinerV2Deadline
  deadline38 &MinerV2Deadline
  deadline39 &MinerV2Deadline
  deadline40 &MinerV2Deadline
  deadline41 &MinerV2Deadline
  deadline42 &MinerV2Deadline
  deadline43 &MinerV2Deadline
  deadline44 &MinerV2Deadline
  deadline45 &MinerV2Deadline
  deadline46 &MinerV2Deadline
  deadline47 &MinerV2Deadline
  deadline48 &MinerV2Deadline
} representation tuple
```

```ipldsch
type MinerV0Deadline struct {
  Partitions &MinerV0PartitionAMT # AMT[PartitionNumber]Partition
  ExpirationEpochs &BitFieldQueueAMT # AMT[ChainEpoch]BitField
  PostSubmissions BitField
  EarlyTerminations BitField
  LiveSectors Int
  TotalSectors Int
  FaultyPower MinerV0PowerPair
} representation tuple
```

**v2**

```ipldsch
type MinerV2Deadline struct {
  Partitions &MinerV2PartitionAMT # AMT[PartitionNumber]Partition
  ExpirationEpochs &BitFieldQueueAMT # AMT[ChainEpoch]BitField
  PostSubmissions BitField
  EarlyTerminations BitField
  LiveSectors Int
  TotalSectors Int
  FaultyPower MinerV0PowerPair
} representation tuple
```

**HAMT**: This is an ADL representing `type SectorPreCommitOnChainInfoMap {SectorNumberString:SectorPreCommitOnChainInfo}`. Where SectorNumberString is the uvarint bytes (string) form of `SectorNumber`.

```ipldsch
type MinerV0SectorPreCommitOnChainInfoHAMT struct {
  map Bytes
  data [ MinerV0SectorPreCommitOnChainInfoHAMTElement ]
} representation tuple

type MinerV0SectorPreCommitOnChainInfoHAMTElement union {
  | MinerV0SectorPreCommitOnChainInfoHAMTLink "0"
  | MinerV0SectorPreCommitOnChainInfoHAMTBucket "1"
} representation keyed

type MinerV0SectorPreCommitOnChainInfoHAMTLink &MinerV0SectorPreCommitOnChainInfoHAMT

type MinerV0SectorPreCommitOnChainInfoHAMTBucket [ MinerV0SectorPreCommitOnChainInfoHAMTBucketEntry ]

type MinerV0SectorPreCommitOnChainInfoHAMTBucketEntry struct {
  key Bytes
  value MinerV0SectorPreCommitOnChainInfo # inline
} representation tuple

type MinerV0SectorPreCommitOnChainInfo struct {
  Info MinerV0SectorPreCommitInfo
  PreCommitDeposit TokenAmount
  PreCommitEpoch ChainEpoch
  DealWeight DealWeight
  VerifiedDealWeight DealWeight
} representation tuple

type MinerV0SectorPreCommitInfo struct {
  SealProof RegisteredSealProof
  SectorNumber SectorNumber
  SealedCID &Any # A CID with fil-commitment-sealed + poseidon-bls12_381-ac-fc1
  SealRandEpoch ChainEpoch
  DealIDs [DealID]
  Expiration ChainEpoch
  ReplaceCapacity Bool
  ReplaceSectorDeadline Int
  ReplaceSectorPartition PartitionNumber
  ReplaceSectorNumber SectorNumber
} representation tuple
```

**AMT**: This is an ADL representing `type BitFieldQueue [BitField]` indexed by `ChainEpoch`.

```ipldsch
type BitFieldQueueAMT struct {
  height Int
  count Int
  node BitFieldQueueAMTNode
} representation tuple

type BitFieldQueueAMTNode struct {
  bitmap Bytes
  children [&BitFieldQueueAMTNode]
  values [Bitfield]
} representation tuple
```

**AMT**: This is an ADL representing `type SectorOnChainInfoList [SectorOnChainInfo]` indexed by `SectorNumber`.

```ipldsch
type MinerV0SectorOnChainInfoAMT struct {
  height Int
  count Int
  node MinerV0SectorOnChainInfoAMTNode
} representation tuple

type MinerV0SectorOnChainInfoAMTNode struct {
  bitmap Bytes
  children [&MinerV0SectorOnChainInfoAMTNode]
  values [MinerV0SectorOnChainInfo]
} representation tuple

type MinerV0SectorOnChainInfo struct {
  SectorNumber SectorNumber
  SealProof RegisteredSealProof
  SealedCID &Any # A CID with fil-commitment-sealed + poseidon-bls12_381-ac-fc1
  DealIDs [DealID]
  Activation ChainEpoch
  Expiration ChainEpoch
  DealWeight DealWeight
  VerifiedDealWeight DealWeight
  InitialPledge TokenAmount
  ExpectedDayReward TokenAmount
  ExpectedStorageReward TokenAmount
  ReplacedSectorAge ChainEpoch
  ReplacedDayReward TokenAmount
} representation tuple
```

**AMT**: This is an ADL representing `type MinerV0PartitionList [MinerV0Partition]` indexed by `PartitionNumber`.

```ipldsch
type MinerV0PartitionAMT struct {
  height Int
  count Int
  node MinerV0PartitionAMTNode
} representation tuple

type MinerV0PartitionAMTNode struct {
  bitmap Bytes
  children [&MinerV0PartitionAMTNode]
  values [MinerV0Partition]
} representation tuple

type MinerV0Partition struct {
  Sectors BitField
  Faults BitField
  Recoveries BitField
  Terminated BitField
  ExpirationsEpochs &MinerV0ExpirationSetAMT # AMT[ChainEpoch]ExpirationSet
  EarlyTerminated &BitFieldQueueAMT # AMT[ChainEpoch]BitField
  LivePower MinerV0PowerPair
  FaultyPower MinerV0PowerPair
  RecoveringPower MinerV0PowerPair
} representation tuple

type MinerV0PowerPair struct {
  Raw StoragePower
  QA StoragePower
} representation tuple
```

**v2**

**AMT**: This is an ADL representing `type MinerV2PartitionList [MinerV2Partition]` indexed by `PartitionNumber`.

```ipldsch
type MinerV2PartitionAMT struct {
  height Int
  count Int
  node MinerV2PartitionAMTNode
} representation tuple

type MinerV2PartitionAMTNode struct {
  bitmap Bytes
  children [&MinerV2PartitionAMTNode]
  values [MinerV2Partition]
} representation tuple

type MinerV2Partition struct {
  Sectors BitField
  Unproven BitField
  Faults BitField
  Recoveries BitField
  Terminated BitField
  ExpirationsEpochs &MinerV0ExpirationSetAMT # AMT[ChainEpoch]ExpirationSet
  EarlyTerminated &BitFieldQueueAMT # AMT[ChainEpoch]BitField
  LivePower MinerV0PowerPair
  UnprovenPower MinerV0PowerPair
  FaultyPower MinerV0PowerPair
  RecoveringPower MinerV0PowerPair
} representation tuple
```

**AMT**: This is an ADL representing `type MinerV0ExpirationSetList [MinerV0ExpirationSet]` indexed by `ChainEpoch`.

```ipldsch
type MinerV0ExpirationSetAMT struct {
  height Int
  count Int
  node MinerV0ExpirationSetAMTNode
} representation tuple

type MinerV0ExpirationSetAMTNode struct {
  bitmap Bytes
  children [&MinerV0ExpirationSetAMTNode]
  values [MinerV0ExpirationSet]
} representation tuple

type MinerV0ExpirationSet struct {
  OnTimeSectors BitField
  EarlySectors BitField
  OnTimePledge TokenAmount
  ActivePower MinerV0PowerPair
  FaultyPower MinerV0PowerPair
} representation tuple
```

### MultisigActor

```ipldsch
type MultisigV0State struct {
  Signers [Address]
  NumApprovalsThreshold Int
  NextTxnID TransactionID
  InitialBalance TokenAmount
  StartEpoch ChainEpoch
  UnlockDuration ChainEpoch
  PendingTxns &MultisigV0TransactionHAMT # HAMT[TransactionID]Multisigv0Transaction
} representation tuple
```

**HAMT**: This is an ADL representing `type MultisigV0TransactionMap {TransactionIDString:MultisigV0Transaction}`. Where TransactionIDString is the varint (not uvarint) bytes (string) form of `TransactionID`.

```ipldsch
type MultisigV0TransactionHAMT struct {
  map Bytes
  data [ MultisigV0TransactionHAMTElement ]
} representation tuple

type MultisigV0TransactionHAMTElement union {
  | MultisigV0TransactionHAMTLink "0"
  | MultisigV0TransactionHAMTBucket "1"
} representation keyed

type MultisigV0TransactionHAMTLink &MultisigV0TransactionHAMT

type MultisigV0TransactionHAMTBucket [ MultisigV0TransactionHAMTBucketEntry ]

type MultisigV0TransactionHAMTBucketEntry struct {
  key Bytes
  value MultisigV0Transaction # inline
} representation tuple

type MultisigV0Transaction struct {
  To Address
  Value TokenAmount
  Method MethodNum
  Params Bytes
  Approved [Address]
} representation tuple
```

### PaymentChannelActor

```ipldsch
type PaychV0State struct {
  From Address
  To Address
  ToSend BigInt
  SettlingAt ChainEpoch
  MinSettleHeight ChainEpoch
  LaneStates &PaychV0LaneStatesAMT # AMT[Int]PaychV0LaneState
} representation tuple
```

**AMT**: This is an ADL representing `type PaychV0LaneStates [PaychV0LaneState]` indexed by Lane ID.

```ipldsch
type PaychV0LaneStatesAMT struct {
  height Int
  count Int
  node PaychV0LaneStatesAMTNode
} representation tuple

type PaychV0LaneStatesAMTNode struct {
  bitmap Bytes
  children [&PaychV0LaneStatesAMTNode]
  values [PaychV0LaneState]
} representation tuple

type PaychV0LaneState struct {
  Redeemed BigInt
  Nonce Int
} representation tuple
```

### StoragePowerActor

```ipldsch
type PowerV0State struct {
  TotalRawBytePower StoragePower
  TotalBytesCommitted StoragePower
  TotalQualityAdjPower StoragePower
  TotalQABytesCommitted StoragePower
  TotalPledgeCollateral TokenAmount
  ThisEpochRawBytePower StoragePower
  ThisEpochQualityAdjPower StoragePower
  ThisEpochPledgeCollateral TokenAmount
  ThisEpochQAPowerSmoothed nullable V0FilterEstimate
  MinerCount Int
  MinerAboveMinPowerCount Int
  CronEventQueue &PowerV0CronEventHAMT # Multimap: HAMT[ChainEpoch]AMT[PowerV0CronEvent]
  FirstCronEpoch ChainEpoch
  LastProcessedCronEpoch ChainEpoch
  Claims &PowerV0ClaimHAMT # HAMT[address]PowerV0Claim
  ProofValidationBatch nullable &ProofValidationBatchHAMT # Multimap: HAMT[Address]AMT[SealVerifyInfo]
} representation tuple
```

**Multimap (HAMT+AMT)**: This is an ADL representing a List within a Map `type PowerV0CronEventMap {ChainEpochString:[PowerV0CronEvent]}`, where `ChainEpochString` is a text form of the `ChainEpoch` integer the `CronEvent` list is a queue.

```ipldsch
# HAMT/map root structure

type PowerV0CronEventHAMT struct {
  map Bytes
  data [ PowerV0CronEventHAMTElement ]
} representation tuple

type PowerV0CronEventHAMTElement union {
  | PowerV0CronEventHAMTLink "0"
  | PowerV0CronEventHAMTBucket "1"
} representation keyed

type PowerV0CronEventHAMTLink &PowerV0CronEventHAMTLink

type PowerV0CronEventHAMTBucket [ PowerV0CronEventHAMTBucketEntry ]

type PowerV0CronEventHAMTBucketEntry struct {
  key Bytes # ChainEpoch as a string
  value &PowerV0CronEventAMT
} representation tuple

# AMT/set leaf structure (list of CronEvents)

type PowerV0CronEventAMT struct {
  height Int
  count Int
  node PowerV0CronEventAMTNode
} representation tuple

# This can also be the root of a block
type PowerV0CronEventAMTNode struct {
  bitmap Bytes
  children [&PowerV0CronEventAMTNode]
  values [PowerV0CronEvent]
} representation tuple

type PowerV0CronEvent struct {
  MinerAddr Address
  CallbackPayload Bytes
} representation tuple
```

**HAMT**: This is an ADL representing `type PowerV0ClaimMap {AddressString:PowerV0Claim}`, where `AddressString` is the string form of the raw bytes of the `Address`.

```ipldsch
type PowerV0ClaimMapHAMT struct {
  map Bytes
  data [ PowerV0ClaimMapHAMTElement ]
} representation tuple

type PowerV0ClaimMapHAMTElement union {
  | PowerV0ClaimMapHAMTLink "0"
  | PowerV0ClaimMapHAMTBucket "1"
} representation keyed

type PowerV0ClaimMapHAMTLink &PowerV0ClaimMapHAMT

type PowerV0ClaimMapHAMTBucket [ PowerV0ClaimMapHAMTBucketEntry ]

type PowerV0ClaimMapHAMTBucketEntry struct {
  key Bytes # Address
  value PowerV0Claim
} representation tuple

type PowerV0Claim struct {
  RawBytePower StoragePower
  QualityAdjPower StoragePower
} representation tuple
```

**Multimap (HAMT+AMT)**: This is an ADL representing a List within a Map `type ProofValidationBatchMap {AddressString:[SealVerifyInfo]}`, where `AddressString` is the string form of the raw bytes of the `Address` and the `CronEvent` list is a queue.

```ipldsch
# HAMT/map root structure

type ProofValidationBatchHAMT struct {
  map Bytes
  data [ ProofValidationBatchHAMTElement ]
} representation tuple

type ProofValidationBatchHAMTElement union {
  | ProofValidationBatchHAMTLink "0"
  | ProofValidationBatchHAMTBucket "1"
} representation keyed

type ProofValidationBatchHAMTLink &ProofValidationBatchHAMTLink

type ProofValidationBatchHAMTBucket [ ProofValidationBatchHAMTBucketEntry ]

type ProofValidationBatchHAMTBucketEntry struct {
  key Bytes # ChainEpoch as a string
  value &ProofValidationBatchAMT
} representation tuple

# AMT/set leaf structure (list of CronEvents)

type ProofValidationBatchAMT struct {
  height Int
  count Int
  node ProofValidationBatchAMTNode
} representation tuple

# This can also be the root of a block
type ProofValidationBatchAMTNode struct {
  bitmap Bytes
  children [&ProofValidationBatchAMTNode]
  values [SealVerifyInfo]
} representation tuple

type SealVerifyInfo struct {
  SealProof RegisteredSealProof
  SectorID SectorID
  DealIDs [DealID]
  Randomness Bytes
  InteractiveRandomness Bytes
  Proof Bytes
  SealedCID &Any # CommR: A CID with fil-commitment-sealed + poseidon-bls12_381-ac-fc1
  UnsealedCID &Any # CommD: A CID with fil-commitment-unsealed + sha2_256-trunc254-padded
} representation tuple
```

### VerifiedRegistryActor

```ipldsch
type VerifregV0State struct {
  RootKey Address
  Verifiers &DataCapHAMT
  VerifiedClients &DataCapHAMT
}
```

**HAMT**: This is an ADL representing `type DataCapMap {AddressString:StoragePower}`, where `AddressString` is the string form of the raw bytes of the `Address`.

```ipldsch
type DataCapHAMT struct {
  map Bytes
  data [ DataCapHAMTElement ]
} representation tuple

type DataCapHAMTElement union {
  | DataCapHAMTLink "0"
  | DataCapHAMTBucket "1"
} representation keyed

type DataCapHAMTLink &DataCapHAMT

type DataCapHAMTBucket [ DataCapHAMTBucketEntry ]

type DataCapHAMTBucketEntry struct {
  key Address # Address
  value StoragePower # inline
} representation tuple
```

### SystemActor

Note that `SystemActor` is an empty struct, which encodes as an empty CBOR array (`0x80`).

```ipldsch
type SystemActor struct {
} representation tuple
```