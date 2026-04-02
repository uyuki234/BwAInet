# 家庭用 WiFi ルーターのイベント利用に関するエビデンス集

> 家庭用 WiFi ルーター（Buffalo 等）が 50〜200 人規模のイベントに適さない根拠を、メーカー公式仕様・業界文献・実測データ・障害事例から整理する。

---

## 1. メーカー公式スペックによる同時接続数の限界

### Buffalo 公式製品仕様

| モデル | 分類 | 推奨同時接続数 | 出典 |
|---|---|---|---|
| WSR-1800AX4P | 家庭用 | **14 台** | [Buffalo 製品ページ](https://www.buffalo.jp/biz/industry/detail/pro-wi-fi.html) |
| WSR-3200AX4S | 家庭用 | **14〜21 台** | [Buffalo 製品ページ](https://www.buffalo.jp/product/detail/wsr-3200ax4s-bk.html), [INTERNET Watch](https://internet.watch.impress.co.jp/docs/column/wifi_qanda/1328501.html) |
| WXR-6000AX12S | 家庭用（フラグシップ） | **36 台 / 12 ユーザー** | [Buffalo 製品ページ](https://www.buffalo.jp/product/detail/wxr-6000ax12s.html) |
| VR-U300W | **業務用** | **40 台** | [Buffalo 製品ページ](https://www.buffalo.jp/biz/industry/detail/pro-wi-fi.html) |

Buffalo 自身が家庭用と業務用を比較し、業務用は家庭用の約 3 倍の同時接続に対応すると明記している。

> 「接続台数が多くなるほど通信が遅延し、業務に支障をきたす恐れがあります」 — Buffalo 公式

> 推奨台数を超えると「**通信速度が低下するなどといったトラブル**」が発生する可能性がある。 — INTERNET Watch

**AOSS 利用時**: セキュリティキー交換は最大 **24 台** に制限される（[Buffalo FAQ](https://www.buffalo.jp/support/faq/detail/15950.html)）。

### イベント規模との比較

| 条件 | 値 |
|---|---|
| 想定来場者数 | 50〜200 人 |
| 想定端末数 | 100〜400 台（1 人あたり PC + スマホ） |
| Buffalo フラグシップの推奨上限 | 36 台 / AP |
| 必要 AP 数（フラグシップの場合） | 最低 3〜12 台 |

Buffalo のフラグシップモデルですら推奨 36 台。100 台規模の端末を収容するには最低 3 台以上が必要だが、**WLC がないため AP 間の負荷分散は制御できない**（後述）。

### エンタープライズ AP の推奨値（業界比較）

**Cisco Meraki — High Density Wi-Fi Deployments**

> **"It is recommended to have around 25 clients per radio or 50 clients per AP in high-density deployments."**

> **"High density is not a coverage problem but a contention problem."**
>
> （高密度は電波の到達範囲の問題ではなく、**エアタイムの奪い合い（コンテンション）の問題**である）

- 出典: [Cisco Meraki - High Density Wi-Fi Deployments](https://documentation.meraki.com/Architectures_and_Best_Practices/Cisco_Meraki_Best_Practice_Design/Best_Practice_Design_-_MR_Wireless/High_Density_Wi-Fi_Deployments)

**これはエアタイムフェアネス・バンドステアリング・WLC を備えたエンタープライズ AP での推奨値**であり、これらの機能を持たない家庭用 AP の実用上限はこれを大幅に下回る。

**横河レンタ・リース**

> 「コンシューマー用 WiFi は数台が一般的。ビジネス用 Aruba AP は同時に数十台でも接続可能で、100 台以上を前提とした設計が可能」

- 出典: [無線LAN コンシューマー用 vs ビジネス用 | 横河レンタ・リース](https://www.yrl.com/column/wlan_c_vs_b.html)

**AccessAgility（米国）**

> 「Consumer-grade APs are designed to support 30-50 devices, while enterprise APs can serve as many devices as found on a college campus.」

- 出典: [Consumer/Home vs Enterprise Access Points | AccessAgility](https://www.accessagility.com/blog/consumer-home-vs-enterprise-access-points)

---

## 2. エアタイムフェアネス（ATF）の不在

### 問題

802.11 は CSMA/CA（半二重・衝突回避）で動作する。低速クライアント（例: 802.11g 6Mbps）は同じデータを送るのに高速クライアント（例: 802.11ax 1200Mbps）の **200 倍のエアタイム** を消費する。エアタイムフェアネスがなければ、**1 台の低速端末が AP 全体のスループットを大幅に低下させる**。

> **"a single client pulling large downloads can consume a disproportionate share of airtime"**

> **"the objective is to avoid situations where the channel is being monopolized by specific clients with slow connection speed"**

- 出典: [Cisco — Air Time Fairness Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/technology/mesh/8-2/b_Air_Time_Fairness_Phase1_and_Phase2_Deployment_Guide.html)

### エンタープライズ専用機能であることの根拠

**Cisco ATF（Air Time Fairness）**

Cisco の ATF は WLC（Wireless LAN Controller）経由で制御され、SSID ごとの重み付け（例: SSID1=70%, SSID2=30%）まで可能。High Density Experience（HDX）機能セットの一部。

- 出典: [Cisco ATF Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/technology/mesh/8-4/b_Air_Time_Fairness_ATF_Deployment_Guide_rel_8_4.html)
- 出典: [Cisco Catalyst 9800 ATF Configuration](https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/config-guide/b_wl_16_10_cg/air-time-fairness.html)

> 「Cisco はすべてのクライアントの通信時間を均等にするだけではなく、SSID ごとに重み付けをすることもできます」 — C&S Engineer Voice

- 出典: [エアタイムフェアネスとトラフィックシェーピング | C&S Engineer Voice](https://licensecounter.jp/engineer-voice/blog/articles/20191225__10.html)

**TP-Link 業務用 EAP シリーズ — 実測データ**

TP-Link の業務用 AP（EAP225-Outdoor, EAP320, EAP330 等）で ATF を有効化した結果：

| 指標 | ATF 無効 | ATF 有効 | 改善率 |
|---|---|---|---|
| 全体スループット | 46 Mbps | **128 Mbps** | **約 2.8 倍** |
| 高速クライアント（802.11n）性能 | — | — | **約 6 倍** |

- 出典: [TP-Link EAP/CAP エアタイムフェアネス機能](https://www.tp-link.com/jp/support/faq/2095/)

**家庭用ルーターにはこの機能が搭載されていない。**

- エンタープライズ AP: エアタイムフェアネスにより各クライアントのエアタイム配分を制御
- 家庭用 AP: **この機能を持たない**。低速端末がいれば全員が遅くなる

---

## 3. WLC 不在による負荷偏り（Sticky Client 問題）

### WLC（無線 LAN コントローラー）がない場合に起きること

| 機能 | WLC あり（エンタープライズ） | WLC なし（家庭用） |
|---|---|---|
| チャネル割り当て | 自動最適化（DCA） | 各 AP が独立して固定。重複すると相互干渉 |
| 送信出力 | 自動調整（TPC） | 最大出力固定。カバレッジが重なり干渉増大 |
| 負荷分散 | AP の負荷を監視し新規接続を空いた AP へ誘導 | **制御手段なし**。特定 AP に集中し他は遊ぶ |
| ローミング | 802.11r/k/v で高速ハンドオフ | **非対応**。端末が遠い AP に掴み続ける |
| スティッキークライアント対策 | 802.11v で移動を促す / 強制切断 | **対策手段なし** |
| バンドステアリング | 5GHz 優先に誘導 | なし。2.4GHz に端末が集中 |

### WLC によるクライアントステアリング

**Cisco 9800 WLC**: 過負荷の AP が 802.11 association response で **status code 17（AP is busy）** を返し、クライアントを別の AP に誘導する。この機能は WLC がなければ動作しない。

- 出典: [Aggressive Client Load Balancing | Cisco 9800](https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/config-guide/b_wl_16_10_cg/client-load-balancing.html)

**Cisco Meraki**: Client balancing は「wireless devices を最も空いている AP に誘導する」機能。過負荷 AP が **association を拒否**し、クライアントを分散させる。**デフォルトでは無効** — つまりコントローラーがあっても明示的に有効化しなければ機能しない。

- 出典: [Client Balancing | Cisco Meraki](https://documentation.meraki.com/MR/Other_Topics/Client_Balancing)

### Sticky Client の影響

端末は最初に接続した AP に留まり続ける傾向がある。移動して電波状況が悪化しても切り替わらず、低速通信で大量のエアタイムを消費する。この 1 台が AP 全体の性能を劣化させる。

> 「WiFi is a shared medium and all devices on the channel are competing for airtime. A sticky client using low data rates and experiencing retransmissions **forces all other devices to wait longer** for transmission opportunities.」

> 「It can thus become a **bad apple — a single client that destroys the performance of the entire wireless network**.」

- 出典: [Sticky Clients | WiFi Professionals](https://www.wifi-professionals.com/2018/10/sticky-clients)
- 出典: [Sticky Clients: When Devices Cling to Poor Wi-Fi | Eye Networks](https://eyenetworks.no/en/sticky-clients/)

WLC があれば 802.11v BSS Transition Management で端末に移動を促すか、最低 RSSI 閾値で強制切断できる。**家庭用 AP にはこの仕組みがない。**

### 802.11k/r/v（ローミング支援規格）

適切なクライアントステアリングには以下の規格への対応が必要であり、いずれもエンタープライズ AP + コントローラーの構成が前提：

| 規格 | 機能 |
|---|---|
| 802.11k | 近隣 AP リストの提供。クライアントがローミング先を判断する材料を与える |
| 802.11r | 高速セキュアハンドオフ。AP 間の切り替え時の認証を高速化する |
| 802.11v | AP からクライアントへの BSS Transition Management。ネットワーク状態に基づき最適な AP を提案する |

---

## 4. 高密度環境のベストプラクティス

### Cisco Meraki 高密度 WiFi 設計ガイド

| 項目 | 推奨値 |
|---|---|
| AP あたりクライアント数 | **25 台/radio、50 台/AP** |
| 高密度の定義 | 1 AP に **30 台以上**が接続する環境 |
| SSID 数 | 最大 **3**（高密度環境では必須要件） |
| SSID 5 以上のオーバーヘッド | エアタイムの **20% 以上**を消費 |
| クライアント帯域制限 | **5 Mbps/client** を推奨 |
| メッシュ | **非推奨** |
| チャネル幅 | **20 MHz**（40/80 MHz は非推奨） |

- 出典: [High Density Wi-Fi Deployments | Cisco Meraki](https://documentation.meraki.com/Architectures_and_Best_Practices/Cisco_Meraki_Best_Practice_Design/Best_Practice_Design_-_MR_Wireless/High_Density_Wi-Fi_Deployments)

### Aruba 高密度設計ガイド

| 項目 | 推奨値 |
|---|---|
| radio あたりクライアント数 | **40〜60 台** |
| 最大アソシエーション数 | 255 台/radio（510 台/AP） |
| 実用推奨値 | **150 台/radio**（最大の約 60%） |

- 出典: [Aruba Very High-Density 802.11ac Networks Planning Guide (PDF)](https://higherlogicdownload.s3.amazonaws.com/HPE/MigratedAttachments/7BC8710B-BC01-4229-A170-41F8F5A5E6F8-5-Aruba_VHD_VRD_Planning_Guide.pdf)

### Cisco 高密度クライアント設計ガイド

高密度環境の定義:

> 「any environment with a high number of concentrated clients (1 client every > 1.5 sq m), such as a conference room, classroom, lecture hall, auditorium, sports arena or conference hall.」

- 出典: [Wireless High Client Density Design Guide | Cisco](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-7/b_wireless_high_client_density_design_guide.html)

### Ruckus スタジアム導入事例

マラカナンスタジアム（76,000 席）: **217 台の Ruckus ZoneFlex AP** を展開。1 AP あたり **同時 100 データユーザー** または **同時 20 音声通話** をサポート。

- 出典: [Ruckus High Density AP Deployment Best Practices](https://support.ruckuswireless.com/documents/1345-high-density-wi-fi-best-practices-ap-deployment-guide)

---

## 5. カンファレンスネットワーク専門チームの知見（CONBU）

CONBU（COnference Network BUilders）は、JANOG・LLNOC 等のネットワークエンジニア有志が、**カンファレンス会場のネットワークを構築するために結成した団体**。YAPC::Asia 等の大規模カンファレンスで NOC を運用している。

### 100 人規模会場のデバイス数

> 100 人の来場者がいる場合、無線デバイス数は **200 台以上** になる。参加者は PC、スマートフォン、タブレット、モバイルルーターを持ち込む。

### カンファレンスネットワークの設計思想

- 家庭用ルーターは**電波出力を最大化**する設計（広い範囲をカバーするため）
- カンファレンスネットワークでは意図的に**送信出力を下げて**各 AP のカバレッジを制限し、クライアントを分散させる
- これは家庭用ルーターの設計思想と**真逆**であり、家庭用機器では実現できない

- 出典: [会場でネットがつながりにくくなる理由 | gihyo.jp](https://gihyo.jp/admin/feature/01/conbu-lan/0001)

### CONBU の機材選定

CONBU は **Cisco vWLC（Virtual Wireless LAN Controller）+ エンタープライズ AP** を使用してカンファレンスネットワークを構築している。

- 出典: [GitHub - conbu/vwlc-manual](https://github.com/conbu/vwlc-manual)

**カンファレンスネットワークの専門家集団が Cisco WLC + エンタープライズ AP を選択していること自体が、家庭用 AP では不十分であることの証明である。**

---

## 6. 実際の障害事例

### カンファレンス WiFi 障害（Joel on Software）

2,000 人のカンファレンスで、会場が家庭用ルーター（LinkSys WRT54g）を使用。DHCP アドレスプールが枯渇し、ネットワークが崩壊。会場スタッフは DHCP サーバの設定について「全く理解していなかった」。

- 出典: [The WiFi at Conferences Problem | Joel on Software](https://www.joelonsoftware.com/2009/10/08/the-wifi-at-conferences-problem/)

### イベント WiFi 障害の実態（Click Telecom）

> 「Domestic routers and WiFi extenders aren't built to withstand the demands of public events.」（家庭用ルーターと WiFi エクステンダーはイベントの需要に耐えるように作られていない）

障害モード: 同時接続過多による帯域逼迫、カバレッジ不足、電波干渉、オンサイトサポートの不在、不適切な機器選定。

- 出典: [The Real Reason Your Event WiFi Keeps Failing | Click Telecom](https://click-telecom.co.uk/guides/the-real-reason-your-event-wifi-keeps-failing-and-how-to-fix/)

### 日本国内の展示会における WiFi 干渉障害

各出展者がポケット WiFi を個別に使用した結果、電波が互いに干渉し通信が劣化。**決済システム障害・ライブ配信中断・モバイルオーダー停止**が発生。

> 「各出展者がそれぞれポケット WiFi を利用すると、電波が互いに干渉し合い、通信が劣化してどの WiFi にも繋がりにくくなる」

- 出典: [展示会での WiFi 干渉問題を解決する方法 | Agility Event](https://agility-event.com/column/exhibition-wi-fi-interference-measure/)
- 出典: [展示会などのイベントで WiFi がつながりにくくなる理由 | WiFi レンタルどっとこむ](https://www.wifi-rental.com/mmedia/service/10052)

---

## 7. 会場固有の問題との複合リスク

[パケットキャプチャ解析レポート](iput-pcapng-analysis-report.md) で確認された会場固有の問題が、上記の一般的なリスクに加わる。

### 実測データ（2026-03-24 下見時キャプチャ）

| 項目 | 値 |
|---|---|
| キャプチャ時間 | 約 475 秒間 |
| 端末接続数 | ほぼゼロ（下見時） |
| ブロードキャストフレーム割合 | **全パケットの 38.3%** |
| ブロードキャスト発生源 | Allied Telesis 製スイッチ 21 台（RRCP, EtherType `0x8899`） |
| ブロードキャストレート | 約 10.5 pps（630 フレーム/分） |

### Buffalo（ブリッジモード）の場合の影響

| 会場固有の問題 | 家庭用ルーターでの影響 |
|---|---|
| ブロードキャストが全パケットの 38.3%（スイッチ 21 台が常時送出） | L2 ブリッジ動作のため WiFi にそのまま流れ、エアタイムを消費 |
| EtherType `0x8899`（Realtek L2） | 家庭用 ACL では弾けないプロトコル |
| /22 単一セグメントにスイッチ 29 台が同居 | ブロードキャストドメインの分割手段なし |

- RRCP は EtherType `0x8899`（L2）のため、**Buffalo の ACL では遮断不可能**
- 端末ゼロの状態で既に帯域の約 4 割がブロードキャストに占有されている
- 当日端末が接続されると ARP / mDNS / NBNS / SSDP 等が加算される

### VyOS 構成（VLAN）の場合

自前 VLAN セグメントを構築するため、会場側スイッチのブロードキャストドメインから**分離される**。RRCP ブロードキャストが WiFi 帯域を消費することはない。

---

## 8. 前回イベント実績との比較

| 項目 | 前回実績 |
|---|---|
| 平均スループット | 117 Mbps |
| 8 時間のトラフィック量 | 421 GB |
| 使用機材 | エンタープライズ構成 |

この規模のトラフィックを家庭用 AP で捌いた実績はない。

---

## 9. 総括

| 観点 | 家庭用ルーター | エンタープライズ AP + WLC |
|---|---|---|
| 推奨同時接続数 | 14〜36 台（メーカー公称） | 25〜150 台/radio |
| エアタイムフェアネス | **非搭載** | 搭載（ATF 有効化でスループット約 3 倍） |
| クライアントステアリング | **不可**（Sticky Client 問題が発生） | WLC による負荷分散・802.11k/r/v |
| 送信出力制御 | 最大固定 | AP ごとに自動調整（TPC） |
| チャネル設計 | 自動（制御不能） | 自動最適化（DCA）/ 手動 |
| バンドステアリング | なし（2.4GHz に集中） | 5GHz 優先に誘導 |
| ブロードキャスト制御 | **不可**（L2 ブリッジ） | VLAN 分離で隔離可能 |
| 障害時の切り分け | 困難（管理機能なし） | ログ・統計・リアルタイム監視 |

**家庭用 WiFi ルーターの 50〜200 人規模イベントへの使用は、メーカー仕様の範囲外であり、業界のベストプラクティスに反し、実際に障害事例が多数報告されている。** これは個人の意見ではなく、Cisco・Aruba・Buffalo・CONBU 等の公式ドキュメントおよび実測データに基づく結論である。

---

*本ドキュメントの出典 URL は 2026-04-02 時点で確認済み。*
