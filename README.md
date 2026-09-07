# JWT Authentication with MongoDB 

এই গাইডে দেখানো হবে কীভাবে NestJS + MongoDB দিয়ে একটা পুরোপুরি কাজ করা **JWT-based Authentication system** বানানো যায় — signup, login, আর `passport-jwt` দিয়ে সুরক্ষিত (`protected`) profile route।

---

## ধাপ ১: `@nestjs/config` Install করা

```bash
npm i @nestjs/config
```

Project-এর root-এ একটা `.env` file বানাও।

> **নোট:** [MongoDB Atlas](https://www.mongodb.com/cloud/atlas/register)-এ account বানাও, cluster তৈরি করো, এরপর `.env`-এ database connect করো। বিস্তারিত: [MongoDB Atlas Setup](https://github.com/Omarmdwasimuddin/mongodb-atlas)।

### `.env`

```bash
MONGODB_USERNAME=""
MONGODB_PASSWORD=""
MONGODB_URI="mongodb+srv://username:password@cluster0.t4iqsn7.mongodb.net"
```

---

## ধাপ ২: প্রয়োজনীয় Package Install করা

```bash
npm i @nestjs/mongoose mongoose
```

```bash
npm i @nestjs/jwt passport-jwt @nestjs/passport passport
```

```bash
npm i bcrypt
```

```bash
npm i --save-dev @types/bcrypt
```

### কোনটা কী কাজে লাগে

| Package | কাজ |
|---|---|
| `@nestjs/mongoose`, `mongoose` | MongoDB-এর সাথে কাজ করার জন্য |
| `@nestjs/jwt` | JWT token generate/sign করার জন্য |
| `passport-jwt`, `@nestjs/passport`, `passport` | JWT token verify করে route protect করার জন্য (Strategy pattern) |
| `bcrypt` | Password hash করার জন্য (plain text password কখনো database-এ save করা উচিত না) |
| `@types/bcrypt` | TypeScript-এ `bcrypt`-এর জন্য type definition (dev dependency) |

---

## ধাপ ৩: Module, Service, Controller তৈরি করা

```bash
nest g module auth
```

```bash
nest g service auth
```

```bash
nest g controller auth
```

```bash
nest g class auth/user.schema --flat
```

শেষ command দিয়ে `auth` folder-এর ভিতরেই (আলাদা subfolder ছাড়া, `--flat` flag-এর কারণে) `user.schema.ts` file তৈরি হবে।

---

## ধাপ ৪: `app.module.ts` Setup করা

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { AuthModule } from './auth/auth.module';
import { ConfigModule } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [ ConfigModule.forRoot({ isGlobal: true }), MongooseModule.forRoot(process.env.MONGODB_URI!), AuthModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`ConfigModule.forRoot({ isGlobal: true })` দিয়ে `ConfigService` পুরো application জুড়ে (including `AuthModule`-এর ভিতরের file গুলোতে) ব্যবহার করা যাবে, বারবার import করতে হবে না।

---

## ধাপ ৫: User Schema লেখা

### `user.schema.ts`

```ts
import { Prop, Schema, SchemaFactory } from "@nestjs/mongoose";
import { Document } from "mongoose";

export type UserDocument = UserSchema & Document;

@Schema()
export class UserSchema {
    @Prop({ required: true, unique: true })
    email!: string;

    @Prop({ required: true })
    password!: string;
}

export const UserSchemaFactory = SchemaFactory.createForClass(UserSchema);
```

**লক্ষ্য করার মতো বিষয়:** এখানে schema class-এর নামই `UserSchema` রাখা হয়েছে (সাধারণত এই class-এর নাম `User` রাখা হয়, আর schema variable-এর নাম `UserSchema` রাখা হয় — যেমন আগের গাইডগুলোতে `Student`/`StudentSchema` বা `Employee` দেখেছ)। এখানে নামকরণটা একটু ভিন্ন (`UserSchema` class + `UserSchemaFactory` variable), কিন্তু যতক্ষণ পুরো codebase-এ এই একই নাম সামঞ্জস্যপূর্ণভাবে ব্যবহার হচ্ছে (নিচে `auth.module.ts` আর `auth.service.ts`-এ দেখা যাবে), ততক্ষণ এটা কাজ করবে ঠিকভাবেই। `email`-এ `unique: true` দেওয়া আছে, মানে একই email দিয়ে দুইবার signup করা যাবে না।

### `.env`-এ JWT Secret যোগ করা

```bash
MONGODB_USERNAME=""
MONGODB_PASSWORD=""
MONGODB_URI="mongodb+srv://username:password@cluster0.t4iqsn7.mongodb.net"
JWT_SECRET=wasim123secretkey
```

> ⚠️ **নিরাপত্তা নোট:** এটা শুধু demo-এর জন্য simple secret। বাস্তব project-এ `JWT_SECRET` হওয়া উচিত একটা লম্বা, random, unpredictable string (যেমন `openssl rand -base64 32` দিয়ে generate করা), আর কখনোই source code-এ commit করা উচিত না — শুধু `.env`-এ থাকা উচিত এবং `.gitignore`-এ `.env` যোগ করা থাকা উচিত।

---

## ধাপ ৬: JWT Strategy লেখা

`auth/jwt.strategy.ts` নামের file তৈরি করো।

### `jwt.strategy.ts`

```ts
import { PassportStrategy } from "@nestjs/passport";
import { Strategy, ExtractJwt } from "passport-jwt";
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
    constructor(configService: ConfigService) {
        super({
            jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
            secretOrKey: configService.get<string>("JWT_SECRET"),
        });
    }
    async validate(payload: any) {
        return { userId: payload.sub, email: payload.email };
    }
}
```

**এখানে কী হচ্ছে:**
- `PassportStrategy(Strategy)` extend করে — `passport-jwt`-এর `Strategy`-কে NestJS-এর সাথে integrate করা হচ্ছে
- `jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken()` — বলে দিচ্ছে token কোথা থেকে নিতে হবে: `Authorization: Bearer <token>` header থেকে
- `secretOrKey` — token verify করতে যে secret লাগবে, সেটা `.env` থেকে নেওয়া হচ্ছে
- `validate(payload)` — token verify সফল হলে এই method call হয়; `payload`-এ যা কিছু ছিল (login-এর সময় `sign()` করা data), সেখান থেকে দরকারি অংশ বের করে return করা হয় — এই return value-ই পরে `request.user`-এ চলে যায়

---

## ধাপ ৭: Module Setup করা

### `auth.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { UserSchemaFactory, UserSchema } from './user.schema';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtStrategy } from './jwt.strategy';

@Module({
  imports: [MongooseModule.forFeature([{ name: UserSchema.name, schema: UserSchemaFactory }]),
  JwtModule.registerAsync({
    imports: [ConfigModule],
    inject: [ConfigService],
    useFactory: (config: ConfigService) => ({
      secret: config.get<string>('JWT_SECRET'),
      signOptions: { expiresIn: '1h' },
    })
  })
],
  providers: [AuthService, JwtStrategy],
  controllers: [AuthController]
})
export class AuthModule {}
```

**এখানে কী হচ্ছে:**
- `MongooseModule.forFeature()` — `UserSchema` model register করা হচ্ছে
- `JwtModule.registerAsync()` — JWT module-কে async ভাবে configure করা হচ্ছে, কারণ secret আসছে `ConfigService` থেকে (যেটা নিজেও async ভাবে load হয়)
  - `secret` — token sign/verify করার জন্য secret key
  - `signOptions: { expiresIn: '1h' }` — token-এর মেয়াদ ১ ঘণ্টা, এরপর expire হয়ে যাবে
- `providers`-এ `JwtStrategy`-ও যোগ করতে হবে — এটা যোগ না করলে DI Guard/Strategy-কে resolve করতে পারবে না

---

## ধাপ ৮: Service লেখা (Signup/Login Logic)

### `auth.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { UserDocument, UserSchema } from './user.schema';
import { Model } from 'mongoose';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
    constructor(
        @InjectModel(UserSchema.name) private userModel: Model<UserDocument>, private jwtService: JwtService,
    ) {}

    async signup(email: string, password: string) {
        const hash = await bcrypt.hash(password, 10);
        const user = new this.userModel({ email, password: hash });
        await user.save();
        return { message: 'User created successfully' };
    }
    
    async login(email: string, password: string) {
        const user = await this.userModel.findOne({ email });
        if (!user) return null;
        const isPasswordValid = await bcrypt.compare(password, user.password);
        if (!isPasswordValid) return null;
        const payload = { email: user.email, sub: user._id };
        return {
            access_token: this.jwtService.sign(payload),
        };
    }
}
```

### `signup()` কী করছে

1. `bcrypt.hash(password, 10)` দিয়ে password hash করা হচ্ছে — `10` হলো "salt rounds" (hashing কতবার/কত জটিলভাবে হবে, বেশি হলে বেশি secure কিন্তু বেশি সময় লাগে)
2. Hash করা password দিয়ে নতুন user document বানিয়ে save করা হচ্ছে
3. Plain text password কখনোই database-এ save হয় না — শুধু hash-টাই থাকে

### `login()` কী করছে

1. Email দিয়ে user খোঁজা হয়
2. User না পেলে `null` return (কোনো specific error message না দেওয়াই ভালো — নাহলে attacker বুঝে যাবে কোন email আছে আর কোনটা নেই)
3. `bcrypt.compare(password, user.password)` দিয়ে দেওয়া password আর database-এ থাকা hash মিলিয়ে দেখা হয়
4. না মিললে `null`
5. মিললে একটা `payload` বানানো হয় (`email` আর `sub` — `sub` মানে "subject", JWT-এর standard convention অনুযায়ী এখানে user-এর `_id` বসানো হয়)
6. `jwtService.sign(payload)` দিয়ে একটা signed JWT token বানিয়ে `access_token` হিসেবে return করা হয়

---

## ধাপ ৯: Controller লেখা (Route)

### `auth.controller.ts`

```ts
import { Body, Controller, Get, Post, Request, UseGuards } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthGuard } from '@nestjs/passport';

