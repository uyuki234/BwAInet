# Cisco Aironet 3700/3800 + WLC 3504 ハードウェア詳細調査レポート

> 調査日: 2026-04-02
> 対象機器: Cisco Aironet 3700 × 10, Aironet 3800 × 10, WLC 3504 × 1

## 調査の経緯

BwAI in Kwansai 2026 フル構成（[PRD.md](../requirements/PRD.md)）において、投入可能な AP 機材が Aironet 3700 × 10 台、Aironet 3800 × 10 台、WLC に Cisco 3504 であることが確定した。既存 PRD では「Aironet 3800 約 20 台」と記載されていたが、実際には 3700/3800 の混成構成となる。本レポートでは各機器の詳細仕様と混在運用時の注意事項をまとめる。

---

## Cisco Aironet 3700 シリーズ

### 基本仕様

| 項目 | 仕様 |
|---|---|
| **CPU** | **Freescale P1020** (Power Architecture e500v2, デュアルコア, 800 MHz) |
| **DRAM** | **512 MB** |
| **Flash** | 64 MB |
| 5 GHz WiFi チップ | Marvell 88W8864 |
| 2.4 GHz WiFi チップ | Marvell 88W8764 |
| WiFi 規格 | **802.11ac Wave 1** (802.11a/b/g/n/ac) |
| 5 GHz MIMO | 4x4:3SS |
| 5 GHz 最大速度 | **1.3 Gbps** (80 MHz) |
| 2.4 GHz MIMO | 4x4:3SS |
| 2.4 GHz 最大速度 | **450 Mbps** (40 MHz, 802.11n) |
| MU-MIMO | **非対応**（Wave 1 のため） |
| 有線ポート | 1 × GbE (1000BASE-T) — **mGig 非対応** |
| PoE 要件 | 802.3at (PoE+) 推奨、802.3af でも動作可（一部機能制限あり） |
| 消費電力 | 約 15.4 W |

### 同時接続台数

| 指標 | 値 |
|---|---|
| 最大（ハードウェア上限） | 200 クライアント/ラジオ（合計 400） |
| 推奨（実運用設計値） | **20〜30 クライアント/ラジオ** |

### 主要機能

- **CleanAir**: 対応 — 干渉源の検知・分類・回避
- **ClientLink 3.0**: 送信ビームフォーミング（802.11a/g/n/ac クライアント対応）
- **FlexConnect**: 対応
- **802.11r/k/v**: 対応（高速ローミング）
- **エアタイムフェアネス**: WLC 経由で対応

### EoL / EoS 状況

| マイルストーン | 日付 |
|---|---|
| 販売終了 (End-of-Sale) | 2019-04-30 |
| SW メンテナンス終了 | 2020-04-29 |
| セキュリティ脆弱性修正終了 | 2022-04-29 |
| **最終サポート日** | **2024-04-30** |

> **2026 年 4 月時点で完全にサポート終了済み。** セキュリティパッチ・TAC サポートともに提供なし。

### 対応 WLC ソフトウェア

- AireOS 8.10.x（最終推奨: 8.10.196.0）

---

## Cisco Aironet 3800 シリーズ

### 基本仕様

| 項目 | 仕様 |
|---|---|
| **CPU** | **Marvell 88F6920 (Armada 390)** (ARMv7, デュアルコア, 1.8 GHz) |
| **DRAM** | **1,024 MB (1 GB)** |
| **Flash** | 256 MB |
| 5 GHz WiFi チップ | Marvell 88W8964 |
| 2.4 GHz WiFi チップ | Marvell 88W8764 |
| WiFi 規格 | **802.11ac Wave 2** (802.11a/b/g/n/ac Wave2) |
| 5 GHz MIMO | 4x4:3SS **MU-MIMO** |
| 5 GHz 最大速度 | **2.6 Gbps** (160 MHz) |
| 2.4 GHz MIMO | 3x3:3SS |
| 2.4 GHz 最大速度 | **450 Mbps** (40 MHz, 802.11n) |
| MU-MIMO | **対応**（ダウンリンク、複数クライアント同時送信） |
| 有線ポート | 1 × **mGig** (100M/1G/2.5G/5G) — Cat5e で 2.5G/5G 対応 |
| PoE 要件 | **802.3at (PoE+) 必須**（802.3af では動作不可） |
| 消費電力 | 22.5 W（USB 除く）/ 25.5 W（USB 有効時） |

