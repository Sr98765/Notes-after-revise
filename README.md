# backend-revise

=========================================================================
nvm install 22
nvm use 22
node -v
==========================================
mkdir backend
cd backend

npm install -g @nestjs/cli
nest --version

npm install @nestjs/schedule
========================================================

sudo apt update
sudo apt install postgresql postgresql-contrib -y

sudo service postgresql start

sudo service postgresql status

sudo su postgres
psql

CREATE DATABASE viral_auth_db;
CREATE DATABASE viral_wallet_db;
CREATE DATABASE viral_game_db;
CREATE DATABASE viral_history_db;

CREATE USER viral_user WITH PASSWORD 'viral123';

GRANT ALL PRIVILEGES ON DATABASE viral_auth_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_wallet_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_game_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_history_db TO viral_user;

\c viral_auth_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\c viral_wallet_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\c viral_game_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\c viral_history_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\q
exit

psql -h localhost -U viral_user -d viral_auth_db
viral123   [password]


sudo su postgres
psql
ALTER USER viral_user CREATEDB;

===============================================================================================
===============================================================================================

nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
nest new auth-service
nest new wallet-service  
nest new game-service  
nest new history-service

[npm]

===================================================================
                                  NGINX
                                  
sudo apt install nginx -y
nginx -v

sudo nano /etc/nginx/sites-available/viral 

===================================
mkdir nginx
touch nginx/default.conf

[nginx/default.conf]

server {
    listen 80;
    server_name localhost;

    location /auth/ {
        proxy_pass http://auth:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /wallet/ {
        proxy_pass http://wallet:3002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /game/ {
        proxy_pass http://game:3003;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /history/ {
        proxy_pass http://history:3004;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /health/auth {
        proxy_pass http://auth:3001/health;
        proxy_set_header Host $host;
    }

    location /health/wallet {
        proxy_pass http://wallet:3002/health;
        proxy_set_header Host $host;
    }

    location /health/game {
        proxy_pass http://game:3003/health;
        proxy_set_header Host $host;
    }

    location /health/history {
        proxy_pass http://history:3004/health;
        proxy_set_header Host $host;
    }
}

[Save with Ctrl+O, then Enter, then Ctrl+X to exit.]

cat /etc/nginx/sites-available/viral       [verify]

=================================================================================
sudo apt install jq -y       [for pretty output .json]

[backend/test.sh]

#!/bin/bash

BASE="http://localhost"
EMAIL="autotest@viral.com"
PASS="123456"

echo "--- Register ---"
curl -s -X POST $BASE/auth/register \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"password\":\"$PASS\"}" | jq

echo "--- Login ---"
RESPONSE=$(curl -s -X POST $BASE/auth/login \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"password\":\"$PASS\"}")
echo $RESPONSE | jq
TOKEN=$(echo $RESPONSE | jq -r '.token')

echo "--- Verify Token ---"
curl -s -X POST $BASE/auth/verify \
  -H "Authorization: Bearer $TOKEN" | jq

echo "--- Deposit 500 ---"
curl -s -X POST $BASE/wallet/deposit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount":500}' | jq

echo "--- Balance ---"
curl -s $BASE/wallet/balance \
  -H "Authorization: Bearer $TOKEN" | jq

echo "--- Create Player ---"
curl -s -X POST $BASE/game/player \
  -H "Content-Type: application/json" \
  -d '{"username":"autotestplayer"}' | jq

echo "--- Play Round ---"
curl -s -X POST $BASE/game/play \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"playerId":1,"bet":100}' | jq

echo "--- Final Balance ---"
curl -s $BASE/wallet/balance \
  -H "Authorization: Bearer $TOKEN" | jq

echo "--- Leaderboard ---"
curl -s $BASE/history/leaderboard | jq



====================================================================================================================
                             auth-service
=====================================================================================================================  



nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
nest new auth-service             [npm]
cd auth-service
==================
[Install Auth dependencies]
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt

npm install class-validator class-transformer  [For input validation]

npm install winston nest-winston  [For Logging]

npm install @nestjs/throttler   [Rate limiting]

npm install ioredis @nestjs/cache-manager cache-manager cache-manager-ioredis-yet      [Redis]
=======================================================================
prisma
---------
npm install prisma@4 --save-dev
npm install @prisma/client@4

npx prisma init
=============================
[.env]

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_auth_db"
JWT_SECRET=your_super_secret_key_change_this_in_production
JWT_EXPIRES_IN=1h
PORT=3001
INTERNAL_API_KEY=viral_internal_secret_key_2024
echo "REDIS_URL=redis://localhost:6379"

========================
[prisma/schema.prisma]
--------------------------
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
}


====================================================
npx prisma generate

npx prisma migrate dev --name init

npx prisma studio
========================================

npx nest g service prisma
--------------------
[src/prisma/prisma.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect()
  }

  async onModuleDestroy() {
    await this.$disconnect()
  }
}
============================
npx nest g module auth
npx nest g service auth
npx nest g controller auth
-----------------------
[src/auth/auth.module.ts]

import { Module } from '@nestjs/common'
import { JwtModule } from '@nestjs/jwt'
import { AuthService } from './auth.service'
import { AuthController } from './auth.controller'
import { PrismaService } from '../prisma/prisma.service'

@Module({
  imports: [
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: (process.env.JWT_EXPIRES_IN as any) || '1h' },
    }),
  ],
  providers: [AuthService, PrismaService],
  controllers: [AuthController],
})
export class AuthModule {}

===================================
[src/auth/auth.service.ts]

import { Injectable, UnauthorizedException, ConflictException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { JwtService } from '@nestjs/jwt'
import * as bcrypt from 'bcrypt'

@Injectable()
export class AuthService {

  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async register(email: string, password: string) {
    // Check if user already exists
    const existing = await this.prisma.user.findUnique({ where: { email } })
    if (existing) {
      throw new ConflictException('Email already registered')
    }

    const hashed = await bcrypt.hash(password, 10)
    const user = await this.prisma.user.create({
      data: { email, password: hashed },
    })
    return { id: user.id, email: user.email }
  }

  async login(email: string, password: string) {
    const user = await this.prisma.user.findUnique({ where: { email } })
    if (!user) throw new UnauthorizedException('User not found')
    const valid = await bcrypt.compare(password, user.password)
    if (!valid) throw new UnauthorizedException('Invalid password')
    const token = this.jwtService.sign({ id: user.id, email: user.email })
    return { token }
  }

  async verifyToken(token: string) {
    try {
      const payload = this.jwtService.verify(token)
      return { id: payload.id, email: payload.email }
    } catch {
      throw new UnauthorizedException('Invalid or expired token')
    }
  }
}


=========================================================
[src/auth/auth.controller.ts]

import { Controller, Post, Body, Headers, UnauthorizedException, UseGuards } from '@nestjs/common'
import { AuthService } from './auth.service'
import { RegisterDto } from './dto/register.dto'
import { LoginDto } from './dto/login.dto'
import { InternalGuard } from '../guards/internal.guard'

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  register(@Body() body: RegisterDto) {
    return this.authService.register(body.email, body.password)
  }

  @Post('login')
  login(@Body() body: LoginDto) {
    return this.authService.login(body.email, body.password)
  }

  // Only internal services can call this
  @Post('verify')
  @UseGuards(InternalGuard)
  verify(@Headers('authorization') auth: string) {
    if (!auth) throw new UnauthorizedException('No token provided')
    const token = auth.split(' ')[1]
    return this.authService.verifyToken(token)
  }
}

