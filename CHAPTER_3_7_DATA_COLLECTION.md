# Chapter 3.7: Data Collection

## 3.7.1 Overview of Data Collection Strategy

The blockchain-based voting system employs a multi-layered data collection approach that captures voter identity, voting preferences, and transactional metadata across three primary tiers: client-side (frontend), relay infrastructure, and the immutable blockchain layer. This distributed architecture ensures data integrity while maintaining voter privacy through cryptographic protocols and decentralized storage mechanisms.

## 3.7.2 Data Sources and Collection Points

### 3.7.2.1 Client-Side Data Collection (Frontend Layer)

The frontend application (React-based) collects data from voters at the point of interaction:

#### **Voter Authentication Data**
- **Wallet Address**: Collected via MetaMask Web3 integration when a voter connects their wallet
- **Network Information**: Chain ID, network name (e.g., Sepolia), and RPC endpoint verification
- **Session Data**: Connection timestamps and wallet connection status

**Implementation Details**:
```javascript
// From App.js - Wallet connection capture
const getSigner = useCallback(async (requestAccounts = false) => {
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    if (requestAccounts) {
        await provider.send('eth_requestAccounts', []);
    }
    const signer = provider.getSigner();
    const address = await signer.getAddress();  // Voter address collected here
    return { provider, signer, address };
}, []);
```

#### **Voting Data**
- **Candidate Selection**: Captured when voter selects their preferred candidate (integer index: 0-n)
- **Voting Status**: Boolean indicator of whether the voter has already cast a vote
- **Vote Confirmation**: Transaction hash and timestamp of vote submission

**Data Collection Trigger**: User interaction with candidate selection buttons triggers collection of:
- `candidateIndex` (0-indexed selection)
- Timestamp of vote initiation
- Account address performing the vote

### 3.7.2.2 Smart Contract Data Collection (Blockchain Layer)

The Voting.sol smart contract automatically collects and immutably records voting data on-chain:

#### **Collected Data Elements**:

```solidity
struct Candidate {
    string name;        // Candidate identifier
    uint256 voteCount;  // Cumulative vote tally
}

mapping(address => bool) public voters;  // Voter participation register
```

**Data Captured at Vote Time**:
- **Voter Address** (`msg.sender` or meta-tx sender): Cryptographically derived from wallet signature
- **Candidate Index**: Which candidate received the vote
- **Block Timestamp**: Exact time of vote submission
- **Transaction Hash**: Unique identifier for the voting transaction
- **Nonce Value**: Sequential counter preventing replay attacks (from MinimalForwarder)

**Contract State Variables Maintained**:
```solidity
uint256 public votingStart;   // Election commencement timestamp
uint256 public votingEnd;     // Election closure timestamp
address public trustedForwarder; // Relay infrastructure identifier
address owner;                // Administrator/election authority
```

### 3.7.2.3 Relayer Infrastructure Data Collection (Sponsored Transaction Layer)

The relayer server (`relayer/server.js`) collects and processes meta-transaction data:

#### **Collected Request Data**:
- **Voter Address**: Source address of the vote transaction
- **Candidate Index**: Target candidate receiving the vote
- **Nonce**: Transaction ordering sequence from forwarder contract
- **Gas Parameters**: Estimated gas consumption (300,000 units)
- **Signature Data**: EIP-712 cryptographic signature from voter's wallet

**Meta-Transaction Structure Captured**:
```javascript
const requestData = {
    from: voter,              // Voter wallet address
    to: VOTING_CONTRACT_ADDRESS,  // Target contract
    value: 0,                 // No ETH transfer
    gas: 300000,              // Gas allocation
    nonce: nonce.toString(),  // Ordered sequence number
    data,                     // Encoded vote(candidateIndex) function call
};
```

#### **EIP-712 Domain Context**:
```javascript
const domain = {
    name: "MinimalForwarder",
    version: "0.0.1",
    chainId: CHAIN_ID,               // Network identifier
    verifyingContract: FORWARDER_ADDRESS  // Forwarder address
};
```

## 3.7.3 Data Collection Methods

### 3.7.3.1 Active Collection (User-Initiated)

| Data Type | Method | Timing | Storage |
|-----------|--------|--------|---------|
| Wallet Address | MetaMask Connection Request | On login | Browser + Contract |
| Candidate Selection | Click Event Handler | On vote submission | Frontend State + Contract |
| Signature | EIP-712 Signing | Pre-submission | Transaction payload |

