---
title: "Chapter 10: スキーマ進化とバージョニング - 長期運用を見据えた設計"
---

# Chapter 10: スキーマ進化とバージョニング

## イントロダクション

データベーススキーマは、アプリケーションの成長とともに進化し続けます。新機能の追加、パフォーマンス改善、ビジネス要件の変更により、スキーマは常に変化します。この変化を適切に管理しなければ、データの整合性が失われ、システムが不安定になります。

本章では、スキーマの進化を管理するための原則、高度なマイグレーション手法、複数のマイグレーションツールの比較、ブルーグリーンデプロイメント、そして本番環境での安全なマイグレーション戦略を詳しく解説します。

### 本章で学ぶこと

- スキーマ進化の原則と設計パターン
- Alembic、Flyway、Liquibaseの実践
- ゼロダウンタイムマイグレーションの高度な手法
- ロールバック戦略とデータマイグレーション
- ブルーグリーンデプロイメント
- 災害復旧計画とバックアップ戦略

---

## スキーマ進化の原則

### 後方互換性の維持

スキーマ変更は、既存のアプリケーションコードが動作し続けるように設計する必要があります。

#### 原則1: 段階的な変更

```sql
-- ❌ 後方互換性なし（古いコードが動かなくなる）
-- Phase 1: カラム名変更
ALTER TABLE users RENAME COLUMN email TO email_address;
-- 古いコードで user.email を参照しているとエラー

-- ✅ 後方互換性あり（段階的移行）
-- Phase 1: 新しいカラムを追加
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);

-- Phase 2: 既存データをコピー
UPDATE users SET email_address = email WHERE email_address IS NULL;

-- Phase 3: アプリケーションコードを更新（両方のカラムに書き込み）
-- アプリケーションコードで両方のカラムを同期

-- Phase 4: アプリケーションコードを更新（新カラムから読み込み）
-- フォールバック付きで新カラムを優先

-- Phase 5: NOT NULL制約を追加
ALTER TABLE users ALTER COLUMN email_address SET NOT NULL;

-- Phase 6: 古いカラムを削除（十分な移行期間後、例: 2週間）
ALTER TABLE users DROP COLUMN email;
```

#### 原則2: 前方互換性の考慮

```sql
-- ✅ 新機能追加は NULL 許可で開始
ALTER TABLE products ADD COLUMN video_url VARCHAR(500);
-- 既存レコードは NULL、新しいレコードのみ値を設定

-- 後で必須にする場合
UPDATE products SET video_url = 'https://example.com/default.mp4'
WHERE video_url IS NULL;

ALTER TABLE products ALTER COLUMN video_url SET NOT NULL;
```

#### 原則3: スモールステップの原則

```sql
-- ❌ 大きな変更を一度に実行（リスク高）
BEGIN;
ALTER TABLE users ADD COLUMN full_name VARCHAR(100);
UPDATE users SET full_name = CONCAT(first_name, ' ', last_name);
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
COMMIT;
-- 失敗時のロールバックが複雑

-- ✅ 小さなステップに分割（各ステップを検証）
-- Migration 1: カラム追加
ALTER TABLE users ADD COLUMN full_name VARCHAR(100);

-- Migration 2: データ移行
UPDATE users SET full_name = CONCAT(first_name, ' ', last_name)
WHERE full_name IS NULL;

-- Migration 3: 制約追加
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;

-- Migration 4: 古いカラム削除（十分な期間後）
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

### スキーマバージョニング戦略

#### 戦略1: セマンティックバージョニング

```
V1.0.0__Initial_schema.sql
V1.1.0__Add_users_table.sql
V1.2.0__Add_posts_table.sql
V2.0.0__Breaking_change_rename_email.sql  # メジャーバージョンアップ
V2.1.0__Add_user_profiles.sql
```

- **メジャー (X.0.0)**: 後方互換性のない変更
- **マイナー (1.X.0)**: 後方互換性のある機能追加
- **パッチ (1.0.X)**: バグ修正、インデックス追加

#### 戦略2: タイムスタンプバージョニング

```
20260103120000_initial_schema.sql
20260103121500_add_users_table.sql
20260103130000_add_posts_table.sql
20260105140000_add_indexes.sql
```

- タイムスタンプで自動的に順序が決まる
- ブランチマージ時の競合が少ない
- Prisma、TypeORM、Knex.jsが採用

#### 戦略3: ブランチバージョニング（Alembic）

```bash
# ブランチ作成
alembic revision -m "feature A" --head=base
alembic revision -m "feature B" --head=base

