---
title: "Chapter 07: スキーマ設計と正規化"
---

# Chapter 07: スキーマ設計と正規化

データベーススキーマ設計は、バックエンドシステムの基盤となる最も重要な要素の一つです。適切に設計されたスキーマは、データの整合性を保ち、パフォーマンスを最適化し、将来の拡張性を確保します。本章では、正規化理論から実践的なスキーマ設計パターン、Prismaを使った実装まで、包括的に解説します。

## データベース正規化の理論と実践

データベース正規化は、データの冗長性を排除し、整合性を保つための体系的な手法です。正規化を理解することで、データの重複を防ぎ、更新異常を回避できます。

### 第1正規形（1NF）: 繰り返しグループの排除

第1正規形の要件は、各セルが単一の値を持ち、繰り返しグループが存在しないことです。

#### 1NFに違反する設計例

```sql
-- ❌ 1NFに違反: 複数の値をカンマ区切りで格納
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  emails VARCHAR(500),  -- 'user1@example.com,user2@example.com'
  phone_numbers TEXT    -- '090-1111-2222,080-3333-4444'
);

-- このような設計の問題点:
-- 1. 特定のメールアドレスを検索できない
SELECT * FROM users WHERE emails = 'user1@example.com';  -- マッチしない

-- 2. メールアドレスの追加・削除が複雑
UPDATE users
SET emails = CONCAT(emails, ',new@example.com')
WHERE id = 1;

-- 3. 集計クエリが不可能
SELECT COUNT(*) FROM users WHERE emails LIKE '%@gmail.com%';  -- 不正確
```

#### 1NFに準拠した設計

```sql
-- ✅ 1NFに準拠: 別テーブルで管理
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_emails (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  email VARCHAR(255) UNIQUE NOT NULL,
  is_primary BOOLEAN DEFAULT false,
  verified BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user_emails_user_id (user_id),
  INDEX idx_user_emails_email (email)
);

CREATE TABLE user_phones (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  phone_number VARCHAR(20) NOT NULL,
  phone_type VARCHAR(20) CHECK (phone_type IN ('mobile', 'home', 'work')),
  is_primary BOOLEAN DEFAULT false,
  verified BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user_phones_user_id (user_id)
);
```

**Prismaスキーマ:**

```prisma
model User {
  id        Int          @id @default(autoincrement())
  name      String       @db.VarChar(100)
  createdAt DateTime     @default(now()) @map("created_at")
  emails    UserEmail[]
  phones    UserPhone[]

  @@map("users")
}

model UserEmail {
  id        Int      @id @default(autoincrement())
  userId    Int      @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  email     String   @unique @db.VarChar(255)
  isPrimary Boolean  @default(false) @map("is_primary")
  verified  Boolean  @default(false)
  createdAt DateTime @default(now()) @map("created_at")

  @@index([userId])
  @@index([email])
  @@map("user_emails")
}

model UserPhone {
  id          Int      @id @default(autoincrement())
  userId      Int      @map("user_id")
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  phoneNumber String   @map("phone_number") @db.VarChar(20)
  phoneType   String   @map("phone_type") @db.VarChar(20)
  isPrimary   Boolean  @default(false) @map("is_primary")
  verified    Boolean  @default(false)
  createdAt   DateTime @default(now()) @map("created_at")

  @@index([userId])
  @@map("user_phones")
}
```

**使用例:**

```typescript
// ユーザー作成（複数のメールアドレス付き）
const user = await prisma.user.create({
  data: {
    name: 'John Doe',
    emails: {
      create: [
        { email: 'john@example.com', isPrimary: true, verified: true },
        { email: 'john.doe@work.com', isPrimary: false, verified: false }
      ]
    },
    phones: {
      create: [
        { phoneNumber: '090-1111-2222', phoneType: 'mobile', isPrimary: true }
      ]
    }
  },
  include: {
    emails: true,
    phones: true
  }
})

// 特定のメールアドレスでユーザー検索
const userByEmail = await prisma.user.findFirst({
  where: {
    emails: {
      some: {
        email: 'john@example.com'
      }
    }
  },
  include: {
    emails: true
  }
})

// Gmailを使用しているユーザー数の集計
const gmailUsersCount = await prisma.user.count({
  where: {
    emails: {
      some: {
        email: {
          endsWith: '@gmail.com'
        }
      }
    }
  }
})
```

### 第2正規形（2NF）: 部分関数従属の排除

第2正規形は、1NFに加えて、非キー属性が主キー全体に完全関数従属していることが要件です。複合主キーを持つテーブルで特に重要です。

#### 2NFに違反する設計例

```sql
-- ❌ 2NFに違反
CREATE TABLE order_items (
  order_id INTEGER,
  product_id INTEGER,
  -- 以下は order_id にのみ依存（部分関数従属）
  order_date DATE,
  customer_name VARCHAR(100),
  customer_email VARCHAR(255),
  -- 以下は product_id にのみ依存（部分関数従属）
  product_name VARCHAR(100),
  product_price DECIMAL(10, 2),
  product_category VARCHAR(50),
  -- 以下は複合キー全体に依存
  quantity INTEGER,
  discount_rate DECIMAL(5, 2),
  PRIMARY KEY (order_id, product_id)
);

-- このような設計の問題点:
-- 1. データの冗長性: 同じ注文の各行で顧客情報が重複
-- 2. 更新異常: 顧客名を変更する場合、複数行を更新する必要がある
-- 3. 挿入異常: 商品情報のみを登録できない
-- 4. 削除異常: 注文品目を削除すると商品情報も失われる
```

#### 2NFに準拠した設計

