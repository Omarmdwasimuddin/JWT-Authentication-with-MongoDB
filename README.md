## JWT Authentication with MongoDB


### install @nestjs/config
```bash
npm i @nestjs/config
```
>root e .env file toiri koro
---

>Note: browse- https://www.mongodb.com/cloud/atlas/register account create koro, clusters create koro .env te database connect koro.
>[MongoDB Atlas Setup](https://github.com/Omarmdwasimuddin/mongodb-atlas)
>##

### `.env`
```bash
MONGODB_USERNAME=""
MONGODB_PASSWORD=""
MONGODB_URI="mongodb+srv://username:password@cluster0.t4iqsn7.mongodb.net"
```
---


#### Install
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


>#### auth/jwt.strategy.ts file create koro
#### `jwt.strategy.ts`
```bash
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
---


#### `auth.service.ts`
```bash
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
        const hash = await bcrypt.hash(password,10);
        const user = new this.userModel({ email, password: hash });
        await user.save();
        return { message: 'User created successfully' };
        // return user.save();
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
---


#### `auth.controller.ts`
```bash
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
---

#### `app.module.ts`
```bash
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { UserController } from './user/user.controller';
import { ProductService } from './product/product.service';
import { ProductController } from './product/product.controller';
import { MynameController } from './myname/myname.controller';
import { UserRolesController } from './user-roles/user-roles.controller';
import { ExceptionController } from './exception/exception.controller';
import { LoggerMiddleware } from './middleware/logger/logger.middleware';
import { DatabaseService } from './database/database.service';
import { DatabaseController } from './database/database.controller';
import { ConfigModule } from '@nestjs/config';
import { UserBdModule } from './user-bd/user-bd.module';
import { TypeOrmModule } from '@nestjs/typeorm';
import { EmployeeBdModule } from './employee-bd/employee-bd.module';
import { AuthModule } from './auth/auth.module';
import { MongooseModule } from '@nestjs/mongoose';


@Module({
  imports: [ ConfigModule.forRoot({
    isGlobal: true,
  }), TypeOrmModule.forRoot({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    autoLoadEntities: true,
    synchronize: true,
  }), MongooseModule.forRoot(process.env.MONGO_URL!), UserBdModule, EmployeeBdModule, AuthModule],
  controllers: [AppController, UserController, ProductController, MynameController, UserRolesController, ExceptionController, DatabaseController ],
  providers: [AppService, ProductService, DatabaseService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer){
    consumer.apply(LoggerMiddleware).forRoutes('*');
  } 
}
```
---


>## OUTPUT
>
><img width="713" height="423" alt="image" src="https://github.com/user-attachments/assets/21cd0a3a-b613-44bf-9c8e-1175bedf7d86" />
>
>#
><img width="950" height="502" alt="image" src="https://github.com/user-attachments/assets/56b5022c-a055-4738-9d65-781f300595e2" />
>
>#
><img width="725" height="482" alt="image" src="https://github.com/user-attachments/assets/e2d6fee6-dc93-411b-b0ee-b8e3511e3eb1" />
>
>#
><img width="713" height="427" alt="image" src="https://github.com/user-attachments/assets/843667df-4653-454d-b85e-22b7c7be2462" />
---
