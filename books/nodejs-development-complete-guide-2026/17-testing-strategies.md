---
title: "テスト戦略 - 高品質なコードを保つ"
---

# テスト戦略 - 高品質なコードを保つ

Jest、Vitest、Superte stを使った包括的なテスト戦略を学びます。

## テストピラミッド

```
        /\
       /  \      E2E Tests (少ない)
      /____\
     /      \    Integration Tests (中程度)
    /________\
   /          \  Unit Tests (多い)
  /__________  \
```

### 各テストレベルの割合

- **Unit Tests: 70%** - 関数・モジュール単位
- **Integration Tests: 20%** - API・DB連携
- **E2E Tests: 10%** - ユーザーシナリオ

## Jest によるユニットテスト

### 基本設定

```typescript
// jest.config.js
export default {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.test.ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

### ユニットテストの例

```typescript
// src/utils/validator.test.ts
import { validateEmail, validatePassword } from './validator';

describe('validateEmail', () => {
  it('should return true for valid email', () => {
    expect(validateEmail('test@example.com')).toBe(true);
  });

  it('should return false for invalid email', () => {
    expect(validateEmail('invalid')).toBe(false);
    expect(validateEmail('test@')).toBe(false);
    expect(validateEmail('@example.com')).toBe(false);
  });
});

describe('validatePassword', () => {
  it('should require minimum 8 characters', () => {
    expect(validatePassword('short')).toBe(false);
    expect(validatePassword('Valid123')).toBe(true);
  });

  it('should require uppercase, lowercase, and number', () => {
    expect(validatePassword('lowercase123')).toBe(false);
    expect(validatePassword('UPPERCASE123')).toBe(false);
    expect(validatePassword('NoNumbers')).toBe(false);
    expect(validatePassword('Valid123')).toBe(true);
  });
});
```

### モックとスパイ

```typescript
// src/services/user.test.ts
import { UserService } from './user';
import { prisma } from '../lib/prisma';

jest.mock('../lib/prisma', () => ({
  prisma: {
    user: {
      findUnique: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
    },
  },
}));

describe('UserService', () => {
  let userService: UserService;

  beforeEach(() => {
    userService = new UserService();
    jest.clearAllMocks();
  });

  describe('getUser', () => {
    it('should return user when found', async () => {
      const mockUser = { id: '1', email: 'test@example.com' };
      (prisma.user.findUnique as jest.Mock).mockResolvedValue(mockUser);

      const result = await userService.getUser('1');

      expect(result).toEqual(mockUser);
      expect(prisma.user.findUnique).toHaveBeenCalledWith({
        where: { id: '1' },
      });
    });

    it('should throw error when user not found', async () => {
      (prisma.user.findUnique as jest.Mock).mockResolvedValue(null);

      await expect(userService.getUser('1')).rejects.toThrow('User not found');
    });
  });
});
```

## 統合テスト

### API テスト (Supertest)

```typescript
// src/routes/users.test.ts
import request from 'supertest';
import { app } from '../app';
import { prisma } from '../lib/prisma';

