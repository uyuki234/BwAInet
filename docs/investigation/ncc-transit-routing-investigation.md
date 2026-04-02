# GCP Network Connectivity Center (NCC) トランジットルーティング調査

## 調査日: 2026-04-02

## 目的

r1-home (AS65002) と r3-venue (AS65001) 間の直接 WireGuard リンクが落ちた場合に、GCP 経由でフォールバックするトランジットルーティングを NCC で実現する構成を調査する。

## 現在の構成 (参考)

```
r3-venue (AS65001) ====[WireGuard wg0]==== r1-home (AS65002)
       |                                        |
   HA VPN (primary)                         HA VPN (backup)
       |                                        |
       +---------- GCP VPC (AS64512) -----------+
                   Cloud Router
```

- r1-r3 間: WireGuard (10.255.0.0/30) + BGP (AS65001 <-> AS65002)
- r3 -> GCP: HA VPN + BGP (AS65001 <-> AS64512) primary
- r1 -> GCP: HA VPN + BGP (AS65002 <-> AS64512) backup
- 現在の設計では GCP は GCE サービス (Grafana, rsyslog) への接続用。r1-r3 間のトランジットは未構成。

## 1. NCC ハブ・スポーク構成の設定手順

### 前提条件

- GCP プロジェクトで Network Connectivity API を有効化
- HA VPN ゲートウェイ、Cloud Router、VPN トンネルが既に存在すること
- VPC のダイナミックルーティングモードが **global** であること (リージョン間ルート交換に必要)

### 1.1 API 有効化

```bash
gcloud services enable networkconnectivity.googleapis.com \
    --project=PROJECT_ID
```

### 1.2 NCC ハブの作成

```bash
gcloud network-connectivity hubs create bwai-transit-hub \
    --project=PROJECT_ID \
    --description="BwAI transit hub for r1-home <-> r3-venue fallback" \
    --preset-topology=mesh
```

`--preset-topology` は `mesh`, `star`, `hybrid-inspection` が選択可能。2 拠点間のフルメッシュ接続には `mesh` が適切。

### 1.3 VPC のダイナミックルーティングモード確認・設定

r1-home と r3-venue の HA VPN が異なるリージョンにある場合、VPC のダイナミックルーティングモードを `global` にする必要がある。

```bash
# 確認
gcloud compute networks describe VPC_NAME \
    --project=PROJECT_ID \
    --format="get(routingConfig.routingMode)"

# global に変更
gcloud compute networks update VPC_NAME \
    --bgp-routing-mode=global \
    --project=PROJECT_ID
```

## 2. HA VPN トンネルをハイブリッドスポークとして登録

### 2.1 r3-venue 用スポークの作成

```bash
gcloud network-connectivity spokes linked-vpn-tunnels create spoke-r3-venue \
    --hub=bwai-transit-hub \
    --vpn-tunnels=projects/PROJECT_ID/regions/REGION_A/vpnTunnels/vpn-tunnel-r3-0,projects/PROJECT_ID/regions/REGION_A/vpnTunnels/vpn-tunnel-r3-1 \
    --region=REGION_A \
    --site-to-site-data-transfer \
    --description="HA VPN spoke for r3-venue (AS65001)" \
    --project=PROJECT_ID
```

### 2.2 r1-home 用スポークの作成

```bash
gcloud network-connectivity spokes linked-vpn-tunnels create spoke-r1-home \
    --hub=bwai-transit-hub \
    --vpn-tunnels=projects/PROJECT_ID/regions/REGION_B/vpnTunnels/vpn-tunnel-r1-0,projects/PROJECT_ID/regions/REGION_B/vpnTunnels/vpn-tunnel-r1-1 \
    --region=REGION_B \
    --site-to-site-data-transfer \
    --description="HA VPN spoke for r1-home (AS65002)" \
    --project=PROJECT_ID
```

### 重要事項

- **各スポークに 2 本のトンネルが必要** (HA VPN の冗長構成)
- **スポークのリージョンは HA VPN ゲートウェイと同じリージョンでなければならない**
- **`--site-to-site-data-transfer` フラグはスポーク作成後に変更不可**。作成時に必ず指定すること。

### 2.3 Terraform の場合