======================================

[src/main.ts]

import { NestFactory } from '@nestjs/core'
import { AppModule } from './app.module'
import { ValidationPipe } from '@nestjs/common'

async function bootstrap() {
  const app = await NestFactory.create(AppModule)
  app.enableCors()
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))
  await app.listen(process.env.PORT || 3001) // change default per service
}
bootstrap()


=============================
[src/auth/dto/login.dto.ts]

import { IsEmail, IsString } from 'class-validator'

export class LoginDto {
  @IsEmail()
  email: string

  @IsString()
  password: string
}
===================================
[src/auth/dto/register.dto.ts]

import { IsEmail, IsString, MinLength } from 'class-validator'

export class RegisterDto {
  @IsEmail()
  email: string

  @IsString()
  @MinLength(6)
  password: string
}

====================================================
[src/middleware/logger.middleware.ts]

import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const now = new Date().toISOString()
    console.log(`[${now}] [${req.method}] ${req.originalUrl}`)
    next()
  }
}

======================================================
[src/app.module.ts]

import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common'
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler'
import { APP_GUARD } from '@nestjs/core'
import { AuthModule } from './auth/auth.module'
import { RedisModule } from './redis/redis.module'
import { HealthController } from './health.controller'
import { LoggerMiddleware } from './middleware/logger.middleware'

@Module({
  imports: [
    ThrottlerModule.forRoot([{ ttl: 60000, limit: 20 }]),
    RedisModule,
    AuthModule,
  ],
  controllers: [HealthController],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*')
  }
}

====================================================================
[ Create src/utils/retry.ts] or [mkdir -p /workspaces/backend-revise/backend/auth-service/src/utils
touch /workspaces/backend-revise/backend/auth-service/src/utils/retry.ts]

export async function callWithRetry<T>(
  fn: () => Promise<T>,
  retries = 3,
  delayMs = 200,
): Promise<T> {
  let lastError: any

  for (let i = 0; i < retries; i++) {
    try {
      return await fn()
    } catch (err) {
      lastError = err
      const wait = delayMs * 2 ** i
      console.warn(`Retry ${i + 1}/${retries} after ${wait}ms...`)
      await new Promise(r => setTimeout(r, wait))
    }
  }

  throw lastError
}
==========================================================
[src/guards/internal.guard.ts]

import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common'

@Injectable()
export class InternalGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest()
    const apiKey = request.headers['x-internal-key']

    if (!apiKey || apiKey !== process.env.INTERNAL_API_KEY) {
      throw new UnauthorizedException('Internal access only')
    }

    return true
  }
}

========================================================================
[src/health.controller.ts]

import { Controller, Get } from '@nestjs/common'

@Controller('health')
export class HealthController {
  @Get()
  check() {
    return {
      status: 'ok',
      service: 'auth-service',
      timestamp: new Date().toISOString(),
    }
  }
}


=======================================================================
                      Redis

[src/redis/redis.module.ts]

import { Module, Global } from '@nestjs/common'
import { RedisService } from './redis.service'

@Global()
@Module({
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}

===========================================================

[src/redis/redis.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common'
import Redis from 'ioredis'

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis
  private readonly logger = new Logger(RedisService.name)

  async onModuleInit() {
    this.client = new Redis(process.env.REDIS_URL || 'redis://localhost:6379')

    this.client.on('connect', () => {
      this.logger.log('Redis connected')
    })

    this.client.on('error', (err) => {
      this.logger.error('Redis error', err)
    })
  }

  async onModuleDestroy() {
    await this.client.quit()
  }

  async get(key: string): Promise<string | null> {
    return this.client.get(key)
  }

  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.client.set(key, value, 'EX', ttlSeconds)
    } else {
      await this.client.set(key, value)
    }
  }

  async del(key: string): Promise<void> {
    await this.client.del(key)
  }

  async delPattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern)
    if (keys.length > 0) {
      await this.client.del(...keys)
    }
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key)
    return result === 1
  }
}


=============================================
npx prisma generate
npx prisma migrate dev --name init

npm run start:dev
npx prisma studio
==========================
[Register]

curl -X POST http://localhost:3001/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

[Login]

curl -X POST http://localhost:3001/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

[verify]

curl -X POST http://localhost:3001/auth/verify \
-H "Authorization: Bearer YOUR_JWT_TOKEN"
===========================================================================================================================================
====================================================================================================================
                             Wallet-service
=====================================================================================================================  
cd -


nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
cd wallet-service
==========================

npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
npm install axios

npm install class-validator class-transformer  [For input validation]

npm install winston nest-winston  [For Logging]

npm install @nestjs/throttler   [Rate limiting]

npm install ioredis @nestjs/cache-manager cache-manager cache-manager-ioredis-yet      [Redis]

npm install prisma@4 --save-dev
npm install @prisma/client@4

npx prisma init

======================================
[.env]

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_wallet_db"
AUTH_SERVICE_URL=http://localhost:3001
PORT=3002
INTERNAL_API_KEY=viral_internal_secret_key_2024
echo "REDIS_URL=redis://localhost:6379"


==========================================
 [prisma/schema.prisma]
 
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Wallet {
  id           Int           @id @default(autoincrement())
  userId       Int           @unique
  balance      Decimal       @default(0) @db.Decimal(18, 2)
  transactions Transaction[]
}

model Transaction {
  id        Int      @id @default(autoincrement())
  walletId  Int
  amount    Decimal  @db.Decimal(18, 2)
  type      String
  createdAt DateTime @default(now())

  wallet    Wallet   @relation(fields: [walletId], references: [id])
}


==============================================================================================
npx prisma generate

npx prisma migrate dev --name init
===============================================

