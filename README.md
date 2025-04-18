# Insighter
# Insighter – Development Master Plan

> **目的** : 個人投資家向け「リアルタイム異常検知 × 中長期シナリオ × 根拠提示 (Explainable AI)」Web アプリを **7 スプリント (約 6 週間)** で PoC→公開まで完成させる。

---
## 0. 目次
1. ゴール定義 / 非ゴール
2. 最終 UX スナップショット
3. システム構成図 (文字 + ポート + API)
4. 作業分解 (WBS) ― ブランチ / 担当 / 完了条件
5. API & SDK インベントリ
6. 開発環境 (uv / Docker) 詳細
7. CI/CD & リリースフロー
8. タイムライン (Gantt)
9. リスク & 対策
10. 参考リンク & Glossary

---
## 1. ゴール / 非ゴール
| 項目 | ゴール | 非ゴール |
|------|--------|---------|
| リアルタイム異常検知 | Timeseries Insights API でサブ秒判定し UI へ反映 | 自前 DL モデル作成
| 中長期シナリオ | Vertex AI Forecast Tabular Workflow で 30d 予測 / SHAP 出力 | オプション価格生成
| Explain | Gemini でニュース要因 + 過去類似例 | テクニカル指標学習
| 規制対応 | FINRA/NISA 開示テンプレ & ログ保存 | 完全自動取引 (アルゴ発注)

---
## 2. 最終 UX (ユーザ視点)
1. **銘柄コード入力** → ミリ秒更新チャート (異常時 ⚡ マーカー)
2. マーカークリック → サイドパネルに  
   **(1) 価格推移 30 日予測**, **(2) SHAP グラフ**, **(3) ニュース要因 (Gemini 要約)**
3. **Scenario Lab** でパラメータ (+1% 金利 等) → 30 秒で P&L ヒートマップ
4. **リスクゲージ**: ポートフォリオ CSV ⇒ VaR & 最大 DD
5. **通知設定**: Slack / Email で “価格スパイク >±3σ” アラート

---
## 3. システム構成 (テキスト図)
```
User  ⇆  React SPA (443)
          │REST / SSE
          ▼
┌───────── Cloud Run ──────────┐
│  a2a-router  (8080)         │
│  │                          │
│  ├─ price-feed-mcp (9000)   │  ← 価格 WebSocket (外部)
│  ├─ tsi-agent      (8001)   │  ↔ Timeseries Insights API
│  ├─ forecast-agent (8002)   │  ↔ Vertex AI Forecast API
│  ├─ explain-agent  (8003)   │  ↔ Gemini Pro / NewsAPI
│  └─ pubsub-emulator         │
└─────────────────────────────┘
Storage: BigQuery (history) / GCS (models) / Secret Manager
Monitoring: Cloud Logging + Error Reporting
```
*コンテナはすべて **CPU (2 vCPU / 4 GiB)** で起動、min‑instances=0。

---
## 4. 作業分解 (WBS)
| ID | ブランチ名 | 担当 | Story Pt | 概要 | 依存 | 完了条件 |
|----|-----------|------|----------|------|------|----------|
| W1 | `feature/mcp-price-feed` | Alice | 3 | FastAPI + MCP + WebSocket 価格受信 | — | `/stream` が JSON tick を Echo & UnitTest ✅ |
| W2 | `agent/tsi` | Bob | 5 | Timeseries Insights Agent, Pub/Sub 異常 publish | W1 | `/detect` にサンプル tick → 異常 JSON |
| W3 | `agent/forecast` | Carol | 8 | Vertex Forecast Pipeline + SHAP API | BigQuery hist | 30d 予測 REST & SHAP CSV |
| W4 | `agent/explain` | Dave | 5 | Gemini call + NewsAPI RAG 要因抽出 | W1 | 3 行要因 + URL list |
| W5 | `orchestrator/a2a-router` | Bob | 5 | LangGraph で C→F→E ルート | W2‑4 | `/insight` で統合レスポンス |
| W6 | `frontend/spa` | Erin | 8 | React (Vite) + Tailwind, Chart.js | W2‑5 | マーカー UI / SHAP plot |
| W7 | `infra/cloudrun` | Frank | 3 | Terraform IaC, Artifacts Reg | W2‑6 | `terraform apply` でデプロイ完了 |
| W8 | `docs` | Carol | 2 | User Guide + OpenAPI yaml | all | README CI Badge |