```hcl
resource "google_network_connectivity_hub" "transit_hub" {
  name        = "bwai-transit-hub"
  project     = var.project_id
  description = "BwAI transit hub for r1-home <-> r3-venue fallback"
}

resource "google_network_connectivity_spoke" "spoke_r3_venue" {
  name     = "spoke-r3-venue"
  project  = var.project_id
  hub      = google_network_connectivity_hub.transit_hub.id
  location = var.region_a

  linked_vpn_tunnels {
    uris                       = [
      google_compute_vpn_tunnel.r3_tunnel_0.self_link,
      google_compute_vpn_tunnel.r3_tunnel_1.self_link,
    ]
    site_to_site_data_transfer = true
  }
}

resource "google_network_connectivity_spoke" "spoke_r1_home" {
  name     = "spoke-r1-home"
  project  = var.project_id
  hub      = google_network_connectivity_hub.transit_hub.id
  location = var.region_b

  linked_vpn_tunnels {
    uris                       = [
      google_compute_vpn_tunnel.r1_tunnel_0.self_link,
      google_compute_vpn_tunnel.r1_tunnel_1.self_link,
    ]
    site_to_site_data_transfer = true
  }
}
```

## 3. site-to-site data transfer の有効化

### 有効化方法

site-to-site data transfer は **スポーク作成時の `--site-to-site-data-transfer` フラグ** で有効化する。スポーク作成後に変更することはできない。

このフラグを指定しないと、デフォルトで `site-to-cloud` モードとなり、オンプレミス間のルート再広告が行われない。

### 動作原理

site-to-site data transfer を有効にすると:

1. 各ハイブリッドスポークの Cloud Router が BGP で学習したルートが、同じハブに接続された他のハイブリッドスポークに **再広告 (re-advertise)** される
2. r3-venue が BGP で広告する `192.168.11.0/24`, `192.168.30.0/24`, `192.168.40.0/22` が r1-home 側の Cloud Router 経由で r1-home に再広告される (逆も同様)
3. これにより、WireGuard が落ちた場合に GCP 経由のフォールバック経路が自動的に利用可能になる

### 要件

- **全てのデータ転送有効スポークが同一 VPC 内であること** (最重要)
- VPC のダイナミックルーティングモードが `global` であること (リージョン間ルート交換時)
- 各スポークが HA 構成であること (トンネル 2 本)

## 4. Cloud Router 側で必要な追加設定

### 4.1 Cloud Router の ASN 要件 (最重要)

NCC で site-to-site data transfer を使用する場合、以下の ASN ルールに従う必要がある:

| 要件 | 詳細 |
|------|------|
| **Cloud Router の ASN** | 同一ハブに関連付けられた全ての Cloud Router は**同じ ASN** を使用すること |
| **ピアルーターの ASN** | 異なるスポークのピアルーターは**異なる ASN** を使用すること |
| **同一スポーク内のピア** | 同一スポーク内の全てのピアルーターは**同じ ASN** を使用すること |

#### 現在の設計との適合性

現在の設計:
- r3-venue (ピア ASN): AS65001
- r1-home (ピア ASN): AS65002
- GCP Cloud Router ASN: AS64512

**異なるスポーク (r3, r1) のピア ASN がそれぞれ AS65001, AS65002 で異なるため、NCC の要件を満たしている。**

ただし、**Cloud Router は全て同じ ASN を使う必要がある**。現在 GCP 側が AS64512 であれば、r1 用・r3 用の Cloud Router 両方とも AS64512 にする必要がある (これは通常のセットアップで自然にそうなる)。

### 4.2 カスタムルート広告

NCC の site-to-site data transfer 有効時、Cloud Router はハイブリッドスポーク間のルートを自動的に再広告する。ただし:

- **Cloud Router のカスタム学習ルート (custom learned routes) は site-to-site data transfer では無視される**。外部サイトに伝播されない。
- カスタム広告ルート (custom advertised routes) は設定可能。デフォルトルート (0.0.0.0/0) の広告などに使用できる。

### 4.3 推奨する Cloud Router 設定

```bash
# r3-venue 用 Cloud Router (REGION_A)
gcloud compute routers create cr-r3-venue \
    --network=VPC_NAME \
    --region=REGION_A \
    --asn=64512 \
    --project=PROJECT_ID

# r1-home 用 Cloud Router (REGION_B)
gcloud compute routers create cr-r1-home \
    --network=VPC_NAME \
    --region=REGION_B \
    --asn=64512 \
    --project=PROJECT_ID
```

両方とも **ASN 64512** を使用する。

## 5. BGP の AS 番号・ルート設計と注意点

### 5.1 AS path loop detection の問題

