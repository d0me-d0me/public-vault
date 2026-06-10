# 調査端末(Kali)ツール 導入・更新手順

対象はツールの「インストール」と「バージョン/データ更新」のみ。エアギャップのため、取得は ネット接続端末で行い媒体で持込む。コマンド先頭に実行ホストを注記(`[ネット接続端末]` / `[調査端末]`)。git clone・pipx は使わない。

---

## 1. インストール(導入)

### 1-1. 標準収録(デフォルトで利用可・作業不要)

nmap, nikto, whatweb, gobuster, ffuf, sslscan, hydra, metasploit-framework, searchsploit(exploitdb), smbmap, onesixtyone, snmpwalk(snmp), netdiscover, wpscan、および dig/ldapsearch/ftp/curl 等の標準クライアント。

### 1-2. リポジトリ(apt・apt-offline で導入)

```bash
# [調査端末] 導入要求(対象パッケージ+依存)を生成
sudo apt-offline set ~/install_req.sig --install-packages \
  feroxbuster nuclei netexec ligolo-ng ligolo-ng-common-binaries \
  enum4linux-ng testssl.sh peass seclists odat chisel powersploit \
  smtp-user-enum unix-privesc-check masscan arp-scan \
  python3-impacket impacket-scripts gvm

# [ネット接続端末] 要求に基づきバンドル取得
apt-offline get ~/install_req.sig --bundle ~/transfer/apt-install.zip

# [調査端末] バンドル適用 → インストール実行
sudo apt-offline install /media/usb_in/apt-install.zip
sudo apt-get install -y feroxbuster nuclei netexec ligolo-ng ligolo-ng-common-binaries \
  enum4linux-ng testssl.sh peass seclists odat chisel powersploit \
  smtp-user-enum unix-privesc-check masscan arp-scan python3-impacket impacket-scripts gvm
```

配置先: `peass`→`/usr/share/peass/`(winPEAS/linPEAS)、`ligolo-ng-common-binaries`→`/usr/share/ligolo-ng-common-binaries/`、`powersploit`→`/usr/share/windows-resources/powersploit/`。
GVM 初期化: `sudo gvm-setup` で DB・証明書を作成(フィード同期はネット不可のため失敗してよい)。フィードは 2-5 の手順で取込み `sudo gvmd --rebuild`。

### 1-3. 未パッケージ(手動ダウンロード→導入)

```bash
# [ネット接続端末] ソースアーカイブ/単一スクリプトを取得して媒体へ
cd ~/transfer
wget https://github.com/vulnersCom/nmap-vulners/archive/refs/heads/master.tar.gz -O nmap-vulners.tar.gz
wget https://raw.githubusercontent.com/The-Z-Labs/linux-exploit-suggester/master/linux-exploit-suggester.sh -O les.sh
wget https://raw.githubusercontent.com/bitsadmin/wesng/master/wes.py -O wes.py
wget https://github.com/arthaud/git-dumper/archive/refs/heads/master.tar.gz -O git-dumper.tar.gz
wget https://github.com/blacklanternsecurity/MANSPIDER/archive/refs/heads/master.tar.gz -O manspider.tar.gz

# [調査端末] 持込後に導入
# nmap-vulners(NSE)
sudo tar xf /media/usb_in/nmap-vulners.tar.gz -C /usr/share/nmap/scripts/
sudo mv /usr/share/nmap/scripts/nmap-vulners-master /usr/share/nmap/scripts/nmap-vulners
sudo nmap --script-updatedb
# linux-exploit-suggester / WES-NG(単一スクリプト)
cp /media/usb_in/les.sh ./ && chmod +x les.sh
cp /media/usb_in/wes.py ./ && python3 wes.py --help >/dev/null
# git-dumper / MANSPIDER(ソース → pip)
tar xf /media/usb_in/git-dumper.tar.gz && pip install ./git-dumper-master --break-system-packages
tar xf /media/usb_in/manspider.tar.gz && pip install ./MANSPIDER-master --break-system-packages
```