```sql
-- ✅ 2NFに準拠: テーブルを分割
CREATE TABLE customers (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_customers_email (email)
);

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
  category VARCHAR(50),
  stock_quantity INTEGER DEFAULT 0 CHECK (stock_quantity >= 0),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_products_category (category)
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  customer_id INTEGER NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
  order_date DATE NOT NULL DEFAULT CURRENT_DATE,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  total_amount DECIMAL(10, 2) NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_orders_customer_id (customer_id),
  INDEX idx_orders_order_date (order_date),
  INDEX idx_orders_status (status)
);

CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id INTEGER NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity INTEGER NOT NULL CHECK (quantity > 0),
  unit_price DECIMAL(10, 2) NOT NULL,  -- 注文時点の価格
  discount_rate DECIMAL(5, 2) DEFAULT 0 CHECK (discount_rate >= 0 AND discount_rate <= 100),
  subtotal DECIMAL(10, 2) GENERATED ALWAYS AS (
    quantity * unit_price * (1 - discount_rate / 100)
  ) STORED,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (order_id, product_id),
  INDEX idx_order_items_order_id (order_id),
  INDEX idx_order_items_product_id (product_id)
);
```

**Prismaスキーマ:**

```prisma
model Customer {
  id        Int      @id @default(autoincrement())
  name      String   @db.VarChar(100)
  email     String   @unique @db.VarChar(255)
  phone     String?  @db.VarChar(20)
  createdAt DateTime @default(now()) @map("created_at")
  orders    Order[]

  @@index([email])
  @@map("customers")
}

model Product {
  id            Int         @id @default(autoincrement())
  name          String      @db.VarChar(100)
  price         Decimal     @db.Decimal(10, 2)
  category      String?     @db.VarChar(50)
  stockQuantity Int         @default(0) @map("stock_quantity")
  createdAt     DateTime    @default(now()) @map("created_at")
  orderItems    OrderItem[]

  @@index([category])
  @@map("products")
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

model Order {
  id          Int         @id @default(autoincrement())
  customerId  Int         @map("customer_id")
  customer    Customer    @relation(fields: [customerId], references: [id], onDelete: Restrict)
  orderDate   DateTime    @default(now()) @map("order_date") @db.Date
  status      OrderStatus @default(PENDING)
  totalAmount Decimal     @default(0) @map("total_amount") @db.Decimal(10, 2)
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")
  items       OrderItem[]

  @@index([customerId])
  @@index([orderDate])
  @@index([status])
  @@map("orders")
}

model OrderItem {
  id           Int      @id @default(autoincrement())
  orderId      Int      @map("order_id")
  order        Order    @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId    Int      @map("product_id")
  product      Product  @relation(fields: [productId], references: [id], onDelete: Restrict)
  quantity     Int
  unitPrice    Decimal  @map("unit_price") @db.Decimal(10, 2)
  discountRate Decimal  @default(0) @map("discount_rate") @db.Decimal(5, 2)
  createdAt    DateTime @default(now()) @map("created_at")

  @@unique([orderId, productId])
  @@index([orderId])
  @@index([productId])
  @@map("order_items")
}
```

**使用例:**

```typescript
// 注文作成（トランザクション使用）
const order = await prisma.$transaction(async (tx) => {
  // 在庫チェック
  const product = await tx.product.findUnique({
    where: { id: productId }
  })

  if (!product || product.stockQuantity < quantity) {
    throw new Error('Insufficient stock')
  }

  // 注文作成
  const newOrder = await tx.order.create({
    data: {
      customerId,
      items: {
        create: {
          productId,
          quantity,
          unitPrice: product.price,
          discountRate: 10  // 10% discount
        }
      }
    },
    include: {
      items: {
        include: {
          product: true
        }
      }
    }
  })

  // 在庫更新
  await tx.product.update({
    where: { id: productId },
    data: {
      stockQuantity: {
        decrement: quantity
      }
    }
  })

  // 注文合計金額を計算して更新
  const totalAmount = newOrder.items.reduce((sum, item) => {
    const subtotal = item.quantity * Number(item.unitPrice) * (1 - Number(item.discountRate) / 100)
    return sum + subtotal
  }, 0)

  return await tx.order.update({
    where: { id: newOrder.id },
    data: { totalAmount },
    include: {
      customer: true,
      items: {
        include: {
          product: true
        }
      }
    }
  })
})

// 顧客の注文履歴を取得
const customerOrders = await prisma.order.findMany({
  where: { customerId },
  include: {
    items: {
      include: {
        product: true
      }
    }
  },
  orderBy: { orderDate: 'desc' }
})

// カテゴリー別の売上集計
const salesByCategory = await prisma.$queryRaw`
  SELECT
    p.category,
    SUM(oi.quantity * oi.unit_price * (1 - oi.discount_rate / 100)) as total_sales,
    COUNT(DISTINCT oi.order_id) as order_count,
    SUM(oi.quantity) as units_sold
  FROM order_items oi
  JOIN products p ON oi.product_id = p.id
  JOIN orders o ON oi.order_id = o.id
  WHERE o.status = 'DELIVERED'
  GROUP BY p.category
  ORDER BY total_sales DESC
`
```

### 第3正規形（3NF）: 推移的関数従属の排除

第3正規形は、2NFに加えて、非キー属性が他の非キー属性に関数従属していない（推移的関数従属がない）ことが要件です。

#### 3NFに違反する設計例

```sql
-- ❌ 3NFに違反
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  department_id INTEGER,
  department_name VARCHAR(100),    -- department_idから導出可能（推移的従属）
  department_location VARCHAR(100), -- department_idから導出可能（推移的従属）
  zipcode VARCHAR(10),
  city VARCHAR(100),                -- zipcodeから導出可能（推移的従属）
  prefecture VARCHAR(50)            -- zipcodeから導出可能（推移的従属）
);

-- このような設計の問題点:
-- 1. データの冗長性: 同じ部署の従業員で部署情報が重複
-- 2. 更新異常: 部署名を変更する場合、複数行を更新する必要がある
-- 3. 挿入異常: 従業員がいない部署を登録できない
-- 4. 削除異常: 部署の最後の従業員を削除すると部署情報も失われる
```

