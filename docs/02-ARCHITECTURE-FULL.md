# 🏗️ ПОЛНАЯ АРХИТЕКТУРА ПРОЕКТА MAX-LOYALTY

## 📋 СОДЕРЖАНИЕ

1. [Общая архитектура системы](#общая-архитектура-системы)
2. [Backend Architecture (NestJS)](#backend-architecture-nestjs)
3. [Frontend Architecture (React + Vite)](#frontend-architecture-react--vite)
4. [Telegram Bot Architecture](#telegram-bot-architecture)
5. [Database Architecture](#database-architecture)
6. [Multi-Tenant Architecture](#multi-tenant-architecture)
7. [Security Architecture](#security-architecture)
8. [Integration Architecture](#integration-architecture)
9. [Analytics & Reporting Architecture](#analytics--reporting-architecture)
10. [DevOps & Infrastructure](#devops--infrastructure)

---

## 1. ОБЩАЯ АРХИТЕКТУРА СИСТЕМЫ

### 1.1 High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        MAX-LOYALTY PLATFORM                      │
│                      Multi-Tenant SaaS System                    │
└─────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
            ┌───────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐
            │   Frontend    │ │  Backend  │ │  Telegram   │
            │  React+Vite   │ │  NestJS   │ │     Bot     │
            └───────────────┘ └─────┬─────┘ └─────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
            ┌───────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐
            │  PostgreSQL  │ │   Redis   │ │    Bull     │
            │   Database   │ │   Cache   │ │    Queue    │
            └──────────────┘ └───────────┘ └─────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
            ┌───────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐
            │   iiko API   │ │ R-Keeper  │ │  Payment    │
            │ Integration  │ │    API    │ │  Gateway    │
            └──────────────┘ └───────────┘ └─────────────┘
```

### 1.2 Архитектурные слои

```
┌────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                       │
│  - Web UI (Owner/Admin/Manager/Cashier dashboards)        │
│  - Telegram Mini App (Guest interface)                     │
│  - Telegram Bot (Guest commands & notifications)           │
└────────────────────────────────────────────────────────────┘
                            │
┌────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                        │
│  - REST API Controllers                                     │
│  - GraphQL Resolvers (optional)                            │
│  - Webhook Handlers                                         │
│  - Background Jobs                                          │
└────────────────────────────────────────────────────────────┘
                            │
┌────────────────────────────────────────────────────────────┐
│                     BUSINESS LAYER                          │
│  - Domain Services                                          │
│  - Business Logic                                           │
│  - Validation Rules                                         │
│  - RBAC & Permissions                                       │
└────────────────────────────────────────────────────────────┘
                            │
┌────────────────────────────────────────────────────────────┐
│                   DATA ACCESS LAYER                         │
│  - Prisma ORM                                               │
│  - Repository Pattern                                       │
│  - Database Migrations                                      │
│  - Query Optimization                                       │
└────────────────────────────────────────────────────────────┘
                            │
┌────────────────────────────────────────────────────────────┐
│                  INFRASTRUCTURE LAYER                       │
│  - PostgreSQL Database                                      │
│  - Redis Cache & Queue                                      │
│  - External APIs                                            │
│  - File Storage                                             │
└────────────────────────────────────────────────────────────┘
```

### 1.3 Компоненты системы

#### Core Components
1. **Authentication & Authorization**
   - JWT-based authentication
   - Role-Based Access Control (RBAC)
   - Multi-tenant isolation
   - Session management

2. **Tenant Management**
   - Tenant provisioning
   - Subscription management
   - Billing & payments
   - Usage limits enforcement

3. **Loyalty System Engine**
   - Points/discount calculation
   - Level management
   - Promo rules engine
   - Transaction processing

4. **Guest Management**
   - Registration (Manual/POS/Link/Telegram)
   - Profile management
   - Card generation (QR + 6-digit)
   - Children management (max 3)

5. **Analytics Engine**
   - Real-time metrics
   - Scheduled reports
   - Data aggregation
   - Export functionality

#### Integration Components
1. **POS Integration**
   - iiko API connector
   - R-Keeper API connector
   - Webhook handlers
   - Data synchronization

2. **Telegram Integration**
   - Bot commands handler
   - Mini App server
   - Webhook processing
   - Notifications service

3. **Payment Integration**
   - Stripe connector
   - YooKassa connector
   - Webhook handlers
   - Billing automation

---

## 2. BACKEND ARCHITECTURE (NestJS)

### 2.1 Структура проекта

```
apps/backend/
├── src/
│   ├── main.ts                    # Точка входа приложения
│   ├── app.module.ts              # Корневой модуль
│   │
│   ├── config/                    # Конфигурация
│   │   ├── app.config.ts
│   │   ├── database.config.ts
│   │   ├── jwt.config.ts
│   │   ├── telegram.config.ts
│   │   └── pos.config.ts
│   │
│   ├── common/                    # Общие компоненты
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts
│   │   │   ├── current-tenant.decorator.ts
│   │   │   ├── has-role.decorator.ts
│   │   │   ├── has-permission.decorator.ts
│   │   │   └── require-restaurant.decorator.ts
│   │   ├── guards/
│   │   │   ├── jwt.guard.ts
│   │   │   ├── role.guard.ts
│   │   │   ├── permission.guard.ts
│   │   │   ├── tenant.guard.ts
│   │   │   └── rate-limit.guard.ts
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts
│   │   │   ├── transform.interceptor.ts
│   │   │   ├── timeout.interceptor.ts
│   │   │   └── tenant.interceptor.ts
│   │   ├── filters/
│   │   │   ├── http-exception.filter.ts
│   │   │   ├── prisma-exception.filter.ts
│   │   │   └── all-exceptions.filter.ts
│   │   ├── pipes/
│   │   │   ├── validation.pipe.ts
│   │   │   ├── parse-uuid.pipe.ts
│   │   │   └── sanitization.pipe.ts
│   │   └── utils/
│   │       ├── crypto.util.ts
│   │       ├── date.util.ts
│   │       ├── phone.util.ts
│   │       └── qr-code.util.ts
│   │
│   ├── modules/
│   │   │
│   │   ├── auth/                  # Аутентификация
│   │   │   ├── auth.module.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── strategies/
│   │   │   │   ├── jwt.strategy.ts
│   │   │   │   ├── jwt-refresh.strategy.ts
│   │   │   │   └── telegram.strategy.ts
│   │   │   ├── dto/
│   │   │   │   ├── register.dto.ts
│   │   │   │   ├── login.dto.ts
│   │   │   │   ├── refresh-token.dto.ts
│   │   │   │   └── password-reset.dto.ts
│   │   │   └── guards/
│   │   │       ├── jwt.guard.ts
│   │   │       └── jwt-refresh.guard.ts
│   │   │
│   │   ├── users/                 # Управление пользователями
│   │   │   ├── users.module.ts
│   │   │   ├── users.controller.ts
│   │   │   ├── users.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── user.entity.ts
│   │   │   │   ├── user-role.entity.ts
│   │   │   │   ├── user-permission.entity.ts
│   │   │   │   └── user-session.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-user.dto.ts
│   │   │       ├── update-user.dto.ts
│   │   │       └── user-response.dto.ts
│   │   │
│   │   ├── rbac/                  # Role-Based Access Control
│   │   │   ├── rbac.module.ts
│   │   │   ├── services/
│   │   │   │   ├── role.service.ts
│   │   │   │   ├── permission.service.ts
│   │   │   │   ├── rbac.service.ts
│   │   │   │   └── user-role.service.ts
│   │   │   ├── guards/
│   │   │   │   ├── role.guard.ts
│   │   │   │   ├── permission.guard.ts
│   │   │   │   └── restaurant.guard.ts
│   │   │   ├── decorators/
│   │   │   │   ├── has-role.decorator.ts
│   │   │   │   ├── has-permission.decorator.ts
│   │   │   │   └── require-restaurant.decorator.ts
│   │   │   └── dto/
│   │   │       ├── grant-permission.dto.ts
│   │   │       ├── revoke-permission.dto.ts
│   │   │       └── assign-role.dto.ts
│   │   │
│   │   ├── tenants/               # Multi-tenant управление
│   │   │   ├── tenants.module.ts
│   │   │   ├── tenants.controller.ts
│   │   │   ├── tenants.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── tenant.entity.ts
│   │   │   │   └── tenant-limits.entity.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-tenant.dto.ts
│   │   │   │   └── update-tenant-limits.dto.ts
│   │   │   └── guards/
│   │   │       └── tenant.guard.ts
│   │   │
│   │   ├── restaurants/           # Управление ресторанами
│   │   │   ├── restaurants.module.ts
│   │   │   ├── restaurants.controller.ts
│   │   │   ├── restaurants.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── restaurant.entity.ts
│   │   │   │   └── restaurant-manager.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-restaurant.dto.ts
│   │   │       └── update-restaurant.dto.ts
│   │   │
│   │   ├── subscriptions/         # Подписки и биллинг
│   │   │   ├── subscriptions.module.ts
│   │   │   ├── subscriptions.controller.ts
│   │   │   ├── subscriptions.service.ts
│   │   │   ├── billing.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── subscription.entity.ts
│   │   │   │   ├── subscription-plan.entity.ts
│   │   │   │   ├── payment.entity.ts
│   │   │   │   └── invoice.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-subscription.dto.ts
│   │   │       ├── upgrade-plan.dto.ts
│   │   │       └── process-payment.dto.ts
│   │   │
│   │   ├── guests/                # Управление гостями
│   │   │   ├── guests.module.ts
│   │   │   ├── controllers/
│   │   │   │   ├── guest.controller.ts
│   │   │   │   ├── guest-card.controller.ts
│   │   │   │   └── ball-transaction.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── guest.service.ts
│   │   │   │   ├── guest-card.service.ts
│   │   │   │   ├── ball-transaction.service.ts
│   │   │   │   ├── guest-visit.service.ts
│   │   │   │   └── ball-transfer.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── guest-profile.entity.ts
│   │   │   │   ├── guest-card.entity.ts
│   │   │   │   ├── guest-child.entity.ts
│   │   │   │   ├── ball-transaction.entity.ts
│   │   │   │   ├── ball-transfer.entity.ts
│   │   │   │   ├── guest-visit.entity.ts
│   │   │   │   └── registration-link.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-guest.dto.ts
│   │   │       ├── update-guest-profile.dto.ts
│   │   │       ├── earn-balls.dto.ts
│   │   │       ├── redeem-balls.dto.ts
│   │   │       ├── manual-add-balls.dto.ts
│   │   │       ├── transfer-balls.dto.ts
│   │   │       └── add-child.dto.ts
│   │   │
│   │   ├── loyalty/               # Система лояльности
│   │   │   ├── loyalty.module.ts
│   │   │   ├── controllers/
│   │   │   │   ├── loyalty-system.controller.ts
│   │   │   │   ├── loyalty-level.controller.ts
│   │   │   │   ├── loyalty-rule.controller.ts
│   │   │   │   └── loyalty-promo.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── loyalty-system.service.ts
│   │   │   │   ├── loyalty-level.service.ts
│   │   │   │   ├── loyalty-rule.service.ts
│   │   │   │   ├── loyalty-promo.service.ts
│   │   │   │   ├── loyalty-calculation.service.ts
│   │   │   │   └── loyalty-design.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── loyalty-system.entity.ts
│   │   │   │   ├── loyalty-level.entity.ts
│   │   │   │   ├── loyalty-rule.entity.ts
│   │   │   │   ├── loyalty-promo.entity.ts
│   │   │   │   └── promo-ball-granted.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-loyalty-system.dto.ts
│   │   │       ├── create-level.dto.ts
│   │   │       ├── create-rule.dto.ts
│   │   │       └── create-promo.dto.ts
│   │   │
│   │   ├── pos-integration/       # Интеграция с POS
│   │   │   ├── pos-integration.module.ts
│   │   │   ├── controllers/
│   │   │   │   ├── iiko-webhook.controller.ts
│   │   │   │   └── rkeeper-webhook.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── iiko.service.ts
│   │   │   │   ├── rkeeper.service.ts
│   │   │   │   ├── pos-sync.service.ts
│   │   │   │   └── pos-transaction.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── pos-integration.entity.ts
│   │   │   │   ├── pos-sync.entity.ts
│   │   │   │   └── pos-transaction.entity.ts
│   │   │   └── dto/
│   │   │       ├── iiko-webhook.dto.ts
│   │   │       └── rkeeper-webhook.dto.ts
│   │   │
│   │   ├── analytics/             # Аналитика и отчеты
│   │   │   ├── analytics.module.ts
│   │   │   ├── controllers/
│   │   │   │   ├── owner-analytics.controller.ts
│   │   │   │   ├── admin-analytics.controller.ts
│   │   │   │   ├── manager-analytics.controller.ts
│   │   │   │   └── guest-analytics.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── daily-analytics.service.ts
│   │   │   │   ├── monthly-analytics.service.ts
│   │   │   │   ├── guest-analytics.service.ts
│   │   │   │   ├── report-generator.service.ts
│   │   │   │   └── export.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── daily-analytics.entity.ts
│   │   │   │   ├── monthly-analytics.entity.ts
│   │   │   │   └── guest-analytics.entity.ts
│   │   │   └── jobs/
│   │   │       ├── daily-aggregation.job.ts
│   │   │       └── monthly-aggregation.job.ts
│   │   │
│   │   ├── telegram/              # Telegram бот
│   │   │   ├── telegram.module.ts
│   │   │   ├── telegram-bot.controller.ts
│   │   │   ├── telegram-bot.service.ts
│   │   │   ├── telegram-mini-app.service.ts
│   │   │   ├── scenes/
│   │   │   │   ├── registration.scene.ts
│   │   │   │   ├── card.scene.ts
│   │   │   │   └── settings.scene.ts
│   │   │   ├── handlers/
│   │   │   │   ├── start.handler.ts
│   │   │   │   ├── card.handler.ts
│   │   │   │   ├── balance.handler.ts
│   │   │   │   └── history.handler.ts
│   │   │   └── keyboards/
│   │   │       ├── main.keyboard.ts
│   │   │       └── card.keyboard.ts
│   │   │
│   │   ├── notifications/         # Уведомления
│   │   │   ├── notifications.module.ts
│   │   │   ├── notification.service.ts
│   │   │   ├── sms.service.ts
│   │   │   ├── email.service.ts
│   │   │   └── telegram-notification.service.ts
│   │   │
│   │   ├── activity-log/          # Журнал активности
│   │   │   ├── activity-log.module.ts
│   │   │   ├── activity-log.controller.ts
│   │   │   ├── activity-log.service.ts
│   │   │   └── entities/
│   │   │       └── activity-log.entity.ts
│   │   │
│   │   └── queue/                 # Очереди задач
│   │       ├── queue.module.ts
│   │       ├── processors/
│   │       │   ├── email.processor.ts
│   │       │   ├── sms.processor.ts
│   │       │   ├── analytics.processor.ts
│   │       │   └── pos-sync.processor.ts
│   │       └── jobs/
│   │           ├── daily-report.job.ts
│   │           └── subscription-check.job.ts
│   │
│   ├── database/                  # База данных
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seeds/
│   │   │       ├── seed.ts
│   │   │       ├── roles.seed.ts
│   │   │       ├── permissions.seed.ts
│   │   │       └── plans.seed.ts
│   │   └── prisma.service.ts
│   │
│   └── test/                      # Тесты
│       ├── unit/
│       ├── integration/
│       └── e2e/
│
├── Dockerfile
├── .env.example
├── package.json
├── tsconfig.json
├── nest-cli.json
└── README.md
```

### 2.2 Модульная архитектура

#### Auth Module
```typescript
// auth/auth.module.ts
@Module({
  imports: [
    JwtModule.registerAsync({
      useFactory: (configService: ConfigService) => ({
        secret: configService.get('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },
      }),
      inject: [ConfigService],
    }),
    PassportModule,
    UsersModule,
    RedisModule,
  ],
  controllers: [AuthController],
  providers: [
    AuthService,
    JwtStrategy,
    JwtRefreshStrategy,
    PasswordService,
    SessionService,
    TokenBlacklistService,
    RateLimitService,
  ],
  exports: [AuthService],
})
export class AuthModule {}
```

#### RBAC Module
```typescript
// rbac/rbac.module.ts
@Module({
  imports: [DatabaseModule, ActivityLogModule],
  providers: [
    RoleService,
    PermissionService,
    RbacService,
    UserRoleService,
    RoleGuard,
    PermissionGuard,
    RestaurantGuard,
  ],
  exports: [
    RoleService,
    PermissionService,
    RbacService,
    UserRoleService,
    RoleGuard,
    PermissionGuard,
    RestaurantGuard,
  ],
})
export class RbacModule {}
```

### 2.3 Guards & Decorators

#### JWT Guard
```typescript
// common/guards/jwt.guard.ts
@Injectable()
export class JwtGuard extends AuthGuard('jwt') {
  constructor(
    private reflector: Reflector,
    private tokenBlacklist: TokenBlacklistService,
  ) {
    super();
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) return true;

    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (await this.tokenBlacklist.isBlacklisted(token)) {
      throw new UnauthorizedException('Token has been revoked');
    }

    return super.canActivate(context) as Promise<boolean>;
  }
}
```

#### Role Guard
```typescript
// rbac/guards/role.guard.ts
@Injectable()
export class RoleGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private rbacService: RbacService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return requiredRoles.some(role => user.roles?.includes(role));
  }
}
```

#### Permission Guard
```typescript
// rbac/guards/permission.guard.ts
@Injectable()
export class PermissionGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private rbacService: RbacService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPermissions = this.reflector.getAllAndOverride<string[]>(
      'permissions',
      [context.getHandler(), context.getClass()],
    );

    if (!requiredPermissions) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const tenantId = request.headers['x-tenant-id'];

    return this.rbacService.checkPermissions(
      user.id,
      tenantId,
      requiredPermissions,
    );
  }
}
```

### 2.4 Interceptors

#### Tenant Interceptor
```typescript
// common/interceptors/tenant.interceptor.ts
@Injectable()
export class TenantInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (user && user.tenantId) {
      request.tenantId = user.tenantId;
      request.headers['x-tenant-id'] = user.tenantId;
    }

    return next.handle();
  }
}
```

#### Logging Interceptor
```typescript
// common/interceptors/logging.interceptor.ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private logger: Logger) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url, body, user } = request;
    const now = Date.now();

    this.logger.log(`[${method}] ${url} - User: ${user?.id} - Start`);

    return next.handle().pipe(
      tap(() => {
        const elapsed = Date.now() - now;
        this.logger.log(
          `[${method}] ${url} - User: ${user?.id} - Completed in ${elapsed}ms`,
        );
      }),
      catchError((error) => {
        this.logger.error(
          `[${method}] ${url} - User: ${user?.id} - Error: ${error.message}`,
        );
        throw error;
      }),
    );
  }
}
```

---

## 3. FRONTEND ARCHITECTURE (React + Vite)

### 3.1 Структура проекта

```
apps/frontend/
├── src/
│   ├── main.tsx                   # Точка входа
│   ├── App.tsx                    # Корневой компонент
│   ├── router.tsx                 # Роутинг
│   │
│   ├── pages/                     # Страницы
│   │   ├── owner/
│   │   │   ├── OwnerDashboard.tsx
│   │   │   ├── AccountsManagement.tsx
│   │   │   ├── GlobalAnalytics.tsx
│   │   │   ├── SubscriptionsManagement.tsx
│   │   │   └── BillingHistory.tsx
│   │   │
│   │   ├── admin/
│   │   │   ├── AdminDashboard.tsx
│   │   │   ├── RestaurantSettings.tsx
│   │   │   ├── LoyaltyBuilder.tsx
│   │   │   ├── GuestsManagement.tsx
│   │   │   ├── StaffManagement.tsx
│   │   │   ├── AnalyticsReports.tsx
│   │   │   └── IntegrationSettings.tsx
│   │   │
│   │   ├── manager/
│   │   │   ├── ManagerDashboard.tsx
│   │   │   ├── GuestsList.tsx
│   │   │   ├── ManualBallsOperation.tsx
│   │   │   ├── RestaurantAnalytics.tsx
│   │   │   └── PromoManagement.tsx
│   │   │
│   │   ├── cashier/
│   │   │   ├── CashierDashboard.tsx
│   │   │   ├── GuestSearch.tsx
│   │   │   ├── QuickRegistration.tsx
│   │   │   └── TransactionHistory.tsx
│   │   │
│   │   ├── guest/
│   │   │   ├── GuestCard.tsx
│   │   │   ├── Profile.tsx
│   │   │   ├── TransactionHistory.tsx
│   │   │   └── Children.tsx
│   │   │
│   │   └── auth/
│   │       ├── Login.tsx
│   │       ├── Register.tsx
│   │       ├── ForgotPassword.tsx
│   │       └── ResetPassword.tsx
│   │
│   ├── components/                # Компоненты
│   │   ├── layout/
│   │   │   ├── MainLayout.tsx
│   │   │   ├── DashboardLayout.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Header.tsx
│   │   │   └── Footer.tsx
│   │   │
│   │   ├── ui/                    # UI компоненты
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Select.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Table.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Tabs.tsx
│   │   │   ├── Tooltip.tsx
│   │   │   └── Loading.tsx
│   │   │
│   │   ├── loyalty/
│   │   │   ├── LoyaltyCard.tsx
│   │   │   ├── LoyaltyLevelBadge.tsx
│   │   │   ├── BallBalance.tsx
│   │   │   ├── QRCodeDisplay.tsx
│   │   │   ├── LevelProgress.tsx
│   │   │   └── CardDesigner.tsx
│   │   │
│   │   ├── analytics/
│   │   │   ├── MetricCard.tsx
│   │   │   ├── LineChart.tsx
│   │   │   ├── BarChart.tsx
│   │   │   ├── PieChart.tsx
│   │   │   ├── DateRangePicker.tsx
│   │   │   └── ExportButton.tsx
│   │   │
│   │   ├── forms/
│   │   │   ├── GuestForm.tsx
│   │   │   ├── RestaurantForm.tsx
│   │   │   ├── LoyaltySystemForm.tsx
│   │   │   ├── PromoForm.tsx
│   │   │   └── UserForm.tsx
│   │   │
│   │   └── shared/
│   │       ├── SearchBar.tsx
│   │       ├── Pagination.tsx
│   │       ├── FilterPanel.tsx
│   │       ├── EmptyState.tsx
│   │       └── ErrorBoundary.tsx
│   │
│   ├── hooks/                     # Custom hooks
│   │   ├── useAuth.ts
│   │   ├── useGuest.ts
│   │   ├── useLoyalty.ts
│   │   ├── useAnalytics.ts
│   │   ├── usePermissions.ts
│   │   ├── usePagination.ts
│   │   ├── useDebounce.ts
│   │   └── useLocalStorage.ts
│   │
│   ├── services/                  # API services
│   │   ├── api.service.ts
│   │   ├── auth.service.ts
│   │   ├── guest.service.ts
│   │   ├── loyalty.service.ts
│   │   ├── analytics.service.ts
│   │   ├── restaurant.service.ts
│   │   ├── subscription.service.ts
│   │   └── notification.service.ts
│   │
│   ├── store/                     # Redux store
│   │   ├── index.ts
│   │   ├── slices/
│   │   │   ├── authSlice.ts
│   │   │   ├── guestSlice.ts
│   │   │   ├── loyaltySlice.ts
│   │   │   ├── restaurantSlice.ts
│   │   │   └── uiSlice.ts
│   │   └── middleware/
│   │       ├── api.middleware.ts
│   │       └── error.middleware.ts
│   │
│   ├── types/                     # TypeScript types
│   │   ├── auth.types.ts
│   │   ├── user.types.ts
│   │   ├── guest.types.ts
│   │   ├── loyalty.types.ts
│   │   ├── analytics.types.ts
│   │   └── api.types.ts
│   │
│   ├── utils/                     # Утилиты
│   │   ├── format.ts
│   │   ├── validation.ts
│   │   ├── date.ts
│   │   ├── phone.ts
│   │   └── constants.ts
│   │
│   ├── styles/                    # Стили
│   │   ├── global.css
│   │   ├── variables.css
│   │   ├── themes/
│   │   │   ├── light.css
│   │   │   └── dark.css
│   │   └── components/
│   │
│   └── assets/                    # Ресурсы
│       ├── images/
│       ├── icons/
│       └── fonts/
│
├── public/
├── Dockerfile
├── .env.example
├── package.json
├── tsconfig.json
├── vite.config.ts
└── README.md
```

### 3.2 State Management (Redux)

#### Auth Slice
```typescript
// store/slices/authSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const login = createAsyncThunk(
  'auth/login',
  async (credentials: LoginDto, { rejectWithValue }) => {
    try {
      const response = await authService.login(credentials);
      return response.data;
    } catch (error) {
      return rejectWithValue(error.response.data);
    }
  }
);

const authSlice = createSlice({
  name: 'auth',
  initialState: {
    user: null,
    token: null,
    isAuthenticated: false,
    loading: false,
    error: null,
  },
  reducers: {
    logout: (state) => {
      state.user = null;
      state.token = null;
      state.isAuthenticated = false;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(login.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(login.fulfilled, (state, action) => {
        state.loading = false;
        state.user = action.payload.user;
        state.token = action.payload.token;
        state.isAuthenticated = true;
      })
      .addCase(login.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      });
  },
});

export const { logout } = authSlice.actions;
export default authSlice.reducer;
```

### 3.3 Custom Hooks

#### useAuth Hook
```typescript
// hooks/useAuth.ts
export const useAuth = () => {
  const dispatch = useDispatch();
  const { user, isAuthenticated, loading } = useSelector(
    (state: RootState) => state.auth
  );

  const login = async (credentials: LoginDto) => {
    await dispatch(authActions.login(credentials));
  };

  const logout = () => {
    dispatch(authActions.logout());
    localStorage.removeItem('token');
    navigate('/login');
  };

  const hasRole = (role: Role): boolean => {
    return user?.roles?.includes(role) || false;
  };

  const hasPermission = (permission: string): boolean => {
    return user?.permissions?.includes(permission) || false;
  };

  return {
    user,
    isAuthenticated,
    loading,
    login,
    logout,
    hasRole,
    hasPermission,
  };
};
```

---

## 4. TELEGRAM BOT ARCHITECTURE

### 4.1 Структура проекта

```
apps/telegram-bot/
├── src/
│   ├── main.ts
│   ├── bot.module.ts
│   │
│   ├── bot/
│   │   ├── bot.service.ts
│   │   ├── bot.update.ts
│   │   │
│   │   ├── commands/
│   │   │   ├── start.command.ts
│   │   │   ├── card.command.ts
│   │   │   ├── balance.command.ts
│   │   │   ├── history.command.ts
│   │   │   ├── children.command.ts
│   │   │   └── settings.command.ts
│   │   │
│   │   ├── scenes/
│   │   │   ├── registration.scene.ts
│   │   │   ├── card.scene.ts
│   │   │   ├── transfer.scene.ts
│   │   │   └── settings.scene.ts
│   │   │
│   │   ├── keyboards/
│   │   │   ├── main.keyboard.ts
│   │   │   ├── card.keyboard.ts
│   │   │   └── settings.keyboard.ts
│   │   │
│   │   └── middleware/
│   │       ├── auth.middleware.ts
│   │       └── session.middleware.ts
│   │
│   ├── mini-app/
│   │   ├── mini-app.controller.ts
│   │   ├── mini-app.service.ts
│   │   └── auth/
│   │       └── telegram-auth.service.ts
│   │
│   └── webhook/
│       └── telegram-webhook.controller.ts
│
├── Dockerfile
├── package.json
└── README.md
```

### 4.2 Bot Commands

#### Start Command
```typescript
// bot/commands/start.command.ts
@Update()
export class StartCommand {
  constructor(
    private botService: BotService,
    private guestService: GuestService,
  ) {}

  @Start()
  async onStart(@Context() ctx: Context) {
    const telegramId = ctx.from.id.toString();
    
    // Проверяем, зарегистрирован ли пользователь
    const guest = await this.guestService.findByTelegramId(telegramId);

    if (guest) {
      await ctx.reply(
        `Добро пожаловать, ${guest.firstname}! 👋\n\nВыберите действие:`,
        Markup.keyboard([
          ['💳 Моя карта', '💰 Баланс'],
          ['📊 История', '⚙️ Настройки'],
        ]).resize()
      );
    } else {
      await ctx.reply(
        'Добро пожаловать в систему лояльности! 🎉\n\n' +
        'Для регистрации нажмите кнопку ниже.',
        Markup.inlineKeyboard([
          Markup.button.callback('📝 Зарегистрироваться', 'register')
        ])
      );
    }
  }
}
```

### 4.3 Telegram Mini App

#### Mini App Service
```typescript
// mini-app/mini-app.service.ts
@Injectable()
export class MiniAppService {
  constructor(
    private guestService: GuestService,
    private loyaltyService: LoyaltyService,
  ) {}

  async validateTelegramAuth(initData: string): Promise<boolean> {
    // Валидация данных от Telegram
    const params = new URLSearchParams(initData);
    const hash = params.get('hash');
    params.delete('hash');

    const dataCheckString = Array.from(params.entries())
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([key, value]) => `${key}=${value}`)
      .join('\n');

    const secretKey = createHmac('sha256', 'WebAppData')
      .update(this.configService.get('TELEGRAM_BOT_TOKEN'))
      .digest();

    const calculatedHash = createHmac('sha256', secretKey)
      .update(dataCheckString)
      .digest('hex');

    return calculatedHash === hash;
  }

  async getGuestCard(telegramId: string) {
    const guest = await this.guestService.findByTelegramId(telegramId);
    if (!guest) throw new NotFoundException('Guest not found');

    const card = await this.guestService.getGuestCard(guest.id);
    return {
      qrcode: card.qrcode,
      code6digit: card.code6digit,
      balance: card.balance,
      level: card.level,
      firstname: guest.firstname,
      lastname: guest.lastname,
    };
  }
}
```

---

## 5. DATABASE ARCHITECTURE

### 5.1 Prisma Schema Overview

```prisma
// database/prisma/schema.prisma

// ENUMS
enum Role {
  OWNER
  RESTAURANTADMIN
  MANAGER
  CASHIER
  GUEST
}

enum CardStatus {
  ACTIVE
  FROZEN
  BLOCKED
  INACTIVE
  EXPIRED
  DELETED
}

enum LoyaltySystemType {
  POINTS
  DISCOUNT
}

enum BallTransactionType {
  EARN_PURCHASE
  REDEEM
  MANUAL_ADD
  MANUAL_SUBTRACT
  PROMO_GRANTED
  TRANSFER_SENT
  TRANSFER_RECEIVED
  EXPIRED
  REFUND
}

// CORE ENTITIES

model User {
  id              String    @id @default(uuid())
  phone           String?   @unique
  email           String?   @unique
  passwordhash    String?
  telegramid      String?   @unique
  role            Role
  phoneverified   Boolean   @default(false)
  emailverified   Boolean   @default(false)
  isactive        Boolean   @default(true)
  createdat       DateTime  @default(now())
  updatedat       DateTime  @updatedAt
  
  // Relations
  guestprofile    GuestProfile?
  userroles       UserRole[]
  userpermissions UserPermission[]
  sessions        UserSession[]
  
  @@index([phone, email, telegramid])
}

model Tenant {
  id          String   @id @default(uuid())
  name        String
  domain      String?  @unique
  isactive    Boolean  @default(true)
  createdat   DateTime @default(now())
  
  // Relations
  subscription    Subscription?
  tenantlimits    TenantLimits?
  restaurants     Restaurant[]
  
  @@index([domain])
}

model Restaurant {
  id          String   @id @default(uuid())
  tenantid    String
  name        String
  address     String?
  phone       String?
  isactive    Boolean  @default(true)
  createdat   DateTime @default(now())
  
  // Relations
  tenant          Tenant           @relation(fields: [tenantid], references: [id])
  loyaltysystem   LoyaltySystem?
  managers        RestaurantManager[]
  guestcards      GuestCard[]
  
  @@index([tenantid])
}

model GuestCard {
  id              String      @id @default(uuid())
  userid          String      @unique
  tenantid        String
  restaurantid    String
  qrcode          String      @unique
  code6digit      String
  balance         Int         @default(0)
  status          CardStatus  @default(ACTIVE)
  levelid         String?
  lastactivityat  DateTime?
  createdat       DateTime    @default(now())
  deletedat       DateTime?
  
  // Relations
  user            User                @relation(fields: [userid], references: [id])
  tenant          Tenant              @relation(fields: [tenantid], references: [id])
  restaurant      Restaurant          @relation(fields: [restaurantid], references: [id])
  level           LoyaltyLevel?       @relation(fields: [levelid], references: [id])
  transactions    BallTransaction[]
  visits          GuestVisit[]
  
  @@index([tenantid, restaurantid, status])
  @@index([qrcode, code6digit])
}

model LoyaltySystem {
  id              String            @id @default(uuid())
  tenantid        String
  restaurantid    String?           @unique
  type            LoyaltySystemType
  isglobal        Boolean           @default(false)
  createdat       DateTime          @default(now())
  
  // Relations
  tenant          Tenant            @relation(fields: [tenantid], references: [id])
  restaurant      Restaurant?       @relation(fields: [restaurantid], references: [id])
  levels          LoyaltyLevel[]
  rules           LoyaltyRule[]
  promos          LoyaltyPromo[]
  
  @@index([tenantid, restaurantid])
}

model BallTransaction {
  id              String                @id @default(uuid())
  guestcardid     String
  tenantid        String
  restaurantid    String?
  type            BallTransactionType
  amount          Int
  balancebefore   Int
  balanceafter    Int
  description     String?
  reason          String?
  checkamount     Decimal?
  postransactionid String?
  createdbyuserid String?
  createdat       DateTime              @default(now())
  
  // Relations
  guestcard       GuestCard             @relation(fields: [guestcardid], references: [id])
  
  @@index([guestcardid, tenantid, createdat])
  @@index([type, createdat])
}
```

### 5.2 Database Relationships

```
┌─────────────┐
│   Tenant    │
└──────┬──────┘
       │ 1:N
       ▼
┌─────────────┐       ┌──────────────┐
│ Restaurant  │◄─────►│ LoyaltySystem│
└──────┬──────┘  1:1  └──────┬───────┘
       │ 1:N                  │ 1:N
       ▼                      ▼
┌─────────────┐       ┌──────────────┐
│  GuestCard  │       │ LoyaltyLevel │
└──────┬──────┘       └──────────────┘
       │ 1:N
       ▼
┌─────────────────────┐
│  BallTransaction    │
└─────────────────────┘
```

### 5.3 Indexes & Optimization

```sql
-- Composite indexes для multi-tenant queries
CREATE INDEX idx_guestcard_tenant_restaurant 
  ON GuestCard(tenantid, restaurantid, status);

-- Index для поиска по QR/Code
CREATE INDEX idx_guestcard_codes 
  ON GuestCard(qrcode, code6digit);

-- Index для транзакций
CREATE INDEX idx_balltransaction_guest_date 
  ON BallTransaction(guestcardid, tenantid, createdat DESC);

-- Full-text search index
CREATE INDEX idx_user_search 
  ON User USING gin(to_tsvector('russian', 
    coalesce(firstname, '') || ' ' || 
    coalesce(lastname, '') || ' ' || 
    coalesce(phone, '')));
```

---

## 6. MULTI-TENANT ARCHITECTURE

### 6.1 Tenant Isolation Strategy

**Pooled Model с tenant_id:**
- Одна база данных
- Все таблицы содержат `tenant_id`
- Row-Level Security (RLS)
- Tenant context в каждом запросе

### 6.2 Tenant Context Flow

```
Request
  ↓
JWT Token (contains tenantId)
  ↓
Tenant Middleware (extract tenantId)
  ↓
Tenant Guard (validate access)
  ↓
Prisma Query (automatic tenant_id filter)
  ↓
Response
```

### 6.3 Tenant Middleware

```typescript
// common/middleware/tenant.middleware.ts
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const user = req['user'];
    
    if (user && user.tenantId) {
      req['tenantId'] = user.tenantId;
      req.headers['x-tenant-id'] = user.tenantId;
    }
    
    next();
  }
}
```

### 6.4 Prisma Tenant Extension

```typescript
// database/prisma-tenant.extension.ts
export const tenantExtension = (tenantId: string) =>
  Prisma.defineExtension((prisma) =>
    prisma.$extends({
      query: {
        $allModels: {
          async $allOperations({ args, query, operation, model }) {
            // Автоматически добавляем tenantId ко всем запросам
            if (operation === 'findMany' || operation === 'findFirst') {
              args.where = { ...args.where, tenantid: tenantId };
            }
            
            if (operation === 'create') {
              args.data = { ...args.data, tenantid: tenantId };
            }
            
            return query(args);
          },
        },
      },
    }),
  );
