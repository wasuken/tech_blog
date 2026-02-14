---
title: "Proxmox LXCコンテナでJupyterLab環境構築 - 試行錯誤とトラブルシューティング"
date: 2026-01-26T18:30:00+09:00
draft: false
tags: ["Proxmox", "LXC", "JupyterLab", "Ubuntu", "Python", "venv", "systemd", "インフラ", "トラブルシューティング"]
---

## はじめに

Proxmox上にJupyterLabのLXC環境を構築しました。当初はGeminiに任せて試行錯誤しましたが、最終的にベストプラクティスに辿り着いたので、その過程と解決策をまとめます。

## 構築の基本方針

当初は「Root + グローバル環境」で構築しようとしましたが、最終的に**「専用ユーザー + 仮想環境（venv）」**による安全でクリーンな構成に落ち着きました。

### 最終構成

- **OS**: Ubuntu 24.04 LTS (LXC Container)
- **ユーザー**: `jupyter` (非Root運用)
- **Jupyter**: JupyterLab (v4.x)
- **環境**: `/opt/jupyter/venv` (OSと分離した仮想環境)

## 環境構築手順

### 1. OSの準備

Ubuntu 24.04の最小構成に必要なパッケージをインストールします。

```bash
apt update && apt upgrade -y
apt install -y python3-full build-essential
```

`python3-full`が重要です。これがないと後述するPEP 668の問題に直面します。

### 2. 専用ユーザーとディレクトリの作成

```bash
# 専用ユーザー作成
useradd -m -s /bin/bash jupyter

# Jupyter本体用のディレクトリ準備
mkdir -p /opt/jupyter
chown jupyter:jupyter /opt/jupyter
```

### 3. 仮想環境の構築

`jupyter`ユーザーとして、OSの制限を受けない独立した環境を作ります。

```bash
su - jupyter
python3 -m venv /opt/jupyter/venv
source /opt/jupyter/venv/bin/activate

# JupyterLabとカーネルのインストール
pip install jupyterlab ipykernel pandas
```

### 4. systemdによるデーモン化

`/etc/systemd/system/jupyter.service`を作成します。

```ini
[Unit]
Description=JupyterLab Server
After=network.target

[Service]
Type=simple
User=jupyter
Group=jupyter
WorkingDirectory=/home/jupyter
ExecStart=/opt/jupyter/venv/bin/jupyter-lab \
    --no-browser \
    --ip=0.0.0.0 \
    --ServerApp.token='' \
    --ServerApp.password='' \
    --ServerApp.allow_remote_access=True \
    --ServerApp.allow_origin='*'
Restart=always

[Install]
WantedBy=multi-user.target
```

**※記事では全許可だが、実際は内部でも非推奨なので可能なら絞る。**

サービスの有効化と起動:

```bash
systemctl daemon-reload
systemctl enable jupyter.service
systemctl start jupyter.service
```

## 遭遇したトラブルと解決策

### 1. `externally-managed-environment` エラー

**原因**: PEP 668によるOS側のPython環境保護機能

**解決策**: `python3-full`導入後に`venv`を使用する。どうしてもグローバルにインストールしたい場合は`/usr/lib/python3.12/EXTERNALLY-MANAGED`ファイルを削除する方法もありますが非推奨です。

### 2. WebSocket接続エラー

**エラーメッセージ**: "A connection to the notebook server could not be established"

**原因**: WebSocketのOriginチェックによる拒否

**解決策**: 起動引数に`--ServerApp.allow_origin='*'`を追加

### 3. `ModuleNotFoundError: jupyter_server`

**原因**: OS版とpip版のJupyterが衝突

**解決策**: 

```bash
apt remove python3-notebook python3-jupyter-core
pip install --force-reinstall jupyterlab
```

### 4. `invalid metadata entry 'name'`

**原因**: メタデータの破損

**解決策**: `/usr/lib/python3.12/dist-packages/`内の該当`.dist-info`フォルダを削除

```bash
# 例
rm -rf /usr/lib/python3.12/dist-packages/jupyter_core-*.dist-info
```

### 5. `TypeError: warn() missing argument`

**原因**: ライブラリのバージョン不一致

**解決策**:

```bash
pip install --upgrade --force-reinstall jupyter-core jupyter-client
```

## プロジェクトごとのKernel追加

JupyterLab本体の環境を汚さず、プロジェクトごとに環境を使い分ける方法です。

```bash
# 新しい環境を作成
python3 -m venv /path/to/project_env

# 必要なパッケージをインストール
/path/to/project_env/bin/pip install ipykernel numpy scipy

# Jupyterに登録
/path/to/project_env/bin/python -m ipykernel install --user --name "project_name"
```

JupyterLabのKernel選択画面に"project_name"が表示されるようになります。

## まとめ

Proxmox上のLXCコンテナでJupyterLab環境を構築する際は、以下のポイントを押さえることで堅牢な環境が作れます:

1. **venvによる環境分離**: OSのPython環境と分離する
2. **専用ユーザーでの運用**: Root実行を避ける
3. **systemdによる管理**: 自動起動と再起動を設定
4. **Origin制限の緩和**: WebSocket接続を許可する設定

Geminiに任せたときは各種エラーに遭遇しましたが、一つずつ解決していくことで最終的に安定した環境を構築できました。

## 参考資料

- [JupyterLab Documentation](https://jupyterlab.readthedocs.io/)
- [PEP 668 – Marking Python base environments as "externally managed"](https://peps.python.org/pep-0668/)
