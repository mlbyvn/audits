# ThunderLoan Security Audit Report

## Overview

This document outlines the findings of a security audit conducted on **ThunderLoan**, a flash loan protocol partially based on **[Compound](https://docs.compound.finance/)** and **[Aave](https://aave.com/docs)**, constructed by the **[Cyfrin](https://www.cyfrin.io/)** team. The purpose of this audit is threefold:

1. **Identify and analyze common vulnerabilities for similar protocols**.
2. **Provide insights into tools and techniques used during the audit process**.
3. **Promote best practices for secure development in decentralized finance (DeFi)**.

## Tools and Techniques

The audit process leveraged the following tools and methodologies:

### 1. Preliminary steps before audtiting the codebase:
I get acquainted with similar protocols (Compound, Aave) to uncover previously identified vulnerabilities and compare the protocol in scope with its counterparts to pinpoint differences that might introduce new risks. I also review audit reports of similar protocols from platforms like [Solodit](https://solodit.cyfrin.io/) for insights.

### 2. Static code analysis with **[Slither](https://github.com/crytic/slither)**  and **[Aderyn](https://github.com/Cyfrin/aderyn)**  

### 4. **Invariant Testing**  
Custom-built invariant tests were employed to verify the correctness and stability of the contract's core functionalities. These tests demonstrated potential exploitation scenarios and vulnerabilities in the TSwap implementation under unexpected conditions.

### 4. **Manual Reviev**
In addition to automated tools and testing, a comprehensive manual review of the TSwap smart contracts was conducted. This process involved carefully examining the code to identify potential vulnerabilities, logic flaws, and areas where the implementation deviated from best practices.

---

## ThunderLoan Overview
ThunderLoan audit onboarding and code: https://github.com/Cyfrin/6-thunder-loan-audit

**Disclaimer:**  
The vulnerabilities identified in ThunderLoan are introduced intentially. This repository is not intended for production use.