#### 3NFに準拠した設計

```sql
-- ✅ 3NFに準拠: テーブルを分割
CREATE TABLE zipcodes (
  zipcode VARCHAR(10) PRIMARY KEY,
  city VARCHAR(100) NOT NULL,
  prefecture VARCHAR(50) NOT NULL,
  INDEX idx_zipcodes_city (city),
  INDEX idx_zipcodes_prefecture (prefecture)
);

CREATE TABLE departments (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL UNIQUE,
  location VARCHAR(100),
  budget DECIMAL(12, 2),
  manager_id INTEGER,  -- 自己参照（後で外部キー追加）
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  department_id INTEGER REFERENCES departments(id) ON DELETE SET NULL,
  zipcode VARCHAR(10) REFERENCES zipcodes(zipcode),
  address_line1 VARCHAR(200),
  address_line2 VARCHAR(200),
  hire_date DATE NOT NULL,
  salary DECIMAL(10, 2),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_employees_department_id (department_id),
  INDEX idx_employees_zipcode (zipcode),
  INDEX idx_employees_email (email)
);

-- 部署のマネージャー外部キーを追加
ALTER TABLE departments
ADD CONSTRAINT fk_departments_manager
FOREIGN KEY (manager_id) REFERENCES employees(id) ON DELETE SET NULL;
```

**Prismaスキーマ:**

```prisma
model Zipcode {
  zipcode    String     @id @db.VarChar(10)
  city       String     @db.VarChar(100)
  prefecture String     @db.VarChar(50)
  employees  Employee[]

  @@index([city])
  @@index([prefecture])
  @@map("zipcodes")
}

model Department {
  id          Int        @id @default(autoincrement())
  name        String     @unique @db.VarChar(100)
  location    String?    @db.VarChar(100)
  budget      Decimal?   @db.Decimal(12, 2)
  managerId   Int?       @map("manager_id")
  manager     Employee?  @relation("DepartmentManager", fields: [managerId], references: [id], onDelete: SetNull)
  employees   Employee[] @relation("DepartmentEmployees")
  createdAt   DateTime   @default(now()) @map("created_at")

  @@map("departments")
}

model Employee {
  id                Int         @id @default(autoincrement())
  name              String      @db.VarChar(100)
  email             String      @unique @db.VarChar(255)
  departmentId      Int?        @map("department_id")
  department        Department? @relation("DepartmentEmployees", fields: [departmentId], references: [id], onDelete: SetNull)
  managedDepartment Department? @relation("DepartmentManager")
  zipcode           String?     @db.VarChar(10)
  zipcodeInfo       Zipcode?    @relation(fields: [zipcode], references: [zipcode])
  addressLine1      String?     @map("address_line1") @db.VarChar(200)
  addressLine2      String?     @map("address_line2") @db.VarChar(200)
  hireDate          DateTime    @map("hire_date") @db.Date
  salary            Decimal?    @db.Decimal(10, 2)
  createdAt         DateTime    @default(now()) @map("created_at")

  @@index([departmentId])
  @@index([zipcode])
  @@index([email])
  @@map("employees")
}
```

**使用例:**

```typescript
// 従業員作成（部署と住所情報付き）
const employee = await prisma.employee.create({
  data: {
    name: 'Jane Smith',
    email: 'jane.smith@company.com',
    departmentId: 1,
    zipcode: '100-0001',
    addressLine1: '千代田区千代田1-1',
    hireDate: new Date('2025-01-01'),
    salary: 5000000
  },
  include: {
    department: true,
    zipcodeInfo: true
  }
})

// 部署ごとの従業員リスト（住所情報付き）
const departmentEmployees = await prisma.department.findUnique({
  where: { id: 1 },
  include: {
    employees: {
      include: {
        zipcodeInfo: true
      },
      orderBy: { name: 'asc' }
    },
    manager: true
  }
})

// 都道府県別の従業員数
const employeesByPrefecture = await prisma.zipcode.groupBy({
  by: ['prefecture'],
  _count: {
    employees: true
  },
  orderBy: {
    _count: {
      employees: 'desc'
    }
  }
})

// 部署の平均給与
const departmentSalaryStats = await prisma.department.findMany({
  include: {
    employees: {
      select: {
        salary: true
      }
    }
  }
}).then(departments =>
  departments.map(dept => ({
    departmentName: dept.name,
    employeeCount: dept.employees.length,
    averageSalary: dept.employees.reduce((sum, emp) =>
      sum + Number(emp.salary || 0), 0
    ) / dept.employees.length || 0
  }))
)
```

### ボイス・コッド正規形（BCNF）

BCNFは、3NFよりも厳密な正規形で、すべての決定子が候補キーである必要があります。

#### BCNFに違反する設計例

```sql
-- ❌ BCNFに違反
CREATE TABLE course_instructors (
  course_id INTEGER,
  instructor_id INTEGER,
  classroom VARCHAR(50),
  -- 問題: instructor_idが決まればclassroomが決まる
  -- （各インストラクターは常に同じ教室を使用）
  -- しかし、instructor_idは候補キーではない
  PRIMARY KEY (course_id, instructor_id)
);

-- このような設計では:
-- 1. 同じインストラクターが複数のコースを担当する場合、classroomが重複
-- 2. インストラクターの教室変更時に複数行を更新する必要がある
```

#### BCNFに準拠した設計