### 同時接続台数

| 指標 | 値 |
|---|---|
| 最大（ハードウェア上限） | 200 クライアント/ラジオ（合計 400） |
| 推奨（高密度環境） | **30〜50 クライアント/ラジオ** |

### 主要機能

- **MU-MIMO**: 対応 — 複数クライアントへの同時ダウンリンク送信
- **CleanAir Express**: 対応 — 干渉源の検知・分類・軽減
- **ClientLink 4.0**: 送信ビームフォーミング
- **FlexConnect**: 対応
- **802.11r/k/v**: 対応（高速ローミング）
- **Flexible Radio Assignment (FRA)**: RF 環境に応じたラジオ役割の自動判定
- **Dual 5 GHz モード**: 両ラジオを 5 GHz に設定可（最大 5.2 Gbps）
- **エアタイムフェアネス**: WLC 経由で対応

### EoL / EoS 状況

| マイルストーン | 日付 |
|---|---|
| 販売終了 (End-of-Sale) | 2022-10-31 |
| SW メンテナンス終了 | 2023-10-31 |
| セキュリティ脆弱性修正終了 | 2025-10-31 |
| **最終サポート日** | **2027-10-31** |

> **2026 年 4 月時点で TAC サポートは継続中。** ただし新規ソフトウェアリリースは終了。

### 対応 WLC ソフトウェア

- AireOS 8.10.x（最終推奨: 8.10.196.0）

---

## Cisco WLC 3504 (AIR-CT3504-K9)

### 基本仕様

| 項目 | 仕様 |
|---|---|
| 最大管理 AP 数 | **150 台** |
| 最大クライアント数 | **3,000** |
| スループット | 4 Gbps |
| CPU | Cavium CN7240-AAP (8 コア, 1.5 GHz) |
| RAM | 8 GB DDR4 |
| Boot Flash | 8 MB SPI NOR |
| Bulk Flash | 32 GB eMMC |
| フォームファクタ | 1RU ラックマウント可（ファンレス ≤30 ℃） |
| 寸法 | 43.94 × 214.3 × 215.9 mm |
| 重量 | 約 2.0 kg |
| 動作温度 | 0〜40 ℃ |

### ポート構成

```
[前面パネル]
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│  mGig    │  GbE 1   │  GbE 2   │  GbE 3   │  GbE 4   │ Service  │ Redund.  │
│  (5G)    │  (1G)    │  (1G)    │ (1G/PoE) │ (1G/PoE) │  Port    │  Port    │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
  Data       Data       Data      Data/PoE   Data/PoE    OOB Mgmt   HA Sync
```

| ポート | 数量 | 仕様 | 役割 |
|---|---|---|---|
| Multigigabit Ethernet | 1 | 1G/2.5G/5G (RJ-45) | データ（AP/CAPWAP） |
| Gigabit Ethernet | 4 | 1 GbE (RJ-45)、うち 2 ポートは 802.3at PoE PSE | データ（AP/CAPWAP） |
| Service Port | 1 | 1 GbE (RJ-45) | OOB 管理専用（VLAN タグ非対応） |
| Redundancy Port | 1 | 1 GbE (RJ-45) | HA SSO 同期 |
| Console | 2 | RJ-45 + Mini-B USB | コンソール |

### PoE 給電能力

WLC 3504 自体が PoE PSE として AP に直接給電可能（最大 2 台）。

| 構成 | 消費電力 |
|---|---|
| PoE なし | 47 W |
| PoE 2 ポート使用時 | 98 W |

**制限**: PoE 有効時、ポート 1・2 の LAG が使用不可。

### ライセンス

RTU (Right-to-Use) ライセンス。20 AP 分のライセンス確保済み。

### RF 管理機能

| 機能 | サポート |
|---|---|
| RRM (Radio Resource Management) | 対応 — 自動チャネル割当 (DCA)、送信電力制御 (TPC)、カバレッジホール検出 |
| CleanAir | 対応 — スペクトラム干渉の検出・報告・緩和 |
| バンドセレクト | 対応 — デュアルバンドクライアントを 5 GHz に誘導 |
| エアタイムフェアネス (ATF) | 対応 — HDX (High Density Experience) の一部 |

