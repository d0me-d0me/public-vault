# 脆弱性調査 実施要領

- 調査端末: Kali Linux (root)
- 対象OS: Windows 7-11 / Server, Linux, RHEL, Solaris
- スコープ前提: **AD(ドメイン)除外** / 外部(未認証)・内部(credentialed)を分離
- 記録: Excel(findings トラッカ)+ Obsidian(再現手順の一次ソース)
- ツールベース: `/home/kali/Desktop/Tool/`(`winset/`, `linset/`, `ligolo/`)、impacket は `~/venvs/pentest/bin/` 優先
- 共通変数(以降で使用):

```bash
T=10.10.10.10                                  # 対象ホスト
RANGE=10.10.10.0/24                            # 対象レンジ
LH=$(ip a show tun0 | grep "inet " | awk '{print $2}' | cut -d/ -f1)   # LHOST(動的)
```

---

## 使用ツール一覧

「対象OS」は各ツールを当てる対象側のOS(実行端末は Kali)。リンクは各フェーズに記載。

| 名前 | 作成者 / 企業 | 用途 | 対象OS |
|---|---|---|---|
| nmap | Nmap Project (Gordon Lyon) | ポート/サービス/OS判定・NSE | 全OS |
| masscan | Robert Graham | 高速ポートスイープ | 全OS |
| arp-scan | Roy Hills / NTA Monitor | L2(ARP)ホスト発見 | 全OS(L2) |
| netdiscover | Jaime Penalba | L2(ARP)ホスト発見 | 全OS(L2) |
| nmap-vulners | Vulners.com | CVE相関 NSE | 全OS |
| OpenVAS / GVM | Greenbone | 統合脆弱性スキャナ(FOSS) | 全OS |
| testssl.sh | Dirk Wetter | TLS/SSL 評価 | 全OS |
| sslscan | rbsec ほか | TLS/SSL 暗号列挙 | 全OS |
| WhatWeb | urbanadventurer (A. Horton) / B. Coles | Web 技術スタック特定 | 全OS(Web) |
| feroxbuster | epi052 (Ben Risher) | ディレクトリ探索(再帰) | 全OS(Web) |
| ffuf | Joona Hoikkala | Web ファジング/探索 | 全OS(Web) |
| gobuster | OJ Reeves | dir/vhost/DNS 探索 | 全OS(Web) |
| git-dumper | Maxime Arthaud | 露出 .git からソース復元 | 全OS(Web) |
| nuclei | ProjectDiscovery | テンプレート型脆弱性検出 | 全OS(Web/サービス) |
| Nikto | Chris Sullo / CIRT.net | Web サーバ脆弱性スキャン | 全OS(Web) |
| WPScan | WPScan Team (Automattic) | WordPress 脆弱性スキャン(無料APIは25req/日制限) | WordPress |
| SecLists | Daniel Miessler ほか | ワードリスト集 | 全OS(辞書) |
| enum4linux-ng | cddmp (M. Henze) | SMB/NetBIOS 列挙 | Windows/Samba |
| smbmap | Shawn Evans | SMB 共有再帰列挙 | Windows/Samba |
| NetExec (nxc) | NetExec team(CrackMapExec 後継) | SMB/MSSQL/WinRM 列挙・横展開 | Windows/Samba |
| MANSPIDER | Black Lantern Security | 共有内ファイル横断探索 | Windows/Samba |
| onesixtyone | Trail of Bits(現メンテナ) | SNMP community 総当り | 全OS(SNMP) |
| Net-SNMP (snmpwalk) | Net-SNMP Project | SNMP MIB 列挙 | 全OS(SNMP) |
| smtp-user-enum | pentestmonkey (Tim Brown) | SMTP ユーザ列挙 | 全OS(SMTP) |
| ODAT | Quentin Hardy | Oracle DB 攻撃 | 全OS(Oracle; Solaris/RHEL 重点) |
| THC-Hydra | van Hauser / THC | ログイン総当り | 全OS(各サービス) |
| winPEAS (PEASS-ng) | Carlos Polop | Windows 権限昇格列挙 | Windows |
| PowerUp (PowerSploit) | PowerShellMafia | Windows 権限昇格チェック | Windows |
| WES-NG | bitsadmin (A. Huijgen) | 欠落パッチ→CVE 特定 | Windows |
| linPEAS (PEASS-ng) | Carlos Polop | Linux 権限昇格列挙 | Linux/RHEL |
| linux-exploit-suggester | The-Z-Labs | カーネル exploit 候補 | Linux/RHEL |
| unix-privesc-check | pentestmonkey (Tim Brown) | Unix 汎用ローカルチェック | Linux/Solaris/Unix |
| ligolo-ng | Nicolas Chatelain | ピボット/トンネリング | 全OS |
| chisel | Jared Pillora | TCP/UDP トンネル | 全OS |
| searchsploit / Exploit-DB | OffSec | エクスプロイト検索 | 全OS |
| Metasploit Framework | Rapid7 | エクスプロイト/検証・補助モジュール | 全OS |
| impacket | Fortra(旧 SecureAuth / Core Security) | プロトコル操作・認証後処理 | Windows 中心 |
| CISA KEV | CISA(米 DHS) | 悪用中脆弱性カタログ(トリアージ) | 全OS(参照) |
| EPSS | FIRST.org | 悪用確率スコア(トリアージ) | 全OS(参照) |

