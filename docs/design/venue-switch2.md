# 会場スイッチ sw02 (Cisco ISR 1100) 実装例

> **前提**: 本ドキュメントは [`venue-switch.md`](./venue-switch.md) で定義したマルチベンダー共通設計 (VLAN モデル、ポート種別 T1〜T5、管理 IP ルール、STP 方針) の **Cisco ISR 1100 (C1111-8P) による実装例** である。共通設計を先に読むこと。

## 概要

構築に使えるスイッチが不足しているため、手元の **Cisco ISR 1100 (C1111-8P)** を L2 スイッチとして流用する。
純粋な L2 スイッチではなくルーターのため、WAN 側ポート (`GigabitEthernet0/0/0`, `0/0/1`) は通常の `switchport` コマンドが使えず、**EVC (service instance) + bridge-domain** 構文で L2 ブリッジ化する必要がある。LAN 側ポート (`GigabitEthernet0/1/0`〜`0/1/7`) は Embedded Switch Module (ESM) のため標準の `switchport` 構文で動作する。

| 項目 | 値 |
|------|-----|
| ホスト名 | sw02 |
| 機種 | Cisco ISR 1100 (C1111-8P) |
| OS | IOS XE 17.x |
| 管理 VLAN | 11 |
| 管理 IP | 192.168.11.6/24 |
| デフォルト GW | 192.168.11.1 (r3-vyos) |
| 位置付け | 現場判断で sw01 下位 / 並列どちらでも投入可能 (論理設計は両対応) |

## 物理ポート構成 (C1111-8P)

| ポート | 種別 | PoE | 速度 | 備考 |
|--------|------|-----|------|------|
| Gi 0/0/0 | WAN (routed) | — | 1G | EVC で L2 トランク化 |
| Gi 0/0/1 | WAN (routed) | — | 1G | EVC で L2 トランク化 |
| Gi 0/1/0 | LAN (ESM) | PoE+ | 1G | AP 直給電 |
| Gi 0/1/1 | LAN (ESM) | PoE+ | 1G | AP 直給電 |
| Gi 0/1/2 | LAN (ESM) | — | 1G | PoE+ インジェクター経由 |
| Gi 0/1/3 | LAN (ESM) | — | 1G | PoE+ インジェクター経由 |
| Gi 0/1/4 | LAN (ESM) | — | 1G | PoE+ インジェクター経由 |
| Gi 0/1/5 | LAN (ESM) | — | 1G | PoE+ インジェクター経由 |
| Gi 0/1/6 | LAN (ESM) | — | 1G | access/trunk 柔軟 (配信 PC or スイッチ連結) |
| Gi 0/1/7 | LAN (ESM) | — | 1G | access/trunk 柔軟 (配信 PC or スイッチ連結) |

> **PoE 電源設計**: C1111-8P の PoE 総バジェットは約 50W。Aironet 3800 は IEEE 802.3at (PoE+, 25.5W) のため、本体から直接給電できるのは `Gi 0/1/0-1` の 2 ポートのみ。それ以上の AP/PoE 機器は外部 PoE+ インジェクターを `Gi 0/1/2-5` に挿入して接続する。

## VLAN 定義

共通設計 ([`venue-switch.md`](./venue-switch.md) §1) と同じ。

| VLAN ID | 名称 | 用途 |
|---------|------|------|
| 11 | mgmt | NW 機器管理、AP 管理 IP (FlexConnect) |
| 30 | staff | 運営スタッフ、配信 PC |
| 40 | user | 来場者 |

## ポートアサイン (共通設計のポート種別にマッピング)

