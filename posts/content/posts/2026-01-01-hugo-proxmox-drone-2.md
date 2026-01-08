---
title: "k3s + Drone CI/CD構築体験記② 手動ビルドでなんとか動いた"
date: 2026-01-01T18:00:00+09:00
draft: false
tags: ["k3s", "Drone", "CI/CD", "Hugo", "Docker", "Kubernetes", "自宅ラボ"]
---

[前回のハマり話](https://mintblog.hatenablog.com/entry/2026/01/01/112426)の続編。

今回は実際にCI/CDパイプラインを動かすところまで進めた。結論から言うと、自動化は99%完成したが、最後の1%（Webhook）で詰んだ。

## 目標設定

理想は当然これ：
- GitHub push → Drone検知 → Hugo自動ビルド → Dockerイメージ作成 → k3sデプロイ更新

ただし、私の環境には致命的な制約がある。

**制約：外部IP持ってない**

自宅サーバーはTailscaleでVPN経由でのみアクセス可能。つまりGitHubからのWebhookが届かない。まあ、DuckDNSでドメインは取ってるけど、それでもTailscale依存の構成。

それでも「やれるとこまでやってみよう」精神で進めた。

## .drone.yml 設定

最終的にはこんな感じになった：

```yaml
kind: pipeline
type: kubernetes
name: hugo-pipeline

steps:
- name: build-hugo
  image: klakegg/hugo:latest
  commands:
  - cd posts
  - hugo --minify
  - ls -la public/

- name: create-docker-context
  image: alpine:latest
  commands:
  - cp -r posts/public ./public
  - ls -la public/

- name: docker-build
  image: plugins/docker
  settings:
    registry: ghcr.io
    repo: ghcr.io/wasuken/tech_blog
    username:
      from_secret: github_username
    password:
      from_secret: github_token
    tags:
    - latest
    - "${DRONE_COMMIT_SHA:0:8}"

- name: deploy-to-k3s
  image: bitnami/kubectl
  environment:
    KUBECONFIG:
      from_secret: kubeconfig
  commands:
    - kubectl set image deployment/hugo-site hugo=ghcr.io/wasuken/tech_blog:latest
    - kubectl rollout status deployment/hugo-site

- name: deploy-complete
  image: alpine:latest
  commands:
  - echo "Hugo build complete!"
  - echo "Image pushed successfully"
```

ポイントは、HugoビルドからDockerイメージ作成、GHCR（GitHub Container Registry）へのプッシュ、最終的なk3sデプロイまで全部自動化したこと。

## hugo --minifyで謎のエラー

最初、Hugo buildで謎のエラーが出た：

```
ERROR error building site: render: failed to process "/posts/xxx/index.html": 
expected comma character or an array or object ending on line 225 and column 40
```

**原因調査：**
- ローカルPC（Ubuntu）: 成功
- k3s環境（LXCコンテナ）: 失敗
- Hugoバージョン: 0.153 vs 0.154（大きな差はない）

結局、**minifyオプションを外したら解決**。

推測だが、minifyライブラリが記事内のコードブロック（YAMLやTOMLの部分）をJSONと誤認識して構文エラーを起こしていた模様。ローカルとコンテナでのライブラリのバージョンや環境の微細な差が影響していると思われる。

まあ、ローカルブログで多少ファイルサイズがでかくても問題ないので、minifyは諦めた。

## RBAC権限でハマる

当然のように権限エラーで弾かれた：

```
Error from server (Forbidden): deployments.apps "hugo-site" is forbidden: 
User "system:serviceaccount:default:default" cannot get resource "deployments"
```

まあ、これは予想通り。Droneのdefault ServiceAccountにはdeploymentを操作する権限がない。

**解決方法：**

```bash
kubectl create clusterrolebinding default-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default
```

はい、**ガバガバ権限付与**。

本当はServiceAccount分けて最小権限で運用すべきだけど、自宅ラボの遊び環境だし、まあいいかということで。学習目的なら動かすことが優先。

## 実際のCI/CDフロー

手動ビルドボタンを押すと、以下の流れで処理される：

1. **Hugo Build**: Markdownファイル群をstaticなHTMLに変換
2. **Docker Context準備**: publicディレクトリをDockerビルド用にコピー
3. **Docker Build & Push**: GHCR（ghcr.io/wasuken/tech_blog）にコンテナイメージをpush
4. **k3s Deploy**: kubectl set imageでdeploymentのイメージを更新
5. **Rollout確認**: 新しいPodが正常に起動するまで待機

全体で3-4分程度。まあまあの速度。

## 成果と課題

**成果：**
- CI/CDパイプライン完全構築
- GitHub Container Registry連携
- k3s自動デプロイ
- 記事更新→手動ビルド→自動反映のワークフロー確立

**課題：**
- **Webhookが動かない**（Tailscale環境の制約）
- 手動トリガーが必要
- RBAC権限が雑

## 今後の改善案

1. **外部IP取得してWebhook有効化**
   - ルーター設定変更してポート開放
   - DuckDNSを外部公開用に設定変更
   - セキュリティリスクとのトレードオフ

2. **RBAC権限の細分化**
   ```bash
   # Drone専用ServiceAccount作成
   kubectl create serviceaccount drone-deployer
   # 最小権限のClusterRole作成
   # RoleBinding設定
   ```

3. **ArgoCD導入でGitOps化**
   - Droneでイメージ作成まで
   - ArgoCDでk3sデプロイ自動化

でも正直、現状でも十分実用的。記事を書いて、Drone UIで手動ビルドボタンを押すだけで自動的にブログが更新される。

## まとめ

自動化の最後の1%（Webhook）で詰んだが、99%は完全自動化できた。

k3s環境でのCI/CD構築、思ったより簡単だった。特にDroneは設定がシンプルでYAMLも分かりやすい。GitLab CIとかJenkinsとかより全然楽。

手動トリガーでも実用上は問題ないし、これで技術ブログの更新が格段に楽になった。記事を書くことに集中できる。

そして何より、**自分でCI/CDパイプラインを組んでデプロイできている**という達成感がある。インフラエンジニアになった気分。

次は監視とかログ収集とかやってみたいな。Prometeus + Grafanaとか。