npx nest g service prisma
-------------------------------
[src/prisma/prisma.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect()
  }

  async onModuleDestroy() {
    await this.$disconnect()
  }
}
=====================================================================

npx nest g module wallet
npx nest g service wallet
npx nest g controller wallet
--------------------------------------
[src/wallet/wallet.module.ts]

import { Module } from '@nestjs/common'
import { WalletService } from './wallet.service'
import { WalletController } from './wallet.controller'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'

@Module({
  providers: [WalletService, PrismaService, RedisService],
  controllers: [WalletController],
})
export class WalletModule {}


========================================================================

[src/wallet/wallet.service.ts]

import { Injectable, BadRequestException, Logger } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'
import { Decimal } from '@prisma/client/runtime/library'

const BALANCE_TTL = 300        // 5 minutes
const TRANSACTIONS_TTL = 120   // 2 minutes

@Injectable()
export class WalletService {
  private readonly logger = new Logger(WalletService.name)

  constructor(
    private prisma: PrismaService,
    private redis: RedisService,
  ) {}

  async getBalance(userId: number) {
    const cacheKey = `wallet:balance:${userId}`

    try {
      const cached = await this.redis.get(cacheKey)
      if (cached) {
        this.logger.log(`Cache HIT for ${cacheKey}`)
        return { balance: parseFloat(cached), cached: true }
      }
      this.logger.log(`Cache MISS for ${cacheKey}`)
    } catch (err) {
      this.logger.warn(`Redis error, falling back to DB: ${err.message}`)
    }

    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: {},
      create: { userId, balance: new Decimal(0) },
    })

    const balance = wallet.balance.toNumber()

    try {
      await this.redis.set(cacheKey, balance.toString(), BALANCE_TTL)
      this.logger.log(`Cache SET for ${cacheKey} TTL=${BALANCE_TTL}s`)
    } catch (err) {
      this.logger.warn(`Redis SET failed: ${err.message}`)
    }

    return { balance, cached: false }
  }

  async deposit(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Deposit amount must be greater than 0')

    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: { balance: { increment: new Decimal(amount) } },
      create: { userId, balance: new Decimal(amount) },
    })

    await this.prisma.transaction.create({
      data: { walletId: wallet.id, type: 'deposit', amount: new Decimal(amount) },
    })

    await this.redis.del(`wallet:balance:${userId}`)
    await this.redis.del(`wallet:transactions:${userId}`)

    return { balance: wallet.balance.toNumber() }
  }

  async withdraw(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Withdraw amount must be greater than 0')

    const wallet = await this.prisma.wallet.findUnique({ where: { userId } })
    if (!wallet) throw new BadRequestException('Wallet not found')
    if (wallet.balance.toNumber() < amount) throw new BadRequestException('Insufficient balance')

    const updated = await this.prisma.wallet.update({
      where: { userId },
      data: { balance: { decrement: new Decimal(amount) } },
    })

    await this.prisma.transaction.create({
      data: { walletId: wallet.id, type: 'withdraw', amount: new Decimal(amount) },
    })

    await this.redis.del(`wallet:balance:${userId}`)
    await this.redis.del(`wallet:transactions:${userId}`)

    return { balance: updated.balance.toNumber() }
  }

  async getTransactions(userId: number) {
    const cacheKey = `wallet:transactions:${userId}`

    try {
      const cached = await this.redis.get(cacheKey)
      if (cached) {
        this.logger.log(`Cache HIT for ${cacheKey}`)
        return JSON.parse(cached)
      }
    } catch (err) {
      this.logger.warn(`Redis error: ${err.message}`)
    }

    const wallet = await this.prisma.wallet.findUnique({ where: { userId } })
    if (!wallet) return []

    const transactions = await this.prisma.transaction.findMany({
      where: { walletId: wallet.id },
      orderBy: { createdAt: 'desc' },
    })

    const result = transactions.map(t => ({
      ...t,
      amount: t.amount.toNumber(),
    }))

    try {
      await this.redis.set(cacheKey, JSON.stringify(result), TRANSACTIONS_TTL)
    } catch (err) {
      this.logger.warn(`Redis SET failed: ${err.message}`)
    }

    return result
  }
}

=========================================================================

[src/wallet/wallet.controller.ts]

import {
  Controller, Get, Post, Body, Headers,
  BadRequestException, UnauthorizedException
} from '@nestjs/common'
import { WalletService } from './wallet.service'
import { AmountDto } from './dto/amount.dto'
import { callWithRetry } from '../utils/retry'
import axios from 'axios'

const AUTH_SERVICE_URL = process.env.AUTH_SERVICE_URL || 'http://localhost:3001'
const INTERNAL_API_KEY = process.env.INTERNAL_API_KEY || ''

@Controller('wallet')
export class WalletController {
  constructor(private walletService: WalletService) {}

  private async getUserId(authHeader: string): Promise<number> {
    if (!authHeader) throw new BadRequestException('No Authorization header provided')
    try {
      const response = await callWithRetry(() =>
        axios.post(
          `${AUTH_SERVICE_URL}/auth/verify`,
          {},
          {
            headers: {
              authorization: authHeader,
              'x-internal-key': INTERNAL_API_KEY,
            },
          },
        )
      )
      return response.data.id
    } catch {
      throw new UnauthorizedException('Invalid or expired token')
    }
  }

  @Get('balance')
  async getBalance(@Headers('authorization') auth: string) {
    const userId = await this.getUserId(auth)
    return this.walletService.getBalance(userId)
  }

  @Post('deposit')
  async deposit(@Headers('authorization') auth: string, @Body() body: AmountDto) {
    const userId = await this.getUserId(auth)
    return this.walletService.deposit(userId, body.amount)
  }

  @Post('withdraw')
  async withdraw(@Headers('authorization') auth: string, @Body() body: AmountDto) {
    const userId = await this.getUserId(auth)
    return this.walletService.withdraw(userId, body.amount)
  }

  @Get('transactions')
  async getTransactions(@Headers('authorization') auth: string) {
    const userId = await this.getUserId(auth)
    return this.walletService.getTransactions(userId)
  }
}


==================================================================================

[src/main.ts]

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common'

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))
  await app.listen(3002)
}
bootstrap();

======================================================
[src/app.module.ts]

import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common'
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler'
import { APP_GUARD } from '@nestjs/core'
import { WalletModule } from './wallet/wallet.module'
import { RedisModule } from './redis/redis.module'
import { HealthController } from './health.controller'
import { LoggerMiddleware } from './middleware/logger.middleware'

@Module({
  imports: [
    ThrottlerModule.forRoot([{ ttl: 60000, limit: 30 }]),
    RedisModule,
    WalletModule,
  ],
  controllers: [HealthController],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*')
  }
}