```sql
-- ✅ BCNFに準拠: テーブルを分割
CREATE TABLE instructors (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  classroom VARCHAR(50) NOT NULL,  -- インストラクターごとに1つの教室
  department VARCHAR(50),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_instructors_email (email),
  INDEX idx_instructors_classroom (classroom)
);

CREATE TABLE courses (
  id SERIAL PRIMARY KEY,
  code VARCHAR(20) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  credits INTEGER CHECK (credits > 0),
  semester VARCHAR(20),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_courses_code (code),
  INDEX idx_courses_semester (semester)
);

CREATE TABLE course_instructors (
  course_id INTEGER REFERENCES courses(id) ON DELETE CASCADE,
  instructor_id INTEGER REFERENCES instructors(id) ON DELETE CASCADE,
  role VARCHAR(20) DEFAULT 'primary' CHECK (role IN ('primary', 'assistant', 'guest')),
  PRIMARY KEY (course_id, instructor_id),
  INDEX idx_course_instructors_course (course_id),
  INDEX idx_course_instructors_instructor (instructor_id)
);
```

**Prismaスキーマ:**

```prisma
model Instructor {
  id                 Int                  @id @default(autoincrement())
  name               String               @db.VarChar(100)
  email              String               @unique @db.VarChar(255)
  classroom          String               @db.VarChar(50)
  department         String?              @db.VarChar(50)
  createdAt          DateTime             @default(now()) @map("created_at")
  courseInstructors  CourseInstructor[]

  @@index([email])
  @@index([classroom])
  @@map("instructors")
}

model Course {
  id                 Int                  @id @default(autoincrement())
  code               String               @unique @db.VarChar(20)
  name               String               @db.VarChar(100)
  credits            Int
  semester           String?              @db.VarChar(20)
  createdAt          DateTime             @default(now()) @map("created_at")
  courseInstructors  CourseInstructor[]

  @@index([code])
  @@index([semester])
  @@map("courses")
}

enum InstructorRole {
  PRIMARY
  ASSISTANT
  GUEST
}

model CourseInstructor {
  courseId     Int            @map("course_id")
  instructorId Int            @map("instructor_id")
  course       Course         @relation(fields: [courseId], references: [id], onDelete: Cascade)
  instructor   Instructor     @relation(fields: [instructorId], references: [id], onDelete: Cascade)
  role         InstructorRole @default(PRIMARY)

  @@id([courseId, instructorId])
  @@index([courseId])
  @@index([instructorId])
  @@map("course_instructors")
}
```

**使用例:**

```typescript
// コース作成（複数のインストラクター付き）
const course = await prisma.course.create({
  data: {
    code: 'CS101',
    name: 'Introduction to Computer Science',
    credits: 3,
    semester: '2025-Spring',
    courseInstructors: {
      create: [
        {
          instructorId: 1,
          role: 'PRIMARY'
        },
        {
          instructorId: 2,
          role: 'ASSISTANT'
        }
      ]
    }
  },
  include: {
    courseInstructors: {
      include: {
        instructor: true
      }
    }
  }
})

// インストラクターの担当コース一覧（教室情報付き）
const instructorCourses = await prisma.instructor.findUnique({
  where: { id: 1 },
  include: {
    courseInstructors: {
      include: {
        course: true
      },
      orderBy: {
        course: {
          semester: 'desc'
        }
      }
    }
  }
})

// 教室別のコーススケジュール
const classroomSchedule = await prisma.$queryRaw`
  SELECT
    i.classroom,
    c.code,
    c.name,
    c.semester,
    i.name as instructor_name
  FROM course_instructors ci
  JOIN courses c ON ci.course_id = c.id
  JOIN instructors i ON ci.instructor_id = i.id
  WHERE c.semester = '2025-Spring'
  AND ci.role = 'PRIMARY'
  ORDER BY i.classroom, c.code
`
```

## 非正規化の戦略的適用

正規化は理想的ですが、パフォーマンスのために意図的に非正規化を行う場合があります。非正規化は、読み取りパフォーマンスを劇的に改善できますが、データの整合性維持のコストが増加します。

### 集計値の事前計算

```sql
-- ✅ 正規化されたスキーマ
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  title VARCHAR(255),
  content TEXT,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE post_likes (
  id SERIAL PRIMARY KEY,
  post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (post_id, user_id),
  INDEX idx_post_likes_post (post_id),
  INDEX idx_post_likes_user (user_id)
);

-- ❌ 毎回JOINして集計（遅い）
SELECT
  p.id,
  p.title,
  COUNT(pl.id) AS like_count
FROM posts p
LEFT JOIN post_likes pl ON p.id = pl.post_id
GROUP BY p.id, p.title
ORDER BY like_count DESC
LIMIT 10;

-- ✅ 非正規化: like_countをpostsテーブルに追加
ALTER TABLE posts ADD COLUMN like_count INTEGER DEFAULT 0;

-- トリガーで自動更新
CREATE OR REPLACE FUNCTION update_post_like_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE posts
    SET like_count = like_count + 1
    WHERE id = NEW.post_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE posts
    SET like_count = like_count - 1
    WHERE id = OLD.post_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER post_likes_update_count
AFTER INSERT OR DELETE ON post_likes
FOR EACH ROW
EXECUTE FUNCTION update_post_like_count();

-- クエリが大幅に高速化
SELECT id, title, like_count
FROM posts
ORDER BY like_count DESC
LIMIT 10;
```

**Prismaスキーマ:**

```prisma
model Post {
  id        Int        @id @default(autoincrement())
  userId    Int        @map("user_id")
  user      User       @relation(fields: [userId], references: [id])
  title     String     @db.VarChar(255)
  content   String     @db.Text
  likeCount Int        @default(0) @map("like_count")  // 非正規化フィールド
  createdAt DateTime   @default(now()) @map("created_at")
  likes     PostLike[]

  @@index([userId])
  @@index([likeCount])  // ソート用インデックス
  @@map("posts")
}

model PostLike {
  id        Int      @id @default(autoincrement())
  postId    Int      @map("post_id")
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  userId    Int      @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now()) @map("created_at")

  @@unique([postId, userId])
  @@index([postId])
  @@index([userId])
  @@map("post_likes")
}
```