# ブランチマージ
alembic merge -m "merge feature A and B" featureA featureB
```

- 並行開発をサポート
- 複数ブランチのマイグレーションを統合可能

---

## Alembic完全ガイド

Alembicは、SQLAlchemyのためのデータベースマイグレーションツールです。Pythonエコシステムで最も人気があり、強力な機能を持っています。

### セットアップ

```bash
# インストール
pip install alembic psycopg2-binary sqlalchemy

# 初期化
alembic init alembic
```

生成されるディレクトリ構造：

```
project/
├── alembic/
│   ├── versions/          # マイグレーションファイル
│   ├── env.py            # 環境設定
│   └── script.py.mako    # マイグレーションテンプレート
├── alembic.ini           # Alembic設定ファイル
└── models.py             # SQLAlchemyモデル定義
```

### 設定

```ini
# alembic.ini
[alembic]
script_location = alembic
sqlalchemy.url = postgresql://user:password@localhost/dbname

# マイグレーションファイル命名規則
file_template = %%(year)d%%(month).2d%%(day).2d_%%(hour).2d%%(minute).2d_%%(rev)s_%%(slug)s

# ログレベル
[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
import os
import sys

# プロジェクトのルートディレクトリをパスに追加
sys.path.insert(0, os.path.realpath(os.path.join(os.path.dirname(__file__), '..')))

# モデル定義をインポート
from models import Base

# Alembic Config オブジェクト
config = context.config

# 環境変数から DATABASE_URL を読み込み
config.set_main_option(
    'sqlalchemy.url',
    os.environ.get('DATABASE_URL', 'postgresql://localhost/dbname')
)

# ロギング設定
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# メタデータ
target_metadata = Base.metadata

def run_migrations_offline() -> None:
    """オフラインモード（SQLファイル生成）"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online() -> None:
    """オンラインモード（データベース直接実行）"""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            # トランザクションごとにマイグレーション実行
            transaction_per_migration=True,
            # PostgreSQL用の検索パス設定
            include_schemas=True,
            # 複数スキーマのサポート
            version_table_schema=target_metadata.schema,
        )

        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### SQLAlchemyモデル定義

```python
# models.py
from sqlalchemy import Column, Integer, String, Text, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship, declarative_base
from sqlalchemy.sql import func

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False, index=True)
    username = Column(String(50), nullable=False)
    password = Column(String(255), nullable=False)
    status = Column(String(20), default='active', nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    # リレーション
    profile = relationship('Profile', back_populates='user', uselist=False, cascade='all, delete-orphan')
    posts = relationship('Post', back_populates='author', cascade='all, delete-orphan')

class Profile(Base):
    __tablename__ = 'profiles'

    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id', ondelete='CASCADE'), unique=True, nullable=False)
    bio = Column(Text)
    avatar = Column(String(500))
    website = Column(String(500))
    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    # リレーション
    user = relationship('User', back_populates='profile')

class Post(Base):
    __tablename__ = 'posts'

    id = Column(Integer, primary_key=True)
    title = Column(String(255), nullable=False)
    content = Column(Text)
    published = Column(Boolean, default=False, nullable=False)
    published_at = Column(DateTime(timezone=True))
    author_id = Column(Integer, ForeignKey('users.id', ondelete='CASCADE'), nullable=False, index=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    # リレーション
    author = relationship('User', back_populates='posts')
```

### マイグレーション作成

```bash
# 自動生成（モデル定義から差分検出）
alembic revision --autogenerate -m "add user profile table"

# 空のマイグレーション作成
alembic revision -m "custom migration"

# ブランチマージ
alembic merge -m "merge branches" head1 head2
```

### マイグレーションファイル

```python
# alembic/versions/20260103_1200_abc123_add_user_profile.py
"""add user profile table

Revision ID: abc123
Revises: xyz789
Create Date: 2026-01-03 12:00:00.000000
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers
revision = 'abc123'
down_revision = 'xyz789'
branch_labels = None
depends_on = None

def upgrade() -> None:
    """アップグレード処理"""
    # テーブル作成
    op.create_table(
        'profiles',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('user_id', sa.Integer(), nullable=False),
        sa.Column('bio', sa.Text(), nullable=True),
        sa.Column('avatar', sa.String(length=500), nullable=True),
        sa.Column('website', sa.String(length=500), nullable=True),
        sa.Column('created_at', sa.DateTime(timezone=True),
                  server_default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(timezone=True),
                  server_default=sa.text('now()'), nullable=False),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'],
                                ondelete='CASCADE'),
    )

    # インデックス作成
    op.create_index(
        'ix_profiles_user_id',
        'profiles',
        ['user_id'],
        unique=True
    )

    # ENUM型作成（PostgreSQL）
    status_enum = postgresql.ENUM(
        'active', 'inactive', 'suspended',
        name='user_status'
    )
    status_enum.create(op.get_bind())

    # カラム追加
    op.add_column(
        'users',
        sa.Column('status', status_enum, nullable=False,
                  server_default='active')
    )

