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
