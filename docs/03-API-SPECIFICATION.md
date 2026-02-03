# 🔌 API SPECIFICATION - MAX-LOYALTY

## 📋 СОДЕРЖАНИЕ

1. [Общие положения](#общие-положения)
2. [Authentication & Authorization](#authentication--authorization)
3. [Users & Roles](#users--roles)
4. [Tenants & Subscriptions](#tenants--subscriptions)
5. [Restaurants](#restaurants)
6. [Guests & Loyalty Cards](#guests--loyalty-cards)
7. [Ball Transactions](#ball-transactions)
8. [Loyalty System](#loyalty-system)
9. [Analytics & Reports](#analytics--reports)
10. [POS Integration](#pos-integration)
11. [Telegram Integration](#telegram-integration)

---

## ОБЩИЕ ПОЛОЖЕНИЯ

### Base URL
```
Production: https://api.max-loyalty.com
Staging: https://api-staging.max-loyalty.com
Development: http://localhost:3000
```

### Аутентификация

Все запросы (кроме `/auth/*`) требуют JWT токен:

```http
Authorization: Bearer <access_token>
```

### Headers

```http
Content-Type: application/json
Accept: application/json
X-Tenant-ID: <tenant_uuid>      # Required for multi-tenant operations
X-Restaurant-ID: <restaurant_uuid>  # Optional, for restaurant-specific ops
```

### Стандартные коды ответов

| Code | Description |
|------|-------------|
| 200 | Success |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 409 | Conflict |
| 422 | Unprocessable Entity |
| 429 | Too Many Requests |
| 500 | Internal Server Error |

### Формат ошибок

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "error": "Bad Request",
  "details": [
    {
      "field": "email",
      "message": "Invalid email format"
    }
  ]
}
```

### Пагинация

```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 30,
    "total": 150,
    "totalPages": 5
  },
  "links": {
    "first": "/api/guests?page=1&limit=30",
    "prev": null,
    "next": "/api/guests?page=2&limit=30",
    "last": "/api/guests?page=5&limit=30"
  }
}
```

---

## AUTHENTICATION & AUTHORIZATION

### Register Owner

```http
POST /api/auth/register/owner
```

**Request:**
```json
{
  "email": "owner@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe",
  "phone": "+79001234567",
  "companyName": "My Restaurant Chain",
  "planType": "STANDARD"
}
```

**Response:** `201 Created`
```json
{
  "user": {
    "id": "uuid",
    "email": "owner@example.com",
    "role": "OWNER",
    "emailVerified": false
  },
  "tenant": {
    "id": "uuid",
    "name": "My Restaurant Chain",
    "slug": "my-restaurant-chain"
  },
  "subscription": {
    "id": "uuid",
    "planType": "STANDARD",
    "status": "TRIAL",
    "trialEnd": "2026-03-05T00:00:00Z"
  },
  "tokens": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900
  }
}
```

### Login

```http
POST /api/auth/login
```

**Request:**
```json
{
  "login": "owner@example.com",  // email or phone
  "password": "SecurePass123!"
}
```

**Response:** `200 OK`
```json
{
  "user": {
    "id": "uuid",
    "email": "owner@example.com",
    "role": "OWNER",
    "tenantId": "uuid",
    "permissions": ["read:analytics", "write:restaurants", ...]
  },
  "tokens": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900
  }
}
```

### Refresh Token

```http
POST /api/auth/refresh
```

**Request:**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response:** `200 OK`
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 900
}
```

### Logout

```http
POST /api/auth/logout
```

**Headers:** `Authorization: Bearer <access_token>`

**Response:** `204 No Content`

### Logout All Sessions

```http
POST /api/auth/logout-all
```

**Response:** `204 No Content`

### Password Reset Request

```http
POST /api/auth/password-reset/request
```

**Request:**
```json
{
  "email": "user@example.com"
}
```

**Response:** `200 OK`
```json
{
  "message": "Password reset link sent to your email"
}
```

### Password Reset Confirm

```http
POST /api/auth/password-reset/confirm
```

**Request:**
```json
{
  "token": "reset_token_from_email",
  "newPassword": "NewSecurePass123!"
}
```

**Response:** `200 OK`
```json
{
  "message": "Password successfully reset"
}
```

---

## USERS & ROLES

### Get Current User

```http
GET /api/users/me
```

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "email": "admin@restaurant.com",
  "phone": "+79001234567",
  "role": "RESTAURANTADMIN",
  "tenantId": "uuid",
  "restaurantIds": ["uuid1", "uuid2"],
  "permissions": [
    "read:guests",
    "write:guests",
    "read:analytics",
    "manage:loyalty"
  ],
  "isActive": true,
  "emailVerified": true,
  "phoneVerified": true,
  "createdAt": "2026-01-15T10:30:00Z",
  "lastLoginAt": "2026-02-03T09:15:00Z"
}
```

### Create Manager

```http
POST /api/users/managers
```

**Required Permission:** `create:manager`
**Required Role:** `RESTAURANTADMIN`

**Request:**
```json
{
  "email": "manager@restaurant.com",
  "phone": "+79001234568",
  "firstName": "Alice",
  "lastName": "Smith",
  "restaurantId": "uuid",
  "permissions": [
    "read:guests",
    "write:guests",
    "manual:add:balls",
    "read:analytics"
  ]
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "email": "manager@restaurant.com",
  "role": "MANAGER",
  "restaurantId": "uuid",
  "permissions": [...],
  "tempPassword": "TempPass123",
  "message": "Manager created. Temporary password sent to email."
}
```

### Create Cashier

```http
POST /api/users/cashiers
```

**Required Permission:** `create:cashier`
**Required Role:** `MANAGER`, `RESTAURANTADMIN`

**Request:**
```json
{
  "phone": "+79001234569",
  "firstName": "Bob",
  "lastName": "Johnson",
  "restaurantId": "uuid"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "phone": "+79001234569",
  "role": "CASHIER",
  "restaurantId": "uuid",
  "tempPassword": "CashierPass123"
}
```

### Grant Permission

```http
POST /api/users/{userId}/permissions
```

**Required Permission:** `manage:permissions`

**Request:**
```json
{
  "permission": "write:balls",
  "restaurantId": "uuid",  // Optional
  "expiresAt": "2026-12-31T23:59:59Z"  // Optional
}
```

**Response:** `201 Created`
```json
{
  "userId": "uuid",
  "permission": "write:balls",
  "restaurantId": "uuid",
  "grantedAt": "2026-02-03T12:00:00Z",
  "expiresAt": "2026-12-31T23:59:59Z"
}
```

### Revoke Permission

```http
DELETE /api/users/{userId}/permissions/{permission}
```

**Query Parameters:**
- `restaurantId` (optional) - Specific restaurant

**Response:** `204 No Content`

---

## TENANTS & SUBSCRIPTIONS

### Get Tenant Info

```http
GET /api/tenants/me
```

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "name": "My Restaurant Chain",
  "slug": "my-restaurant-chain",
  "domain": "mychain.max-loyalty.com",
  "logo": "https://cdn.max-loyalty.com/logos/uuid.png",
  "isActive": true,
  "subscription": {
    "planType": "PRO",
    "status": "ACTIVE",
    "currentPeriodEnd": "2026-03-03T00:00:00Z"
  },
  "limits": {
    "maxRestaurants": 5,
    "maxGuests": 6000,
    "currentRestaurants": 3,
    "currentGuests": 2150
  },
  "createdAt": "2026-01-15T00:00:00Z"
}
```

### Upgrade Subscription

```http
POST /api/subscriptions/upgrade
```

**Required Role:** `OWNER`

**Request:**
```json
{
  "newPlan": "ULTIMATE",
  "billingCycle": "YEARLY",  // MONTHLY or YEARLY
  "paymentMethodId": "pm_stripe_xxx"
}
```

**Response:** `200 OK`
```json
{
  "subscription": {
    "id": "uuid",
    "planType": "ULTIMATE",
    "status": "ACTIVE",
    "currentPeriodStart": "2026-02-03T00:00:00Z",
    "currentPeriodEnd": "2027-02-03T00:00:00Z",
    "amount": 100000.00,
    "currency": "RUB"
  },
  "limits": {
    "maxRestaurants": -1,  // unlimited
    "maxGuests": -1
  }
}
```

### Get Billing History

```http
GET /api/subscriptions/billing-history
```

**Query Parameters:**
- `page` (default: 1)
- `limit` (default: 30)
- `from` (date) - Filter from date
- `to` (date) - Filter to date

**Response:** `200 OK`
```json
{
  "data": [
    {
      "id": "uuid",
      "invoiceNumber": "INV-2026-001",
      "amount": 15000.00,
      "currency": "RUB",
      "status": "PAID",
      "paidAt": "2026-02-01T10:30:00Z",
      "pdfUrl": "https://cdn.max-loyalty.com/invoices/uuid.pdf"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 30,
    "total": 12
  }
}
```

---

## RESTAURANTS

### List Restaurants

```http
GET /api/restaurants
```

**Query Parameters:**
- `page` (default: 1)
- `limit` (default: 30)
- `isActive` (boolean)

**Response:** `200 OK`
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Main Branch",
      "slug": "main-branch",
      "address": "123 Main St, Moscow",
      "phone": "+74951234567",
      "isActive": true,
      "loyaltySystem": {
        "id": "uuid",
        "type": "POINTS",
        "scope": "LOCAL"
      },
      "stats": {
        "totalGuests": 1250,
        "activeGuests": 890
      },
      "createdAt": "2026-01-20T00:00:00Z"
    }
  ],
  "meta": {...}
}
```

### Create Restaurant

```http
POST /api/restaurants
```

**Required Permission:** `create:restaurant`
**Required Role:** `RESTAURANTADMIN`, `OWNER`

**Request:**
```json
{
  "name": "Downtown Branch",
  "slug": "downtown-branch",
  "address": "456 Central Ave, Moscow",
  "city": "Moscow",
  "phone": "+74951234568",
  "email": "downtown@restaurant.com",
  "timezone": "Europe/Moscow"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "tenantId": "uuid",
  "name": "Downtown Branch",
  "slug": "downtown-branch",
  "address": "456 Central Ave, Moscow",
  "isActive": true,
  "createdAt": "2026-02-03T12:00:00Z"
}
```

### Update Restaurant

```http
PUT /api/restaurants/{id}
```

**Request:**
```json
{
  "name": "Downtown Branch (Updated)",
  "phone": "+74951234569",
  "isActive": true
}
```

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "name": "Downtown Branch (Updated)",
  "updatedAt": "2026-02-03T12:15:00Z"
}
```

---

## GUESTS & LOYALTY CARDS

### Register Guest (Manual)

```http
POST /api/guests
```

**Required Permission:** `create:guest`
**Required Role:** `MANAGER`, `RESTAURANTADMIN`

**Request:**
```json
{
  "phone": "+79001234570",
  "firstName": "Ivan",
  "lastName": "Petrov",
  "middleName": "Sergeevich",
  "gender": "MALE",
  "birthdate": "1990-05-15",
  "email": "ivan@example.com",
  "city": "Moscow",
  "restaurantId": "uuid",
  "privacyPolicyAccepted": true,
  "marketingAccepted": true
}
```

**Response:** `201 Created`
```json
{
  "user": {
    "id": "uuid",
    "phone": "+79001234570",
    "role": "GUEST"
  },
  "guestProfile": {
    "id": "uuid",
    "firstName": "Ivan",
    "lastName": "Petrov",
    "birthdate": "1990-05-15"
  },
  "guestCard": {
    "id": "uuid",
    "qrCode": "https://api.max-loyalty.com/qr/uuid",
    "code6Digit": "123456",
    "balance": 0,
    "status": "ACTIVE",
    "level": {
      "id": "uuid",
      "name": "Bronze",
      "color": "#CD7F32"
    }
  },
  "message": "Гость успешно зарегистрирован"
}
```

### Register Guest (POS)

```http
POST /api/guests/register/pos
```

**Required Permission:** `create:guest`
**Required Role:** `CASHIER`, `MANAGER`

**Request:**
```json
{
  "phone": "+79001234571",
  "firstName": "Maria",
  "restaurantId": "uuid"
}
```

**Response:** `201 Created`
```json
{
  "guestCard": {
    "id": "uuid",
    "qrCode": "https://api.max-loyalty.com/qr/uuid",
    "code6Digit": "789012",
    "balance": 0
  },
  "message": "Карта лояльности выдана"
}
```

### Register Guest (Telegram)

```http
POST /api/guests/register/telegram
```

**Public endpoint** (no auth required)

**Request:**
```json
{
  "telegramId": "123456789",
  "username": "ivan_user",
  "firstName": "Ivan",
  "phone": "+79001234572",
  "birthdate": "1990-05-15",
  "restaurantId": "uuid",
  "privacyPolicyAccepted": true
}
```

**Response:** `201 Created`
```json
{
  "guestCard": {
    "id": "uuid",
    "qrCode": "https://api.max-loyalty.com/qr/uuid",
    "code6Digit": "345678",
    "balance": 0
  },
  "telegramBotLink": "https://t.me/loyalty_bot",
  "message": "Вы успешно зарегистрировались!"
}
```

### Get Guest List

```http
GET /api/guests
```

**Query Parameters:**
- `page` (default: 1)
- `limit` (default: 30)
- `search` - Search by name/phone/email
- `status` - Filter by card status
- `levelId` - Filter by loyalty level
- `restaurantId` - Filter by restaurant
- `registeredFrom` - Date from
- `registeredTo` - Date to

**Response:** `200 OK`
```json
{
  "data": [
    {
      "id": "uuid",
      "firstName": "Ivan",
      "lastName": "Petrov",
      "phone": "+79001234570",
      "email": "ivan@example.com",
      "card": {
        "id": "uuid",
        "code6Digit": "123456",
        "balance": 550,
        "status": "ACTIVE",
        "level": {
          "name": "Silver",
          "color": "#C0C0C0"
        }
      },
      "stats": {
        "totalVisits": 12,
        "totalSpent": 45000.00,
        "lastVisit": "2026-02-01T18:30:00Z"
      },
      "registeredAt": "2026-01-15T10:00:00Z"
    }
  ],
  "meta": {...}
}
```

### Get Guest by ID

```http
GET /api/guests/{id}
```

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "user": {
    "phone": "+79001234570",
    "email": "ivan@example.com",
    "telegramId": "123456789"
  },
  "profile": {
    "firstName": "Ivan",
    "lastName": "Petrov",
    "middleName": "Sergeevich",
    "gender": "MALE",
    "birthdate": "1990-05-15",
    "city": "Moscow"
  },
  "card": {
    "id": "uuid",
    "qrCode": "https://api.max-loyalty.com/qr/uuid",
    "code6Digit": "123456",
    "balance": 550,
    "status": "ACTIVE",
    "level": {
      "id": "uuid",
      "name": "Silver",
      "thresholdAmount": 150000.00,
      "discountPercentage": 10,
      "color": "#C0C0C0"
    },
    "lastActivityAt": "2026-02-01T18:30:00Z"
  },
  "children": [
    {
      "id": "uuid",
      "name": "Alex",
      "birthdate": "2015-03-10",
      "gender": "MALE"
    }
  ],
  "stats": {
    "totalVisits": 12,
    "totalSpent": 45000.00,
    "averageCheck": 3750.00,
    "firstVisit": "2026-01-15T10:00:00Z",
    "lastVisit": "2026-02-01T18:30:00Z",
    "daysSinceLastVisit": 2
  },
  "registeredAt": "2026-01-15T10:00:00Z"
}
```

### Get Guest by QR Code

```http
POST /api/guests/by-qr
```

**Request:**
```json
{
  "qrCode": "https://api.max-loyalty.com/qr/uuid"
}
```

**Response:** `200 OK` (same as Get Guest by ID)

### Get Guest by 6-Digit Code

```http
POST /api/guests/by-code
```

**Required Role:** `CASHIER`, `MANAGER`

**Request:**
```json
{
  "code6Digit": "123456"
}
```

**Response:** `200 OK` (same as Get Guest by ID)

### Get Guest by Phone

```http
POST /api/guests/by-phone
```

**Required Role:** `CASHIER`, `MANAGER`

**Request:**
```json
{
  "phone": "+79001234570",
  "restaurantId": "uuid"
}
```

**Response:** `200 OK` (same as Get Guest by ID)

### Update Guest Profile

```http
PUT /api/guests/{id}/profile
```

**Request:**
```json
{
  "firstName": "Ivan",
  "lastName": "Petrov",
  "email": "newemail@example.com",
  "city": "Saint Petersburg"
}
```

**Note:** Можно редактировать 1 раз в 90 дней

**Response:** `200 OK`
```json
{
  "profile": {...},
  "nextEditAllowedAt": "2026-05-03T00:00:00Z"
}
```

### Add Child

```http
POST /api/guests/{id}/children
```

**Request:**
```json
{
  "name": "Sofia",
  "birthdate": "2018-08-22",
  "gender": "FEMALE"
}
```

**Note:** Максимум 3 ребенка

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "name": "Sofia",
  "birthdate": "2018-08-22",
  "gender": "FEMALE"
}
```

### Block Guest Card

```http
POST /api/guests/{id}/card/block
```

**Required Permission:** `manage:guest:card`
**Required Role:** `MANAGER`, `RESTAURANTADMIN`

**Request:**
```json
{
  "reason": "Нарушение правил"
}
```

**Response:** `200 OK`
```json
{
  "cardId": "uuid",
  "status": "BLOCKED",
  "blockedAt": "2026-02-03T12:00:00Z",
  "reason": "Нарушение правил"
}
```

### Delete Guest Card

```http
DELETE /api/guests/{id}/card
```

**Required Permission:** `manage:guest:card`

**Request:**
```json
{
  "reason": "Запрос пользователя (GDPR)"
}
```

**Response:** `204 No Content`

---

## BALL TRANSACTIONS

### Earn Balls (from Purchase)

```http
POST /api/balls/earn
```

**Required Permission:** `write:balls`
**Required Role:** `CASHIER`, `MANAGER`

**Request:**
```json
{
  "guestCardId": "uuid",
  "checkAmount": 5000.00,
  "restaurantId": "uuid",
  "posTransactionId": "pos_txn_123"
}
```

**Response:** `201 Created`
```json
{
  "transaction": {
    "id": "uuid",
    "type": "EARN_PURCHASE",
    "amount": 250,
    "balanceBefore": 550,
    "balanceAfter": 800,
    "checkAmount": 5000.00
  },
  "card": {
    "balance": 800,
    "level": {
      "name": "Silver"
    }
  },
  "levelUpgrade": null  // or level info if upgraded
}
```

### Redeem Balls

```http
POST /api/balls/redeem
```

**Required Permission:** `write:balls`

**Request:**
```json
{
  "guestCardId": "uuid",
  "amount": 100,
  "checkAmount": 3000.00,
  "restaurantId": "uuid"
}
```

**Response:** `201 Created`
```json
{
  "transaction": {
    "id": "uuid",
    "type": "REDEEM",
    "amount": -100,
    "balanceBefore": 800,
    "balanceAfter": 700,
    "checkAmount": 3000.00
  },
  "card": {
    "balance": 700
  }
}
```

### Manual Add Balls

```http
POST /api/balls/manual/add
```

**Required Permission:** `manual:add:balls`
**Required Role:** `MANAGER`, `RESTAURANTADMIN`

**Request:**
```json
{
  "guestCardId": "uuid",
  "amount": 500,
  "reason": "Компенсация за неудобства"
}
```

**Response:** `201 Created`
```json
{
  "transaction": {
    "id": "uuid",
    "type": "MANUAL_ADD",
    "amount": 500,
    "balanceBefore": 700,
    "balanceAfter": 1200,
    "reason": "Компенсация за неудобства",
    "createdBy": {
      "id": "uuid",
      "firstName": "Alice",
      "lastName": "Smith"
    }
  }
}
```

### Transfer Balls (Initiate)

```http
POST /api/balls/transfer/initiate
```

**Required Role:** `GUEST`

**Request:**
```json
{
  "receiverPhone": "+79001234571",
  "amount": 100
}
```

**Response:** `200 OK`
```json
{
  "transferId": "uuid",
  "receiver": {
    "firstName": "Maria",
    "lastName": "Ivanova"
  },
  "amount": 100,
  "message": "Код подтверждения отправлен на ваш телефон",
  "expiresIn": "10 минут"
}
```

### Transfer Balls (Confirm)

```http
POST /api/balls/transfer/confirm
```

**Request:**
```json
{
  "transferId": "uuid",
  "verificationCode": "123456"
}
```

**Response:** `200 OK`
```json
{
  "message": "Перевод успешно выполнен",
  "senderBalance": 600,
  "receiverBalance": 350,
  "amountTransferred": 100
}
```

### Get Ball Transaction History

```http
GET /api/balls/transactions
```

**Query Parameters:**
- `guestCardId` (required)
- `page` (default: 1)
- `limit` (default: 30)
- `type` - Filter by transaction type
- `from` - Date from
- `to` - Date to

**Response:** `200 OK`
```json
{
  "data": [
    {
      "id": "uuid",
      "type": "EARN_PURCHASE",
      "amount": 250,
      "balanceBefore": 550,
      "balanceAfter": 800,
      "description": "Начислено за покупку",
      "checkAmount": 5000.00,
      "createdAt": "2026-02-03T18:30:00Z"
    },
    {
      "id": "uuid",
      "type": "REDEEM",
      "amount": -100,
      "balanceBefore": 800,
      "balanceAfter": 700,
      "description": "Списано при оплате",
      "checkAmount": 3000.00,
      "createdAt": "2026-02-03T19:15:00Z"
    }
  ],
  "meta": {...}
}
```

---

## LOYALTY SYSTEM

### Create Loyalty System

```http
POST /api/loyalty/systems
```

**Required Permission:** `manage:loyalty:system`
**Required Role:** `RESTAURANTADMIN`

**Request:**
```json
{
  "restaurantId": "uuid",  // null for global
  "type": "POINTS",
  "scope": "LOCAL",
  "name": "Система баллов",
  "description": "Накопительная система баллов"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "restaurantId": "uuid",
  "type": "POINTS",
  "scope": "LOCAL",
  "name": "Система баллов",
  "isActive": true,
  "createdAt": "2026-02-03T12:00:00Z"
}
```

### Create Loyalty Level

```http
POST /api/loyalty/levels
```

**Request:**
```json
{
  "loyaltySystemId": "uuid",
  "name": "Silver",
  "description": "Серебряный уровень",
  "thresholdAmount": 150000.00,
  "ballsPerRub": 10,  // For POINTS type
  "order": 2,
  "color": "#C0C0C0",
  "icon": "silver-medal"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "loyaltySystemId": "uuid",
  "name": "Silver",
  "thresholdAmount": 150000.00,
  "ballsPerRub": 10,
  "order": 2,
  "color": "#C0C0C0"
}
```

### Create Loyalty Rule

```http
POST /api/loyalty/rules
```

**Request:**
```json
{
  "loyaltySystemId": "uuid",
  "name": "Бонус за 3-й визит",
  "description": "100 баллов после 3-го визита",
  "conditionType": "NTH_VISIT",
  "conditionValue": {
    "visit_number": 3
  },
  "rewardType": "BALLS",
  "rewardValue": {
    "balls": 100
  },
  "validFrom": "2026-02-01T00:00:00Z",
  "validUntil": "2026-12-31T23:59:59Z"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "name": "Бонус за 3-й визит",
  "isActive": true,
  "createdAt": "2026-02-03T12:00:00Z"
}
```

### Create Promo

```http
POST /api/loyalty/promos
```

**Request:**
```json
{
  "loyaltySystemId": "uuid",
  "name": "Весенняя акция",
  "description": "500 баллов за покупку от 5000 рублей",
  "ballAmount": 500,
  "maxRedeemsPerGuest": 1,
  "maxRedeems": 100,
  "requiresCondition": true,
  "conditionType": "MIN_PURCHASE",
  "conditionValue": {
    "amount": 5000.00
  },
  "validFrom": "2026-03-01T00:00:00Z",
  "validUntil": "2026-03-31T23:59:59Z"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "name": "Весенняя акция",
  "ballAmount": 500,
  "totalRedeems": 0,
  "maxRedeems": 100,
  "validFrom": "2026-03-01T00:00:00Z",
  "validUntil": "2026-03-31T23:59:59Z",
  "isActive": true
}
```

### Get Loyalty System

```http
GET /api/loyalty/systems/{id}
```

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "type": "POINTS",
  "scope": "LOCAL",
  "name": "Система баллов",
  "restaurant": {
    "id": "uuid",
    "name": "Main Branch"
  },
  "levels": [
    {
      "id": "uuid",
      "name": "Bronze",
      "thresholdAmount": 0,
      "ballsPerRub": 5,
      "order": 1,
      "color": "#CD7F32"
    },
    {
      "id": "uuid",
      "name": "Silver",
      "thresholdAmount": 150000.00,
      "ballsPerRub": 10,
      "order": 2,
      "color": "#C0C0C0"
    },
    {
      "id": "uuid",
      "name": "Gold",
      "thresholdAmount": 400000.00,
      "ballsPerRub": 15,
      "order": 3,
      "color": "#FFD700"
    }
  ],
  "rules": [
    {
      "id": "uuid",
      "name": "Бонус за 3-й визит",
      "conditionType": "NTH_VISIT",
      "rewardValue": { "balls": 100 },
      "isActive": true
    }
  ],
  "promos": [
    {
      "id": "uuid",
      "name": "Весенняя акция",
      "ballAmount": 500,
      "validFrom": "2026-03-01T00:00:00Z",
      "validUntil": "2026-03-31T23:59:59Z",
      "isActive": true
    }
  ]
}
```

---

## ANALYTICS & REPORTS

### Get Owner Analytics

```http
GET /api/analytics/owner
```

**Required Role:** `OWNER`

**Query Parameters:**
- `from` (date) - Start date
- `to` (date) - End date

**Response:** `200 OK`
```json
{
  "period": {
    "from": "2026-01-01T00:00:00Z",
    "to": "2026-02-03T23:59:59Z"
  },
  "summary": {
    "totalTenants": 125,
    "activeTenants": 108,
    "totalRestaurants": 487,
    "totalGuests": 150000,
    "mrr": 2500000.00,
    "churnRate": 3.2
  },
  "byPlan": [
    {
      "plan": "STANDARD",
      "count": 45,
      "mrr": 675000.00
    },
    {
      "plan": "PRO",
      "count": 38,
      "mrr": 1330000.00
    },
    {
      "plan": "ULTIMATE",
      "count": 25,
      "mrr": 2500000.00
    }
  ],
  "growth": {
    "newTenants": 12,
    "churned": 4,
    "netGrowth": 8,
    "growthRate": 6.7
  }
}
```

### Get Admin Analytics

```http
GET /api/analytics/admin
```

**Required Role:** `RESTAURANTADMIN`

**Query Parameters:**
- `restaurantId` (optional) - Specific restaurant
- `from` (date)
- `to` (date)

**Response:** `200 OK`
```json
{
  "period": {
    "from": "2026-01-01T00:00:00Z",
    "to": "2026-02-03T23:59:59Z"
  },
  "summary": {
    "totalGuests": 2150,
    "activeGuests": 1580,
    "newGuests": 320,
    "totalVisits": 8940,
    "totalRevenue": 18500000.00,
    "averageCheck": 2070.00,
    "ballsEarned": 1850000,
    "ballsRedeemed": 450000
  },
  "byRestaurant": [
    {
      "restaurantId": "uuid",
      "name": "Main Branch",
      "guests": 1250,
      "visits": 5200,
      "revenue": 11000000.00,
      "averageCheck": 2115.00
    },
    {
      "restaurantId": "uuid",
      "name": "Downtown Branch",
      "guests": 900,
      "visits": 3740,
      "revenue": 7500000.00,
      "averageCheck": 2005.00
    }
  ],
  "loyaltyEffectiveness": {
    "participationRate": 73.5,
    "repeatVisitRate": 65.2,
    "avgBallsPerGuest": 860,
    "redemptionRate": 24.3
  }
}
```

### Get Manager Analytics

```http
GET /api/analytics/manager
```

**Required Role:** `MANAGER`

**Query Parameters:**
- `restaurantId` (required)
- `from` (date)
- `to` (date)

**Response:** `200 OK`
```json
{
  "restaurant": {
    "id": "uuid",
    "name": "Main Branch"
  },
  "period": {
    "from": "2026-01-01T00:00:00Z",
    "to": "2026-02-03T23:59:59Z"
  },
  "metrics": {
    "totalGuests": 1250,
    "activeGuests": 890,
    "newGuests": 180,
    "totalVisits": 5200,
    "totalRevenue": 11000000.00,
    "averageCheck": 2115.00
  },
  "guestSegmentation": {
    "vip": 85,
    "active": 590,
    "atRisk": 145,
    "churned": 70
  },
  "topGuests": [
    {
      "guestId": "uuid",
      "name": "Ivan Petrov",
      "totalSpent": 125000.00,
      "visits": 42,
      "level": "Gold"
    }
  ]
}
```

### Export Analytics Report

```http
POST /api/analytics/export
```

**Request:**
```json
{
  "reportType": "ADMIN_SUMMARY",
  "format": "PDF",  // PDF, CSV, XLSX
  "from": "2026-01-01T00:00:00Z",
  "to": "2026-02-03T23:59:59Z",
  "restaurantId": "uuid"  // Optional
}
```

**Response:** `200 OK`
```json
{
  "reportId": "uuid",
  "status": "PROCESSING",
  "estimatedTime": "30 seconds",
  "checkUrl": "/api/analytics/export/uuid"
}
```

### Get Export Status

```http
GET /api/analytics/export/{reportId}
```

**Response:** `200 OK`
```json
{
  "reportId": "uuid",
  "status": "COMPLETED",
  "downloadUrl": "https://cdn.max-loyalty.com/reports/uuid.pdf",
  "expiresAt": "2026-02-10T00:00:00Z"
}
```

---

## POS INTEGRATION

### Create POS Integration

```http
POST /api/pos/integrations
```

**Required Permission:** `manage:pos:integration`
**Required Role:** `RESTAURANTADMIN`

**Request:**
```json
{
  "restaurantId": "uuid",
  "posType": "IIKO",
  "syncDirection": "PUSH",
  "apiUrl": "https://api.iiko.com",
  "apiKey": "iiko_api_key_encrypted",
  "webhookSecret": "webhook_secret_key"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "restaurantId": "uuid",
  "posType": "IIKO",
  "syncDirection": "PUSH",
  "webhookUrl": "https://api.max-loyalty.com/webhooks/pos/uuid",
  "isActive": true,
  "createdAt": "2026-02-03T12:00:00Z"
}
```

### POS Webhook (Transaction)

```http
POST /webhooks/pos/{integrationId}
```

**Public endpoint** (signature validation required)

**Headers:**
```http
X-POS-Signature: sha256_signature
Content-Type: application/json
```

**Request (iiko format):**
```json
{
  "transactionId": "iiko_txn_123",
  "timestamp": "2026-02-03T18:30:00Z",
  "guestPhone": "+79001234570",
  "checkAmount": 5000.00,
  "items": [
    {
      "name": "Бургер",
      "quantity": 2,
      "price": 450.00
    }
  ],
  "paymentMethod": "CARD"
}
```

**Response:** `200 OK`
```json
{
  "success": true,
  "ballsEarned": 250,
  "newBalance": 800,
  "guestLevel": "Silver"
}
```

### Get POS Sync History

```http
GET /api/pos/integrations/{id}/syncs
```

**Query Parameters:**
- `page` (default: 1)
- `limit` (default: 30)
- `status` - Filter by sync status

**Response:** `200 OK`
```json
{
  "data": [
    {
      "id": "uuid",
      "syncStart": "2026-02-03T10:00:00Z",
      "syncEnd": "2026-02-03T10:02:30Z",
      "status": "SUCCESS",
      "recordsSynced": 42,
      "errorMessage": null
    }
  ],
  "meta": {...}
}
```

---

## TELEGRAM INTEGRATION

### Telegram Mini App Auth

```http
POST /api/telegram/mini-app/auth
```

**Public endpoint**

**Request:**
```json
{
  "initData": "telegram_init_data_string"
}
```

**Response:** `200 OK`
```json
{
  "isValid": true,
  "user": {
    "telegramId": "123456789",
    "username": "ivan_user",
    "firstName": "Ivan"
  },
  "guest": {
    "id": "uuid",
    "card": {
      "qrCode": "https://api.max-loyalty.com/qr/uuid",
      "code6Digit": "123456",
      "balance": 800,
      "level": {
        "name": "Silver",
        "color": "#C0C0C0"
      }
    }
  },
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Get Guest Card (Mini App)

```http
GET /api/telegram/mini-app/card
```

**Headers:** `Authorization: Bearer <telegram_access_token>`

**Response:** `200 OK`
```json
{
  "qrCode": "https://api.max-loyalty.com/qr/uuid",
  "qrCodeImage": "data:image/png;base64,...",
  "code6Digit": "123456",
  "balance": 800,
  "level": {
    "name": "Silver",
    "color": "#C0C0C0",
    "nextLevel": {
      "name": "Gold",
      "threshold": 400000.00,
      "progress": 62.5
    }
  },
  "restaurant": {
    "name": "Main Branch",
    "phone": "+74951234567"
  }
}
```

### Get Transaction History (Mini App)

```http
GET /api/telegram/mini-app/transactions
```

**Query Parameters:**
- `limit` (default: 30)
- `offset` (default: 0)

**Response:** `200 OK`
```json
{
  "transactions": [
    {
      "id": "uuid",
      "type": "EARN_PURCHASE",
      "amount": 250,
      "balanceAfter": 800,
      "description": "Начислено за покупку",
      "restaurant": "Main Branch",
      "createdAt": "2026-02-03T18:30:00Z"
    }
  ],
  "hasMore": true
}
```

---

## 🔒 RATE LIMITING

### Лимиты по endpoint

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/auth/login` | 5 requests | 15 minutes |
| `/auth/register` | 3 requests | 1 hour |
| `/api/*` | 1000 requests | 1 minute |
| `/api/analytics/*` | 100 requests | 1 minute |
| `/webhooks/*` | 10000 requests | 1 minute |

### Заголовки ответа

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1706965200
```

### Ошибка 429

```json
{
  "statusCode": 429,
  "message": "Too many requests",
  "retryAfter": 45
}
```

---

## 📚 ПОЛЕЗНЫЕ ССЫЛКИ

- [Swagger UI](https://api.max-loyalty.com/docs)
- [Postman Collection](https://api.max-loyalty.com/postman-collection.json)
- [OpenAPI Spec](https://api.max-loyalty.com/openapi.json)

---

**Версия:** 1.0  
**Дата:** 2026-02-03  
**Статус:** Полная API спецификация