==========================================
[src/wallet/dto/amount.dto.ts]

import { IsNumber, IsPositive } from 'class-validator'

export class AmountDto {
  @IsNumber()
  @IsPositive()
  amount: number
}
=========================================================
[src/utils/retry.ts]

export async function callWithRetry<T>(
  fn: () => Promise<T>,
  retries = 3,
  delayMs = 200,
): Promise<T> {
  let lastError: any

  for (let i = 0; i < retries; i++) {
    try {
      return await fn()
    } catch (err) {
      lastError = err
      const wait = delayMs * 2 ** i
      console.warn(`Retry ${i + 1}/${retries} after ${wait}ms...`)
      await new Promise(r => setTimeout(r, wait))
    }
  }

  throw lastError
}

==========================================================
[src/middleware/logger.middleware.ts]

import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const now = new Date().toISOString()
    console.log(`[${now}] [${req.method}] ${req.originalUrl}`)
    next()
  }
}
==================================================
[src/health.controller.ts]

import { Controller, Get } from '@nestjs/common'

@Controller('health')
export class HealthController {
  @Get()
  check() {
    return {
      status: 'ok',
      service: 'wallet-service',
      timestamp: new Date().toISOString(),
    }
  }
}

======================================================================================
                      Redis

[src/redis/redis.module.ts]

import { Module, Global } from '@nestjs/common'
import { RedisService } from './redis.service'

@Global()
@Module({
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}

===========================================================

[src/redis/redis.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common'
import Redis from 'ioredis'

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis
  private readonly logger = new Logger(RedisService.name)

  async onModuleInit() {
    this.client = new Redis(process.env.REDIS_URL || 'redis://localhost:6379')

    this.client.on('connect', () => {
      this.logger.log('Redis connected')
    })

    this.client.on('error', (err) => {
      this.logger.error('Redis error', err)
    })
  }

  async onModuleDestroy() {
    await this.client.quit()
  }

  async get(key: string): Promise<string | null> {
    return this.client.get(key)
  }

  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.client.set(key, value, 'EX', ttlSeconds)
    } else {
      await this.client.set(key, value)
    }
  }

  async del(key: string): Promise<void> {
    await this.client.del(key)
  }

  async delPattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern)
    if (keys.length > 0) {
      await this.client.del(...keys)
    }
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key)
    return result === 1
  }
}


====================================================================

npx prisma generate
npx prisma migrate dev --name init

npm run start:dev
npx prisma studio

========================================
[Register]
curl -X POST http://localhost:3001/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

[Login]
curl -X POST http://localhost:3001/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

[Balance]
curl -X GET http://localhost:3002/wallet/balance \
-H "Authorization: Bearer JWT_TOKEN"

[Deposit]
curl -X POST http://localhost:3002/wallet/deposit \
-H "Authorization: Bearer JWT_TOKEN" \
-H "Content-Type: application/json" \
-d '{"amount":100}'

[Withdraw]
curl -X POST http://localhost:3002/wallet/withdraw \
-H "Authorization: Bearer JWT_TOKEN" \
-H "Content-Type: application/json" \
-d '{"amount":50}'

[Transactions]
curl -X GET http://localhost:3002/wallet/transactions \
-H "Authorization: Bearer JWT_TOKEN"

================================================================================================================
================================================================================================================
                                  Game-service setup
=============================================================================================================
cd -

nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
cd game-service
=====================================================================
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
npm install axios

npm install class-validator class-transformer  [For input validation]

npm install winston nest-winston  [For Logging]

npm install @nestjs/throttler   [Rate limiting]

npm install ioredis @nestjs/cache-manager cache-manager cache-manager-ioredis-yet      [Redis]

npm install prisma@4 --save-dev
npm install @prisma/client@4

npx prisma init
======================================================
[.env]

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_game_db"
AUTH_SERVICE_URL=http://localhost:3001
WALLET_SERVICE_URL=http://localhost:3002
HISTORY_SERVICE_URL=http://localhost:3004
PORT=3003
INTERNAL_API_KEY=viral_internal_secret_key_2024
echo "REDIS_URL=redis://localhost:6379"

==========================================================
prisma/schema.prisma
-----------------------

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Player {
  id       Int         @id @default(autoincrement())
  username String      @unique
  rounds   GameRound[]
}

model GameRound {
  id        Int      @id @default(autoincrement())
  playerId  Int
  bet       Decimal  @db.Decimal(18, 2)
  result    String
  payout    Decimal  @db.Decimal(18, 2)
  createdAt DateTime @default(now())

  player    Player   @relation(fields: [playerId], references: [id])
}

model GameTransaction {
  id        Int      @id @default(autoincrement())
  playerId  Int
  bet       Decimal  @db.Decimal(18, 2)
  status    String   @default("pending")
  result    String?
  payout    Decimal  @default(0) @db.Decimal(18, 2)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}



============================================================================

npx prisma generate
npx prisma migrate dev --name init
npx prisma studio

==================================================================

