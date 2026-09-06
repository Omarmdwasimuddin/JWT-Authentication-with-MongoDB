## JWT Authentication with MongoDB

#### Install
```bash
npm i @nestjs/jwt passport-jwt @nestjs/passport passport
```
```bash
npm i bcrypt
```
```bash
npm i --save-dev @types/bcrypt
```
---


#### Create module, service & controller
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
---


#### `user.schema.ts`
```bash
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
---

#### `.env`
```bash
# .env
# DATABASE_URL=postgresql://postgres.tpawbhcettriiskwfjwl:q5Ml5Z4fMLqnsgzT@aws-1-ap-southeast-2.pooler.supabase.com:5432/postgres
# SUPABASE_JWT_SECRET=T+sfZYNnyCfjlXvUwS9gh2FD/l/fZjYqDNi79DnbWwdWdLvPFmLE0W7Jt0PJ9v2YhO7mpobrxm0Ti1Klhyr6MA==

# .env
MONGO_URL=mongodb+srv://mdwasimu015_db_user:KSavMq0fMHVqQUQF@cluster0.kt25fpa.mongodb.net/?appName=Cluster0
# mongodbPassword=KSavMq0fMHVqQUQF

JWT_SECRET=wasim123secretkey
```
---


#### `auth.module.ts`
```bash
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { UserSchemaFactory, UserSchema } from './user.schema';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtStrategy } from './jwt.strategy';

@Module({
  imports: [MongooseModule.forFeature([{ name: UserSchema.name, schema: UserSchemaFactory}]),
  JwtModule.registerAsync({
    imports: [ConfigModule],
    inject: [ConfigService],
    useFactory: (config: ConfigService)=> ({
      secret: config.get<string>('JWT_SECRET'),
      signOptions: { expiresIn: '1h' },
    })
  })
],
  providers: [AuthService, JwtStrategy], ##add-koro-JwtStrategy
  controllers: [AuthController]
})
export class AuthModule {}

```
---