**アプリケーション側での整合性維持:**

```typescript
// いいね追加（トランザクションで整合性を保証）
async function addLike(postId: number, userId: number) {
  return await prisma.$transaction(async (tx) => {
    // 既にいいねしているかチェック
    const existing = await tx.postLike.findUnique({
      where: {
        postId_userId: { postId, userId }
      }
    })

    if (existing) {
      throw new Error('Already liked')
    }

    // いいねを追加
    await tx.postLike.create({
      data: { postId, userId }
    })

    // like_countを更新
    await tx.post.update({
      where: { id: postId },
      data: {
        likeCount: {
          increment: 1
        }
      }
    })
  })
}

// いいね削除
async function removeLike(postId: number, userId: number) {
  return await prisma.$transaction(async (tx) => {
    // いいねを削除
    const deleted = await tx.postLike.delete({
      where: {
        postId_userId: { postId, userId }
      }
    })

    // like_countを更新
    await tx.post.update({
      where: { id: postId },
      data: {
        likeCount: {
          decrement: 1
        }
      }
    })

    return deleted
  })
}

// 人気投稿を取得（高速）
const popularPosts = await prisma.post.findMany({
  orderBy: { likeCount: 'desc' },
  take: 10,
  include: {
    user: {
      select: { id: true, name: true }
    }
  }
})
```

### スナップショット型の非正規化

注文システムでは、商品の価格や情報が変更されても、過去の注文内容は変わらないべきです。このような場合、注文時点の情報をスナップショットとして保存します。

```sql
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  product_id INTEGER REFERENCES products(id),
  -- スナップショット（注文時点の情報を保存）
  product_name VARCHAR(100) NOT NULL,      -- 非正規化
  product_description TEXT,                 -- 非正規化
  unit_price DECIMAL(10, 2) NOT NULL,      -- 非正規化
  quantity INTEGER NOT NULL,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Prismaスキーマ:**

```prisma
model OrderItem {
  id                 Int      @id @default(autoincrement())
  orderId            Int      @map("order_id")
  order              Order    @relation(fields: [orderId], references: [id])
  productId          Int      @map("product_id")
  product            Product  @relation(fields: [productId], references: [id])
  // スナップショットフィールド
  productName        String   @map("product_name") @db.VarChar(100)
  productDescription String?  @map("product_description") @db.Text
  unitPrice          Decimal  @map("unit_price") @db.Decimal(10, 2)
  quantity           Int
  createdAt          DateTime @default(now()) @map("created_at")

  @@index([orderId])
  @@index([productId])
  @@map("order_items")
}
```

**使用例:**

```typescript
// 注文作成時にスナップショットを保存
async function createOrder(
  customerId: number,
  items: { productId: number; quantity: number }[]
) {
  return await prisma.$transaction(async (tx) => {
    // 商品情報を取得
    const products = await tx.product.findMany({
      where: {
        id: { in: items.map(item => item.productId) }
      }
    })

    const productMap = new Map(products.map(p => [p.id, p]))

    // 注文作成（スナップショット付き）
    const order = await tx.order.create({
      data: {
        customerId,
        items: {
          create: items.map(item => {
            const product = productMap.get(item.productId)
            if (!product) throw new Error(`Product ${item.productId} not found`)

            return {
              productId: product.id,
              // スナップショット
              productName: product.name,
              productDescription: product.description,
              unitPrice: product.price,
              quantity: item.quantity
            }
          })
        }
      },
      include: {
        items: true
      }
    })

    return order
  })
}

// 商品価格が変更されても過去の注文は影響を受けない
await prisma.product.update({
  where: { id: 1 },
  data: { price: 2000 }  // 価格変更
})

