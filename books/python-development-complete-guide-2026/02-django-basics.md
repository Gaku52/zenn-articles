---
title: "Django基礎とMVTパターン"
---

# Chapter 02: Django基礎とMVTパターン

## この章で学べること

Djangoは、「Batteries Included」(必要なものが全て揃っている)という哲学を持つフルスタックWebフレームワークです。この章では、DjangoのMVTパターンと実務で使える開発手法を習得します。

- ✅ DjangoのMVT(Model-View-Template)パターンの理解
- ✅ Django ORMによるデータベース操作
- ✅ Django REST Frameworkを使ったAPI開発
- ✅ Admin管理画面の活用法
- ✅ 想定される効果に基づく開発効率改善

**前提知識**: Pythonの基本文法、MVCパターンの基礎、データベースの基礎知識

**所要時間**: 50-60分

---

## 目次

1. [DjangoとMVTパターン](#1-djangoとmvtパターン)
2. [プロジェクトセットアップ](#2-プロジェクトセットアップ)
3. [モデル定義とORM](#3-モデル定義とorm)
4. [ビューとURL設定](#4-ビューとurl設定)
5. [Django REST Framework](#5-django-rest-framework)
6. [Admin管理画面](#6-admin管理画面)
7. [トラブルシューティング](#7-トラブルシューティング)

---

## 1. DjangoとMVTパターン

### 1.1 MVTパターンとは

DjangoはMVT (Model-View-Template) パターンを採用しています。

```
MVCパターン              MVTパターン (Django)
┌────────────┐          ┌────────────┐
│   Model    │  ───→    │   Model    │ (データベース層)
│ (データ層)  │          │            │
└────────────┘          └────────────┘
       ↕                       ↕
┌────────────┐          ┌────────────┐
│ Controller │  ───→    │    View    │ (ビジネスロジック層)
│ (処理層)    │          │            │
└────────────┘          └────────────┘
       ↕                       ↕
┌────────────┐          ┌────────────┐
│    View    │  ───→    │  Template  │ (プレゼンテーション層)
│ (表示層)    │          │            │
└────────────┘          └────────────┘
```

**各層の役割**:
- **Model**: データベーススキーマ、データ操作ロジック
- **View**: ビジネスロジック、リクエスト処理
- **Template**: HTML生成、レスポンス表示

### 1.2 Djangoの特徴

**強み**:
- 🔥 **豊富な機能**: ORM、Admin、認証、フォーム処理、テンプレートエンジン
- 🛡️ **セキュリティ**: CSRF、XSS、SQLインジェクション対策が標準装備
- 📦 **バッテリー同梱**: 必要な機能がほぼ全て含まれている
- 🚀 **スケーラビリティ**: Instagram、Spotify、Dropboxなどで採用

**想定される効果: Djangoの効果**:

```
Admin管理画面の開発時間: 2週間 → 1時間 (-99%)
CRUD API開発速度: 週5エンドポイント → 週15エンドポイント (+200%)
セキュリティ脆弱性: 手動実装で発見される → 標準機能で100%防御
マイグレーション管理: 手動SQL作成 → 完全自動化 (100%削減)
```

---

## 2. プロジェクトセットアップ

### 2.1 インストールと初期設定

```bash
# プロジェクトディレクトリ作成
mkdir myproject && cd myproject

# 仮想環境作成
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Djangoインストール
pip install django djangorestframework psycopg2-binary django-environ

# 開発ツール
pip install ruff mypy pytest pytest-django
```

**パッケージ説明**:
- `django`: Djangoフレームワーク本体
- `djangorestframework`: REST API開発
- `psycopg2-binary`: PostgreSQLドライバ
- `django-environ`: 環境変数管理

### 2.2 プロジェクト作成

```bash
# Djangoプロジェクト作成
django-admin startproject config .

# アプリケーション作成
python manage.py startapp users
python manage.py startapp posts
```

**ディレクトリ構造**:
```
myproject/
├── config/                    # プロジェクト設定
│   ├── __init__.py
│   ├── settings.py            # 設定ファイル
│   ├── urls.py                # ルートURL設定
│   ├── wsgi.py                # WSGIエントリーポイント
│   └── asgi.py                # ASGIエントリーポイント
├── users/                     # ユーザーアプリ
│   ├── migrations/            # マイグレーションファイル
│   ├── __init__.py
│   ├── admin.py               # Admin設定
│   ├── apps.py                # アプリ設定
│   ├── models.py              # モデル定義
│   ├── serializers.py         # DRFシリアライザー
│   ├── views.py               # ビュー
│   └── urls.py                # URL設定
├── posts/                     # 投稿アプリ
│   └── (同上)
├── manage.py                  # 管理コマンド
├── .env                       # 環境変数
└── requirements.txt           # 依存関係
```

### 2.3 環境変数管理

```bash
# django-environをインストール
pip install django-environ
```

**config/settings.py**:
```python
import environ
import os
from pathlib import Path

# Build paths
BASE_DIR = Path(__file__).resolve().parent.parent

# 環境変数読み込み
env = environ.Env(
    DEBUG=(bool, False)
)
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

# セキュリティ設定
SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS', default=[])

# データベース
DATABASES = {
    'default': env.db()  # DATABASE_URLから自動解析
}

# アプリケーション定義
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'users',
    'posts',
]
```

**.env**:
```bash
SECRET_KEY="django-insecure-change-this-in-production"
DEBUG=True
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
ALLOWED_HOSTS=localhost,127.0.0.1
```

---

## 3. モデル定義とORM

### 3.1 基本的なモデル定義

**users/models.py**:
```python
from django.db import models
from django.contrib.auth.models import AbstractUser


class User(AbstractUser):
    """カスタムユーザーモデル"""
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    class Meta:
        db_table = 'users'
        ordering = ['-created_at']

    def __str__(self):
        return self.email
```

**posts/models.py**:
```python
from django.db import models
from django.conf import settings


class Post(models.Model):
    """投稿モデル"""
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='posts'
    )
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = 'posts'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['-created_at']),
            models.Index(fields=['author', 'published']),
        ]

    def __str__(self):
        return self.title
```

**config/settings.py** (カスタムユーザーモデル設定):
```python
AUTH_USER_MODEL = 'users.User'
```

### 3.2 マイグレーション

```bash
# マイグレーションファイル作成
python manage.py makemigrations

# 出力例:
# Migrations for 'users':
#   users/migrations/0001_initial.py
#     - Create model User
# Migrations for 'posts':
#   posts/migrations/0001_initial.py
#     - Create model Post

# マイグレーション適用
python manage.py migrate

# 出力例:
# Operations to perform:
#   Apply all migrations: admin, auth, contenttypes, sessions, users, posts
# Running migrations:
#   Applying users.0001_initial... OK
#   Applying posts.0001_initial... OK

# マイグレーションの確認
python manage.py showmigrations
```

### 3.3 Django ORMの基本操作

**Create (作成)**:
```python
from users.models import User
from posts.models import Post

# ユーザー作成
user = User.objects.create_user(
    email='john@example.com',
    username='john',
    password='password123'
)

# 投稿作成
post = Post.objects.create(
    title='First Post',
    content='This is my first post',
    author=user,
    published=True
)
```

**Read (取得)**:
```python
# 全件取得
all_posts = Post.objects.all()

# フィルタ
published_posts = Post.objects.filter(published=True)
user_posts = Post.objects.filter(author=user)

# 取得(1件)
post = Post.objects.get(id=1)

# 取得または404エラー
from django.shortcuts import get_object_or_404
post = get_object_or_404(Post, id=1)

# 条件付き取得
recent_posts = Post.objects.filter(
    published=True,
    created_at__gte='2024-01-01'
).order_by('-created_at')[:10]
```

**Update (更新)**:
```python
# 単一オブジェクトの更新
post = Post.objects.get(id=1)
post.title = 'Updated Title'
post.save()

# 一括更新
Post.objects.filter(author=user).update(published=True)
```

**Delete (削除)**:
```python
# 単一オブジェクトの削除
post = Post.objects.get(id=1)
post.delete()

# 一括削除
Post.objects.filter(published=False).delete()
```

### 3.4 リレーションとN+1問題の解決

**N+1問題の例 (遅い)**:
```python
# ❌ N+1問題: 各投稿ごとにauthorを取得するクエリが発行される
posts = Post.objects.all()
for post in posts:
    print(post.author.email)  # 追加のクエリが発生
```

**select_related (1対1、ForeignKey)**:
```python
# ✅ JOINで1回のクエリで取得
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.email)  # 追加のクエリなし
```

**prefetch_related (ManyToMany、逆参照)**:
```python
# ✅ 2回のクエリで効率的に取得
users = User.objects.prefetch_related('posts').all()
for user in users:
    print(f"{user.email}: {user.posts.count()} posts")
```

**想定される効果: N+1問題の影響**:
```
N+1問題あり: 101クエリ、応答時間 1200ms
select_related: 1クエリ、応答時間 45ms (-96%)
prefetch_related: 2クエリ、応答時間 60ms (-95%)
```

---

## 4. ビューとURL設定

### 4.1 関数ベースビュー

**posts/views.py**:
```python
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from .models import Post


def post_list(request):
    """投稿一覧"""
    posts = Post.objects.filter(published=True).select_related('author')
    return JsonResponse({
        'posts': list(posts.values('id', 'title', 'author__email'))
    })


def post_detail(request, pk):
    """投稿詳細"""
    post = get_object_or_404(Post.objects.select_related('author'), pk=pk)
    return JsonResponse({
        'id': post.id,
        'title': post.title,
        'content': post.content,
        'author': post.author.email,
        'created_at': post.created_at.isoformat()
    })
```

**posts/urls.py**:
```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.post_list, name='post-list'),
    path('<int:pk>/', views.post_detail, name='post-detail'),
]
```

**config/urls.py**:
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/posts/', include('posts.urls')),
]
```

### 4.2 クラスベースビュー

```python
from django.views.generic import ListView, DetailView
from .models import Post


class PostListView(ListView):
    """投稿一覧ビュー"""
    model = Post
    queryset = Post.objects.filter(published=True).select_related('author')
    context_object_name = 'posts'
    template_name = 'posts/list.html'


class PostDetailView(DetailView):
    """投稿詳細ビュー"""
    model = Post
    queryset = Post.objects.select_related('author')
    context_object_name = 'post'
    template_name = 'posts/detail.html'
```

---

## 5. Django REST Framework

### 5.1 セットアップ

**config/settings.py**:
```python
INSTALLED_APPS = [
    # ...
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
}
```

### 5.2 シリアライザー

**posts/serializers.py**:
```python
from rest_framework import serializers
from .models import Post
from users.models import User


class UserSerializer(serializers.ModelSerializer):
    """ユーザーシリアライザー"""
    posts_count = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'email', 'username', 'bio', 'posts_count', 'created_at']
        read_only_fields = ['id', 'created_at']

    def get_posts_count(self, obj):
        return obj.posts.count()


class PostSerializer(serializers.ModelSerializer):
    """投稿シリアライザー"""
    author = UserSerializer(read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'published', 'created_at', 'updated_at']
        read_only_fields = ['id', 'author', 'created_at', 'updated_at']


class PostCreateSerializer(serializers.ModelSerializer):
    """投稿作成シリアライザー"""
    class Meta:
        model = Post
        fields = ['title', 'content', 'published']
```

### 5.3 ViewSet

**posts/views.py**:
```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, AllowAny
from .models import Post
from .serializers import PostSerializer, PostCreateSerializer


class PostViewSet(viewsets.ModelViewSet):
    """投稿ViewSet"""
    queryset = Post.objects.select_related('author').all()

    def get_serializer_class(self):
        if self.action in ['create', 'update', 'partial_update']:
            return PostCreateSerializer
        return PostSerializer

    def get_queryset(self):
        queryset = super().get_queryset()
        # 未認証ユーザーには公開済みのみ表示
        if not self.request.user.is_authenticated:
            queryset = queryset.filter(published=True)
        return queryset

    def perform_create(self, serializer):
        """投稿作成時にauthorを自動設定"""
        serializer.save(author=self.request.user)

    @action(detail=False, methods=['get'])
    def my_posts(self, request):
        """自分の投稿一覧"""
        posts = self.queryset.filter(author=request.user)
        page = self.paginate_queryset(posts)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        serializer = self.get_serializer(posts, many=True)
        return Response(serializer.data)
```

**posts/urls.py**:
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import PostViewSet

router = DefaultRouter()
router.register('posts', PostViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

**想定される効果: ViewSetの効果**:
```
CRUD API実装時間: 1日 → 2時間 (-75%)
コード量: 200行 → 50行 (-75%)
テストケース作成時間: 4時間 → 1時間 (-75%)
```

---

## 6. Admin管理画面

### 6.1 基本的なAdmin設定

**users/admin.py**:
```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from .models import User


@admin.register(User)
class UserAdmin(BaseUserAdmin):
    """ユーザーAdmin"""
    list_display = ['email', 'username', 'is_staff', 'is_active', 'created_at']
    list_filter = ['is_staff', 'is_active', 'created_at']
    search_fields = ['email', 'username']
    ordering = ['-created_at']

    fieldsets = BaseUserAdmin.fieldsets + (
        ('追加情報', {'fields': ('bio', 'avatar')}),
    )
```

**posts/admin.py**:
```python
from django.contrib import admin
from .models import Post


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    """投稿Admin"""
    list_display = ['title', 'author', 'published', 'created_at']
    list_filter = ['published', 'created_at']
    search_fields = ['title', 'content']
    raw_id_fields = ['author']
    date_hierarchy = 'created_at'
    actions = ['make_published', 'make_unpublished']

    def make_published(self, request, queryset):
        """一括公開"""
        queryset.update(published=True)
    make_published.short_description = "選択した投稿を公開する"

    def make_unpublished(self, request, queryset):
        """一括非公開"""
        queryset.update(published=False)
    make_unpublished.short_description = "選択した投稿を非公開にする"
```

### 6.2 スーパーユーザー作成

```bash
python manage.py createsuperuser
# Email: admin@example.com
# Username: admin
# Password: ********

# サーバー起動
python manage.py runserver

# Admin管理画面にアクセス
# http://localhost:8000/admin/
```

**想定される効果: Admin管理画面の効果**:
```
管理画面開発時間: 2週間 → 1時間 (-99%)
CRUD操作実装: 手動実装 → 自動生成 (100%削減)
データ一覧・検索機能: 1週間 → 10分 (-99%)
```

---

## 7. トラブルシューティング

### 7.1 "No module named 'users'"

**問題**:
```
ModuleNotFoundError: No module named 'users'
```

**原因**: アプリケーションがINSTALLED_APPSに登録されていない

**解決策**:
```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'users',
    'posts',
]
```

### 7.2 "auth.User has been swapped for 'users.User'"

**問題**:
```
django.db.migrations.exceptions.InconsistentMigrationHistory
```

**原因**: カスタムユーザーモデルの設定が既存マイグレーション後に行われた

**解決策**:
```bash
# データベースを削除して再作成
python manage.py flush
python manage.py migrate
```

### 7.3 "CSRF verification failed"

**問題**:
```
Forbidden (403)
CSRF verification failed. Request aborted.
```

**解決策**:
```python
# DRFのセッション認証を使う場合
from rest_framework.decorators import api_view
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt  # CSRFトークン不要（開発時のみ）
@api_view(['POST'])
def my_view(request):
    return Response({'message': 'ok'})
```

---

## まとめ

この章では、DjangoのMVTパターンと実務での活用法を学びました:

✅ **MVTパターン**: Model-View-Templateの役割と設計
✅ **Django ORM**: モデル定義、マイグレーション、N+1問題解決
✅ **Django REST Framework**: シリアライザー、ViewSetによる高速API開発
✅ **Admin管理画面**: 自動生成による開発時間の99%削減

**想定されるシナリオで期待できる効果**:
- Admin管理画面開発時間: -99%
- CRUD API開発速度: +200%
- N+1問題解決によるAPI応答時間: -96%

**次の章では**: Flask、FastAPI、Djangoの比較と、プロジェクトに応じたフレームワーク選定基準を学びます。

---

## 参考リンク

- [Django公式ドキュメント](https://docs.djangoproject.com/)
- [Django REST Framework](https://www.django-rest-framework.org/)
- [Django ORM Cookbook](https://books.agiliq.com/projects/django-orm-cookbook/)

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**
