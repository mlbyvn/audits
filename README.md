# Smart Contract Audit Repository

This repository contains some example and competitive audits of smart contracts, aiming to provide insights into the tools and best practices that are used.

## Purpose

The goal of this repository is to:  
- Showcase reference audits.  
- Provide **example audits** of intentionally vulnerable contracts to highlight common pitfalls.  

## Repository Structure

- **[Example Audits](./example-audits):**  
  Contains audits of mock protocols designed with intentional vulnerabilities.  
  - [TSwap](https://github.com/mlbyvn/audits/tree/main/example-audits/tswap): A fork of [Uniswap V1](https://docs.uniswap.org/contracts/v1/overview).
  - [ThunderLoan](https://github.com/mlbyvn/audits/tree/main/example-audits/thunderloan): A flash loan protocol based on [Aave](https://aave.com/docs) and [Compound](https://docs.compound.finance/).
  - [Boss Bridge](https://github.com/mlbyvn/audits/tree/main/example-audits/boss-bridge): a bridge protocol to move ERC20 tokens from L1 to L2.


- **[Competitive Audits](./competitive-audits):**  
  Reports of competitive audits from [Cyfrin CodeHawks](https://codehawks.cyfrin.io/) and other audit marketplaces.
  - [Alchemix Transmuter](): The strategy utilises Yearn V3 strategy template & builds on top of Alchemix providing an automated strategy which allows users to earn yield on Alchemix tokens (primiarly alETH) by taking advantage of potential       depegs. The strategy deposits to Alchemix's transmuter contract, an external keeper can claim alETH for WETH & execute a swap back to alETH at a premium to take advantage of any depeg of alETH vs WETH.

## Tools and Techniques

- **Preliminary Steps Before Auditing the Codebase** 
    1. Map Contract Interactions
       - Construct a detailed interaction graph for contracts, particularly in large and complex protocols. This visualization helps identify key dependencies and potential high-risk points in the architecture.
    2. Analyze Similar Protocols
       - Research the history of comparable protocols to uncover previously identified vulnerabilities.
       - Review audit reports from platforms like [Solodit](https://solodit.cyfrin.io/) for insights and lessons learned.
       - Compare the protocol in scope with its counterparts to pinpoint differences that might introduce new risks.
    3. Review Past Audits (if applicable)
       - Examine previous audit reports to understand the historical context, addressed issues, and remaining areas of concern.
    4. Evaluate External Dependencies
       - Scrutinize the libraries and frameworks used by the protocol for known vulnerabilities or unverified code. Ensure dependencies are up to date and trusted.
- **Static Analysis:** Tools like [Slither](https://github.com/crytic/slither) and [Aderyn](https://github.com/crytic/aderyn).  
- **Dynamic Testing:** Fuzz testing with [Echidna](https://github.com/crytic/echidna) and [Foundry](https://book.getfoundry.sh/), invariant-based approaches to detect runtime vulnerabilities.  
- **Manual Review:** Identifying nuanced logic errors and ensuring adherence to best practices. Using [Solodit checklist](https://solodit.cyfrin.io/checklist) for the initial review.