### 高可用性 (HA)

| 項目 | 詳細 |
|---|---|
| AP SSO | 対応（CAPWAP 状態同期） |
| Client SSO | 対応（クライアントセッション同期） |
| フェイルオーバー時間 | サブセコンド（1 秒未満） |
| 接続 | Redundancy Port 経由で直結または L2 スイッチ経由 |

### EoL / EoS 状況

| マイルストーン | 日付 |
|---|---|
| 販売終了 (End-of-Sale) | 2021-01-31 |
| SW メンテナンス終了 | 2023-01-31 |
| セキュリティ脆弱性修正終了 | **2025-01-30** |
| **最終サポート日** | **2027-01-31** |

> **2026 年 4 月時点で TAC サポートは継続中だが、セキュリティパッチは既に提供終了。**

### AireOS ソフトウェア

| 項目 | バージョン |
|---|---|
| 初回対応 | AireOS 8.5.103.0 |
| TAC 推奨（最新） | **AireOS 8.10.196.0** |
| AireOS 8.10 SW メンテナンス終了 | 2023-05-01 |
| AireOS 8.10 セキュリティ修正終了 | 2025-01 |

---

## 3700 + 3800 混在管理

### 互換性

- WLC 3504 は Aironet 3700 と 3800 の**同時管理が可能**
- 推奨 AireOS バージョン: **8.10.196.0**（三者が互換となる唯一の推奨トレイン）
- 機能差異はあるが、WLAN ポリシー・QoS 設定は共通で適用可能

### 3700 vs 3800 機能差異

| 機能 | Aironet 3700 | Aironet 3800 |
|---|---|---|
| **CPU** | Freescale P1020 (PPC 2C 800MHz) | **Marvell 88F6920 (ARM 2C 1.8GHz)** |
| **DRAM** | 512 MB | **1,024 MB (2 倍)** |
| **Flash** | 64 MB | **256 MB (4 倍)** |
| WiFi 規格 | Wave 1 | **Wave 2** |
| 5 GHz WiFi チップ | Marvell 88W8864 | **Marvell 88W8964** |
| MU-MIMO | 非対応 | **対応** |
| mGig (有線) | 非対応 (1 GbE) | **対応** (最大 5 GbE) |
| 5 GHz 最大速度 | 1.3 Gbps | **2.6 Gbps** |
| 推奨クライアント数/ラジオ | 20〜30 | **30〜50** |
| CleanAir | 対応 | 対応 (Express) |
| ClientLink | 3.0 | **4.0** |
| FRA (Flexible Radio) | 非対応 | **対応** |
| Dual 5 GHz | 非対応 | **対応** |
| EoL | **2024-04 終了** | 2027-10 まで |
| PoE 要件 | 802.3at 推奨 (af 可) | **802.3at 必須** |

### 混在運用時の注意事項

1. **AP イメージ形式**: AireOS 8.3 以前から 8.5 以降にアップグレード時、3700 は AP イメージ形式が `ap3g2` → `c3700` に変更されるため、2 回ダウンロード + 2 回再起動が発生
2. **FlexConnect**: 3700 と 3800 を同一 FlexConnect グループに混在配置可能。高速ローミング (802.11r) は同一グループ内の AP 間でのみ動作
3. **クライアント密度の考慮**: 3800 は MU-MIMO 対応のため高密度環境に強い。来場者が集中するエリア（大教室）には 3800 を優先配置し、比較的人数が少ないエリアに 3700 を配置するのが合理的

---

## キャパシティ分析

### 来場者数と想定クライアント台数

| 項目 | 値 | 根拠 |
|---|---|---|
| 来場者目標 | **200 名以上** | 運営側の目標値 |
| 前回実績 | 来場約 100 名 / 接続約 100 台 | 1 人あたり約 1 台 |
| 想定クライアント台数（実績ベース） | **200 台** | 200 名 × 1 台/人 |
| 想定クライアント台数（設計想定） | **400〜600 台** | 200 名 × 2〜3 台/人（前回は /22 をこの想定で確保） |