| ポート | 接続先 | Type | モード | VLAN | 備考 |
|--------|--------|------|--------|------|------|
| Gi 0/0/0 | sw01 / r3-vyos / 他スイッチ | T1 | trunk (EVC) | 11,30,40 (native 11) | アップリンク候補 1 |
| Gi 0/0/1 | sw01 / r3-vyos / 他スイッチ | T1 | trunk (EVC) | 11,30,40 (native 11) | アップリンク候補 2 / ダウンリンク |
| Gi 0/1/0 | AP (Aironet 3800) | T2 | trunk | 11,30,40 (native 11) | FlexConnect、PoE+ 直給電 |
| Gi 0/1/1 | AP (Aironet 3800) | T2 | trunk | 11,30,40 (native 11) | FlexConnect、PoE+ 直給電 |
| Gi 0/1/2 | AP (Aironet 3800) | T2 | trunk | 11,30,40 (native 11) | FlexConnect、PoE+ インジェクター経由 |
| Gi 0/1/3 | AP (Aironet 3800) | T2 | trunk | 11,30,40 (native 11) | FlexConnect、PoE+ インジェクター経由 |
| Gi 0/1/4 | AP (Aironet 3800) | T2 | trunk | 11,30,40 (native 11) | FlexConnect、PoE+ インジェクター経由 |
| Gi 0/1/5 | AP (Aironet 3800) | T2 | trunk | 11,30,40 (native 11) | FlexConnect、PoE+ インジェクター経由 |
| Gi 0/1/6 | 配信 PC or 他スイッチ | T3/T1 | access/trunk | 現場決定 | デフォルト access VLAN 30 |
| Gi 0/1/7 | 配信 PC or 他スイッチ | T3/T1 | access/trunk | 現場決定 | デフォルト access VLAN 30 |

## コンフィグ

```
! ============================================================
! sw02 Configuration (Cisco ISR 1100 / C1111-8P, IOS XE 17.x)
! ============================================================

! --- 基本設定 ---
hostname sw02
no ip domain lookup
ip domain name bwai.local
service timestamps debug datetime msec localtime
service timestamps log datetime msec localtime
clock timezone JST 9 0

! --- VLAN 定義 (ESM switchport 用) ---
vlan 11
 name mgmt
vlan 30
 name staff
vlan 40
 name user

! --- Bridge-Domain 定義 (EVC 用。VLAN と同一番号で対応付け) ---
bridge-domain 11
bridge-domain 30
bridge-domain 40

! --- 管理 SVI (L3) ---
interface Vlan11
 description Management SVI
 ip address 192.168.11.6 255.255.255.0
 no shutdown

ip default-gateway 192.168.11.1

! --- VLAN SVI を bridge-domain にブリッジ (L2 結合) ---
! ESM 側の VLAN 11/30/40 を WAN 側 EVC と同じ L2 ドメインに結合する
interface Vlan11
 service instance 11 ethernet
  encapsulation untagged
  bridge-domain 11
!
interface Vlan30
 no ip address
 no shutdown
 service instance 30 ethernet
  encapsulation untagged
  bridge-domain 30
!
interface Vlan40
 no ip address
 no shutdown
 service instance 40 ethernet
  encapsulation untagged
  bridge-domain 40

! ============================================================
! WAN 側ポート (0/0/x): EVC で L2 トランク化
! ============================================================

! Gi 0/0/0: L2 trunk (アップリンク候補 1)
interface GigabitEthernet0/0/0
 description L2-Trunk-EVC (uplink or downlink)
 no ip address
 negotiation auto
 no shutdown
 service instance 11 ethernet
  encapsulation dot1q 11
  rewrite ingress tag pop 1 symmetric
  bridge-domain 11
 !
 service instance 30 ethernet
  encapsulation dot1q 30
  rewrite ingress tag pop 1 symmetric
  bridge-domain 30
 !
 service instance 40 ethernet
  encapsulation dot1q 40
  rewrite ingress tag pop 1 symmetric
  bridge-domain 40

! Gi 0/0/1: L2 trunk (アップリンク候補 2 / ダウンリンク)
interface GigabitEthernet0/0/1
 description L2-Trunk-EVC (uplink or downlink)
 no ip address
 negotiation auto
 no shutdown
 service instance 11 ethernet
  encapsulation dot1q 11
  rewrite ingress tag pop 1 symmetric
  bridge-domain 11
 !
 service instance 30 ethernet
  encapsulation dot1q 30
  rewrite ingress tag pop 1 symmetric
  bridge-domain 30
 !
 service instance 40 ethernet
  encapsulation dot1q 40
  rewrite ingress tag pop 1 symmetric
  bridge-domain 40

! ============================================================
! LAN 側ポート (0/1/x): ESM switchport
! ============================================================

! Gi 0/1/0-1: Type T2 — AP trunk (FlexConnect、SSID ローカルスイッチング)
!   PoE+ 直給電
interface range GigabitEthernet0/1/0-1
 description T2-AP-Aironet3800 (PoE+ direct)
 switchport mode trunk
 switchport trunk allowed vlan 11,30,40
 switchport trunk native vlan 11
 spanning-tree portfast trunk
 spanning-tree bpduguard enable
 power inline auto
 no shutdown

! Gi 0/1/2-5: Type T2 — AP trunk (FlexConnect、SSID ローカルスイッチング)
!   PoE+ インジェクター経由 (本体 PoE は OFF)
interface range GigabitEthernet0/1/2-5
 description T2-AP-Aironet3800 (via PoE+ injector)
 switchport mode trunk
 switchport trunk allowed vlan 11,30,40
 switchport trunk native vlan 11
 spanning-tree portfast trunk
 spanning-tree bpduguard enable
 power inline never
 no shutdown

! Gi 0/1/6-7: Type T3 (default) or T1 (flex) — 配信 PC or 他スイッチ連結
! デフォルトは access VLAN 30 (配信 PC 用)。trunk へ変更する場合は下記「現場での柔軟運用」参照
interface range GigabitEthernet0/1/6-7
 description T3-Flex-Port (staff PC or switch uplink/downlink)
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! --- Spanning Tree ---
spanning-tree mode rapid-pvst
spanning-tree vlan 11,30,40 priority 8192

! --- LLDP ---
lldp run

! --- SSH ---
ip ssh version 2
line vty 0 4
 transport input ssh
 login local

end
```