```

---

## 7. SECURITY ARCHITECTURE

### 7.1 Security Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    1. Network Layer                          │
│  - HTTPS/TLS                                                 │
│  - Rate Limiting                                             │
│  - DDoS Protection                                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                2. Authentication Layer                       │
│  - JWT Access Token (15min)                                  │
│  - Refresh Token (30 days)                                   │
│  - Token Blacklist (Redis)                                   │
│  - MFA (optional for Owner)                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                3. Authorization Layer                        │
│  - RBAC (Role-Based Access Control)                         │
│  - Dynamic Permissions                                       │
│  - Tenant Isolation                                          │
│  - Restaurant-level access control                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                 4. Data Protection Layer                     │
│  - Encryption at rest (AES-256)                             │
│  - Encryption in transit (TLS 1.3)                          │
│  - Password hashing (bcrypt, 10 rounds)                     │
│  - Sensitive data masking                                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  5. Audit & Monitoring                       │
│  - Activity Log (всё действия)                              │
│  - Failed login attempts                                     │
│  - Permission changes                                        │
│  - Data access logs                                          │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 JWT Token Structure

```typescript
// Access Token (15 minutes)
{
  "userId": "uuid",
  "email": "user@example.com",
  "role": "RESTAURANTADMIN",
  "tenantId": "uuid",
  "permissions": ["readguest", "createguest", ...],
  "restaurantIds": ["uuid1", "uuid2"],
  "iat": 1234567890,
  "exp": 1234568790
}