### 3.7.3.2 Passive Collection (Automatic)

| Data Type | Method | Timing | Storage |
|-----------|--------|--------|---------|
| Timestamps | `block.timestamp` (contract) | Vote execution | Blockchain |
| Tx Hash | Network validation | Post-execution | Blockchain + Explorer |
| Nonce | Forwarder contract | Per transaction | Blockchain |
| Gas consumption | EVM execution | Transaction processing | Blockchain |

### 3.7.3.3 State Query Collection

Real-time data retrieval through read-only operations:
```javascript
const [candidatesList, status, time, voteStatus] = await Promise.all([
    contract.getAllVotesOfCandiates(),    // Fetch all candidates + vote counts
    contract.getVotingStatus(),           // Active/inactive voting period
    contract.getRemainingTime(),          // Time until election closes
    contract.voters(address),             // Has this voter already voted?
]);
```

## 3.7.4 Data Collection Architecture

### 3.7.4.1 Multi-Tier Data Flow

```
┌─────────────────┐
│  Voter Wallet   │
│  (MetaMask)     │
└────────┬────────┘
         │ (Signature)
         ▼
┌─────────────────────┐
│  Frontend React App  │ ◄─── Captures: Address, Candidate Index, Timestamps
│  (App.js, Login.jsx)│
└────────┬────────────┘
         │ (Vote Transaction)
         ├─────────────────────────────────────┐
         │                                     │
         ▼ (Direct)                    ▼ (Sponsored)
    ┌─────────────┐              ┌──────────────┐
    │  Voting     │              │   Relayer    │
    │ Contract    │              │  Server      │
    └─────────────┘              └──────┬───────┘
         │                              │
         └──────────────┬───────────────┘
                        ▼
                  ┌─────────────────┐
                  │   Blockchain    │
                  │   (Sepolia)     │
                  └─────────────────┘
```

## 3.7.5 Specific Data Elements Collected

### 3.7.5.1 Voter Identity Data

| Field | Type | Source | Immutable |
|-------|------|--------|-----------|
| Wallet Address | address (20 bytes) | MetaMask via Web3Provider | Yes (blockchain) |
| Public Key | bytes32 | Recovered from ECDSA signature | Yes |
| Vote Status | boolean | Smart contract mapping | Yes |

### 3.7.5.2 Vote Data

| Field | Type | Source | Example |
|-------|------|--------|---------|
| Candidate Index | uint256 | User selection (0-3) | 1 (selects "Rajnish") |
| Vote Count | uint256 | Contract accumulator | 127 total votes |
| Candidate Name | string | Contract storage | "Ratnesh Kumar" |

### 3.7.5.3 Temporal Data

| Field | Type | Source | Format |
|-------|------|--------|--------|
| Voting Start | uint256 | Constructor (block.timestamp) | Unix epoch (seconds) |
| Voting End | uint256 | Constructor + duration | Unix epoch (seconds) |
| Vote Timestamp | uint256 | block.timestamp at vote() call | Unix epoch (seconds) |
| Remaining Time | uint256 | Computed: votingEnd - now | Seconds |

### 3.7.5.4 Transaction Data

| Field | Type | Source | Notes |
|-------|------|--------|-------|
| Transaction Hash | bytes32 | Network | Uniquely identifies each vote |
| Block Number | uint256 | Blockchain | Confirms transaction finality |
| Nonce | uint256 | MinimalForwarder | Prevents replay attacks |
| Gas Used | uint256 | EVM execution | Transaction cost |

## 3.7.6 Data Storage Mechanisms

### 3.7.6.1 Blockchain Storage (Immutable, On-Chain)

**Storage Locations**:
```solidity
mapping(address => bool) public voters;     // Maps voter address to voting status
Candidate[] public candidates;              // Array of candidate structs
```

**Characteristics**:
- **Immutability**: Once written, data cannot be altered or deleted
- **Transparency**: All state changes are publicly auditable
- **Persistence**: Data survives network restarts and system failures
- **Cost**: Storage operations consume gas fees (approximately 20,000 gas per vote)

### 3.7.6.2 Temporary Storage (Frontend Session)

**Memory Storage**:
```javascript
const [account, setAccount] = useState(null);           // Current voter address
const [candidates, setCandidates] = useState([]);       // Candidate list
const [hasVoted, setHasVoted] = useState(false);        // Vote status
const [activityFeed, setActivityFeed] = useState([]);   // Transaction history
```

