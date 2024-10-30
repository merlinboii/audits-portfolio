# Portfolio
The repository showcases my experience in blockchain security, focusing on smart contract security audits, bug bounty contests, and Capture The Flag (CTF) events. Additionally, it features summary articles I've authored on smart contract security

## Profile
* [LinkedIn](https://linkedin.com/in/filmptz)
* [Code4rena (merlinboii)](https://code4rena.com/@merlinboii) 
* [Sherlock (merlinboii)](https://audits.sherlock.xyz/watson/merlinboii)
* [Immunefi (merlinboii)](https://immunefi.com/profile/merlinboii/)

## Highlights

💡 **Liquidity provider loses Liquidity during collection initialization**
> The first liquidity provider loses ownership of their position during initialization because ƒlayer's uniswap hook becomes the position owner instead of the user.
>
> 🔗 [2024-09-ƒlayer-issues-#737](https://github.com/sherlock-audit/2024-08-flayer-judging/issues/737)
---
💡 **Incorrect use of `1000` for converting basis points to decimals in `compoundedFactor_` calculation**
> The incorrect use of `1000` instead of `10000` for converting basis points to decimals leads to incorrect interest calculations.
>
> 🔗 [2024-09-ƒlayer-issues-#736](https://github.com/sherlock-audit/2024-08-flayer-judging/issues/736)
---
💡 **The attacker will prevent eligible users from claiming the liquidated balance**
> The combination of flawed logic allows an attacker to prevent eligible users from claiming their liquidated balance after external liquidation.
>
> 🔗 [2024-09-ƒlayer-issues-#742](https://github.com/sherlock-audit/2024-08-flayer-judging/issues/742)
---
💡 **Incorrect timestamp updating for invalid plots due to USD price fluctuation**
> Outdated plotMetadata.timestamp from varying configurations and external dependencies can lead to unfair rewards and potential DoS.
>
> 🔗 [2024-07-munchables-issues-#37](https://github.com/code-423n4/2024-07-munchables-findings/issues/37)
---
💡 **Users can farm on zero-tax land if the landlord locked tokens before the LandManager deployment**
> Oversight in contract validation allows users to stake with a 0% tax rate and farm schnibbles without paying tax.
>
> 🔗 [2024-07-munchables-issues-#30](https://github.com/code-423n4/2024-07-munchables-findings/issues/30)
---

## Team Audits

| Report                              | Date | Team   |
| ----------------------------------- | --   | ----   |
| [(Private) FWX - Future Trading](https://fwx.finance/)| October 2024 | [Valix](https://github.com/valixconsulting) |
| [(Private) FWX - Permissionless Future Trading](https://fwx.finance/)| October 2024 | [Valix](https://github.com/valixconsulting) |
| [(Private) FWX - DeFi Perpeptual Futures](https://fwx.finance/)| September 2024 | [Valix](https://github.com/valixconsulting) |
| [(Private) REAME - Token & NFT Smart Contract](https://reame.io/)| April 2024 | [Valix](https://github.com/valixconsulting) |
| [(Private) Starlet - Music NFT Smart Contract](https://www.starlet.world/)| April 2024 | [Valix](https://github.com/valixconsulting) |
| [(Private) FWX - Permissionless Future Trading](https://fwx.finance/)| March 2024 | [Valix](https://github.com/valixconsulting) |
| [BIG BANG THEORY - Smart Contract (Token & NFT)](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-TheBigbangTheory-Final-v1.0.pdf)| September 2023 | [Valix](https://github.com/valixconsulting) |
| [NFTGT Co., Ltd. - NFTGT Factory and Contract](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-CodeSekai-NFT-Minting-and-Transferring-In-game-Out-game-v1.1.pdf)| April 2023 | [Valix](https://github.com/valixconsulting) |
| [Code Sekai - NFT Minting & Transferring In-game/Out-game](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-CodeSekai-NFT-Minting-and-Transferring-In-game-Out-game-v1.1.pdf)| April 2023 | [Valix](https://github.com/valixconsulting) |
| [Xtatuz DMCC - XTATUZ asset tokenization](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-XtatuzDMCC-XTATUZ-Asset-Tokenization-v1.0.pdf)| March 2023 | [Valix](https://github.com/valixconsulting) |
| [Vega Investment Group Limited - CrownToken and VucaStaking](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-VegaInvestmentGroupLimited-CrownToken-and-VucaStaking-v1.0.pdf)| December 2022 | [Valix](https://github.com/valixconsulting) |
| [Vega Investment Group Limited - CrownToken](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-VegaInvestmentGroupLimited-CrownToken-v1.0.pdf)| November 2022 | [Valix](https://github.com/valixconsulting) |
| [Aniverse - ANIV721Land](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-Aniverse-ANIV721Land-v1.0.pdf)| September 2022 | [Valix](https://github.com/valixconsulting) |
| [Warden Finance - Wondrous-X](https://github.com/valixconsulting/audit-reports/blob/main/ValixConsulting-Audit-Report-WardenFinance-Wondrous-X-v1.0.pdf)| September 2022 | [Valix](https://github.com/valixconsulting) |

---

## Bug Bounty Contests

| Contest | Type | Awards | Findings | Language | Date | @ |Platform | Contest Report | My Report |
|:--:|:--:|:--:|:--:| ---- | -------- |:--:|:--:|:--:|:--:|
| [Flayer - NFT Liquidity Protocol](https://audits.sherlock.xyz/contests/468) | NFT Liquidity Protocol, Uniswap v4 Hooks | 28th | 8H, 2M | Solidity | Sep 2024 | Individual | Sherlock | [📑](https://audits.sherlock.xyz/contests/468/report) | [💾](./sherlock/2024-09-flayer.md) |
| [Midas - Instant Minter/Redeemer](https://audits.sherlock.xyz/contests/495) | RWA | 8th | 1M | Solidity | Aug 2024 | Individual | Sherlock | [📑](https://audits.sherlock.xyz/contests/495/report) |[💾](./sherlock/2024-08-midas-minter-redeemer.md)|
| [Munchables: LandManager](https://Code4rena.com/audits/2024-07-munchables) | GameFi, Staking, Farming | 1st 🥇 | 5H, 1M (1 selected for report) (cover ALL valid H/M) | Solidity | July 2024 | Individual | Code4rena | [📑](https://Code4rena.com/reports/2024-07-munchables) |[💾](./code4rena/2024-07-munchables.md)|
| [Biconomy: Nexus](https://codehawks.cyfrin.io/c/2024-07-biconomy) | Account Abstraction, Modular Smart Accounts| 27th | 1L (selected for report) | Solidity | July 2024 | Individual | CodeHawks | [📑](https://codehawks.cyfrin.io/c/2024-07-biconomy/results?lt=contest&sc=reward&sj=reward&page=1&t=report) | [💾](./codeHawks/2024-07-biconomy.md) |
| [Munchables: LockManager](https://Code4rena.com/audits/2024-05-munchables) | GameFi, Staking, Farming | 8th | 2H, 2M (1 selected for report) | Solidity | May 2024 | Individual | Code4rena | [📑](https://Code4rena.com/reports/2024-05-munchables) | [💾](./code4rena/2024-05-munchables.md) |
| [Jala Swap](https://audits.sherlock.xyz/contests/233) | AMM | 3rd 🥉 | 1M | Solidity | Mar 2024 | Individual | Sherlock | [📑](https://audits.sherlock.xyz/contests/233/report) | [💾](./sherlock/2024-02-jala-swap.md) |
| [UniStaker Infrastructure](https://Code4rena.com/audits/2024-02-unistaker-infrastructure) | Governance | Group of 5th | Grade-B QA Report | Solidity | Feb 2024 | Individual | Code4rena | [📑](https://Code4rena.com/reports/2024-02-uniswap-foundation) | [💾](./code4rena/2024-02-uniswap-foundation.md) |
| [AI Arena](https://Code4rena.com/audits/2024-02-ai-arena) | GameFi | 17th | 4H, 4M, Grade-B QA Report, Grade-B Gas Report | Solidity | Feb 2024 | Individual | Code4rena | [📑](https://Code4rena.com/reports/2024-02-ai-arena)| [💾](./code4rena/2024-02-ai-arena.md) |
| [Curves](https://Code4rena.com/audits/2024-01-curves) | SocialFi | 68th | 1H, 2M, Grade-A QA Report | Solidity | Jan 2024 | Individual | Code4rena | [📑](https://Code4rena.com/reports/2024-01-curves) | [💾](./code4rena/2024-01-curves.md) |

---

## Competitions
| Competition | Placed | Flag Captured | @ | Date | Provider |
|:--:|:--:| ---- | -------- |:--:|:--:|
| [Ethernaut CTF 2024](https://ctf.openzeppelin.com/) | 46th | [3rd-start.exe](https://ctf.openzeppelin.com/challenges#start.exe-6), [35th-Dutch](https://ctf.openzeppelin.com/challenges#Dutch-5), [15th-Alien Spaceship](https://ctf.openzeppelin.com/challenges#Alien%20Spaceship-2) | Individual | March 2024 | [OpenZeppelin](https://www.openzeppelin.com/) |
| [CTF_challenge_February 2024](https://github.com/AuditOneCTFs/CTF_challenge_Feb2024) | 2nd | [RollsRoyce](https://github.com/AuditOneCTFs/CTF_challenge_Feb2024/blob/main/RollsRoyce.sol) | Individual | February 2024 | [AuditOne](https://www.auditone.io/) |

---

## Blogs
| Title | Date |
|-------|------|
| [Deployment to Defense Security Strategies for Blockchain Protocols](https://medium.com/valixconsulting/deployment-to-defense-security-strategies-for-blockchain-protocols-8c4c714365a7) | October 2024 |
| [Openzeppelin Ethernaut CTF 2024 — Alien Spaceship Writeup](https://link.medium.com/OVLyVYzB5Hb ) | March 2024 |
| [Something Behind the — SELFDESTRUCT —](https://filmptz.medium.com/something-behind-the-selfdestruct-6ec46e007440) | January 2024 |
| [Upgradeable Notes - Disable initializer](https://x.com/m3rlinbx0/status/1742940872562590079?s=20) | January 2024 |
| [Breakdown of Rollups — Layer 2 Scaling Solution](https://medium.com/valixconsulting/breakdown-of-rollups-layer-2-scaling-solution-afe73ebb0bec) | November 2023 |
| [Is Dead Code Really Dead?](https://medium.com/valixconsulting/is-dead-code-really-dead-3578b36e0a91) | September 2023 |
| [Risky UUPS Pattern 💣](https://filmptz.medium.com/risky-uups-pattern-8ff0fdc424ba) | May 2022 |
| [Deep dive into UniswapV2🦄 : UniswapV2Router02](https://filmptz.medium.com/deep-dive-into-uniswapv2-uniswapv2router02-55b500342295) | May 2022 |
| [Deep dive into UniswapV2🦄 : UniswapV2Factory](https://filmptz.medium.com/deep-dive-into-uniswapv2-uniswapv2factory-620c5950f928) | May 2022 |
| [Deep dive into UniswapV2🦄 : UniswapV2Pair](https://coinsbench.com/deep-dive-into-uniswapv2-uniswapv2pair-e88f0ed3bb6e) | May 2022 |
| [Deep dive into UniswapV2🦄 : UniswapV2ERC20](https://filmptz.medium.com/deep-dive-into-uniswapv2-uniswapv2erc20-ab50dfcccc30) | May 2022 |
| [Ethereum smart contract CTFs — Review](https://coinsbench.com/ethereum-smart-contract-ctfs-review-d7c6da726102) | May 2022 |