// Refresh Token (30 days)
{
  "userId": "uuid",
  "tenantId": "uuid",
  "tokenId": "uuid", // для возможности revoke
  "iat": 1234567890,
  "exp": 1237159890
}
```

### 7.3 Rate Limiting Strategy

```typescript
// Rate limits по endpoint
const RATE_LIMITS = {
  '/auth/login': {
    points: 5,
    duration: 900, // 15 minutes
    blockDuration: 1800, // 30 minutes
  },
  '/auth/register': {
    points: 3,
    duration: 3600, // 1 hour
  },
  '/api/*': {
    points: 100,
    duration: 60, // 1 minute
  },
  '/api/analytics/*': {
    points: 20,
    duration: 60,
  },
};
```

---

## 8. INTEGRATION ARCHITECTURE

### 8.1 POS Integration Flow

#### PUSH Model (Webhook от POS)
```
POS System (iiko/R-Keeper)
  ↓
Webhook: POST /webhooks/pos/transaction
  ↓
Validate signature
  ↓
Find GuestCard by phone
  ↓
Calculate balls (via LoyaltyCalculationService)
  ↓
Create BallTransaction
  ↓
Update GuestCard.balance
  ↓
Check level upgrade
  ↓
Create POSTransaction record
  ↓
Log to ActivityLog
  ↓