def downgrade() -> None:
    """ダウングレード処理"""
    # カラム削除
    op.drop_column('users', 'status')

    # ENUM型削除
    op.execute('DROP TYPE user_status')

    # インデックス削除
    op.drop_index('ix_profiles_user_id', table_name='profiles')

    # テーブル削除
    op.drop_table('profiles')
```

### データマイグレーション

```python
# alembic/versions/20260103_1300_def456_migrate_user_data.py
"""migrate user data to full name

Revision ID: def456
Revises: abc123
Create Date: 2026-01-03 13:00:00.000000
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.orm import Session

revision = 'def456'
down_revision = 'abc123'
branch_labels = None
depends_on = None

def upgrade() -> None:
    # 新しいカラムを追加
    op.add_column('users', sa.Column('full_name', sa.String(length=200), nullable=True))

    # バッチ処理で既存データを変換
    connection = op.get_bind()
    session = Session(bind=connection)

    batch_size = 1000
    offset = 0

    while True:
        # バッチでデータ取得
        result = session.execute(
            sa.text("""
                SELECT id, first_name, last_name
                FROM users
                WHERE full_name IS NULL
                LIMIT :limit OFFSET :offset
            """),
            {'limit': batch_size, 'offset': offset}
        )

        users = result.fetchall()
        if not users:
            break

        # バッチで更新
        for user in users:
            full_name = f"{user.first_name} {user.last_name}".strip()
            session.execute(
                sa.text("""
                    UPDATE users
                    SET full_name = :full_name
                    WHERE id = :user_id
                """),
                {'full_name': full_name, 'user_id': user.id}
            )

        session.commit()
        offset += batch_size
        print(f"Processed {offset} users")

    # NOT NULL制約を追加
    op.alter_column('users', 'full_name', nullable=False)

    # 古いカラムを削除
    op.drop_column('users', 'first_name')
    op.drop_column('users', 'last_name')

def downgrade() -> None:
    # 古いカラムを復元
    op.add_column('users', sa.Column('first_name', sa.String(length=50), nullable=True))
    op.add_column('users', sa.Column('last_name', sa.String(length=50), nullable=True))

    # full_nameから first_name と last_name を復元
    connection = op.get_bind()
    connection.execute(
        sa.text("""
            UPDATE users
            SET
                first_name = SPLIT_PART(full_name, ' ', 1),
                last_name = SPLIT_PART(full_name, ' ', 2)
        """)
    )

    # full_nameを削除
    op.drop_column('users', 'full_name')
```

### マイグレーション実行

```bash
# 最新までマイグレーション
alembic upgrade head

# 特定のリビジョンまでマイグレーション
alembic upgrade abc123

# 1ステップアップグレード
alembic upgrade +1

# 2ステップアップグレード
alembic upgrade +2

# ダウングレード（1ステップ）
alembic downgrade -1

# 特定のリビジョンまでダウングレード
alembic downgrade xyz789

# 完全にダウングレード
alembic downgrade base

# 現在のバージョン確認
alembic current

# マイグレーション履歴（詳細）
alembic history --verbose

# マイグレーション履歴（範囲指定）
alembic history -r base:head

# SQL生成（実行なし）
alembic upgrade head --sql

# ドライラン（実行せずに表示）
alembic upgrade head --sql > migration.sql
```

---

## Flyway完全ガイド

Flywayは、Javaエコシステムのデファクトスタンダードなマイグレーションツールです。シンプルで強力、SQLベースのマイグレーションが特徴です。

### セットアップ

```bash
# Flywayダウンロード
wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.4.1/flyway-commandline-10.4.1-linux-x64.tar.gz | tar xvz

# またはDockerで実行
docker run --rm \
  -v $(pwd)/sql:/flyway/sql \
  flyway/flyway:10.4.1 \
  -url=jdbc:postgresql://host.docker.internal:5432/dbname \
  -user=postgres \
  -password=password \
  migrate
```

### 設定

```properties
# flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=postgres
flyway.password=password
flyway.schemas=public
flyway.locations=filesystem:sql
flyway.table=flyway_schema_history
flyway.baselineOnMigrate=true
flyway.validateOnMigrate=true
flyway.cleanDisabled=true  # 本番では必須（データ削除防止）
flyway.outOfOrder=false     # 順序外のマイグレーションを許可しない
flyway.placeholderReplacement=true
flyway.placeholders.environment=production
```

### マイグレーションファイル命名規則

Flywayは厳格な命名規則を持ちます：

```
V<バージョン>__<説明>.sql
R__<説明>.sql  # Repeatable（常に再実行）

例：
V1__Initial_schema.sql
V1.1__Add_users_table.sql
V2__Add_posts_table.sql
V2.1__Add_posts_index.sql
V3__Migrate_user_data.sql
R__Create_view_user_stats.sql  # Repeatable
```

### マイグレーションファイル

```sql
-- V1__Initial_schema.sql
-- Description: Create initial database schema

-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(255) NOT NULL,
    status VARCHAR(20) DEFAULT 'active' NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_users_email ON users(email);

-- Profiles table
CREATE TABLE profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar VARCHAR(500),
    website VARCHAR(500),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_profiles_user_id ON profiles(user_id);

-- Posts table
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    published BOOLEAN DEFAULT false NOT NULL,
    published_at TIMESTAMPTZ,
    author_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_posts_author_id ON posts(author_id);
CREATE INDEX idx_posts_published_at ON posts(published_at) WHERE published_at IS NOT NULL;

-- Updated at trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_profiles_updated_at BEFORE UPDATE ON profiles
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_posts_updated_at BEFORE UPDATE ON posts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Comments
COMMENT ON TABLE users IS 'User accounts';
COMMENT ON TABLE profiles IS 'User profiles';
COMMENT ON TABLE posts IS 'User blog posts';
```

```sql
-- V2__Add_full_text_search.sql
-- Description: Add full-text search functionality

-- Add search_vector column to posts
ALTER TABLE posts ADD COLUMN search_vector tsvector;

-- Create trigger for automatic search_vector update
CREATE OR REPLACE FUNCTION posts_search_trigger() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.content, '')), 'B');
    RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_search_update
    BEFORE INSERT OR UPDATE ON posts
    FOR EACH ROW
    EXECUTE FUNCTION posts_search_trigger();

-- Create GIN index for fast full-text search
CREATE INDEX posts_search_idx ON posts USING GIN(search_vector);

-- Update existing rows
UPDATE posts
SET search_vector =
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(content, '')), 'B');
```

```sql
-- R__Create_view_user_stats.sql
-- Description: Create or replace user statistics view
-- Repeatable: This script runs on every migration if changed

CREATE OR REPLACE VIEW user_stats AS
SELECT
    u.id,
    u.username,
    u.email,
    COUNT(DISTINCT p.id) AS post_count,
    COUNT(DISTINCT p.id) FILTER (WHERE p.published = true) AS published_post_count,
    MAX(p.published_at) AS last_published_at,
    u.created_at AS user_created_at
FROM users u
LEFT JOIN posts p ON u.id = p.author_id
GROUP BY u.id, u.username, u.email, u.created_at;

COMMENT ON VIEW user_stats IS 'User statistics including post counts';
```

### マイグレーション実行

```bash
# マイグレーション情報表示
flyway info

# マイグレーション実行
flyway migrate

# バリデーション（適用済みマイグレーションのチェックサム検証）
flyway validate

# ベースライン設定（既存DBに対してFlywayを導入）
flyway baseline -baselineVersion=1.0

# 修復（チェックサムエラーの修正）
flyway repair

# クリーン（全データ削除、開発環境のみ！）
# flyway.cleanDisabled=falseの場合のみ実行可能
flyway clean
```

### コールバック（イベントハンドリング）

Flywayは、マイグレーションの各段階でコールバックスクリプトを実行できます。

```sql
-- beforeMigrate.sql
-- マイグレーション前に実行
INSERT INTO migration_log (event, executed_at)
VALUES ('beforeMigrate', CURRENT_TIMESTAMP);

-- afterMigrate.sql
-- マイグレーション後に実行
INSERT INTO migration_log (event, executed_at)
VALUES ('afterMigrate', CURRENT_TIMESTAMP);

-- beforeEachMigrate.sql
-- 各マイグレーション前に実行
INSERT INTO migration_log (event, executed_at, details)
VALUES ('beforeEachMigrate', CURRENT_TIMESTAMP, '${flyway:filename}');

-- afterEachMigrate.sql
-- 各マイグレーション後に実行
INSERT INTO migration_log (event, executed_at, details)
VALUES ('afterEachMigrate', CURRENT_TIMESTAMP, '${flyway:filename}');
```

### Java統合

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    <version>10.4.1</version>
    <configuration>
        <url>jdbc:postgresql://localhost:5432/mydb</url>
        <user>postgres</user>
        <password>password</password>
        <locations>
            <location>filesystem:src/main/resources/db/migration</location>
        </locations>
        <schemas>
            <schema>public</schema>
        </schemas>
        <table>flyway_schema_history</table>
        <baselineOnMigrate>true</baselineOnMigrate>
    </configuration>
</plugin>
```

```java
// Java コード内でマイグレーション
import org.flywaydb.core.Flyway;

public class Application {
    public static void main(String[] args) {
        Flyway flyway = Flyway.configure()
            .dataSource("jdbc:postgresql://localhost:5432/mydb", "postgres", "password")
            .locations("classpath:db/migration")
            .table("flyway_schema_history")
            .baselineOnMigrate(true)
            .load();

        // マイグレーション情報表示
        flyway.info().all();

        // マイグレーション実行
        int appliedMigrations = flyway.migrate().migrationsExecuted;
        System.out.println("Applied " + appliedMigrations + " migrations");
    }
}
```

---

## Liquibase完全ガイド

Liquibaseは、データベース変更を複数のフォーマット（XML、YAML、JSON、SQL）で管理できる柔軟なマイグレーションツールです。

### セットアップ

```bash
# Liquibaseダウンロード
wget https://github.com/liquibase/liquibase/releases/download/v4.25.1/liquibase-4.25.1.tar.gz
tar -xzf liquibase-4.25.1.tar.gz

# またはDockerで実行
docker run --rm \
  -v $(pwd):/liquibase/changelog \
  liquibase/liquibase:4.25.1 \
  --url="jdbc:postgresql://host.docker.internal:5432/mydb" \
  --changeLogFile=changelog.xml \
  --username=postgres \
  --password=password \
  update
```

### 設定

```properties
# liquibase.properties
changeLogFile=db/changelog/db.changelog-master.xml
url=jdbc:postgresql://localhost:5432/mydb
username=postgres
password=password
driver=org.postgresql.Driver
classpath=./lib/postgresql-42.7.1.jar
```

### チェンジログ（XML形式）

```xml
<!-- db/changelog/db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.25.xsd">

    <!-- チェンジセットファイルをインクルード -->
    <include file="db/changelog/changes/001-initial-schema.xml"/>
    <include file="db/changelog/changes/002-add-indexes.xml"/>
    <include file="db/changelog/changes/003-add-full-text-search.xml"/>
</databaseChangeLog>
```

```xml
<!-- db/changelog/changes/001-initial-schema.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.25.xsd">

    <changeSet id="001-create-users-table" author="developer">
        <createTable tableName="users">
            <column name="id" type="SERIAL">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints unique="true" nullable="false"/>
            </column>
            <column name="username" type="VARCHAR(50)">
                <constraints nullable="false"/>
            </column>
            <column name="password" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="status" type="VARCHAR(20)" defaultValue="active">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP WITH TIME ZONE"
                    defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP WITH TIME ZONE"
                    defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>

    <changeSet id="002-create-profiles-table" author="developer">
        <createTable tableName="profiles">
            <column name="id" type="SERIAL">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="user_id" type="INTEGER">
                <constraints nullable="false" unique="true"
                             foreignKeyName="fk_profiles_user_id"
                             references="users(id)"
                             deleteCascade="true"/>
            </column>
            <column name="bio" type="TEXT"/>
            <column name="avatar" type="VARCHAR(500)"/>
            <column name="website" type="VARCHAR(500)"/>
            <column name="created_at" type="TIMESTAMP WITH TIME ZONE"
                    defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP WITH TIME ZONE"
                    defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <rollback>
            <dropTable tableName="profiles"/>
        </rollback>
    </changeSet>

    <changeSet id="003-create-posts-table" author="developer">
        <createTable tableName="posts">
            <column name="id" type="SERIAL">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="title" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="content" type="TEXT"/>
            <column name="published" type="BOOLEAN" defaultValueBoolean="false">
                <constraints nullable="false"/>
            </column>
            <column name="published_at" type="TIMESTAMP WITH TIME ZONE"/>
            <column name="author_id" type="INTEGER">
                <constraints nullable="false"
                             foreignKeyName="fk_posts_author_id"
                             references="users(id)"
                             deleteCascade="true"/>
            </column>
            <column name="created_at" type="TIMESTAMP WITH TIME ZONE"
                    defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP WITH TIME ZONE"
                    defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <rollback>
            <dropTable tableName="posts"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

### チェンジログ（YAML形式）

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-initial-schema.yaml
  - include:
      file: db/changelog/changes/002-add-indexes.yaml
```

```yaml
# db/changelog/changes/001-initial-schema.yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-users-table
      author: developer
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: SERIAL
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    unique: true
                    nullable: false
              - column:
                  name: username
                  type: VARCHAR(50)
                  constraints:
                    nullable: false
              - column:
                  name: password
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
              - column:
                  name: status
                  type: VARCHAR(20)
                  defaultValue: active
                  constraints:
                    nullable: false
              - column:
                  name: created_at
                  type: TIMESTAMP WITH TIME ZONE
                  defaultValueComputed: CURRENT_TIMESTAMP
                  constraints:
                    nullable: false
              - column:
                  name: updated_at
                  type: TIMESTAMP WITH TIME ZONE
                  defaultValueComputed: CURRENT_TIMESTAMP
                  constraints:
                    nullable: false
      rollback:
        - dropTable:
            tableName: users
```

### チェンジログ（SQL形式）

```sql
-- db/changelog/changes/001-initial-schema.sql
--liquibase formatted sql

--changeset developer:001
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(255) NOT NULL,
    status VARCHAR(20) DEFAULT 'active' NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
--rollback DROP TABLE users;

--changeset developer:002
CREATE TABLE profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar VARCHAR(500),
    website VARCHAR(500),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
--rollback DROP TABLE profiles;

--changeset developer:003
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    published BOOLEAN DEFAULT false NOT NULL,
    published_at TIMESTAMPTZ,
    author_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
--rollback DROP TABLE posts;
```

### マイグレーション実行

```bash
# マイグレーション状態確認
liquibase status

# マイグレーション実行
liquibase update

# ロールバック（直前のチェンジセット）
liquibase rollbackCount 1

# ロールバック（特定のタグまで）
liquibase rollback v1.0

# タグ付け
liquibase tag v1.0

# SQL生成（実行なし）
liquibase updateSQL

# 差分生成（既存DBから）
liquibase diffChangeLog \
  --referenceUrl=jdbc:postgresql://localhost/dbname \
  --referenceUsername=postgres \
  --referencePassword=password

# バリデーション
liquibase validate

# チェンジログの同期（手動変更後）
liquibase changelogSync

# チェックサムクリア
liquibase clearCheckSums
```

### 前提条件（Preconditions）

```xml
<changeSet id="004-add-comments" author="developer">
    <preConditions onFail="MARK_RAN">
        <!-- テーブルが存在しない場合のみ実行 -->
        <not>
            <tableExists tableName="comments"/>
        </not>
    </preConditions>

    <createTable tableName="comments">
        <!-- ... -->
    </createTable>
</changeSet>
```

### コンテキスト（環境別実行）

```xml
<changeSet id="005-seed-test-data" author="developer" context="dev">
    <!-- 開発環境のみ実行 -->
    <insert tableName="users">
        <column name="email" value="test@example.com"/>
        <column name="username" value="testuser"/>
        <column name="password" value="hashed_password"/>
    </insert>
</changeSet>

<changeSet id="006-add-production-indexes" author="developer" context="prod">
    <!-- 本番環境のみ実行 -->
    <sql>
        CREATE INDEX CONCURRENTLY idx_posts_author_id ON posts(author_id);
    </sql>
</changeSet>
```

```bash
# コンテキスト指定で実行
liquibase update --contexts=dev
liquibase update --contexts=prod
```

---

## ブルーグリーンデプロイメント

アプリケーションとデータベースを同時に切り替え、ゼロダウンタイムを実現します。

### アーキテクチャ

```
┌─────────────────┐
│ Load Balancer   │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───▼──┐  ┌──▼───┐
│ Blue │  │Green │
│ App  │  │ App  │
│ v1.0 │  │ v2.0 │
└───┬──┘  └──┬───┘
    │         │
┌───▼─────────▼───┐
│   Database      │
│ (共通または分離) │
└─────────────────┘
```

### データベース共通パターン

マイグレーションを後方互換性のある形で適用し、Blue/Green両方のアプリケーションが同じデータベースを使用します。

```bash
#!/bin/bash
# blue-green-deploy.sh - データベース共通パターン

set -e

echo "=== Blue-Green Deployment with Shared Database ==="

CURRENT_ENV="blue"
NEW_ENV="green"
DATABASE_URL="postgresql://localhost:5432/mydb"

# 1. Green環境にアプリケーションをデプロイ
echo "Step 1: Deploying to $NEW_ENV environment..."
kubectl apply -f k8s/green-deployment.yaml

# 2. マイグレーション適用（後方互換性あり）
echo "Step 2: Running backward-compatible migrations..."
DATABASE_URL=$DATABASE_URL npx prisma migrate deploy

# 3. Green環境のヘルスチェック
echo "Step 3: Health checking $NEW_ENV environment..."
for i in {1..30}; do
  if curl -f http://green-api/health; then
    echo "$NEW_ENV is healthy"
    break
  fi

  if [ $i -eq 30 ]; then
    echo "ERROR: Health check failed"
    exit 1
  fi

  echo "Waiting for $NEW_ENV to be ready..."
  sleep 2
done

# 4. カナリアリリース（10%のトラフィックをGreenに）
echo "Step 4: Canary release (10% traffic to $NEW_ENV)..."
kubectl patch service api-service -p '{
  "spec": {
    "selector": {"version":"green"},
    "sessionAffinity": "ClientIP"
  }
}'

# カナリア期間中の監視（5分間）
echo "Monitoring canary release for 5 minutes..."
sleep 300

# エラー率確認
ERROR_RATE=$(curl -s http://monitoring/api/error-rate?env=green)
if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
  echo "ERROR: High error rate ($ERROR_RATE%) in $NEW_ENV"
  echo "Rolling back to $CURRENT_ENV..."
  kubectl patch service api-service -p '{
    "spec": {"selector": {"version":"blue"}}
  }'
  kubectl delete deployment green-api
  exit 1
fi

# 5. 完全切り替え（100%のトラフィックをGreenに）
echo "Step 5: Full switch to $NEW_ENV (100% traffic)..."
kubectl patch service api-service -p '{
  "spec": {"selector": {"version":"green"}}
}'

# 6. Blue環境の監視継続（30分間）
echo "Step 6: Monitoring $NEW_ENV for 30 minutes..."
sleep 1800

# 7. Blue環境削除
echo "Step 7: Shutting down $CURRENT_ENV..."
kubectl delete deployment blue-api

echo "=== Blue-Green deployment completed successfully ==="
```

### データベース分離パターン

Blue/Green環境がそれぞれ独立したデータベースを持ち、データレプリケーションで同期します。

```bash
#!/bin/bash
# blue-green-deploy-isolated-db.sh - データベース分離パターン

set -e

echo "=== Blue-Green Deployment with Isolated Databases ==="

BLUE_DB="mydb_blue"
GREEN_DB="mydb_green"
CURRENT_ENV="blue"
NEW_ENV="green"

# 1. Green環境用のデータベース作成
echo "Step 1: Creating $GREEN_DB database..."
createdb $GREEN_DB

# 2. データレプリケーション（Blueから Greenへ）
echo "Step 2: Replicating data from $BLUE_DB to $GREEN_DB..."
pg_dump -U postgres -d $BLUE_DB -F c | pg_restore -U postgres -d $GREEN_DB

# 3. Green環境にマイグレーション適用
echo "Step 3: Running migrations on $GREEN_DB..."
DATABASE_URL="postgresql://localhost:5432/$GREEN_DB" npx prisma migrate deploy

# 4. Green環境のアプリケーションをデプロイ
echo "Step 4: Deploying $NEW_ENV application..."
kubectl apply -f k8s/green-deployment.yaml

# 5. Green環境のヘルスチェック
echo "Step 5: Health checking $NEW_ENV environment..."
for i in {1..30}; do
  if curl -f http://green-api/health; then
    echo "$NEW_ENV is healthy"
    break
  fi

  if [ $i -eq 30 ]; then
    echo "ERROR: Health check failed"
    exit 1
  fi

  echo "Waiting for $NEW_ENV to be ready..."
  sleep 2
done

# 6. ロードバランサーをGreen環境に切り替え
echo "Step 6: Switching traffic to $NEW_ENV..."
kubectl patch service api-service -p '{
  "spec": {"selector": {"version":"green"}}
}'

# 7. 動作確認（10分間監視）
echo "Step 7: Monitoring $NEW_ENV for 10 minutes..."
sleep 600

# エラー率確認
ERROR_RATE=$(curl -s http://monitoring/api/error-rate?env=green)
if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
  echo "ERROR: High error rate in $NEW_ENV"
  echo "Rolling back to $CURRENT_ENV..."
  kubectl patch service api-service -p '{
    "spec": {"selector": {"version":"blue"}}
  }'
  kubectl delete deployment green-api
  dropdb $GREEN_DB
  exit 1
fi

# 8. Blue環境とデータベース削除
echo "Step 8: Cleaning up $CURRENT_ENV..."
kubectl delete deployment blue-api
dropdb $BLUE_DB

echo "=== Blue-Green deployment completed successfully ==="
```

---

## 災害復旧計画

### Point-in-Time Recovery（PITR）

PostgreSQLの継続的アーカイブ機能を使用して、任意の時点までデータベースを復旧できます。

```bash
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /mnt/archive/%f'
max_wal_senders = 3
```

```bash
# ベースバックアップ作成
pg_basebackup -D /mnt/backup/base -F tar -z -P -U postgres

# 特定時点まで復旧
# recovery.conf
restore_command = 'cp /mnt/archive/%f %p'
recovery_target_time = '2026-01-03 12:00:00'
recovery_target_action = 'promote'
```

### レプリケーション

```bash
# マスターサーバー（postgresql.conf）
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64

# レプリカサーバー
pg_basebackup -h master -D /var/lib/postgresql/data -U replication -P -v

# standby.signal ファイル作成
touch /var/lib/postgresql/data/standby.signal

# postgresql.auto.conf
primary_conninfo = 'host=master port=5432 user=replication password=secret'
```

### 定期バックアップ

```bash
#!/bin/bash
# backup.sh - 定期バックアップスクリプト

BACKUP_DIR="/mnt/backups"
RETENTION_DAYS=30
DB_NAME="mydb"
DB_USER="postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=== Database Backup Script ==="
echo "Date: $(date)"
echo "Database: $DB_NAME"

# バックアップ作成
pg_dump -U $DB_USER -d $DB_NAME -F c -f "$BACKUP_DIR/backup_$TIMESTAMP.dump"

if [ $? -eq 0 ]; then
  echo "Backup created: backup_$TIMESTAMP.dump"
else
  echo "ERROR: Backup failed"
  exit 1
fi

# スキーマのみバックアップ
pg_dump -U $DB_USER -d $DB_NAME -s -f "$BACKUP_DIR/schema_$TIMESTAMP.sql"

# S3にアップロード（オプション）
aws s3 cp "$BACKUP_DIR/backup_$TIMESTAMP.dump" \
  s3://my-bucket/db-backups/backup_$TIMESTAMP.dump

# 古いバックアップ削除
find "$BACKUP_DIR" -name "backup_*.dump" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "schema_*.sql" -mtime +$RETENTION_DAYS -delete

echo "Backup completed successfully"
```

```cron
# crontab -e
# 毎日午前3時にバックアップ
0 3 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# 毎週日曜日午前2時にフルバックアップ
0 2 * * 0 /usr/local/bin/full-backup.sh >> /var/log/backup.log 2>&1
```

---

## ベンチマーク指標

### 想定シナリオ: SaaSプロダクトのスキーマ進化管理改善効果

#### 導入前の想定課題
- マイグレーション失敗: 年12回（手動実行ミス）
- ダウンタイム: 平均45分/回
- データ消失インシデント: 年4回
- 環境間の不整合: 常時発生
- ロールバック時間: 平均2時間（手動）

#### 導入後に期待できる効果（6ヶ月想定）

| 指標 | 導入前 | 導入後 | 改善率 |
|------|--------|--------|--------|
| マイグレーション失敗 | 年12回 | 年0回 | **-100%** |
| 環境間の不整合 | 常時発生 | 0件 | **-100%** |
| マイグレーションダウンタイム | 45分 | 0分 | **-100%** |
| データ消失インシデント | 年4回 | 年0回 | **-100%** |
| ロールバック時間 | 2時間 | 2分 | **-98%** |
| マイグレーション作成時間 | 30分 | 5分 | **-83%** |
| デプロイ頻度 | 月1回 | 週3回 | **+1100%** |

#### 想定される改善策

1. **Alembic導入** - 自動マイグレーション生成
2. **ゼロダウンタイムマイグレーション** - Expand-Contractパターン
3. **Blue-Greenデプロイ** - 即座に切り戻し可能
4. **自動バックアップ** - Point-in-Time Recovery
5. **CI/CD統合** - GitHub ActionsでマイグレーションとデプロイをCI/CDに組み込み

---

## まとめ

### スキーマ進化管理の成功の鍵

1. **後方互換性** - 段階的な変更で既存コードを動作させ続ける
2. **自動化** - マイグレーション生成とCI/CDパイプライン統合
3. **ゼロダウンタイム** - Expand-Contractパターン、CONCURRENTLY、Blue-Green
4. **バックアップ** - Point-in-Time Recovery、定期バックアップ
5. **監視** - マイグレーション前後のメトリクス監視

### 次のステップ

1. **今すぐ実装**: Alembic/Flyway/Liquibase導入
2. **ゼロダウンタイム**: Expand-Contractパターンの実践
3. **Blue-Green**: データベース分離パターンの検証
4. **PITR**: 継続的アーカイブ設定
5. **ドキュメント化**: マイグレーション計画書のテンプレート作成

### 参考資料

- [Alembic Documentation](https://alembic.sqlalchemy.org/)
- [Flyway Documentation](https://flywaydb.org/documentation/)
- [Liquibase Documentation](https://docs.liquibase.com/)
- [PostgreSQL Continuous Archiving](https://www.postgresql.org/docs/current/continuous-archiving.html)

---

**文字数: 約30,000文字**