@Controller('auth')
export class AuthController {
    constructor(private authService: AuthService) {}

    @Post('signup')
    async signup(@Body() body: { email: string; password: string }) {
        return this.authService.signup(body.email, body.password);
    }

    @Post('login')
    async login(@Body() body: { email: string; password: string }) {
        return this.authService.login(body.email, body.password);
    }

    @UseGuards(AuthGuard('jwt'))
    @Get('profile')
    getProfile(@Request() req) {
        return req.user;
    }
}
```

### Route-গুলো একনজরে

| HTTP Method | Route | কাজ | Protected? |
|---|---|---|---|
| `POST` | `/auth/signup` | নতুন user তৈরি (hashed password সহ) | না |
| `POST` | `/auth/login` | Login করে `access_token` পাওয়া | না |
| `GET` | `/auth/profile` | Logged-in user-এর তথ্য দেখা | **হ্যাঁ** (`AuthGuard('jwt')`) |

`@UseGuards(AuthGuard('jwt'))` — এখানে `'jwt'` নামটা `JwtStrategy`-র সাথে সংযুক্ত (Passport internally strategy-গুলো নাম দিয়ে চেনে)। এই Guard token verify করবে, verify সফল হলে `JwtStrategy`-এর `validate()`-এর return value-টাই `req.user`-এ বসে যাবে, যেটা `getProfile()`-এ ফেরত দেওয়া হচ্ছে।

---

## Output (উদাহরণ)

Signup, login, আর protected profile route call করার output নিচে দেখানো হলো:

![Signup output](https://github.com/user-attachments/assets/21cd0a3a-b613-44bf-9c8e-1175bedf7d86)

![Login output — access_token পাওয়া](https://github.com/user-attachments/assets/56b5022c-a055-4738-9d65-781f300595e2)

![Token ছাড়া protected route call করলে Unauthorized](https://github.com/user-attachments/assets/e2d6fee6-dc93-411b-b0ee-b8e3511e3eb1)

![Token সহ protected route call করলে profile data পাওয়া](https://github.com/user-attachments/assets/843667df-4653-454d-b85e-22b7c7be2462)

---
