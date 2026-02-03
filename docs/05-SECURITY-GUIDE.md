# 🔒 SECURITY GUIDE - MAX-LOYALTY

## 📋 СОДЕРЖАНИЕ

1. [Обзор безопасности](#обзор-безопасности)
2. [Аутентификация и авторизация](#аутентификация-и-авторизация)
3. [Защита данных](#защита-данных)
4. [API Security](#api-security)
5. [Защита от атак](#защита-от-атак)
6. [Безопасность платежей](#безопасность-платежей)
7. [Логирование и мониторинг](#логирование-и-мониторинг)
8. [Соответствие стандартам](#соответствие-стандартам)
9. [Security Checklist](#security-checklist)

---

## ОБЗОР БЕЗОПАСНОСТИ

### Security Architecture

```
┌─────────────────────────────────────┐
│         SECURITY LAYERS               │
├─────────────────────────────────────┤
│ 1. Network Layer                     │
│    - Firewall                        │
│    - DDoS Protection                 │
│    - SSL/TLS                         │
├─────────────────────────────────────┤
│ 2. Application Layer                 │
│    - Authentication                  │
│    - Authorization (RBAC)            │
│    - Input Validation                │
│    - Rate Limiting                   │
├─────────────────────────────────────┤
│ 3. Data Layer                        │
│    - Encryption at Rest              │
│    - Encryption in Transit           │
│    - Data Masking                    │
├─────────────────────────────────────┤
│ 4. Infrastructure Layer              │
│    - Container Security              │
│    - Secrets Management              │
│    - Backup & Recovery               │
└─────────────────────────────────────┘
```

### Security Principles

1. **Defense in Depth** - Множественные слои защиты
2. **Least Privilege** - Минимальные необходимые права
3. **Zero Trust** - Проверяй все, не доверяй ничему
4. **Fail Secure** - Безопасный отказ
5. **Security by Design** - Безопасность с самого начала

---

## АУТЕНТИФИКАЦИЯ И АВТОРИЗАЦИЯ

### JWT Authentication

#### Token Structure

```typescript
// auth/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get('JWT_SECRET'),
      algorithms: ['HS256'],
    });
  }

  async validate(payload: any) {
    if (!payload.sub || !payload.role) {
      throw new UnauthorizedException('Invalid token payload');
    }

    return {
      userId: payload.sub,
      role: payload.role,
      businessId: payload.businessId,
    };
  }
}
```

#### Refresh Token Rotation

```typescript
// auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private jwtService: JwtService,
    private configService: ConfigService,
    private prisma: PrismaService,
  ) {}

  async generateTokens(userId: string, role: string, businessId?: string) {
    const accessToken = this.jwtService.sign(
      { sub: userId, role, businessId },
      {
        secret: this.configService.get('JWT_SECRET'),
        expiresIn: '15m', // Short-lived access token
      },
    );

    const refreshToken = this.jwtService.sign(
      { sub: userId, type: 'refresh' },
      {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
        expiresIn: '30d', // Long-lived refresh token
      },
    );

    // Store refresh token hash in database
    const refreshTokenHash = await bcrypt.hash(refreshToken, 10);
    await this.prisma.refreshToken.create({
      data: {
        userId,
        token: refreshTokenHash,
        expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      },
    });

    return { accessToken, refreshToken };
  }

  async refreshTokens(refreshToken: string) {
    try {
      const payload = this.jwtService.verify(refreshToken, {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
      });

      // Find and validate stored token
      const storedTokens = await this.prisma.refreshToken.findMany({
        where: { userId: payload.sub, isRevoked: false },
      });

      const validToken = await Promise.all(
        storedTokens.map(async (token) => {
          const isValid = await bcrypt.compare(refreshToken, token.token);
          return isValid ? token : null;
        }),
      ).then((results) => results.find((t) => t !== null));

      if (!validToken) {
        throw new UnauthorizedException('Invalid refresh token');
      }

      // Revoke old refresh token
      await this.prisma.refreshToken.update({
        where: { id: validToken.id },
        data: { isRevoked: true },
      });

      // Generate new tokens
      const user = await this.prisma.user.findUnique({
        where: { id: payload.sub },
      });

      return this.generateTokens(user.id, user.role, user.businessId);
    } catch (error) {
      throw new UnauthorizedException('Invalid or expired refresh token');
    }
  }
}
```

### Role-Based Access Control (RBAC)

#### Roles Definition

```typescript
// auth/roles.enum.ts
export enum Role {
  SUPER_ADMIN = 'SUPER_ADMIN',
  BUSINESS_OWNER = 'BUSINESS_OWNER',
  BUSINESS_ADMIN = 'BUSINESS_ADMIN',
  BUSINESS_STAFF = 'BUSINESS_STAFF',
  CUSTOMER = 'CUSTOMER',
}

export const RoleHierarchy = {
  [Role.SUPER_ADMIN]: 5,
  [Role.BUSINESS_OWNER]: 4,
  [Role.BUSINESS_ADMIN]: 3,
  [Role.BUSINESS_STAFF]: 2,
  [Role.CUSTOMER]: 1,
};
```

#### Roles Guard

```typescript
// auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role, RoleHierarchy } from '../roles.enum';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    
    return requiredRoles.some((role) => 
      RoleHierarchy[user.role] >= RoleHierarchy[role]
    );
  }
}
```

#### Usage in Controllers

```typescript
// business/business.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { Role } from '../auth/roles.enum';

@Controller('business')
@UseGuards(JwtAuthGuard, RolesGuard)
export class BusinessController {
  @Get('all')
  @Roles(Role.SUPER_ADMIN)
  async getAllBusinesses() {
    // Only SUPER_ADMIN can access
  }

  @Get('my-business')
  @Roles(Role.BUSINESS_OWNER, Role.BUSINESS_ADMIN)
  async getMyBusiness() {
    // BUSINESS_OWNER and BUSINESS_ADMIN can access
  }
}
```

### Telegram Authentication

#### Validate Telegram Init Data

```typescript
// telegram/telegram-auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class TelegramAuthService {
  constructor(private configService: ConfigService) {}

  validateTelegramWebAppData(initData: string): boolean {
    const urlParams = new URLSearchParams(initData);
    const hash = urlParams.get('hash');
    urlParams.delete('hash');

    const dataCheckString = Array.from(urlParams.entries())
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([key, value]) => `${key}=${value}`)
      .join('\n');

    const botToken = this.configService.get('TELEGRAM_BOT_TOKEN');
    const secretKey = crypto
      .createHmac('sha256', 'WebAppData')
      .update(botToken)
      .digest();

    const calculatedHash = crypto
      .createHmac('sha256', secretKey)
      .update(dataCheckString)
      .digest('hex');

    if (calculatedHash !== hash) {
      throw new UnauthorizedException('Invalid Telegram data');
    }

    // Check auth_date is not too old (max 1 hour)
    const authDate = parseInt(urlParams.get('auth_date') || '0');
    const now = Math.floor(Date.now() / 1000);
    if (now - authDate > 3600) {
      throw new UnauthorizedException('Auth data is too old');
    }

    return true;
  }
}
```

---

## ЗАЩИТА ДАННЫХ

### Encryption at Rest

#### Database Field Encryption

```typescript
// utils/encryption.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class EncryptionService {
  private algorithm = 'aes-256-gcm';
  private key: Buffer;

  constructor(private configService: ConfigService) {
    const secret = this.configService.get('ENCRYPTION_KEY');
    this.key = crypto.scryptSync(secret, 'salt', 32);
  }

  encrypt(text: string): string {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
  }

  decrypt(encryptedText: string): string {
    const [ivHex, authTagHex, encrypted] = encryptedText.split(':');
    
    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');
    const decipher = crypto.createDecipheriv(this.algorithm, this.key, iv);
    
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}
```

#### Sensitive Fields

```typescript
// prisma/schema.prisma
model Customer {
  id            String   @id @default(cuid())
  phoneNumber   String   @unique // Encrypted
  email         String?  @unique // Encrypted
  firstName     String   // Not encrypted
  lastName      String   // Not encrypted
  // ...
}

// Implementation
@Injectable()
export class CustomerService {
  constructor(
    private prisma: PrismaService,
    private encryption: EncryptionService,
  ) {}

  async createCustomer(data: CreateCustomerDto) {
    const encryptedPhone = this.encryption.encrypt(data.phoneNumber);
    const encryptedEmail = data.email 
      ? this.encryption.encrypt(data.email) 
      : null;

    return this.prisma.customer.create({
      data: {
        ...data,
        phoneNumber: encryptedPhone,
        email: encryptedEmail,
      },
    });
  }

  async findByPhone(phoneNumber: string) {
    // Note: Searching encrypted fields requires special handling
    const allCustomers = await this.prisma.customer.findMany();
    
    return allCustomers.find((customer) => {
      const decryptedPhone = this.encryption.decrypt(customer.phoneNumber);
      return decryptedPhone === phoneNumber;
    });
  }
}
```

### Password Hashing

```typescript
// auth/password.service.ts
import { Injectable } from '@nestjs/common';
import * as bcrypt from 'bcrypt';
import * as argon2 from 'argon2';

@Injectable()
export class PasswordService {
  private readonly SALT_ROUNDS = 12;

  // Using bcrypt (recommended for most cases)
  async hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, this.SALT_ROUNDS);
  }

  async comparePassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }

  // Using Argon2 (more secure, but slower)
  async hashPasswordArgon2(password: string): Promise<string> {
    return argon2.hash(password, {
      type: argon2.argon2id,
      memoryCost: 65536,
      timeCost: 3,
      parallelism: 4,
    });
  }

  async comparePasswordArgon2(password: string, hash: string): Promise<boolean> {
    return argon2.verify(hash, password);
  }

  // Password strength validation
  validatePasswordStrength(password: string): boolean {
    const minLength = 8;
    const hasUpperCase = /[A-Z]/.test(password);
    const hasLowerCase = /[a-z]/.test(password);
    const hasNumbers = /\d/.test(password);
    const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(password);

    return (
      password.length >= minLength &&
      hasUpperCase &&
      hasLowerCase &&
      hasNumbers &&
      hasSpecialChar
    );
  }
}
```

### Data Masking

```typescript
// utils/masking.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class MaskingService {
  maskEmail(email: string): string {
    const [username, domain] = email.split('@');
    if (username.length <= 3) {
      return `${username[0]}***@${domain}`;
    }
    return `${username.slice(0, 3)}***@${domain}`;
  }

  maskPhone(phone: string): string {
    if (phone.length <= 4) return '****';
    return `***${phone.slice(-4)}`;
  }

  maskCardNumber(cardNumber: string): string {
    if (cardNumber.length <= 4) return '****';
    return `****-****-****-${cardNumber.slice(-4)}`;
  }

  maskBankAccount(accountNumber: string): string {
    if (accountNumber.length <= 4) return '****';
    return `****${accountNumber.slice(-4)}`;
  }
}

// Usage in DTOs
export class CustomerResponseDto {
  id: string;
  firstName: string;
  lastName: string;
  
  @Transform(({ obj, value }) => {
    const maskingService = new MaskingService();
    return maskingService.maskEmail(value);
  })
  email: string;

  @Transform(({ obj, value }) => {
    const maskingService = new MaskingService();
    return maskingService.maskPhone(value);
  })
  phoneNumber: string;
}
```

---

## API SECURITY

### Input Validation

```typescript
// dto/create-customer.dto.ts
import { 
  IsString, 
  IsEmail, 
  IsPhoneNumber, 
  Length,
  Matches,
  IsOptional,
  IsEnum,
} from 'class-validator';
import { Transform } from 'class-transformer';
import { ApiProperty } from '@nestjs/swagger';

export class CreateCustomerDto {
  @ApiProperty({ example: '+79001234567' })
  @IsPhoneNumber('RU')
  @Transform(({ value }) => value.trim())
  phoneNumber: string;

  @ApiProperty({ example: 'user@example.com' })
  @IsEmail()
  @IsOptional()
  @Transform(({ value }) => value?.toLowerCase().trim())
  email?: string;

  @ApiProperty({ example: 'Иван' })
  @IsString()
  @Length(1, 50)
  @Matches(/^[a-zA-Zа-яА-Я\s-]+$/, {
    message: 'First name can only contain letters, spaces and hyphens',
  })
  @Transform(({ value }) => value.trim())
  firstName: string;

  @ApiProperty({ example: 'Иванов' })
  @IsString()
  @Length(1, 50)
  @Matches(/^[a-zA-Zа-яА-Я\s-]+$/, {
    message: 'Last name can only contain letters, spaces and hyphens',
  })
  @Transform(({ value }) => value.trim())
  lastName: string;
}
```

### SQL Injection Prevention

```typescript
// Using Prisma (automatically prevents SQL injection)
// ✅ SAFE
await this.prisma.customer.findMany({
  where: {
    phoneNumber: userInput, // Prisma uses parameterized queries
  },
});

// ❌ DANGEROUS (if using raw queries)
await this.prisma.$queryRaw`
  SELECT * FROM customers WHERE phone = ${userInput}
`; // DON'T DO THIS!

// ✅ SAFE (if you must use raw queries)
await this.prisma.$queryRaw`
  SELECT * FROM customers WHERE phone = ${Prisma.sql`${userInput}`}
`;
```

### XSS Prevention

```typescript
// utils/sanitizer.service.ts
import { Injectable } from '@nestjs/common';
import * as DOMPurify from 'isomorphic-dompurify';

@Injectable()
export class SanitizerService {
  sanitizeHtml(html: string): string {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
      ALLOWED_ATTR: ['href'],
    });
  }

  escapeHtml(text: string): string {
    return text
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;')
      .replace(/\//g, '&#x2F;');
  }
}
```

### CSRF Protection

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import * as csurf from 'csurf';
import * as cookieParser from 'cookie-parser';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.use(cookieParser());
  app.use(
    csurf({
      cookie: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
      },
    }),
  );
  
  await app.listen(3000);
}
```

### Rate Limiting

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

// app.module.ts
import { ThrottlerModule } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,  // 1 second
        limit: 10,  // 10 requests
      },
      {
        name: 'medium',
        ttl: 60000, // 1 minute
        limit: 100, // 100 requests
      },
      {
        name: 'long',
        ttl: 3600000, // 1 hour
        limit: 1000,  // 1000 requests
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}

// Custom rate limit for specific endpoints
@Controller('auth')
export class AuthController {
  @Post('login')
  @Throttle({ short: { limit: 5, ttl: 60000 } }) // 5 attempts per minute
  async login() {
    // ...
  }
}
```

---

## ЗАЩИТА ОТ АТАК

### DDoS Protection

#### nginx Configuration

```nginx
# nginx.conf
http {
  # Rate limiting zones
  limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
  limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/s;
  limit_conn_zone $binary_remote_addr zone=addr:10m;

  # Connection limits
  limit_conn addr 10;
  limit_req_status 429;

  server {
    listen 443 ssl http2;
    server_name api.max-loyalty.com;

    # General API rate limit
    location /api/ {
      limit_req zone=api burst=200 nodelay;
      proxy_pass http://backend:3000;
    }

    # Stricter limit for auth endpoints
    location /api/auth/ {
      limit_req zone=auth burst=10 nodelay;
      proxy_pass http://backend:3000;
    }

    # Block suspicious patterns
    location ~ \.(php|asp|aspx|jsp)$ {
      return 404;
    }
  }
}
```

### Brute Force Protection

```typescript
// auth/login-attempt.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

@Injectable()
export class LoginAttemptService {
  private readonly MAX_ATTEMPTS = 5;
  private readonly BLOCK_DURATION = 15 * 60; // 15 minutes
  private readonly ATTEMPT_WINDOW = 5 * 60;  // 5 minutes

  constructor(@InjectRedis() private readonly redis: Redis) {}

  async recordFailedAttempt(identifier: string): Promise<void> {
    const key = `login_attempts:${identifier}`;
    const attempts = await this.redis.incr(key);
    
    if (attempts === 1) {
      await this.redis.expire(key, this.ATTEMPT_WINDOW);
    }

    if (attempts >= this.MAX_ATTEMPTS) {
      await this.blockUser(identifier);
    }
  }

  async blockUser(identifier: string): Promise<void> {
    const blockKey = `blocked:${identifier}`;
    await this.redis.setex(blockKey, this.BLOCK_DURATION, 'true');
  }

  async isBlocked(identifier: string): Promise<boolean> {
    const blockKey = `blocked:${identifier}`;
    const blocked = await this.redis.get(blockKey);
    return blocked === 'true';
  }

  async clearAttempts(identifier: string): Promise<void> {
    const key = `login_attempts:${identifier}`;
    await this.redis.del(key);
  }

  async getRemainingAttempts(identifier: string): Promise<number> {
    const key = `login_attempts:${identifier}`;
    const attempts = await this.redis.get(key);
    return this.MAX_ATTEMPTS - (parseInt(attempts || '0'));
  }
}
```

### Bot Detection

```typescript
// middleware/bot-detection.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class BotDetectionMiddleware implements NestMiddleware {
  private suspiciousPatterns = [
    /bot/i,
    /crawler/i,
    /spider/i,
    /scraper/i,
    /curl/i,
    /wget/i,
  ];

  use(req: Request, res: Response, next: NextFunction) {
    const userAgent = req.headers['user-agent'] || '';
    
    // Check suspicious user agents
    const isSuspicious = this.suspiciousPatterns.some(pattern => 
      pattern.test(userAgent)
    );

    if (isSuspicious) {
      // Log for monitoring
      console.warn('Suspicious bot detected:', {
        ip: req.ip,
        userAgent,
        path: req.path,
      });

      // Optionally block or challenge
      // return res.status(403).json({ message: 'Access denied' });
    }

    // Check for missing or suspicious headers
    if (!req.headers['accept'] || !req.headers['accept-language']) {
      console.warn('Missing common headers:', {
        ip: req.ip,
        path: req.path,
      });
    }

    next();
  }
}
```

---

## БЕЗОПАСНОСТЬ ПЛАТЕЖЕЙ

### PCI DSS Compliance

**Не хранить:**
- Полные номера карт
- CVV/CVC коды
- PIN коды
- Магнитные дорожки

**Можно хранить:**
- Последние 4 цифры карты
- Имя держателя
- Срок действия (только месяц/год)

### Webhook Signature Verification

```typescript
// payments/webhook.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class WebhookService {
  constructor(private configService: ConfigService) {}

  // Stripe webhook verification
  verifyStripeSignature(
    payload: string,
    signature: string,
  ): boolean {
    const secret = this.configService.get('STRIPE_WEBHOOK_SECRET');
    
    try {
      const expectedSignature = crypto
        .createHmac('sha256', secret)
        .update(payload)
        .digest('hex');

      return crypto.timingSafeEqual(
        Buffer.from(signature),
        Buffer.from(expectedSignature),
      );
    } catch (error) {
      throw new UnauthorizedException('Invalid webhook signature');
    }
  }

  // YooKassa webhook verification
  verifyYooKassaSignature(
    payload: any,
    signature: string,
  ): boolean {
    const secret = this.configService.get('YOOKASSA_SECRET_KEY');
    
    const data = JSON.stringify(payload);
    const expectedSignature = crypto
      .createHmac('sha256', secret)
      .update(data)
      .digest('hex');

    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expectedSignature),
    );
  }
}
```

### Idempotency

```typescript
// payments/idempotency.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  ConflictException,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const idempotencyKey = request.headers['idempotency-key'];

    if (!idempotencyKey) {
      return next.handle();
    }

    const cacheKey = `idempotency:${idempotencyKey}`;
    const cachedResponse = await this.redis.get(cacheKey);

    if (cachedResponse) {
      // Return cached response
      return new Observable((observer) => {
        observer.next(JSON.parse(cachedResponse));
        observer.complete();
      });
    }

    // Check if request is in progress
    const lockKey = `idempotency_lock:${idempotencyKey}`;
    const isLocked = await this.redis.set(lockKey, '1', 'EX', 60, 'NX');

    if (!isLocked) {
      throw new ConflictException('Request already in progress');
    }

    return next.handle().pipe(
      tap(async (response) => {
        // Cache response for 24 hours
        await this.redis.setex(
          cacheKey,
          86400,
          JSON.stringify(response),
        );
        await this.redis.del(lockKey);
      }),
    );
  }
}
```

---

## ЛОГИРОВАНИЕ И МОНИТОРИНГ

### Security Event Logging

```typescript
// logging/security-logger.service.ts
import { Injectable } from '@nestjs/common';
import { InjectPinoLogger, PinoLogger } from 'nestjs-pino';

export enum SecurityEventType {
  LOGIN_SUCCESS = 'login_success',
  LOGIN_FAILED = 'login_failed',
  LOGOUT = 'logout',
  PASSWORD_CHANGE = 'password_change',
  PASSWORD_RESET = 'password_reset',
  ACCOUNT_LOCKED = 'account_locked',
  TOKEN_REFRESH = 'token_refresh',
  UNAUTHORIZED_ACCESS = 'unauthorized_access',
  PERMISSION_DENIED = 'permission_denied',
  DATA_ACCESS = 'data_access',
  DATA_MODIFICATION = 'data_modification',
  PAYMENT_INITIATED = 'payment_initiated',
  PAYMENT_COMPLETED = 'payment_completed',
  PAYMENT_FAILED = 'payment_failed',
  SUSPICIOUS_ACTIVITY = 'suspicious_activity',
}

@Injectable()
export class SecurityLoggerService {
  constructor(
    @InjectPinoLogger(SecurityLoggerService.name)
    private readonly logger: PinoLogger,
  ) {}

  logSecurityEvent(
    eventType: SecurityEventType,
    userId: string | null,
    details: Record<string, any>,
    request?: Request,
  ) {
    const logData = {
      eventType,
      userId,
      timestamp: new Date().toISOString(),
      ip: request?.ip,
      userAgent: request?.headers['user-agent'],
      ...details,
    };

    // Log to file/service
    this.logger.info(logData, `Security event: ${eventType}`);

    // Send to SIEM if critical
    if (this.isCriticalEvent(eventType)) {
      this.sendToSIEM(logData);
    }
  }

  private isCriticalEvent(eventType: SecurityEventType): boolean {
    return [
      SecurityEventType.ACCOUNT_LOCKED,
      SecurityEventType.UNAUTHORIZED_ACCESS,
      SecurityEventType.SUSPICIOUS_ACTIVITY,
      SecurityEventType.PAYMENT_FAILED,
    ].includes(eventType);
  }

  private sendToSIEM(logData: any) {
    // Integration with Splunk, ELK, etc.
    // Implementation depends on your SIEM solution
  }
}
```

### Audit Trail

```typescript
// audit/audit.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

export enum AuditAction {
  CREATE = 'CREATE',
  UPDATE = 'UPDATE',
  DELETE = 'DELETE',
  VIEW = 'VIEW',
  EXPORT = 'EXPORT',
}

@Injectable()
export class AuditService {
  constructor(private prisma: PrismaService) {}

  async logAction(
    userId: string,
    action: AuditAction,
    entityType: string,
    entityId: string,
    changes?: Record<string, any>,
    metadata?: Record<string, any>,
  ) {
    return this.prisma.auditLog.create({
      data: {
        userId,
        action,
        entityType,
        entityId,
        changes: changes ? JSON.stringify(changes) : null,
        metadata: metadata ? JSON.stringify(metadata) : null,
        timestamp: new Date(),
      },
    });
  }

  async getAuditTrail(
    entityType: string,
    entityId: string,
  ) {
    return this.prisma.auditLog.findMany({
      where: { entityType, entityId },
      orderBy: { timestamp: 'desc' },
      include: { user: true },
    });
  }
}
```

---

## СООТВЕТСТВИЕ СТАНДАРТАМ

### GDPR Compliance

```typescript
// gdpr/gdpr.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class GDPRService {
  constructor(private prisma: PrismaService) {}

  // Right to Access
  async exportUserData(userId: string) {
    const user = await this.prisma.customer.findUnique({
      where: { id: userId },
      include: {
        loyaltyCards: true,
        transactions: true,
        rewards: true,
      },
    });

    return {
      personalData: user,
      exportDate: new Date().toISOString(),
    };
  }

  // Right to be Forgotten
  async deleteUserData(userId: string) {
    // Anonymize instead of delete (for audit compliance)
    return this.prisma.customer.update({
      where: { id: userId },
      data: {
        firstName: 'DELETED',
        lastName: 'USER',
        email: `deleted_${userId}@anonymized.com`,
        phoneNumber: '+0000000000',
        deletedAt: new Date(),
        gdprDeletedAt: new Date(),
      },
    });
  }

  // Right to Rectification
  async updateUserData(userId: string, data: any) {
    return this.prisma.customer.update({
      where: { id: userId },
      data: {
        ...data,
        lastModified: new Date(),
      },
    });
  }

  // Data Portability
  async generateDataExport(userId: string) {
    const data = await this.exportUserData(userId);
    
    // Generate JSON file
    const json = JSON.stringify(data, null, 2);
    
    // Or generate CSV
    // const csv = this.jsonToCsv(data);
    
    return json;
  }
}
```

### Федеральный закон № 152-ФЗ (Персональные данные)

```typescript
// consent/consent.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

export enum ConsentType {
  DATA_PROCESSING = 'DATA_PROCESSING',
  MARKETING = 'MARKETING',
  ANALYTICS = 'ANALYTICS',
  THIRD_PARTY_SHARING = 'THIRD_PARTY_SHARING',
}

@Injectable()
export class ConsentService {
  constructor(private prisma: PrismaService) {}

  async recordConsent(
    userId: string,
    consentType: ConsentType,
    granted: boolean,
  ) {
    return this.prisma.userConsent.create({
      data: {
        userId,
        consentType,
        granted,
        grantedAt: granted ? new Date() : null,
        revokedAt: !granted ? new Date() : null,
        ipAddress: '...', // From request
        userAgent: '...', // From request
      },
    });
  }

  async checkConsent(
    userId: string,
    consentType: ConsentType,
  ): Promise<boolean> {
    const consent = await this.prisma.userConsent.findFirst({
      where: { userId, consentType },
      orderBy: { createdAt: 'desc' },
    });

    return consent?.granted ?? false;
  }
}
```

---

## SECURITY CHECKLIST

### Перед запуском

- [ ] Все секреты в переменных окружения
- [ ] SSL/TLS сертификаты установлены
- [ ] HTTPS обязателен для production
- [ ] CORS настроен правильно
- [ ] Rate limiting включен
- [ ] Helmet.js для заголовков безопасности
- [ ] CSRF защита включена
- [ ] SQL injection защита (используйте Prisma)
- [ ] XSS защита (санитайзация ввода)
- [ ] Пароли хешируются (bcrypt/argon2)
- [ ] JWT tokens с коротким TTL
- [ ] Refresh token rotation
- [ ] Чувствительные данные зашифрованы
- [ ] Бэкапы настроены
- [ ] Logging и monitoring работают
- [ ] Security headers настроены
- [ ] Документация API не содержит секретов

### Регулярное обслуживание

- [ ] Обновление зависимостей
- [ ] Security audit logs review
- [ ] Проверка уязвимостей
- [ ] Penetration testing
- [ ] Ротация секретов
- [ ] Проверка SSL сертификатов

---

**Версия:** 1.0  
**Дата:** 2026-02-03  
**Статус:** Готово к использованию
