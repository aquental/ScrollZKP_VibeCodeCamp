### **1. ZK-Proof Verifier Logic**  
**Purpose**: Prove a user’s review is legitimate without revealing sensitive data (e.g., who submitted it).  

#### **Circuit Design** (Using Circom)  
```circom
pragma circom 2.0.0;

template ReviewVerifier() {
    // Inputs: Private (user's review data) & Public (root hash of approved reviews)
    signal input privateReviewHash;  // Hash of (userId + spaceId + rating)
    signal input publicRootHash;     // Merkle root of all valid reviews (stored on-chain)

    // Verify the review hash is in the Merkle tree
    component merkleProof = MerkleProofChecker(levels);
    merkleProof.leaf <== privateReviewHash;
    merkleProof.root <== publicRootHash;
    // ... (constraints to validate proof)
}

// Export the circuit
component main = ReviewVerifier();
```

#### **On-Chain Verifier Contract**  
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@scroll-tech/contracts/ZkVerifier.sol";  // Scroll's ZK verifier lib

contract ReviewVerifier is ZkVerifier {
    bytes32 public merkleRoot;  // Updated off-chain by Spare's backend

    function submitReview(
        uint[2] memory a,          // ZK-proof params
        uint[2][2] memory b,
        uint[2] memory c,
        uint[1] memory input       // publicRootHash
    ) public {
        require(verifyProof(a, b, c, input), "Invalid proof");
        require(input[0] == uint(merkleRoot), "Root mismatch");
        
        // If proof checks out, mint SBT
        Reputation.reputationNFT.mint(msg.sender, newReputationScore);
    }
}
```

#### **Workflow**  
1. **Off-Chain**: User generates a ZK-proof proving their review is in the Merkle tree (without revealing details).  
2. **On-Chain**: `ReviewVerifier` checks the proof against the current `merkleRoot`.  
3. **Outcome**: If valid, the `Reputation` contract mints an SBT.  

**Why Scroll?**  
- ZK-proofs are **native to Scroll’s zkEVM**, reducing verification costs vs. L1 Ethereum.  

---

### **2. Gas Cost Estimates**  
| **Action**               | **Scroll zkEVM Cost** | **Ethereum L1 Cost** |  
|---------------------------|-----------------------|----------------------|  
| Mint `RentalNFT`          | ~$0.02               | ~$5.00              |  
| Mint `Reputation` SBT     | ~$0.01               | ~$3.00              |  
| ZK-proof verification     | ~$0.05               | ~$12.00             |  

**Assumptions**:  
- Gas price: 0.1 Gwei (Scroll) vs. 30 Gwei (Ethereum).  
- Exchange rate: $1,800/ETH.  

**Optimizations**:  
- **Batching**: Process multiple SBT mints in one ZK-proof (e.g., 100 reviews → 1 proof).  
- **Storage Tricks**: Use `uint32` for timestamps to save storage slots.  

---

### **3. Attack Vectors & Mitigations**  

#### **A. RentalNFT Exploits**  
1. **Reentrancy on `completeRental`**  
   - **Risk**: Attacker re-enters before `_burn` to drain funds.  
   - **Fix**: Use OpenZeppelin’s `ReentrancyGuard`.  

2. **Fake Listings**  
   - **Risk**: Host lists non-existent spaces.  
   - **Fix**: Require IoT lock integration or geo-tagged photos.  

#### **B. Reputation SBT Exploits**  
1. **Sybil Attacks**  
   - **Risk**: Users create multiple wallets to inflate scores.  
   - **Fix**: Link SBTs to **ZK-verified identities** (e.g., Worldcoin).  

2. **Oracle Manipulation**  
   - **Risk**: Attacker spoofs the off-chain review database.  
   - **Fix**: Use a decentralized oracle (e.g., Chainlink) to update `merkleRoot`.  

#### **C. ZK-Proof Risks**  
1. **Fake Proofs**  
   - **Risk**: Submit malformed proofs to mint false SBTs.  
   - **Fix**: Use audited circuits (e.g., via [Veridise](https://www.veridise.com/)).  

2. **Merkle Root Collision**  
   - **Risk**: Two different review sets hash to the same `merkleRoot`.  
   - **Fix**: Use a **unique nonce** per root update.  

---

### **4. Scroll-Specific Risks**  
1. **Sequencer Censorship**  
   - **Risk**: Scroll’s sequencer delays your tx.  
   - **Mitigation**: Build a fallback to Ethereum L1 after N blocks.  

2. **Upgradability Bugs**  
   - **Risk**: Scroll’s zkEVM upgrades break your contracts.  
   - **Mitigation**: Use proxy patterns with emergency pauses.  

---

### **Tools for Testing**  
1. **ZK Circuit Audits**:  
   - [Ecne](https://github.com/0xPARC/ecne) (static analyzer for Circom).  
2. **Gas Profiling**:  
   ```bash
   npx hardhat test --gas-report
   ```  
3. **Attack Simulations**:  
   - Use [Foundry’s `forge`](https://book.getfoundry.sh/forge/) to fuzz test.  

---

### **Example Attack Scenario**  
**Frontrunning `completeRental`**:  
1. Attacker sees a `completeRental` tx in mempool.  
2. They send a higher-gas tx to `completeRental` first, stealing funds.  
**Fix**:  
- Use commit-reveal schemes or Scroll’s **native privacy features**.  

---

**Next Steps**:  
1. **Deploy test contracts** on Scroll’s [Sepolia testnet](https://scroll.io/developers).  
2. **Fuzz test** with `forge inspect RentalNFT --gas`.  
3. **Audit circuits** with Veridise before mainnet.  

---

# **detailed ZK-circuit walkthrough**

Here’s a **step-by-step walkthrough** for building a ZK circuit to verify Spare’s reputation reviews, using Circom and Scroll’s zkEVM. We’ll cover circuit design, proof generation, and on-chain verification:

### **1. Problem Definition**  
**Goal**: Prove that a user’s review is legitimate (i.e., it’s part of Spare’s approved review database) **without revealing**:  
- Who submitted the review.  
- The exact rating/comment.  

**Inputs**:  
- **Private**: `(userId, spaceId, rating, secretSalt)` → Hashed into `reviewHash`.  
- **Public**: Merkle root of all valid reviews (`merkleRoot`).  

---

### **2. Circuit Design (Circom)**  
#### **A. Hash the Review**  
First, hash the private inputs to create a leaf for the Merkle tree:  
```circom
// circuits/ReviewHasher.circom
pragma circom 2.0.0;