Send notification to guest
  ↓
Return success/fail to POS
```

#### PULL Model (Polling POS API)
```
Cron Job (every 5 minutes)
  ↓
GET /api/pos/guests?since=lastSyncTime
  ↓
Parse response
  ↓
For each new transaction:
  ↓
  Find/Create Guest
  ↓
  Calculate balls
  ↓
  Create BallTransaction
  ↓
  Update POSSync record
  ↓
Save lastSyncTime
```

### 8.2 iiko Integration

```typescript
// pos-integration/services/iiko.service.ts
@Injectable()
export class IikoService {
  async syncGuests(restaurantId: string, since: Date) {
    const integration = await this.getIntegration(restaurantId);
    
    // Получаем токен iiko
    const token = await this.getIikoToken(integration.apikey);
    
    // Загружаем транзакции
    const transactions = await this.fetchTransactions(token, since);
    
    for (const transaction of transactions) {
      await this.processTransaction(transaction, restaurantId);
    }
    
    await this.updateSyncTimestamp(restaurantId);
  }
  
  private async processTransaction(transaction: any, restaurantId: string) {
    // Находим гостя по телефону
    const guest = await this.guestService.findByPhone(
      transaction.guestPhone,
      restaurantId
    );
    
    if (!guest) return; // Пропускаем незарегистрированных
    
    // Рассчитываем баллы
    const balls = await this.loyaltyService.calculateBalls(
      transaction.sum,
      guest.card.levelId,
      restaurantId
    );
    
    // Создаём транзакцию
    await this.ballTransactionService.earnBalls({
      guestCardId: guest.card.id,
      amount: balls,
      checkAmount: transaction.sum,
      posTransactionId: transaction.id,
      restaurantId,
    });
  }
}
```

### 8.3 Telegram Integration

```typescript
// telegram/telegram-bot.service.ts
@Injectable()
export class TelegramBotService {
  constructor(
    @InjectBot() private bot: Telegraf<Context>,
    private guestService: GuestService,
  ) {}
  