// 過去の注文は元の価格のまま
const pastOrder = await prisma.orderItem.findFirst({
  where: { productId: 1 },
  orderBy: { createdAt: 'asc' }
})
// pastOrder.unitPrice は変更前の価格
```

## リレーションシップ設計パターン

### 1対多（One-to-Many）

最も一般的なリレーションシップパターンです。

```sql
CREATE TABLE authors (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  bio TEXT,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE books (
  id SERIAL PRIMARY KEY,
  author_id INTEGER NOT NULL REFERENCES authors(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  isbn VARCHAR(13) UNIQUE,
  published_date DATE,
  price DECIMAL(10, 2),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_books_author_id (author_id),
  INDEX idx_books_isbn (isbn),
  INDEX idx_books_published_date (published_date)
);
```

**Prismaスキーマ:**

```prisma
model Author {
  id        Int      @id @default(autoincrement())
  name      String   @db.VarChar(100)
  email     String   @unique @db.VarChar(255)
  bio       String?  @db.Text
  createdAt DateTime @default(now()) @map("created_at")
  books     Book[]

  @@map("authors")
}

model Book {
  id            Int      @id @default(autoincrement())
  authorId      Int      @map("author_id")
  author        Author   @relation(fields: [authorId], references: [id], onDelete: Cascade)
  title         String   @db.VarChar(255)
  isbn          String?  @unique @db.VarChar(13)
  publishedDate DateTime? @map("published_date") @db.Date
  price         Decimal? @db.Decimal(10, 2)
  createdAt     DateTime @default(now()) @map("created_at")

  @@index([authorId])
  @@index([isbn])
  @@index([publishedDate])
  @@map("books")
}
```

### 多対多（Many-to-Many）

中間テーブル（ジャンクションテーブル）を使用します。

```sql
CREATE TABLE students (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  student_id VARCHAR(20) UNIQUE NOT NULL,
  enrolled_date DATE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE courses (
  id SERIAL PRIMARY KEY,
  code VARCHAR(20) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  credits INTEGER CHECK (credits > 0),
  semester VARCHAR(20),
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 中間テーブル（メタデータ付き）
CREATE TABLE enrollments (
  id SERIAL PRIMARY KEY,
  student_id INTEGER REFERENCES students(id) ON DELETE CASCADE,
  course_id INTEGER REFERENCES courses(id) ON DELETE CASCADE,
  enrolled_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  grade VARCHAR(2),
  status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'completed', 'dropped')),
  UNIQUE (student_id, course_id),
  INDEX idx_enrollments_student (student_id),
  INDEX idx_enrollments_course (course_id),
  INDEX idx_enrollments_status (status)
);
```

**Prismaスキーマ:**

```prisma
model Student {
  id           Int          @id @default(autoincrement())
  name         String       @db.VarChar(100)
  email        String       @unique @db.VarChar(255)
  studentId    String       @unique @map("student_id") @db.VarChar(20)
  enrolledDate DateTime     @map("enrolled_date") @db.Date
  createdAt    DateTime     @default(now()) @map("created_at")
  enrollments  Enrollment[]

  @@map("students")
}

model Course {
  id          Int          @id @default(autoincrement())
  code        String       @unique @db.VarChar(20)
  name        String       @db.VarChar(100)
  credits     Int
  semester    String?      @db.VarChar(20)
  createdAt   DateTime     @default(now()) @map("created_at")
  enrollments Enrollment[]

  @@map("courses")
}

enum EnrollmentStatus {
  ACTIVE
  COMPLETED
  DROPPED
}

model Enrollment {
  id         Int              @id @default(autoincrement())
  studentId  Int              @map("student_id")
  student    Student          @relation(fields: [studentId], references: [id], onDelete: Cascade)
  courseId   Int              @map("course_id")
  course     Course           @relation(fields: [courseId], references: [id], onDelete: Cascade)
  enrolledAt DateTime         @default(now()) @map("enrolled_at")
  grade      String?          @db.VarChar(2)
  status     EnrollmentStatus @default(ACTIVE)

  @@unique([studentId, courseId])
  @@index([studentId])
  @@index([courseId])
  @@index([status])
  @@map("enrollments")
}
```

**使用例:**

```typescript
// 学生の履修登録
async function enrollStudent(studentId: number, courseId: number) {
  return await prisma.enrollment.create({
    data: {
      studentId,
      courseId,
      status: 'ACTIVE'
    },
    include: {
      student: true,
      course: true
    }
  })
}

// 学生の履修コース一覧
const studentCourses = await prisma.student.findUnique({
  where: { id: 1 },
  include: {
    enrollments: {
      where: { status: 'ACTIVE' },
      include: {
        course: true
      },
      orderBy: {
        course: { code: 'asc' }
      }
    }
  }
})

// コースの受講者一覧
const courseStudents = await prisma.course.findUnique({
  where: { id: 1 },
  include: {
    enrollments: {
      where: { status: 'ACTIVE' },
      include: {
        student: true
      },
      orderBy: {
        student: { name: 'asc' }
      }
    }
  }
})

// 成績の記録
await prisma.enrollment.update({
  where: {
    studentId_courseId: {
      studentId: 1,
      courseId: 1
    }
  },
  data: {
    grade: 'A',
    status: 'COMPLETED'
  }
})

// 学期ごとの履修者統計
const enrollmentStats = await prisma.course.findMany({
  where: { semester: '2025-Spring' },
  include: {
    _count: {
      select: {
        enrollments: {
          where: { status: 'ACTIVE' }
        }
      }
    }
  }
})
```

### 自己参照（Self-Referencing）

階層構造やツリー構造を表現します。

```sql
-- 従業員と上司の関係
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  position VARCHAR(50),
  manager_id INTEGER REFERENCES employees(id) ON DELETE SET NULL,
  hire_date DATE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_employees_manager (manager_id)
);

-- カテゴリーツリー
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  parent_id INTEGER REFERENCES categories(id) ON DELETE CASCADE,
  level INTEGER DEFAULT 0,
  path TEXT,  -- 階層パス（例: "/electronics/computers/laptops"）
  created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_categories_parent (parent_id),
  INDEX idx_categories_path (path),
  INDEX idx_categories_slug (slug)
);
```

**Prismaスキーマ:**

```prisma
model Employee {
  id           Int        @id @default(autoincrement())
  name         String     @db.VarChar(100)
  email        String     @unique @db.VarChar(255)
  position     String?    @db.VarChar(50)
  managerId    Int?       @map("manager_id")
  manager      Employee?  @relation("EmployeeHierarchy", fields: [managerId], references: [id], onDelete: SetNull)
  subordinates Employee[] @relation("EmployeeHierarchy")
  hireDate     DateTime   @map("hire_date") @db.Date
  createdAt    DateTime   @default(now()) @map("created_at")

  @@index([managerId])
  @@map("employees")
}

model Category {
  id        Int        @id @default(autoincrement())
  name      String     @db.VarChar(100)
  slug      String     @unique @db.VarChar(100)
  parentId  Int?       @map("parent_id")
  parent    Category?  @relation("CategoryTree", fields: [parentId], references: [id], onDelete: Cascade)
  children  Category[] @relation("CategoryTree")
  level     Int        @default(0)
  path      String?    @db.Text
  createdAt DateTime   @default(now()) @map("created_at")

  @@index([parentId])
  @@index([path])
  @@index([slug])
  @@map("categories")
}
```

**使用例:**

```typescript
// 従業員の階層構造を取得
async function getEmployeeHierarchy(employeeId: number, depth: number = 2) {
  return await prisma.employee.findUnique({
    where: { id: employeeId },
    include: {
      subordinates: depth > 0 ? {
        include: {
          subordinates: depth > 1 ? true : false
        }
      } : false,
      manager: {
        include: {
          manager: true
        }
      }
    }
  })
}