### 構成別キャパシティ比較

| 指標 | Buffalo × 8 (最小構成) | Cisco × 20 + WLC (フル構成) |
|---|---|---|
| AP 台数 | 8 | 20 (3700×10 + 3800×10) |
| 安定運用域 (全体) | 80〜120 台 | **400〜1,000 台** |
| 200 台時の **1 台あたり平均** | **25 台/AP** | **10 台/AP** |
| 200 台時の **偏り考慮** | **30〜37 台/AP** (WLC なし、分散制御不可) | **10〜15 台/AP** (WLC が負荷分散) |
| 来場者 200 名 × 1 台 = 200 台 | **破綻** | **余裕あり** |
| 来場者 200 名 × 2〜3 台 = 400〜600 台 | **完全に範囲外** | **安定域〜やや高負荷** |
| WLC クライアント管理上限 | — | **3,000 台** |

**最小構成は均等分散ですら 1 台 25 台で安定域（10〜15 台）を大幅超過する。** さらに WLC がないため負荷分散が制御できず、スティッキークライアント問題により特定 AP に 30〜37 台が集中する。エアタイムフェアネスのない家庭用ルーターで 30 台超は通信タイムアウトが頻発し、事実上使い物にならない。

フル構成であれば 200 台は 1 台あたり平均 10 台に過ぎず、WLC の RRM・負荷分散により AP 間の偏りも自動的に是正される。設計想定の 400〜600 台でも WLC の管理上限 3,000 台に対して十分な余裕がある。

### 実トラフィック下の RTT シミュレーション

トラフィックが流れる状態ではクライアント数に応じて WiFi 区間のコンテンション遅延が増大する。到達先への RTT = WiFi RTT + Internet RTT で計算される。

**Internet RTT 想定**: 自宅経由 VPN で 50 ms、会場回線直接で 100 ms 程度

**来場者 200 名（200 台）での比較:**

| | Buffalo × 8 (最小構成) | Cisco × 20 + WLC (フル構成) |
|---|---|---|
| 台数/AP | 25 台（均等）/ 30〜37 台（偏り） | 10 台（WLC 分散） |
| WiFi RTT | 35 ms（均等）/ 100〜200 ms（偏り） | **2〜3 ms** |
| +Internet 50ms | **85 ms〜250 ms+** | **52〜53 ms** |
| +Internet 100ms | **135 ms〜300 ms+** | **102〜103 ms** |
| 体感 | 均等でも許容ギリギリ、偏りで破綻 | **いずれも快適** |

Buffalo では WiFi 区間だけで 35〜200 ms の遅延が発生し、Internet RTT に上乗せされる。Cisco + WLC では WiFi 区間の遅延が 2〜3 ms に抑制され、Internet RTT がそのまま体感 RTT になる。

**RTT と体感の閾値:**

| Total RTT | 体感 |
|---|---|
| 50〜80 ms | 快適（Web, SSH, ビデオ通話すべて可） |
| 80〜150 ms | やや遅い（ビデオ通話品質低下） |
| 150〜250 ms | 明確に遅い（ビデオ通話困難） |
| 250 ms+ | 使い物にならない（タイムアウト頻発） |

詳細な WiFi 遅延モデルは [WSR-2533DHP3 調査レポート](wsr-2533dhp3-hardware-investigation.md) の「実トラフィック下の現実的な性能限界」セクションを参照。

---

## 配置戦略の提案

### AP 20 台の配分方針

3800 は MU-MIMO・mGig 対応で高密度環境に適しており、3700 は EoL 済みだが基本的な WiFi 5 機能は十分に使える。

| 配置エリア | 機種 | 台数目安 | 理由 |
|---|---|---|---|
| メイン会場（大教室） | **Aironet 3800** | 4〜6 台 | 来場者が最も集中。MU-MIMO の恩恵が最大 |
| サブ会場 | **Aironet 3800** | 2〜4 台 | 中程度の密度 |
| 廊下・共有スペース | **Aironet 3700** | 4〜6 台 | ローミング中継、低密度 |
| 運営控室・配信室 | **Aironet 3700** | 2〜4 台 | スタッフのみ、低密度 |
| 予備 | 3700 / 3800 混合 | 2〜4 台 | 障害時の交換用 |

