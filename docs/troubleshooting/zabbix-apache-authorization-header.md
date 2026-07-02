# Troubleshooting: Zabbix API — "Not authorized" despite valid credentials

**Affected component:** Grafana → Zabbix datasource (`alexanderzobnin-zabbix-app` plugin)
**Environment:** Grafana 13.1.0, Zabbix 7.4.11, Ubuntu 26.04, PHP 8.5, Apache 2.4.66 + PHP-FPM
**Time to resolve:** ~2 sessions
**Status:** Resolved

---

## Symptoms

- `Save & test` in Grafana Zabbix datasource returned `"Could not connect to given url"` or hung for 60 seconds then failed with `"context deadline exceeded"`
- Direct `curl` to `http://10.20.0.10/zabbix/api_jsonrpc.php` worked perfectly (13ms, valid JSON-RPC response)
- `user.login` via curl returned a valid session token
- All network checks passed — 0% packet loss, correct routing through OPNsense

## What was eliminated

| Hypothesis | Test | Result |
|---|---|---|
| Network/firewall issue | `curl -X POST` from Grafana-srv to Zabbix API | ✅ 13ms, valid response |
| Wrong credentials | Direct web UI login | ✅ Successful |
| Zabbix account locked | Web login test | ✅ Not locked |
| Plugin binary permissions | `ls -la` on datasource binary | ✅ `-rwxr-xr-x`, owned by `grafana` |
| Unsigned plugin | Plugin signature in Grafana UI | ✅ Signed |
| No internet access for plugin checks | `curl` to `grafana.com/api/plugins/...` | ✅ 200 OK |
| Plugin version 6.4.0 | Downgraded to 6.3.0, then 6.2.0 | ❌ Same error |
| Grafana version 13.1.0 | Downgraded to 12.4.5 | ❌ Same error |
| API token auth | Generated Zabbix API token, tested via curl | ❌ `Not authorized` |
| Zabbix 7.4 auth breaking change | Tested `Authorization: Bearer` with session token | ❌ `Not authorized` |

## Root cause discovery

**tcpdump** on `grafana-srv` during a `Save & test` revealed the actual HTTP exchange:

```
# Step 1 — user.login succeeds
POST /zabbix/api_jsonrpc.php
{"method":"user.login","params":{"password":"zabbix","username":"Admin"}}
→ {"result":"a8ec8c7eaf423b688034588f4ed9b9e7"}  ✅

# Step 2 — hostgroup.get with Bearer token fails
POST /zabbix/api_jsonrpc.php
Authorization: Bearer a8ec8c7eaf423b688034588f4ed9b9e7
{"method":"hostgroup.get",...}
→ {"error":{"data":"Not authorized."}}  ❌
```

`user.login` worked, returned a valid token — but subsequent calls with `Authorization: Bearer <token>` were rejected.

**Key insight:** the same `Authorization: Bearer` call via curl worked fine when tested manually. The difference: curl hit Apache directly, but Grafana's plugin calls were proxied through PHP-FPM via a Unix socket.

**Root cause:** `php8.5-fpm.conf` was present in `/etc/apache2/conf-available/` but **not enabled** in `conf-enabled/`. This conf contains the critical directive:

```apache
SetEnvIfNoCase ^Authorization$ "(.+)" HTTP_AUTHORIZATION=$1
```

Without this, Apache strips the `Authorization` header before passing the request to PHP-FPM. PHP never sees the token, Zabbix never receives it, every authenticated call fails.

Confirmed by:
```bash
ls /etc/apache2/conf-enabled/ | grep fpm
# empty — php8.5-fpm.conf was not enabled
```

## Fix

```bash
sudo a2enconf php8.5-fpm
sudo systemctl reload apache2
```

**Result:** immediate fix. Next `Save & test` returned `"Zabbix API version 7.4.11"` ✅

## Reproduce

```bash
# Break it
sudo a2disconf php8.5-fpm
sudo systemctl reload apache2

# Fix it
sudo a2enconf php8.5-fpm
sudo systemctl reload apache2
```

## Why this happened

Zabbix 7.4 on Ubuntu 26.04 installs with `zabbix-apache-conf` which configures Apache to proxy PHP requests to PHP-FPM via a Unix socket (`proxy:unix:/var/run/php/zabbix.sock|fcgi://localhost`). The `php8.5-fpm.conf` that fixes the Authorization header passthrough is a **separate Apache conf** that ships with the `php8.5-fpm` package — it needs to be explicitly enabled with `a2enconf`. The Zabbix installer doesn't do this automatically.

This is easy to miss because:
1. The Zabbix web interface works fine without it (it doesn't use Bearer auth internally)
2. `curl` tests bypass the issue (they don't go through PHP-FPM in the same way)
3. The error (`Not authorized`) looks like a credentials problem, not an Apache config problem

## References

- [grafana/grafana-zabbix #1957](https://github.com/grafana/grafana-zabbix/issues/1957) — similar symptoms reported after Grafana upgrades
- [Grafana Community Forum thread](https://community.grafana.com/t/grafana-and-zabbix-plugin-issue) — `SetEnvIf Authorization` fix mentioned by user `jamesw6`
