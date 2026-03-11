# Clash Verge 内网域名直连/解析失败处理经验

## 现象
- 规则已命中直连：`match DomainSuffix/uniontech.com`
- 连接仍失败：`dns resolve failed: couldn't find ip`
- 关闭代理后可访问内网域名

## 结论
- 问题不是代理规则，而是 Clash DNS 未使用内网 DNS 导致内网域名解析失败。

## 处理步骤
1. 在规则中确保 `uniontech.com` 直连（`DOMAIN-SUFFIX,uniontech.com,DIRECT`）。
2. 在 `dns_config.yaml` 增加域名定向 DNS，例如：
   - `nameserver-policy` 为 `+.uniontech.com` 指定内网 DNS（如 `10.20.0.10`）。
3. 关键：在 UI 中开启“DNS 复写/自定义 DNS 设置”（`enable_dns_settings: true`）。
4. 重载配置或重启内核使配置生效。

## 关键点
- 仅修改 `dns_config.yaml` 不够，必须在 UI/配置中启用 DNS 复写才能生效。
- 直连规则命中但 DNS 解析失败时，优先排查 DNS 配置是否走内网。
