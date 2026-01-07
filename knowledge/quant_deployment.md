---
title: quant_deployment
tags: ['deployment', 'config']
---
# quant_deployment

## QuantServer Deployment Config (Update)
- Python version requirement: 3.12 (supersedes 3.11)
- Default port: 9090
- WSGI server: gunicorn with 4 workers
- Access log path: /var/log/qs/access.log