---
title: "Realm と SQLite: 代替データベース"
---

# Realm と SQLite: 代替データベース

Core Data 以外にも、iOS で使用できるデータベースがあります。この章では、Realm と SQLite の基本的な使用方法を学びます。

## Realm の特徴

Realm は、モバイルアプリに最適化されたオブジェクトデータベースです:

- **高速**: Core Data より高速な読み書き性能
- **シンプルな API**: 直感的で学習コストが低い
- **リアルタイム同期**: Realm Cloud で複数デバイス間の同期が可能
- **クロスプラットフォーム**: iOS、Android、Web で共通のデータモデル

## Realm のセットアップ

まず、Realm をプロジェクトに追加します:

```swift
// Package.swift または Podfile に追加
// dependencies: [
//     .package(url: "https://github.com/realm/realm-swift.git", from: "10.0.0")
// ]

import RealmSwift

// Realm の初期化
class RealmManager {
    static let shared = RealmManager()

    var realm: Realm {
        try! Realm()
    }

    private init() {
        // カスタム設定
        let config = Realm.Configuration(
            schemaVersion: 1,
            migrationBlock: { migration, oldSchemaVersion in
                if oldSchemaVersion < 1 {
                    // マイグレーション処理
                }
            }
        )

        Realm.Configuration.defaultConfiguration = config
    }
}
```

## Realm モデルの定義

Realm のオブジェクトモデルを定義します:

```swift
class RealmUser: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var name: String
    @Persisted var email: String
    @Persisted var createdAt: Date
    @Persisted var posts: List<RealmPost>

    convenience init(name: String, email: String) {
        self.init()
        self.id = UUID()
        self.name = name
        self.email = email
        self.createdAt = Date()
    }
}

class RealmPost: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var title: String
    @Persisted var content: String
    @Persisted var createdAt: Date
    @Persisted(originProperty: "posts") var owner: LinkingObjects<RealmUser>

    convenience init(title: String, content: String) {
        self.init()
        self.id = UUID()
        self.title = title
        self.content = content
        self.createdAt = Date()
    }
}
```

## CRUD 操作

Realm での基本的な CRUD 操作を実装します:

```swift
class RealmUserStore {
    private var realm: Realm {
        try! Realm()
    }

    // Create
    func create(name: String, email: String) throws -> RealmUser {
        let user = RealmUser(name: name, email: email)

        try realm.write {
            realm.add(user)
        }

        return user
    }

    // Read
    func fetchAll() -> [RealmUser] {
        Array(realm.objects(RealmUser.self).sorted(byKeyPath: "createdAt", ascending: false))
    }

    func fetch(byId id: UUID) -> RealmUser? {
        realm.object(ofType: RealmUser.self, forPrimaryKey: id)
    }

    // Update
    func update(_ user: RealmUser, name: String? = nil, email: String? = nil) throws {
        try realm.write {
            if let name = name {
                user.name = name
            }
            if let email = email {
                user.email = email
            }
        }
    }

    // Delete
    func delete(_ user: RealmUser) throws {
        try realm.write {
            realm.delete(user)
        }
    }

    // Query
    func search(nameContains query: String) -> [RealmUser] {
        Array(realm.objects(RealmUser.self).filter("name CONTAINS[cd] %@", query))
    }

    func fetchUsers(createdAfter date: Date) -> [RealmUser] {
        Array(realm.objects(RealmUser.self).filter("createdAt >= %@", date))
    }
}
```

## リアルタイム通知

Realm の変更を監視します:

```swift
class RealmObserver: ObservableObject {
    @Published var users: [RealmUser] = []

    private var notificationToken: NotificationToken?
    private let realm: Realm

    init() {
        realm = try! Realm()
        observeUsers()
    }

    deinit {
        notificationToken?.invalidate()
    }

    private func observeUsers() {
        let results = realm.objects(RealmUser.self).sorted(byKeyPath: "createdAt", ascending: false)

        notificationToken = results.observe { [weak self] changes in
            switch changes {
            case .initial(let collection):
                self?.users = Array(collection)

            case .update(let collection, _, _, _):
                self?.users = Array(collection)

            case .error(let error):
                print("Realm 監視エラー: \(error)")
            }
        }
    }
}
```

## SQLite の基本

SQLite は、軽量なリレーショナルデータベースです。iOS では C API を直接使用できます。

### SQLite ラッパーの実装

