# GitHub Actions ワークフロー動作確認

このリポジトリでは、GitHub Actionsのワークフロー動作を詳細に検証しました。

## ワークフロー概要

- **debug**: GitHub Contextの情報を出力（常時動作）
- **deploy-prod**: 本番環境デプロイ用（mainブランチのみ）
- **deploy-sandbox**: サンドボックス環境デプロイ用（PR操作時のみ）

## 検証済み動作パターン

### ✅ 正常動作パターン

| イベント | debug | deploy-prod | deploy-sandbox | 備考 |
|---------|-------|-------------|----------------|------|
| **PR作成** (opened) | ✅ | ❌ | ✅ | 新しいPR作成時にsandbox環境を構築 |
| **PR更新** (synchronize) | ✅ | ❌ | ✅ | PRにコミット追加時にsandbox環境をリフレッシュ |
| **PRラベル付け** (labeled) | ✅ | ❌ | ✅* | `deploy-sandbox`ラベル付与時のみ |
| **mainブランチマージ** | ✅ | ✅ | ❌ | 本番環境のみデプロイ、sandboxは動作しない |
| **他ブランチpush** | ✅ | ❌ | ❌ | debug情報のみ出力 |
| **ブランチ間マージ** | ✅ | ❌ | ❌ | マージコミット時は不要な処理を回避 |

### 🎯 主要な解決課題

**課題**: PRをmainブランチにマージした時にdeploy-sandboxが意図せず動作していた

**解決**: deploy-sandboxの実行条件をPRイベント（`opened`, `synchronize`, `labeled`）のみに限定

### 📋 各ジョブの実行条件

#### deploy-prod
```yaml
if: ${{ github.ref_name == 'main' }}
```
- mainブランチでのみ実行

#### deploy-sandbox  
```yaml
if: |
  (
    github.event_name == 'pull_request'
    && github.event.action == 'opened'
  )
  ||
  (
    github.event_name == 'pull_request'
    && github.event.action == 'synchronize'
  )
  ||
  (
    github.event_name == 'pull_request'
    && github.event.action == 'labeled'
    && github.event.label.name == 'deploy-sandbox'
  )
```
- PRの作成、更新、特定ラベル付与時のみ実行
- pushイベントでは実行されない（ブランチ間マージを含む）

## 検証方法

1. **PR作成テスト**: 新しいブランチからmainへのPR作成
2. **PR更新テスト**: PRブランチへの追加コミット
3. **mainマージテスト**: PRをmainブランチにマージ
4. **ブランチ間マージテスト**: feature → developのようなブランチ間マージ

全てのパターンで期待通りの動作を確認済み。

## 設定ファイル

- `.github/workflows/deploy.yml`: メインワークフロー設定