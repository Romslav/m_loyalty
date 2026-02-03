# 🧪 TESTING GUIDE - MAX-LOYALTY

## 📋 СОДЕРЖАНИЕ

1. [Стратегия тестирования](#стратегия-тестирования)
2. [Unit Testing](#unit-testing)
3. [Integration Testing](#integration-testing)
4. [E2E Testing](#e2e-testing)
5. [API Testing](#api-testing)
6. [Performance Testing](#performance-testing)
7. [Security Testing](#security-testing)
8. [Test Coverage](#test-coverage)
9. [CI/CD Integration](#cicd-integration)

---

## СТРАТЕГИЯ ТЕСТИРОВАНИЯ

### Testing Pyramid

```
           /\
          /  \          E2E Tests (10%)
         /____\         - Critical user flows
        /      \        - UI interactions
       /________\       - Full system tests
      /          \      
     / Integration \    Integration Tests (30%)
    /     Tests     \   - API endpoints
   /________________\  - Service interactions
  /                  \  - Database operations
 /                    \ 
/    Unit Tests (60%)  \ Unit Tests (60%)
________________________ - Pure functions
                         - Business logic
                         - Individual components
```

### Test Types

| Type | Purpose | Tools | Coverage Target |
|------|---------|-------|----------------|
| **Unit** | Тестирование отдельных функций/классов | Jest | 80%+ |
| **Integration** | Тестирование взаимодействия компонентов | Jest + Supertest | 70%+ |
| **E2E** | Тестирование полных сценариев | Playwright | Critical flows |
| **API** | Тестирование REST API | Postman/Newman | All endpoints |
| **Performance** | Нагрузочное тестирование | k6, Artillery | Key endpoints |
| **Security** | Тестирование уязвимостей | OWASP ZAP | All attack vectors |

---

## UNIT TESTING

### Setup

```bash
# Install dependencies
pnpm add -D @nestjs/testing jest @types/jest ts-jest
```

### Jest Configuration

```javascript
// jest.config.js
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: [
    '**/*.(t|j)s',
    '!**/*.spec.ts',
    '!**/node_modules/**',
    '!**/dist/**',
  ],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
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

### Service Unit Tests

```typescript
// loyalty/loyalty.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { LoyaltyService } from './loyalty.service';
import { PrismaService } from '../prisma/prisma.service';
import { LoyaltyType } from '@prisma/client';

describe('LoyaltyService', () => {
  let service: LoyaltyService;
  let prisma: PrismaService;

  const mockPrismaService = {
    loyaltyProgram: {
      create: jest.fn(),
      findUnique: jest.fn(),
      update: jest.fn(),
    },
    loyaltyCard: {
      create: jest.fn(),
      findUnique: jest.fn(),
    },
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        LoyaltyService,
        {
          provide: PrismaService,
          useValue: mockPrismaService,
        },
      ],
    }).compile();

    service = module.get<LoyaltyService>(LoyaltyService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('createLoyaltyProgram', () => {
    it('should create a points-based loyalty program', async () => {
      const createDto = {
        name: 'Test Program',
        type: LoyaltyType.POINTS,
        businessId: 'business-123',
        settings: {
          pointsPerRuble: 1,
          minPointsForReward: 100,
        },
      };

      const expectedProgram = {
        id: 'program-123',
        ...createDto,
        createdAt: new Date(),
      };

      mockPrismaService.loyaltyProgram.create.mockResolvedValue(expectedProgram);

      const result = await service.createLoyaltyProgram(createDto);

      expect(result).toEqual(expectedProgram);
      expect(mockPrismaService.loyaltyProgram.create).toHaveBeenCalledWith({
        data: createDto,
      });
    });

    it('should throw error for invalid business', async () => {
      mockPrismaService.loyaltyProgram.create.mockRejectedValue(
        new Error('Business not found'),
      );

      await expect(
        service.createLoyaltyProgram({
          name: 'Test',
          type: LoyaltyType.POINTS,
          businessId: 'invalid',
        }),
      ).rejects.toThrow('Business not found');
    });
  });

  describe('calculatePoints', () => {
    it('should calculate points correctly for purchase', () => {
      const amount = 1000; // 1000 rubles
      const pointsPerRuble = 1;

      const result = service.calculatePoints(amount, pointsPerRuble);

      expect(result).toBe(1000);
    });

    it('should handle fractional amounts', () => {
      const amount = 1550.50;
      const pointsPerRuble = 1;

      const result = service.calculatePoints(amount, pointsPerRuble);

      expect(result).toBe(1551); // Rounded up
    });

    it('should apply multiplier correctly', () => {
      const amount = 1000;
      const pointsPerRuble = 2;

      const result = service.calculatePoints(amount, pointsPerRuble);

      expect(result).toBe(2000);
    });
  });

  describe('isCardValid', () => {
    it('should return true for active card', () => {
      const card = {
        id: 'card-123',
        status: 'ACTIVE',
        expiresAt: new Date(Date.now() + 86400000), // Tomorrow
      };

      const result = service.isCardValid(card);

      expect(result).toBe(true);
    });

    it('should return false for expired card', () => {
      const card = {
        id: 'card-123',
        status: 'ACTIVE',
        expiresAt: new Date(Date.now() - 86400000), // Yesterday
      };

      const result = service.isCardValid(card);

      expect(result).toBe(false);
    });

    it('should return false for blocked card', () => {
      const card = {
        id: 'card-123',
        status: 'BLOCKED',
        expiresAt: new Date(Date.now() + 86400000),
      };

      const result = service.isCardValid(card);

      expect(result).toBe(false);
    });
  });
});
```

### Controller Unit Tests

```typescript
// loyalty/loyalty.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { LoyaltyController } from './loyalty.controller';
import { LoyaltyService } from './loyalty.service';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';

describe('LoyaltyController', () => {
  let controller: LoyaltyController;
  let service: LoyaltyService;

  const mockLoyaltyService = {
    createLoyaltyProgram: jest.fn(),
    getLoyaltyProgram: jest.fn(),
    updateLoyaltyProgram: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [LoyaltyController],
      providers: [
        {
          provide: LoyaltyService,
          useValue: mockLoyaltyService,
        },
      ],
    })
      .overrideGuard(JwtAuthGuard)
      .useValue({ canActivate: jest.fn(() => true) })
      .overrideGuard(RolesGuard)
      .useValue({ canActivate: jest.fn(() => true) })
      .compile();

    controller = module.get<LoyaltyController>(LoyaltyController);
    service = module.get<LoyaltyService>(LoyaltyService);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  describe('createProgram', () => {
    it('should create loyalty program', async () => {
      const createDto = {
        name: 'Test Program',
        type: 'POINTS',
        businessId: 'business-123',
      };

      const expectedResult = {
        id: 'program-123',
        ...createDto,
      };

      mockLoyaltyService.createLoyaltyProgram.mockResolvedValue(expectedResult);

      const result = await controller.createProgram(createDto);

      expect(result).toEqual(expectedResult);
      expect(service.createLoyaltyProgram).toHaveBeenCalledWith(createDto);
    });
  });
});
```

### Run Unit Tests

```bash
# Run all unit tests
pnpm test

# Run with coverage
pnpm test:cov

# Run in watch mode
pnpm test:watch

# Run specific test file
pnpm test loyalty.service.spec.ts
```

---

## INTEGRATION TESTING

### Setup Test Database

```typescript
// test/setup.ts
import { PrismaClient } from '@prisma/client';
import { execSync } from 'child_process';

const prisma = new PrismaClient();

beforeAll(async () => {
  // Reset database
  execSync('pnpm prisma:migrate:reset --force --skip-seed', {
    env: {
      ...process.env,
      DATABASE_URL: process.env.DATABASE_TEST_URL,
    },
  });

  // Run migrations
  execSync('pnpm prisma:migrate:deploy', {
    env: {
      ...process.env,
      DATABASE_URL: process.env.DATABASE_TEST_URL,
    },
  });
});

afterAll(async () => {
  await prisma.$disconnect();
});

afterEach(async () => {
  // Clean up data after each test
  const tables = [
    'Transaction',
    'LoyaltyCard',
    'LoyaltyProgram',
    'Customer',
    'Business',
  ];

  for (const table of tables) {
    await prisma.$executeRawUnsafe(`TRUNCATE TABLE "${table}" CASCADE;`);
  }
});
```

### Integration Test Example

```typescript
// test/loyalty.integration.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { PrismaService } from '../src/prisma/prisma.service';

describe('Loyalty Integration Tests', () => {
  let app: INestApplication;
  let prisma: PrismaService;
  let authToken: string;
  let businessId: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    prisma = app.get<PrismaService>(PrismaService);
    
    await app.init();

    // Create test business and get auth token
    const business = await prisma.business.create({
      data: {
        name: 'Test Business',
        email: 'test@business.com',
        phone: '+79001234567',
      },
    });
    businessId = business.id;

    // Login and get token
    const loginResponse = await request(app.getHttpServer())
      .post('/auth/login')
      .send({
        email: 'test@business.com',
        password: 'testpassword',
      });

    authToken = loginResponse.body.accessToken;
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /loyalty/programs', () => {
    it('should create a loyalty program', async () => {
      const createDto = {
        name: 'Test Loyalty Program',
        type: 'POINTS',
        businessId,
        settings: {
          pointsPerRuble: 1,
          minPointsForReward: 100,
        },
      };

      const response = await request(app.getHttpServer())
        .post('/loyalty/programs')
        .set('Authorization', `Bearer ${authToken}`)
        .send(createDto)
        .expect(201);

      expect(response.body).toMatchObject({
        name: createDto.name,
        type: createDto.type,
        businessId,
      });

      // Verify in database
      const program = await prisma.loyaltyProgram.findUnique({
        where: { id: response.body.id },
      });

      expect(program).toBeDefined();
      expect(program.name).toBe(createDto.name);
    });

    it('should return 400 for invalid data', async () => {
      await request(app.getHttpServer())
        .post('/loyalty/programs')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          name: '', // Invalid: empty name
          type: 'INVALID_TYPE',
        })
        .expect(400);
    });

    it('should return 401 without auth token', async () => {
      await request(app.getHttpServer())
        .post('/loyalty/programs')
        .send({ name: 'Test' })
        .expect(401);
    });
  });

  describe('POST /loyalty/cards', () => {
    let programId: string;

    beforeEach(async () => {
      // Create loyalty program first
      const program = await prisma.loyaltyProgram.create({
        data: {
          name: 'Test Program',
          type: 'POINTS',
          businessId,
        },
      });
      programId = program.id;
    });

    it('should create loyalty card for customer', async () => {
      const createDto = {
        programId,
        customerId: 'customer-123',
      };

      const response = await request(app.getHttpServer())
        .post('/loyalty/cards')
        .set('Authorization', `Bearer ${authToken}`)
        .send(createDto)
        .expect(201);

      expect(response.body).toMatchObject({
        programId,
        customerId: createDto.customerId,
        status: 'ACTIVE',
        balance: 0,
      });
    });
  });

  describe('POST /loyalty/transactions', () => {
    let cardId: string;

    beforeEach(async () => {
      // Setup program and card
      const program = await prisma.loyaltyProgram.create({
        data: {
          name: 'Test Program',
          type: 'POINTS',
          businessId,
        },
      });

      const card = await prisma.loyaltyCard.create({
        data: {
          programId: program.id,
          customerId: 'customer-123',
          status: 'ACTIVE',
          balance: 0,
        },
      });
      cardId = card.id;
    });

    it('should process purchase and add points', async () => {
      const transactionDto = {
        cardId,
        type: 'EARN',
        amount: 1000, // 1000 rubles
        description: 'Test purchase',
      };

      const response = await request(app.getHttpServer())
        .post('/loyalty/transactions')
        .set('Authorization', `Bearer ${authToken}`)
        .send(transactionDto)
        .expect(201);

      expect(response.body.points).toBeGreaterThan(0);

      // Verify card balance updated
      const card = await prisma.loyaltyCard.findUnique({
        where: { id: cardId },
      });

      expect(card.balance).toBe(response.body.points);
    });
  });
});
```

---

## E2E TESTING

### Playwright Setup

```bash
# Install Playwright
pnpm add -D @playwright/test
px playwright install
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13'] },
    },
  ],
  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```

### E2E Test Example

```typescript
// e2e/customer-journey.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Customer Loyalty Journey', () => {
  test.beforeEach(async ({ page }) => {
    // Navigate to app
    await page.goto('/');
  });

  test('should complete full customer journey', async ({ page }) => {
    // 1. Open Telegram Mini App
    await expect(page.locator('h1')).toContainText('MAX-LOYALTY');

    // 2. Customer scans QR code at cafe
    await page.click('button:has-text("Scan QR")');
    
    // Simulate QR code scan
    await page.fill('input[name="qrCode"]', 'CAFE123');
    await page.click('button:has-text("Submit")');

    // 3. View loyalty card
    await expect(page.locator('.loyalty-card')).toBeVisible();
    await expect(page.locator('.card-balance')).toContainText('0 points');

    // 4. Make purchase
    await page.click('button:has-text("Make Purchase")');
    await page.fill('input[name="amount"]', '500');
    await page.click('button:has-text("Pay")');

    // 5. Verify points earned
    await expect(page.locator('.success-message')).toContainText('Points earned!');
    await expect(page.locator('.card-balance')).toContainText('500 points');

    // 6. View transaction history
    await page.click('a:has-text("History")');
    await expect(page.locator('.transaction-list')).toBeVisible();
    await expect(page.locator('.transaction-item').first()).toContainText('500');

    // 7. Check available rewards
    await page.click('a:has-text("Rewards")');
    await expect(page.locator('.rewards-list')).toBeVisible();

    // 8. Redeem reward (if enough points)
    const rewardButton = page.locator('button:has-text("Redeem")').first();
    if (await rewardButton.isEnabled()) {
      await rewardButton.click();
      await expect(page.locator('.reward-success')).toBeVisible();
    }
  });

  test('should handle offline mode', async ({ page, context }) => {
    // Go offline
    await context.setOffline(true);

    await page.goto('/');

    // Should show offline indicator
    await expect(page.locator('.offline-indicator')).toBeVisible();

    // Go back online
    await context.setOffline(false);

    // Should sync data
    await expect(page.locator('.sync-indicator')).toContainText('Synced');
  });

  test('should validate form inputs', async ({ page }) => {
    await page.click('button:has-text("Make Purchase")');

    // Try to submit empty form
    await page.click('button:has-text("Pay")');
    await expect(page.locator('.error-message')).toContainText('Amount is required');

    // Try invalid amount
    await page.fill('input[name="amount"]', '-100');
    await page.click('button:has-text("Pay")');
    await expect(page.locator('.error-message')).toContainText('Invalid amount');
  });
});
```

### Run E2E Tests

```bash
# Run all E2E tests
pnpm exec playwright test