上記のほか `dig`, `ldapsearch`(OpenLDAP), `ftp`, `mysql`/`psql`/`redis-cli`, `rpcinfo`/`showmount`, `curl` 等の標準クライアントを各列挙で使用。

> 有料/制限ツールは不採用: Nessus は除外し統合スキャナは OpenVAS/GVM(FOSS)。WPScan は本体 FOSS だが脆弱性APIが無料25req/日制限のため、無制限運用は nuclei の WordPress テンプレで代替可。

---

## ツール入手・環境構築(Kali)

Kali は rolling。多くは標準収録だが、近年追加分はリポジトリにあり apt で追加、ごく一部は未パッケージで手動導入。

### (1) 標準収録(デフォルトイメージ・即利用可)

nmap, nikto, whatweb, gobuster, ffuf, sslscan, hydra, metasploit-framework, searchsploit(exploitdb), smbmap, onesixtyone, snmpwalk(snmp), netdiscover, wpscan, および dig/ldapsearch/ftp/curl 等の標準クライアント。

### (2) リポジトリ(apt で1コマンド)

```bash
sudo apt update && sudo apt install -y \
  feroxbuster nuclei netexec ligolo-ng ligolo-ng-common-binaries \
  enum4linux-ng testssl.sh peass seclists odat chisel powersploit \
  smtp-user-enum unix-privesc-check masscan arp-scan \
  python3-impacket impacket-scripts gvm
```

- `peass` → `/usr/share/peass/`(winPEAS 各種 + linPEAS、コマンド `winpeas`/`linpeas`)
- `ligolo-ng-common-binaries` → `/usr/share/ligolo-ng-common-binaries/`(Linux/Windows エージェント同梱)
- `powersploit` → `/usr/share/windows-resources/powersploit/`(PowerUp.ps1 含む)
- OpenVAS/GVM の初期化:

```bash
sudo gvm-setup        # フィード同期(初回数十分〜)
sudo gvm-check-setup
sudo gvm-start        # https://127.0.0.1:9392
```

### (3) 未パッケージ(手動導入)

```bash
# nmap-vulners(NSE)
sudo git clone https://github.com/vulnersCom/nmap-vulners /usr/share/nmap/scripts/nmap-vulners
sudo nmap --script-updatedb

# pipx 系
pipx install git-dumper          # git-dumper
pipx install man-spider          # MANSPIDER(コマンド manspider)
pipx install wesng               # WES-NG(コマンド wes)

# linux-exploit-suggester(git)
git clone https://github.com/The-Z-Labs/linux-exploit-suggester ~/Desktop/Tool/linset/les
```

### 導入確認

```bash
for t in nmap nuclei feroxbuster nxc ffuf gobuster nikto sslscan testssl.sh \
  hydra msfconsole searchsploit smbmap onesixtyone snmpwalk odat chisel \
  linpeas winpeas ligolo-proxy git-dumper manspider wes; do
  command -v "$t" >/dev/null && echo "OK  $t" || echo "--  $t (未導入)"; done
```

---

## Phase 0: 準備・記録

```bash
mkdir -p ~/eng/{scope,scans,web,loot,report} && cd ~/eng/scans
printf '%s\n' "$RANGE" > ../scope/targets.txt   # RoE 承認済みレンジのみ

# 全コマンド出力を保全(再現性確保)
script -q -a ~/eng/session_$(date +%F).log
# もしくは tmux + tmux-logging プラグインで pane を自動ログ
```

RoE で対象内外・除外IP・スキャン時間帯・帯域上限を確定。報告で「未実施」と「対象外」を区別するため範囲を明文化。

---

## Phase 1: ホスト発見(外部/内部)

