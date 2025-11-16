# API 設計書（ドラフト）

## 1. 共通仕様
- ベース URL: `https://api.example.com/api/v1`
- 認証: `Authorization: Bearer <JWT>`（OAuth/OIDC で取得）
- コンテンツタイプ: `application/json`（一部ログ取得系は `text/plain` を許容）
- エラーフォーマット:
  ```json
  {
    "error": {
      "code": "SUBMISSION_NOT_FOUND",
      "message": "対象の提出が存在しません",
      "details": {}
    }
  }
  ```
- レートリミット: 学習者 60 req/min、講師 120 req/min、管理者 200 req/min。

## 2. 認証・ユーザー系
| メソッド | パス | 概要 | 主要レスポンス |
| --- | --- | --- | --- |
| POST | `/auth/login` | OAuth コードを受け取り JWT を発行 | `{ accessToken, refreshToken, user }` |
| POST | `/auth/refresh` | リフレッシュトークンで再発行 | `{ accessToken, refreshToken }` |
| GET | `/users/me` | 自分のプロフィール取得 | `User` |
| PATCH | `/users/me` | プロフィール編集（表示名、アバターなど） | `User` |

## 3. コース・課題系
| メソッド | パス | 概要 | 主要レスポンス |
| --- | --- | --- | --- |
| GET | `/courses` | レベル・タグフィルタを含むコース一覧 | `Course[]` |
| POST | `/courses` | 新規コース作成（講師/管理者） | `Course` |
| GET | `/courses/:courseId/assignments` | コース内課題一覧 | `Assignment[]` |
| GET | `/assignments/:id` | 課題詳細・スターターリポジトリ情報 | `Assignment` |
| POST | `/assignments` | 課題作成（講師） | `Assignment` |

## 4. 提出・レビュー系
| メソッド | パス | 概要 | 主要レスポンス |
| --- | --- | --- | --- |
| POST | `/submissions` | 課題提出（コードスナップショット、メタデータ） | `Submission` |
| GET | `/submissions/:id` | 提出詳細＋フィードバック | `Submission` |
| PATCH | `/submissions/:id/status` | 再提出、ステータス更新 | `Submission` |
| POST | `/reviews` | 講師レビュー登録（スコア、コメント） | `Review` |
| GET | `/reviews/:submissionId` | 指定提出のレビュー一覧 | `Review[]` |

## 5. サンドボックス・実行系
| メソッド | パス | 概要 | 主要レスポンス |
| --- | --- | --- | --- |
| POST | `/sandbox/session` | サンドボックス開始（Node+React ビルド） | `{ sessionId, status, expiration }` |
| GET | `/sandbox/session/:id` | セッション状態確認 | `{ sessionId, status, previewUrl }` |
| GET | `/sandbox/session/:id/logs` | 実行／ビルドログ取得 | `text/plain` or `{ logs[] }` |
| DELETE | `/sandbox/session/:id` | セッション終了・リソース解放 | `{ sessionId, status: "TERMINATED" }` |

## 6. 通知・分析系
| メソッド | パス | 概要 | 主要レスポンス |
| --- | --- | --- | --- |
| GET | `/notifications` | 通知一覧 | `Notification[]` |
| PATCH | `/notifications/:id/read` | 既読処理 | `{ id, readFlag: true }` |
| GET | `/progress/summary` | 学習者の進捗サマリ | `{ completedAssignments, badges, weeklyStats }` |

## 7. LLM アシスタント系
| メソッド | パス | 概要 | 主要レスポンス |
| --- | --- | --- | --- |
| POST | `/assistants/questions` | IDE から LLM へ質問を送信し回答を生成 | `QuestionThread` |
| GET | `/assistants/questions` | 自分の質問履歴・ステータス取得 | `QuestionThread[]` |
| POST | `/assistants/questions/:id/feedback` | 回答に対する評価・タグ付け | `{ id, rating, tags[] }` |
| GET | `/assistants/moderation` (講師) | 学習者の質問ログをレビュー | `QuestionThread[]` |

## 8. 学習管理 (LMS) 系
| メソッド | パス | 概要 | 主要レスポンス |
| --- | --- | --- | --- |
| GET | `/classes` | 担当クラス／受講グループ一覧 | `Classroom[]` |
| POST | `/classes` | 新規クラス作成・受講者割当 | `Classroom` |
| PATCH | `/classes/:id` | クラス情報更新（スケジュール、担当） | `Classroom` |
| GET | `/classes/:id/progress` | クラス全体の進捗・提出状況 | `{ learners: ProgressSummary[] }` |
| POST | `/classes/:id/lock` | 課題進行のロック/解放、出席管理 | `{ classId, locked: boolean }` |
| GET | `/classes/:id/reports` | レポート生成・保護者共有リンク取得 | `Report[]` |

## 9. データモデル（抜粋）
```json
User {
  id: string,
  role: "learner" | "coach" | "admin",
  name: string,
  avatarUrl?: string,
  badges: string[],
  progress: {
    completedAssignments: number,
    currentCourseId?: string
  }
}
```
```json
Assignment {
  id: string,
  courseId: string,
  title: string,
  difficulty: "advanced" | "expert",
  instructions: string,
  starterRepo: string,
  dueDate?: string
}
```
```json
Submission {
  id: string,
  assignmentId: string,
  userId: string,
  codeSnapshot: {
    repoUrl: string,
    commitHash: string,
    files: FileSummary[]
  },
  status: "draft" | "submitted" | "reviewed" | "resubmission",
  score?: number,
  feedback?: string
}
```
```json
QuestionThread {
  id: string,
  userId: string,
  assignmentId?: string,
  prompt: string,
  response: string,
  model: string,
  rating?: number,
  tags?: string[],
  visibility: "private" | "shared",
  createdAt: string
}
```
```json
Classroom {
  id: string,
  title: string,
  instructorId: string,
  learnerIds: string[],
  schedule: {
    startDate: string,
    endDate?: string,
    sessions: SessionSlot[]
  },
  reports: ReportSummary[]
}
```

## 10. エラーコード例
| コード | HTTP | 意味 |
| --- | --- | --- |
| `AUTH_INVALID_CODE` | 401 | OAuth コードが不正 |
| `COURSE_NOT_FOUND` | 404 | コースが存在しない |
| `SUBMISSION_LOCKED` | 409 | レビュー中のため更新不可 |
| `SANDBOX_TIMEOUT` | 504 | 実行タイムアウト |
| `ASSISTANT_RATE_LIMIT` | 429 | LLM 質問のレート制限超過 |
| `CLASSROOM_LOCKED` | 423 | クラスがロック状態で操作不可 |

## 11. サンドボックス制約
- 各セッションに CPU 1core / RAM 2GB / ディスク 2GB を割り当て。
- 実行タイムアウト 90 秒、プレビュー URL は署名付きで 5 分有効。
- アウトバウンド通信は npm registry / Git リポジトリのみ許可（将来拡張可）。

## 12. 今後の拡張候補
- GraphQL API の並行提供（IDE からの型安全な取得）。
- メトリクス配信用の Webhook / SSE。
- LTI / Classroom 連携用 API セット。
- AI モデル選択 API、回答テンプレートのカスタマイズ、保護者ポータル向けレポート API。
