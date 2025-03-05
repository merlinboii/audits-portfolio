# Portfolio
The repository showcases my experience in blockchain security, focusing on smart contract security audits, bug bounty contests, and Capture The Flag (CTF) events. Additionally, it features summary articles I've authored on smart contract security.

**Contributions** : Pashov Audit Group, Code4rena, Sherlock Audits, Cantina, CodeHawks, Immunefi, etc.

## Profile & Contact

### Contact
* [Telegram](https://t.me/merlinboii)

### Audit Contest
* [Code4rena](https://code4rena.com/@merlinboii) 
* [Sherlock](https://audits.sherlock.xyz/watson/merlinboii)
* [Immunefi](https://immunefi.com/profile/merlinboii/)

## Highlights
💡 **`FluidLocker::_getUnlockingPercentage()` will cause incorrect penalty calculations, impacting all users**
> The issue occurs because the calculation function's use of **incorrect scaling** and **does not properly convert days to seconds**, results in an incorrect penalty calculation.
>
> 🔗 [2024-11-superfluid-locking-contract-#64](https://github.com/sherlock-audit/2024-11-superfluid-locking-contract-judging/issues/64)
---
💡 **Liquidity provider loses Liquidity during collection initialization**
> The first liquidity provider loses ownership of their position during initialization because ƒlayer's uniswap hook becomes the position owner instead of the user.
>
> 🔗 [2024-09-ƒlayer-issues-#737](https://github.com/sherlock-audit/2024-08-flayer-judging/issues/737)
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

## Private Audit Engagements

### 🧑🏻‍💻 Pashov Audit Group: [🔗](https://www.pashov.net/)
> 🛡️: Undisclosed client | 🕙: Unpublished report

| Project     | Date | Report | 📂 |
| ---------------------------------------------------------------- | --   |:--:|:--:|
| 🛡️ - (CDP Stablecoin) | February 2025 | 🕙 | 🕙 |
| 🛡️ - (GameFi) | February 2025 | 🕙 | 🕙 |
| 🛡️ - (Deployment Scripts) | February 2025 | 🕙 | 🕙 |
| 🛡️ - (Private) | January 2025 | 🕙 | 🕙 |
| Dinari - Stablecoin | December 2024 | [.pdf](https://github.com/pashov/audits/blob/master/team/pdf/Dinari-security-review_2024-12-07.pdf) | [.md](./pashov-audit-group/md/2024-12-Dinari-security-review.md) |
| Ion - Lending| December 2024 | 🕙 | 🕙 |
| Nexus - Yield Aggregator| November 2024 | [.pdf](https://github.com/pashov/audits/blob/master/team/pdf/Nexus-security-review_2024-11-29.pdf) | [.md](./pashov-audit-group/md/2024-11-Nexus-security-review.md) |

---

### 🧑🏻‍💻 Valix Consulting: [🔗](https://valix.io/)

| Project                                                          | Date |
| ---------------------------------------------------------------- | --   |
| [FWX - Future Trading](https://fwx.finance/)| October 2024 |
| [FWX - Permissionless Future Trading](https://fwx.finance/)| October 2024 |
| [FWX - DeFi Perpetual Futures](https://fwx.finance/)| September 2024 |
| [REAME - Token & NFT Smart Contract](https://reame.io/)| April 2024 |
| [Starlet - Music NFT Smart Contract](https://www.starlet.world/)| April 2024 |
| [FWX - Permissionless Future Trading](https://fwx.finance/)| March 2024 |
| [**See more ↗**](https://github.com/valixconsulting/audit-reports) |  |

---

## Audit Contests

| Contest | Type | Awards | Findings | Language | Date | @ |Platform | Contest Report | My Report |
|:--:|:--:|:--:|:--:| ---- | -------- |:--:|:--:|:--:|:--:|
| [Superfluid Locker System](https://audits.sherlock.xyz/contests/648) | User's Locker of Money streaming protocol | 3rd 🥉 | 2H (reported in one) | Solidity | Nov 2024 | Individual | Sherlock | [📑](https://audits.sherlock.xyz/contests/648/report) | [💾](./sherlock/2024-11-superfluid-locking-contract.md) |
| [vVv Launchpad - Investments & Token distribution](https://audits.sherlock.xyz/contests/647) | Investments & Token distribution | 1st 🥇 | 1H | Solidity | Nov 2024 | Individual | Sherlock | [📑](https://audits.sherlock.xyz/contests/647/report) | [💾](./sherlock/2024-11-vvv-exchange-update.md) |
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