```bash
# IPv4 生存確認(複数プローブ)
nmap -sn -PE -PS22,80,443,445,3389 -PA80 -iL ../scope/targets.txt -oA discovery
grep Up discovery.gnmap | awk '{print $2}' > live.txt

# IPv6(デュアルスタックホストの取りこぼし防止)
nmap -6 -sn fe80::/64 -e eth1 -oA discovery6      # 内部リンクローカル
# 既知のIPv6レンジがあれば対象指定

# L2 発見(内部・FW有効でICMP/TCPに応答しないホスト)
sudo arp-scan -I eth1 --localnet
sudo netdiscover -i eth1 -r "$RANGE"

# 広域ポートスイープ(高速)
sudo masscan -p1-65535 --rate 1000 -iL live.txt -oG masscan.gnmap
```

- nmap https://nmap.org
- arp-scan https://github.com/royhills/arp-scan
- netdiscover https://github.com/netdiscover-scanner/netdiscover
- masscan https://github.com/robertdavidgraham/masscan

---

## Phase 2: ポート/サービス精査

```bash
# TCP 全ポート + バージョン + デフォルトスクリプト
sudo nmap -sS -sV -sC -p- --min-rate 2000 -oA tcp_$T $T

# UDP(top-ports だけでなく主要サービスを明示)
sudo nmap -sU -sV --top-ports 100 -oA udp_$T $T
sudo nmap -sU -p53,69,123,137,161,500,1434 -sV -oA udp_key_$T $T
```

`-min-rate`/タイミング(`-T3` 等)は対象負荷とIDSを見て調整。

---

## Phase 3: 既知脆弱性 NSE + 統合スキャナ

```bash
# vulners NSE 導入(初回のみ)
sudo git clone https://github.com/vulnersCom/nmap-vulners /usr/share/nmap/scripts/nmap-vulners
sudo nmap --script-updatedb

# 既知脆弱性スクリプト
sudo nmap -sV --script "vuln or vulners" -oA nse_$T $T
```

統合スキャナ(OpenVAS/GVM・FOSS、credentialed で内部精度↑):

```bash
sudo gvm-start                                   # https://127.0.0.1:9392
```

Credentials に Windows=SMB ローカル管理アカウント、Linux/RHEL/Solaris=SSH を設定して credentialed スキャンを実施。外部(未認証)と内部(credentialed)を別タスクで流し差分を取る。

- nmap-vulners https://github.com/vulnersCom/nmap-vulners
- OpenVAS/GVM(Kali導入) https://greenbone.github.io/docs/latest/22.4/kali/index.html

---

## Phase 4: TLS/SSL 評価(全OS・レガシーで頻出)

```bash
# 包括チェック(弱暗号・証明書・既知脆弱性 Heartbleed/POODLE 等)
testssl.sh https://$T | tee ../web/testssl_$T.txt
# 軽量
sslscan $T:443
# nmap 補完
nmap -p443 --script ssl-enum-ciphers,ssl-cert,ssl-heartbleed $T
```

- testssl.sh https://github.com/testssl/testssl.sh
- sslscan https://github.com/rbsec/sslscan

---

## Phase 5: Web アプリ探索

```bash
# 技術スタック特定
whatweb -a3 https://$T

# ディレクトリ/コンテンツ探索(再帰・拡張子拡張・ソフト404フィルタ)
feroxbuster -u https://$T \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,asp,aspx,jsp,html,txt,bak,old,zip,config,json,xml,env,git --depth 3
ffuf -u https://$T/FUZZ -ac \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -e .php,.bak,.zip,.old,.txt,.config -fc 404 -t 40

# バーチャルホスト列挙(DNSに無いvhost)
ffuf -u https://$T/ -H "Host: FUZZ.target.tld" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -ac

# VCS/設定ファイルの直撃
curl -s https://$T/.git/HEAD ; git-dumper https://$T/.git/ ./loot/git_$T
for f in .env web.config robots.txt sitemap.xml .DS_Store \
  swagger.json api-docs .well-known/security.txt; do
  echo "== $f =="; curl -s -o /dev/null -w "%{http_code}\n" https://$T/$f; done

# 認証後再列挙(未認証では見えない領域)
ffuf -u https://$T/FUZZ -H "Cookie: session=<token>" -ac -w <list>

# 既知脆弱性 / 設定不備
nuclei -u https://$T -severity medium,high,critical
nikto -h https://$T -ssl
# WordPress 検出時
wpscan --url https://$T --enumerate vp,u --api-token <token>
```

