# 会場スイッチ共通設計 (マルチベンダー対応)

## 目的

会場に投入するスイッチは **sw01 (FS)**, **sw02 (Cisco ISR 1100)** に加え、現場調達や持ち込み機材としてマルチベンダー (Cisco / HPE Aruba / Juniper / MikroTik / 各社中古品) が参加する可能性がある。本ドキュメントはベンダー・機種に依存しない **共通 L2 設計ルール** を定義し、各機種の具体コンフィグは実装例ドキュメントに委譲する。

| 項目 | 参照先 |
|------|--------|
| 抽象的な共通設計 (本ドキュメント) | `venue-switch.md` |
| sw01 実装例 (FS, Cisco-like CLI) | [`venue-switch1.md`](./venue-switch1.md) |
| sw02 実装例 (Cisco ISR 1100 / C1111-8P) | [`venue-switch2.md`](./venue-switch2.md) |
| 管理 VLAN のアドレス一覧 | [`mgmt-vlan-address.md`](./mgmt-vlan-address.md) |

---

## 1. VLAN 定義 (全スイッチ共通)

| VLAN ID | 名称 | 用途 | IPv4 | IPv6 |
|---------|------|------|------|------|
| 11 | mgmt | NW 機器管理、AP 管理 IP | 192.168.11.0/24 | なし (v4 only) |
| 30 | staff | 運営スタッフ、配信 PC、スピーカー | 192.168.30.0/24 | DHCPv6-PD /64 |
| 40 | user | 来場者 | 192.168.40.0/22 | DHCPv6-PD /64 |

全スイッチで上記 3 VLAN を一貫して定義する。VLAN ID・名称・用途の揃えはベンダー横断でのトランク接続成立に必須。

---

## 2. ポート種別の抽象モデル

会場スイッチに接続されるすべてのポートは、以下 5 種類のいずれかに分類する。ベンダーが変わっても **この分類と設定意図は変わらず**、各ベンダーの構文にマッピングするだけで設定できる。

### Type T1: Uplink/Downlink Trunk (スイッチ・ルーター・コントローラ間相互接続)

**用途**: r3-vyos、Proxmox ミニ PC、**WLC (Cisco 3504)**、他スイッチ相互接続 (sw01 ↔ sw02, 追加スイッチのアップリンク/ダウンリンク)

**WLC を T1 に含める理由**: Cisco 3504 は管理 I/F + AP Manager I/F + Dynamic Interface (VLAN 30/40) を dot1q で 1 本に収容する前提。FlexConnect 運用でも Central DHCP / ウェブ認証 / ゲストアンカーなどで Dynamic Interface を使う可能性があり、access に絞ると拡張性を失う。WLC 側では管理 I/F の VLAN Identifier を 11 に設定して tagged VLAN 11 で自力参加させる (§6 の思想)。

| 項目 | 値 |
|------|-----|
| モード | trunk (802.1Q dot1q) |
| Allowed VLAN | 11, 30, 40 |
| Native VLAN | 11 |
| STP | BPDU 疎通、portfast trunk は使わない (または portfast trunk + BPDU guard 併用) |
| LLDP | 有効 |

**補足**: Native VLAN を 11 (mgmt) に統一する理由は、ベンダー混在時に tagging の不整合が発生した場合でも管理経路 (VLAN 11) だけは untagged fallback で疎通する可能性を残すため。ただし原則として全機器が tagged VLAN 11 で自力参加する設計なので、native VLAN は保険 (詳細は §6 参照)。

### Type T2: AP Trunk (SSID ローカルスイッチング)

**用途**: Aironet 3800 などの AP。AP 自身が管理 IP を VLAN 11 から DHCP 取得し、SSID ごとに VLAN 30/40 を付与してエッジに送出する (FlexConnect またはスタンドアロン運用)。

| 項目 | 値 |
|------|-----|
| モード | trunk (802.1Q dot1q) |
| Allowed VLAN | 11, 30, 40 |
| Native VLAN | 11 |
| PoE | PoE+ (IEEE 802.3at, 25.5W) 給電。本体給電能力を超える場合は外部 PoE+ インジェクター経由 |
| STP | portfast trunk + BPDU guard (AP は BPDU を送出しない前提) |

