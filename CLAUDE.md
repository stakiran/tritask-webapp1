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