---
## 5. API & SDK インベントリ
| カテゴリ | 名称 | Python SDK | 料金メモ |
|----------|------|-----------|----------|
| RT 異常 | Timeseries Insights API | `google-cloud-timeseriesinsights` | リクエスト数 + 保存期間 |
| Forecast | Vertex AI Tabular WF | `google-cloud-aiplatform` | トレーニング時間 + オンライン予測 |
| LLM       | Gemini Pro | `google-generativeai` | 1k tokens ≈ $0.000125 |
| Data | BigQuery Public Crypto, FRED | `google-cloud-bigquery` | 公開分は無料枠あり |

---
## 6. 開発環境
```bash
uv venv .venv && source .venv/bin/activate
uv pip install --requirement requirements.txt
# コンテナ起動
cd infra/compose
docker compose up --build
```
*ホットリロード: `--reload` オプション使用*  
*ポート一覧: 9000 (WS), 8001‑8003 (REST), 8080 (router), 5173 (SPA dev)*

---
## 7. CI/CD & リリース
1. **Cloud Build Trigger**: develop → docker build > Artifact Registry
2. **Preview URL**: develop バージョンを `insight-dev.run.app`
3. **main merge**: `terraform apply` で prod 更新
4. **タグ付け**: `v0.3.0-poc` – デモ用固定

---
## 8. タイムライン (Gantt 概要)
```
Week1  W1 W2
Week2  W3 W4
Week3  W5 front skeleton
Week4  W6 完成 + UX polish
Week5  W7 IaC / Monitoring
Week6  バグ修正 & デモリハ
```

---
## 9. リスク & 対策
| リスク | インパクト | 対策 |
|--------|-----------|------|
| Timeseries Insights 限度超過 | 異常検知遅延 | サンプリングレート下げ / キャッシュ |
| Vertex Forecast 学習待ち >2h | スプリント遅延 | 小規模サンプルでパラメータ確認後、本番学習 |
| Gemini レート制限 | Explain 欠落 | 失敗時に NewsAPI タイトルのみ代替 |

---
## 10. 参考リンク & Glossary
* **Starter Pack**: <https://github.com/GoogleCloudPlatform/agent-starter-pack>
* **Timeseries Insights** docs
* **Vertex AI Forecast** Quickstart
* **A2A Spec** & **MCP Spec**
* **Glossary**: MCP, A2A, SHAP, VaR, VaRP

---
## 6.1 共通 Dockerfile (開発用)

初心者でも迷わないよう、**1 行ずつ意味を解説** しながら Dockerfile を示します。コピーして `Dockerfile.dev` として保存してください。

