# 外部設計書（ドラフト）

## 1. 画面一覧と役割
- **ダッシュボード**: 進捗グラフ、次の課題、通知を集約。
- **コース一覧**: レベル・タグでフィルタ、課題難易度を視覚表示。
- **IDE 画面**:
  - 左：ファイルツリー、依存パッケージ表示。
  - 中央：エディタタブ（React/Node/テストコード）。
  - 右：プレビュー（SPA）および API レスポンスビュー。
  - 下：ターミナル（npm install / test / lint 実行ログ）。
- **実行確認画面**: WebContainer 内で起動したフロントエンド/バックエンドをマルチウィンドウで表示し、リクエスト・レスポンスログ、HTTP ヘッダ、レスポンスタイム、デバイスプレビュー（PC/タブレット/スマホ）を切り替え可能。API エンドポイントを指定して手動テストできるパネルも配置。
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
- UC-08: 実行確認画面でフロント/バックエンドの動作を検証し、API リクエスト履歴やデバイス別 UI を確認。

## 4. LLM アシスタント UX フロー
1. **質問作成**  
   - IDE の AI パネルから起動。ユーザーはテキスト、選択したコードスニペット、課題IDを添付できる。  
   - 入力検証（最大文字数、禁止語チェック）後、`draft` → `submitted` へ遷移。
2. **AI 推論**  
   - バックエンドがコンテキスト（課題仕様、直近コミット）をまとめ、LLM API へ送信。  
   - 応答を受け取り `answered` 状態で IDE にストリーミング表示。コードブロックとステップごとの説明を分離表示。
3. **学習者フィードバック**  
   - 👍/👎 やタグで回答評価。補足質問は同スレッドで追記し、ステータスは `follow_up`。  
   - 評価はモデル改善・講師レビューの優先度に利用。
4. **モデレーションキュー**  
   - すべての質問は教師ダッシュボードのレビュー待ち一覧に入り、`answered` → `review_pending`。  
   - 講師は内容を確認し、「公開」「修正依頼」「非公開」のいずれかを選択。  
   - 公開された回答は他の学習者と共有（`published`）、修正依頼はアシスタントに再質問（`rework`）、非公開は閲覧者を質問者本人に限定。
5. **監査ログ**  
   - プロンプト/レスポンス/評価/講師判定を監査テーブルに保存。検索UIで期間・キーワードフィルタが可能。
6. **通知**  
   - モデレーション結果が出ると学習者に通知。公開済み回答は課題 Tips として表⽰される。

### ステータス一覧
| ステータス | 説明 | 遷移先 |
| --- | --- | --- |
| `draft` | 入力中。送信前のローカル状態 | `submitted` |
| `submitted` | LLM 推論キューに投入済み | `answered`, `rejected` |
| `answered` | LLM から回答を受領 | `review_pending`, `follow_up` |
| `review_pending` | 講師による審査待ち | `published`, `rework`, `hidden` |
| `published` | 他学習者と共有可能 | `archived` |
| `rework` | 追加情報が必要。LLM に再送または学習者に再入力依頼 | `answered` |
| `hidden` | モデレーションで非公開 | `archived` |
| `archived` | 保持のみ。監査・レポート用 | - |

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