  async sendBallsEarnedNotification(
    telegramId: string,
    amount: number,
    balance: number,
    restaurantName: string,
  ) {
    await this.bot.telegram.sendMessage(
      telegramId,
      `🎉 Вы получили ${amount} баллов в ${restaurantName}!\n\n` +
      `💰 Текущий баланс: ${balance} баллов`,
      {
        reply_markup: {
          inline_keyboard: [
            [
              { 
                text: '💳 Открыть карту', 
                web_app: { url: process.env.MINI_APP_URL } 
              }
            ]
          ]
        }
      }
    );
  }
  
  async sendPromoNotification(
    telegramId: string,
    promo: LoyaltyPromo,
    granted: boolean,
  ) {
    if (granted) {
      await this.bot.telegram.sendMessage(
        telegramId,
        `🎁 ${promo.name}\n\n` +
        `Вам начислено ${promo.ballamount} баллов!\n\n` +
        `Акция действует до ${format(promo.validuntil, 'dd.MM.yyyy')}`,
      );
    } else {
      await this.bot.telegram.sendMessage(
        telegramId,
        `📢 Новая акция: ${promo.name}\n\n` +
        `${promo.description}\n\n` +
        `Получите ${promo.ballamount} баллов при выполнении условий!`,
      );
    }
  }
}
```

---

## 9. ANALYTICS & REPORTING ARCHITECTURE

### 9.1 Analytics Data Flow

```
Operational Database (PostgreSQL)
  ↓