- feroxbuster https://github.com/epi052/feroxbuster
- ffuf https://github.com/ffuf/ffuf
- gobuster(vhost/dir 代替) https://github.com/OJ/gobuster
- git-dumper https://github.com/arthaud/git-dumper
- nuclei https://github.com/projectdiscovery/nuclei
- nikto https://github.com/sullo/nikto
- wpscan https://github.com/wpscanteam/wpscan
- SecLists https://github.com/danielmiessler/SecLists

高ノイズのため `-t`/`-rate` とフィルタで調整。WAF/レート制限を踏まえる。

---

## Phase 6: サービス別列挙

### SMB / ファイル共有(非AD・ワークグループでも有効)

```bash
enum4linux-ng -A $T | tee enum4linux_$T.txt
nxc smb $T -u '' -p '' --shares                  # null セッション
nxc smb $T -u user -p 'pass' --shares --users
smbmap -H $T -u user -p 'pass' -R                 # 再帰列挙
# アクセス可能共有から機密ファイル横断探索
manspider $T -u user -p 'pass'
```

### FTP / NFS / RPC

```bash
# 匿名FTP
ftp $T   # user: anonymous
# RPC / NFS エクスポート
rpcinfo -p $T
showmount -e $T
sudo mount -t nfs $T:/export /mnt/nfs -o nolock   # no_root_squash 確認
```

### SNMP / DNS / SMTP / LDAP

```bash
# SNMP
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt $T
snmpwalk -v2c -c public $T
# DNS ゾーン転送
dig axfr @$T target.tld
# SMTP ユーザ列挙
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t $T
# LDAP 匿名バインド(非AD: OpenLDAP 等)
ldapsearch -x -H ldap://$T -b "" -s base
ldapsearch -x -H ldap://$T -b "dc=target,dc=tld"
```

### データベース(Solaris/RHEL は Oracle 重点)

```bash
# MSSQL
nxc mssql $T -u sa -p 'pass'                       # PowerUpSQL は winset/ に同梱
# MySQL / PostgreSQL
mysql -h $T -u root -p ; nmap -p3306 --script mysql-* $T
psql -h $T -U postgres ; nmap -p5432 --script pgsql-brute $T
# Oracle TNS
odat all -s $T                                     # SID/アカウント/権限
odat sidguesser -s $T
# Redis / MongoDB 未認証
redis-cli -h $T ; nmap -p6379 --script redis-info $T
nmap -p27017 --script mongodb-info $T
```

### IPMI / iLO / iDRAC / プリンタ

```bash
# IPMI ハッシュ取得(CVE-2013-4786)
msfconsole -q -x "use auxiliary/scanner/ipmi/ipmi_dumphashes; set RHOSTS $T; run; exit"
nmap -sU -p623 --script ipmi-version $T
# Web 管理画面(iLO/iDRAC/プリンタ)は既定資格情報を Phase 7 で確認
```

- enum4linux-ng https://github.com/cddmp/enum4linux-ng
- smbmap https://github.com/ShawnDEvans/smbmap
- NetExec https://github.com/Pennyw0rth/NetExec
- MANSPIDER https://github.com/blacklanternsecurity/MANSPIDER
- onesixtyone https://github.com/trailofbits/onesixtyone
- net-snmp https://www.net-snmp.org
- smtp-user-enum https://github.com/pentestmonkey/smtp-user-enum
- odat https://github.com/quentinhardy/odat

---

## Phase 7: デフォルト/弱資格情報(横展開の起点)

```bash
# サービス別ブルートフォース
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://$T
hydra -L users.txt -P pass.txt rdp://$T
hydra -L users.txt -P pass.txt vnc://$T
hydra -l admin -P pass.txt $T http-post-form "/login:user=^USER^&pass=^PASS^:F=invalid"
# Web 既定ログインの一括確認
nuclei -u https://$T -t http/default-logins/
```

iLO/iDRAC/プリンタ/ネットワーク機器のベンダ既定資格情報を併せて試行。

- thc-hydra https://github.com/vanhauser-thc/thc-hydra

---

## Phase 8: OS別ローカル列挙(認証後)

### Windows(7-11 / Server・スタンドアロン)

```powershell
# /home/kali/Desktop/Tool/winset/ から対象へ配布して実行
.\winPEASx64.exe quiet cmd
powershell -ep bypass -c "Import-Module .\PowerUp.ps1; Invoke-AllChecks"
```

```bash
# 欠落パッチ → CVE(Kali側で解析)
wes.py systeminfo.txt
# ホストレベル脆弱性の確認(AD非依存)
nmap -p3389 --script rdp-ntlm-info,rdp-enum-encryption $T   # BlueKeep/CVE-2019-0708・NLA有無
nmap -p445 --script smb-vuln-ms17-010 $T                    # EternalBlue
# PrintNightmare(CVE-2021-34527): Spooler 稼働確認後に検証
```