**最も注意すべき点**: NCC の site-to-site data transfer では、スポーク A のピアが広告したルートがスポーク B のピアに再広告される際、AS path にスポーク A のピア ASN が含まれる。

この動作は正常だが、**2 つのスポークが同じピア ASN を使っている場合、AS path loop detection によりルートがドロップされる**。

現在の設計では:
- spoke-r3-venue のピア ASN: **65001** (r3-venue)
- spoke-r1-home のピア ASN: **65002** (r1-home)

**異なる ASN なので AS path loop detection の問題は発生しない。**

### 5.2 フォールバック優先度の制御 (AS path prepend)

WireGuard 直接リンクを優先し、GCP 経由をフォールバックとして使うため、**GCP 経由のルートの優先度を下げる**必要がある。

#### 方法 1: VyOS 側で AS path prepend (推奨)

r1-home と r3-venue の VyOS で、GCP 向けの BGP セッションで広告するルートに AS path prepend を付加する。

```
# r3-venue (VyOS) — GCP 向け BGP で AS path prepend
set policy route-map GCP-OUT rule 10 action permit
set policy route-map GCP-OUT rule 10 set as-path prepend '65001 65001 65001'

set protocols bgp neighbor <gcp-cloud-router-ip> address-family ipv4-unicast route-map export GCP-OUT
```

```
# r1-home (VyOS) — GCP 向け BGP で AS path prepend
set policy route-map GCP-OUT rule 10 action permit
set policy route-map GCP-OUT rule 10 set as-path prepend '65002 65002 65002'

set protocols bgp neighbor <gcp-cloud-router-ip> address-family ipv4-unicast route-map export GCP-OUT
```

これにより:
- WireGuard 直接 BGP: AS path 長 = 1 (優先)
- GCP 経由 BGP: AS path 長 = 4+ (フォールバック)

#### 方法 2: VyOS 側で local-preference

WireGuard ピアから受信するルートに高い local-preference を設定する。

```
# r3-venue — WireGuard ピア (r1) からのルートを優先
set policy route-map WG-IN rule 10 action permit
set policy route-map WG-IN rule 10 set local-preference 200

set protocols bgp neighbor 10.255.0.1 address-family ipv4-unicast route-map import WG-IN

# GCP ピアからのルートはデフォルト (local-preference 100)
```

#### 方法 3: BGP distance の活用 (現在の設計との整合性)

現在 r3 では BGP distance global external を 20 に設定している。GCP 経由のルートも eBGP のため同じ AD 20 になる。WireGuard 直接と GCP 経由の区別には local-preference か AS path prepend が必要。

### 5.3 ルート広告の設計

NCC site-to-site data transfer が有効な場合のルート伝播:

```
r3-venue が BGP で広告するルート:
  192.168.11.0/24, 192.168.30.0/24, 192.168.40.0/22
  → GCP Cloud Router (spoke-r3) が学習
  → NCC hub が spoke-r1 の Cloud Router に再広告
  → r1-home が BGP で受信

r1-home が BGP で広告するルート:
  192.168.10.0/24 (家族用 LAN)
  → GCP Cloud Router (spoke-r1) が学習
  → NCC hub が spoke-r3 の Cloud Router に再広告
  → r3-venue が BGP で受信
```

### 5.4 ルート重複時の優先順位

同じプレフィックスが WireGuard 直接と GCP 経由の両方から学習される場合:

1. **BGP best path selection** で AS path の短い方 (WireGuard 直接) が選択される
2. WireGuard が落ちると BGP セッションも落ち、WireGuard 経由のルートが withdraw される
3. GCP 経由のルートのみが残り、自動的にフォールバックが発生

## 6. NCC の料金体系

### 6.1 コスト構成

| 項目 | 課金 | 備考 |
|------|------|------|
| NCC ハブ | 無料 | ハブ自体に料金は発生しない |
| VPN スポーク時間 | **最初の 3 VPN スポークまで無料** | スポーク時間 = ACTIVE 状態の時間/月 |
| Interconnect スポーク時間 | 最初の 3 Interconnect スポークまで無料 | |
| ADN (Advanced Data Networking) | VPC スポーク発: $0.02/GiB/月 | ハイブリッドスポーク (VPN) 発は**現在無料** |
| site-to-site データ転送 | ベストエフォート | 帯域・レイテンシの保証なし |

### 6.2 BwAI 構成での追加コスト見積もり

