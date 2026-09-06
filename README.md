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