# Run with UI
pnpm exec playwright test --ui

# Run specific test
pnpm exec playwright test customer-journey

# Debug test
pnpm exec playwright test --debug
```

---

## API TESTING

### Postman Collection

```json
// postman/max-loyalty.postman_collection.json
{
  "info": {
    "name": "MAX-LOYALTY API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Authentication",
      "item": [
        {
          "name": "Login",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"test@example.com\",\n  \"password\": \"password123\"\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/auth/login",
              "host": ["{{baseUrl}}"],
              "path": ["auth", "login"]
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status code is 200', function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "",
                  "pm.test('Response has accessToken', function () {",
                  "    var jsonData = pm.response.json();",
                  "    pm.expect(jsonData).to.have.property('accessToken');",
                  "    pm.environment.set('authToken', jsonData.accessToken);",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ]
        }
      ]
    },
    {
      "name": "Loyalty Programs",
      "item": [
        {
          "name": "Create Loyalty Program",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Authorization",
                "value": "Bearer {{authToken}}"
              },
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"name\": \"Test Program\",\n  \"type\": \"POINTS\",\n  \"businessId\": \"{{businessId}}\"\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/loyalty/programs",
              "host": ["{{baseUrl}}"],
              "path": ["loyalty", "programs"]
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status code is 201', function () {",
                  "    pm.response.to.have.status(201);",
                  "});",
                  "",
                  "pm.test('Response has program id', function () {",
                  "    var jsonData = pm.response.json();",
                  "    pm.expect(jsonData).to.have.property('id');",
                  "    pm.environment.set('programId', jsonData.id);",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ]
        },
        {
          "name": "Get Loyalty Program",
          "request": {
            "method": "GET",
            "header": [
              {
                "key": "Authorization",
                "value": "Bearer {{authToken}}"
              }
            ],
            "url": {
              "raw": "{{baseUrl}}/loyalty/programs/{{programId}}",
              "host": ["{{baseUrl}}"],
              "path": ["loyalty", "programs", "{{programId}}"]
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status code is 200', function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "",
                  "pm.test('Response matches schema', function () {",
                  "    var jsonData = pm.response.json();",
                  "    pm.expect(jsonData).to.have.property('id');",
                  "    pm.expect(jsonData).to.have.property('name');",
                  "    pm.expect(jsonData).to.have.property('type');",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ]
        }
      ]
    }
  ]
}
```

### Newman CLI Testing

```bash
# Install Newman
pnpm add -D newman newman-reporter-html