- PEASS-ng(winPEAS) https://github.com/peass-ng/PEASS-ng
- PowerUp(PowerSploit) https://github.com/PowerShellMafia/PowerSploit
- WES-NG https://github.com/bitsadmin/wesng

### Linux / RHEL

```bash
# /home/kali/Desktop/Tool/linset/ から配布
./linpeas.sh -a | tee linpeas_$T.txt
./les.sh                                          # linux-exploit-suggester
# RHEL は backport によりバナー版数≠脆弱。実装版で照合:
rpm -qa --last | head ; rpm -q --changelog openssh | head
sestatus ; sudo -l
# sudo/polkit(PwnKit CVE-2021-4034 / Baron Samedit CVE-2021-3156)
sudo --version ; pkexec --version
```

- PEASS-ng(linPEAS) https://github.com/peass-ng/PEASS-ng
- linux-exploit-suggester https://github.com/The-Z-Labs/linux-exploit-suggester

### Solaris(SPARC/x86・**Linuxバイナリ非互換**)

```sh
# linpeas/pspy/les は不可。シェルベースで:
./unix-privesc-check standard > upc.txt
showrev -p ; pkginfo -l ; uname -a
# レガシーサービス面(telnet/rlogin/finger/rpc)は Phase 6 で列挙済み
```

- unix-privesc-check https://github.com/pentestmonkey/unix-privesc-check

### ファイル/ディレクトリ探索(全OS・privesc補完)

```bash
find / -perm -4000 -type f 2>/dev/null            # SUID
find / -perm -2 -type d 2>/dev/null               # world-writable ディレクトリ
find / -name "*.conf" -o -name "*.bak" -o -name "id_rsa" 2>/dev/null
grep -RiE "password|secret|api[_-]?key" /etc /opt /home 2>/dev/null
cat ~/.bash_history ~/.*_history 2>/dev/null
ls -la /var/spool/cron/crontabs /etc/cron* 2>/dev/null
```

---

## Phase 9: 横展開(AD代替 = ローカルアカウント再利用)

```bash
# スタンドアロン Windows 間の local admin 共有/パスワード再利用(PtH)
nxc smb ../scope/live.txt -u Administrator -H <NTLM> --local-auth --continue-on-success
nxc smb ../scope/live.txt -u Administrator -p 'pass' --local-auth
# 必要に応じてピボット(スコープ内)
# ligolo-ng(~/Desktop/Tool/ligolo/)/ chisel / sshuttle
```

- NetExec https://github.com/Pennyw0rth/NetExec
- ligolo-ng https://github.com/nicocha30/ligolo-ng
- chisel https://github.com/jpillora/chisel

---

## Phase 10: 検証・トリアージ

```bash
# エクスプロイト相関
searchsploit "openssh 7.2"
# 非破壊検証
msfconsole -q -x "use <module>; set RHOSTS $T; set LHOST $LH; check; exit"
# impacket(認証後の確認)
~/venvs/pentest/bin/secretsdump.py './Administrator:pass'@$T
```

CVSS に加え CISA KEV(実際に悪用中)と EPSS(悪用確率)を相関させ是正優先度を実態化。誤検知は必ず手動/`check`/`rpm -qa` で裏取りしてから所見化。

- searchsploit/ExploitDB https://gitlab.com/exploit-database/exploitdb
- Metasploit https://github.com/rapid7/metasploit-framework
- impacket https://github.com/fortra/impacket
- CISA KEV https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- EPSS https://www.first.org/epss

---

## Phase 11: 記録

- 所見の一覧・トリアージ・進捗: Excel findings トラッカ(CVSS入力→深刻度自動判定、系統/OSフィルタ)
- 再現手順の一次ソース: Obsidian(Markdown、コマンド+出力を保全)
- 系統(Windows/Linux/RHEL/Solaris)は Findings シートの「系統/OS」列でフィルタ・集計

---

## OPSEC / 注意点

- 統合スキャナ・ディレクトリ列挙・ブルートフォースは高ノイズ。`-rate`/`-t`/タイミングで調整し、対象負荷とIDS/EDRを考慮。
- 「広域発見 → 該当機のみ精査 → 認証後深掘り」と段階化してノイズと誤検知を抑制。
- Solaris は Linux 製バイナリが動かないため手動列挙へ切替。RHEL は backport で版数判定が崩れる点に常時注意。
