# 脆弱性分析 具体的手順

収集フェーズ(Phase 0-9)の出力を「確定所見+対策+証跡」へ変換する手順。前提は FOSS のみ・AD除外・系統混在(Windows/Linux/RHEL/Solaris)。入力は `~/assessment/scans`(nmap XML/gnmap, NSE)、`~/assessment/web`(nuclei JSON 等)、OpenVAS エクスポート、手動列挙ノート。

共通変数: `T`=対象ホスト。作業は `~/assessment/analysis`。

---

## Step 0: 準備

```bash
mkdir -p ~/assessment/analysis && cd ~/assessment/analysis
command -v jq >/dev/null || sudo apt install -y jq
# nmaptocsv(任意・手動DL、git clone不使用)
wget https://raw.githubusercontent.com/maaaaz/nmaptocsv/master/nmaptocsv.py -O nmaptocsv.py
```

---

## Step 1: 集約・正規化

```bash
# 全ホストの open ポート(grepable から)
cat ../scans/tcp_*.gnmap | grep -E "Host:|/open/"

# サービス一覧を CSV 化(任意)
python3 nmaptocsv.py -i ../scans/tcp_$T.xml \
  -f ip-fqdn-port-protocol-service-version > hosts_services.csv

# nuclei JSON → host/severity/template/CVE を抽出
jq -r '[.host, .info.severity, .["template-id"],
        ((.info.classification?.["cve-id"]//[])|join(";"))]|@tsv' \
   ../web/nuclei_$T.json

# OpenVAS は Web UI から CSV エクスポート → openvas_$T.csv として保存

# 全ソースから CVE を抽出し一意化(名寄せ)
grep -rhoE "CVE-[0-9]{4}-[0-9]+" ../scans ../web openvas_*.csv 2>/dev/null \
  | sort -u > cves.txt
wc -l cves.txt
```

ホスト×サービス×所見のマトリクスを所見トラッカ(表計算)に取り込み、複数スキャナが出した同一 CVE は1行に統合する。

---

## Step 2: 検証・誤検知排除(候補→確定)

```bash
# nmap 結果 → ExploitDB 自動相関(実在エクスプロイトの有無で裏取り)
searchsploit --nmap ../scans/tcp_$T.xml

# 個別CVEのエクスプロイト検索
searchsploit $(grep -m1 'CVE-2017-0144' cves.txt)

# 非破壊 PoC(候補ごと、確定の決め手)
msfconsole -q -x "use <module>; set RHOSTS $T; check; exit"

# RHEL/CentOS backport 照合(取得済み rpm 情報 or 対象上で)
rpm -q --changelog openssh-server | grep -m1 "CVE-"
```

各所見に検出根拠(version 依存 / behavior / PoC)と判定(候補/確定)をトラッカへ記録。version 依存のみは確定に昇格させない。

---

## Step 3: エンリッチ(KEV / EPSS / CVSS)

```bash
# CISA KEV カタログ取得
curl -s https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json -o kev.json

# 検出CVEが KEV 該当かを判定
while read c; do
  hit=$(jq -r --arg c "$c" '.vulnerabilities[]|select(.cveID==$c)|.cveID' kev.json)
  echo "$c KEV=$([ -n "$hit" ] && echo YES || echo no)"
done < cves.txt | tee kev_match.txt

# EPSS バルクCSV取得(悪用確率)
curl -s https://epss.cyentia.com/epss_scores-current.csv.gz | gunzip > epss.csv
while read c; do
  printf "%s," "$c"; grep -m1 "^$c," epss.csv | cut -d, -f2 || echo "n/a"
done < cves.txt | tee epss_match.txt
```

CVSS 基本値は OpenVAS/NVD 由来をそのまま採用し、上記で温度感を補正する。

- CISA KEV https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- EPSS https://www.first.org/epss

---

## Step 4: 攻撃経路・チェーン分析

個別の確定所見を「初期侵入 → 権限昇格 → 横展開 → 影響」に連結する。AD除外のため横展開はローカルアカウント再利用が軸。

```bash
# loot 内の取得済み資格情報を棚卸し
grep -rhiE "password|secret|PRIVATE KEY|id_rsa" ../loot 2>/dev/null

# ローカル資格再利用の実証(チェーンの裏付け)
nxc smb ../scope/live.txt -u Administrator -H <NTLM> --local-auth --continue-on-success
```

