# task-management-app-proto

タスク管理アプリの gRPC 通信仕様（Protocol Buffers 定義）を管理する独立リポジトリ。

- リポジトリ: `https://github.com/rinrinkurukuru/task-management-app-proto`
- Go モジュールパス: `github.com/rinrinkurukuru/task-management-app-proto`

API は CQRS 構成で client / command / query の 3 コンテナに分割される予定で、本リポジトリはそのコンテナ間通信のスキーマを単一の正として扱う。各サービスはこのモジュールを `go get` で取り込み、`gen/` 配下の生成済み Go コードを利用する。

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
| `Create` | タスクを新規作成する |
| `Move` | タスクを別カラム / 別ポジションへ移動する |
| `Edit` | タスクのタイトル / 説明 / 優先度を編集する |
| `Delete` | タスクを削除する |

### QueryService（読み取り）

`task/v1/query/task_query.proto`

| RPC | 説明 |
| --- | --- |
| `List` | タスク一覧を取得する（`date` 指定時はその週のタスクのみ） |
| `Get` | タスクを 1 件取得する |

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
- **query コンテナ** — `QueryService` を実装。読み取り専用の処理を担う

## コード生成（pb ファイルの作成手順）

proto ファイルを編集したら、以下の手順で Go の pb コードを再生成する。

### 前提

- `protoc` がインストールされていること（macOS の場合 `brew install protobuf`）
- Go 1.21 以上
- `$(go env GOPATH)/bin` に `PATH` が通っていること

### 1. プラグインのインストール（初回のみ）

```sh
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

`protoc-gen-go` / `protoc-gen-go-grpc` が `~/go/bin/` に配置される。`protoc` はビルド時にこれらを `PATH` から探すため、シェル設定で `export PATH="$PATH:$(go env GOPATH)/bin"` を入れておくこと。

### 2. Go モジュール初期化（初回のみ）

このリポジトリは `github.com/rinrinkurukuru/task-management-app-proto` という独立モジュール。ローカル開発時はモノレポ側で `go.work` を使って api と一緒に束ねる。

```sh
# protobuf/ 配下で実行
go mod init github.com/rinrinkurukuru/task-management-app-proto

# モノレポルートで実行（ローカル開発のみ。本番ビルドでは使わない）
go work init ./api ./protobuf
```

### 3. pb ファイル生成

protobuf ディレクトリ直下で以下を実行：

```sh
protoc \
  -I=proto \
  --go_out=. --go_opt=module=github.com/rinrinkurukuru/task-management-app-proto \
  --go-grpc_out=. --go-grpc_opt=module=github.com/rinrinkurukuru/task-management-app-proto \
  $(find proto -name "*.proto")
```

各オプションの意味：

| オプション | 役割 |
| --- | --- |
| `-I=proto` | import パスの解決基点。`task/v1/common/task.proto` という import 文を proto ディレクトリ起点で探す |
| `--go_out=.` | メッセージ用 `.pb.go` の出力先（カレント） |
| `--go-grpc_out=.` | サービス用 `_grpc.pb.go` の出力先 |
| `--go_opt=module=...` / `--go-grpc_opt=module=...` | 各 proto の `option go_package` で指定したパスから、このモジュール名プレフィックス分を取り除いた相対パスに出力する。これで `gen/task/v1/...` 配下に正しく展開される |
| `$(find proto -name "*.proto")` | サブディレクトリも含めて全 proto を対象にする（`proto/*.proto` だと再帰しないので注意） |

### 4. 依存モジュールの解決

```sh
go mod tidy
```

生成された pb コードが `google.golang.org/grpc` / `google.golang.org/protobuf` などに依存しているため、初回および依存追加時は必ず実行する。

### 5. 生成物の配置

```
protobuf/gen/task/v1/
├── common/task.pb.go               # メッセージ・enum
├── command/
│   ├── task_command.pb.go          # メッセージ
│   └── task_command_grpc.pb.go     # サービス（client / server stub）
└── query/
    ├── task_query.pb.go            # メッセージ
    └── task_query_grpc.pb.go       # サービス（client / server stub）
```

`gen/` 配下は自動生成物。手で編集しない。

## 各サービスからの利用

### 本番（Cloud Run 等のビルド環境）

各サービスの `go.mod` に依存として追加する：

```sh
# command / query / client それぞれのサービスディレクトリで実行
go get github.com/rinrinkurukuru/task-management-app-proto@v0.1.0
```

import 例：

```go
import (
    commonpb  "github.com/rinrinkurukuru/task-management-app-proto/gen/task/v1/common"
    commandpb "github.com/rinrinkurukuru/task-management-app-proto/gen/task/v1/command"
    querypb   "github.com/rinrinkurukuru/task-management-app-proto/gen/task/v1/query"
)
```

### ローカル開発

モノレポ内で `protobuf/` を編集しながら `api/` 側で即座に反映したい場合は `go.work` を使う。各サービスの `go.mod` に書かれたバージョンを上書きして、ローカルディレクトリを参照させる仕組み：

```sh
# モノレポルート
go work init ./api ./protobuf
```

これで `api` 側で import している `github.com/rinrinkurukuru/task-management-app-proto/...` は、リモートの GitHub ではなく**ローカルの `./protobuf` ディレクトリ**を参照するようになる。

ただし `go.work` は **ローカル開発専用** で、本番ビルドでは無効化する：

- Cloud Build / Cloud Run のビルドでは環境変数 `GOWORK=off` を設定
- もしくは Dockerfile で `go.work` を `COPY` しない

### バージョニング

破壊的変更を入れる際は Git タグでバージョンを切る：

```sh
# protobuf/ で実行
git tag v0.2.0
git push origin v0.2.0
```

各サービスは必要なタイミングで `go get github.com/rinrinkurukuru/task-management-app-proto@v0.2.0` で更新する。サービス間で異なるバージョンに留まってもよく、ロールアウトを段階的に行える。

## 開発ガイドライン

### 互換性

proto ファイルの変更時は以下を守る：

- フィールド番号は再利用しない（削除する場合は `reserved` を付与）
- 既存フィールドの型変更は禁止（破壊的変更）
- enum の値は途中で意味を変えない（追加は末尾に）
- 後方互換性を破る変更は新パッケージ（`v2` など）として追加する

### 命名規則

- パッケージ: `task.v1.<role>`（role は `common` / `command` / `query`）
- サービス名: 役割を表す PascalCase（例: `TaskCommandService` / `QueryService`）
- RPC 名: 動詞始まりの PascalCase（`Create`, `Move`, `List` 等）
- メッセージ名: `<RPC名>Request` / `<RPC名>Response`
- enum: `SCREAMING_SNAKE_CASE` でプレフィックスに enum 名を付ける（例: `PRIORITY_HIGH`）。0 番は `*_UNSPECIFIED` を予約する
