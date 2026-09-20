# Security

**Status: First design drafted — Provisional，尚未冻结**

当前第一版安全模型见 [architecture.md](architecture.md)，覆盖最小权限、服务隔离、SSH、供应链、漏洞响应、审计与事件响应方向。

仍需在 Security 阶段定稿（部分依赖 Network / Observability）：

- privilege model
- service-user isolation
- filesystem permissions / ACL
- credentials / token management
- SSH policy
- firewall
- exposed ports
- TLS certificates
- abuse / rate limiting
- LMT Server / Agent trust boundary
- package / supply-chain policy
- audit
- vulnerability and security update response
