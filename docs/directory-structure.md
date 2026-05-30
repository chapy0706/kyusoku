# directory-structure

> kyusoku のディレクトリ構成と、各層の責務を定義する。

---

## 全体構成

```
kyusoku/
├── apps/
│   └── web/                        # Next.js フロントエンド
├── workers/
│   ├── submit/                     # 送信受付 Worker
│   ├── ai-processor/               # AI処理 Worker
│   ├── daily-batch/                # 日次バッチ Worker
│   └── monthly-batch/              # 月次バッチ Worker
├── packages/
│   ├── domain/                     # ドメイン層（Worker・web 共有）
│   └── contracts/                  # 型定義・バリデーションスキーマ
├── db/
│   └── migrations/                 # D1 マイグレーションファイル
└── docs/
    ├── design-spec.md
    ├── directory-structure.md      # このファイル
    ├── glossary.md
    └── adr/
```

---

## apps/web — Next.js フロントエンド

```
apps/web/
├── src/
│   ├── app/
│   │   ├── (user)/                 # 派遣社員向け画面グループ
│   │   │   ├── page.tsx            # ログイン後TOP（マスコット + スタンプ）
│   │   │   └── report/
│   │   │       └── page.tsx        # 日報入力画面
│   │   ├── (admin)/                # 管理画面グループ
│   │   │   ├── dashboard/
│   │   │   │   └── page.tsx        # 担当者一覧・リスクサマリー
│   │   │   └── members/
│   │   │       └── [id]/
│   │   │           └── page.tsx    # 個人詳細・日報履歴
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   ├── components/
│   │   ├── ui/                     # shadcn/ui（自動生成。直接編集しない）
│   │   ├── mascot/                 # マスコットコンポーネント
│   │   │   ├── Mascot.tsx          # 状態（起きてる・寝そう・寝てる）をランダム決定
│   │   │   └── mascot.css          # SVGアニメーション
│   │   ├── stamp/                  # 気分スタンプコンポーネント
│   │   ├── report/                 # 日報フォームコンポーネント
│   │   └── admin/                  # 管理画面専用コンポーネント
│   │       ├── RiskBadge.tsx       # リスクレベル表示バッジ
│   │       ├── FlagStatus.tsx      # 確認フラグ操作UI
│   │       └── SilenceIndicator.tsx# 沈黙シグナル表示
│   ├── lib/
│   │   ├── api.ts                  # Workers API クライアント
│   │   └── auth.ts                 # 認証ユーティリティ
│   └── types/
│       └── index.ts                # フロントエンド固有の型
├── public/
└── tailwind.config.ts
```

### 設計上の注意

- `(user)` と `(admin)` はNext.jsのルートグループ。URLには影響しない
- `components/ui/` はshadcn/uiの自動生成物。直接編集せず、ラップして使う
- `mascot/` はデータを持たない。APIを呼ばない。フロントエンドで完結する（ADR-0008）

---

## workers/ — Cloudflare Workers

### workers/submit — 送信受付

```
workers/submit/
├── src/
│   ├── index.ts                    # エントリーポイント・ルーティング
│   ├── handler/
│   │   └── ReportHandler.ts        # リクエスト受付・バリデーション・即時200返却
│   └── queue/
│       └── AiJobEnqueuer.ts        # AI処理をwaitUntilに委譲
└── wrangler.toml
```

- 受け取ったリクエストをD1に保存し、即座に200を返す
- AI処理は `ctx.waitUntil` で非同期に委譲する
- このWorkerはAIを直接呼ばない

### workers/ai-processor — AI処理

```
workers/ai-processor/
├── src/
│   ├── index.ts
│   ├── usecase/
│   │   └── ProcessReportUseCase.ts # 判定→マイルド化→DB書き込みの一連を調整
│   ├── service/
│   │   ├── RiskClassifier.ts       # インターフェース定義
│   │   ├── TextSoftener.ts         # インターフェース定義
│   │   └── impl/
│   │       ├── WorkersAIRiskClassifier.ts
│   │       └── WorkersAITextSoftener.ts
│   └── repository/
│       └── ReportRepository.ts     # D1への書き込み
└── wrangler.toml
```