## 現場での柔軟運用

### ケース 1: `Gi 0/1/6-7` をトランクに切り替え (他スイッチのアップリンク/ダウンリンク)

```
interface GigabitEthernet0/1/6
 no switchport access vlan
 switchport mode trunk
 switchport trunk allowed vlan 11,30,40
 switchport trunk native vlan 11
```

### ケース 2: `Gi 0/1/6-7` を access VLAN 40 (来場者セグメント) に切り替え

```
interface GigabitEthernet0/1/6
 switchport access vlan 40
```

### ケース 3: PoE トラブル時の PoE 無効化

```
interface GigabitEthernet0/1/0
 power inline never
```

## 設計メモ

### ISR 1100 を L2 スイッチとして使う際の制約

- **`Gi 0/0/0`, `Gi 0/0/1` は routed interface 固定**: `switchport` コマンドは拒否される。L2 ブリッジ化には EVC (`service instance ethernet` + `encapsulation dot1q` + `bridge-domain`) が必須。
- **ESM の VLAN と WAN 側 `bridge-domain` は別の L2 ドメイン**: 同一番号にしただけでは結合されない。`interface Vlan X` に `service instance + bridge-domain X` を設定することで ESM 側 VLAN と EVC bridge-domain を結合する必要がある。
- **PoE バジェットが限られる**: C1111-8P は総 PoE 50W。Aironet 3800 の PoE+ (25.5W) は 2 台が限界。以降は外部 PoE+ インジェクターで対応。
- **`rewrite ingress tag pop 1 symmetric`**: 受信時に dot1q タグを剥がして bridge-domain に転送、送信時に再付与する定型句。VLAN 番号と `encapsulation dot1q` の値が一致する前提で使う。

### AP ポートを trunk (Type T2) にする意図

共通設計 ([`venue-switch.md`](./venue-switch.md) §2 Type T2) に従い、AP ポートを access VLAN 11 から **trunk (allowed 11,30,40, native 11)** に変更している。これにより AP は以下のように動作する:

- 管理 IP は native VLAN 11 (untagged) で r3-vyos の DHCP プール (.100–.199) から取得
- クライアントトラフィックは AP 側 (FlexConnect / スタンドアロン) で SSID ごとに VLAN 30 / 40 タグを付与してスイッチへ送出
- WLC 3504 は設定管理と認証のみを担当 (トラフィックパスから外れる)

sw01 と揃えた設計。WLC 側での FlexConnect local switching 有効化と合わせて運用する。

### 論理的な位置付け (未確定 — 現場柔軟)

本設計書は sw02 を **「現場で sw01 の下位、または r3-vyos 直結の並列スイッチとして投入できる」** 前提で書いている。どちらで使っても設定は変わらない:

- **下位配置 (sw01 ダウンリンク)**: `Gi 0/0/0` を sw01 の `Ge 0/10` などにトランク接続。下位で AP を収容する拡張スイッチとして動作。
- **並列配置 (r3-vyos 直結)**: `Gi 0/0/0` を Proxmox ホストの追加 NIC / VLAN 対応ブリッジに接続。sw01 と独立した L2 で AP を収容。
  - ただし BPDU が疎通する経路構成では STP ループに注意。`spanning-tree mode rapid-pvst` を sw01 と揃えていれば RPVST+ で収束する。

### 実機検証チェックリスト (投入前に必須)

ISR 1100 の EVC + ESM VLAN の結合は機種・IOS バージョン依存の挙動があるため、**本番投入前に自宅環境で必ず疎通確認を行うこと**。

- [ ] `show bridge-domain 11`, `30`, `40` で service instance と Vlan interface の両方が member として表示される
- [ ] sw02 の `Gi 0/1/0` に繋いだ機器から、sw02 の `Gi 0/0/0` にトランク接続した上位 (r3-vyos / sw01) を経由して `192.168.11.1` に ping が通る
- [ ] sw02 `Vlan11` (192.168.11.6) 自身が上位からの SSH を受け付ける
- [ ] AP (Aironet 3800) が `Gi 0/1/0` の native VLAN 11 経由で管理 IP (192.168.11.100–.199) を DHCP 取得できる
- [ ] AP の SSID ごとに VLAN 30 / 40 タグ付きトラフィックが送出され、スイッチを経由して r3-vyos で受信される
- [ ] VLAN 30/40 の DHCP リースが来場者/スタッフ PC で取得できる
- [ ] `show mac address-table bridge-domain 11` で ESM 側と EVC 側両方の MAC が見える

## 関連ドキュメント

- [`venue-switch.md`](./venue-switch.md) — 会場スイッチ共通設計 (マルチベンダー)
- [`venue-switch1.md`](./venue-switch1.md) — sw01 (FS) 実装例
- [`mgmt-vlan-address.md`](./mgmt-vlan-address.md) — 管理 VLAN アドレス割当表

## 参考文献

- [Cisco 1000 Series Software Configuration Guide — Configuring Ethernet Switch Ports (XE 17)](https://www.cisco.com/c/en/us/td/docs/routers/access/isr1100/software/configuration/xe-17/isr1100-sw-config-xe-17/configuring_ethernet_switchports.html)
- [Cisco 1000 Series Software Configuration Guide — Configuring Bridge Domain Interfaces (XE 17)](https://www.cisco.com/c/en/us/td/docs/routers/access/isr1100/software/configuration/xe-17/isr1100-sw-config-xe-17/bdi_isr1k.html)
- [Cisco Community — Bridging on IOS XE (ISR 1100 Series)](https://community.cisco.com/t5/routing/bridging-on-ios-xe-isr-1100-series/td-p/3988120)
- [Cisco Community — Bridge between routing and switching interface on ISR 1100](https://community.cisco.com/t5/routing/bridge-between-a-routing-interface-and-a-switching-interface-on/td-p/3747590)