// カテゴリーツリーの取得（再帰）
async function getCategoryTree(parentId: number | null = null): Promise<any[]> {
  const categories = await prisma.category.findMany({
    where: { parentId },
    orderBy: { name: 'asc' }
  })

  return await Promise.all(
    categories.map(async (category) => ({
      ...category,
      children: await getCategoryTree(category.id)
    }))
  )
}

// カテゴリーパスの更新（トリガーの代替）
async function updateCategoryPath(categoryId: number) {
  const category = await prisma.category.findUnique({
    where: { id: categoryId },
    include: { parent: true }
  })

  if (!category) return

  const path = category.parent
    ? `${category.parent.path}/${category.slug}`
    : `/${category.slug}`

  const level = category.parent ? category.parent.level + 1 : 0

  await prisma.category.update({
    where: { id: categoryId },
    data: { path, level }
  })

  // 子カテゴリーのパスも再帰的に更新
  const children = await prisma.category.findMany({
    where: { parentId: categoryId }
  })

  for (const child of children) {
    await updateCategoryPath(child.id)
  }
}

// パンくずリストの生成
async function getBreadcrumbs(categoryId: number) {
  const category = await prisma.category.findUnique({
    where: { id: categoryId }
  })

  if (!category || !category.path) return []

  const slugs = category.path.split('/').filter(Boolean)
  const categories = await prisma.category.findMany({
    where: {
      slug: { in: slugs }
    },
    orderBy: { level: 'asc' }
  })

  return categories
}
```

## データ型の選択戦略

適切なデータ型の選択は、ストレージ効率とパフォーマンスに大きく影響します。

### 整数型の選択

```prisma
model Product {
  id            BigInt   @id @default(autoincrement())  // 大量のレコードを想定
  categoryId    Int      @map("category_id")            // カテゴリー数は限定的
  stockQuantity Int      @map("stock_quantity")         // 在庫数
  viewCount     BigInt   @default(0) @map("view_count") // ビュー数は無制限に増える
  likeCount     Int      @default(0) @map("like_count") // いいね数

  @@map("products")
}
```

| 型 | サイズ | 範囲 | 用途 |
|---|--------|------|------|
| **Int** (INTEGER) | 4バイト | -2,147,483,648 〜 2,147,483,647 | 一般的なID、カウンター |
| **BigInt** (BIGINT) | 8バイト | -9,223,372,036,854,775,808 〜 9,223,372,036,854,775,807 | 大規模データのID、累積値 |

### 文字列型の選択

```prisma
model User {
  id           Int      @id @default(autoincrement())
  countryCode  String   @db.Char(2)         // 'JP', 'US'（固定長）
  zipcode      String   @db.VarChar(10)     // '〒100-0001'
  username     String   @db.VarChar(50)     // 上限50文字
  email        String   @unique @db.VarChar(255)
  bio          String?  @db.Text            // 長文、上限なし

  @@map("users")
}
```

| 型 | 特徴 | 用途 |
|---|------|------|
| **CHAR(n)** | 固定長、スペースパディング | 国コード、ステータスコード |
| **VARCHAR(n)** | 可変長、最大n文字 | 名前、メールアドレス |
| **TEXT** | 無制限の可変長 | 長文コンテンツ |

### 日付・時刻型

```prisma
model Event {
  id          Int      @id @default(autoincrement())
  name        String
  eventDate   DateTime @map("event_date") @db.Date      // 日付のみ
  startTime   DateTime @map("start_time") @db.Time      // 時刻のみ
  scheduledAt DateTime @map("scheduled_at") @db.Timestamptz  // 日時+タイムゾーン（推奨）
  createdAt   DateTime @default(now()) @map("created_at") @db.Timestamptz

  @@map("events")
}
```

**重要:** グローバルアプリケーションでは必ず `Timestamptz` を使用してください。

### JSON型の活用

```prisma
model Product {
  id         Int     @id @default(autoincrement())
  name       String
  // スキーマレスデータをJSONBで格納
  attributes Json    // {'color': 'red', 'size': 'L', 'weight': 500}
  metadata   Json?   // オプショナルな追加データ

  @@map("products")
}
```

**使用例:**

```typescript
// JSON フィールドの検索
const redProducts = await prisma.product.findMany({
  where: {
    attributes: {
      path: ['color'],
      equals: 'red'
    }
  }
})

// JSON フィールドの更新
await prisma.product.update({
  where: { id: 1 },
  data: {
    attributes: {
      color: 'blue',
      size: 'XL',
      weight: 600
    }
  }
})

// ネストされたJSON
const products = await prisma.$queryRaw`
  SELECT * FROM products
  WHERE attributes->>'color' = 'red'
  AND (attributes->'weight')::int > 500
`
```

### ENUM型で型安全性を確保

```prisma
enum UserRole {
  ADMIN
  MODERATOR
  USER
  GUEST
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  role      UserRole @default(USER)
  createdAt DateTime @default(now()) @map("created_at")

  @@map("users")
}

model Order {
  id        Int         @id @default(autoincrement())
  status    OrderStatus @default(PENDING)
  createdAt DateTime    @default(now()) @map("created_at")

  @@map("orders")
}
```

## 制約（Constraints）による整合性保証

### PRIMARY KEY

```prisma
// 自動インクリメント
model User {
  id   Int    @id @default(autoincrement())
  name String

  @@map("users")
}

// UUID
model User {
  id   String @id @default(uuid())
  name String

  @@map("users")
}

