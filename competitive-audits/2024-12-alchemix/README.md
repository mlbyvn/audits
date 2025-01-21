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

### 3. **Manual Reviev**
In addition to automated tools and testing, a comprehensive manual review of the Alchemix Transmuter smart contracts was conducted. This process involved carefully examining the code to identify potential vulnerabilities, logic flaws, and areas where the implementation deviated from best practices.

---

**Disclaimer:**  
I as security researcher make all effort to find as many vulnerabilities in the code in the given time period,
but hold no responsibilities for the findings provided in this repo. A security audit is not an
endorsement of the underlying business or product. The audit was time-boxed and the review of the code
was solely on the security aspects of the Solidity implementation of the contracts. In order to ckeck other findings (if any), wisit the audit page on [Cyftin CodeHawks](https://codehawks.cyfrin.io/c/2024-12-alchemix/submissions/?filterCommunityJudgingDecision=%5B0%2C100%5D&filterCommunityJudgingMinVotes=0&filterDone=%7B%22value%22%3A%22all%22%2C%22label%22%3A%22-%22%7D&filterSelectedForReport=%7B%22value%22%3A%22all%22%2C%22label%22%3A%22-%22%7D&filterSeverity=%5B%7B%22value%22%3A%22high%22%2C%22label%22%3A%22High%22%2C%22disabled%22%3Afalse%7D%2C%7B%22value%22%3A%22medium%22%2C%22label%22%3A%22Medium%22%2C%22disabled%22%3Afalse%7D%2C%7B%22value%22%3A%22low%22%2C%22label%22%3A%22Low%22%2C%22disabled%22%3Afalse%7D%2C%7B%22value%22%3A%22unknown%22%2C%22label%22%3A%22Unknown%22%2C%22disabled%22%3Afalse%7D%5D&filterTags=%255B%255D&filterValidated=%7B%22value%22%3A%22all%22%2C%22label%22%3A%22All%22%7D&page=1&search=)


