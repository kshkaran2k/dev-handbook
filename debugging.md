# Debugging & Root Cause Analysis (RCA) Guide

This document will guide for software engineers, tech leads, and solution-
architects to debug production issues systematically, identify root causes, 
and prevent recurrence.

-------------------------------------------------------------------------------

## 1. Debugging and RCA

### Debugging

**Goal:** Find and fix the immediate technical problem.

Typical questions:

- What is failing?
- Where is it failing?
- What changed?
- What data/input triggered it?
- What is the fastest safe mitigation?
- What code/configuration/infrastructure component is responsible?

### Root Cause Analysis (RCA)

**Goal:** Understand **why the problem happened and why it was not
prevented or detected earlier**.

Typical questions:

- Why did the failure happen?
- Why was the faulty condition possible?
- Why did our tests not catch it?
- Why did monitoring not detect it?
- Why did the deployment process allow it?
- What prevents the same class of issue from happening again?

> **Rule of thumb:** Debugging finds the failure. RCA explains the
> system of causes behind the failure.

-------------------------------------------------------------------------------

# 2. A General Debugging Framework

Can use this as the default process before choosing a specific RCA technique.

``` text
1. Detect
   ↓
2. Define the problem
   ↓
3. Stabilize / Mitigate
   ↓
4. Collect evidence
   ↓
5. Reproduce
   ↓
6. Narrow the scope
   ↓
7. Form hypotheses
   ↓
8. Test hypotheses
   ↓
9. Identify root cause
   ↓
10. Fix
   ↓
11. Validate
   ↓
12. Prevent recurrence
   ↓
13. Document RCA
```

-------------------------------------------------------------------------------
