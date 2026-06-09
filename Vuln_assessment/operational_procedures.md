# 運用手順

2台構成を前提とする。

- ステージング: インターネット接続。取得・検証・配布物作成を担う(調査結果は置かない)。
- 調査端末: エアギャップ。スキャン・分析を担う。オンライン取得は一切行わない。

授受はリムーバブルメディアで一方向(ステージング→調査端末。結果回収は別媒体で逆方向、媒体管理下)。コマンド先頭に実行ホストを注記する。

---

## A. エアギャップ運用(媒体授受)

### A-1. ステージングで集約・検証情報を付与

```bash
# [staging]
STAMP=$(date +%F)
mkdir -p ~/transfer/$STAMP && cd ~/transfer/$STAMP
# (各取得物は section B で配置)
sha256sum -- * > SHA256SUMS
gpg --output SHA256SUMS.sig --detach-sign SHA256SUMS   # 任意: 完全性のため署名
```

### A-2. 媒体へ書出し・マルウェアスキャン

```bash
# [staging]
clamscan -r ~/transfer/$STAMP                          # 持込前にAVスキャン(署名もstaging更新)
rsync -a ~/transfer/$STAMP/ /media/usb/$STAMP/
```

### A-3. 調査端末で検証してから適用

```bash
# [調査端末]
clamscan -r /media/usb/$STAMP                          # 二重スキャン
cd /media/usb/$STAMP && sha256sum -c SHA256SUMS
gpg --verify SHA256SUMS.sig SHA256SUMS                  # 署名運用時
# 検証通過後に section B の各適用へ
```

### A-4. 時刻設定(NTP 不可)

```bash
# [調査端末]
sudo timedatectl set-ntp false
sudo timedatectl set-time "2026-06-09 09:00:00"         # 信頼できる時刻に手動設定
date -u                                                 # 設定時刻を証跡に記録
```

### A-5. オフライン動作確認

```bash
# [調査端末]
searchsploit -h >/dev/null && echo "exploitdb: ローカルDB OK"
msfconsole -q -x "version; exit"
nmap --script-help vuln >/dev/null && echo "NSE: OK"
```

---

## B. 調査端末のオフライン更新

更新前に調査端末のスナップショット(VM 推奨)を取得しロールバック可能にする。各取得物は `~/transfer/$STAMP/` に置き、section A の手順で持込む。

### B-1. APT(apt-offline を主とする)

```bash
# [調査端末] 更新要求の署名ファイルを生成
sudo apt-offline set ~/transfer_req.sig --update --upgrade
# → req を媒体でステージングへ

# [staging] 署名に基づきパッケージ+依存をバンドル取得
apt-offline get ~/transfer_req.sig --bundle ~/transfer/$STAMP/apt-bundle.zip

# [調査端末] バンドルを適用(依存閉包込み)
sudo apt-offline install /media/usb/$STAMP/apt-bundle.zip
sudo apt-get update && sudo apt-get upgrade
```

代替: 少数パッケージなら `[staging] sudo apt-get install --download-only -y <pkgs>` で `/var/cache/apt/archives/*.deb` を収集 → 持込 → `[調査端末] sudo dpkg -i *.deb && sudo apt-get -f install`。大規模・恒常運用は apt-mirror でローカルミラーを作り `file:` リポジトリ登録。

### B-2. ExploitDB(searchsploit)

```bash
# [staging]
searchsploit -u                                         # /usr/share/exploitdb 更新
sudo tar czf ~/transfer/$STAMP/exploitdb.tgz -C / usr/share/exploitdb
# [調査端末]
sudo tar xzf /media/usb/$STAMP/exploitdb.tgz -C /
```

### B-3. Metasploit Framework

```bash
# [staging] 最新 .deb を取得
sudo apt-get install --download-only -y metasploit-framework
cp /var/cache/apt/archives/metasploit-framework*.deb ~/transfer/$STAMP/
# [調査端末]
sudo dpkg -i /media/usb/$STAMP/metasploit-framework*.deb
```

### B-4. nuclei テンプレート

```bash
# [staging]
nuclei -ut                                              # ~/.config/nuclei-templates 更新
tar czf ~/transfer/$STAMP/nuclei-templates.tgz -C ~/.config nuclei-templates
# [調査端末]
tar xzf /media/usb/$STAMP/nuclei-templates.tgz -C ~/.config/
```

### B-5. OpenVAS/GVM フィード

```bash
# [staging]
sudo greenbone-feed-sync
sudo tar czf ~/transfer/$STAMP/gvm-feeds.tgz \
  /var/lib/gvm /var/lib/notus /var/lib/openvas/plugins
# [調査端末]
sudo tar xzf /media/usb/$STAMP/gvm-feeds.tgz -C /
sudo gvmd --rebuild
```

