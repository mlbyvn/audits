# MathMasters Security Audit Report

## Overview

This document outlines the findings of a security audit conducted on **MathMasters**, a fixed point math library, constructed by the **[Cyfrin](https://www.cyfrin.io/)** team. The purpose of this audit is to highlight symbolic execution tools like [**Halmos**](https://github.com/a16z/halmos) and [**Certora Prover**](https://www.certora.com/prover) used to spot the vulnerabilities. 

## Tools and Techniques

The audit process leveraged the following tools and methodologies:

### 1. Using symbolic execution where fuzzing fails
In some cases fuzzing cannot spot the bug, even if the number of runs is cranked up. An alternative (or additional) approach could be **symbolic execution** -  program analysis technique used to explore multiple execution paths of a program by treating inputs as symbolic values rather than concrete values. 

---

## MathMasters Overview
MathMasters audit onboarding and code: https://github.com/Cyfrin/2-math-master-audit

**Disclaimer:**  
The vulnerabilities identified in MathMasters are introduced intentially. This repository is not intended for production use.