WES-NG の定義DB(`wes.py --update`)もネット不可のため、ネット接続端末 で `--update` 実行後に生成物を持込む。

---

## 2. バージョン/データ更新

更新前に調査端末のスナップショットを取得(VM 推奨)。各取得物は ネット接続端末 で作成し媒体で持込む。

```bash
# 2-1 OS/パッケージ(apt-offline)
# [調査端末]
sudo apt-offline set ~/upgrade_req.sig --update --upgrade
# [ネット接続端末]
apt-offline get ~/upgrade_req.sig --bundle ~/transfer/apt-upgrade.zip
# [調査端末]
sudo apt-offline install /media/usb_in/apt-upgrade.zip
sudo apt-get update && sudo apt-get upgrade
#   注: Kali rolling は大規模更新になりやすい。範囲を絞るか更新済みイメージ展開も選択肢。

# 2-2 ExploitDB(searchsploit)
# [ネット接続端末]
searchsploit -u
sudo tar czpf ~/transfer/exploitdb.tgz -C / usr/share/exploitdb
# [調査端末]
sudo tar xzpf /media/usb_in/exploitdb.tgz -C /

# 2-3 Metasploit Framework
# [ネット接続端末]
sudo apt-get install --download-only -y metasploit-framework
cp /var/cache/apt/archives/metasploit-framework*.deb ~/transfer/
# [調査端末]
sudo dpkg -i /media/usb_in/metasploit-framework*.deb

# 2-4 nuclei(エンジンは 2-1 に含め、テンプレートを同期。版を揃える)
# [ネット接続端末]
nuclei -ut
tar czf ~/transfer/nuclei-templates.tgz -C ~/.config nuclei-templates
# [調査端末]
tar xzf /media/usb_in/nuclei-templates.tgz -C ~/.config/

# 2-5 OpenVAS/GVM フィード(停止→入替→再構築→起動)
# [ネット接続端末]
sudo greenbone-feed-sync
sudo tar czpf ~/transfer/gvm-feeds.tgz /var/lib/gvm /var/lib/notus /var/lib/openvas/plugins
# [調査端末]
sudo systemctl stop gvmd ospd-openvas
sudo tar xzpf /media/usb_in/gvm-feeds.tgz -C /
sudo gvmd --rebuild
sudo systemctl start gvmd ospd-openvas

# 2-6 KEV / EPSS(分析用データ)
# [ネット接続端末]
curl -s https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json -o ~/transfer/kev.json
curl -s https://epss.cyentia.com/epss_scores-current.csv.gz | gunzip > ~/transfer/epss.csv
# [調査端末]
cp /media/usb_in/kev.json /media/usb_in/epss.csv ~/assessment/analysis/
```

更新物の版・取得日を記録(調査結果の鮮度根拠)。不具合時はスナップショットへロールバック。

---

## 3. 導入確認

```bash
# [調査端末]
for t in nmap nuclei feroxbuster nxc ffuf gobuster nikto sslscan testssl.sh \
  hydra msfconsole searchsploit smbmap onesixtyone snmpwalk odat chisel \
  linpeas winpeas ligolo-proxy git-dumper manspider; do
  command -v "$t" >/dev/null && echo "OK  $t" || echo "--  $t (未導入)"; done
```

## Key takeaways
- 導入は 3 層: 標準収録(作業不要)、apt(apt-offline でバンドル取得→install)、未パッケージ(ネット接続端末 で wget 取得→媒体→tar/pip install。git clone・pipx 不使用)。
- 更新は OS(apt-offline)、ExploitDB、Metasploit、nuclei エンジン+テンプレ、GVM フィード(停止→入替→`--rebuild`→起動)、KEV/EPSS を ネット接続端末 取得→媒体同期。版・取得日を記録。
- すべて ネット接続端末↔媒体↔調査端末の持込前提。更新前スナップショットでロールバック可能にする。
