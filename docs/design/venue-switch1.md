# 会場スイッチ sw01 (FS) 実装例

> **前提**: 本ドキュメントは [`venue-switch.md`](./venue-switch.md) で定義したマルチベンダー共通設計 (VLAN モデル、ポート種別 T1〜T5、管理 IP ルール、STP 方針) の **FS 製スイッチ (Cisco-like CLI) による実装例** である。共通設計を先に読むこと。

## 概要

| 項目 | 値 |
|------|-----|
| ホスト名 | sw01 |
| 機種 | FS (Cisco ライク CLI) |
| 役割 | 会場の主系 L2 集約スイッチ |
| 管理 VLAN | 11 |
| 管理 IP | 192.168.11.4/24 |
| デフォルト GW | 192.168.11.1 (r3-vyos) |
| 構築時 GW | 192.168.11.254 (r1-home eth3.11) |

## ポート一覧

| ポート | 速度 | タイプ |
|--------|------|--------|
| XSGigabitEthernet 0/1 | 10G | fiber (SFP+) |
| XSGigabitEthernet 0/2 | 10G | fiber (SFP+) |
| FiveGigabitEthernet 0/1 | 5G | copper |
| FiveGigabitEthernet 0/2 | 5G | copper |
| GigabitEthernet 0/3–10 | 1G | copper (×8) |

## ポートアサイン (共通設計のポート種別にマッピング)

| ポート | 接続先 | Type | モード | VLAN | 備考 |
|--------|--------|------|--------|------|------|
| XSGe 0/1 | 予備 | T5 | shutdown | — | 10G fiber 予備 |
| XSGe 0/2 | r1-home eth3 (構築時) | T1 | trunk | 11,30,40 (native 11) | 構築時のみ、会場では未使用 |
| 5Ge 0/1 | Proxmox onboard NIC | T1 | trunk | 11,30,40 (native 11) | r3-vyos VLAN トランク (1GbE ネゴ) |
| 5Ge 0/2 | WLC (Cisco 3504) | T1 | trunk | 11,30,40 (native 11) | FlexConnect AP 管理 |
| Ge 0/3–7 | AP (Aironet 3800) | T2 | trunk | 11,30,40 (native 11) | SSID ローカルスイッチング、PoE+ 給電、5 台まで |
| Ge 0/8–9 | 配信 PC、スピーカー | T3 | access | 30 | 運営有線 |
| Ge 0/10 | 予備 | T5 | shutdown | — | 誤接続防止 |

**AP 収容数**: sw01 では PoE+ ポート数 (Ge 0/3–7) の制約で最大 5 台。残り 15 台は sw02 および追加 PoE スイッチ経由で収容。

## コンフィグ

```
! ============================================================
! sw01 Configuration (FS switch, Cisco-like CLI)
! 共通設計: venue-switch.md を参照
! ============================================================

! --- 基本設定 ---
hostname sw01
no ip domain-lookup

! --- VLAN 定義 ---
vlan 11
 name mgmt
vlan 30
 name staff
vlan 40
 name user

! --- 管理インターフェース (T3 相当、SVI) ---
interface vlan 11
 ip address 192.168.11.4 255.255.255.0
 no shutdown

ip default-gateway 192.168.11.1

! --- 未使用 VLAN 1 無効化 ---
interface vlan 1
 shutdown

! ============================================================
! Type T1: Uplink/Downlink Trunk ポート
! ============================================================

! XSGe 0/2: r1-home eth3 SFP+ (構築時のアップリンク)
interface XSGigabitEthernet 0/2
 description T1-Uplink-r1-home(construction)
 switchport mode trunk
 switchport trunk allowed vlan 11,30,40
 switchport trunk native vlan 11
 spanning-tree portfast trunk
 no shutdown

! 5Ge 0/1: Proxmox onboard NIC → r3-vyos (VM) の trunk
interface FiveGigabitEthernet 0/1
 description T1-Trunk-Proxmox(r3-vyos)
 switchport mode trunk
 switchport trunk allowed vlan 11,30,40
 switchport trunk native vlan 11
 spanning-tree portfast trunk
 no shutdown

! 5Ge 0/2: WLC (Cisco 3504)
interface FiveGigabitEthernet 0/2
 description T1-Trunk-WLC3504
 switchport mode trunk
 switchport trunk allowed vlan 11,30,40
 switchport trunk native vlan 11
 spanning-tree portfast trunk
 no shutdown

! ============================================================
! Type T2: AP Trunk ポート (FlexConnect / SSID ローカルスイッチング)
! ============================================================

! Ge 0/3-7: AP (Aironet 3800) — trunk、SSID → VLAN 振り分けは AP 側
interface range GigabitEthernet 0/3-7
 description T2-AP-Aironet3800
 switchport mode trunk
 switchport trunk allowed vlan 11,30,40
 switchport trunk native vlan 11
 spanning-tree portfast trunk
 spanning-tree bpduguard enable
 no shutdown

! ============================================================
! Type T3: Endpoint Access ポート
! ============================================================

! Ge 0/8-9: 運営有線 (配信 PC、スピーカー)
interface range GigabitEthernet 0/8-9
 description T3-Staff-Wired
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ============================================================
! Type T5: 未使用ポート (shutdown)
! ============================================================

interface XSGigabitEthernet 0/1
 description T5-Reserved
 shutdown

interface GigabitEthernet 0/10
 description T5-Reserved
 shutdown

! --- Spanning Tree ---
spanning-tree mode rapid-pvst
spanning-tree vlan 11,30,40 priority 4096

! --- LLDP ---
lldp run

! --- SSH ---
ip ssh version 2
line vty 0 15
 transport input ssh
 login local

end
```

