---
title: "歴史地図アプリを雑にk3sへデプロイした"
date: 2026-02-25T22:00:00+09:00
draft: false
tags: ["k3s", "Kubernetes", "React", "Vite", "MapLibre", "デプロイ", "Proxmox"]
categories: ["インフラ", "開発"]
description: "React+Vite+MapLibreで作った歴史地図アプリをk3sにデプロイした話。hostPathマウント、Viteの罠、ngrokとTailscaleを駆使したファイル転送など泥臭い記録。"
---

## 歴史地図アプリの構成

React + TypeScript + Vite + MapLibre GL のSPA。歴史的国境データ（GeoJSON）を表示するアプリ。

データは`public/data/`以下にGeoJSONを置く構成で、`.gitignore`に含めているためリポジトリには入っていない。

ちなみに、生成するスクリプトはあるが、GEMINIを利用しないといけない。

しかし、APIキーのレート制限が入ってしまったので、ローカルで生成済みのデータを持ち込むことにした。

## インフラ構成

自宅のProxmox上にLXCコンテナとしてk3sクラスタを構築している。マスター1台＋ノード1台の最小構成。

外部公開はNginx Proxy Manager（NPM）でポートフォワーディングしており、DuckDNSのドメインにSSL終端している。

```
インターネット
    ↓
Nginx Proxy Manager（SSL終端）
    ↓
k3s NodePort
    ↓
Pod
```

## 問題：データファイルをどう持ち込むか

`public/data/`がgitignoreされているため、コンテナ内でgit cloneしてもデータが存在しない。

選択肢はいくつかあったが、今回はk3sのhostPathボリュームでマウントする方針にした。

## だるいファイル転送

データファイルをk3sノードに転送するのが一番面倒だった。

- Proxmoxのファイルアップロード → UIの制限でNG
- ngrok経由 → Tailscale環境のためlocalhostの名前解決失敗
- 結局TailscaleのIPでProxmoxホストに転送 → `pct push`でLXCコンテナへ

```bash
# ProxmoxホストからLXCへ
pct push <CTID> /path/to/data.tar.gz /tmp/data.tar.gz

# k3sマスターで解凍
mkdir -p /opt/history-map-data
tar -xzf /tmp/data.tar.gz -C /opt/history-map-data
```

## 融通の効かないViteとふわふわClaude君の罠

`npm run preview`はデフォルトで許可ホストを制限する。Nginx Proxy Manager経由でアクセスすると`Blocked request`が出る。

環境変数で全許可とかできたらよかったけど、結論だけ言うとできなかった。少なくともClaude君の指示では何をどうしても駄目だったので、最終的に`vite.config.ts`をデプロイ時に動的に書き換えることで回避した。

```yaml
cat > /app/vite.config.ts << 'EOF'
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
export default defineConfig({
  plugins: [react()],
  preview: {
    allowedHosts: ['your-domain.example.com'],
  },
})
EOF
```

## 最終的なYAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: history-map
spec:
  replicas: 1
  selector:
    matchLabels:
      app: history-map
  template:
    metadata:
      labels:
        app: history-map
    spec:
      nodeName: k3s-master
      containers:
      - name: history-map
        image: node:20-alpine
        workingDir: /app
        command: ["sh", "-c"]
        args:
        - |
          apk add --no-cache git
          git clone https://github.com/wasuken/history-map-app.git /app --depth=1
          cat > /app/vite.config.ts << 'EOF'
          import { defineConfig } from 'vite'
          import react from '@vitejs/plugin-react'
          export default defineConfig({
            plugins: [react()],
            preview: {
              allowedHosts: ['your-domain.example.com'],
            },
          })
          EOF
          mkdir -p /app/public/data
          cp -r /data/historical /app/public/data/historical
          cp -r /data/modern /app/public/data/modern
          cp /data/translation-cache.json /app/public/data/translation-cache.json
          npm install
          npx vite build
          npm run preview -- --host 0.0.0.0 --port 3000
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: map-data
          mountPath: /data
      volumes:
      - name: map-data
        hostPath:
          path: /opt/history-map-data
          type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: history-map-service
spec:
  selector:
    app: history-map
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080
  type: NodePort
```

`nodeName: k3s-master`を指定しているのはhostPathがPodの動くノード上に存在する必要があるため。ProxmoxのNPM(Nginx Proxy Manager)から
このNodePortに向けてプロキシを設定している。

## まとめ

本番運用するなら素直にDockerfileでビルドしてイメージに焼いた方がいいとかあるだろうが、今回は雑に動かすことを優先した。