template ReviewHasher() {
    signal input userId;
    signal input spaceId;
    signal input rating;
    signal input secretSalt;
    signal output hash;  // reviewHash = H(userId | spaceId | rating | salt)

    component sha256 = SHA256();  // Standard Circom SHA256 template
    sha256.in[0] <== userId;
    sha256.in[1] <== spaceId;
    // ... (fill remaining inputs with rating/salt)
    hash <== sha256.out;
}
```

#### **B. Merkle Proof Verification**  
Verify that `reviewHash` is in the Merkle tree with root `merkleRoot`:  
```circom
// circuits/ReviewVerifier.circom
pragma circom 2.0.0;

include "ReviewHasher.circom";
include "merkle.circom";  // Standard Merkle tree lib

template ReviewVerifier() {
    // Private inputs
    signal input userId;
    signal input spaceId;
    signal input rating;
    signal input secretSalt;
    signal input pathElements[levels];  // Sibling nodes for Merkle proof
    signal input pathIndices[levels];   // 0/1 for left/right branches

    // Public inputs
    signal input merkleRoot;

    // Hash the review
    component hasher = ReviewHasher();
    hasher.userId <== userId;
    hasher.spaceId <== spaceId;
    hasher.rating <== rating;
    hasher.secretSalt <== secretSalt;

    // Verify Merkle proof
    component merkleProof = MerkleProof(levels);
    merkleProof.leaf <== hasher.hash;
    merkleProof.pathElements <== pathElements;
    merkleProof.pathIndices <== pathIndices;
    merkleProof.root <== merkleRoot;
}