| 項目 | コスト |
|------|--------|
| NCC ハブ | 無料 |
| spoke-r3-venue (VPN) | **無料** (3 VPN スポーク以内) |
| spoke-r1-home (VPN) | **無料** (3 VPN スポーク以内) |
| ハイブリッドスポーク間データ転送 (ADN) | **現在無料** (VPN スポーク発は waived) |
| 基盤の HA VPN トンネル料金 | 既存コスト (NCC とは別に発生) |
| VPN 経由のエグレス通信料 | 既存コスト (NCC とは別に発生) |

**結論: BwAI の 2 VPN スポーク構成では、NCC 自体の追加コストは実質ゼロ。** 既存の HA VPN 料金のみ。

ただし、Google が ADN 無料期間を終了した場合、ハイブリッドスポーク間のデータ転送にも $0.02/GiB が発生する可能性がある。フォールバック用途であればデータ量は限定的なため、大きな影響はない。

## 7. 制約事項・注意点

### 7.1 技術的制約

| 制約 | 影響 |
|------|------|
| **IPv4 のみ** | ハイブリッドスポークは IPv4 のみ対応。IPv6 トランジットには使えない |
| **Classic VPN 非対応** | HA VPN のみ対応。Classic VPN トンネルはスポークに登録できない |
| **同一 VPC 必須** | site-to-site data transfer を使う全スポークが同一 VPC 内にあること |
| **カスタム学習ルートが無視される** | Cloud Router の custom learned routes は site-to-site では伝播しない |
| **BGP communities 非対応** | NCC はBGP communities をサポートしない |
| **スポーク作成後に data transfer モード変更不可** | 間違えたら削除→再作成が必要 |
| **ベストエフォート** | site-to-site data transfer は帯域・レイテンシの保証なし |
| **レガシーネットワーク非対応** | スポークリソースはレガシーネットワークに配置できない |

### 7.2 BwAI 固有の注意点

| 項目 | 詳細 |
|------|------|
| **IPv6 フォールバック不可** | NCC ハイブリッドスポークは IPv4 のみ。DHCPv6-PD /64 の GCP 経由フォールバックは NCC では不可能。IPv6 は WireGuard 直接リンクに完全依存する |
| **AS path 設計** | WireGuard 直接の BGP と GCP 経由の BGP が同じプレフィックスを広告するため、AS path prepend か local-preference で優先制御が必須 |
| **HA VPN 2 トンネル必須** | 現在の設計に HA VPN が含まれているか要確認。各スポークに 2 本のトンネルが必要 |
| **Cloud Router ASN 統一** | r1 用と r3 用の Cloud Router が同じ ASN (64512) であること |
| **フォールバック時のレイテンシ** | GCP バックボーン経由のため、WireGuard 直接より大幅にレイテンシが増加する可能性 |
| **VPC ルーティングモード** | `global` にしないとリージョン間のルート交換ができない |

### 7.3 運用上の注意

1. **フェイルオーバー時間**: WireGuard が落ちてから BGP が withdraw し、GCP 経由に切り替わるまでの時間は BGP タイマー依存 (デフォルト hold-time 90 秒)。BFD を設定すればサブ秒単位の検出が可能だが、WireGuard は BFD をネイティブサポートしない。

2. **戻りの経路**: フォールバック中は r3→GCP→r1→Internet というパスになるため、レイテンシが増加する。ただし、WireGuard 復旧後は BGP が自動的に最適パスに戻る。

3. **MTU**: GCP HA VPN の MTU は 1460 (IPsec オーバーヘッド)。NCC 経由のパスでは WireGuard トンネルを使わないため MTU 設計は異なる。

## 8. 推奨構成 (まとめ)

### 構成図

```
r3-venue (AS65001) =====[WireGuard wg0 / BGP]====== r1-home (AS65002)
  |  AS path: 65001                 AS path: 65002  |
  |  (優先: short path)             (優先: short path) |
  |                                                   |
  HA VPN tunnel x2                    HA VPN tunnel x2
  (AS path prepend: 65001x3)         (AS path prepend: 65002x3)
  |                                                   |
  spoke-r3-venue                       spoke-r1-home
  [site-to-site=true]                  [site-to-site=true]
  |                                                   |
  +------- NCC Hub (bwai-transit-hub) ----------------+
           GCP VPC (global routing mode)
           Cloud Router ASN: 64512
```

### 必要な変更点

