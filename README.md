# FileSync.py – 双方向フォルダ／ファイル同期ツール

Unity プロジェクトなどのフォルダを **左右2か所で双方向同期** するためのシンプルな Python スクリプトです。

- フォルダ同士の同期  
- JSON（リアルタイムメモ）の自動マージ  
- 競合ファイルの退避  
- **単一ファイル同期モード**  
- **上書き前バックアップ（世代管理付き）**  

といった機能を、**標準ライブラリのみ**で実現しています。

---

## 特徴

- ✅ フォルダ同士の双方向同期（A ↔ B）
- ✅ 左右に同名ファイルを指定して **単一ファイルだけ同期** するモード
- ✅ Unity プロジェクト向けの既定除外（Library, Temp, Logs, Build, .git など）
- ✅ 競合ファイルは `<root>/.sync_conflicts/` 以下に自動退避
- ✅ Collab 系 JSON（presence / memos / locks）を **自動マージ** して両側に書き戻し
- ✅ 上書き前に `<root>/.sync_backups/` にバックアップを保存し、**最新 N 世代だけ保持**
- ✅ `--loop` + `--interval` で常駐ポーリング実行も可能
- ✅ 外部ライブラリ不要（Python 標準ライブラリのみ）

---

## 動作環境

- Python 3.8 以降（標準ライブラリのみ使用）
- Windows / macOS / Linux いずれでも動作想定

---

## 使い方（フォルダ同期）

```bash
python FileSync.py "<左rootA>" "<右rootB>" [オプション]
```

### 使用例

```bash
# Unity プロジェクトをローカルと OneDrive の間で同期
python FileSync.py "C:/Dev/MyProject" "C:/Users/You/OneDrive/Sync/MyProject"   --loop --interval 15 --verbose --collab-sync
```

---

## 使い方（単一ファイル同期モード）

左右に **ファイルパス** を渡すと、指定ファイルのみを対象にした「単一ファイル同期モード」になります。

```bash
python FileSync.py "<左のjsonファイル>" "<右のjsonファイル>" [オプション]
```

- 左右の **ファイル名（ベース名）が同じ** である必要があります  
  - 例: `CollabSyncLocal.json` ↔ `CollabSyncLocal.json`
- 内部的には「親ディレクトリ同士の同期 + `--only-files` でそのファイルのみ対象」に近い挙動です
- 状態ファイルは自動的に  
  `<右dir>/.<basename>_state.json`  
  として作成されます（例：`.CollabSyncLocal_state.json`）

### 使用例

```bash
python FileSync.py   "C:/Dev/MyProject/CollabSyncLocal.json"   "C:/Users/You/OneDrive/Sync/MyProject/CollabSyncLocal.json"   --loop --interval 5 --verbose --collab-sync
```

---

## オプション一覧

| オプション | 引数 | 説明 |
|-----------|------|------|
| `--loop` | なし | 有効にすると、指定間隔で同期を繰り返す（無指定なら1回だけ実行） |
| `--interval` | 秒 | ループ時のポーリング間隔（既定: 15 秒） |
| `--state` | パス | 同期状態の保存先ファイル or ディレクトリ。ディレクトリを指定した場合は `<dir>/.unity_sync_state.json` を使用。省略時は `<右root>/.unity_sync_state.json`（単一ファイル同期時は `<右dir>/.<basename>_state.json`）|
| `--no-delete` | なし | 削除伝播を行わない（片側で削除しても、もう片側のファイルを復活させる）|
| `--dry-run` | なし | 実際のコピー/削除を行わず、ログ出力のみ |
| `--verbose` | なし | 詳細ログを表示 |
| `--extra-exclude` | パス | 追加の除外パターン定義ファイル（JSON 配列 or 改行区切りテキスト） |
| `--only-folders` | `X,Y,...` | 対象を特定サブフォルダのみに限定（カンマ区切り、ルートからの相対パス） |
| `--only-files` | `X,Y,...` | 対象を特定ファイルに限定（カンマ区切り、相対パス or ワイルドカード） |
| `--conflict-dir` | 名前 | 競合ファイル退避フォルダ名（各 root 直下、既定: `.sync_conflicts`） |
| `--collab-sync` | なし | Collab 系 JSON スキーマが両側で更新された際、自動マージして両側へ書き戻す |
| `--backup-dir` | 名前 | 上書き前バックアップを保存するフォルダ名（各 root 直下、既定: `.sync_backups`） |
| `--keep-backups` | 数値 | 各ファイルに対して保持するバックアップ世代数（既定: `5`） |

---

## 既定除外パス（Unity 向け）

コード中の `DEFAULT_EXCLUDES` により、以下は自動的に除外されます。