**管理 IP の取り方**:
- AP は起動時に untagged (native VLAN 11) または tagged VLAN 11 で DHCP Discover を送出
- r3-vyos の DHCP プール `192.168.11.100–.199` から管理 IP をリース
- クライアントトラフィックは AP が SSID 設定に従い VLAN 30 / 40 タグを付与してスイッチに送出

### Type T3: Endpoint Access (PC, スピーカー, 固定機器)

**用途**: 配信 PC、スピーカー、運営有線席、来場者向け有線 (稀)

| 項目 | 値 |
|------|-----|
| モード | access |
| VLAN | 30 (staff/配信 PC/スピーカー) または 40 (user) |
| STP | portfast 有効 + BPDU guard 併用 |
| Storm control | 推奨 (ブロードキャスト/マルチキャストを上限 1〜5%) |
| ポートセキュリティ | 任意 (MAC 数制限) |

### Type T4: Management Access (稀)

**用途**: NMS 機材、ベンダー持込の管理ワークステーションなど、VLAN 11 に直接ぶら下げるケース

| 項目 | 値 |
|------|-----|
| モード | access |
| VLAN | 11 |
| STP | portfast 有効 + BPDU guard 併用 |

原則として VLAN 40 (user) からは VLAN 11 へのアクセスは拒否 (ACL は r3-vyos 側)。本タイプは運営スタッフのみ使用。

### Type T5: Unused (shutdown)

**用途**: 未使用ポート。誤接続による L2 ループ、VLAN 侵害を防ぐため **必ず shutdown**。

| 項目 | 値 |
|------|-----|
| 状態 | shutdown |
| Access VLAN | (任意の未使用 VLAN、あるいは "black hole" VLAN 999) |

---

## 3. 管理 IP ルール (忘れずに)

**すべての会場スイッチは例外なく、以下を設定する。**

1. **VLAN 11 の SVI (Switch Virtual Interface) を作成**
2. **192.168.11.0/24 から静的 IP を 1 つ割り当てる** (割り当て表は [`mgmt-vlan-address.md`](./mgmt-vlan-address.md))
3. **デフォルトゲートウェイを 192.168.11.1 (r3-vyos) に設定**
4. **SSH 経由で疎通できることを確認**

| 機器 | 管理 IP | 備考 |
|------|---------|------|
| sw01 (FS) | 192.168.11.4 | 主系 L2 スイッチ |
| sw02 (Cisco ISR 1100) | 192.168.11.6 | 補助 L2 スイッチ |
| sw03+ | 192.168.11.7〜 | 追加スイッチ (現場で予約) |

**この手順を忘れると、スイッチに SSH で入れず現地で物理コンソールを探す羽目になる。必ず構築時に自宅環境で疎通確認してから搬入すること。**

---

## 4. STP (スパニングツリー) 方針

### モード選定

- **既定**: **Rapid PVST+** (`rapid-pvst`) — Cisco/FS/一部 HPE でサポート。VLAN ごとにインスタンスを持つので収束が早く、VLAN トラフィック分散にも使える
- **ベンダー混在時**: **MSTP** を検討 — Aruba CX / Juniper / MikroTik と混ぜる場合はこちら。MST インスタンスに VLAN 11/30/40 をまとめて載せる
- **同一ベンダーのみ** (例: Cisco ISR + Cisco Catalyst) なら PVST+/RPVST+ 一択

### ルートブリッジ

- **プライマリルート**: sw01 (FS、集約スイッチ)
- **セカンダリルート**: sw02 (Cisco ISR 1100)
- priority は 4096 単位で設定 (sw01=4096, sw02=8192, その他=32768 デフォルト)

### ポート別 STP 設定

