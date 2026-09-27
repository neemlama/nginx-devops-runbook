# DevOps Beginner Project — Nginx Static Site + DNS (Local WSL + Oracle Cloud + DuckDNS)

Date: 2026-09-27
Status: Completed — Local 200 OK, Public 200 OK
Scope: Local-only then Cloud. Guidance + verification.

## 0. Objective
Serve a custom static site with nginx locally via fake DNS, then publicly via real DNS A record.
Learn: apt/nginx, server blocks by Host header, hosts file vs real DNS, Oracle VCN Security List + NSG, iptables order, SSH keys WSL vs Windows.

## 1. Prerequisites
- Windows with OpenSSH client, WSL2 Ubuntu 24.04
- Oracle Cloud Always Free account
- DuckDNS account
- Placeholders used below to hide sensitive data:
  - `<WSL_USER>` — WSL username
  - `<VM_PUBLIC_IP>` — Oracle VM public IPv4 (e.g., 129.154.x.x)
  - `<DUCKDNS_NAME>` — e.g., `<name>.duckdns.org`
  - `<SSH_KEY_PATH>` — private key path
  - `<FINGERPRINT>` — redacted host key fingerprint

## 2. Architecture
```
Browser -> mysite.local (127.0.0.1 via /etc/hosts) -> WSL nginx:80 -> /var/www/mysite/html
Internet -> <DUCKDNS_NAME> (A -> <VM_PUBLIC_IP>) -> Oracle VCN SL/NSG:80 -> VM iptables:80 -> VM nginx:80 -> /var/www/mysite/html
```

## 3. Part 1 — Local WSL Static Site

### 3.1 Verify WSL
PowerShell:
```
wsl -l -v
```
Expected: Ubuntu Running v2.
Inside WSL:
```
lsb_release -a
whoami
```
Result: Ubuntu 24.04.1 LTS.

### 3.2 Install nginx
WSL:
```
sudo apt update
sudo apt install -y nginx
nginx -v
```
Result: nginx/1.24.0 (Ubuntu). Master + workers running, listening 0.0.0.0:80.

### 3.3 Create site
WSL:
```
sudo mkdir -p /var/www/mysite/html
echo '<html><head><title>MySite</title></head><body><h1>It works - mysite</h1></body></html>' | sudo tee /var/www/mysite/html/index.html
ls -l /var/www/mysite/html
```
Verify: file exists, 80-110 bytes.

### 3.4 Server block + fake DNS
WSL — create `/etc/nginx/sites-available/mysite`:
```
server {
  listen 80;
  server_name mysite.local;
  root /var/www/mysite/html;
  index index.html;
  location / { try_files $uri $uri/ =404; }
}
```
Enable:
```
sudo ln -sf /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/mysite
sudo nginx -t
sudo service nginx reload
```
Fake DNS WSL:
```
echo "127.0.0.1 mysite.local" | sudo tee -a /etc/hosts
```
Windows (Admin) for Chrome: add to `C:\Windows\System32\drivers\etc\hosts`:
```
127.0.0.1 mysite.local
```
Note: WSL2 IP changes on reboot; if browser fails use `wsl hostname -I` IP instead of 127.0.0.1.

Verify WSL:
```
curl -s http://mysite.local/
```
Expected: custom HTML, not Welcome to nginx.
Verified: symlink exists, hosts entry present, curl returns custom page.

## 4. Part 2 — Cloud VM + Real DNS

### 4.1 Oracle VM creation
Console cloud.oracle.com:
- Compute > Instances > Create: Name `mysite-vm`, Image Ubuntu 24.04, Shape VM.Standard.E2.1.Micro (Always Free eligible)
- Network: new VCN, public subnet, Assign public IP Yes
- Add SSH public key (`ssh-ed25519 AAAA...REDACTED`)
- Create, wait Running, note `<VM_PUBLIC_IP>`.

### 4.2 DuckDNS A record
duckdns.org login > create `<DUCKDNS_NAME>` > set Current IP to `<VM_PUBLIC_IP>` > Save.
TTL auto ~60s.
Verify Windows PowerShell:
```
Resolve-DnsName <DUCKDNS_NAME>
```
Expected: A `<VM_PUBLIC_IP>`. Verified TTL ~27s.

### 4.3 SSH fix (WSL path issue)
Symptom: `ssh ubuntu@<VM_PUBLIC_IP>` Permission denied; `ssh -i $HOME\.ssh\...` fails with `/home/<WSL_USER>.sshoracle_mysite not accessible`.
Cause: Windows path syntax in Linux; key at `/mnt/c/Users/<WIN_USER>/.ssh/` with 777 perms rejected.
Fix WSL:
```
cp /mnt/c/Users/<WIN_USER>/.ssh/oracle_mysite ~/.ssh/oracle_mysite
chmod 600 ~/.ssh/oracle_mysite
ssh -i ~/.ssh/oracle_mysite ubuntu@<VM_PUBLIC_IP>
```
Result: `ubuntu@mysite-vnic` prompt. Host fingerprint `<FINGERPRINT>` redacted, accepted once.