# Run collection
newman run postman/max-loyalty.postman_collection.json \
  --environment postman/environment.json \
  --reporters cli,html \
  --reporter-html-export newman-report.html
```

---

## PERFORMANCE TESTING

### k6 Load Testing

```javascript
// k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200 users
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests under 500ms
    http_req_failed: ['rate<0.01'],   // Error rate under 1%
    errors: ['rate<0.1'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export function setup() {
  // Login and get token
  const loginRes = http.post(`${BASE_URL}/auth/login`, JSON.stringify({
    email: 'test@example.com',
    password: 'password123',
  }), {
    headers: { 'Content-Type': 'application/json' },
  });

  return { token: loginRes.json('accessToken') };
}

export default function(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  // Test 1: Get loyalty programs
  let res = http.get(`${BASE_URL}/loyalty/programs`, { headers });
  check(res, {
    'programs status is 200': (r) => r.status === 200,
    'programs response time OK': (r) => r.timings.duration < 500,
  }) || errorRate.add(1);

  sleep(1);

  // Test 2: Create transaction
  res = http.post(
    `${BASE_URL}/loyalty/transactions`,
    JSON.stringify({
      cardId: 'test-card-id',
      type: 'EARN',
      amount: 1000,
    }),
    { headers }
  );
  check(res, {
    'transaction status is 201': (r) => r.status === 201,
    'transaction response time OK': (r) => r.timings.duration < 1000,
  }) || errorRate.add(1);

  sleep(1);
}

export function teardown(data) {
  // Cleanup if needed
}
```

### Run Load Tests

```bash
# Install k6
brew install k6  # macOS
# or
sudo apt-get install k6  # Ubuntu

# Run load test
k6 run k6/load-test.js

# Run with custom parameters
k6 run --vus 100 --duration 30s k6/load-test.js

# Run with cloud reporting
k6 run --out cloud k6/load-test.js
```

---

## SECURITY TESTING

### OWASP ZAP Automated Scan

```bash
# Pull OWASP ZAP Docker image
docker pull owasp/zap2docker-stable

# Run baseline scan
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t http://localhost:3000 \
  -r zap-report.html

# Run full scan
docker run -t owasp/zap2docker-stable zap-full-scan.py \
  -t http://localhost:3000 \
  -r zap-full-report.html
```

### Security Test Cases

```typescript
// test/security/auth.security.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../../src/app.module';

describe('Security Tests - Authentication', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('SQL Injection Prevention', () => {
    it('should prevent SQL injection in login', async () => {
      const maliciousPayloads = [
        { email: "admin' OR '1'='1", password: 'test' },
        { email: "admin'--", password: 'test' },
        { email: "admin' /*", password: 'test' },
      ];

      for (const payload of maliciousPayloads) {
        const response = await request(app.getHttpServer())
          .post('/auth/login')
          .send(payload);

        expect(response.status).not.toBe(200);
        expect(response.body.accessToken).toBeUndefined();
      }
    });
  });

  describe('XSS Prevention', () => {
    it('should sanitize HTML in inputs', async () => {
      const xssPayload = {
        name: '<script>alert("XSS")</script>',
        email: 'test@example.com',
      };

      const response = await request(app.getHttpServer())
        .post('/customers')
        .send(xssPayload);

      if (response.status === 201) {
        expect(response.body.name).not.toContain('<script>');
      }
    });
  });

  describe('Rate Limiting', () => {
    it('should block after too many requests', async () => {
      const requests = [];
      
      // Send 20 requests rapidly
      for (let i = 0; i < 20; i++) {
        requests.push(
          request(app.getHttpServer())
            .post('/auth/login')
            .send({ email: 'test@example.com', password: 'wrong' })
        );
      }

      const responses = await Promise.all(requests);
      const blocked = responses.some(r => r.status === 429);

      expect(blocked).toBe(true);
    });
  });

  describe('JWT Security', () => {
    it('should reject expired tokens', async () => {
      const expiredToken = 'expired.jwt.token';

      const response = await request(app.getHttpServer())
        .get('/loyalty/programs')
        .set('Authorization', `Bearer ${expiredToken}`);

      expect(response.status).toBe(401);
    });

    it('should reject invalid tokens', async () => {
      const invalidToken = 'invalid.token.here';

      const response = await request(app.getHttpServer())
        .get('/loyalty/programs')
        .set('Authorization', `Bearer ${invalidToken}`);

      expect(response.status).toBe(401);
    });
  });
});
```

---

## TEST COVERAGE

### Generate Coverage Report

```bash
# Run tests with coverage
pnpm test:cov

# View coverage report
open coverage/lcov-report/index.html
```

### Coverage Thresholds

```javascript
// jest.config.js
module.exports = {
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    './src/loyalty/**/*.ts': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90,
    },
  },
};
```

---

## CI/CD INTEGRATION

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install pnpm
        run: npm install -g pnpm
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Run unit tests
        run: pnpm test:cov
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
  
  integration-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: maxloyalty_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install pnpm
        run: npm install -g pnpm
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Run migrations
        run: pnpm prisma:migrate:deploy
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/maxloyalty_test
      
      - name: Run integration tests
        run: pnpm test:e2e
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/maxloyalty_test
          REDIS_URL: redis://localhost:6379
  
  e2e-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install pnpm
        run: npm install -g pnpm
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Install Playwright
        run: pnpm exec playwright install --with-deps
      
      - name: Run E2E tests
        run: pnpm exec playwright test
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

---

**Версия:** 1.0  
**Дата:** 2026-02-03  
**Статус:** Готово к использованию