component main = ReviewVerifier();
```

---

### **3. Compile & Generate Proofs**  
#### **A. Install Dependencies**  
```bash
npm install circom circomlib snarkjs
```

#### **B. Compile the Circuit**  
```bash
circom circuits/ReviewVerifier.circom --r1cs --wasm --sym
```

#### **C. Generate ZK-Proof (Off-Chain)**  
```javascript
// scripts/generateProof.js
const { wtns, proof } = await snarkjs.groth16.fullProve(
  {
    userId: "123",
    spaceId: "456",
    rating: "5",
    secretSalt: "789",
    pathElements: ["0xabc...", "0xdef..."],  // From your Merkle tree
    pathIndices: [0, 1],                     // Left/right branches
    merkleRoot: "0x123..."
  },
  "circuits/ReviewVerifier.wasm",
  "circuits/ReviewVerifier.zkey"
);

console.log(proof);  // { a, b, c, publicSignals }
```

---

### **4. On-Chain Verification (Scroll)**  
#### **A. Deploy Verifier Contract**  
```solidity
// contracts/ReviewVerifier.sol
pragma solidity ^0.8.0;

import "@scroll-tech/contracts/ZkVerifier.sol";

contract SpareReviewVerifier is ZkVerifier {
    bytes32 public merkleRoot;

    function setMerkleRoot(bytes32 newRoot) external onlyOwner {
        merkleRoot = newRoot;
    }

    function submitReview(
        uint[2] memory a,
        uint[2][2] memory b,
        uint[2] memory c,
        uint[1] memory input  // publicSignals (merkleRoot)
    ) public {
        require(input[0] == uint256(merkleRoot), "Invalid root");
        require(verifyProof(a, b, c, input), "Invalid proof");
        
        // Call Reputation.sol to mint SBT
        Reputation.mint(msg.sender);
    }
}
```

#### **B. Verify Proof On-Chain**  
```javascript
// Call from frontend
const calldata = await snarkjs.groth16.exportSolidityCallData(proof, publicSignals);
const tx = await verifierContract.submitReview(...JSON.parse(calldata));
```

---

### **5. Key Concepts Explained**  
#### **A. Merkle Trees**  
- **Why?** Efficiently prove membership without storing all reviews on-chain.  
- **How?** Reviews are hashed into leaves; only the root (`merkleRoot`) is stored on-chain.  

#### **B. Zero-Knowledge**  
- The proof convinces the verifier that:  
  - You know `(userId, spaceId, rating, salt)` that hash to a leaf in the tree.  
  - You **don’t reveal** the actual values.  

#### **C. Scroll Optimizations**  
- **zkEVM Native Verification**: Scroll’s L2 batches proofs, reducing gas costs.  
- **Cheap Storage**: Merkle roots are small (32 bytes) and cheap to update.  

---

### **6. Security Checks**  
1. **Circuit Constraints**  
   - Ensure all signals are properly constrained (e.g., `rating` is between 1-5).  
   ```circom
   signal input rating;
   rating === 1 || rating === 2 || ... || rating === 5;  // Enforce range
   ```  
2. **Trusted Setup**  
   - Use Scroll’s trusted ceremony for `zkey` generation (or run your own).  

3. **Merkle Root Updates**  
   - Only allow Spare’s backend (or a decentralized oracle) to update `merkleRoot`.  

---

### **7. Full Flow Summary**  
1. **User Submits Review**:  
   - Frontend hashes `(userId, spaceId, rating, salt)` → `reviewHash`.  
2. **Generate Proof**:  
   - Prove `reviewHash` is in the Merkle tree with root `merkleRoot`.  
3. **On-Chain Verification**:  
   - Scroll contract checks proof + root → mints SBT if valid.  

---

### **Tools & References**  
- **Circom Docs**: https://docs.circom.io  
- **Scroll zkEVM**: https://scroll.io/developers  
- **Merkle Tree Lib**: https://github.com/iden3/circomlib  

**Next Steps**:  
1. Test with a **local Merkle tree** (e.g., `merkletreejs`).  
2. Deploy verifier to **Scroll testnet**.  
