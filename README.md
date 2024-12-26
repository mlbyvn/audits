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
  - [ThunderLoan](): A flash loan protocol based on [Aave](https://aave.com/docs) and [Compound](https://docs.compound.finance/).


- **[Competitive Audits](./competitive-audits):**  
  Reports of competitive audits from [Cyfrin CodeHawks](https://codehawks.cyfrin.io/) and other audit marketplaces.
  - [Alchemix Transmuter](): The strategy utilises Yearn V3 strategy template & builds on top of Alchemix providing an automated strategy which allows users to earn yield on Alchemix tokens (primiarly alETH) by taking advantage of potential       depegs. The strategy deposits to Alchemix's transmuter contract, an external keeper can claim alETH for WETH & execute a swap back to alETH at a premium to take advantage of any depeg of alETH vs WETH.

## Tools and Techniques

The audits in this repository leverage a combination of industry-standard tools and manual techniques, including:  
- **Static Analysis:** Tools like [Slither](https://github.com/crytic/slither) and [Aderyn](https://github.com/crytic/aderyn).  
- **Dynamic Testing:** Fuzz testing and invariant-based approaches to detect runtime vulnerabilities.  
- **Manual Review:** Identifying nuanced logic errors and ensuring adherence to best practices.

## Useful resources

