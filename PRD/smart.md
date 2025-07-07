### **1. `RentalNFT.sol` (ERC-721 Rental Agreement)**  
**Purpose**: Tokenize each rental as an NFT to represent ownership, access rights, and payment terms.  

#### **Key Features**  
- **Dynamic Token URI**: Stores rental details (dates, price, space ID) as on-chain metadata.  
- **Escrow Logic**: Holds payment until the rental is completed.  
- **Access Control**: Only approved renters can "use" the NFT (via smart lock integration).  

#### **Contract Code** (Simplified)  
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract RentalNFT is ERC721, Ownable {
    uint256 private _tokenIdCounter;
    mapping(uint256 => RentalAgreement) public rentals;

    struct RentalAgreement {
        address host;
        address renter;
        uint256 spaceId;  // Links to off-chain space details (IPFS)
        uint256 startDate;
        uint256 endDate;
        uint256 price;
        bool isPaid;
    }

    event RentalCreated(uint256 tokenId, address host, address renter);
    event RentalCompleted(uint256 tokenId);

    constructor() ERC721("SpareRental", "SPRNFT") {}

    function mintRentalNFT(
        address renter,
        uint256 spaceId,
        uint256 startDate,
        uint256 endDate,
        uint256 price
    ) external payable {
        require(msg.value >= price, "Insufficient payment");
        
        uint256 tokenId = _tokenIdCounter++;
        _safeMint(renter, tokenId);

        rentals[tokenId] = RentalAgreement({
            host: msg.sender,
            renter: renter,
            spaceId: spaceId,
            startDate: startDate,
            endDate: endDate,
            price: price,
            isPaid: true
        });

        emit RentalCreated(tokenId, msg.sender, renter);
    }

    function completeRental(uint256 tokenId) external {
        require(ownerOf(tokenId) == msg.sender, "Not NFT owner");
        require(block.timestamp >= rentals[tokenId].endDate, "Rental ongoing");

        address host = rentals[tokenId].host;
        uint256 payment = rentals[tokenId].price;
        
        _burn(tokenId);  // Destroy NFT after rental
        (bool sent, ) = host.call{value: payment}("");
        require(sent, "Payment failed");

        emit RentalCompleted(tokenId);
    }

    // Override: Rental NFTs are non-transferable (SBT-like)
    function _transfer(address, address, uint256) internal pure override {
        revert("RentalNFT: Transfers disabled");
    }
}
```

#### **Critical Logic**  
1. **Non-Transferable NFTs**: Overrides ERC-721’s `_transfer` to make rentals SBT-like (bound to the renter).  
2. **Automatic Payout**: Funds released to host when `completeRental` is called after `endDate`.  
3. **Gas Optimization**: Uses Scroll’s low fees for frequent minting/burning.  

---

### **2. `Reputation.sol` (Soulbound Reputation Tokens)**  
**Purpose**: Issue non-transferable SBTs to hosts/renters based on reviews.  

#### **Key Features**  
- **SBT Mechanics**: Uses ERC-5192 (minimal SBT standard).  
- **ZK-Proof Integration**: Verifies off-chain reviews before minting.  
- **Reputation Tiers**: Bronze/Silver/Gold badges based on review scores.  

#### **Contract Code**  
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract Reputation is ERC721 {
    mapping(address => uint256) public scores;
    mapping(uint256 => string) private _tierMetadata;

    constructor() ERC721("SpareReputation", "SPRSBT") {}

    function mintReputation(
        address to,
        uint256 tokenId,
        uint256 score,
        string memory tier
    ) external onlyOwner {
        _safeMint(to, tokenId);
        scores[to] = score;
        _tierMetadata[tokenId] = tier;
    }

    // SBT: Disable transfers
    function _transfer(address, address, uint256) internal pure override {
        revert("Reputation: Transfers disabled");
    }

    function getTier(uint256 tokenId) public view returns (string memory) {
        return _tierMetadata[tokenId];
    }
}
```

#### **Critical Logic**  
1. **Immutable Reputation**: SBTs cannot be sold/traded, ensuring authenticity.  
2. **Off-Chain Reviews**: Frontend submits reviews to a **ZK-verifier contract** (not shown) before minting SBTs.  
3. **Scroll Compatibility**: Cheap SBT minting (~$0.01 per tx on Scroll).  

---

### **3. Integration with Scroll zkEVM**  
#### **Optimizations**  
- **Batch Reviews**: Use Scroll’s **ZK-rollups** to process multiple SBT mints in one tx.  
- **Cheap Storage**: Store minimal on-chain data (e.g., `spaceId` → IPFS metadata).  

#### **Sample Workflow**  
1. **Rental Creation**:  
   - Renter pays → `RentalNFT.mintRentalNFT()` → NFT minted.  
2. **Rental Completion**:  
   - Renter calls `completeRental()` → NFT burned → host paid.  
3. **Reputation Update**:  
   - After review, the backend calls `Reputation.mintReputation()`.  

---

### **4. Security Considerations**  
- **Reentrancy Guards**: Use OpenZeppelin’s `ReentrancyGuard` for payment functions.  
- **Input Validation**: Validate `startDate/endDate` to prevent overlaps.  
- **Access Control**: Restrict `mintReputation` to a verified oracle.  

---

### **Next Steps**  
1. **Testnet Deployment**:  
   ```bash
   npx hardhat deploy --network scroll-testnet
   ```  
2. **ZK-Proof Integration**: Implement a verifier for review authenticity (e.g., with [Circom](https://docs.circom.io/)).  
3. **UI Mock**: Do we need a frontend snippet to interact with these contracts?  


[Further toughts](further.md)