### PoE 給電

PoE スイッチおよび PoE インジェクター確保済み。3800 は **802.3at (PoE+) 必須**（802.3af では動作不可）。

### WLC 3504 の接続

WLC 3504 は AP と同一 L2 ドメインに存在する必要がある（CAPWAP ディスカバリのため）。

```
[sw01 (FS)]
  5Ge 0/2 ── trunk ── [WLC 3504] (VLAN 11,30,40; native 11)
  Ge 0/3-7 ── access VLAN 11 ── [AP × 5]
  
[追加 PoE スイッチ] ── trunk (VLAN 11) ── [sw01]
  PoE ポート × 15 ── access VLAN 11 ── [AP × 15]
```

AP は VLAN 11 (mgmt) に access ポートで接続し、CAPWAP トンネルで WLC に到達する。WLC がクライアントトラフィックを VLAN 30/40 にマッピングする。

---

## EoL に関するリスク評価

### リスクサマリー

| 機器 | EoL 状況 | リスク |
|---|---|---|
| Aironet 3700 | **完全終了** (2024-04) | セキュリティパッチなし。既知脆弱性が修正されない |
| Aironet 3800 | サポート中 (〜2027-10) | SW メンテナンスは終了。セキュリティ修正のみ (〜2025-10) |
| WLC 3504 | サポート中 (〜2027-01) | セキュリティ修正は **2025-01 に終了済み** |
| AireOS 8.10 | SW メンテナンス終了 (2023-05) | 新機能追加なし。セキュリティ修正も 2025-01 に終了 |

### 軽減策

- 1 日限りのイベント利用であるため、長期運用のリスクとは性質が異なる
- VLAN 分離により AP/WLC は管理 VLAN (11) に隔離されており、来場者 VLAN (40) からの直接アクセスは ACL で制限
- WLC の管理画面アクセスを VLAN 11 からのみに制限することで攻撃面を最小化
- **イベント前に AireOS 8.10.196.0 を適用し、利用可能な最新パッチ状態にしておく**

---

## WLC 性能比較: Cisco 3504 vs Buffalo（コントローラなし）

最小構成の Buffalo WSR-2533DHP3 には WLC が存在しない。各 AP が独立して動作するため、AP 間の協調制御は一切行われない。フル構成の WLC 3504 との差は、AP 単体のスペック差以上に決定的である。

### ハードウェア性能比較

| 項目 | WSR-2533DHP3 (AP 単体) | WLC 3504 | **倍率** |
|---|---|---|---|
| **CPU** | MT7622B ARM A53 2C **1.35 GHz** | Cavium CN7240-AAP **8C 1.5 GHz** | コア数 **4 倍**、総演算能力 **約 4.4 倍** |
| **DRAM** | **256 MB** DDR3 | **8 GB (8,192 MB)** DDR4 | **32 倍** |
| **Flash** | 128 MB SLC NAND | **32 GB** eMMC | **256 倍** |
| **ネットワーク I/O** | 1 GbE × 5 (SoC + RTL8367S) | mGig (5G) × 1 + GbE × 4 + Service + Redundancy | **mGig 対応 + 多ポート** |

Buffalo の AP 1 台分の DRAM (256 MB) に対して、WLC 3504 は **8 GB (32 倍)** を搭載しており、3,000 クライアントの状態管理、CAPWAP トンネル維持、RRM 計算、CleanAir 解析などを同時に処理できる。

CPU は 8 コア × 1.5 GHz (Cavium OCTEON) で、ネットワーク処理に特化した MIPS64 アーキテクチャ。Buffalo の 2 コア ARM とは設計思想が根本的に異なる。

### 管理能力の比較

| 項目 | WSR-2533DHP3 × 8 (最小構成) | WLC 3504 + AP × 20 (フル構成) |
|---|---|---|
| **AP 管理** | 各 AP 独立。統一管理不可 | **最大 150 AP を集中管理** |
| **クライアント状態管理** | AP 内部のみ（AP 間で状態共有なし） | **最大 3,000 クライアントを一元管理** |
| **スループット** | AP 内蔵 HNAT 3.5 Gbps（ブリッジモードでは無効） | **4 Gbps（制御プレーン）** |
| **SSID 数** | AP ごとに個別設定（統一運用に手動設定が必要） | **最大 4,096 WLAN、AP あたり 16 SSID** |
| **VLAN マッピング** | 不可（L2 ブリッジ固定） | **SSID → VLAN 自動マッピング** |