1. **GCP 側**:
   - Network Connectivity API 有効化
   - NCC ハブ作成
   - 既存 HA VPN トンネルをハイブリッドスポークとして登録 (`--site-to-site-data-transfer`)
   - VPC ダイナミックルーティングモードを `global` に設定

2. **r1-home (VyOS)**:
   - GCP 向け BGP で AS path prepend (3 回) を設定
   - 192.168.10.0/24 を GCP 向け BGP で広告 (r3 がフォールバック時にインターネットへ到達するため)
   - GCP ピアからの受信ルートに低い local-preference を設定 (任意)

3. **r3-venue (VyOS)**:
   - GCP 向け BGP で AS path prepend (3 回) を設定
   - GCP ピアからの受信ルートに低い local-preference を設定 (任意)
   - GCP ピアからデフォルトルートを受信するよう設定 (フォールバック用)

### VyOS 設定例 (r3-venue に追加)

```
# GCP Cloud Router とのBGP
set protocols bgp neighbor <gcp-cr-ip> remote-as 64512
set protocols bgp neighbor <gcp-cr-ip> description 'GCP Cloud Router (NCC spoke)'
set protocols bgp neighbor <gcp-cr-ip> address-family ipv4-unicast

# GCP 向けルート広告に AS path prepend (フォールバック優先度を下げる)
set policy route-map GCP-EXPORT rule 10 action permit
set policy route-map GCP-EXPORT rule 10 set as-path prepend '65001 65001 65001'
set protocols bgp neighbor <gcp-cr-ip> address-family ipv4-unicast route-map export GCP-EXPORT

# GCP から受信するルートの local-preference を下げる
set policy route-map GCP-IMPORT rule 10 action permit
set policy route-map GCP-IMPORT rule 10 set local-preference 50
set protocols bgp neighbor <gcp-cr-ip> address-family ipv4-unicast route-map import GCP-IMPORT

# WireGuard ピアからの受信ルートを優先
set policy route-map WG-IMPORT rule 10 action permit
set policy route-map WG-IMPORT rule 10 set local-preference 200
set protocols bgp neighbor 10.255.0.1 address-family ipv4-unicast route-map import WG-IMPORT
```

## 9. 結論・所見

### NCC 導入の利点

- 追加コストが実質ゼロ (VPN スポーク 3 つまで無料、ハイブリッドスポーク ADN 無料)
- 既存の HA VPN インフラをそのまま流用可能
- BGP による自動フェイルオーバーが実現可能
- 設定変更はスポーク登録と AS path prepend 追加のみ

### NCC 導入の制約

- **IPv6 フォールバックが不可能** (最大の制約。DHCPv6-PD /64 の転送は WireGuard に完全依存)
- ベストエフォートであり帯域・レイテンシ保証なし
- カスタム学習ルートが伝播しない制約がある

### 推奨

BwAI の用途 (イベント期間中の WireGuard 障害時フォールバック) では、NCC は**コスト・複雑さの面で合理的な選択肢**。ただし IPv6 のフォールバックは別途検討が必要 (例: GRE over HA VPN, または IPv6 は WireGuard 依存で割り切る)。

---

Sources:
- [NCC overview](https://docs.cloud.google.com/network-connectivity/docs/network-connectivity-center/concepts/overview)
- [Work with hubs and spokes](https://docs.cloud.google.com/network-connectivity/docs/network-connectivity-center/how-to/working-with-hubs-spokes)
- [ASN requirements for site-to-site data transfer](https://docs.cloud.google.com/network-connectivity/docs/network-connectivity-center/concepts/asn-requirements)
- [Site-to-site data transfer overview](https://docs.cloud.google.com/network-connectivity/docs/network-connectivity-center/concepts/data-transfer)
- [Connect two sites by using VPN spokes (Tutorial)](https://docs.cloud.google.com/network-connectivity/docs/network-connectivity-center/tutorials/connecting-two-offices-with-vpns)
- [Locations supported for data transfer](https://docs.cloud.google.com/network-connectivity/docs/network-connectivity-center/concepts/locations)
- [Network Connectivity pricing](https://cloud.google.com/network-connectivity/pricing)
- [gcloud network-connectivity spokes linked-vpn-tunnels create](https://cloud.google.com/sdk/gcloud/reference/network-connectivity/spokes/linked-vpn-tunnels/create)
- [BGP route policies overview](https://cloud.google.com/network-connectivity/docs/router/concepts/bgp-route-policies-overview)
- [google_network_connectivity_spoke (Terraform)](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/network_connectivity_spoke)