npx nest g service prisma
==================================
[src/prisma/prisma.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
=====================================================

npx nest g module game
npx nest g service game
npx nest g controller game

========================================
[src/game/game.module.ts]

import { Module } from '@nestjs/common'
import { GameService } from './game.service'
import { GameController } from './game.controller'
import { PrismaService } from '../prisma/prisma.service'
import { GameCron } from './game.cron'

@Module({
  imports: [],
  providers: [GameService, PrismaService, GameCron],
  controllers: [GameController],
})
export class GameModule {}

==========================================
[src/game.service.ts]

import { Injectable, BadRequestException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { callWithRetry } from '../utils/retry'
import { Decimal } from '@prisma/client/runtime/library'
import axios from 'axios'

const WALLET_SERVICE_URL = process.env.WALLET_SERVICE_URL || 'http://localhost:3002'
const HISTORY_SERVICE_URL = process.env.HISTORY_SERVICE_URL || 'http://localhost:3004'
const INTERNAL_API_KEY = process.env.INTERNAL_API_KEY || ''

@Injectable()
export class GameService {
  constructor(private prisma: PrismaService) {}

  async createPlayer(username: string) {
    return this.prisma.player.create({ data: { username } })
  }

  async getPlayers() {
    return this.prisma.player.findMany()
  }

  async playRound(playerId: number, bet: number, authHeader: string) {
    if (bet <= 0) throw new BadRequestException('Bet must be > 0')

    const txLog = await this.prisma.gameTransaction.create({
      data: { playerId, bet: new Decimal(bet), status: 'pending' },
    })

    try {
      await callWithRetry(() =>
        axios.post(
          `${WALLET_SERVICE_URL}/wallet/withdraw`,
          { amount: bet },
          {
            headers: {
              authorization: authHeader,
              'x-internal-key': INTERNAL_API_KEY,
            },
          },
        )
      )
    } catch (err) {
      await this.prisma.gameTransaction.update({
        where: { id: txLog.id },
        data: { status: 'failed' },
      })
      throw new BadRequestException(
        err.response?.data?.message || 'Wallet withdraw failed'
      )
    }

    const win = Math.random() < 0.5
    const payout = win ? bet * 2 : 0
    const result = win ? 'win' : 'lose'

    if (win) {
      try {
        await callWithRetry(() =>
          axios.post(
            `${WALLET_SERVICE_URL}/wallet/deposit`,
            { amount: payout },
            {
              headers: {
                authorization: authHeader,
                'x-internal-key': INTERNAL_API_KEY,
              },
            },
          )
        )
      } catch {
        await this.prisma.gameTransaction.update({
          where: { id: txLog.id },
          data: { status: 'failed', result, payout: new Decimal(payout) },
        })
        throw new BadRequestException('Payout failed — your win has been logged and will be retried')
      }
    }

    await this.prisma.gameTransaction.update({
      where: { id: txLog.id },
      data: { status: 'complete', result, payout: new Decimal(payout) },
    })

    const round = await this.prisma.gameRound.create({
      data: {
        playerId,
        bet: new Decimal(bet),
        result,
        payout: new Decimal(payout),
      },
    })

    const player = await this.prisma.player.findUnique({ where: { id: playerId } })
    if (player) {
      axios.post(`${HISTORY_SERVICE_URL}/history/upsert`, {
        username: player.username,
        bets: bet,
        wins: win ? payout : 0,
        losses: win ? 0 : bet,
      }).catch(() => {})
    }

    return {
      ...round,
      bet: round.bet.toNumber(),
      payout: round.payout.toNumber(),
    }
  }

  async getRounds(playerId: number) {
    const rounds = await this.prisma.gameRound.findMany({
      where: { playerId },
      orderBy: { createdAt: 'desc' },
    })
    return rounds.map(r => ({
      ...r,
      bet: r.bet.toNumber(),
      payout: r.payout.toNumber(),
    }))
  }

  async recoverPendingTransactions() {
    const cutoff = new Date(Date.now() - 30_000)
    const stuck = await this.prisma.gameTransaction.findMany({
      where: { status: 'pending', createdAt: { lt: cutoff } },
    })
    for (const tx of stuck) {
      await this.prisma.gameTransaction.update({
        where: { id: tx.id },
        data: { status: 'failed' },
      })
      console.warn(`Marked stuck transaction ${tx.id} as failed`)
    }
  }
}


==================================================================
[src/game.controller.ts]

import { Controller, Post, Body, Get, Param, Headers, BadRequestException } from '@nestjs/common'
import { GameService } from './game.service'
import { PlayDto } from './dto/play.dto'
import { CreatePlayerDto } from './dto/create-player.dto'

@Controller('game')
export class GameController {
  constructor(private gameService: GameService) {}

  @Post('player')
  createPlayer(@Body() body: CreatePlayerDto) {
    return this.gameService.createPlayer(body.username)
  }

  @Get('players')
  getPlayers() {
    return this.gameService.getPlayers()
  }

  @Post('play')
  playRound(@Headers('authorization') auth: string, @Body() body: PlayDto) {
    if (!auth) throw new BadRequestException('Authorization header required')
    return this.gameService.playRound(body.playerId, body.bet, auth)
  }

  @Get('rounds/:playerId')
  getRounds(@Param('playerId') playerId: string) {
    return this.gameService.getRounds(Number(playerId))
  }
}

=======================================================================
[install package for scheduling tasks in nestjs application / runs automatic background jobs, like a cron system]

npm install @nestjs/schedule

[src/game/game.cron.ts]

import { Injectable } from '@nestjs/common'
import { Cron, CronExpression } from '@nestjs/schedule'
import { GameService } from './game.service'

@Injectable()
export class GameCron {
  constructor(private gameService: GameService) {}

  @Cron(CronExpression.EVERY_MINUTE)
  async recoverStuckTransactions() {
    await this.gameService.recoverPendingTransactions()
  }
}
============================================================
[src/game/dto/create-player.dto.ts]

import { IsString, MinLength } from 'class-validator'

export class CreatePlayerDto {
  @IsString()
  @MinLength(2)
  username: string
}

============================================================
[src/game/dto/play.dto.ts]

import { IsNumber, IsPositive, IsInt } from 'class-validator'

export class PlayDto {
  @IsInt()
  @IsPositive()
  playerId: number

  @IsNumber()
  @IsPositive()
  bet: number
}

=================================================================
[src/middleware/logger.middleware.ts]

import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const now = new Date().toISOString()
    console.log(`[${now}] [${req.method}] ${req.originalUrl}`)
    next()
  }
}

==================================================================
[src/utils/retry.ts]

export async function callWithRetry<T>(
  fn: () => Promise<T>,
  retries = 3,
  delayMs = 200,
): Promise<T> {
  let lastError: any

  for (let i = 0; i < retries; i++) {
    try {
      return await fn()
    } catch (err) {
      lastError = err
      const wait = delayMs * 2 ** i
      console.warn(`Retry ${i + 1}/${retries} after ${wait}ms...`)
      await new Promise(r => setTimeout(r, wait))
    }
  }

  throw lastError
}

==========================================================================

[main.ts]


import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common'

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))
  await app.listen(3003); // Game-Service port
}
bootstrap();

=================================================================================
[src/app.module.ts]

import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common'
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler'
import { ScheduleModule } from '@nestjs/schedule'
import { APP_GUARD } from '@nestjs/core'
import { GameModule } from './game/game.module'
import { RedisModule } from './redis/redis.module'
import { HealthController } from './health.controller'
import { LoggerMiddleware } from './middleware/logger.middleware'

@Module({
  imports: [
    ThrottlerModule.forRoot([{ ttl: 60000, limit: 60 }]),
    ScheduleModule.forRoot(),
    RedisModule,
    GameModule,
  ],
  controllers: [HealthController],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*')
  }
}


================================================================================
[src/health.controller.ts]

import { Controller, Get } from '@nestjs/common'

@Controller('health')
export class HealthController {
  @Get()
  check() {
    return {
      status: 'ok',
      service: 'game-service',
      timestamp: new Date().toISOString(),
    }
  }
}

==========================================================================================
                      Redis

[src/redis/redis.module.ts]