ETL Process (Bull Queue Jobs)
  ↓
Aggregated Analytics Tables
  ├── DailyAnalytics
  ├── MonthlyAnalytics
  └── GuestAnalytics
  ↓
Redis Cache (hot data, 1 hour TTL)
  ↓
API Endpoints
  ↓
Frontend Dashboard
```

### 9.2 Analytics Tables Schema

```prisma
model DailyAnalytics {
  id                String   @id @default(uuid())
  tenantid          String
  restaurantid      String?
  date              DateTime @db.Date
  
  // Метрики
  totalchecks       Int      @default(0)
  totalamount       Decimal  @default(0)
  totalvisits       Int      @default(0)
  activeguests      Int      @default(0)
  newguests         Int      @default(0)
  returningguests   Int      @default(0)
  averagecheck      Decimal  @default(0)
  ballsearned       Int      @default(0)
  ballsredeemed     Int      @default(0)
  
  createdat         DateTime @default(now())
  
  @@unique([tenantid, restaurantid, date])
  @@index([tenantid, restaurantid, date])
}

model MonthlyAnalytics {
  id                String   @id @default(uuid())
  tenantid          String
  restaurantid      String?
  year              Int
  month             Int
  
  // Метрики
  totalchecks       Int      @default(0)
  totalamount       Decimal  @default(0)
  totalvisits       Int      @default(0)
  activeguests      Int      @default(0)
  newguests         Int      @default(0)
  churnrate         Decimal  @default(0)
  averagecheck      Decimal  @default(0)
  ltv               Decimal  @default(0)
  
  createdat         DateTime @default(now())
  
  @@unique([tenantid, restaurantid, year, month])
  @@index([tenantid, restaurantid, year, month])
}