各チェーンに前提条件(external 到達 / internal 限定)と toxic combination(低リスク単体が連鎖で高インパクト化)を明記。例: `.env` 露出 → 平文資格 → 別ホスト SSH 再利用。

---

## Step 5: リスク評価・優先度付け

```bash
# KEV該当(即時対応候補)
grep "KEV=YES" kev_match.txt

# EPSS 高(0.5以上)を抽出
awk -F, '$2!="n/a" && $2+0>=0.5{print}' epss_match.txt

# KEV と EPSS をCVE単位で結合(トラッカ取り込み用)
join -t, <(sort kev_match.txt | tr ' ' ',') <(sort epss_match.txt) 2>/dev/null
```

合成軸は 深刻度(CVSS)× 悪用容易性(KEV/EPSS/exploit 有無)× 露出(外部/内部)× 資産重要度。下表で優先度を確定する。

| | 外部露出 または KEV該当 | 内部のみ |
|---|---|---|
| 高CVSS + exploit有 | Critical(即時) | High |
| 高CVSS + exploit無 | High | Medium |
| 中・低CVSS | Medium | Low |

KEV 該当は即時、EPSS 高 + 外部露出は最優先へ繰り上げる。

---

## Step 6: 根本原因・テーマ分析

```bash
# レガシープロトコル露出の横断抽出
grep -rhiE "telnet|rlogin|rsh|finger|ftp|microsoft-ds" ../scans/tcp_*.gnmap | sort -u

# 弱TLS/旧プロトコルの集約(Phase4 出力から)
grep -rhiE "SSLv|TLSv1\.0|TLSv1\.1|RC4|3DES|EXPORT" ../web/testssl_* 2>/dev/null
```

確定所見をルート原因で分類(未適用パッチ / 弱・既定資格 / 設定不備 / レガシープロトコル / 暗号設定)。系統別傾向(Solaris=レガシーサービス、RHEL=パッチ運用と backport、Windows=ホストパッチ)を把握し、戦術的修正と運用改善を分離。

---

## Step 7: 対策導出(攻撃↔防御の併記)

確定所見ごとに対策をペア付けする。代表的な対応:

| ルート原因 | 代表所見 | 対策(Quick win / 構造的) |
|---|---|---|
| 未適用パッチ | MS17-010, BlueKeep, PrintNightmare, PwnKit | 該当パッチ適用 / パッチ運用プロセス確立 |
| レガシープロトコル | SMBv1, telnet, rlogin, finger | 無効化・SSH 移行 / 境界遮断 |
| 暗号設定 | 旧TLS, 弱スイート, 期限切れ証明書 | TLS1.2以上+強スイート / 証明書管理 |
| 弱・既定資格 | SSH/RDP/DB/IPMI 既定値 | 変更+複雑化+ロックアウト / 鍵認証・集中管理 |
| サービス設定不備 | SNMP public, NFS no_root_squash, 未認証DB | v3化/squash有効/認証強制 / 最小公開原則 |
| 情報露出 | .git, .env, ディレクトリ列挙 | 公開停止・アクセス制限 / デプロイ手順是正 |

設定変更で済む Quick win と、パッチ運用・最小権限化などの構造的修正を分け、リスク低減効果×工数で実施順を決める。

---

## Step 8: 網羅性・品質保証

```bash
# 対象内ホストのうち未分析を検出
comm -23 <(sort ../scope/live.txt) <(cut -d, -f1 hosts_services.csv | sort -u)
```

「未実施」と「対象外」を区別し、深刻度判定の一貫性、各所見の証跡(コマンド+出力)の完備、再現性を確認してから報告へ渡す。

---

## 出力

確定所見(根拠・KEV/EPSS・優先度・チェーン・対策・証跡)を所見トラッカへ集約し、網羅性照合を経て報告フェーズへ引き渡す。

## Key takeaways
- 候補→確定の二段管理と誤検知排除(特に RHEL backport と version-only 検出の補正)が分析精度を決める。
- KEV(実悪用)/EPSS(確率)/exploit 有無/露出/資産重要度の多軸合成で優先度を確定し、CVSS 単独判定を補正する。
- 個別所見をローカル資格再利用軸でチェーン化し、toxic combination を抽出する。
- KEV/EPSS は `curl`+`jq`/`grep` で機械的に突合でき、結果をトラッカへ取り込んで優先度マトリクスを適用する。
