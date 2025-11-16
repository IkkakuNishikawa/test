# 外部設計書（ドラフト）

## 1. 画面一覧と役割
- **ダッシュボード**: 進捗グラフ、次の課題、通知を集約。
- **コース一覧**: レベル・タグでフィルタ、課題難易度を視覚表示。
- **IDE 画面**:
  - 左：ファイルツリー、依存パッケージ表示。
  - 中央：エディタタブ（React/Node/テストコード）。
  - 右：プレビュー（SPA）および API レスポンスビュー。
  - 下：ターミナル（npm install / test / lint 実行ログ）。
- **レビュー画面**: 講師コメント、差分ビュー、再提出ボタン。
- **設定画面**: プロファイル、通知設定、キーバインド変更。
- **AI アシスタントパネル**: IDE サイドバーで LLM への質問、コード補完ヒント、過去質問履歴を表示。
- **学習管理 (LMS) 画面**: クラス／受講者一覧、出席・課題提出状況、レポート出力、保護者共有リンクの管理。

## 2. 業務フロー（学習者）
1. ログイン（SSO/OAuth）→初回ガイダンス。
2. コース／課題選択→要件・成果物テンプレートを確認。
3. IDE でコード編集→プレビュー検証→自動テスト実行。
4. 完了後に提出→講師レビュー待ち→通知で結果確認。
5. フィードバックを踏まえ再提出 or 次課題へ。

## 3. 主要ユースケース
- UC-01: 課題の開始とスタータープロジェクト展開。
- UC-02: Web IDE での React/Node 編集・プレビュー。
- UC-03: 提出・講師レビュー・コメント確認。
- UC-04: 学習分析ダッシュボード閲覧。
- UC-05: 講師による課題作成・公開とレビューフロー。
- UC-06: IDE 内 AI アシスタントへの質問と回答フィードバック。
- UC-07: 学習管理者によるクラス編成・進捗ロック/解放、レポート出力。

## 4. データ項目（抜粋）
| エンティティ | 主な属性 |
| --- | --- |
| User | id, role, name, avatarUrl, providerId, badges[], progress |
| Course | id, title, description, difficulty, modules[], tags[] |
| Assignment | id, courseId, title, instructions, starterRepo, dueDate |
| Submission | id, assignmentId, userId, codeSnapshot, status, score, feedback |
| SandboxSession | id, submissionId, nodeVersion, reactVersion, resources, logs[] |
| Notification | id, userId, type, message, readFlag |
| QuestionThread | id, userId, assignmentId, prompt, response, rating, visibility |
| Classroom | id, title, instructorId, learnerIds[], schedule, reports[] |

## 5. 外部インターフェース
- **認証**: 学校・塾の OIDC プロバイダ、社内 OAuth クライアント。
- **通知**: SMTP / SendGrid、Webhook（Slack 等）。
- **ストレージ**: S3 互換オブジェクトストア（コードスナップショット・ログ保持）。
- **モニタリング**: Datadog / CloudWatch API でメトリクス収集。

## 6. アーキテクチャ概要
- フロントエンド: React 18 + Vite、SPA として配信。
- バックエンド: Node.js 20 + Express + GraphQL/REST ハイブリッド（v1 では REST）。LLM アダプタサービスを内包し、外部 LLM API へ Server-to-Server で接続。
- サンドボックス: WebContainer を IDE に組み込み。ネイティブ依存課題のみクラウド実行（フォールバック API）を提供。
- データベース: PostgreSQL（学習データ）、Redis（セッション、ジョブキュー）。
- インフラ: Kubernetes 上で API/IDE/サンドボックスをマイクロサービスとして展開。
- AI 接続: OpenAI / Azure OpenAI / Anthropic 等の API を HTTPS で呼び出し、プロンプト・レスポンスを監査テーブルに保存。

## 7. 権限設計（外部仕様）
- 学習者: 自身の課題閲覧・編集、提出、レビュー閲覧。
- 講師: 受講生提出物閲覧・コメント、課題作成、バッジ付与。
- 管理者: コース公開設定、ロール管理、外部接続設定。

## 8. エラーハンドリング・通知
- 重要イベント（レビュー完了、期限切れ）はアプリ内通知＋メール。
- サンドボックス失敗時は IDE 内でエラー概要とログ取得リンクを提示。
