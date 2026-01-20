---
title: "K3s実験環境構築マニュアル：安全に実験→破壊→復元のサイクルを回す"
date: 2026-01-19T10:30:00+09:00
draft: false
tags: ["kubernetes", "k3s", "proxmox", "infrastructure", "devops"]
categories: ["Infrastructure"]
summary: "Proxmox上のK3s環境で安全に実験・破壊・復元サイクルを回すための完全ガイド"
---

## 概要

このマニュアルでは、Proxmox上のK3s環境で「設定を試しまくる→壊れる→完全に元に戻す」という実験サイクルを安全に行うためのセットアップと運用方法を説明します。

## 前提条件

- Proxmox VE環境
- VM上にK3sがインストール済み
- kubectlが使用可能

## 1. Dashboard環境の構築

### 1.0 Helmのインストール

まずはHelmをインストールします：

```bash
# 公式スクリプトでインストール（推奨）
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# インストール確認
helm version

# 必要なリポジトリを追加
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
```

**重要**: Helmでエラーが出る場合は、kubeconfigの設定を確認してください：

```bash
# kubeconfigを設定
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# または永続的に設定
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc

# 動作確認
kubectl get nodes
```

### 1.1 公式Kubernetes Dashboard

#### インストール

```bash
# kubernetes-dashboard namespaceを作成
kubectl create namespace kubernetes-dashboard

# Dashboardをインストール
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -n kubernetes-dashboard
```

#### アクセス方法

**方法A: ポートフォワード（開発用）**
```bash
# フォアグラウンドで実行（ログが見やすい）
kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard-kong-proxy 8443:443

# バックグラウンドで実行
kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard-kong-proxy 8443:443 &
```

**方法B: NodePort（推奨）**
```bash
# ServiceをNodePortに変更
kubectl patch svc kubernetes-dashboard-kong-proxy -n kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}'

# 割り当てられたポートを確認
kubectl get svc kubernetes-dashboard-kong-proxy -n kubernetes-dashboard

# ブラウザでアクセス
# https://<k3s-master-ip>:<nodeport>
```

#### 認証設定

管理者用のサービスアカウントとトークンを作成：

```bash
# サービスアカウント作成
kubectl create serviceaccount dashboard-admin-sa -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin-sa

# トークン生成（都度実行推奨）
kubectl -n kubernetes-dashboard create token dashboard-admin-sa

# 長時間有効なトークンが必要な場合
kubectl -n kubernetes-dashboard create token dashboard-admin-sa --duration=24h
```

**セキュリティベストプラクティス**: トークンは都度生成することを推奨します。デフォルトで1時間の有効期限があり、セキュリティ的により安全です。

#### アクセス確認

- ブラウザでDashboardにアクセス
- ログイン画面で「Token」を選択
- 上記で生成したトークンを入力
- 証明書警告が出た場合は「詳細設定」→「続行」で進む

#### トラブルシューティング

**エラー: "400 Bad Request - The plain HTTP request was sent to HTTPS port"**
- HTTPSでアクセスしてください: `https://` を使用
- このエラーはサービスが正常に動作している証拠です

**エラー: "502 Bad Gateway"**
- サービスが起動していない可能性があります
- `kubectl get pods -n kubernetes-dashboard` でPod状態を確認

**エラー: namespaces "kubernetes-dashboard" not found**
- デフォルトnamespaceにインストールされている可能性があります
- `kubectl get svc --all-namespaces | grep dashboard` で確認

アクセス: https://localhost:8443 (ポートフォワード) または https://<master-ip>:<nodeport> (NodePort)

### 1.2 Lens（推奨デスクトップアプリ）