```dockerfile
############################################################
# 0. ベースイメージ
############################################################
# ─ python:3.11-slim ──────────────────────────────────────
#   Debian Slim + CPython11。軽量でセキュリティアップデートも早い。
#   GPU を使わない CPU 開発には十分。
############################################################
FROM python:3.11-slim AS base

############################################################
# 1. OS 依存パッケージ
############################################################
# build-essential : C 拡張付きライブラリを pip でビルドする際に必須
# curl            : ヘルスチェックやデバッグで使用
# git             : SDK の一部が git+URL 指定の場合に必要
# ca-certificates : HTTPS 通信の証明書束を更新
############################################################
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        curl \
        git \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*

############################################################
# 2. Homebrew (Linuxbrew) のインストール
############################################################
# 公式スクリプトは 300 行超。CPU 時間節約のため NONINTERACTIVE=1。
# インストール先: /home/linuxbrew/.linuxbrew
############################################################
RUN /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)" && \
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> /etc/profile && \
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)" && \
    brew update --quiet

# ─ PATH を通す ─
ENV PATH="/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:$PATH"

############################################################
# 3. uv のインストール
############################################################
# uv は Poetry 互換の超高速パッケージマネージャ。
# brew 版は Python ライブラリを静的リンク済みでサイズ極小。
############################################################
RUN brew install uv && uv --version

############################################################
# 4. Python 依存のインストール
############################################################
# requirements.txt はリポジトリルートに置く想定。
# --system-site-packages を付けないことで環境隔離。
############################################################
WORKDIR /app
COPY requirements.txt .
RUN uv pip install -r requirements.txt --no-cache-dir

############################################################
# 5. アプリケーションコードのコピー
#    .dockerignore で不要なファイル (.git など) を除外してください
############################################################
COPY ./src ./src

############################################################
# 6. 環境変数と公開ポート
############################################################
ENV PYTHONUNBUFFERED=1 \
    UVICORN_PORT=8000
EXPOSE 8000

############################################################
# 7. エントリポイント
############################################################
# ここでは FastAPI + Uvicorn を想定。
# 環境変数 PORT を参照すると Cloud Run / Docker 両対応。
############################################################
CMD ["uvicorn", "src.insight_copilot.main:app", "--host", "0.0.0.0", "--port", "${PORT:-8000}", "--reload"]
```

### 6.1.1 ビルド ↔️ 起動手順

1. **Docker Desktop をインストール** (macOS は `brew install --cask docker`).
2. ターミナルでリポジトリ直下へ移動し、次を実行：
   ```bash
   # --platform linux/amd64 を付けると Apple Silicon でも互換ビルド
   docker build -f Dockerfile.dev -t insight-dev --platform linux/amd64 .
   ```
   > ✅ *Expected output*: `Successfully tagged insight-dev:latest`
3. コンテナを起動し、API にアクセス：
   ```bash
   docker run -it --rm -p 8000:8000 -e PORT=8000 insight-dev
   # もう一タブで
   curl http://localhost:8000/health
   # -> {"status":"ok"}
   ```

### 6.1.2 よくあるエラー & 対処
| 症状 | 原因 | 解決策 |
|------|------|--------|
| `module not found` | requirements.txt に無い | `uv pip install <pkg> && echo <pkg> >> requirements.txt` |
| `brew: command not found` | PATH 未反映 | `source ~/.profile` または Dockerfile の PATH 行を再確認 |
| `Address already in use` | 8000 ポート競合 | `-p 8080:8000` のようにホスト側ポートを変える |

### 6.1.3 使い分け Q&A
> **Q. GPU 付けたい時は？**  
> → `python:3.11-slim` → `nvidia/cuda:12.2.0-devel-ubuntu22.04` に変更し、後段の apt ラインに `nvidia-cuda-toolkit` を追加。
>
> **Q. 開発と本番を分けたい**  
> → 本番用 Dockerfile では `--reload` を外し `pip install -r requirements.txt --no-dev` のように dev 依存を除外。

---
これで **Docker + uv + Homebrew** の開発環境を、初心者でもコピペ→ビルド→即 API 動作確認まで進められます。## ポイント
| 目的 | 説明 |
|------|------|
| **高速ビルド** | 依存インストールを独立レイヤ化 (`requirements.txt` だけ COPY)→コード変更でもキャッシュヒット |
| **uv ベース** | `uv venv` でローカルと同じ環境構造 (`.venv` 直下) |
| **ホットリロード** | `--reload` を有効にし、`docker compose` で volume mount すると保存即反映 |
| **非 root** | セキュリティ向上 (`USER appuser`) |

> **Tip**: Cloud Run デプロイ時は `--reload` を削除して `--workers 4` 等に置き換えるとスループットが向上

