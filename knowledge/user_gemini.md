---
title: user_gemini
tags: []
---
# user_gemini
 ... (truncated if too long) 

## Gemini Added Memories for QuantServer Project
- For QuantServer project, always follow these core principles:  
  1) NO hidden defaults for quant variables (must be explicit/from config);  
  2) Transparent error reporting (no masking errors with fake 100% lines);  
  3) Strict data integrity validation before calculation;  
  4) Absolute logic parity between backtest and production modules.  
- MANDATORY: All UI/Frontend changes in QuantServer must be verified locally before pushing to Render. Workflow:  
  1) Run local server via run_server.py;  
  2) Use browser tools to visit http://localhost:8000/view/QQQ/1y;  
  3) Check console for Syntax/Type errors;  
  4) Verify chart rendering and interactions.  
  Do NOT push if local verification fails.