### RF 管理能力の比較

| 機能 | WSR-2533DHP3 (コントローラなし) | WLC 3504 |
|---|---|---|
| **RRM (自動チャネル割当)** | なし。手動設定のみ。8 台の AP が同一チャネルで干渉する可能性 | **DCA (Dynamic Channel Assignment)** で全 AP のチャネルを自動最適化 |
| **送信電力制御 (TPC)** | なし。全 AP が最大出力で送信し相互干渉 | **TPC** で AP ごとの出力を自動調整。カバレッジホールを検出して補正 |
| **CleanAir** | なし | **スペクトラム干渉の検出・分類・回避を自動実行** |
| **バンドセレクト** | なし。クライアント任せ（多くが 2.4 GHz に滞留） | **デュアルバンドクライアントを 5 GHz に自動誘導** |
| **エアタイムフェアネス (ATF)** | なし。低速クライアントがエアタイムを占有 | **SSID/デバイスグループ別にエアタイム配分を制御** |
| **負荷分散** | なし。スティッキークライアント問題が発生 | **過負荷 AP が status code 17 (AP busy) を返し他 AP に誘導** |
| **ローミング制御** | 802.11r/k/v 非対応 | **802.11r/k/v によるシームレスローミング** |
| **カバレッジホール検出** | なし | **RRM がクライアント RSSI を監視し自動検出** |

### 運用管理能力の比較

| 機能 | WSR-2533DHP3 (コントローラなし) | WLC 3504 |
|---|---|---|
| **設定変更** | AP 1 台ずつ Web GUI で個別設定 | **一括設定配信（SSID/セキュリティ/QoS を全 AP に即時反映）** |
| **ファームウェア管理** | 1 台ずつ手動更新 | **AP イメージの自動配信・事前ダウンロード** |
| **障害検出** | なし（管理画面に手動アクセスして確認） | **SNMP Trap, Syslog, CAPWAP keepalive による自動検出** |
| **障害時の自動復旧** | なし（電源抜き差しで手動復旧） | **AP 自動再起動、カバレッジホール自動補正** |
| **監視** | なし | **クライアント統計、トラフィック分析、干渉分析をリアルタイム提供** |
| **セキュリティポリシー** | WPA2-PSK のみ | **WPA2-PSK + WLAN ACL + クライアント除外 + Rogue AP 検出** |
| **高可用性** | なし（AP 故障 = そのエリア全断） | **AP SSO + Client SSO（サブセコンドフェイルオーバー）** |

### 要約

WLC 3504 は Buffalo AP 1 台に対して DRAM **32 倍**、CPU コア数 **4 倍**を備え、20 台の AP を統一的に制御する頭脳として機能する。最小構成では 8 台の Buffalo AP がそれぞれ独立した 256 MB の世界で動作し、AP 間の協調は一切ない。フル構成では 8 GB の DRAM と 8 コア CPU を持つ WLC が全 AP の RF 環境をリアルタイムに最適化し、3,000 クライアントまでの状態管理を一元的に処理する。ハードウェアの数字以上に、**「管理する知性の有無」** が最大の差である。

---

## WSR-2533DHP3 との比較（AP 単体）

最小構成（[WSR-2533DHP3 調査レポート](wsr-2533dhp3-hardware-investigation.md)）と比較した場合のフル構成の優位性:

| 項目 | WSR-2533DHP3 × 8 (最小構成) | Aironet 3700 × 10 (フル構成) | Aironet 3800 × 10 (フル構成) |
|---|---|---|---|
| **CPU** | MT7622B (ARM Cortex-A53 2C 1.35GHz) | Freescale P1020 (PPC e500v2 2C 800MHz) | **Marvell 88F6920 (ARMv7 2C 1.8GHz)** |
| **DRAM** | 256 MB | **512 MB (2 倍)** | **1,024 MB (4 倍)** |
| **Flash** | 128 MB | 64 MB | 256 MB |
| **WiFi チップ** | SoC 内蔵 (2.4G) + MT7615N (5G) | Marvell 88W8864 (5G) + 88W8764 (2.4G) | Marvell 88W8964 (5G) + 88W8764 (2.4G) |
| WiFi 規格 | WiFi 5 Wave 1 | WiFi 5 Wave 1 | WiFi 5 **Wave 2** |
| MU-MIMO | 非対応 | 非対応 | **対応** |
| 推奨同時接続/台 | 10〜15 台 | **20〜30 台/ラジオ** | **30〜50 台/ラジオ** |
| 全体推奨同時接続 | 80〜120 台 | **400〜600 台** | **600〜1,000 台** |
| WLC | なし | **Cisco 3504** | **Cisco 3504** |
| エアタイムフェアネス | なし | **WLC 経由で対応** | **WLC 経由で対応** |
| ローミング (802.11r/k/v) | 非対応 | **対応** | **対応** |
| バンドステアリング | 非対応 | **WLC 経由で対応** | **WLC 経由で対応** |
| VLAN マッピング | 不可（L2 ブリッジ） | **WLC で SSID→VLAN** | **WLC で SSID→VLAN** |
| QoS | なし | **SSID 別帯域制御** | **SSID 別帯域制御** |
| 監視 | なし（管理画面のみ） | **SNMP / CleanAir / RRM** | **SNMP / CleanAir / RRM** |
| スティッキークライアント対策 | なし | **WLC が status 17 で誘導** | **WLC が status 17 で誘導** |
| プロキシ回避 | 不可 | **VyOS + SoftEther VPN** | **VyOS + SoftEther VPN** |

---

## 参考資料

- [Cisco Aironet 3700 Data Sheet](https://www.cisco.com/c/dam/global/en_ph/solutions/smb/velocity/Downloads/ap3700_datasheet.pdf)
- [Cisco Aironet 3700 Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/technology/apdeploy/8-0/Cisco_Aironet_3700AP.html)
- [Cisco Aironet 3700 EoL Notice](https://www.cisco.com/c/en/us/products/collateral/wireless/aironet-3700-series/eos-eol-notice-c51-740710.html)
- [Cisco Aironet 3800 Data Sheet](https://www.cisco.com/c/en/us/products/collateral/wireless/aironet-3800-series-access-points/datasheet-c78-741682.html)
- [Cisco Aironet 3800 EoL Notice](https://www.cisco.com/c/en/us/products/collateral/wireless/aironet-3800-series-access-points/aironet-3800-series-access-points-eol.html)
- [Cisco 2800/3800 Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-3/b_cisco_aironet_series_2800_3800_access_point_deployment_guide.pdf)
- [Cisco WLC 3504 Data Sheet](https://www.cisco.com/c/en/us/products/collateral/wireless/3504-wireless-controller/datasheet-c78-738484.html)
- [Cisco WLC 3504 Installation Guide](https://www.cisco.com/c/en/us/td/docs/wireless/controller/3500/3504/install-guide/b-wlc-ig-3504/overview.html)
- [Cisco WLC 3504 EoL Notice](https://www.cisco.com/c/en/us/products/collateral/wireless/3504-wireless-controller/eos-eol-notice-c51-744737.html)
- [AireOS 8.10 EoL Notice](https://www.cisco.com/c/en/us/products/collateral/wireless/8500-series-wireless-controllers/wireless-software-8-10-pb.html)
- [TAC Recommended AireOS Releases](https://www.cisco.com/c/en/us/support/docs/wireless/wireless-lan-controller-software/200046-tac-recommended-aireos.html)
- [RTU Licensing FAQ for 3504/5520/8540](https://www.cisco.com/c/en/us/support/docs/wireless/3504-wireless-controller/214106-licensing-on-3504-5520-and-8540-wireles.html)
- [FlexConnect Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-8/FlexConnect_DG.html)
- [HA SSO Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-7/High_Availability_DG.html)
- [Cisco Wireless Compatibility Matrix](https://www.cisco.com/c/en/us/td/docs/wireless/compatibility/matrix/compatibility-matrix.html)
