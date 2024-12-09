# TSwap Security Audit Report

## Overview

This document outlines the findings of a security audit conducted on **TSwap**, a decentralized exchange (DEX) forked from **Uniswap V1** with intentionally introduced vulnerabilities, constructed by the [Cyfrin](https://www.cyfrin.io/) team. The purpose of this audit is threefold:

1. **Identify and analyze common vulnerabilities in DEX smart contracts**.
2. **Provide insights into tools and techniques used during the audit process**.
3. **Promote best practices for secure development in decentralized finance (DeFi)**.

## Tools and Techniques

The audit process leveraged the following tools and methodologies:

### 1. **[Slither](https://github.com/crytic/slither)**  

### 2. **[Aderyn](https://github.com/Cyfrin/aderyn)**  

### 3. **Invariant Testing**  
Custom-built invariant tests were employed to verify the correctness and stability of the contract's core functionalities. These tests demonstrated potential exploitation scenarios and vulnerabilities in the TSwap implementation under unexpected conditions.

### 4. **Manual Reviev**
In addition to automated tools and testing, a comprehensive manual review of the TSwap smart contracts was conducted. This process involved carefully examining the code to identify potential vulnerabilities, logic flaws, and areas where the implementation deviated from best practices.

---

## TSwap Overview
TSwap audit onboarding and code: https://github.com/Cyfrin/5-t-swap-audit

TSwap is a deliberately vulnerable fork of Uniswap V1. Its purpose is to showcase how subtle issues in DEX contracts can lead to significant security risks. 

**Disclaimer:**  
The vulnerabilities identified in TSwap are introduced intentially. This repository is not intended for production use.

---

## Getting started 

To run invariant tests just add "invariant" and "mocks" folders into the "test" folder of the foundry project in [cyfrin repo](https://github.com/Cyfrin/5-t-swap-audit).