- `Library`, `Temp`, `Logs`, `Obj`, `Build`, `Builds`
- `.git`, `.svn`, `.idea`, `.vs`
- `MemoryCaptures`, `UserSettings`
- `*.csproj`, `*.sln`, `*.tmp`, `*.bak`, `*.swp`
- `.DS_Store`, `Thumbs.db`
- `Recordings`, `CrashReports`

追加で除外したい場合は、`--extra-exclude` で JSON / テキストファイルを渡してください。

---

## 競合ファイルの扱い

前回スナップショットと比較して、

- 左右両方で同じパスが「更新された」と判定された場合 → **競合**

処理は以下の通りです。

1. `--collab-sync` が指定されていて、かつ `.json` ファイルで対応スキーマなら  
   → 自動マージして **両側に書き戻し**
2. それ以外の場合  
   - 更新日時が新しい側を元のパスとして採用
   - 古い側のファイルは `<root>/.sync_conflicts/...` に退避  
     - 例: `Assets/Example.txt` → `.sync_conflicts/Assets/Example.conflict-A-YYYYmmdd-HHMMSS.txt`

---

## Collab JSON の自動マージ仕様

`--collab-sync` が指定され、かつ JSON が以下のようなスキーマを持つ場合に自動マージされます：

```jsonc
{
  "presences": [ { "user": "...", "assetPath": "...", "context": "...", "heartbeat": 1759719744427 } ],
  "memos":     [ { "id": "...", "text": "...", "author": "...", "createdAt": 1759719712101, "pinned": false } ],
  "locks":     [ { "assetPath": "...", "owner": "...", "reason": "...", "createdAt": 1759719704529, "ttlMs": 0 } ],
  "updatedAt": 1759719744526
}
```

### マージルール（要約）

- **presences**
  - キー: `(user, assetPath, context)`
  - 同一キーが両側にある場合は `heartbeat` が大きい方を採用
- **memos**
  - キー: `id`
  - 同一 ID が両側にある場合は、`createdAt` が新しい方を採用
  - 出力時は `createdAt` 昇順でソート
- **locks**
  - キー: `(assetPath, owner, reason)`
  - 同一キーが両側にある場合は、`createdAt` が新しい方を採用
- **updatedAt**
  - `max(A.updatedAt, B.updatedAt)`
- 上記以外のトップレベルキーは **B 側優先で上書き**

---

## バックアップ機能の仕様

ファイルを **上書きする前** に、コピー先にすでにファイルが存在していれば、その旧バージョンをバックアップします。

- 保存先: `<root>/<backup-dir>/<元ファイルと同じ相対パスディレクトリ>/`
- ファイル名形式:  
  `basename-YYYYmmdd-HHMMSS.ext`

### 例

- 元ファイル: `Assets/Config/settings.json`
- rootB が `D:/ProjectB` の場合
- バックアップ先（既定設定）:

```text
D:/ProjectB/.sync_backups/Assets/settings-20251118-235959.json
```

### 世代管理

- 各ファイルごとに `--keep-backups` で指定した数だけ保持
- 既定値は `5`
- それを超えたバックアップファイルは **古いものから自動削除**

---

## 削除の扱い

前回存在していたファイルやフォルダが、片側からだけ消えていた場合：

- もう片側が未変更なら → 削除を伝播（両側から削除）
- もう片側が変更済みなら → 削除より変更を優先し、“復活（コピー）” させる

`--no-delete` を指定すると、削除伝播そのものをせず、常に「復活」側の挙動になります。

---

## よくある使い方パターン

### 1. Unity プロジェクトを PC ↔ クラウドで同期

```bash
python FileSync.py "C:/Dev/MyGame" "D:/CloudSync/MyGame"   --loop --interval 10 --verbose --collab-sync   --backup-dir .sync_backups --keep-backups 10
```

### 2. Collab 用 JSON だけ単一ファイル同期

```bash
python FileSync.py   "C:/Dev/MyGame/CollabSyncLocal.json"   "D:/CloudSync/MyGame/CollabSyncLocal.json"   --loop --interval 3 --verbose --collab-sync   --keep-backups 20
```

### 3. 一回だけ同期して終了（バッチ用途）

```bash
python FileSync.py "C:/Dev/MyGame" "D:/Backup/MyGame" --verbose
```

---

## 注意事項

- ファイル数が多い大規模プロジェクトでは、初回同期に時間がかかることがあります
- ネットワークドライブやクラウドストレージとの併用時は、プラットフォーム固有の制限に注意してください
- 重要データに対して導入する前に、**テスト用フォルダで挙動確認を行うことを強く推奨** します

---

## ライセンス

MIT LICENSE 
