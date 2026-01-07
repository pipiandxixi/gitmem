---
created_at: '2026-01-07T08:00:44.335298'
dependencies: []
tags:
- quantserver
- principles
title: quantserver_rules
updated_at: '2026-01-07T08:00:44.335298'
uuid: d3abbf8c-e5b3-4c6d-8561-5bb0407636b5
---

QuantServer Core Principles:
1. NO hidden defaults for quant variables (must be explicit/from config).
2. Transparent error reporting (no masking errors with fake 100% lines).
3. Strict data integrity validation before calculation.
4. Absolute logic parity between backtest and production modules.