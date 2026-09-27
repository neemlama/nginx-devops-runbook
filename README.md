# nginx-devops-runbook

DevOps beginner project: Nginx static site + DNS, local WSL then Oracle Cloud + DuckDNS.

- Local: `mysite.local` via hosts file, nginx 1.24.0, custom `/var/www/mysite/html`
- Cloud: DuckDNS A record to VM public IP, nginx server block, SL/NSG + iptables port 80, verified 200 OK

See `docs/2026-09-27_nginx-devops-project-full-runbook.md` for exact redacted steps. No secrets committed (placeholders for IP/DNS/keys).