**Characteristics**:
- **Ephemeral**: Lost on page refresh or browser close
- **Client-side**: Not transmitted or stored on servers
- **Privacy**: No server-side logging of voter actions

### 3.7.6.3 Browser Local Storage (Demo Mode)

```javascript
const storageKey = 'demo_voter_address';
window.localStorage.setItem(storageKey, demoAddress);  // Persistent browser storage
```

**Demo Mode Function**: Maintains voter identity across browser sessions for demonstration purposes.

### 3.7.6.4 Relayer Transaction Log (Backend)

**Server Endpoints Collecting Data**:

1. **`POST /meta-tx/prepare`**: Receives and processes
   - Voter address
   - Candidate index
   - Prepares nonce and request structure

2. **`POST /meta-tx/submit`**: Records
   - Request signature
   - Transaction hash
   - Block number confirmation

## 3.7.7 Data Collection Timeline

### 3.7.7.1 Sequence of Data Capture Events

```
Event 1: Connection
├─ Timestamp: T0 (user clicks "Connect")
├─ Data: Wallet address requested via eth_requestAccounts
└─ Status: Account captured from MetaMask

Event 2: Network Verification
├─ Timestamp: T1
├─ Data: Current chain ID (eth_chainId)
└─ Status: Target network validation (Sepolia = 0xaa36a7)

Event 3: Contract Queries
├─ Timestamp: T2
├─ Data: Candidates[], votingStatus, remainingTime, voterStatus
└─ Status: All contract state loaded

Event 4: Vote Selection
├─ Timestamp: T3 (user clicks candidate)
├─ Data: candidateIndex captured
└─ Status: Pending submission

Event 5: Signature Request
├─ Timestamp: T4
├─ Data: EIP-712 signature from wallet
└─ Status: Vote authorization

Event 6: Transaction Submission
├─ Timestamp: T5
├─ Data: vote(candidateIndex) executed on blockchain
└─ Status: On-chain record created
```

## 3.7.8 Data Privacy and Security Measures During Collection

### 3.7.8.1 Cryptographic Protection

- **ECDSA Signatures**: All votes cryptographically signed by voter private key
- **EIP-712 Standard**: Structured data hashing prevents signature misuse
- **Nonce Protection**: Sequential counter (from MinimalForwarder) prevents transaction replay

### 3.7.8.2 Voter Privacy

- **Address-Only Identity**: No personally identifiable information collected
- **Pseudonymous Voting**: Voter identity is their blockchain address (0x...)
- **No IP Logging**: Frontend does not capture IP addresses or geolocation
- **No Server Tracking**: When using direct transactions (non-relayed), no backend records voting data

### 3.7.8.3 Vote Confidentiality

- **Vote Encryption**: Votes secured until submission to blockchain
- **No Vote Tampering**: Smart contract checks prevent duplicate votes
- **Replay Protection**: Nonce mechanism prevents resubmitting same vote

## 3.7.9 Data Collection Scope and Limitations

### 3.7.9.1 What IS Collected

✓ Voter blockchain address (pseudonymous identity)  
✓ Candidate selection (vote content)  
✓ Vote timestamp (on-chain block time)  
✓ Voting transaction hash  
✓ Nonce and gas parameters  

### 3.7.9.2 What IS NOT Collected

✗ Voter name or real identity  
✗ Voter email or contact information  
✗ Voter IP address or location  
✗ Voting reason or motivation  
✗ Device information or user agent  
✗ Session cookies or tracking tokens  

## 3.7.10 Summary

The blockchain-based voting system implements a **cryptographically-secured, distributed data collection model** where:

1. **Primary data source**: User wallet and voting selection (frontend)
2. **Secondary recording**: Immutable blockchain storage via Voting contract
3. **Collection method**: Event-driven (vote submission) + continuous query (status checks)
4. **Privacy model**: Pseudonymous (address-based, not identity-based)
5. **Immutability guarantee**: Smart contract enforces one-vote-per-address rule
6. **Transparency**: All votes publicly auditable via blockchain explorer

This approach balances **voter privacy** (no personal data) with **voting verifiability** (cryptographically proven participation) and **election integrity** (tamper-proof on-chain records).

---

**Data Collection Compliance Notes**:
- Aligns with blockchain transparency principles
- Minimizes data collection to essential voting parameters only
- Employs cryptographic rather than legal privacy controls
- Suitable for regulatory environments requiring audit trails and non-repudiation