## AP を trunk native VLAN 11 にする意図

共通設計 (Type T2) に従い、AP ポートを access VLAN 11 から **trunk (allowed 11,30,40, native 11)** に変更した。これに伴う運用モデルの違い:

| 項目 | 従前 (access VLAN 11 + CAPWAP 中央スイッチング) | 現行 (trunk + FlexConnect ローカルスイッチング) |
|------|-----------------------------------------------|-----------------------------------------------|
| AP 物理ポート | access VLAN 11 | trunk allowed 11,30,40 / native 11 |
| AP 管理 IP | VLAN 11 untagged で DHCP 取得 | VLAN 11 untagged (native) で DHCP 取得 |
| クライアントトラフィック | CAPWAP トンネルで WLC に集約 → WLC で VLAN 30/40 タグ付け | AP が直接 VLAN 30/40 タグ付けしてスイッチへ送出 |
| WLC 3504 の役割 | 全トラフィック経路・設定管理 | 設定管理のみ (トラフィックは AP ローカルで処理) |
| 利点 | WLC で集中 ACL・QoS 適用可 | WLC 経由の帯域消費を回避、WLC 故障時もクライアント疎通維持 |
| 必要な AP 設定 | CAPWAP discover のみ | FlexConnect モード有効化 + Native VLAN 設定 |

**重要**: AP 側 (Aironet 3800 + WLC 3504) で FlexConnect モードを有効化し、各 SSID に対し "FlexConnect local switching" を設定する必要がある。この設定は WLC 3504 の WebUI から行う。

## 構築時の変更点

構築時 (自宅) はデフォルト GW を r1-home に向ける:

```
ip default-gateway 192.168.11.254
```

会場搬入時に戻す:

```
ip default-gateway 192.168.11.1
```

## 設計メモ

- **XSGe 0/2 を構築用に使用**: r1-home eth3 は X710-DA4 の SFP+ ポート。10G fiber 同士で接続
- **5Ge ポートに Proxmox/WLC を収容**: 1GbE 機器でも auto-negotiate で接続可能。Ge ポートを AP/有線用に確保
- **STP priority 4096**: sw01 を VLAN 11/30/40 のプライマリルートブリッジに指定 (sw02 は 8192 でセカンダリ)
- **BPDU guard を AP/端末ポートに**: portfast 系ポートで BPDU を受信したら即座に errdisable 化、誤接続による L2 ループを防止

## 関連ドキュメント

- [`venue-switch.md`](./venue-switch.md) — 会場スイッチ共通設計 (マルチベンダー)
- [`venue-switch2.md`](./venue-switch2.md) — sw02 (Cisco ISR 1100) 実装例
- [`mgmt-vlan-address.md`](./mgmt-vlan-address.md) — 管理 VLAN アドレス割当表
- [`venue-proxmox.md`](./venue-proxmox.md) — Proxmox 側の VLAN-aware ブリッジ設計