model GuestAnalytics {
  id                String   @id @default(uuid())
  guestcardid       String   @unique
  tenantid          String
  
  // Метрики
  totalvisits       Int      @default(0)
  totalspent        Decimal  @default(0)
  averagecheck      Decimal  @default(0)
  lastvisit         DateTime?
  daysincelastvisi  Int?
  visitfrequency    Decimal  @default(0)
  ltv               Decimal  @default(0)
  rfmscore          String?
  
  updatedat         DateTime @updatedAt
  
  @@index([tenantid, guestcardid])
}
```

### 9.3 Analytics Jobs

```typescript
// analytics/jobs/daily-aggregation.job.ts
@Processor('analytics')
export class DailyAggregationProcessor {
  @Process('daily-aggregation')
  async handleDailyAggregation(job: Job) {
    const { tenantId, restaurantId, date } = job.data;
    
    // Агрегируем данные за день
    const analytics = await this.prisma.$queryRaw`
      SELECT 
        COUNT(DISTINCT gv.id) as totalvisits,
        COUNT(DISTINCT gv.guestcardid) as activeguests,
        SUM(gv.checkamount) as totalamount,
        AVG(gv.checkamount) as averagecheck,
        COUNT(DISTINCT CASE 
          WHEN gc.createdat::date = ${date} 
          THEN gc.id 
        END) as newguests
      FROM guest_visit gv
      JOIN guest_card gc ON gv.guestcardid = gc.id
      WHERE gv.visitdate::date = ${date}
        AND gv.tenantid = ${tenantId}
        AND gv.restaurantid = ${restaurantId}
    `;
    
    // Сохраняем в DailyAnalytics
    await this.prisma.dailyAnalytics.upsert({
      where: {
        tenantid_restaurantid_date: {
          tenantid: tenantId,
          restaurantid: restaurantId,
          date: new Date(date),
        },
      },
      create: {
        tenantid: tenantId,
        restaurantid: restaurantId,
        date: new Date(date),
        ...analytics[0],
      },
      update: analytics[0],
    });
  }
}
```

---

## 10. DEVOPS & INFRASTRUCTURE

### 10.1 Docker Architecture

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Backend API
  backend:
    build: ./apps/backend
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/maxloyalty
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      - postgres
      - redis
    volumes:
      - ./apps/backend:/app
      - /app/node_modules
    command: npm run start:dev

  # Frontend
  frontend:
    build: ./apps/frontend
    ports:
      - "5173:5173"
    environment:
      VITE_API_URL: http://localhost:3000
    volumes:
      - ./apps/frontend:/app
      - /app/node_modules
    command: npm run dev

  # Telegram Bot
  telegram-bot:
    build: ./apps/telegram-bot
    environment:
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      API_URL: http://backend:3000
    depends_on:
      - backend
    volumes:
      - ./apps/telegram-bot:/app
      - /app/node_modules

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: maxloyalty
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Bull Board (Queue monitoring)
  bull-board:
    build: ./apps/bull-board
    ports:
      - "3001:3001"
    environment:
      REDIS_URL: redis://redis:6379
    depends_on:
      - redis

  # Nginx (Reverse Proxy)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - backend
      - frontend

volumes:
  postgres_data:
  redis_data:
```