```swift
import SQLite3

class SQLiteManager {
    private var db: OpaquePointer?
    private let dbPath: String

    init(dbName: String = "app.db") {
        let fileURL = try! FileManager.default
            .url(for: .documentDirectory, in: .userDomainMask, appropriateFor: nil, create: false)
            .appendingPathComponent(dbName)

        dbPath = fileURL.path

        if sqlite3_open(dbPath, &db) != SQLITE_OK {
            print("データベースのオープンに失敗しました")
        }
    }

    deinit {
        sqlite3_close(db)
    }

    // テーブル作成
    func createTable() {
        let createTableQuery = """
        CREATE TABLE IF NOT EXISTS users (
            id TEXT PRIMARY KEY,
            name TEXT NOT NULL,
            email TEXT NOT NULL,
            created_at REAL NOT NULL
        );
        """

        if sqlite3_exec(db, createTableQuery, nil, nil, nil) != SQLITE_OK {
            print("テーブルの作成に失敗しました")
        }
    }

    // INSERT
    func insertUser(id: UUID, name: String, email: String, createdAt: Date) -> Bool {
        let insertQuery = "INSERT INTO users (id, name, email, created_at) VALUES (?, ?, ?, ?);"
        var statement: OpaquePointer?

        if sqlite3_prepare_v2(db, insertQuery, -1, &statement, nil) == SQLITE_OK {
            sqlite3_bind_text(statement, 1, id.uuidString, -1, nil)
            sqlite3_bind_text(statement, 2, name, -1, nil)
            sqlite3_bind_text(statement, 3, email, -1, nil)
            sqlite3_bind_double(statement, 4, createdAt.timeIntervalSince1970)

            if sqlite3_step(statement) == SQLITE_DONE {
                sqlite3_finalize(statement)
                return true
            }
        }

        sqlite3_finalize(statement)
        return false
    }

    // SELECT
    func fetchAllUsers() -> [(id: UUID, name: String, email: String, createdAt: Date)] {
        let query = "SELECT id, name, email, created_at FROM users ORDER BY created_at DESC;"
        var statement: OpaquePointer?
        var users: [(UUID, String, String, Date)] = []

        if sqlite3_prepare_v2(db, query, -1, &statement, nil) == SQLITE_OK {
            while sqlite3_step(statement) == SQLITE_ROW {
                let idString = String(cString: sqlite3_column_text(statement, 0))
                let name = String(cString: sqlite3_column_text(statement, 1))
                let email = String(cString: sqlite3_column_text(statement, 2))
                let timestamp = sqlite3_column_double(statement, 3)

                if let id = UUID(uuidString: idString) {
                    let user = (
                        id: id,
                        name: name,
                        email: email,
                        createdAt: Date(timeIntervalSince1970: timestamp)
                    )
                    users.append(user)
                }
            }
        }

        sqlite3_finalize(statement)
        return users
    }

    // UPDATE
    func updateUser(id: UUID, name: String?, email: String?) -> Bool {
        var setParts: [String] = []
        if name != nil { setParts.append("name = ?") }
        if email != nil { setParts.append("email = ?") }

        guard !setParts.isEmpty else { return false }

        let updateQuery = "UPDATE users SET \(setParts.joined(separator: ", ")) WHERE id = ?;"
        var statement: OpaquePointer?

        if sqlite3_prepare_v2(db, updateQuery, -1, &statement, nil) == SQLITE_OK {
            var bindIndex: Int32 = 1

            if let name = name {
                sqlite3_bind_text(statement, bindIndex, name, -1, nil)
                bindIndex += 1
            }

            if let email = email {
                sqlite3_bind_text(statement, bindIndex, email, -1, nil)
                bindIndex += 1
            }

            sqlite3_bind_text(statement, bindIndex, id.uuidString, -1, nil)

            if sqlite3_step(statement) == SQLITE_DONE {
                sqlite3_finalize(statement)
                return true
            }
        }

        sqlite3_finalize(statement)
        return false
    }

    // DELETE
    func deleteUser(id: UUID) -> Bool {
        let deleteQuery = "DELETE FROM users WHERE id = ?;"
        var statement: OpaquePointer?

        if sqlite3_prepare_v2(db, deleteQuery, -1, &statement, nil) == SQLITE_OK {
            sqlite3_bind_text(statement, 1, id.uuidString, -1, nil)

            if sqlite3_step(statement) == SQLITE_DONE {
                sqlite3_finalize(statement)
                return true
            }
        }

        sqlite3_finalize(statement)
        return false
    }
}
```

## データベースの選択ガイド

### Core Data を選ぶ場合

- Apple の標準フレームワークを使いたい
- iCloud 同期が必要
- オブジェクトグラフ管理が重要
- マイグレーション機能が充実している

### Realm を選ぶ場合

- シンプルな API が好ましい
- 高速な読み書き性能が必要
- リアルタイム同期が必要
- クロスプラットフォーム対応

### SQLite を選ぶ場合

- 細かな SQL 制御が必要
- 軽量なデータベースが必要
- カスタムクエリを多用する
- フレームワークの依存を避けたい

## パフォーマンス比較

一般的なケースでは、以下のような特性があります:

### 読み取り性能

- **Realm**: リアルタイム通知機能により、効率的な更新が可能
- **Core Data**: フェッチリクエストとキャッシュにより、良好な性能
- **SQLite**: 直接的なクエリにより、高速な読み取りが可能

### 書き込み性能

- **Realm**: バッチ書き込みが高速
- **Core Data**: バッチ操作のサポートあり
- **SQLite**: トランザクションで高速化が可能

### メモリ使用量

- **Realm**: 遅延読み込みによりメモリ効率が良い
- **Core Data**: フォールト機能で効率的
- **SQLite**: 最もメモリ効率が良い

## まとめ

この章では、Realm と SQLite について学びました:

- Realm の基本的な使用方法
- CRUD 操作の実装
- リアルタイム通知
- SQLite の基本操作
- データベースの選択ガイド

次の章では、効率的なキャッシュ戦略について学びます。

## 参考リソース

- [Realm Swift - GitHub](https://github.com/realm/realm-swift)
- [SQLite - Official Site](https://www.sqlite.org/)
- [Realm Documentation](https://www.mongodb.com/docs/atlas/device-sdks/)