import { Module, Global } from '@nestjs/common'
import { RedisService } from './redis.service'

@Global()
@Module({
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}

===========================================================

[src/redis/redis.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common'
import Redis from 'ioredis'

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis
  private readonly logger = new Logger(RedisService.name)

  async onModuleInit() {
    this.client = new Redis(process.env.REDIS_URL || 'redis://localhost:6379')

    this.client.on('connect', () => {
      this.logger.log('Redis connected')
    })

    this.client.on('error', (err) => {
      this.logger.error('Redis error', err)
    })
  }

  async onModuleDestroy() {
    await this.client.quit()
  }

  async get(key: string): Promise<string | null> {
    return this.client.get(key)
  }

  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.client.set(key, value, 'EX', ttlSeconds)
    } else {
      await this.client.set(key, value)
    }
  }

  async del(key: string): Promise<void> {
    await this.client.del(key)
  }

  async delPattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern)
    if (keys.length > 0) {
      await this.client.del(...keys)
    }
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key)
    return result === 1
  }
}

===============================================================================

npm run start:dev
=============================
[Create player]

curl -X POST http://localhost:3003/game/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'

[List players]
curl -X GET http://localhost:3003/game/players

[Play round]
curl -X POST http://localhost:3003/game/play \
-H "Content-Type: application/json" \
-d '{"playerId":1,"bet":100}'

[Get player Rounds]
curl -X GET http://localhost:3003/game/rounds/1

============================================================================================================
=========================================================================================================
                                 History-service
======================================================================================================
cd -

nvm use 22
node -v
=====================================

cd /workspaces/backend-revise/backend

cd history-service

==========================

npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt

npm install class-validator class-transformer  [For input validation]

npm install winston nest-winston  [For Logging]

npm install @nestjs/throttler   [Rate limiting]

npm install ioredis @nestjs/cache-manager cache-manager cache-manager-ioredis-yet      [Redis]

npm install prisma@4 --save-dev
npm install @prisma/client@4

npx prisma init

==========================================
[.env]

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_history_db"
PORT=3004
INTERNAL_API_KEY=viral_internal_secret_key_2024
echo "REDIS_URL=redis://localhost:6379"


===========================================
[ schema.prisma]

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model PlayerStats {
  id          Int      @id @default(autoincrement())
  username    String   @unique
  totalBets   Float    @default(0)
  totalWins   Float    @default(0)
  totalLosses Float    @default(0)
  createdAt   DateTime @default(now())
}

========================================================

npx prisma generate
npx prisma migrate dev --name init

npx prisma studio  

==========================================

npx nest g service prisma

===============================

[src/prisma/prisma.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}

======================================================

npx nest g module history
npx nest g service history
npx nest g controller history

=================================================

[src/history/history.service.ts]

import { Injectable, BadRequestException, Logger } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'

const LEADERBOARD_TTL = 120   // 2 minutes
const PLAYER_STATS_TTL = 300  // 5 minutes

@Injectable()
export class HistoryService {
  private readonly logger = new Logger(HistoryService.name)

  constructor(
    private prisma: PrismaService,
    private redis: RedisService,
  ) {}

  async getPlayerStats(username: string) {
    if (!username) throw new BadRequestException('Username is required')

    const cacheKey = `history:player:${username}`

    try {
      const cached = await this.redis.get(cacheKey)
      if (cached) {
        this.logger.log(`Cache HIT for ${cacheKey}`)
        return { ...JSON.parse(cached), cached: true }
      }
      this.logger.log(`Cache MISS for ${cacheKey}`)
    } catch (err) {
      this.logger.warn(`Redis error: ${err.message}`)
    }

    const player = await this.prisma.playerStats.findUnique({
      where: { username },
    })

    if (!player) return null

    try {
      await this.redis.set(cacheKey, JSON.stringify(player), PLAYER_STATS_TTL)
      this.logger.log(`Cache SET for ${cacheKey}`)
    } catch (err) {
      this.logger.warn(`Redis SET failed: ${err.message}`)
    }

    return player
  }

  async upsertPlayerStats(username: string, bets: number, wins: number, losses: number) {
    if (!username) throw new BadRequestException('Username is required')

    const result = await this.prisma.playerStats.upsert({
      where: { username },
      create: { username, totalBets: bets, totalWins: wins, totalLosses: losses },
      update: {
        totalBets: { increment: bets },
        totalWins: { increment: wins },
        totalLosses: { increment: losses },
      },
    })

    await this.redis.del(`history:player:${username}`)
    await this.redis.del('history:leaderboard')
    this.logger.log(`Cache invalidated for player:${username} and leaderboard`)

    return result
  }

  async getLeaderboard(limit = 10) {
    const cacheKey = 'history:leaderboard'

    try {
      const cached = await this.redis.get(cacheKey)
      if (cached) {
        this.logger.log(`Cache HIT for ${cacheKey}`)
        return JSON.parse(cached)
      }
      this.logger.log(`Cache MISS for ${cacheKey}`)
    } catch (err) {
      this.logger.warn(`Redis error: ${err.message}`)
    }

    const leaderboard = await this.prisma.playerStats.findMany({
      orderBy: { totalWins: 'desc' },
      take: limit,
    })

    try {
      await this.redis.set(cacheKey, JSON.stringify(leaderboard), LEADERBOARD_TTL)
      this.logger.log(`Cache SET for ${cacheKey}`)
    } catch (err) {
      this.logger.warn(`Redis SET failed: ${err.message}`)
    }

    return leaderboard
  }
}


===============================================

[src/history/history.controller.ts]

import { Controller, Get, Post, Body, Query } from '@nestjs/common'
import { HistoryService } from './history.service'
import { UpsertDto } from './dto/upsert.dto'

@Controller('history')
export class HistoryController {
  constructor(private historyService: HistoryService) {}

  @Get('leaderboard')
  async leaderboard() {
    return this.historyService.getLeaderboard()
  }

  @Post('upsert')
  async upsert(@Body() body: UpsertDto) {
    return this.historyService.upsertPlayerStats(
      body.username, body.bets, body.wins, body.losses
    )
  }

  // Fixed: was @Body(), now @Query()
  @Get('player')
  async getPlayer(@Query('username') username: string) {
    return this.historyService.getPlayerStats(username)
  }
}

=======================================

[src/history/history.module.ts]


import { Module } from '@nestjs/common'
import { HistoryService } from './history.service'
import { HistoryController } from './history.controller'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'

@Module({
  providers: [HistoryService, PrismaService, RedisService],
  controllers: [HistoryController],
})
export class HistoryModule {}


====================================================================
[src/history/dto/upsert.dto.ts]