- `RiskClassifier` と `TextSoftener` はインターフェースとして定義する
- Workers AIの実装は `impl/` に閉じ込める。ドメインがモデル名を知らない状態を保つ
- 処理順序: 判定（原文）→ マイルド化（原文）→ D1書き込み（ADR-0002・ADR-0003）

### workers/daily-batch — 日次バッチ

```
workers/daily-batch/
├── src/
│   ├── index.ts                    # Cron Trigger エントリーポイント
│   ├── analyzer/
│   │   └── SilenceAnalyzer.ts      # 沈黙シグナルの算出（ADR-0004）
│   └── assessor/
│       └── RiskAssessment.ts       # SilenceスコアとTextスコアの合流・5段階評価算出
└── wrangler.toml
```

- AIを使わない。SQLクエリとルールのみで完結する（ADR-0004）
- 全ユーザー分を1日1回処理する。50人規模なら無料枠で収まる

### workers/monthly-batch — 月次バッチ

```
workers/monthly-batch/
├── src/
│   ├── index.ts                    # Cron Trigger エントリーポイント
│   ├── summarizer/
│   │   └── MonthlySummarizer.ts    # monthly_summaries への集計書き込み
│   └── cleaner/
│       └── RawDataCleaner.ts       # daily_reports・access_logs の削除
└── wrangler.toml
```

- 集計 → 書き込み → 削除 の順序を必ず守る（ADR-0005）
- summaryに個人の言葉（テキスト）を含めない

---

## packages/domain — ドメイン層

```
packages/domain/
├── src/
│   ├── report/
│   │   ├── Report.ts               # 日報エンティティ
│   │   ├── MoodStamp.ts            # 気分スタンプ値オブジェクト（0・1・2）
│   │   └── RiskLevel.ts            # リスクレベル値オブジェクト（0〜3）
│   ├── assessment/
│   │   ├── SilenceScore.ts         # 沈黙スコア値オブジェクト
│   │   ├── TextScore.ts            # テキストスコア値オブジェクト
│   │   └── RiskAssessmentResult.ts # 最終評価結果（1〜5）
│   └── user/
│       ├── User.ts                 # ユーザーエンティティ
│       └── Role.ts                 # ロール値オブジェクト（派遣社員・担当営業・管理者）
└── package.json
```

- 外部依存を持たない。Workers AI・D1・Next.jsのいずれも import しない
- Worker と web の両方から参照される唯一の共有層

---

## packages/contracts — 型定義・スキーマ

```
packages/contracts/
├── src/
│   ├── api/
│   │   ├── submit.ts               # 送信APIのリクエスト・レスポンス型
│   │   └── admin.ts                # 管理画面APIの型
│   └── db/
│       └── schema.ts               # D1テーブルの型定義
└── package.json
```

- APIの入出力型とDBスキーマの型を一箇所で管理する
- フロントエンドとWorkerで型が食い違わないようにするための層

---

## db/migrations — D1マイグレーション

```
db/
└── migrations/
    ├── 0001_create_users.sql
    ├── 0002_create_daily_reports.sql
    ├── 0003_create_access_logs.sql
    ├── 0004_create_risk_flags.sql
    └── 0005_create_monthly_summaries.sql
```

- マイグレーションは番号順に管理する
- 一度適用したマイグレーションファイルは書き換えない。修正は新ファイルで行う

---

## docs/

```
docs/
├── design-spec.md                  # 設計仕様書（思想・機能・データ・技術スタック）
├── directory-structure.md          # このファイル
├── glossary.md                     # 用語集
└── adr/
    ├── ADR-0001-ai-is-suggestion-not-verdict.md
    ├── ADR-0002-judge-on-raw-text.md
    ├── ADR-0003-no-softening-for-sos.md
    ├── ADR-0004-silence-detection-rule-based.md
    ├── ADR-0005-data-retention-one-month.md
    ├── ADR-0006-confirmation-flag.md
    ├── ADR-0007-access-control-by-assignment.md
    ├── ADR-0008-mascot-holds-no-data.md
    ├── ADR-0009-five-level-score-by-ai.md
    └── ADR-0010-no-push-notification-v1.md
```