| ポートタイプ | portfast | BPDU guard | 備考 |
|--------------|----------|-----------|------|
| T1 (upstream trunk) | no | no | BPDU 疎通を維持 |
| T2 (AP trunk) | portfast trunk | yes | AP は BPDU 送出しない |
| T3 (endpoint access) | yes | yes | 端末は BPDU 送出しない |
| T4 (mgmt access) | yes | yes | 同上 |
| T5 (shutdown) | — | — | 無効化 |

---

## 5. マルチベンダー実装コマンド対比

共通設計を各ベンダーの CLI にマッピングする際の参考表。**厳密な構文はベンダー公式ドキュメントで確認すること** (バージョン依存あり)。

| 操作 | Cisco IOS / IOS-XE (ESM) / FS (Cisco-like) | HPE Aruba CX | Juniper Junos (ELS) | MikroTik RouterOS |
|------|--------------------------------------------|--------------|---------------------|-------------------|
| VLAN 作成 | `vlan 11` `name mgmt` | `vlan 11` `name mgmt` | `set vlans mgmt vlan-id 11` | `/interface bridge vlan add bridge=bridge1 vlan-ids=11` |
| Access ポート (T3/T4) | `switchport mode access` `switchport access vlan 30` | `interface 1/1/1` `no routing` `vlan access 30` | `set interfaces ge-0/0/0 unit 0 family ethernet-switching interface-mode access vlan members staff` | `/interface bridge port add bridge=bridge1 interface=ether3 pvid=30` |
| Trunk ポート (T1/T2) | `switchport mode trunk` `switchport trunk allowed vlan 11,30,40` `switchport trunk native vlan 11` | `interface 1/1/1` `no routing` `vlan trunk allowed 11,30,40` `vlan trunk native 11` | `set interfaces ge-0/0/0 unit 0 family ethernet-switching interface-mode trunk vlan members [mgmt staff user]` `set interfaces ge-0/0/0 native-vlan-id 11` | `/interface bridge port add bridge=bridge1 interface=ether1 pvid=11 frame-types=admit-all` + `/interface bridge vlan add bridge=bridge1 tagged=ether1 vlan-ids=11,30,40` |
| 管理 SVI (VLAN 11 IP) | `interface Vlan11` `ip address 192.168.11.x 255.255.255.0` | `interface vlan 11` `ip address 192.168.11.x/24` | `set interfaces irb unit 11 family inet address 192.168.11.x/24` `set vlans mgmt l3-interface irb.11` | `/interface vlan add interface=bridge1 name=vlan11 vlan-id=11` + `/ip address add address=192.168.11.x/24 interface=vlan11` |
| デフォルト GW | `ip default-gateway 192.168.11.1` | `ip route 0.0.0.0/0 192.168.11.1` | `set routing-options static route 0.0.0.0/0 next-hop 192.168.11.1` | `/ip route add dst-address=0.0.0.0/0 gateway=192.168.11.1` |
| STP (RPVST+) | `spanning-tree mode rapid-pvst` | `spanning-tree mode rapid-pvst` (CX は機種限定) | Junos 既定 RSTP、`set protocols rstp` | `/interface bridge set bridge1 protocol-mode=rstp` |
| STP portfast/edge | `spanning-tree portfast` | `spanning-tree port-type admin-edge` | (Junos は `edge`) | `/interface bridge port set edge=yes` |
| BPDU guard | `spanning-tree bpduguard enable` | `spanning-tree bpdu-guard` | `set protocols rstp bpdu-block-on-edge` | `/interface bridge port set bpdu-guard=yes` |
| LLDP 有効化 | `lldp run` | `lldp` | `set protocols lldp interface all` | `/interface bridge port set auto-isolate=no` + LLDP は別設定 |

**注意点**:

- **Native VLAN 表記の差異**: Cisco は `native vlan 11`、Junos は `native-vlan-id 11`、MikroTik は `pvid 11`。意味は同じだが構文が異なる
- **Aruba CX の native VLAN**: デフォルトで `vlan trunk native 1`。明示的に `vlan trunk native 11` を設定しないと VLAN 1 untagged になる罠あり
- **Junos ELS の `interface-mode` vs 旧 `port-mode`**: ELS (EX2300/3400 以降) は `interface-mode`、旧機種は `port-mode`。設定前にバージョン確認
- **MikroTik は bridge VLAN filtering** を `/interface bridge set bridge1 vlan-filtering=yes` で有効化しないと VLAN が機能しない。忘れやすい
- **PVST+/RPVST+ の IEEE 互換**: Cisco PVST+ は独自、RPVST+ は IEEE 802.1w 互換。ベンダー混在時は **MSTP が最も安全**