### B-6. KEV / EPSS / NVD スナップショット

```bash
# [staging]
curl -s https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json \
  -o ~/transfer/$STAMP/kev.json
curl -s https://epss.cyentia.com/epss_scores-current.csv.gz | gunzip > ~/transfer/$STAMP/epss.csv
# [調査端末] 分析ディレクトリへ配置(取得日=鮮度をレポートに明記)
cp /media/usb/$STAMP/kev.json /media/usb/$STAMP/epss.csv ~/assessment/analysis/
```

### B-7. 記録・ロールバック

適用した更新物の版・取得日・SHA256 を更新台帳に記録(調査結果の鮮度根拠)。不具合時はスナップショットへロールバック。Kali rolling は部分更新で壊れやすいため、定期は完全再イメージ(更新済みイメージを媒体持込)も選択肢。

---

## C. 結果配布・対処状況管理

### C-1. 配布物の生成(技術詳細を分離)

```bash
# [調査端末] トラッカからシステム部門向けCSVを生成(認証情報・再現コマンドは除外)
mkdir -p ~/assessment/report/dist_$STAMP
# 列: 所見ID,ホスト,タイトル,深刻度,優先度,推奨対策,ステータス,対処期限
#   (技術詳細・PoC・資格情報は別添 appendix として分離)
cp findings_for_itdept.csv recommendations.pdf ~/assessment/report/dist_$STAMP/
```

### C-2. 暗号化・チェックサム・限定配布

```bash
# [調査端末] 配布パッケージを暗号化(GnuPG 対称鍵 または 7z AES-256)
tar c -C ~/assessment/report dist_$STAMP | gpg -c --cipher-algo AES256 -o dist_$STAMP.gpg
# もしくは
7z a -p -mhe=on dist_$STAMP.7z ~/assessment/report/dist_$STAMP/
sha256sum dist_$STAMP.gpg > dist_$STAMP.sha256
# 復号鍵は別経路で受け渡し。配布は限定・媒体管理下。
```

### C-3. やりとり様式(交換シートのカラム)

所見ID を一意キーに、相互記入する単一の交換シートを運用する。

| 所見ID | ホスト | タイトル | 深刻度 | 優先度 | 推奨対策 | 対処期限(SLA) | 対処内容(部門) | 完了日(部門) | エビデンス(部門) | 再検証結果 | 確認日 |
|---|---|---|---|---|---|---|---|---|---|---|---|

左側(指摘・優先度・SLA)は調査側が記入、中央(対処内容・完了日・エビデンス)はシステム部門が記入、右側(再検証結果・確認日)は調査側が記入。

### C-4. 対処状況ループ

Open → In Progress → Fixed(部門報告)→ retest → Verified/Closed または差戻し。Risk Accepted / False Positive も状態として保持。優先度別 SLA を設定(例: Critical=即時〜数日、High=2週間、Medium=1か月)。期限超過は別途エスカレーション。

### C-5. 再検証(retest)と差分追跡

```bash
# [調査端末] 完了報告(Fixed)の所見だけを再検証対象に抽出
awk -F, '$11=="Fixed"{print $1","$2}' itdept_returned.csv > retest_targets.csv   # ID,host
# 該当所見のみ再確認(例: ポート/スクリプトを所見に合わせて指定)
while IFS=, read -r id host; do
  echo "[$id] retest $host"
  nmap -p<port> --script <check> "$host" -oA retest_${id}
done < retest_targets.csv
# 前回確定所見と再検証結果を所見IDで突合し Verified/Closed か差戻しを判定
```

### C-6. 監査証跡・版管理

配布版(指摘)と回収版(対処報告)を版番号+日付で管理し、配布・対処・検証の各時刻と担当を記録する。エアギャップ間の授受も媒体ログに残す。

---

## Key takeaways
- 2台構成(ステージング=取得/検証/配布物作成、調査端末=スキャン/分析)を固定し、授受は一方向・AVスキャン・SHA256(必要なら署名)検証を経る。NTP 不可のため時刻は手動設定し証跡化。
- 更新は apt-offline を主(依存閉包を自動処理)、ExploitDB/Metasploit/nuclei テンプレ/GVM フィード/KEV・EPSS は staging で取得し tar/コピーで同期。版・取得日・ハッシュを台帳記録、ロールバック前提。
- 配布は技術詳細を分離した非専門家向け CSV/PDF を暗号化・限定配布。所見IDを一意キーにした交換シートで Open→Fixed→Verified を相互管理し、完了報告分のみ再検証して差分追跡。