### 4.4 Install nginx on VM
VM:
```
sudo apt update
sudo apt install -y nginx
sudo mkdir -p /var/www/mysite/html
echo '<html><head><title>MySite Cloud</title></head><body><h1>cloud works</h1></body></html>' | sudo tee /var/www/mysite/html/index.html
sudo tee /etc/nginx/sites-available/mysite <<'EOF'
server {
  listen 80;
  server_name <DUCKDNS_NAME> <VM_PUBLIC_IP>;
  root /var/www/mysite/html;
  index index.html;
  location / { try_files $uri $uri/ =404; }
}
EOF
sudo ln -sf /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/mysite
sudo nginx -t
sudo systemctl reload nginx
```
Initial symptom: `curl localhost` returned Welcome (default site), not custom. `nginx -T` showed both default and mysite loaded.
Fix: removed default:
```
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
Verify VM:
```
curl -s http://127.0.0.1/
curl -s -H "Host: <DUCKDNS_NAME>" http://localhost/
```
Expected both return cloud HTML. Verified Content-Length 101 matches custom file (Welcome is 615 bytes).

### 4.5 Firewall — iptables order
Symptom: local curl works, public `curl.exe ...` code 000, `Test-NetConnection -Port 80` False while 22 True.
VM check:
```
sudo iptables -L INPUT -n --line-numbers
```
Found: REJECT at 5, ACCEPT 80 at 6,7 after REJECT — never reached.
Fix VM:
```
sudo iptables -D INPUT 7
sudo iptables -D INPUT 6
sudo iptables -I INPUT 5 -p tcp --dport 80 -j ACCEPT
sudo iptables -L INPUT -n --line-numbers
sudo netfilter-persistent save
```
Correct order: ACCEPT 80 at 5 before REJECT at 6. `ufw` not installed — expected.

### 4.6 Firewall — Oracle VCN SL + NSG
Correct Ingress rule:
- Source Type: CIDR, Source: 0.0.0.0/0
- IP Protocol: TCP
- Source Port Range: All (empty) — NOT 80
- Destination Port Range: 80
- Description: HTTP mysite
Add 443 similarly for later HTTPS, ensure 22 allowed for SSH.
Root cause in this run: NSG allowed 80 but NSG not attached to instance VNIC. Fixed by attaching NSG to instance.
Verify Windows:
```
Test-NetConnection -ComputerName <VM_PUBLIC_IP> -Port 80
curl.exe -s http://<DUCKDNS_NAME>/
curl.exe -s -o NUL -w "code:%{http_code}" http://<DUCKDNS_NAME>/
```
Final: TcpTestSucceeded True, body cloud HTML, code 200.

## 5. Verification Summary
- `wsl nginx -v` → 1.24.0, listening 80
- `curl http://mysite.local/` → custom local page
- `Resolve-DnsName <DUCKDNS_NAME>` → A <VM_PUBLIC_IP>
- `curl http://<DUCKDNS_NAME>/` → 200 cloud page
- `Test-NetConnection <VM_PUBLIC_IP> 22 True, 80 True` after fix

## 6. Troubleshooting Log
1. `sudo service nginx status` + `curl localhost` hung 120s in automation — use `ps aux`, `ss -tlnp`, `curl --max-time 5`.
2. PowerShell `| grep/head` fails — use `wsl bash -c '... | grep ...'` or `curl.exe` without pipe.
3. `nginx -t` via automation hung on sudo password — verify via `ls sites-enabled`, `cat hosts`, `curl` without sudo.
4. Cloud curl Host header still Welcome — reload required, then remove default_server for beginner setup.
5. Still Welcome after remove — check `Content-Length`; 101 proved new config active despite stale manual curl output.
6. Public 000 — iptables order + unattached NSG + Source vs Destination port confusion.

## 7. Security Notes
- Redacted: private key body, `*.pub` full string, host ED25519 fingerprint, Oracle OCIDs/emails, Windows username path.
- Public by design (not redacted in live DNS): `<DUCKDNS_NAME>` and `<VM_PUBLIC_IP>` are world-readable via DNS. Masked here as placeholders per request.
- Do NOT commit `~/.ssh/oracle_mysite`, `known_hosts`, or Oracle console screenshots with OCIDs to git.
- Safe to push this runbook to GitHub (public or private) as-is with placeholders. If publishing real IP/DNS, prefer private repo or rotate after learning.

## 8. Next Options (Not Started)
- WWW CNAME: `www.<DUCKDNS_NAME>` alias, add to server_name, verify nslookup/curl.
- Real files deploy: multi-file site, `chown www-data`, try_files 404.
- HTTPS: open 443 SL/NSG/iptables, certbot --nginx, 80→443 redirect, renew.

## 9. Exact Command Reference (Copy-Paste, Replace Placeholders)
WSL local, VM cloud (see 3–4), Oracle console SL/NSG, DuckDNS web UI.
