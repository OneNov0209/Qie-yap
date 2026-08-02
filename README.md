# QIE Yap — The First On-Chain Social Network on QIE Mainnet

QIE Yap is a **hybrid on-chain social platform** built on the **QIE blockchain**.  
It combines the familiarity of Web2 social media with the transparency and monetization of Web3.  
Users can post "Yaps", upload images, share link previews, send on-chain **Tips**, create **Super Yaps**, and **Boost** posts — all powered by smart contracts and a self-hosted PostgreSQL backend.

---

## 🚀 Live Demo

- **Website:** [https://qie.yap.onenov.xyz/](https://qie.yap.onenov.xyz/)
- **Blockchain Explorer (QIE Mainnet):** [https://qie.explorer.onenov.xyz](https://qie.explorer.onenov.xyz)

---

## 📦 Tech Stack

| Layer | Technology |
|-------|------------|
| **Blockchain** | QIE Mainnet (Chain ID: 1990) |
| **Smart Contract** | Solidity ^0.8.24 |
| **Frontend** | React + Vite + TanStack Router |
| **Backend** | Node.js + Express + PostgreSQL (Self-hosted on VPS) |
| **Storage / IPFS** | Pinata (For image/media uploads) |
| **Hosting** | Vercel (Frontend) & Contabo VPS (Backend) |
| **Wallet Integration** | QIE Wallet (Official) / MetaMask (via `viem`) |

---

## 🔗 Smart Contract (QIE Mainnet)

- **Network:** QIE Mainnet (Chain ID: 1990)
- **RPC URL:** `https://rpc-evm.qie.onenov.xyz`
- **Explorer:** `https://qie.explorer.onenov.xyz`
- **Contract Address:** `0x2A433A09e0Ce2766affb89E774BbdC3c29044D0a`
- **Treasury / Dev Wallet:** `0x109b18b9cb9ec1C9CED114A13FfA86F8c13CFa7a`

### 🧾 Contract Source Code (`QieYap.sol`)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract QieYap {
    address public owner;
    address public treasury;
    address public devWallet;
    
    uint256 public constant SUPER_YAP_PRICE = 0.01 ether;
    uint256 public constant BOOST_PRICE_PER_HOUR = 0.005 ether;
    uint256 public constant PLATFORM_FEE_PERCENT = 2;
    
    uint256 public totalTipsSent;
    uint256 public totalSuperYaps;
    uint256 public totalBoosts;
    
    mapping(address => uint256) public tipsSent;
    mapping(address => uint256) public tipsReceived;
    mapping(address => uint256) public superYapsCreated;
    mapping(address => uint256) public boostsCreated;
    
    event TipSent(address indexed from, address indexed to, uint256 amount, string message);
    event SuperYapCreated(address indexed user, uint256 timestamp);
    event PostBoosted(address indexed user, uint256 indexed postId, uint256 durationHours, uint256 amount);
    event Yapped(address indexed author, bytes32 indexed contentHash, uint256 timestamp, string cid);
    
    constructor(address _treasury, address _devWallet) {
        owner = msg.sender;
        treasury = _treasury;
        devWallet = _devWallet;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function yap(bytes32 contentHash, string calldata cid) external {
        emit Yapped(msg.sender, contentHash, block.timestamp, cid);
    }
    
    function tip(address recipient, string calldata message) external payable {
        require(msg.value > 0, "Tip must be > 0");
        require(recipient != address(0), "Invalid recipient");
        require(recipient != msg.sender, "Cannot tip yourself");
        
        uint256 devFee = (msg.value * PLATFORM_FEE_PERCENT) / 100;
        uint256 recipientAmount = msg.value - devFee;
        
        (bool feeSent, ) = payable(devWallet).call{value: devFee}("");
        require(feeSent, "Fee transfer failed");
        
        (bool sent, ) = payable(recipient).call{value: recipientAmount}("");
        require(sent, "Transfer failed");
        
        tipsSent[msg.sender] += recipientAmount;
        tipsReceived[recipient] += recipientAmount;
        totalTipsSent += recipientAmount;
        
        emit TipSent(msg.sender, recipient, recipientAmount, message);
    }
    
    function createSuperYap() external payable {
        require(msg.value >= SUPER_YAP_PRICE, "Insufficient fee");
        
        uint256 burnAmount = msg.value / 2;
        (bool burnSuccess, ) = payable(0x000000000000000000000000000000000000dEaD).call{value: burnAmount}("");
        require(burnSuccess, "Burn failed");
        
        uint256 treasuryAmount = msg.value - burnAmount;
        (bool treasurySuccess, ) = payable(treasury).call{value: treasuryAmount}("");
        require(treasurySuccess, "Treasury transfer failed");
        
        superYapsCreated[msg.sender]++;
        totalSuperYaps++;
        
        emit SuperYapCreated(msg.sender, block.timestamp);
    }
    
    function boost(uint256 postId, uint256 durationHours) external payable {
        require(durationHours > 0 && durationHours <= 72, "Duration must be 1-72 hours");
        uint256 cost = BOOST_PRICE_PER_HOUR * durationHours;
        require(msg.value >= cost, "Insufficient fee");
        
        (bool sent, ) = payable(treasury).call{value: msg.value}("");
        require(sent, "Transfer failed");
        
        boostsCreated[msg.sender]++;
        totalBoosts++;
        
        emit PostBoosted(msg.sender, postId, durationHours, msg.value);
    }
    
    function updateTreasury(address _treasury) external onlyOwner {
        treasury = _treasury;
    }
    
    function updateDevWallet(address _devWallet) external onlyOwner {
        devWallet = _devWallet;
    }
    
    function withdrawTips() external {
        uint256 amount = tipsReceived[msg.sender];
        require(amount > 0, "No tips to withdraw");
        tipsReceived[msg.sender] = 0;
        (bool sent, ) = payable(msg.sender).call{value: amount}("");
        require(sent, "Withdraw failed");
    }
    
    function getUserStats(address user) external view returns (
        uint256 _tipsSent,
        uint256 _tipsReceived,
        uint256 _superYaps,
        uint256 _boosts
    ) {
        return (
            tipsSent[user],
            tipsReceived[user],
            superYapsCreated[user],
            boostsCreated[user]
        );
    }
    
    function getGlobalStats() external view returns (
        uint256 _totalTips,
        uint256 _totalSuperYaps,
        uint256 _totalBoosts
    ) {
        return (totalTipsSent, totalSuperYaps, totalBoosts);
    }
}
```

Contract Highlights

Feature Details
Tip 2% platform fee → devWallet, 98% → recipient wallet instantly
Super Yap 50% burned, 50% → treasury (Creates premium gold border posts)
Boost 100% → treasury (Promotes posts with a "Promoted" badge)
Yap Emits on-chain event hash (content stored off-chain in PostgreSQL)

---

🗄️ Backend Database (PostgreSQL on VPS)

All data is stored in a self-hosted PostgreSQL database managed via a custom Node.js + Express API.

Key Tables

Table Purpose
profiles User profiles linked to wallet addresses
posts Yap posts (content, media_url, link_preview, is_boosted, is_super_yap)
likes User likes on posts
reposts User reposts
bookmarks User bookmarks
follows Follower relationships
tips On-chain tip transaction records
comments Replies to posts
notifications Aggregated real-time notifications (Follow, Like, Reply, Tip)

---

✨ Core Features

Feature Description
✅ Post Yaps (Text) Unlimited characters (max 2500) with Markdown-like support.

✅ Image Upload Upload images directly via Pinata IPFS.

✅ Link Preview Auto-fetches OG metadata (title, image, description) when pasting a URL.

✅ On-Chain Tips Send QIE directly to creators. 2% platform fee, 98% to recipient.

✅ Super Yaps Premium post with gold border. Burns 50% of the fee.

✅ Boost Promoted post badge. Shows "⚡ Promoted" indicator in feed.

✅ Social Interactions Follow, Unfollow, Like, Repost, Bookmark.

✅ Real-Time Feed "Latest" feed with real-time updates via Supabase (deprecated) / Polling.

✅ Leaderboard Top creators ranked by total tips received.

✅ Dashboard View tips received, wallet balance, and withdraw earnings.

✅ Explorer On-chain activity explorer (Tips, Yaps, Super Yaps, Boosts).

✅ Notifications Real-time notifications for Follows, Likes, Replies, and Tips.

✅ Delete Post Users can delete their own posts (Cascading deletion).

---

🎨 Design Philosophy

· Futuristic UI with glassmorphism, subtle gold dot patterns, and neon gradients (#FF0A6C → #0040FF).

· Responsive Design with a dedicated sidebar for desktop and a bottom navigation bar for mobile.

· Premium Typography using Inter, Space Grotesk, and JetBrains Mono.

---

⚙️ Environment Variables (Vercel)

Key Value
```
VITE_API_BASE https://api.yap.onenov.xyz/api
VITE_QIE_YAP_CONTRACT 0x2A433A09e0Ce2766affb89E774BbdC3c29044D0a
VITE_DEV_WALLET 0x109b18b9cb9ec1C9CED114A13FfA86F8c13CFa7a
VITE_PINATA_JWT (Your Pinata JWT for IPFS uploads)
VITE_APP_URL https://qie.yap.onenov.xyz
```
---

🛠️ Local Development

Clone the repository and install dependencies:

```bash
git clone https://github.com/OneNov0209/qie-yap-social.git
cd qie-yap-social
bun install   # or npm install
```

Create a .env file with the variables listed above, then run:

```bash
bun run dev
```

---

🚀 Deployment

The frontend is deployed on Vercel via GitHub integration.
The backend is hosted on a Contabo VPS running Ubuntu 22.04 with Node.js v22.

To redeploy manually:

1. Push changes to the main branch.
2. Go to Vercel Dashboard → Project → Deployments.
3. Click Redeploy (ensure "Use existing build cache" is unchecked).

---

🔐 Security Notes

· Never commit .env or VITE_PINATA_JWT to GitHub.

· All sensitive keys are stored only in Vercel Environment Variables.

· The backend uses parameterized SQL queries to prevent injection attacks.

· Only post owners can delete their own posts (server-side validation).

---

📄 License

MIT License — feel free to use and modify this project for your own purposes.

---

🏆 Acknowledgments

Built for the QIE Hackathon 3rd Edition (2026).
Special thanks to the QIE ecosystem, the OneNov for frontend scaffolding, and the Contabo VPS for backend hosting.

---

Happy Yapping! 🚀✨

```