---

## 6. tagged VLAN 11 自力参加設計 (運用上の重要注記)

本プロジェクトでは **配下機器が全員 tagged VLAN 11 で自力参加する** 設計になっている:

- **VyOS r3**: `eth2.11 address 192.168.11.1/24` (tagged)
- **Proxmox ホスト**: `vmbr_trunk.11 address 192.168.11.3/24` (tagged、VLAN-aware ブリッジの子 IF)
- **local-srv CT**: `net0 bridge=vmbr_trunk,tag=11` (Proxmox が tagged で渡す)
- **WLC / AP**: trunk 経由で VLAN 11 tagged で管理 IP 取得
- **sw01 / sw02 / 追加スイッチ**: VLAN 11 SVI は tagged VLAN 11 で L3 参加

この設計により、スイッチ側の `switchport trunk native vlan 11` は「あれば動作が安定する」程度の **ベストエフォート** であり、**スイッチの設定ミス・初期化・代替機材への差し替えが発生しても L3 疎通に影響しない**。

### 運用上避けるべきこと

- **スイッチ側で VLAN 11 を untagged (access) に変換してしまう構成変更**: これを行うと tagged 参加している機器 (Proxmox、VyOS、WLC、他スイッチの VLAN 11 SVI) 側と L2 が分断される
- **Native VLAN の ID をスイッチごとに変える**: Native mismatch が発生するとベンダーによっては dot1q フレームを落とす
- **VLAN 1 を残すこと**: 未使用 VLAN 1 は shutdown / 別 VLAN に置換すること

### 逆に許容されること

- 共通設計の trunk 設定をそのまま投入して動かない場合でも、**機器側が tagged で自力参加するので先に進める**判断が可能
- ベンダー固有の構文エラーで trunk が組めない場合、一時的に access VLAN 11 で管理経路だけ確保し、後で trunk 化するワークフローも許容

---

## 7. 構築・投入チェックリスト (全スイッチ共通)

- [ ] VLAN 11 / 30 / 40 を作成し、名称を揃えた (`mgmt` / `staff` / `user`)
- [ ] VLAN 11 SVI に 192.168.11.x/24 を割り当てた (x は [`mgmt-vlan-address.md`](./mgmt-vlan-address.md) 参照)
- [ ] デフォルト GW = 192.168.11.1 を設定した
- [ ] 上位 trunk (T1) を allowed 11,30,40 / native 11 で設定した
- [ ] AP trunk (T2) を allowed 11,30,40 / native 11 / portfast trunk + BPDU guard で設定した
- [ ] 端末 access (T3) を VLAN 30 または 40 / portfast + BPDU guard で設定した
- [ ] 未使用ポート (T5) を shutdown した
- [ ] STP モードを rapid-pvst または mstp に設定した
- [ ] LLDP を有効化した (ベンダー横断トポロジ確認のため)
- [ ] 自宅環境で r3-vyos 経由で SSH ログインできることを確認した
- [ ] 自宅環境で AP / 配信 PC / 来場者端末エミュレーションで DHCP リースが取得できることを確認した

## 8. 会場搬入時の差分作業

会場搬入後、VyOS (r3) が起動して BGP が上がったら、各スイッチは特に設定変更不要 (デフォルト GW `192.168.11.1` はそのまま)。

ただし **構築時に r1-home 経由で運用している場合** は、デフォルト GW が `192.168.11.254` (r1-home eth3.11) を指しているので、会場搬入時に `192.168.11.1` (r3-vyos) に戻すこと。この差し替え手順は各実装例ドキュメント ([`venue-switch1.md`](./venue-switch1.md), [`venue-switch2.md`](./venue-switch2.md)) を参照。
