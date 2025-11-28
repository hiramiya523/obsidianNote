---
title: Hyper-V NAT ネットワーク構築
created: 2025-11-28
tags:
  - windows
  - hyper-v
  - vm
  - network
  - nat
aliases:
  - VM構築
  - 仮想マシン
---

# Hyper-V NAT ネットワーク構築

> [!abstract] 概要
> Hyper-VでVMを構築し、内部通信用のNATネットワークを設定する手順。

---

## 前提: ネットワーク帯域の確認

### ipconfigで確認

```
イーサネット アダプター イーサネット:
  IPv4 アドレス: 192.168.100.112
  サブネット マスク: 255.255.240.0
  デフォルト ゲートウェイ: 192.168.101.1

イーサネット アダプター vEthernet (Default Switch):
  IPv4 アドレス: 172.25.112.1
  サブネット マスク: 255.255.240.0

イーサネット アダプター vEthernet (WSL):
  IPv4 アドレス: 172.29.128.1
  サブネット マスク: 255.255.240.0
```

> [!warning] 帯域の衝突に注意
> ホストの社内ネットは `192.168.96.0/20`（= 255.255.240.0）に属している。
> 
> **内部用のNATセグメントは絶対に「かぶらない帯」にする必要がある。**
> 
> → 例: `192.168.200.0/24`（Default/WSLとも衝突しない）

---

## 構築手順

### Step 1: 内部スイッチを作成

```powershell
# 内部スイッチを作成
# -SwitchType Internal: ホストOSと同ホスト上のVMだけがL2接続（外の物理LANとは未接続）
New-VMSwitch -SwitchName "NATSwitchForVM" -SwitchType Internal
```

#### 確認

```powershell
Get-VMSwitch
```

```
Name                   SwitchType NetAdapterInterfaceDescription
----                   ---------- ------------------------------
Default Switch         Internal
WSL (Hyper-V firewall) Internal
NATSwitchForVM         Internal
```

---

### Step 2: ホスト側IPを付与

```powershell
# vEthernet(NATSwitchForVM) にホスト側IPを付与
# ※社内/既存帯とかぶらない 192.168.200.0/24 を採用
New-NetIPAddress -InterfaceAlias "vEthernet (NATSwitchForVM)" -IPAddress 192.168.200.1 -PrefixLength 24
```

#### 出力例

```
IPAddress         : 192.168.200.1
InterfaceIndex    : 56
InterfaceAlias    : vEthernet (NATSwitchForVM)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Tentative
```

> [!info] AddressStateについて
> `Tentative`（一時）/ `Invalid`（永続）と2つ出るのは設定直後の一時的表示。
> 数秒〜十数秒で `Preferred`（有効）に遷移する。

---

### Step 3: Windows NATを作成

```powershell
# Windows NAT を作成（内部→外部へNAT）
New-NetNat -Name "NATNetwork" -InternalIPInterfaceAddressPrefix 192.168.200.0/24
```

#### 出力例

```
Name                             : NATNetwork
ExternalIPInterfaceAddressPrefix :
InternalIPInterfaceAddressPrefix : 192.168.200.0/24
IcmpQueryTimeout                 : 30
TcpEstablishedConnectionTimeout  : 1800
TcpTransientConnectionTimeout    : 120
TcpFilteringBehavior             : AddressDependentFiltering
UdpFilteringBehavior             : AddressDependentFiltering
UdpIdleSessionTimeout            : 120
UdpInboundRefresh                : False
Store                            : Local
Active                           : True
```

> [!success] 確認ポイント
> `Active: True` → 正常にNAT有効化完了 ✅

---

### Step 4: 設定確認

```powershell
# 数秒待ってから確認
Get-NetIPAddress -InterfaceAlias "vEthernet (NATSwitchForVM)" | ft IPAddress,PrefixLength,AddressState
Get-NetNat
```

---

## VM側の設定

VMのネットワーク設定で以下を指定:

| 項目 | 値 |
|------|-----|
| **IP Address** | 192.168.200.10/24 |
| **Gateway** | 192.168.200.1 |
| **DNS** | 192.168.200.1, 8.8.8.8 |

> [!tip] ポイント
> - VMは同じ `192.168.200.0/24` 帯に設定
> - ゲートウェイはホスト側の `192.168.200.1`

---

## 削除・リセット手順

```powershell
# NATの削除
Remove-NetNat -Name "NATNetwork"

# IPアドレスの削除
Remove-NetIPAddress -InterfaceAlias "vEthernet (NATSwitchForVM)" -IPAddress 192.168.200.1

# スイッチの削除
Remove-VMSwitch -SwitchName "NATSwitchForVM"
```

---

## 関連ドキュメント

- [[Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*

