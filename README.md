# task-management-app / protobuf

タスク管理アプリの gRPC 通信仕様（Protocol Buffers 定義）を管理するリポジトリ。

API は CQRS 構成で client / command / query の 3 コンテナに分割される予定で、本リポジトリはそのコンテナ間通信のスキーマを単一の正として扱う。

## ディレクトリ構成

```
protobuf/
└── proto/
    └── task/
        └── v1/
            ├── common/
            │   └── task.proto          # 共通型（Task / Priority / Status / Column）
            ├── command/
            │   └── task_command.proto  # 書き込みサービス（Create / Move / Edit / Delete）
            └── query/
                └── task_query.proto    # 読み取りサービス（ListTasks / GetTask）
```

`v1` はスキーマのメジャーバージョンを表す。後方互換性を破る変更が必要な場合は `v2` を新設する。

## サービス概要

### TaskCommandService（書き込み）

`task/v1/command/task_command.proto`

| RPC | 説明 |
| --- | --- |
| `CreateTask` | タスクを新規作成する |
| `MoveTask` | タスクを別カラム / 別ポジションへ移動する |
| `EditTask` | タスクのタイトル / 説明 / 優先度を編集する |
| `DeleteTask` | タスクを削除する |

### TaskQueryService（読み取り）

`task/v1/query/task_query.proto`

| RPC | 説明 |
| --- | --- |
| `ListTasks` | タスク一覧を取得する（`date` 指定時はその週のタスクのみ） |
| `GetTask` | タスクを 1 件取得する |

### 共通型

`task/v1/common/task.proto` に `Task` メッセージと `Priority` / `Status` / `Column` の enum を定義し、command / query 双方から import している。CQRS 間でモデル定義が二重化しないようにするため、Task の構造を変更する場合は必ず common 側を更新する。

## コンテナ構成（予定）

```
            ┌──────────────┐
   HTTP ──▶ │   client     │ ── pb ⇄ JSON 変換、ルーティング
            └──────────────┘
                   │ gRPC
        ┌──────────┴──────────┐
        ▼                     ▼
 ┌──────────────┐      ┌──────────────┐
 │   command    │      │    query     │
 │  (書き込み)  │      │   (読み取り) │
 └──────────────┘      └──────────────┘
```

- **client コンテナ** — API の入り口。HTTP リクエストを受け、command / query への gRPC 呼び出しにルーティングし、レスポンスを JSON へ変換する
- **command コンテナ** — `TaskCommandService` を実装。書き込み系の処理を担う
- **query コンテナ** — `TaskQueryService` を実装。読み取り専用の処理を担う

## 開発ガイドライン

### 互換性

proto ファイルの変更時は以下を守る：

- フィールド番号は再利用しない（削除する場合は `reserved` を付与）
- 既存フィールドの型変更は禁止（破壊的変更）
- enum の値は途中で意味を変えない（追加は末尾に）
- 後方互換性を破る変更は新パッケージ（`v2` など）として追加する

### 命名規則

- パッケージ: `task.v1.<role>`（role は `common` / `command` / `query`）
- サービス名: `<Domain><Role>Service`（例: `TaskCommandService`）
- RPC 名: 動詞始まりの PascalCase（`CreateTask`, `ListTasks` 等）
- メッセージ名: `<RPC名>Request` / `<RPC名>Response`
- enum: `SCREAMING_SNAKE_CASE` でプレフィックスに enum 名を付ける（例: `PRIORITY_HIGH`）。0 番は `*_UNSPECIFIED` を予約する