import { IsString, IsNumber, Min } from 'class-validator'

export class UpsertDto {
  @IsString()
  username: string

  @IsNumber()
  @Min(0)
  bets: number

  @IsNumber()
  @Min(0)
  wins: number

  @IsNumber()
  @Min(0)
  losses: number
}

=================================================================
[src/middleware/logger.middleware.ts]

import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const now = new Date().toISOString()
    console.log(`[${now}] [${req.method}] ${req.originalUrl}`)
    next()
  }
}

=========================================================================
[src/utils/retry.ts]

export async function callWithRetry<T>(
  fn: () => Promise<T>,
  retries = 3,
  delayMs = 200,
): Promise<T> {
  let lastError: any

  for (let i = 0; i < retries; i++) {
    try {
      return await fn()
    } catch (err) {
      lastError = err
      const wait = delayMs * 2 ** i
      console.warn(`Retry ${i + 1}/${retries} after ${wait}ms...`)
      await new Promise(r => setTimeout(r, wait))
    }
  }

  throw lastError
}

===============================================
[src/main.ts]

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common'

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))
  await app.listen(3004); // history-service port
}
bootstrap();

============================================================
[src/app.module.ts]

import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common'
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler'
import { APP_GUARD } from '@nestjs/core'
import { HistoryModule } from './history/history.module'
import { RedisModule } from './redis/redis.module'
import { HealthController } from './health.controller'
import { LoggerMiddleware } from './middleware/logger.middleware'

@Module({
  imports: [
    ThrottlerModule.forRoot([{ ttl: 60000, limit: 30 }]),
    RedisModule,
    HistoryModule,
  ],
  controllers: [HealthController],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*')
  }
}


=======================================================================
[src/health.controller.ts]

import { Controller, Get } from '@nestjs/common'

@Controller('health')
export class HealthController {
  @Get()
  check() {
    return {
      status: 'ok',
      service: 'history-service',
      timestamp: new Date().toISOString(),
    }
  }
}

==========================================================================================
                      Redis

[src/redis/redis.module.ts]

import { Module, Global } from '@nestjs/common'
import { RedisService } from './redis.service'

@Global()
@Module({
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}

===========================================================

[src/redis/redis.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common'
import Redis from 'ioredis'

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis
  private readonly logger = new Logger(RedisService.name)

  async onModuleInit() {
    this.client = new Redis(process.env.REDIS_URL || 'redis://localhost:6379')

    this.client.on('connect', () => {
      this.logger.log('Redis connected')
    })

    this.client.on('error', (err) => {
      this.logger.error('Redis error', err)
    })
  }

  async onModuleDestroy() {
    await this.client.quit()
  }

  async get(key: string): Promise<string | null> {
    return this.client.get(key)
  }

  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.client.set(key, value, 'EX', ttlSeconds)
    } else {
      await this.client.set(key, value)
    }
  }

  async del(key: string): Promise<void> {
    await this.client.del(key)
  }

  async delPattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern)
    if (keys.length > 0) {
      await this.client.del(...keys)
    }
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key)
    return result === 1
  }
}

=============================================

npm run start:dev

========================================

[Upsert stats]

curl -X POST http://localhost:3004/history/upsert \
-H "Content-Type: application/json" \
-d '{"username":"player1","bets":5,"wins":3,"losses":2}'

[Get player stats]

curl -X GET http://localhost:3004/history/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'

[Get leaderboard]

curl -X GET http://localhost:3004/history/leaderboard

==============================================================================================================================
==============================================================================================================================


Step 6 — Start all 4 services and test the full flow

Open 4 terminals:

[cd backend/auth-service]
npm run start:dev

[cd wallet-service]
npm run start:dev

[cd game-service]
npm run start:dev

[cd history-service] 
npm run start:dev
=====================================
Then run the full flow:

[Register]

curl -X POST http://localhost:3001/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"player@viral.com","password":"123456"}'

[Login]
curl -X POST http://localhost:3001/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"player@viral.com","password":"123456"}'

[Set token as variable (replace with your actual token)]
TOKEN="eyJhbGci..."

[Deposit funds]
curl -X POST http://localhost:3002/wallet/deposit \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"amount":500}'

[Create a game player]
curl -X POST http://localhost:3003/game/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'

[Play a round (deducts bet, pays out if win, updates history)]
curl -X POST http://localhost:3003/game/play \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"playerId":1,"bet":100}'

[Check wallet balance — should reflect the result]
curl -X GET http://localhost:3002/wallet/balance \
-H "Authorization: Bearer $TOKEN"

[Check leaderboard]
curl -X GET http://localhost:3004/history/leaderboard
===================================================================================================================
















                                         DOCKER
==============================================================================================================

[Install Docker in Codespace]

curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
docker --version

======================================================

Create Dockerfile for each service

[auth-service/Dockerfile]

FROM node:22-slim

RUN apt-get update -y && apt-get install -y openssl ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY prisma ./prisma
RUN npx prisma generate

COPY . .
RUN npm run build

EXPOSE 3001

CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]



====================================================

[wallet-service/Dockerfile]

FROM node:22-slim

RUN apt-get update -y && apt-get install -y openssl ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY prisma ./prisma
RUN npx prisma generate

COPY . .
RUN npm run build

EXPOSE 3002

CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]


====================================================
[game-service/Dockerfile]

FROM node:22-slim

RUN apt-get update -y && apt-get install -y openssl ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY prisma ./prisma
RUN npx prisma generate

COPY . .
RUN npm run build

EXPOSE 3003

CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]


====================================================
[history-service/Dockerfile]

FROM node:22-slim

RUN apt-get update -y && apt-get install -y openssl ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY prisma ./prisma
RUN npx prisma generate

COPY . .
RUN npm run build

EXPOSE 3004

CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]


=========================================================================

Create .dockerignore for each service and add this for all

[backend/$service/.dockerignore]

node_modules
dist
.env
*.log

=========================================================================
Create the main docker-compose.yml

[backend/docker-compose.yml]