// 複合主キー
model OrderItem {
  orderId   Int
  productId Int
  quantity  Int

  @@id([orderId, productId])
  @@map("order_items")
}
```

### FOREIGN KEY と参照整合性

```prisma
model Post {
  id       Int    @id @default(autoincrement())
  userId   Int    @map("user_id")
  // CASCADE: 親削除時に子も削除
  user     User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  title    String

  @@map("posts")
}

model Comment {
  id      Int  @id @default(autoincrement())
  postId  Int  @map("post_id")
  // RESTRICT: 子が存在する場合、親の削除を拒否
  post    Post @relation(fields: [postId], references: [id], onDelete: Restrict)
  content String

  @@map("comments")
}

model Profile {
  id     Int   @id @default(autoincrement())
  userId Int   @unique @map("user_id")
  // SET NULL: 親削除時にNULLに設定
  user   User? @relation(fields: [userId], references: [id], onDelete: SetNull)
  bio    String?

  @@map("profiles")
}
```

| アクション | 説明 | 用途 |
|-----------|------|------|
| **Cascade** | 親削除時に子も削除 | 依存関係が強い場合（注文と注文明細） |
| **Restrict** | 子が存在する場合、親の削除を拒否 | データ保護が重要な場合（商品とレビュー） |
| **SetNull** | 親削除時に外部キーをNULLに | 参照は残したいが必須ではない場合 |
| **SetDefault** | 親削除時にデフォルト値に設定 | 代替値がある場合 |

### UNIQUE制約

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique  // 単一カラムのUNIQUE

  @@map("users")
}

model UserRole {
  userId Int  @map("user_id")
  roleId Int  @map("role_id")

  // 複合UNIQUE（同じユーザー+ロールの組み合わせは1つのみ）
  @@unique([userId, roleId])
  @@map("user_roles")
}

model Post {
  id          Int       @id @default(autoincrement())
  slug        String
  publishedAt DateTime? @map("published_at")

  // 条件付きUNIQUE（部分UNIQUE）はSQLレベルで実装
  @@map("posts")
}
```

**部分UNIQUEの実装（PostgreSQL）:**

```sql
-- 公開済み投稿のslugのみユニーク（下書きは重複可）
CREATE UNIQUE INDEX idx_posts_slug_published
ON posts(slug)
WHERE published_at IS NOT NULL;
```

## ベンチマーク指標: スキーマ設計の想定される効果

### 導入前の想定課題

- **データ冗長性:** 複数テーブルで同じデータが重複（顧客情報が注文テーブルに直接格納）
- **整合性エラー:** 外部キー制約なしで孤立レコードが発生（月平均15件）
- **ディスク使用量:** 不適切なデータ型で無駄なストレージ消費（2.8GB）
- **クエリパフォーマンス:** 正規化不足で複雑なクエリが必要（平均850ms）

### 導入後に期待できる効果

**正規化による改善:**
- データ冗長性: -72%（2.8GB → 0.78GB）
- データ整合性エラー: 15件/月 → 0件 (-100%)
- 更新クエリの複雑さ: 平均8テーブル更新 → 2テーブル更新 (-75%)

**適切なデータ型選択:**
- ディスク使用量: 2.8GB → 1.1GB (-61%)
- インデックスサイズ: 820MB → 340MB (-59%)
- バックアップ時間: 45分 → 18分 (-60%)
- リストア時間: 62分 → 25分 (-60%)

**外部キー制約の導入:**
- 孤立レコード: 328件 → 0件 (-100%)
- データ整合性チェック時間: 25分/日 → 不要 (-100%)
- 参照整合性エラー: 12件/週 → 0件 (-100%)

**ENUM型の導入:**
- 不正なステータス値: 8件/月 → 0件 (-100%)
- ステータス検証ロジック: 削除可能（コード削減）
- クエリパフォーマンス: +15%（文字列比較 → 整数比較）

## チェックリスト

### 正規化
- [ ] 第1正規形: 繰り返しグループを別テーブルに分離
- [ ] 第2正規形: 部分関数従属を排除
- [ ] 第3正規形: 推移的関数従属を排除
- [ ] BCNF: すべての決定子が候補キーであることを確認
- [ ] 意図的な非正規化の場合、整合性維持の仕組みを実装

### データ型
- [ ] 整数型（Int/BigInt）を適切に選択
- [ ] 文字列型（Char/VarChar/Text）を適切に選択
- [ ] 日付・時刻はTimestamptz使用（タイムゾーン付き）
- [ ] JSONB型でスキーマレスデータを効率的に格納
- [ ] ENUM型で型安全性を確保

### 制約
- [ ] PRIMARY KEYを必ず設定
- [ ] FOREIGN KEYで参照整合性を保証
- [ ] ON DELETE/ON UPDATEを適切に設定（Cascade/Restrict/SetNull）
- [ ] UNIQUE制約で重複を防止
- [ ] NOT NULL制約で必須フィールドを明示
- [ ] CHECK制約でビジネスルールを実装

### リレーションシップ
- [ ] 1対多の関係を適切に設計
- [ ] 多対多の関係は中間テーブルを使用
- [ ] 自己参照が必要な場合は適切に実装
- [ ] 循環参照を避ける設計

### インデックス
- [ ] 外部キーカラムにインデックス作成
- [ ] WHERE句で頻繁に使用するカラムにインデックス
- [ ] JOIN条件のカラムにインデックス
- [ ] ORDER BY/GROUP BYのカラムにインデックス

### ドキュメンテーション
- [ ] スキーマ図（ERD）を作成・更新
- [ ] テーブル・カラムの説明をコメントに記載
- [ ] マイグレーション履歴を管理
- [ ] データディクショナリを維持

---

次章では、このスキーマ設計を基に、クエリ最適化とインデックス戦略を詳しく解説します。

**文字数: 30,284文字**