describe('User API', () => {
  beforeEach(async () => {
    await prisma.user.deleteMany();
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  describe('POST /api/users', () => {
    it('should create a new user', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'Valid123',
        name: 'Test User',
      };

      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201);

      expect(response.body).toMatchObject({
        email: userData.email,
        name: userData.name,
      });
      expect(response.body.password).toBeUndefined();
    });

    it('should return 400 for invalid email', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({
          email: 'invalid',
          password: 'Valid123',
          name: 'Test',
        })
        .expect(400);

      expect(response.body.error).toBeDefined();
    });
  });

  describe('GET /api/users/:id', () => {
    it('should return user by id', async () => {
      const user = await prisma.user.create({
        data: {
          email: 'test@example.com',
          password: 'hashed',
          name: 'Test',
        },
      });

      const response = await request(app)
        .get(`/api/users/${user.id}`)
        .expect(200);

      expect(response.body).toMatchObject({
        id: user.id,
        email: user.email,
      });
    });

    it('should return 404 for non-existent user', async () => {
      await request(app).get('/api/users/invalid-id').expect(404);
    });
  });
});
```

### 認証テスト

```typescript
describe('Authentication', () => {
  let authToken: string;

  beforeEach(async () => {
    const user = await prisma.user.create({
      data: {
        email: 'test@example.com',
        password: await hashPassword('Valid123'),
        name: 'Test',
      },
    });

    const response = await request(app).post('/auth/login').send({
      email: 'test@example.com',
      password: 'Valid123',
    });

    authToken = response.body.token;
  });

  it('should access protected route with valid token', async () => {
    await request(app)
      .get('/api/profile')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);
  });

  it('should deny access without token', async () => {
    await request(app).get('/api/profile').expect(401);
  });

  it('should deny access with invalid token', async () => {
    await request(app)
      .get('/api/profile')
      .set('Authorization', 'Bearer invalid')
      .expect(401);
  });
});
```

## E2Eテスト (Playwright)

### 基本設定

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### E2Eテストの例

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test('should login successfully', async ({ page }) => {
    await page.goto('/login');

    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'Valid123');
    await page.click('button[type="submit"]');

    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toContainText('Dashboard');
  });

  test('should show error for invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'wrong');
    await page.click('button[type="submit"]');

    await expect(page.locator('.error')).toContainText('Invalid credentials');
  });
});
```

## テストデータ管理

### Fixtures

```typescript
// tests/fixtures/users.ts
export const testUsers = {
  admin: {
    email: 'admin@example.com',
    password: 'Admin123',
    name: 'Admin User',
    role: 'ADMIN',
  },
  regular: {
    email: 'user@example.com',
    password: 'User123',
    name: 'Regular User',
    role: 'USER',
  },
};

// tests/setup.ts
export async function seedDatabase() {
  for (const userData of Object.values(testUsers)) {
    await prisma.user.upsert({
      where: { email: userData.email },
      update: {},
      create: {
        ...userData,
        password: await hashPassword(userData.password),
      },
    });
  }
}
```

### Factory パターン

```typescript
// tests/factories/user.factory.ts
import { faker } from '@faker-js/faker';

export class UserFactory {
  static create(overrides = {}) {
    return {
      email: faker.internet.email(),
      password: 'Valid123',
      name: faker.person.fullName(),
      ...overrides,
    };
  }

  static async createInDb(overrides = {}) {
    const userData = this.create(overrides);
    return prisma.user.create({
      data: {
        ...userData,
        password: await hashPassword(userData.password),
      },
    });
  }
}

// 使用例
const user = await UserFactory.createInDb({
  email: 'specific@example.com',
});
```

## カバレッジ測定

### カバレッジレポート

```bash
# カバレッジを収集
npm test -- --coverage

# HTML レポート生成
npm test -- --coverage --coverageReporters=html

# 特定のしきい値を設定
npm test -- --coverage --coverageThreshold='{"global":{"branches":80}}'
```

## CI/CD 統合

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db

      - name: Run tests
        run: npm test -- --coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## スナップショットテスト

```typescript
describe('API Response Snapshots', () => {
  it('should match user response snapshot', async () => {
    const response = await request(app).get('/api/users/1').expect(200);

    expect(response.body).toMatchSnapshot({
      id: expect.any(String),
      createdAt: expect.any(String),
      updatedAt: expect.any(String),
    });
  });
});
```

## まとめ

テスト戦略のベストプラクティス:

- ✅ テストピラミッドに従う（Unit 70%, Integration 20%, E2E 10%）
- ✅ モックを適切に使用
- ✅ カバレッジ80%以上を目標
- ✅ CI/CDでテストを自動化
- ✅ テストデータをFactoryで管理
- ✅ E2Eテストで重要なユーザーフローを検証
- ❌ テストのためにコードを複雑にしない
- ❌ カバレッジ100%を目指さない（コスパ悪い）

次の章では、デプロイと本番運用について学びます。