1. [Lens公式サイト](https://k8slens.dev/)からダウンロード
2. インストール後、kubeconfigを読み込み
3. 直感的なGUIでクラスター管理が可能

### 1.3 k9s（ターミナルUI）

```bash
# インストール（各OS対応）
# macOS
brew install k9s

# Linux
curl -sS https://webinstall.dev/k9s | bash

# 起動
k9s
```

## 2. バックアップ・リストア戦略

### 2.1 レイヤー別復元戦略

| レベル | 方法 | 復元時間 | 粒度 | 用途 |
|--------|------|----------|------|------|
| L1: VM全体 | Proxmoxスナップショット | 1-2分 | 粗い | 大規模な変更前 |
| L2: K8s設定 | kubectl yaml出力 | 30秒-5分 | 中程度 | アプリデプロイ前 |
| L3: 個別アプリ | Helmバックアップ | 10-30秒 | 細かい | 設定変更前 |

### 2.2 Proxmoxスナップショット運用

#### スナップショット作成
```bash
# VMのスナップショット作成
qm snapshot <vmid> clean-k3s-$(date +%Y%m%d-%H%M)

# スナップショット一覧確認
qm listsnapshot <vmid>
```

#### 復元
```bash
# 特定のスナップショットに復元
qm rollback <vmid> clean-k3s-20240119-1400

# VM再起動
qm reboot <vmid>
```

### 2.3 Kubernetes設定レベルバックアップ

#### 全体バックアップスクリプト
```bash
#!/bin/bash
# backup-k8s.sh

BACKUP_DIR="./k8s-backups"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="$BACKUP_DIR/backup-$TIMESTAMP"

mkdir -p "$BACKUP_PATH"

# 全リソースバックアップ
echo "Backing up all resources..."
kubectl get all --all-namespaces -o yaml > "$BACKUP_PATH/all-resources.yaml"

# 重要な設定別バックアップ
kubectl get configmaps --all-namespaces -o yaml > "$BACKUP_PATH/configmaps.yaml"
kubectl get secrets --all-namespaces -o yaml > "$BACKUP_PATH/secrets.yaml"
kubectl get persistentvolumes -o yaml > "$BACKUP_PATH/pv.yaml"
kubectl get persistentvolumeclaims --all-namespaces -o yaml > "$BACKUP_PATH/pvc.yaml"

# Helmリリース一覧
helm list --all-namespaces -o yaml > "$BACKUP_PATH/helm-releases.yaml"

echo "Backup completed: $BACKUP_PATH"
```

#### 復元スクリプト
```bash
#!/bin/bash
# restore-k8s.sh

if [ -z "$1" ]; then
    echo "Usage: $0 <backup-timestamp>"
    echo "Available backups:"
    ls -1 ./k8s-backups/ | grep backup-
    exit 1
fi

BACKUP_PATH="./k8s-backups/backup-$1"

if [ ! -d "$BACKUP_PATH" ]; then
    echo "Backup not found: $BACKUP_PATH"
    exit 1
fi

# 既存リソースの削除（注意）
read -p "This will delete existing resources. Continue? (y/N): " confirm
if [ "$confirm" != "y" ]; then
    echo "Aborted."
    exit 1
fi

# 復元実行
echo "Restoring from $BACKUP_PATH..."
kubectl delete --all deployments --all-namespaces
kubectl delete --all services --all-namespaces
kubectl delete --all configmaps --all-namespaces

kubectl apply -f "$BACKUP_PATH/all-resources.yaml"

echo "Restore completed."
```

### 2.4 個別アプリケーションバックアップ

#### Helmを使った管理
```bash
# アプリケーションの現在の設定を取得
helm get values my-app > my-app-backup-values.yaml

# 復元
helm upgrade my-app bitnami/nginx -f my-app-backup-values.yaml
```

## 3. 実験フローの実践

### 3.1 実験前の準備
```bash
# 1. Proxmoxスナップショット作成
qm snapshot <vmid> before-experiment-$(date +%Y%m%d-%H%M)

# 2. K8s設定バックアップ
./backup-k8s.sh

# 3. 現在のHelmリリース確認
helm list --all-namespaces
```

### 3.2 実験中のモニタリング
```bash
# k9sでリアルタイム監視
k9s

# または特定リソースの監視
kubectl get pods --all-namespaces -w
```

### 3.3 問題発生時の復元手順

#### パターン1: アプリレベルの問題
```bash
# Helmで個別復元
helm rollback my-app 1
```

#### パターン2: 複数リソースの問題
```bash
# K8s設定レベルで復元
./restore-k8s.sh 20240119-140500
```

#### パターン3: システム全体の問題
```bash
# Proxmoxスナップショットで復元
qm rollback <vmid> before-experiment-20240119-1400
qm reboot <vmid>
```

## 4. 効率的な実験のTips

### 4.1 namespace分離戦略
```bash
# 実験用namespaceを作成
kubectl create namespace experiment

# 実験はこのnamespace内で実行
helm install test-app bitnami/nginx -n experiment

# 実験終了後、namespace削除で全クリーンアップ
kubectl delete namespace experiment
```

### 4.2 定期的なクリーンアップ
```bash
# 古いスナップショットの削除（手動確認後）
qm delsnapshot <vmid> old-snapshot-name

# 古いバックアップファイルの削除
find ./k8s-backups -type d -mtime +30 -exec rm -rf {} \;
```

### 4.3 よく使うコマンドのエイリアス
```bash
# ~/.bashrc or ~/.zshrc に追加
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kdp='kubectl describe pod'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
```

## 5. トラブルシューティング

### 5.1 よくある問題

**Q: スナップショット復元後、k3sが起動しない**
```bash
# k3sサービスの状態確認
sudo systemctl status k3s

# 手動再起動
sudo systemctl restart k3s
```

**Q: kubectl接続できない**
```bash
# kubeconfigの確認
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# または
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

**Q: PersistentVolumeが復元されない**
- PVは物理ストレージと連動するため、VM復元では不整合が起こる可能性
- 重要なデータは別途バックアップを推奨

### 5.2 緊急時対応

システム全体が応答しない場合：
1. Proxmoxコンソールから直接アクセス
2. 最新の安定スナップショットに復元
3. k3sの手動再インストール（最終手段）

## 6. 運用ルール

### 6.1 バックアップのタイミング
- **毎日**: 自動でK8s設定バックアップ
- **実験前**: 必ずProxmoxスナップショット
- **週次**: 古いバックアップのクリーンアップ

### 6.2 実験の記録
```bash
# 実験ログの記録
echo "$(date): Starting experiment with new ingress config" >> experiment.log
```

### 6.3 安全な実験のガイドライン
- 本番環境では絶対に実験しない
- 重要な設定変更前は必ずバックアップ
- 実験用namespaceを活用
- 変更内容をGitで管理

## まとめ

この環境により、安全に「実験→破壊→復元」のサイクルを高速で回すことが可能です。Proxmoxスナップショット、Kubernetes設定バックアップ、Helm管理を組み合わせることで、様々なレベルの復元オプションを用意し、効率的な学習と開発を支援します。
