# Tritask Web 仕様書

## 概要

Tritask Web は、テキストベースのタスク管理ツール [Tritask](https://github.com/tritask/tritask-vscode) のウェブアプリ版である。ブラウザ上でキーボードショートカットを使い、1行1タスクのテキストをサクサク操作できる。

- 公開URL: https://stakiran.github.io/tritask-webapp1/
- リポジトリ: https://github.com/stakiran/tritask-webapp1

## アーキテクチャ

### 構成

```
GitHub Pages（静的ホスティング）
  └── index.html（単一ファイル、フロントエンドで完結）
        ├── 認証 → Firebase Authentication（Google ログイン）
        └── データ保存 → Cloud Firestore（ユーザーごとに1ドキュメント）
```

- バックエンドサーバーは不要。Firebase の BaaS（Backend as a Service）を利用。
- フロントエンドは `index.html` 1ファイルのみ。ビルド不要。
- Firebase SDK は CDN（gstatic.com）から ESModules で読み込み。

### 技術スタック

| 項目 | 技術 |
|------|------|
| ホスティング | GitHub Pages |
| フロントエンド | HTML + CSS + vanilla JavaScript（フレームワークなし） |
| 認証 | Firebase Authentication（Google プロバイダ） |
| データベース | Cloud Firestore |
| Firebase SDK | v11.6.0（ESModules、CDN） |
| フォント | JetBrains Mono（エディタ）、Noto Sans JP（UI） |

## Firebase プロジェクト

### 基本情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | tritask-web |
| プロジェクトID | tritask-web |
| プロジェクト番号 | 610643806865 |
| プラン | Spark（無料） |

### Firebase Config

```javascript
const firebaseConfig = {
  apiKey: "AIzaSyA4J10WimhpWtfdwqJHBDxNmPwF4WVJnPg",
  authDomain: "tritask-web.firebaseapp.com",
  projectId: "tritask-web",
  storageBucket: "tritask-web.firebasestorage.app",
  messagingSenderId: "610643806865",
  appId: "1:610643806865:web:1b6816ad4ee5b6b93c0d30",
  measurementId: "G-NRSFKX34R0"
};
```

### Authentication 設定

- ログインプロバイダ: Google（有効）
- 承認済みドメイン:
  - localhost（Default）
  - tritask-web.firebaseapp.com（Default）
  - tritask-web.web.app（Default）
  - stakiran.github.io（Custom）

### Firestore 設定

- エディション: Standard
- ロケーション: asia-northeast1（東京）
- データベースID: (default)

### Firestore データ構造

```
コレクション: users
  └── ドキュメントID: {Firebase Auth の uid}
        ├── content: string    ← .trita ファイルの中身（全テキスト）
        └── updatedAt: timestamp
```

- ユーザー1人につき1ドキュメント。
- content フィールドに .trita ファイル相当のテキストを丸ごと保存。
- Firestore の1ドキュメント上限は 1MB なので、テキストベースのタスクデータには十分。

### Firestore セキュリティルール

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

- ログイン済みユーザーが自分のドキュメントだけ読み書きできる。
- 他人のデータにはアクセス不可。

## Tritask フォーマット

### タスク行の形式

```
<ソートキー> <日付> <曜日> [開始時刻] [終了時刻] <タスク名> [属性...]
```

例:

```
2 2026/03/17 Mon            新しいタスク
2 2026/03/17 Mon 09:00      作業中のタスク
1 2026/03/17 Mon 09:00 09:45 完了したタスク
```

### ソートキーの意味

| キー | 意味 |
|------|------|
| 1 | 今日の完了タスク（TODAY DONE） |
| 2 | 今日のTODO（TODAY TODO） |
| 3 | 明日以降のTODO（TOMORROW TODO） |
| 4 | 昨日以前の完了タスク（YESTERDAY DONE） |

### 区切り行

ソート時に自動生成される区切り:

```
                              ---- INBOX
1 2026/03/17 Mon 00:00 00:00  ---- TODAY DONE hold:0
2 2026/03/17 Mon              ---- TODAY TODO hold:0
3 2026/03/18 Tue              ---- TOMORROW TODO hold:1
4 2026/03/16 Sun 00:00 00:00  ---- YESTERDAY DONE
```

## 機能一覧

### ショートカットキー

| ショートカット | 機能 | 説明 |
|----------------|------|------|
| `Ctrl+Enter` | タスク追加 | カーソル行の下に、今日の日付で新しいタスクを挿入 |
| `Ctrl+S` | タスク開始 | カーソル行のタスクに現在時刻を開始時刻として記録 |
| `Ctrl+E` | タスク終了 | カーソル行のタスクに現在時刻を終了時刻として記録し、ソートキーを1（完了）に変更 |
| `Ctrl+D` | 日付変更 | ダイアログを開き、日数指定（+1=明日、-1=昨日）または日付直接入力で変更 |
| `Ctrl+Shift+S` | ソート | 全タスクを日付・状態に基づいて自動分類・並べ替え |
| `Ctrl+Shift+D` | タスク複製 | カーソル行のタスクを今日の日付でコピー |
| `Ctrl+Shift+X` | 行削除 | カーソル行を削除 |
| `Ctrl+↑` | 行を上に移動 | カーソル行を1行上に移動 |
| `Ctrl+↓` | 行を下に移動 | カーソル行を1行下に移動 |

### その他の機能

- **自動保存**: テキストが変更されるたびに 500ms の debounce 後、Firestore に保存
- **エクスポート**: Export ボタンで `.trita` ファイルとしてダウンロード
- **ステータスバー**: 現在行、タスク数、今日のタスク数、完了数をリアルタイム表示
- **新規ユーザー**: 初回ログイン時にデフォルトの区切り付きテンプレートを自動生成

## 保存の仕組み

- テキストエリアの `input` イベントおよびショートカット操作のたびに保存をスケジュール。
- 500ms の debounce で連続操作時の書き込み回数を抑制。
- Firestore の `setDoc` で `content` フィールドを丸ごと上書き。
- Spark プラン（無料）の書き込み上限は 1日2万回。debounce により通常利用で問題なし。

## 無料枠について

Firebase Spark プラン（無料、クレジットカード登録不要）:

| リソース | 上限 |
|----------|------|
| Authentication | 50,000 MAU |
| Firestore 書き込み | 20,000 回/日 |
| Firestore 読み取り | 50,000 回/日 |
| Firestore ストレージ | 1 GiB |

テキストベースのタスク管理という性質上、数百〜数千人規模でも無料枠内に収まる見込み。

## デプロイ手順

1. GitHub に `tritask-webapp1` リポジトリを作成
2. `index.html` をリポジトリにプッシュ
3. Settings → Pages でソースを設定し公開
4. Firebase コンソール → Authentication → 設定 → 承認済みドメインに `stakiran.github.io` を追加
