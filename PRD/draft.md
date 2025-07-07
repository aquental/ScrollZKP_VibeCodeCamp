# **Product Requirements Document (PRD): _Spare_**  
**Version**: 1.0  
**Author**: Antonio Quental  
**Date**: 2025-07-07

---

## **1. Overview**  
### **Product Vision**  
*"Enable anyone to rent out underutilized space (garages, closets, driveways) seamlessly—with blockchain-backed trust and low fees."*  

### **Target Users**  
- **Hosts**: Homeowners and small businesses with idle space.  
- **Renters**: Urban dwellers, students, and small businesses needing storage.  

### **Key Differentiators**  
1. **Daily/Micro-Rentals** (vs. Traditional Monthly Leases).  
2. **ZK-proof reputation system** (on Scroll zkEVM).  
3. **Smart lock integration** (IoT access control).
4. **Integriry monitoring** (IoT monitoring).  

---

## **2. Objectives**  
| **Goal**                     | **Metric**                          |  
|-------------------------------|-------------------------------------|  
| Acquire 100 hosts             | Waitlist signups                   |  
| Reduce fraud incidents to <1% | On-chain reputation adoption       |  
| 30% cheaper than competitors  | Avg. rental price vs. Neighbor.com |  

---

## **3. Features & Requirements**  
### **Core Features**  
#### **A. User Roles**  
1. **Hosts**  
   - List space units (e.g., garage, closet) with photos, number of units, and pricing.  
   - Set availability calendar and access rules (e.g., "24/7" or "By appointment").
   - Commit to long or short-term rental.
2. **Renters**  
   - Search/filter spaces by location, size, and price.  
   - Book instantly or request approval.
   - Choose long or short-term rental.

#### **B. Trust & Safety**  
1. **Reputation System**  
   - On-chain SBTs (Soulbound Tokens) for reviews/ratings (stored on Scroll).  
   - ZK-proofs to verify identity without exposing personal data.  
2. **Insurance**  ❓
   - Optional $10K coverage (partner with local and/or traditional insurer).  

#### **C. Payments**  
1. **Smart Contract Escrow**  
   - Rent held in escrow until the rental period ends (released automatically).  
2. **Multi-Currency Support**  
   - ETH (for crypto-native users), USDC, or fiat via Stripe.  

#### **D. Access Control**  
1. **Smart Lock Integration**  
   - Renters receive temporary digital keys (via August/Yale APIs).  
2. **QR Code Check-In**  
   - Hosts generate one-time codes for in-person access.  

#### **E. Admin Dashboard**  
1. **Dispute Resolution**  
   - Hosts/renters can flag issues (triggering manual review).
2. **Promote my storage**
   - Renters will be able to promote their space for an additional fee (amount to be determined).

#### **F. Reward system**  
1. **Native token**
   - Gamify with the SPARE token.

---

## **4. Technical Specifications**  
### **Tech Stack**  
| **Layer**       | **Technology**                     |  
|------------------|------------------------------------|  
| **Frontend**     | Next.js (React), Tailwind CSS      |  
| **Backend**      | Node.js, Firebase (for non-blockchain data)|  
| **Blockchain**   | Scroll zkEVM, Solidity (smart contracts)|  
| **IoT**          | Smart Lock API              |  
| **Monitoring**   | Smart Monitoring API              |  

### **[Smart Contracts](smart.md)** (Scroll zkEVM)  
1. **RentalNFT.sol**  
   - ERC-721 representing each rental agreement.  
2. **Reputation.sol**  
   - SBTs for host/renter ratings.  

### **Data Flow**  
1. Renter books → Payment locked in escrow → NFT minted → Smart lock access granted.  
2. Rental ends → Payment released → NFT burned → Review SBT minted.  

---

## **5. User Flow**  
### **Host Flow**  
1. Sign up → Verify identity (ZK-proof).  
2. List space → Set price/availability → Connect smart lock (optional).  
3. Receive booking request → Approve → Payment locked in escrow.  
4. Rental ends → Auto-payout → Earn SBT for review.  

### **Renter Flow**  
1. Search → Filter → Book → Pay (crypto/fiat).  
2. Access the space via smart lock or QR code.  
3. Leave review → Earn SBT.  

---

## **6. Milestones & Timeline**  
| **Phase**       | **Deliverable**                    | **Timeline** |  
|------------------|------------------------------------|--------------|  
| MVP (Testnet)   | Smart contracts + basic listings   | Week 1      |  
| Beta (Scroll)   | ZK-reputation, insurance          | Week 2      |  
| Alpha (where?)   | 50 real hosts, IoT integration    | Week 3      |  


---

## **7. Risks & Mitigation**  
| **Risk**                          | **Solution**                          |  
|------------------------------------|---------------------------------------|  
| Low host adoption                 | Partner with REITs/storage companies.|  
| Regulatory hurdles (tokenization) | Start with non-NFT rentals in MVP.    |  
| Smart lock hacking                | Audit IoT APIs + use time-bound keys. |  
| Storage risks                   | Audit and monitor content            |

---

## **8. Metrics for Success**  
- **Host Acquisition Cost (CAC)**: <$50/host.  
- **Monthly Active Renters (MAU)**: 5,000 by Year 1.  
- **Avg. Host Earnings**: $300+/month.  

---

### **Next Steps**  
1. **Prioritize MVP features**: Start with listings + payments.  
2. **Wireframe review**: UI mockups.  
3. **Grant applications**: Target Scroll/EF grants for dev funding.  