services:

  postgres:
    image: postgres:15-alpine
    container_name: viral_postgres
    restart: always
    environment:
      POSTGRES_USER: viral_user
      POSTGRES_PASSWORD: viral123
      POSTGRES_DB: viral_auth_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U viral_user -d viral_auth_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: viral_redis
    restart: always
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  auth:
    build: ./auth-service
    container_name: viral_auth
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://viral_user:viral123@postgres:5432/viral_auth_db
      JWT_SECRET: ${JWT_SECRET}
      JWT_EXPIRES_IN: ${JWT_EXPIRES_IN:-1h}
      INTERNAL_API_KEY: ${INTERNAL_API_KEY}
      REDIS_URL: redis://redis:6379
      PORT: 3001
    ports:
      - "3001:3001"
    healthcheck:
      test: ["CMD-SHELL", "node -e \"require('http').get('http://localhost:3001/health', r => process.exit(r.statusCode === 200 ? 0 : 1))\""]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s

  wallet:
    build: ./wallet-service
    container_name: viral_wallet
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      auth:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://viral_user:viral123@postgres:5432/viral_wallet_db
      AUTH_SERVICE_URL: http://auth:3001
      INTERNAL_API_KEY: ${INTERNAL_API_KEY}
      REDIS_URL: redis://redis:6379
      PORT: 3002
    ports:
      - "3002:3002"
    healthcheck:
      test: ["CMD-SHELL", "node -e \"require('http').get('http://localhost:3002/health', r => process.exit(r.statusCode === 200 ? 0 : 1))\""]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s

  game:
    build: ./game-service
    container_name: viral_game
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      wallet:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://viral_user:viral123@postgres:5432/viral_game_db
      WALLET_SERVICE_URL: http://wallet:3002
      HISTORY_SERVICE_URL: http://history:3004
      INTERNAL_API_KEY: ${INTERNAL_API_KEY}
      REDIS_URL: redis://redis:6379
      PORT: 3003
    ports:
      - "3003:3003"
    healthcheck:
      test: ["CMD-SHELL", "node -e \"require('http').get('http://localhost:3003/health', r => process.exit(r.statusCode === 200 ? 0 : 1))\""]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s

  history:
    build: ./history-service
    container_name: viral_history
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://viral_user:viral123@postgres:5432/viral_history_db
      INTERNAL_API_KEY: ${INTERNAL_API_KEY}
      REDIS_URL: redis://redis:6379
      PORT: 3004
    ports:
      - "3004:3004"
    healthcheck:
      test: ["CMD-SHELL", "node -e \"require('http').get('http://localhost:3004/health', r => process.exit(r.statusCode === 200 ? 0 : 1))\""]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s

  nginx:
    image: nginx:alpine
    container_name: viral_nginx
    restart: always
    depends_on:
      - auth
      - wallet
      - game
      - history
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"

volumes:
  postgres_data:
  
====================================================================================================

Create database init script

[backend/init-db.sql]

SELECT 'CREATE DATABASE viral_auth_db' WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'viral_auth_db')\gexec
SELECT 'CREATE DATABASE viral_wallet_db' WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'viral_wallet_db')\gexec
SELECT 'CREATE DATABASE viral_game_db' WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'viral_game_db')\gexec
SELECT 'CREATE DATABASE viral_history_db' WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'viral_history_db')\gexec

GRANT ALL PRIVILEGES ON DATABASE viral_auth_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_wallet_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_game_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_history_db TO viral_user;

=====================================================================================

# Create the .env file in the backend folder
cat > /workspaces/Notes-after-revise/backend/.env << 'EOF'
JWT_SECRET=your_super_secret_key_change_this_in_production
JWT_EXPIRES_IN=1h
INTERNAL_API_KEY=viral_internal_secret_key_2024
EOF


============================================================================================================




sed -i '/^version:/d' /workspaces/Backend-viral/backend/docker-compose.yml          [version error]
========================================================================================================
=======================================================================================================

============ Starting up each day ======================
Go to your project


cd /workspaces/Backend-viral/backend

# Start everything with one command
docker-compose up -d

# Check everything is healthy
docker-compose ps

# Watch logs if needed
docker-compose logs -f

================= Shutting down at end of day ========
cd /workspaces/Backend-viral/backend

# Stop everything but keep your data
docker-compose down

# OR stop and wipe all data to start fresh
docker-compose down -v

[If something is broken on startup]

Check which container is failing

docker-compose ps

# Check that specific service logs
docker-compose logs auth
docker-compose logs wallet
docker-compose logs game
docker-compose logs history
docker-compose logs postgres

# Restart one specific service
docker-compose restart auth

# Rebuild and restart everything from scratch
docker-compose down
docker-compose build --no-cache
docker-compose up -d

Important Codespace note
GitHub Codespaces shut down after inactivity and your Docker containers stop. When you reopen the Codespace just run:
bashcd /workspaces/Backend-viral/backend
docker-compose up -d
That's it. Your data is preserved in the postgres volume as long as you don't run docker-compose down -v.

Quick reference — all your endpoints
bash# Set token after login
TOKEN="your_token_here"

# Auth
curl -X POST http://localhost/auth/register -H "Content-Type: application/json" -d '{"email":"x@x.com","password":"123456"}'
curl -X POST http://localhost/auth/login -H "Content-Type: application/json" -d '{"email":"x@x.com","password":"123456"}'

# Wallet
curl http://localhost/wallet/balance -H "Authorization: Bearer $TOKEN"
curl -X POST http://localhost/wallet/deposit -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"amount":500}'
curl -X POST http://localhost/wallet/withdraw -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"amount":100}'
curl http://localhost/wallet/transactions -H "Authorization: Bearer $TOKEN"

# Game
curl -X POST http://localhost/game/player -H "Content-Type: application/json" -d '{"username":"player1"}'
curl http://localhost/game/players
curl -X POST http://localhost/game/play -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"playerId":1,"bet":100}'
curl http://localhost/game/rounds/1

# History
curl http://localhost/history/leaderboard
curl "http://localhost/history/player?username=player1"

# Health
curl http://localhost/health/auth
curl http://localhost/health/wallet
curl http://localhost/health/game
curl http://localhost/health/history

=======================================================================================================================================================================














                                 Errors

sudo rm -rf /workspaces/Notes-after-revise/backend/nginx/default.conf


mkdir -p /workspaces/Notes-after-revise/backend/nginx

cat > /workspaces/Notes-after-revise/backend/nginx/default.conf << 'EOF'
server {
    listen 80;
    server_name localhost;

    location /auth/ {
        proxy_pass http://auth:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /wallet/ {
        proxy_pass http://wallet:3002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /game/ {
        proxy_pass http://game:3003;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /history/ {
        proxy_pass http://history:3004;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /health/auth {
        proxy_pass http://auth:3001/health;
        proxy_set_header Host $host;
    }

    location /health/wallet {
        proxy_pass http://wallet:3002/health;
        proxy_set_header Host $host;
    }

    location /health/game {
        proxy_pass http://game:3003/health;
        proxy_set_header Host $host;
    }

    location /health/history {
        proxy_pass http://history:3004/health;
        proxy_set_header Host $host;
    }
}
EOF
