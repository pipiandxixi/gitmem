---
created_at: '2026-01-07T08:00:52.666798'
dependencies: []
tags:
- quantserver
- workflow
- frontend
title: quantserver_verification
updated_at: '2026-01-07T08:00:52.666798'
uuid: 8ec08d32-7b43-4a5e-932b-8c1d05a282c6
---

MANDATORY Workflow for QuantServer:
1. Run local server via run_server.py.
2. Use browser tools to visit http://localhost:8000/view/QQQ/1y.
3. Check console for Syntax/Type errors.
4. Verify chart rendering and interactions.
Do NOT push if local verification fails.