### 10.2 Deployment Architecture

```
                    ┌─────────────────┐
                    │   Load Balancer │
                    │   (Nginx/Traefik)│
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
      ┌───────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐
      │  Backend 1   │ │Backend 2 │ │ Backend 3  │
      │  (NestJS)    │ │(NestJS)  │ │  (NestJS)  │
      └───────┬──────┘ └────┬─────┘ └─────┬──────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼─────────┐
                    │   PostgreSQL     │
                    │   (Primary +     │
                    │    Replica)      │
                    └──────────────────┘
                             │
                    ┌────────▼─────────┐
                    │   Redis Cluster  │
                    │   (Master +      │
                    │    Replicas)     │
                    └──────────────────┘
```

### 10.3 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test
      - run: npm run test:e2e

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v4
        with:
          context: ./apps/backend
          push: true
          tags: ${{ secrets.DOCKER_REGISTRY }}/backend:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        uses: steebchen/kubectl@v2.0.0
        with:
          config: ${{ secrets.KUBE_CONFIG }}
          command: |
            kubectl set image deployment/backend \
              backend=${{ secrets.DOCKER_REGISTRY }}/backend:${{ github.sha }}
            kubectl rollout status deployment/backend
```

### 10.4 Monitoring & Logging

```typescript
// Prometheus metrics
import { PrometheusModule } from '@willsoto/nestjs-prometheus';

@Module({
  imports: [
    PrometheusModule.register({
      defaultMetrics: {
        enabled: true,
      },
      path: '/metrics',
    }),
  ],
})

// Custom metrics
@Injectable()
export class MetricsService {
  private readonly requestCounter: Counter;
  private readonly requestDuration: Histogram;

  constructor(private prometheus: PrometheusService) {
    this.requestCounter = new prometheus.Counter({
      name: 'http_requests_total',
      help: 'Total HTTP requests',
      labelNames: ['method', 'path', 'status'],
    });

    this.requestDuration = new prometheus.Histogram({
      name: 'http_request_duration_seconds',
      help: 'HTTP request duration',
      labelNames: ['method', 'path'],
    });
  }
}
```

---

## 📊 АРХИТЕКТУРНЫЕ ДИАГРАММЫ

### Sequence Diagram: Guest Registration

```
Guest -> Frontend: Register (phone, name)
Frontend -> Backend: POST /guests/register
Backend -> Prisma: Create User
Backend -> Prisma: Create GuestProfile
Backend -> Prisma: Create GuestCard
Backend -> QRService: Generate QR Code
Backend -> Backend: Generate 6-digit code
Backend -> LoyaltyService: Get initial level
Backend -> ActivityLog: Log registration
Backend -> Telegram: Send welcome message
Backend -> Frontend: Return GuestCard
Frontend -> Guest: Show QR + 6-digit code
```

### Sequence Diagram: Ball Transaction (POS Webhook)

```
POS -> Backend: POST /webhooks/pos/transaction
Backend -> Backend: Validate signature
Backend -> Prisma: Find GuestCard by phone
Backend -> LoyaltyService: Calculate balls
Backend -> Prisma: Create BallTransaction
Backend -> Prisma: Update GuestCard.balance
Backend -> LoyaltyService: Check level upgrade
Backend -> ActivityLog: Log transaction
Backend -> Telegram: Send notification to guest
Backend -> POS: Return success
```

---

## 🎯 СЛЕДУЮЩИЕ ШАГИ

1. ✅ Общая архитектура - **Готово**
2. ⏭️ Детальная структура каждого модуля
3. ⏭️ API спецификация (OpenAPI/Swagger)
4. ⏭️ Database migrations
5. ⏭️ Testing strategy
6. ⏭️ Deployment guides

---

**Статус:** Архитектура определена, готова к детализации модулей.
**Версия:** 1.0
**Дата:** 2026-02-03
