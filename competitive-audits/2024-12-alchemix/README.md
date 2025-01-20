# Alchemix Transmuter Security Audit Report

## Overview

This document outlines the findings of a security audit conducted on **Alchemix Transmuter**, an automated strategy which allows users to earn yield on Alchemix tokens (primiarly alETH) by taking advantage of potential depegs built on top of Alchemix.

- [Documentation](https://docs.alchemix.fi/)
- [Code & Anboarding](https://github.com/Cyfrin/2024-12-alchemix)
- [Previous Audits](https://github.com/Cyfrin/2024-12-alchemix/issues/1)

---

## Protocol overview

The protocol is crude, not all intended functionality is implemented. Developers are aware of possible MEVs, though fo not implement any protection. It is also unclear, what oracle would be used to track if the token is depegged. I would also suggest using [Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html).

---

## Tools and Techniques

The audit process leveraged the following tools and methodologies:

### 1. **[Slither](https://github.com/crytic/slither)**  

### 2. **[Aderyn](https://github.com/Cyfrin/aderyn)**  

### 3. **Invariant Testing**  
Custom-built invariant tests were employed to verify the correctness and stability of the contract's core functionalities. These tests demonstrated potential exploitation scenarios and vulnerabilities in the TSwap implementation under unexpected conditions.

### 4. **Manual Reviev**
In addition to automated tools and testing, a comprehensive manual review of the TSwap smart contracts was conducted. This process involved carefully examining the code to identify potential vulnerabilities, logic flaws, and areas where the implementation deviated from best practices.

---

**Disclaimer:**  
The vulnerabilities identified in TSwap are introduced intentially. This repository is not intended for production use.

---

## Getting started 

To run invariant tests just add "invariant" and "mocks" folders into the "test" folder of the foundry project in [cyfrin repo](https://github.com/Cyfrin/5-t-swap-audit).
