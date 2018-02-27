# Step 12: TypeORM with PostgreSQL

[//]: # (head-end)


## Server

First of all you will have to install PostgreSQL on your operating system. Since there so many options (different Linux distributions, MacOS X, Windows...) I will assume that you already know how to install a software in your OS and take that part for granted.

Then you will have to install a couple of packages:

    yarn add pg reflect-metadata typeorm
    yarn add -D @types/pg

We aren't going to use plain SQL, instead we will use an Object-relational mapping framework (ORM) called `TypeORM`.
`TypeORM` takes advantage of Typescript classes and type declarations in order to infer the db structure.

We will need to enable support for experimental decorators, emit type metadata for decorators and disable strict property initialization:

[{]: <helper> (diffStep "7.1" files="tsconfig.json" module="server")

#### [Step 7.1: TypeORM with PostgreSQL](https://github.com/Urigo/WhatsApp-Clone-Server/commit/8c4af9f)

##### Changed tsconfig.json
```diff
@@ -30,7 +30,7 @@
 ┊30┊30┊    // See https://github.com/DefinitelyTyped/DefinitelyTyped/issues/21359
 ┊31┊31┊    "strictFunctionTypes": false,             /* Enable strict checking of function types. */
 ┊32┊32┊    // "strictBindCallApply": true,           /* Enable strict 'bind', 'call', and 'apply' methods on functions. */
-┊33┊  ┊    // "strictPropertyInitialization": true,  /* Enable strict checking of property initialization in classes. */
+┊  ┊33┊    "strictPropertyInitialization": false,    /* Enable strict checking of property initialization in classes. */
 ┊34┊34┊    // "noImplicitThis": true,                /* Raise error on 'this' expressions with an implied 'any' type. */
 ┊35┊35┊    // "alwaysStrict": true,                  /* Parse in strict mode and emit "use strict" for each source file. */
 ┊36┊36┊
```
```diff
@@ -48,7 +48,7 @@
 ┊48┊48┊    // "typeRoots": [],                       /* List of folders to include type definitions from. */
 ┊49┊49┊    // "types": [],                           /* Type declaration files to be included in compilation. */
 ┊50┊50┊    // "allowSyntheticDefaultImports": true,  /* Allow default imports from modules with no default export. This does not affect code emit, just typechecking. */
-┊51┊  ┊    "esModuleInterop": true                   /* Enables emit interoperability between CommonJS and ES Modules via creation of namespace objects for all imports. Implies 'allowSyntheticDefaultImports'. */
+┊  ┊51┊    "esModuleInterop": true,                  /* Enables emit interoperability between CommonJS and ES Modules via creation of namespace objects for all imports. Implies 'allowSyntheticDefaultImports'. */
 ┊52┊52┊    // "preserveSymlinks": true,              /* Do not resolve the real path of symlinks. */
 ┊53┊53┊
 ┊54┊54┊    /* Source Map Options */
```
```diff
@@ -58,7 +58,7 @@
 ┊58┊58┊    // "inlineSources": true,                 /* Emit the source alongside the sourcemaps within a single file; requires '--inlineSourceMap' or '--sourceMap' to be set. */
 ┊59┊59┊
 ┊60┊60┊    /* Experimental Options */
-┊61┊  ┊    // "experimentalDecorators": true,        /* Enables experimental support for ES7 decorators. */
-┊62┊  ┊    // "emitDecoratorMetadata": true,         /* Enables experimental support for emitting type metadata for decorators. */
+┊  ┊61┊    "experimentalDecorators": true,           /* Enables experimental support for ES7 decorators. */
+┊  ┊62┊    "emitDecoratorMetadata": true             /* Enables experimental support for emitting type metadata for decorators. */
 ┊63┊63┊  }
 ┊64┊64┊}
```

[}]: #

The next step is to create Entities. An Entity is a class that maps to a database table. You can create a entity by defining a new class and mark it with @Entity():

[{]: <helper> (diffStep "7.1" files="entity" module="server")

#### [Step 7.1: TypeORM with PostgreSQL](https://github.com/Urigo/WhatsApp-Clone-Server/commit/8c4af9f)

##### Added entity&#x2F;Chat.ts
```diff
@@ -0,0 +1,95 @@
+┊  ┊ 1┊import {
+┊  ┊ 2┊  Entity,
+┊  ┊ 3┊  Column,
+┊  ┊ 4┊  PrimaryGeneratedColumn,
+┊  ┊ 5┊  OneToMany,
+┊  ┊ 6┊  JoinTable,
+┊  ┊ 7┊  ManyToMany,
+┊  ┊ 8┊  ManyToOne,
+┊  ┊ 9┊  CreateDateColumn
+┊  ┊10┊} from "typeorm";
+┊  ┊11┊import { Message } from "./Message";
+┊  ┊12┊import { User } from "./User";
+┊  ┊13┊import { Recipient } from "./Recipient";
+┊  ┊14┊
+┊  ┊15┊interface ChatConstructor {
+┊  ┊16┊  createdAt?: Date,
+┊  ┊17┊  name?: string;
+┊  ┊18┊  picture?: string;
+┊  ┊19┊  allTimeMembers?: User[];
+┊  ┊20┊  listingMembers?: User[];
+┊  ┊21┊  actualGroupMembers?: User[];
+┊  ┊22┊  admins?: User[];
+┊  ┊23┊  owner?: User;
+┊  ┊24┊  messages?: Message[];
+┊  ┊25┊}
+┊  ┊26┊
+┊  ┊27┊@Entity()
+┊  ┊28┊export class Chat {
+┊  ┊29┊  @PrimaryGeneratedColumn()
+┊  ┊30┊  id: number;
+┊  ┊31┊
+┊  ┊32┊  @CreateDateColumn({nullable: true})
+┊  ┊33┊  createdAt: Date;
+┊  ┊34┊
+┊  ┊35┊  @Column({nullable: true})
+┊  ┊36┊  name: string;
+┊  ┊37┊
+┊  ┊38┊  @Column({nullable: true})
+┊  ┊39┊  picture: string;
+┊  ┊40┊
+┊  ┊41┊  @ManyToMany(type => User, user => user.allTimeMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊42┊  @JoinTable()
+┊  ┊43┊  allTimeMembers: User[];
+┊  ┊44┊
+┊  ┊45┊  @ManyToMany(type => User, user => user.listingMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊46┊  @JoinTable()
+┊  ┊47┊  listingMembers: User[];
+┊  ┊48┊
+┊  ┊49┊  @ManyToMany(type => User, user => user.actualGroupMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊50┊  @JoinTable()
+┊  ┊51┊  actualGroupMembers?: User[];
+┊  ┊52┊
+┊  ┊53┊  @ManyToMany(type => User, user => user.adminChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊54┊  @JoinTable()
+┊  ┊55┊  admins?: User[];
+┊  ┊56┊
+┊  ┊57┊  @ManyToOne(type => User, user => user.ownerChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊58┊  owner?: User | null;
+┊  ┊59┊
+┊  ┊60┊  @OneToMany(type => Message, message => message.chat, {cascade: ["insert", "update"], eager: true})
+┊  ┊61┊  messages: Message[];
+┊  ┊62┊
+┊  ┊63┊  @OneToMany(type => Recipient, recipient => recipient.chat)
+┊  ┊64┊  recipients: Recipient[];
+┊  ┊65┊
+┊  ┊66┊  constructor({createdAt, name, picture, allTimeMembers, listingMembers, actualGroupMembers, admins, owner, messages}: ChatConstructor = {}) {
+┊  ┊67┊    if (createdAt) {
+┊  ┊68┊      this.createdAt = createdAt;
+┊  ┊69┊    }
+┊  ┊70┊    if (name) {
+┊  ┊71┊      this.name = name;
+┊  ┊72┊    }
+┊  ┊73┊    if (picture) {
+┊  ┊74┊      this.picture = picture;
+┊  ┊75┊    }
+┊  ┊76┊    if (allTimeMembers) {
+┊  ┊77┊      this.allTimeMembers = allTimeMembers;
+┊  ┊78┊    }
+┊  ┊79┊    if (listingMembers) {
+┊  ┊80┊      this.listingMembers = listingMembers;
+┊  ┊81┊    }
+┊  ┊82┊    if (actualGroupMembers) {
+┊  ┊83┊      this.actualGroupMembers = actualGroupMembers;
+┊  ┊84┊    }
+┊  ┊85┊    if (admins) {
+┊  ┊86┊      this.admins = admins;
+┊  ┊87┊    }
+┊  ┊88┊    if (owner) {
+┊  ┊89┊      this.owner = owner;
+┊  ┊90┊    }
+┊  ┊91┊    if (messages) {
+┊  ┊92┊      this.messages = messages;
+┊  ┊93┊    }
+┊  ┊94┊  }
+┊  ┊95┊}
```

##### Added entity&#x2F;Message.ts
```diff
@@ -0,0 +1,70 @@
+┊  ┊ 1┊import {
+┊  ┊ 2┊  Entity, Column, PrimaryGeneratedColumn, OneToMany, ManyToOne, ManyToMany, JoinTable, CreateDateColumn
+┊  ┊ 3┊} from "typeorm";
+┊  ┊ 4┊import { Chat } from "./Chat";
+┊  ┊ 5┊import { User } from "./User";
+┊  ┊ 6┊import { Recipient } from "./Recipient";
+┊  ┊ 7┊import { MessageType } from "../db";
+┊  ┊ 8┊
+┊  ┊ 9┊interface MessageConstructor {
+┊  ┊10┊  sender?: User;
+┊  ┊11┊  content?: string;
+┊  ┊12┊  createdAt?: Date,
+┊  ┊13┊  type?: MessageType;
+┊  ┊14┊  recipients?: Recipient[];
+┊  ┊15┊  holders?: User[];
+┊  ┊16┊  chat?: Chat;
+┊  ┊17┊}
+┊  ┊18┊
+┊  ┊19┊@Entity()
+┊  ┊20┊export class Message {
+┊  ┊21┊  @PrimaryGeneratedColumn()
+┊  ┊22┊  id: number;
+┊  ┊23┊
+┊  ┊24┊  @ManyToOne(type => User, user => user.senderMessages, {eager: true})
+┊  ┊25┊  sender: User;
+┊  ┊26┊
+┊  ┊27┊  @Column()
+┊  ┊28┊  content: string;
+┊  ┊29┊
+┊  ┊30┊  @CreateDateColumn({nullable: true})
+┊  ┊31┊  createdAt: Date;
+┊  ┊32┊
+┊  ┊33┊  @Column()
+┊  ┊34┊  type: number;
+┊  ┊35┊
+┊  ┊36┊  @OneToMany(type => Recipient, recipient => recipient.message, {cascade: ["insert", "update"], eager: true})
+┊  ┊37┊  recipients: Recipient[];
+┊  ┊38┊
+┊  ┊39┊  @ManyToMany(type => User, user => user.holderMessages, {cascade: ["insert", "update"], eager: true})
+┊  ┊40┊  @JoinTable()
+┊  ┊41┊  holders: User[];
+┊  ┊42┊
+┊  ┊43┊  @ManyToOne(type => Chat, chat => chat.messages)
+┊  ┊44┊  chat: Chat;
+┊  ┊45┊
+┊  ┊46┊  constructor({sender, content, createdAt, type, recipients, holders, chat}: MessageConstructor = {}) {
+┊  ┊47┊    if (sender) {
+┊  ┊48┊      this.sender = sender;
+┊  ┊49┊    }
+┊  ┊50┊    if (content) {
+┊  ┊51┊      this.content = content;
+┊  ┊52┊    }
+┊  ┊53┊    if (createdAt) {
+┊  ┊54┊      this.createdAt = createdAt;
+┊  ┊55┊    }
+┊  ┊56┊    if (type) {
+┊  ┊57┊      this.type = type;
+┊  ┊58┊    }
+┊  ┊59┊    if (recipients) {
+┊  ┊60┊      recipients.forEach(recipient => recipient.message = this);
+┊  ┊61┊      this.recipients = recipients;
+┊  ┊62┊    }
+┊  ┊63┊    if (holders) {
+┊  ┊64┊      this.holders = holders;
+┊  ┊65┊    }
+┊  ┊66┊    if (chat) {
+┊  ┊67┊      this.chat = chat;
+┊  ┊68┊    }
+┊  ┊69┊  }
+┊  ┊70┊}
```

##### Added entity&#x2F;Recipient.ts
```diff
@@ -0,0 +1,44 @@
+┊  ┊ 1┊import { Entity, ManyToOne, Column } from "typeorm";
+┊  ┊ 2┊import { Message } from "./Message";
+┊  ┊ 3┊import { User } from "./User";
+┊  ┊ 4┊import { Chat } from "./Chat";
+┊  ┊ 5┊
+┊  ┊ 6┊interface RecipientConstructor {
+┊  ┊ 7┊  user?: User;
+┊  ┊ 8┊  message?: Message;
+┊  ┊ 9┊  receivedAt?: Date;
+┊  ┊10┊  readAt?: Date;
+┊  ┊11┊}
+┊  ┊12┊
+┊  ┊13┊@Entity()
+┊  ┊14┊export class Recipient {
+┊  ┊15┊  @ManyToOne(type => User, user => user.recipients, { primary: true })
+┊  ┊16┊  user: User;
+┊  ┊17┊
+┊  ┊18┊  @ManyToOne(type => Message, message => message.recipients, { primary: true })
+┊  ┊19┊  message: Message;
+┊  ┊20┊
+┊  ┊21┊  @ManyToOne(type => Chat, chat => chat.recipients)
+┊  ┊22┊  chat: Chat;
+┊  ┊23┊
+┊  ┊24┊  @Column({nullable: true})
+┊  ┊25┊  receivedAt: Date;
+┊  ┊26┊
+┊  ┊27┊  @Column({nullable: true})
+┊  ┊28┊  readAt: Date;
+┊  ┊29┊
+┊  ┊30┊  constructor({user, message, receivedAt, readAt}: RecipientConstructor = {}) {
+┊  ┊31┊    if (user) {
+┊  ┊32┊      this.user = user;
+┊  ┊33┊    }
+┊  ┊34┊    if (message) {
+┊  ┊35┊      this.message = message;
+┊  ┊36┊    }
+┊  ┊37┊    if (receivedAt) {
+┊  ┊38┊      this.receivedAt = receivedAt;
+┊  ┊39┊    }
+┊  ┊40┊    if (readAt) {
+┊  ┊41┊      this.readAt = readAt;
+┊  ┊42┊    }
+┊  ┊43┊  }
+┊  ┊44┊}
```

##### Added entity&#x2F;User.ts
```diff
@@ -0,0 +1,75 @@
+┊  ┊ 1┊import { Entity, Column, PrimaryGeneratedColumn, ManyToMany, OneToMany } from "typeorm";
+┊  ┊ 2┊import { Chat } from "./Chat";
+┊  ┊ 3┊import { Message } from "./Message";
+┊  ┊ 4┊import { Recipient } from "./Recipient";
+┊  ┊ 5┊
+┊  ┊ 6┊interface UserConstructor {
+┊  ┊ 7┊  username?: string;
+┊  ┊ 8┊  password?: string;
+┊  ┊ 9┊  name?: string;
+┊  ┊10┊  picture?: string;
+┊  ┊11┊  phone?: string;
+┊  ┊12┊}
+┊  ┊13┊
+┊  ┊14┊@Entity()
+┊  ┊15┊export class User {
+┊  ┊16┊  @PrimaryGeneratedColumn()
+┊  ┊17┊  id: number;
+┊  ┊18┊
+┊  ┊19┊  @Column()
+┊  ┊20┊  username: string;
+┊  ┊21┊
+┊  ┊22┊  @Column()
+┊  ┊23┊  password: string;
+┊  ┊24┊
+┊  ┊25┊  @Column()
+┊  ┊26┊  name: string;
+┊  ┊27┊
+┊  ┊28┊  @Column({nullable: true})
+┊  ┊29┊  picture: string;
+┊  ┊30┊
+┊  ┊31┊  @Column({nullable: true})
+┊  ┊32┊  phone?: string;
+┊  ┊33┊
+┊  ┊34┊  @ManyToMany(type => Chat, chat => chat.allTimeMembers)
+┊  ┊35┊  allTimeMemberChats: Chat[];
+┊  ┊36┊
+┊  ┊37┊  @ManyToMany(type => Chat, chat => chat.listingMembers)
+┊  ┊38┊  listingMemberChats: Chat[];
+┊  ┊39┊
+┊  ┊40┊  @ManyToMany(type => Chat, chat => chat.actualGroupMembers)
+┊  ┊41┊  actualGroupMemberChats: Chat[];
+┊  ┊42┊
+┊  ┊43┊  @ManyToMany(type => Chat, chat => chat.admins)
+┊  ┊44┊  adminChats: Chat[];
+┊  ┊45┊
+┊  ┊46┊  @ManyToMany(type => Message, message => message.holders)
+┊  ┊47┊  holderMessages: Message[];
+┊  ┊48┊
+┊  ┊49┊  @OneToMany(type => Chat, chat => chat.owner)
+┊  ┊50┊  ownerChats: Chat[];
+┊  ┊51┊
+┊  ┊52┊  @OneToMany(type => Message, message => message.sender)
+┊  ┊53┊  senderMessages: Message[];
+┊  ┊54┊
+┊  ┊55┊  @OneToMany(type => Recipient, recipient => recipient.user)
+┊  ┊56┊  recipients: Recipient[];
+┊  ┊57┊
+┊  ┊58┊  constructor({username, password, name, picture, phone}: UserConstructor = {}) {
+┊  ┊59┊    if (username) {
+┊  ┊60┊      this.username = username;
+┊  ┊61┊    }
+┊  ┊62┊    if (password) {
+┊  ┊63┊      this.password = password;
+┊  ┊64┊    }
+┊  ┊65┊    if (name) {
+┊  ┊66┊      this.name = name;
+┊  ┊67┊    }
+┊  ┊68┊    if (picture) {
+┊  ┊69┊      this.picture = picture;
+┊  ┊70┊    }
+┊  ┊71┊    if (phone) {
+┊  ┊72┊      this.phone = phone;
+┊  ┊73┊    }
+┊  ┊74┊  }
+┊  ┊75┊}
```

[}]: #

Basic entities consist of columns and relations. Each entity MUST have a primary column.

Each entity must be registered in your connection options:

[{]: <helper> (diffStep "7.1" files="ormconfig.json" module="server")

#### [Step 7.1: TypeORM with PostgreSQL](https://github.com/Urigo/WhatsApp-Clone-Server/commit/8c4af9f)

##### Added ormconfig.json
```diff
@@ -0,0 +1,24 @@
+┊  ┊ 1┊{
+┊  ┊ 2┊   "type": "postgres",
+┊  ┊ 3┊   "host": "localhost",
+┊  ┊ 4┊   "port": 5432,
+┊  ┊ 5┊   "username": "test",
+┊  ┊ 6┊   "password": "test",
+┊  ┊ 7┊   "database": "whatsapp",
+┊  ┊ 8┊   "synchronize": true,
+┊  ┊ 9┊   "logging": false,
+┊  ┊10┊   "entities": [
+┊  ┊11┊      "entity/**/*.ts"
+┊  ┊12┊   ],
+┊  ┊13┊   "migrations": [
+┊  ┊14┊      "migration/**/*.ts"
+┊  ┊15┊   ],
+┊  ┊16┊   "subscribers": [
+┊  ┊17┊      "subscriber/**/*.ts"
+┊  ┊18┊   ],
+┊  ┊19┊   "cli": {
+┊  ┊20┊      "entitiesDir": "entity",
+┊  ┊21┊      "migrationsDir": "migration",
+┊  ┊22┊      "subscribersDir": "subscriber"
+┊  ┊23┊   }
+┊  ┊24┊}
```

[}]: #

Since database table consist of columns your entities must consist of columns too. Each entity class property you marked with @Column will be mapped to a database table column.
Each entity must have at least one primary column. There are several types of primary columns, but in our case `@PrimaryGeneratedColumn()` creates a primary column which value will be automatically generated with an auto-increment value.
`@CreateDateColumn` is a special column that is automatically set to the entity's insertion date. You don't need set this column - it will be automatically set.
For the Recipient Entity we use a composite primary key that consists of two foreign keys.

The next thing to do is to create a connection with the database before firing up the web server:

[{]: <helper> (diffStep "7.1" files="index.ts" module="server")

#### [Step 7.1: TypeORM with PostgreSQL](https://github.com/Urigo/WhatsApp-Clone-Server/commit/8c4af9f)

##### Changed index.ts
```diff
@@ -1,4 +1,5 @@
 ┊1┊1┊/// <reference path="./cloudinary.d.ts" />
+┊ ┊2┊import "reflect-metadata"; // For TypeORM
 ┊2┊3┊require('dotenv').config();
 ┊3┊4┊import { schema } from "./schema";
 ┊4┊5┊import bodyParser from "body-parser";
```
```diff
@@ -11,11 +12,11 @@
 ┊11┊12┊import passport from "passport";
 ┊12┊13┊import basicStrategy from 'passport-http';
 ┊13┊14┊import bcrypt from 'bcrypt-nodejs';
-┊14┊  ┊import { db, UserDb } from "./db";
 ┊15┊15┊import { createServer } from "http";
 ┊16┊16┊import { pubsub } from "./schema/resolvers";
-┊17┊  ┊
-┊18┊  ┊let users = db.users;
+┊  ┊17┊import { createConnection } from "typeorm";
+┊  ┊18┊import { User } from "./entity/User";
+┊  ┊19┊import { addSampleData } from "./db";
 ┊19┊20┊
 ┊20┊21┊function generateHash(password: string) {
 ┊21┊22┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
```
```diff
@@ -25,126 +26,131 @@
 ┊ 25┊ 26┊  return bcrypt.compareSync(password, localPassword);
 ┊ 26┊ 27┊}
 ┊ 27┊ 28┊
-┊ 28┊   ┊passport.use('basic-signin', new basicStrategy.BasicStrategy(
-┊ 29┊   ┊  function (username: string, password: string, done: any) {
-┊ 30┊   ┊    const user = users.find(user => user.username == username);
-┊ 31┊   ┊    if (user && validPassword(password, user.password)) {
-┊ 32┊   ┊      return done(null, user);
-┊ 33┊   ┊    }
-┊ 34┊   ┊    return done(null, false);
-┊ 35┊   ┊  }
-┊ 36┊   ┊));
-┊ 37┊   ┊
-┊ 38┊   ┊passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
-┊ 39┊   ┊  function (req: any, username: any, password: any, done: any) {
-┊ 40┊   ┊    const userExists = !!users.find(user => user.username === username);
-┊ 41┊   ┊    if (!userExists && password && req.body.name) {
-┊ 42┊   ┊      const user: UserDb = {
-┊ 43┊   ┊        id: (users.length && users[users.length - 1].id + 1) || 1,
-┊ 44┊   ┊        username,
-┊ 45┊   ┊        password: generateHash(password),
-┊ 46┊   ┊        name: req.body.name,
-┊ 47┊   ┊      };
-┊ 48┊   ┊      users.push(user);
-┊ 49┊   ┊
-┊ 50┊   ┊      pubsub.publish('userAdded', {
-┊ 51┊   ┊        userAdded: user,
-┊ 52┊   ┊      });
-┊ 53┊   ┊
-┊ 54┊   ┊      return done(null, user);
-┊ 55┊   ┊    }
-┊ 56┊   ┊    return done(null, false);
+┊   ┊ 29┊createConnection().then(async connection => {
+┊   ┊ 30┊  if (process.argv.includes('--add-sample-data')) {
+┊   ┊ 31┊    addSampleData(connection);
 ┊ 57┊ 32┊  }
-┊ 58┊   ┊));
 ┊ 59┊ 33┊
-┊ 60┊   ┊const PORT = 4000;
-┊ 61┊   ┊const CLOUDINARY_URL = process.env.CLOUDINARY_URL || '';
-┊ 62┊   ┊
-┊ 63┊   ┊const app = express();
+┊   ┊ 34┊  passport.use('basic-signin', new basicStrategy.BasicStrategy(
+┊   ┊ 35┊    async function (username: string, password: string, done: any) {
+┊   ┊ 36┊      const user = await connection.getRepository(User).findOne({where: { username }});
+┊   ┊ 37┊      if (user && validPassword(password, user.password)) {
+┊   ┊ 38┊        return done(null, user);
+┊   ┊ 39┊      }
+┊   ┊ 40┊      return done(null, false);
+┊   ┊ 41┊    }
+┊   ┊ 42┊  ));
+┊   ┊ 43┊
+┊   ┊ 44┊  passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
+┊   ┊ 45┊    async function (req: any, username: string, password: string, done: any) {
+┊   ┊ 46┊      const userExists = !!(await connection.getRepository(User).findOne({where: { username }}));
+┊   ┊ 47┊      if (!userExists && password && req.body.name) {
+┊   ┊ 48┊        const user = await connection.manager.save(new User({
+┊   ┊ 49┊          username,
+┊   ┊ 50┊          password: generateHash(password),
+┊   ┊ 51┊          name: req.body.name,
+┊   ┊ 52┊        }));
+┊   ┊ 53┊
+┊   ┊ 54┊        pubsub.publish('userAdded', {
+┊   ┊ 55┊          userAdded: user,
+┊   ┊ 56┊        });
+┊   ┊ 57┊
+┊   ┊ 58┊        return done(null, user);
+┊   ┊ 59┊      }
+┊   ┊ 60┊      return done(null, false);
+┊   ┊ 61┊    }
+┊   ┊ 62┊  ));
 ┊ 64┊ 63┊
-┊ 65┊   ┊app.use(cors());
-┊ 66┊   ┊app.use(bodyParser.json());
-┊ 67┊   ┊app.use(passport.initialize());
+┊   ┊ 64┊  const PORT = 4000;
+┊   ┊ 65┊  const CLOUDINARY_URL = process.env.CLOUDINARY_URL || '';
 ┊ 68┊ 66┊
-┊ 69┊   ┊app.post('/signup',
-┊ 70┊   ┊  passport.authenticate('basic-signup', {session: false}),
-┊ 71┊   ┊  function (req: any, res) {
-┊ 72┊   ┊    res.json(req.user);
-┊ 73┊   ┊  });
+┊   ┊ 67┊  const app = express();
 ┊ 74┊ 68┊
-┊ 75┊   ┊app.use(passport.authenticate('basic-signin', {session: false}));
+┊   ┊ 69┊  app.use(cors());
+┊   ┊ 70┊  app.use(bodyParser.json());
+┊   ┊ 71┊  app.use(passport.initialize());
 ┊ 76┊ 72┊
-┊ 77┊   ┊app.post('/signin', function (req: any, res) {
-┊ 78┊   ┊  res.json(req.user);
-┊ 79┊   ┊});
+┊   ┊ 73┊  app.post('/signup',
+┊   ┊ 74┊    passport.authenticate('basic-signup', {session: false}),
+┊   ┊ 75┊    function (req: any, res) {
+┊   ┊ 76┊      res.json(req.user);
+┊   ┊ 77┊    });
 ┊ 80┊ 78┊
-┊ 81┊   ┊const match = CLOUDINARY_URL.match(/cloudinary:\/\/(\d+):(\w+)@(\.+)/);
+┊   ┊ 79┊  app.use(passport.authenticate('basic-signin', {session: false}));
 ┊ 82┊ 80┊
-┊ 83┊   ┊if (match) {
-┊ 84┊   ┊  const [api_key, api_secret, cloud_name] = match.slice(1);
-┊ 85┊   ┊  cloudinary.config({ api_key, api_secret, cloud_name });
-┊ 86┊   ┊}
+┊   ┊ 81┊  app.post('/signin', function (req, res) {
+┊   ┊ 82┊    res.json(req.user);
+┊   ┊ 83┊  });
 ┊ 87┊ 84┊
-┊ 88┊   ┊const upload = multer({
-┊ 89┊   ┊  dest: tmp.dirSync({ unsafeCleanup: true }).name,
-┊ 90┊   ┊});
+┊   ┊ 85┊  const match = CLOUDINARY_URL.match(/cloudinary:\/\/(\d+):(\w+)@(\.+)/);
 ┊ 91┊ 86┊
-┊ 92┊   ┊app.post('/upload-profile-pic', upload.single('file'), async (req: any, res, done) => {
-┊ 93┊   ┊  try {
-┊ 94┊   ┊    res.json(await new Promise((resolve, reject) => {
-┊ 95┊   ┊      cloudinary.v2.uploader.upload(req.file.path, (error, result) => {
-┊ 96┊   ┊        if (error) {
-┊ 97┊   ┊          reject(error);
-┊ 98┊   ┊        } else {
-┊ 99┊   ┊          resolve(result);
-┊100┊   ┊        }
-┊101┊   ┊      })
-┊102┊   ┊    }));
-┊103┊   ┊  } catch (e) {
-┊104┊   ┊    done(e);
+┊   ┊ 87┊  if (match) {
+┊   ┊ 88┊    const [api_key, api_secret, cloud_name] = match.slice(1);
+┊   ┊ 89┊    cloudinary.config({ api_key, api_secret, cloud_name });
 ┊105┊ 90┊  }
-┊106┊   ┊});
 ┊107┊ 91┊
-┊108┊   ┊const apollo = new ApolloServer({
-┊109┊   ┊  schema,
-┊110┊   ┊  context(received: any) {
-┊111┊   ┊    return {
-┊112┊   ┊      currentUser: received.connection ? received.connection.context.currentUser : received.req!['user'],
-┊113┊   ┊    }
-┊114┊   ┊  },
-┊115┊   ┊  subscriptions: {
-┊116┊   ┊    onConnect: (connectionParams: any, webSocket: any) => {
-┊117┊   ┊      if (connectionParams.authToken) {
-┊118┊   ┊        // create a buffer and tell it the data coming in is base64
-┊119┊   ┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
-┊120┊   ┊        // read it back out as a string
-┊121┊   ┊        const [username, password]: string[] = buf.toString().split(':');
-┊122┊   ┊        if (username && password) {
-┊123┊   ┊          const currentUser = users.find(user => user.username == username);
-┊124┊   ┊
-┊125┊   ┊          if (currentUser && validPassword(password, currentUser.password)) {
-┊126┊   ┊            // Set context for the WebSocket
-┊127┊   ┊            return {currentUser};
+┊   ┊ 92┊  const upload = multer({
+┊   ┊ 93┊    dest: tmp.dirSync({ unsafeCleanup: true }).name,
+┊   ┊ 94┊  });
+┊   ┊ 95┊
+┊   ┊ 96┊  app.post('/upload-profile-pic', upload.single('file'), async (req: any, res, done) => {
+┊   ┊ 97┊    try {
+┊   ┊ 98┊      res.json(await new Promise((resolve, reject) => {
+┊   ┊ 99┊        cloudinary.v2.uploader.upload(req.file.path, (error, result) => {
+┊   ┊100┊          if (error) {
+┊   ┊101┊            reject(error);
 ┊128┊102┊          } else {
-┊129┊   ┊            throw new Error('Wrong credentials!');
+┊   ┊103┊            resolve(result);
+┊   ┊104┊          }
+┊   ┊105┊        })
+┊   ┊106┊      }));
+┊   ┊107┊    } catch (e) {
+┊   ┊108┊      done(e);
+┊   ┊109┊    }
+┊   ┊110┊  });
+┊   ┊111┊
+┊   ┊112┊  const apollo = new ApolloServer({
+┊   ┊113┊    schema,
+┊   ┊114┊    context(received: any) {
+┊   ┊115┊      return {
+┊   ┊116┊        currentUser: received.connection ? received.connection.context.currentUser : received.req!['user'],
+┊   ┊117┊        connection,
+┊   ┊118┊      }
+┊   ┊119┊    },
+┊   ┊120┊    subscriptions: {
+┊   ┊121┊      onConnect: async (connectionParams: any, webSocket: any) => {
+┊   ┊122┊        if (connectionParams.authToken) {
+┊   ┊123┊          // Create a buffer and tell it the data coming in is base64
+┊   ┊124┊          const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊   ┊125┊          // Read it back out as a string
+┊   ┊126┊          const [username, password]: string[] = buf.toString().split(':');
+┊   ┊127┊          if (username && password) {
+┊   ┊128┊            const currentUser = await connection.getRepository(User).findOne({where: { username }});
+┊   ┊129┊
+┊   ┊130┊            if (currentUser && validPassword(password, currentUser.password)) {
+┊   ┊131┊              // Set context for the WebSocket
+┊   ┊132┊              return {currentUser};
+┊   ┊133┊            } else {
+┊   ┊134┊              throw new Error('Wrong credentials!');
+┊   ┊135┊            }
 ┊130┊136┊          }
 ┊131┊137┊        }
+┊   ┊138┊        throw new Error('Missing auth token!');
 ┊132┊139┊      }
-┊133┊   ┊      throw new Error('Missing auth token!');
 ┊134┊140┊    }
-┊135┊   ┊  }
-┊136┊   ┊});
+┊   ┊141┊  });
 ┊137┊142┊
-┊138┊   ┊apollo.applyMiddleware({
-┊139┊   ┊  app,
-┊140┊   ┊  path: '/graphql'
-┊141┊   ┊});
+┊   ┊143┊  apollo.applyMiddleware({
+┊   ┊144┊    app,
+┊   ┊145┊    path: '/graphql'
+┊   ┊146┊  });
 ┊142┊147┊
-┊143┊   ┊// Wrap the Express server
-┊144┊   ┊const ws = createServer(app);
+┊   ┊148┊  // Wrap the Express server
+┊   ┊149┊  const ws = createServer(app);
 ┊145┊150┊
-┊146┊   ┊apollo.installSubscriptionHandlers(ws);
+┊   ┊151┊  apollo.installSubscriptionHandlers(ws);
 ┊147┊152┊
-┊148┊   ┊ws.listen(PORT, () => {
-┊149┊   ┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊153┊  ws.listen(PORT, () => {
+┊   ┊154┊    console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊155┊  });
 ┊150┊156┊});
```

[}]: #

We will also remove our fake db and replace it with some real data:

[{]: <helper> (diffStep "7.1" files="db.ts" module="server")

#### [Step 7.1: TypeORM with PostgreSQL](https://github.com/Urigo/WhatsApp-Clone-Server/commit/8c4af9f)

##### Changed db.ts
```diff
@@ -1,4 +1,11 @@
+┊  ┊ 1┊// For TypeORM
+┊  ┊ 2┊import "reflect-metadata";
+┊  ┊ 3┊import { Chat } from "./entity/Chat";
+┊  ┊ 4┊import { Recipient } from "./entity/Recipient";
 ┊ 1┊ 5┊import moment from 'moment';
+┊  ┊ 6┊import { Message } from "./entity/Message";
+┊  ┊ 7┊import { User } from "./entity/User";
+┊  ┊ 8┊import { Connection } from "typeorm";
 ┊ 2┊ 9┊
 ┊ 3┊10┊export enum MessageType {
 ┊ 4┊11┊  PICTURE,
```
```diff
@@ -6,443 +13,289 @@
 ┊  6┊ 13┊  LOCATION,
 ┊  7┊ 14┊}
 ┊  8┊ 15┊
-┊  9┊   ┊export interface UserDb {
-┊ 10┊   ┊  id: number,
-┊ 11┊   ┊  username: string,
-┊ 12┊   ┊  password: string,
-┊ 13┊   ┊  name: string,
-┊ 14┊   ┊  picture?: string | null,
-┊ 15┊   ┊  phone?: string | null,
-┊ 16┊   ┊}
-┊ 17┊   ┊
-┊ 18┊   ┊export interface ChatDb {
-┊ 19┊   ┊  id: number,
-┊ 20┊   ┊  createdAt: Date,
-┊ 21┊   ┊  name?: string | null,
-┊ 22┊   ┊  picture?: string | null,
-┊ 23┊   ┊  // All members, current and past ones.
-┊ 24┊   ┊  allTimeMemberIds: number[],
-┊ 25┊   ┊  // Whoever gets the chat listed. For groups includes past members who still didn't delete the group.
-┊ 26┊   ┊  listingMemberIds: number[],
-┊ 27┊   ┊  // Actual members of the group (they are not the only ones who get the group listed). Null for chats.
-┊ 28┊   ┊  actualGroupMemberIds?: number[] | null,
-┊ 29┊   ┊  adminIds?: number[] | null,
-┊ 30┊   ┊  ownerId?: number | null,
-┊ 31┊   ┊  messages: MessageDb[],
-┊ 32┊   ┊}
-┊ 33┊   ┊
-┊ 34┊   ┊export interface MessageDb {
-┊ 35┊   ┊  id: number,
-┊ 36┊   ┊  chatId: number,
-┊ 37┊   ┊  senderId: number,
-┊ 38┊   ┊  content: string,
-┊ 39┊   ┊  createdAt: Date,
-┊ 40┊   ┊  type: MessageType,
-┊ 41┊   ┊  recipients: RecipientDb[],
-┊ 42┊   ┊  holderIds: number[],
-┊ 43┊   ┊}
-┊ 44┊   ┊
-┊ 45┊   ┊export interface RecipientDb {
-┊ 46┊   ┊  userId: number,
-┊ 47┊   ┊  messageId: number,
-┊ 48┊   ┊  chatId: number,
-┊ 49┊   ┊  receivedAt: Date | null,
-┊ 50┊   ┊  readAt: Date | null,
-┊ 51┊   ┊}
-┊ 52┊   ┊
-┊ 53┊   ┊const users: UserDb[] = [
-┊ 54┊   ┊  {
-┊ 55┊   ┊    id: 1,
+┊   ┊ 16┊export async function addSampleData(connection: Connection) {
+┊   ┊ 17┊  const user1 = new User({
 ┊ 56┊ 18┊    username: 'ethan',
 ┊ 57┊ 19┊    password: '$2a$08$NO9tkFLCoSqX1c5wk3s7z.JfxaVMKA.m7zUDdDwEquo4rvzimQeJm', // 111
 ┊ 58┊ 20┊    name: 'Ethan Gonzalez',
 ┊ 59┊ 21┊    picture: 'https://randomuser.me/api/portraits/thumb/men/1.jpg',
 ┊ 60┊ 22┊    phone: '+391234567890',
-┊ 61┊   ┊  },
-┊ 62┊   ┊  {
-┊ 63┊   ┊    id: 2,
+┊   ┊ 23┊  });
+┊   ┊ 24┊  await connection.manager.save(user1);
+┊   ┊ 25┊
+┊   ┊ 26┊  const user2 = new User({
 ┊ 64┊ 27┊    username: 'bryan',
 ┊ 65┊ 28┊    password: '$2a$08$xE4FuCi/ifxjL2S8CzKAmuKLwv18ktksSN.F3XYEnpmcKtpbpeZgO', // 222
 ┊ 66┊ 29┊    name: 'Bryan Wallace',
 ┊ 67┊ 30┊    picture: 'https://randomuser.me/api/portraits/thumb/men/2.jpg',
 ┊ 68┊ 31┊    phone: '+391234567891',
-┊ 69┊   ┊  },
-┊ 70┊   ┊  {
-┊ 71┊   ┊    id: 3,
+┊   ┊ 32┊  });
+┊   ┊ 33┊  await connection.manager.save(user2);
+┊   ┊ 34┊
+┊   ┊ 35┊  const user3 = new User({
 ┊ 72┊ 36┊    username: 'avery',
 ┊ 73┊ 37┊    password: '$2a$08$UHgH7J8G6z1mGQn2qx2kdeWv0jvgHItyAsL9hpEUI3KJmhVW5Q1d.', // 333
 ┊ 74┊ 38┊    name: 'Avery Stewart',
 ┊ 75┊ 39┊    picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
 ┊ 76┊ 40┊    phone: '+391234567892',
-┊ 77┊   ┊  },
-┊ 78┊   ┊  {
-┊ 79┊   ┊    id: 4,
+┊   ┊ 41┊  });
+┊   ┊ 42┊  await connection.manager.save(user3);
+┊   ┊ 43┊
+┊   ┊ 44┊  const user4 = new User({
 ┊ 80┊ 45┊    username: 'katie',
 ┊ 81┊ 46┊    password: '$2a$08$wR1k5Q3T9FC7fUgB7Gdb9Os/GV7dGBBf4PLlWT7HERMFhmFDt47xi', // 444
 ┊ 82┊ 47┊    name: 'Katie Peterson',
 ┊ 83┊ 48┊    picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
 ┊ 84┊ 49┊    phone: '+391234567893',
-┊ 85┊   ┊  },
-┊ 86┊   ┊  {
-┊ 87┊   ┊    id: 5,
+┊   ┊ 50┊  });
+┊   ┊ 51┊  await connection.manager.save(user4);
+┊   ┊ 52┊
+┊   ┊ 53┊  const user5 = new User({
 ┊ 88┊ 54┊    username: 'ray',
 ┊ 89┊ 55┊    password: '$2a$08$6.mbXqsDX82ZZ7q5d8Osb..JrGSsNp4R3IKj7mxgF6YGT0OmMw242', // 555
 ┊ 90┊ 56┊    name: 'Ray Edwards',
 ┊ 91┊ 57┊    picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
 ┊ 92┊ 58┊    phone: '+391234567894',
-┊ 93┊   ┊  },
-┊ 94┊   ┊  {
-┊ 95┊   ┊    id: 6,
+┊   ┊ 59┊  });
+┊   ┊ 60┊  await connection.manager.save(user5);
+┊   ┊ 61┊
+┊   ┊ 62┊  const user6 = new User({
 ┊ 96┊ 63┊    username: 'niko',
 ┊ 97┊ 64┊    password: '$2a$08$fL5lZR.Rwf9FWWe8XwwlceiPBBim8n9aFtaem.INQhiKT4.Ux3Uq.', // 666
 ┊ 98┊ 65┊    name: 'Niccolò Belli',
 ┊ 99┊ 66┊    picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
 ┊100┊ 67┊    phone: '+391234567895',
-┊101┊   ┊  },
-┊102┊   ┊  {
-┊103┊   ┊    id: 7,
+┊   ┊ 68┊  });
+┊   ┊ 69┊  await connection.manager.save(user6);
+┊   ┊ 70┊
+┊   ┊ 71┊  const user7 = new User({
 ┊104┊ 72┊    username: 'mario',
 ┊105┊ 73┊    password: '$2a$08$nDHDmWcVxDnH5DDT3HMMC.psqcnu6wBiOgkmJUy9IH..qxa3R6YrO', // 777
 ┊106┊ 74┊    name: 'Mario Rossi',
 ┊107┊ 75┊    picture: 'https://randomuser.me/api/portraits/thumb/men/5.jpg',
 ┊108┊ 76┊    phone: '+391234567896',
-┊109┊   ┊  },
-┊110┊   ┊];
+┊   ┊ 77┊  });
+┊   ┊ 78┊  await connection.manager.save(user7);
 ┊111┊ 79┊
-┊112┊   ┊const chats: ChatDb[] = [
-┊113┊   ┊  {
-┊114┊   ┊    id: 1,
+┊   ┊ 80┊
+┊   ┊ 81┊
+┊   ┊ 82┊
+┊   ┊ 83┊  await connection.manager.save(new Chat({
 ┊115┊ 84┊    createdAt: moment().subtract(10, 'weeks').toDate(),
-┊116┊   ┊    name: null,
-┊117┊   ┊    picture: null,
-┊118┊   ┊    allTimeMemberIds: [1, 3],
-┊119┊   ┊    listingMemberIds: [1, 3],
-┊120┊   ┊    adminIds: null,
-┊121┊   ┊    ownerId: null,
+┊   ┊ 85┊    allTimeMembers: [user1, user3],
+┊   ┊ 86┊    listingMembers: [user1, user3],
 ┊122┊ 87┊    messages: [
-┊123┊   ┊      {
-┊124┊   ┊        id: 1,
-┊125┊   ┊        chatId: 1,
-┊126┊   ┊        senderId: 1,
+┊   ┊ 88┊      new Message({
+┊   ┊ 89┊        sender: user1,
 ┊127┊ 90┊        content: 'You on your way?',
 ┊128┊ 91┊        createdAt: moment().subtract(1, 'hours').toDate(),
 ┊129┊ 92┊        type: MessageType.TEXT,
+┊   ┊ 93┊        holders: [user1, user3],
 ┊130┊ 94┊        recipients: [
-┊131┊   ┊          {
-┊132┊   ┊            userId: 3,
-┊133┊   ┊            messageId: 1,
-┊134┊   ┊            chatId: 1,
-┊135┊   ┊            receivedAt: null,
-┊136┊   ┊            readAt: null,
-┊137┊   ┊          },
+┊   ┊ 95┊          new Recipient({
+┊   ┊ 96┊            user: user3,
+┊   ┊ 97┊          }),
 ┊138┊ 98┊        ],
-┊139┊   ┊        holderIds: [1, 3],
-┊140┊   ┊      },
-┊141┊   ┊      {
-┊142┊   ┊        id: 2,
-┊143┊   ┊        chatId: 1,
-┊144┊   ┊        senderId: 3,
+┊   ┊ 99┊      }),
+┊   ┊100┊      new Message({
+┊   ┊101┊        sender: user3,
 ┊145┊102┊        content: 'Yep!',
 ┊146┊103┊        createdAt: moment().subtract(1, 'hours').add(5, 'minutes').toDate(),
 ┊147┊104┊        type: MessageType.TEXT,
+┊   ┊105┊        holders: [user1, user3],
 ┊148┊106┊        recipients: [
-┊149┊   ┊          {
-┊150┊   ┊            userId: 1,
-┊151┊   ┊            messageId: 2,
-┊152┊   ┊            chatId: 1,
-┊153┊   ┊            receivedAt: null,
-┊154┊   ┊            readAt: null,
-┊155┊   ┊          },
+┊   ┊107┊          new Recipient({
+┊   ┊108┊            user: user1,
+┊   ┊109┊          }),
 ┊156┊110┊        ],
-┊157┊   ┊        holderIds: [3, 1],
-┊158┊   ┊      },
+┊   ┊111┊      }),
 ┊159┊112┊    ],
-┊160┊   ┊  },
-┊161┊   ┊  {
-┊162┊   ┊    id: 2,
+┊   ┊113┊  }));
+┊   ┊114┊
+┊   ┊115┊  await connection.manager.save(new Chat({
 ┊163┊116┊    createdAt: moment().subtract(9, 'weeks').toDate(),
-┊164┊   ┊    name: null,
-┊165┊   ┊    picture: null,
-┊166┊   ┊    allTimeMemberIds: [1, 4],
-┊167┊   ┊    listingMemberIds: [1, 4],
-┊168┊   ┊    adminIds: null,
-┊169┊   ┊    ownerId: null,
+┊   ┊117┊    allTimeMembers: [user1, user4],
+┊   ┊118┊    listingMembers: [user1, user4],
 ┊170┊119┊    messages: [
-┊171┊   ┊      {
-┊172┊   ┊        id: 1,
-┊173┊   ┊        chatId: 2,
-┊174┊   ┊        senderId: 1,
+┊   ┊120┊      new Message({
+┊   ┊121┊        sender: user1,
 ┊175┊122┊        content: 'Hey, it\'s me',
 ┊176┊123┊        createdAt: moment().subtract(2, 'hours').toDate(),
 ┊177┊124┊        type: MessageType.TEXT,
+┊   ┊125┊        holders: [user1, user4],
 ┊178┊126┊        recipients: [
-┊179┊   ┊          {
-┊180┊   ┊            userId: 4,
-┊181┊   ┊            messageId: 1,
-┊182┊   ┊            chatId: 2,
-┊183┊   ┊            receivedAt: null,
-┊184┊   ┊            readAt: null,
-┊185┊   ┊          },
+┊   ┊127┊          new Recipient({
+┊   ┊128┊            user: user4,
+┊   ┊129┊          }),
 ┊186┊130┊        ],
-┊187┊   ┊        holderIds: [1, 4],
-┊188┊   ┊      },
+┊   ┊131┊      }),
 ┊189┊132┊    ],
-┊190┊   ┊  },
-┊191┊   ┊  {
-┊192┊   ┊    id: 3,
+┊   ┊133┊  }));
+┊   ┊134┊
+┊   ┊135┊  await connection.manager.save(new Chat({
 ┊193┊136┊    createdAt: moment().subtract(8, 'weeks').toDate(),
-┊194┊   ┊    name: null,
-┊195┊   ┊    picture: null,
-┊196┊   ┊    allTimeMemberIds: [1, 5],
-┊197┊   ┊    listingMemberIds: [1, 5],
-┊198┊   ┊    adminIds: null,
-┊199┊   ┊    ownerId: null,
+┊   ┊137┊    allTimeMembers: [user1, user5],
+┊   ┊138┊    listingMembers: [user1, user5],
 ┊200┊139┊    messages: [
-┊201┊   ┊      {
-┊202┊   ┊        id: 1,
-┊203┊   ┊        chatId: 3,
-┊204┊   ┊        senderId: 1,
+┊   ┊140┊      new Message({
+┊   ┊141┊        sender: user1,
 ┊205┊142┊        content: 'I should buy a boat',
 ┊206┊143┊        createdAt: moment().subtract(1, 'days').toDate(),
 ┊207┊144┊        type: MessageType.TEXT,
+┊   ┊145┊        holders: [user1, user5],
 ┊208┊146┊        recipients: [
-┊209┊   ┊          {
-┊210┊   ┊            userId: 5,
-┊211┊   ┊            messageId: 1,
-┊212┊   ┊            chatId: 3,
-┊213┊   ┊            receivedAt: null,
-┊214┊   ┊            readAt: null,
-┊215┊   ┊          },
+┊   ┊147┊          new Recipient({
+┊   ┊148┊            user: user5,
+┊   ┊149┊          }),
 ┊216┊150┊        ],
-┊217┊   ┊        holderIds: [1, 5],
-┊218┊   ┊      },
-┊219┊   ┊      {
-┊220┊   ┊        id: 2,
-┊221┊   ┊        chatId: 3,
-┊222┊   ┊        senderId: 1,
+┊   ┊151┊      }),
+┊   ┊152┊      new Message({
+┊   ┊153┊        sender: user1,
 ┊223┊154┊        content: 'You still there?',
 ┊224┊155┊        createdAt: moment().subtract(1, 'days').add(16, 'hours').toDate(),
 ┊225┊156┊        type: MessageType.TEXT,
+┊   ┊157┊        holders: [user1, user5],
 ┊226┊158┊        recipients: [
-┊227┊   ┊          {
-┊228┊   ┊            userId: 5,
-┊229┊   ┊            messageId: 2,
-┊230┊   ┊            chatId: 3,
-┊231┊   ┊            receivedAt: null,
-┊232┊   ┊            readAt: null,
-┊233┊   ┊          },
+┊   ┊159┊          new Recipient({
+┊   ┊160┊            user: user5,
+┊   ┊161┊          }),
 ┊234┊162┊        ],
-┊235┊   ┊        holderIds: [1, 5],
-┊236┊   ┊      },
+┊   ┊163┊      }),
 ┊237┊164┊    ],
-┊238┊   ┊  },
-┊239┊   ┊  {
-┊240┊   ┊    id: 4,
+┊   ┊165┊  }));
+┊   ┊166┊
+┊   ┊167┊  await connection.manager.save(new Chat({
 ┊241┊168┊    createdAt: moment().subtract(7, 'weeks').toDate(),
-┊242┊   ┊    name: null,
-┊243┊   ┊    picture: null,
-┊244┊   ┊    allTimeMemberIds: [3, 4],
-┊245┊   ┊    listingMemberIds: [3, 4],
-┊246┊   ┊    adminIds: null,
-┊247┊   ┊    ownerId: null,
+┊   ┊169┊    allTimeMembers: [user3, user4],
+┊   ┊170┊    listingMembers: [user3, user4],
 ┊248┊171┊    messages: [
-┊249┊   ┊      {
-┊250┊   ┊        id: 1,
-┊251┊   ┊        chatId: 4,
-┊252┊   ┊        senderId: 3,
+┊   ┊172┊      new Message({
+┊   ┊173┊        sender: user3,
 ┊253┊174┊        content: 'Look at my mukluks!',
 ┊254┊175┊        createdAt: moment().subtract(4, 'days').toDate(),
 ┊255┊176┊        type: MessageType.TEXT,
+┊   ┊177┊        holders: [user3, user4],
 ┊256┊178┊        recipients: [
-┊257┊   ┊          {
-┊258┊   ┊            userId: 4,
-┊259┊   ┊            messageId: 1,
-┊260┊   ┊            chatId: 4,
-┊261┊   ┊            receivedAt: null,
-┊262┊   ┊            readAt: null,
-┊263┊   ┊          },
+┊   ┊179┊          new Recipient({
+┊   ┊180┊            user: user4,
+┊   ┊181┊          }),
 ┊264┊182┊        ],
-┊265┊   ┊        holderIds: [3, 4],
-┊266┊   ┊      },
+┊   ┊183┊      }),
 ┊267┊184┊    ],
-┊268┊   ┊  },
-┊269┊   ┊  {
-┊270┊   ┊    id: 5,
+┊   ┊185┊  }));
+┊   ┊186┊
+┊   ┊187┊  await connection.manager.save(new Chat({
 ┊271┊188┊    createdAt: moment().subtract(6, 'weeks').toDate(),
-┊272┊   ┊    name: null,
-┊273┊   ┊    picture: null,
-┊274┊   ┊    allTimeMemberIds: [2, 5],
-┊275┊   ┊    listingMemberIds: [2, 5],
-┊276┊   ┊    adminIds: null,
-┊277┊   ┊    ownerId: null,
+┊   ┊189┊    allTimeMembers: [user2, user5],
+┊   ┊190┊    listingMembers: [user2, user5],
 ┊278┊191┊    messages: [
-┊279┊   ┊      {
-┊280┊   ┊        id: 1,
-┊281┊   ┊        chatId: 5,
-┊282┊   ┊        senderId: 2,
+┊   ┊192┊      new Message({
+┊   ┊193┊        sender: user2,
 ┊283┊194┊        content: 'This is wicked good ice cream.',
 ┊284┊195┊        createdAt: moment().subtract(2, 'weeks').toDate(),
 ┊285┊196┊        type: MessageType.TEXT,
+┊   ┊197┊        holders: [user2, user5],
 ┊286┊198┊        recipients: [
-┊287┊   ┊          {
-┊288┊   ┊            userId: 5,
-┊289┊   ┊            messageId: 1,
-┊290┊   ┊            chatId: 5,
-┊291┊   ┊            receivedAt: null,
-┊292┊   ┊            readAt: null,
-┊293┊   ┊          },
+┊   ┊199┊          new Recipient({
+┊   ┊200┊            user: user5,
+┊   ┊201┊          }),
 ┊294┊202┊        ],
-┊295┊   ┊        holderIds: [2, 5],
-┊296┊   ┊      },
-┊297┊   ┊      {
-┊298┊   ┊        id: 2,
-┊299┊   ┊        chatId: 6,
-┊300┊   ┊        senderId: 5,
+┊   ┊203┊      }),
+┊   ┊204┊      new Message({
+┊   ┊205┊        sender: user5,
 ┊301┊206┊        content: 'Love it!',
 ┊302┊207┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
 ┊303┊208┊        type: MessageType.TEXT,
+┊   ┊209┊        holders: [user2, user5],
 ┊304┊210┊        recipients: [
-┊305┊   ┊          {
-┊306┊   ┊            userId: 2,
-┊307┊   ┊            messageId: 2,
-┊308┊   ┊            chatId: 5,
-┊309┊   ┊            receivedAt: null,
-┊310┊   ┊            readAt: null,
-┊311┊   ┊          },
+┊   ┊211┊          new Recipient({
+┊   ┊212┊            user: user2,
+┊   ┊213┊          }),
 ┊312┊214┊        ],
-┊313┊   ┊        holderIds: [5, 2],
-┊314┊   ┊      },
+┊   ┊215┊      }),
 ┊315┊216┊    ],
-┊316┊   ┊  },
-┊317┊   ┊  {
-┊318┊   ┊    id: 6,
+┊   ┊217┊  }));
+┊   ┊218┊
+┊   ┊219┊  await connection.manager.save(new Chat({
 ┊319┊220┊    createdAt: moment().subtract(5, 'weeks').toDate(),
-┊320┊   ┊    name: null,
-┊321┊   ┊    picture: null,
-┊322┊   ┊    allTimeMemberIds: [1, 6],
-┊323┊   ┊    listingMemberIds: [1],
-┊324┊   ┊    adminIds: null,
-┊325┊   ┊    ownerId: null,
-┊326┊   ┊    messages: [],
-┊327┊   ┊  },
-┊328┊   ┊  {
-┊329┊   ┊    id: 7,
+┊   ┊221┊    allTimeMembers: [user1, user6],
+┊   ┊222┊    listingMembers: [user1],
+┊   ┊223┊  }));
+┊   ┊224┊
+┊   ┊225┊  await connection.manager.save(new Chat({
 ┊330┊226┊    createdAt: moment().subtract(4, 'weeks').toDate(),
-┊331┊   ┊    name: null,
-┊332┊   ┊    picture: null,
-┊333┊   ┊    allTimeMemberIds: [2, 1],
-┊334┊   ┊    listingMemberIds: [2],
-┊335┊   ┊    adminIds: null,
-┊336┊   ┊    ownerId: null,
-┊337┊   ┊    messages: [],
-┊338┊   ┊  },
-┊339┊   ┊  {
-┊340┊   ┊    id: 8,
+┊   ┊227┊    allTimeMembers: [user2, user1],
+┊   ┊228┊    listingMembers: [user2],
+┊   ┊229┊  }));
+┊   ┊230┊
+┊   ┊231┊  await connection.manager.save(new Chat({
 ┊341┊232┊    createdAt: moment().subtract(3, 'weeks').toDate(),
-┊342┊   ┊    name: 'A user 0 group',
+┊   ┊233┊    name: 'Ethan\'s group',
 ┊343┊234┊    picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
-┊344┊   ┊    allTimeMemberIds: [1, 3, 4, 6],
-┊345┊   ┊    listingMemberIds: [1, 3, 4, 6],
-┊346┊   ┊    actualGroupMemberIds: [1, 4, 6],
-┊347┊   ┊    adminIds: [1, 6],
-┊348┊   ┊    ownerId: 1,
+┊   ┊235┊    allTimeMembers: [user1, user3, user4, user6],
+┊   ┊236┊    listingMembers: [user1, user3, user4, user6],
+┊   ┊237┊    actualGroupMembers: [user1, user4, user6],
+┊   ┊238┊    admins: [user1, user6],
+┊   ┊239┊    owner: user1,
 ┊349┊240┊    messages: [
-┊350┊   ┊      {
-┊351┊   ┊        id: 1,
-┊352┊   ┊        chatId: 8,
-┊353┊   ┊        senderId: 1,
+┊   ┊241┊      new Message({
+┊   ┊242┊        sender: user1,
 ┊354┊243┊        content: 'I made a group',
 ┊355┊244┊        createdAt: moment().subtract(2, 'weeks').toDate(),
 ┊356┊245┊        type: MessageType.TEXT,
+┊   ┊246┊        holders: [user1, user3, user4, user6],
 ┊357┊247┊        recipients: [
-┊358┊   ┊          {
-┊359┊   ┊            userId: 3,
-┊360┊   ┊            messageId: 1,
-┊361┊   ┊            chatId: 8,
-┊362┊   ┊            receivedAt: null,
-┊363┊   ┊            readAt: null,
-┊364┊   ┊          },
-┊365┊   ┊          {
-┊366┊   ┊            userId: 4,
-┊367┊   ┊            messageId: 1,
-┊368┊   ┊            chatId: 8,
-┊369┊   ┊            receivedAt: moment().subtract(2, 'weeks').add(1, 'minutes').toDate(),
-┊370┊   ┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').toDate(),
-┊371┊   ┊          },
-┊372┊   ┊          {
-┊373┊   ┊            userId: 6,
-┊374┊   ┊            messageId: 1,
-┊375┊   ┊            chatId: 8,
-┊376┊   ┊            receivedAt: null,
-┊377┊   ┊            readAt: null,
-┊378┊   ┊          },
+┊   ┊248┊          new Recipient({
+┊   ┊249┊            user: user3,
+┊   ┊250┊          }),
+┊   ┊251┊          new Recipient({
+┊   ┊252┊            user: user4,
+┊   ┊253┊          }),
+┊   ┊254┊          new Recipient({
+┊   ┊255┊            user: user6,
+┊   ┊256┊          }),
 ┊379┊257┊        ],
-┊380┊   ┊        holderIds: [1, 3, 4, 6],
-┊381┊   ┊      },
-┊382┊   ┊      {
-┊383┊   ┊        id: 2,
-┊384┊   ┊        chatId: 8,
-┊385┊   ┊        senderId: 1,
-┊386┊   ┊        content: 'Ops, user 3 was not supposed to be here',
+┊   ┊258┊      }),
+┊   ┊259┊      new Message({
+┊   ┊260┊        sender: user1,
+┊   ┊261┊        content: 'Ops, Avery was not supposed to be here',
 ┊387┊262┊        createdAt: moment().subtract(2, 'weeks').add(2, 'minutes').toDate(),
 ┊388┊263┊        type: MessageType.TEXT,
+┊   ┊264┊        holders: [user1, user4, user6],
 ┊389┊265┊        recipients: [
-┊390┊   ┊          {
-┊391┊   ┊            userId: 4,
-┊392┊   ┊            messageId: 2,
-┊393┊   ┊            chatId: 8,
-┊394┊   ┊            receivedAt: moment().subtract(2, 'weeks').add(3, 'minutes').toDate(),
-┊395┊   ┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').toDate(),
-┊396┊   ┊          },
-┊397┊   ┊          {
-┊398┊   ┊            userId: 6,
-┊399┊   ┊            messageId: 2,
-┊400┊   ┊            chatId: 8,
-┊401┊   ┊            receivedAt: null,
-┊402┊   ┊            readAt: null,
-┊403┊   ┊          },
+┊   ┊266┊          new Recipient({
+┊   ┊267┊            user: user4,
+┊   ┊268┊          }),
+┊   ┊269┊          new Recipient({
+┊   ┊270┊            user: user6,
+┊   ┊271┊          }),
 ┊404┊272┊        ],
-┊405┊   ┊        holderIds: [1, 4, 6],
-┊406┊   ┊      },
-┊407┊   ┊      {
-┊408┊   ┊        id: 3,
-┊409┊   ┊        chatId: 8,
-┊410┊   ┊        senderId: 4,
+┊   ┊273┊      }),
+┊   ┊274┊      new Message({
+┊   ┊275┊        sender: user4,
 ┊411┊276┊        content: 'Awesome!',
 ┊412┊277┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
 ┊413┊278┊        type: MessageType.TEXT,
+┊   ┊279┊        holders: [user1, user4, user6],
 ┊414┊280┊        recipients: [
-┊415┊   ┊          {
-┊416┊   ┊            userId: 1,
-┊417┊   ┊            messageId: 3,
-┊418┊   ┊            chatId: 8,
-┊419┊   ┊            receivedAt: null,
-┊420┊   ┊            readAt: null,
-┊421┊   ┊          },
-┊422┊   ┊          {
-┊423┊   ┊            userId: 6,
-┊424┊   ┊            messageId: 3,
-┊425┊   ┊            chatId: 8,
-┊426┊   ┊            receivedAt: null,
-┊427┊   ┊            readAt: null,
-┊428┊   ┊          },
+┊   ┊281┊          new Recipient({
+┊   ┊282┊            user: user1,
+┊   ┊283┊          }),
+┊   ┊284┊          new Recipient({
+┊   ┊285┊            user: user6,
+┊   ┊286┊          }),
 ┊429┊287┊        ],
-┊430┊   ┊        holderIds: [1, 4, 6],
-┊431┊   ┊      },
+┊   ┊288┊      }),
 ┊432┊289┊    ],
-┊433┊   ┊  },
-┊434┊   ┊  {
-┊435┊   ┊    id: 9,
-┊436┊   ┊    createdAt: moment().subtract(2, 'weeks').toDate(),
-┊437┊   ┊    name: 'A user 5 group',
-┊438┊   ┊    picture: null,
-┊439┊   ┊    allTimeMemberIds: [6, 3],
-┊440┊   ┊    listingMemberIds: [6, 3],
-┊441┊   ┊    actualGroupMemberIds: [6, 3],
-┊442┊   ┊    adminIds: [6],
-┊443┊   ┊    ownerId: 6,
-┊444┊   ┊    messages: [],
-┊445┊   ┊  },
-┊446┊   ┊];
+┊   ┊290┊  }));
 ┊447┊291┊
-┊448┊   ┊export const db = {users, chats};
+┊   ┊292┊  await connection.manager.save(new Chat({
+┊   ┊293┊    createdAt: moment().subtract(2, 'weeks').toDate(),
+┊   ┊294┊    name: 'Ray\'s group',
+┊   ┊295┊    allTimeMembers: [user3, user6],
+┊   ┊296┊    listingMembers: [user3, user6],
+┊   ┊297┊    actualGroupMembers: [user3, user6],
+┊   ┊298┊    admins: [user6],
+┊   ┊299┊    owner: user6,
+┊   ┊300┊  }));
+┊   ┊301┊}
```

[}]: #

It's time to deal with resolvers:

[{]: <helper> (diffStep "7.1" files="schema/resolvers.ts" module="server")

#### [Step 7.1: TypeORM with PostgreSQL](https://github.com/Urigo/WhatsApp-Clone-Server/commit/8c4af9f)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,11 +1,11 @@
 ┊ 1┊ 1┊import { PubSub, withFilter } from 'apollo-server-express';
 ┊ 2┊ 2┊import { GraphQLDateTime } from 'graphql-iso-date';
-┊ 3┊  ┊import { ChatDb, db, MessageDb, MessageType, RecipientDb, UserDb } from "../db";
+┊  ┊ 3┊import { MessageType } from "../db";
 ┊ 4┊ 4┊import { IResolvers, MessageAddedSubscriptionArgs } from "../types";
-┊ 5┊  ┊import moment from "moment";
-┊ 6┊  ┊
-┊ 7┊  ┊let users = db.users;
-┊ 8┊  ┊let chats = db.chats;
+┊  ┊ 5┊import { User } from "../entity/User";
+┊  ┊ 6┊import { Chat } from "../entity/Chat";
+┊  ┊ 7┊import { Message } from "../entity/Message";
+┊  ┊ 8┊import { Recipient } from "../entity/Recipient";
 ┊ 9┊ 9┊
 ┊10┊10┊export const pubsub = new PubSub();
 ┊11┊11┊
```
```diff
@@ -13,21 +13,81 @@
 ┊13┊13┊  Date: GraphQLDateTime,
 ┊14┊14┊  Query: {
 ┊15┊15┊    me: (obj, args, {currentUser}) => currentUser,
-┊16┊  ┊    users: (obj, args, {currentUser}) => users.filter(user => user.id !== currentUser.id),
-┊17┊  ┊    chats: (obj, args, {currentUser}) => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
-┊18┊  ┊    chat: (obj, {chatId}) => chats.find(chat => chat.id === Number(chatId)) || null,
+┊  ┊16┊    users: async (obj, args, {currentUser, connection}) => {
+┊  ┊17┊      return await connection
+┊  ┊18┊        .createQueryBuilder(User, "user")
+┊  ┊19┊        .where('user.id != :id', {id: currentUser.id})
+┊  ┊20┊        .getMany();
+┊  ┊21┊    },
+┊  ┊22┊    chats: async (obj, args, {currentUser, connection}) => {
+┊  ┊23┊      // TODO: make a proper query instead of this mess (see https://github.com/typeorm/typeorm/issues/2132)
+┊  ┊24┊      const chats = await connection
+┊  ┊25┊        .createQueryBuilder(Chat, "chat")
+┊  ┊26┊        .leftJoin('chat.listingMembers', 'listingMembers')
+┊  ┊27┊        .where('listingMembers.id = :id', {id: currentUser.id})
+┊  ┊28┊        .getMany();
+┊  ┊29┊
+┊  ┊30┊      for (let chat of chats) {
+┊  ┊31┊        chat.messages = (await connection
+┊  ┊32┊          .createQueryBuilder(Message, "message")
+┊  ┊33┊          .innerJoin('message.chat', 'chat', 'chat.id = :chatId', { chatId: chat.id })
+┊  ┊34┊          .innerJoin('message.holders', 'holders', 'holders.id = :userId', {
+┊  ┊35┊            userId: currentUser.id,
+┊  ┊36┊          })
+┊  ┊37┊          .orderBy({ 'message.createdAt': { order: 'DESC', nulls: 'NULLS LAST' } })
+┊  ┊38┊          .getMany())
+┊  ┊39┊          .reverse();
+┊  ┊40┊      }
+┊  ┊41┊
+┊  ┊42┊      return chats.sort((chatA, chatB) => {
+┊  ┊43┊        const dateA = chatA.messages.length ? chatA.messages[chatA.messages.length - 1].createdAt : chatA.createdAt;
+┊  ┊44┊        const dateB = chatB.messages.length ? chatB.messages[chatB.messages.length - 1].createdAt : chatB.createdAt;
+┊  ┊45┊        return dateB.valueOf() - dateA.valueOf();
+┊  ┊46┊      });
+┊  ┊47┊    },
+┊  ┊48┊    chat: async (obj, {chatId}, {connection}) => {
+┊  ┊49┊      return await connection
+┊  ┊50┊        .createQueryBuilder(Chat, "chat")
+┊  ┊51┊        .whereInIds(chatId)
+┊  ┊52┊        .getOne();
+┊  ┊53┊    },
 ┊19┊54┊  },
 ┊20┊55┊  Mutation: {
-┊21┊  ┊    updateUser: (obj, {name, picture}, {currentUser}) => {
+┊  ┊56┊    updateUser: async (obj, {name, picture}, {currentUser, connection}) => {
+┊  ┊57┊      if (name === currentUser.name && picture === currentUser.picture) {
+┊  ┊58┊        return currentUser;
+┊  ┊59┊      }
+┊  ┊60┊
 ┊22┊61┊      currentUser.name = name || currentUser.name;
 ┊23┊62┊      currentUser.picture = picture || currentUser.picture;
 ┊24┊63┊
+┊  ┊64┊      await connection.getRepository(User).save(currentUser);
+┊  ┊65┊
 ┊25┊66┊      pubsub.publish('userUpdated', {
 ┊26┊67┊        userUpdated: currentUser,
 ┊27┊68┊      });
 ┊28┊69┊
-┊29┊  ┊      // Get a list of the chats who have/had currentUser involved
-┊30┊  ┊      const chatsAffected = chats.filter(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser.id));
+┊  ┊70┊      const data = await connection
+┊  ┊71┊        .createQueryBuilder(User, 'user')
+┊  ┊72┊        .where('user.id = :id', { id: currentUser.id })
+┊  ┊73┊        // Get a list of the chats who have/had currentUser involved
+┊  ┊74┊        .innerJoinAndSelect(
+┊  ┊75┊          'user.allTimeMemberChats',
+┊  ┊76┊          'allTimeMemberChats',
+┊  ┊77┊          // Groups are unaffected
+┊  ┊78┊          'allTimeMemberChats.name IS NULL',
+┊  ┊79┊        )
+┊  ┊80┊        // We need to notify only those who get the chat listed (except currentUser of course)
+┊  ┊81┊        .innerJoin(
+┊  ┊82┊          'allTimeMemberChats.listingMembers',
+┊  ┊83┊          'listingMembers',
+┊  ┊84┊          'listingMembers.id != :currentUserId',
+┊  ┊85┊          {
+┊  ┊86┊            currentUserId: currentUser.id,
+┊  ┊87┊          })
+┊  ┊88┊        .getOne();
+┊  ┊89┊
+┊  ┊90┊      const chatsAffected = data && data.allTimeMemberChats || [];
 ┊31┊91┊
 ┊32┊92┊      chatsAffected.forEach(chat => {
 ┊33┊93┊        pubsub.publish('chatUpdated', {
```
```diff
@@ -38,74 +98,87 @@
 ┊ 38┊ 98┊
 ┊ 39┊ 99┊      return currentUser;
 ┊ 40┊100┊    },
-┊ 41┊   ┊    addChat: (obj, {userId}, {currentUser}) => {
-┊ 42┊   ┊      if (!users.find(user => user.id === Number(userId))) {
+┊   ┊101┊    addChat: async (obj, {userId}, {currentUser, connection}) => {
+┊   ┊102┊      const user = await connection
+┊   ┊103┊        .createQueryBuilder(User, "user")
+┊   ┊104┊        .whereInIds(userId)
+┊   ┊105┊        .getOne();
+┊   ┊106┊
+┊   ┊107┊        if (!user) {
 ┊ 43┊108┊        throw new Error(`User ${userId} doesn't exist.`);
 ┊ 44┊109┊      }
 ┊ 45┊110┊
-┊ 46┊   ┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser.id) && chat.allTimeMemberIds.includes(Number(userId)));
+┊   ┊111┊      let chat = await connection
+┊   ┊112┊        .createQueryBuilder(Chat, "chat")
+┊   ┊113┊        .where('chat.name IS NULL')
+┊   ┊114┊        .innerJoin('chat.allTimeMembers', 'allTimeMembers1', 'allTimeMembers1.id = :currentUserId', {currentUserId: currentUser.id})
+┊   ┊115┊        .innerJoin('chat.allTimeMembers', 'allTimeMembers2', 'allTimeMembers2.id = :userId', {userId})
+┊   ┊116┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊117┊        .getOne();
+┊   ┊118┊
 ┊ 47┊119┊      if (chat) {
-┊ 48┊   ┊        // Chat already exists. Both users are already in the allTimeMemberIds array
-┊ 49┊   ┊        const chatId = chat.id;
-┊ 50┊   ┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
+┊   ┊120┊        // Chat already exists. Both users are already in the userIds array
+┊   ┊121┊        const listingMembers = await connection
+┊   ┊122┊          .createQueryBuilder(User, "user")
+┊   ┊123┊          .innerJoin('user.listingMemberChats', 'listingMemberChats', 'listingMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊124┊          .getMany();
+┊   ┊125┊
+┊   ┊126┊        if (!listingMembers.find(user => user.id === currentUser.id)) {
 ┊ 51┊127┊          // The chat isn't listed for the current user. Add him to the memberIds
-┊ 52┊   ┊          chat.listingMemberIds.push(currentUser.id);
-┊ 53┊   ┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser.id);
-┊ 54┊   ┊          return chat;
+┊   ┊128┊          chat.listingMembers.push(currentUser);
+┊   ┊129┊          chat = await connection.getRepository(Chat).save(chat);
+┊   ┊130┊
+┊   ┊131┊          return chat || null;
 ┊ 55┊132┊        } else {
 ┊ 56┊133┊          throw new Error(`Chat already exists.`);
 ┊ 57┊134┊        }
 ┊ 58┊135┊      } else {
 ┊ 59┊136┊        // Create the chat
-┊ 60┊   ┊        const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
-┊ 61┊   ┊        const chat: ChatDb = {
-┊ 62┊   ┊          id,
-┊ 63┊   ┊          createdAt: moment().toDate(),
-┊ 64┊   ┊          name: null,
-┊ 65┊   ┊          picture: null,
-┊ 66┊   ┊          adminIds: null,
-┊ 67┊   ┊          ownerId: null,
-┊ 68┊   ┊          allTimeMemberIds: [currentUser.id, Number(userId)],
+┊   ┊137┊        chat = await connection.getRepository(Chat).save(new Chat({
+┊   ┊138┊          allTimeMembers: [currentUser, user],
 ┊ 69┊139┊          // Chat will not be listed to the other user until the first message gets written
-┊ 70┊   ┊          listingMemberIds: [currentUser.id],
-┊ 71┊   ┊          actualGroupMemberIds: null,
-┊ 72┊   ┊          messages: [],
-┊ 73┊   ┊        };
-┊ 74┊   ┊        chats.push(chat);
-┊ 75┊   ┊        return chat;
+┊   ┊140┊          listingMembers: [currentUser],
+┊   ┊141┊        }));
+┊   ┊142┊        return chat || null;
 ┊ 76┊143┊      }
 ┊ 77┊144┊    },
-┊ 78┊   ┊    addGroup: (obj, {userIds, groupName, groupPicture}, {currentUser}) => {
-┊ 79┊   ┊      userIds.forEach(userId => {
-┊ 80┊   ┊        if (!users.find(user => user.id === Number(userId))) {
+┊   ┊145┊    addGroup: async (obj, {userIds, groupName, groupPicture}, {currentUser, connection}) => {
+┊   ┊146┊      let users: User[] = [];
+┊   ┊147┊      for (let userId of userIds) {
+┊   ┊148┊        const user = await connection
+┊   ┊149┊          .createQueryBuilder(User, "user")
+┊   ┊150┊          .whereInIds(userId)
+┊   ┊151┊          .getOne();
+┊   ┊152┊
+┊   ┊153┊        if (!user) {
 ┊ 81┊154┊          throw new Error(`User ${userId} doesn't exist.`);
 ┊ 82┊155┊        }
-┊ 83┊   ┊      });
 ┊ 84┊156┊
-┊ 85┊   ┊      const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
-┊ 86┊   ┊      const chat: ChatDb = {
-┊ 87┊   ┊        id,
-┊ 88┊   ┊        createdAt: moment().toDate(),
+┊   ┊157┊        users.push(user);
+┊   ┊158┊      }
+┊   ┊159┊
+┊   ┊160┊      const chat = await connection.getRepository(Chat).save(new Chat({
 ┊ 89┊161┊        name: groupName,
-┊ 90┊   ┊        picture: groupPicture || null,
-┊ 91┊   ┊        adminIds: [currentUser.id],
-┊ 92┊   ┊        ownerId: currentUser.id,
-┊ 93┊   ┊        allTimeMemberIds: [currentUser.id, ...userIds.map(id => Number(id))],
-┊ 94┊   ┊        listingMemberIds: [currentUser.id, ...userIds.map(id => Number(id))],
-┊ 95┊   ┊        actualGroupMemberIds: [currentUser.id, ...userIds.map(id => Number(id))],
-┊ 96┊   ┊        messages: [],
-┊ 97┊   ┊      };
-┊ 98┊   ┊      chats.push(chat);
+┊   ┊162┊        picture: groupPicture || undefined,
+┊   ┊163┊        admins: [currentUser],
+┊   ┊164┊        owner: currentUser,
+┊   ┊165┊        allTimeMembers: [...users, currentUser],
+┊   ┊166┊        listingMembers: [...users, currentUser],
+┊   ┊167┊        actualGroupMembers: [...users, currentUser],
+┊   ┊168┊      }));
 ┊ 99┊169┊
 ┊100┊170┊      pubsub.publish('chatAdded', {
 ┊101┊171┊        creatorId: currentUser.id,
 ┊102┊172┊        chatAdded: chat,
 ┊103┊173┊      });
 ┊104┊174┊
-┊105┊   ┊      return chat;
+┊   ┊175┊      return chat || null;
 ┊106┊176┊    },
-┊107┊   ┊    updateGroup: (obj, {chatId, groupName, groupPicture}, {currentUser}) => {
-┊108┊   ┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊177┊    updateGroup: async (obj, {chatId, groupName, groupPicture}, {currentUser, connection}) => {
+┊   ┊178┊      const chat = await connection
+┊   ┊179┊        .createQueryBuilder(Chat, 'chat')
+┊   ┊180┊        .whereInIds(chatId)
+┊   ┊181┊        .getOne();
 ┊109┊182┊
 ┊110┊183┊      if (!chat) {
 ┊111┊184┊        throw new Error(`The chat ${chatId} doesn't exist.`);
```
```diff
@@ -118,6 +191,9 @@
 ┊118┊191┊      chat.name = groupName || chat.name;
 ┊119┊192┊      chat.picture = groupPicture || chat.picture;
 ┊120┊193┊
+┊   ┊194┊      // Update the chat
+┊   ┊195┊      await connection.getRepository(Chat).save(chat);
+┊   ┊196┊
 ┊121┊197┊      pubsub.publish('chatUpdated', {
 ┊122┊198┊        updaterId: currentUser.id,
 ┊123┊199┊        chatUpdated: chat,
```
```diff
@@ -125,8 +201,17 @@
 ┊125┊201┊
 ┊126┊202┊      return chat;
 ┊127┊203┊    },
-┊128┊   ┊    removeChat: (obj, {chatId}, {currentUser}) => {
-┊129┊   ┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊204┊    removeChat: async (obj, {chatId}, {currentUser, connection}) => {
+┊   ┊205┊      const chat = await connection
+┊   ┊206┊        .createQueryBuilder(Chat, "chat")
+┊   ┊207┊        .whereInIds(Number(chatId))
+┊   ┊208┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊209┊        .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊210┊        .leftJoinAndSelect('chat.admins', 'admins')
+┊   ┊211┊        .leftJoinAndSelect('chat.owner', 'owner')
+┊   ┊212┊        .leftJoinAndSelect('chat.messages', 'messages')
+┊   ┊213┊        .leftJoinAndSelect('messages.holders', 'holders')
+┊   ┊214┊        .getOne();
 ┊130┊215┊
 ┊131┊216┊      if (!chat) {
 ┊132┊217┊        throw new Error(`The chat ${chatId} doesn't exist.`);
```
```diff
@@ -134,184 +219,189 @@
 ┊134┊219┊
 ┊135┊220┊      if (!chat.name) {
 ┊136┊221┊        // Chat
-┊137┊   ┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
-┊138┊   ┊          throw new Error(`The user is not a member of the chat ${chatId}.`);
+┊   ┊222┊        if (!chat.listingMembers.find(user => user.id === currentUser.id)) {
+┊   ┊223┊          throw new Error(`The user is not a listing member of the chat ${chatId}.`);
 ┊139┊224┊        }
 ┊140┊225┊
 ┊141┊226┊        // Instead of chaining map and filter we can loop once using reduce
-┊142┊   ┊        const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
-┊143┊   ┊          // Remove the current user from the message holders
-┊144┊   ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊227┊        chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊228┊          const filtered = await filtered$;
+┊   ┊229┊
+┊   ┊230┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
 ┊145┊231┊
-┊146┊   ┊          if (message.holderIds.length !== 0) {
+┊   ┊232┊          if (message.holders.length !== 0) {
+┊   ┊233┊            // Remove the current user from the message holders
+┊   ┊234┊            await connection.getRepository(Message).save(message);
 ┊147┊235┊            filtered.push(message);
-┊148┊   ┊          } // else discard the message
+┊   ┊236┊          } else {
+┊   ┊237┊            // Simply remove the message
+┊   ┊238┊            const recipients = await connection
+┊   ┊239┊              .createQueryBuilder(Recipient, "recipient")
+┊   ┊240┊              .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊241┊              .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊242┊              .getMany();
+┊   ┊243┊            for (let recipient of recipients) {
+┊   ┊244┊              await connection.getRepository(Recipient).remove(recipient);
+┊   ┊245┊            }
+┊   ┊246┊            await connection.getRepository(Message).remove(message);
+┊   ┊247┊          }
 ┊149┊248┊
 ┊150┊249┊          return filtered;
-┊151┊   ┊        }, []);
+┊   ┊250┊        }, Promise.resolve([]));
 ┊152┊251┊
 ┊153┊252┊        // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
-┊154┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊253┊        chat.listingMembers = chat.listingMembers.filter(user => user.id !== currentUser.id);
 ┊155┊254┊
 ┊156┊255┊        // Check how many members are left
-┊157┊   ┊        if (listingMemberIds.length === 0) {
+┊   ┊256┊        if (chat.listingMembers.length === 0) {
 ┊158┊257┊          // Delete the chat
-┊159┊   ┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊258┊          await connection.getRepository(Chat).remove(chat);
 ┊160┊259┊        } else {
 ┊161┊260┊          // Update the chat
-┊162┊   ┊          chats = chats.map(chat => {
-┊163┊   ┊            if (chat.id === Number(chatId)) {
-┊164┊   ┊              chat = {...chat, listingMemberIds, messages};
-┊165┊   ┊            }
-┊166┊   ┊            return chat;
-┊167┊   ┊          });
+┊   ┊261┊          await connection.getRepository(Chat).save(chat);
 ┊168┊262┊        }
 ┊169┊263┊        return chatId;
 ┊170┊264┊      } else {
 ┊171┊265┊        // Group
-┊172┊   ┊        if (chat.ownerId !== currentUser.id) {
-┊173┊   ┊          throw new Error(`Group ${chatId} is not owned by the user.`);
-┊174┊   ┊        }
 ┊175┊266┊
 ┊176┊267┊        // Instead of chaining map and filter we can loop once using reduce
-┊177┊   ┊        const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
-┊178┊   ┊          // Remove the current user from the message holders
-┊179┊   ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊268┊        chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊269┊          const filtered = await filtered$;
+┊   ┊270┊
+┊   ┊271┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
 ┊180┊272┊
-┊181┊   ┊          if (message.holderIds.length !== 0) {
+┊   ┊273┊          if (message.holders.length !== 0) {
+┊   ┊274┊            // Remove the current user from the message holders
+┊   ┊275┊            await connection.getRepository(Message).save(message);
 ┊182┊276┊            filtered.push(message);
-┊183┊   ┊          } // else discard the message
+┊   ┊277┊          } else {
+┊   ┊278┊            // Simply remove the message
+┊   ┊279┊            const recipients = await connection
+┊   ┊280┊              .createQueryBuilder(Recipient, "recipient")
+┊   ┊281┊              .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊282┊              .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊283┊              .getMany();
+┊   ┊284┊            for (let recipient of recipients) {
+┊   ┊285┊              await connection.getRepository(Recipient).remove(recipient);
+┊   ┊286┊            }
+┊   ┊287┊            await connection.getRepository(Message).remove(message);
+┊   ┊288┊          }
 ┊184┊289┊
 ┊185┊290┊          return filtered;
-┊186┊   ┊        }, []);
+┊   ┊291┊        }, Promise.resolve([]));
 ┊187┊292┊
 ┊188┊293┊        // Remove the current user from who gets the group listed. The group will no longer appear in his list
-┊189┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊294┊        chat.listingMembers = chat.listingMembers.filter(user => user.id !== currentUser.id);
 ┊190┊295┊
 ┊191┊296┊        // Check how many members (including previous ones who can still access old messages) are left
-┊192┊   ┊        if (listingMemberIds.length === 0) {
+┊   ┊297┊        if (chat.listingMembers.length === 0) {
 ┊193┊298┊          // Remove the group
-┊194┊   ┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊299┊          await connection.getRepository(Chat).remove(chat);
 ┊195┊300┊        } else {
 ┊196┊301┊          // Update the group
 ┊197┊302┊
 ┊198┊303┊          // Remove the current user from the chat members. He is no longer a member of the group
-┊199┊   ┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser.id);
+┊   ┊304┊          chat.actualGroupMembers = chat.actualGroupMembers && chat.actualGroupMembers.filter(user => user.id !== currentUser.id);
 ┊200┊305┊          // Remove the current user from the chat admins
-┊201┊   ┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser.id);
-┊202┊   ┊          // Set the owner id to be null. A null owner means the group is read-only
-┊203┊   ┊          let ownerId: number | null = null;
-┊204┊   ┊
-┊205┊   ┊          // Check if there is any admin left
-┊206┊   ┊          if (adminIds!.length) {
-┊207┊   ┊            // Pick an admin as the new owner. The group is no longer read-only
-┊208┊   ┊            ownerId = chat.adminIds![0];
-┊209┊   ┊          }
+┊   ┊306┊          chat.admins = chat.admins && chat.admins.filter(user => user.id !== currentUser.id);
+┊   ┊307┊          // If there are no more admins left the group goes read only
+┊   ┊308┊          chat.owner = chat.admins && chat.admins[0] || null; // A null owner means the group is read-only
 ┊210┊309┊
-┊211┊   ┊          chats = chats.map(chat => {
-┊212┊   ┊            if (chat.id === Number(chatId)) {
-┊213┊   ┊              chat = {...chat, messages, listingMemberIds, actualGroupMemberIds, adminIds, ownerId};
-┊214┊   ┊            }
-┊215┊   ┊            return chat;
-┊216┊   ┊          });
+┊   ┊310┊          await connection.getRepository(Chat).save(chat);
 ┊217┊311┊        }
 ┊218┊312┊        return chatId;
 ┊219┊313┊      }
 ┊220┊314┊    },
-┊221┊   ┊    addMessage: (obj, {chatId, content}, {currentUser}) => {
+┊   ┊315┊    addMessage: async (obj, {chatId, content}, {currentUser, connection}) => {
 ┊222┊316┊      if (content === null || content === '') {
 ┊223┊317┊        throw new Error(`Cannot add empty or null messages.`);
 ┊224┊318┊      }
 ┊225┊319┊
-┊226┊   ┊      let chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊320┊      let chat = await connection
+┊   ┊321┊        .createQueryBuilder(Chat, "chat")
+┊   ┊322┊        .whereInIds(chatId)
+┊   ┊323┊        .innerJoinAndSelect('chat.allTimeMembers', 'allTimeMembers')
+┊   ┊324┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊325┊        .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊326┊        .getOne();
 ┊227┊327┊
 ┊228┊328┊      if (!chat) {
 ┊229┊329┊        throw new Error(`Cannot find chat ${chatId}.`);
 ┊230┊330┊      }
 ┊231┊331┊
-┊232┊   ┊      let holderIds = chat.listingMemberIds;
+┊   ┊332┊      let holders: User[];
 ┊233┊333┊
 ┊234┊334┊      if (!chat.name) {
 ┊235┊335┊        // Chat
-┊236┊   ┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊336┊        if (!chat.listingMembers.map(user => user.id).includes(currentUser.id)) {
 ┊237┊337┊          throw new Error(`The chat ${chatId} must be listed for the current user before adding a message.`);
 ┊238┊338┊        }
 ┊239┊339┊
 ┊240┊340┊        // Receiver's user
-┊241┊   ┊        const receiverId = chat.allTimeMemberIds.find(userId => userId !== currentUser.id);
+┊   ┊341┊        const receiver = chat.allTimeMembers.find(user => user.id !== currentUser.id);
 ┊242┊342┊
-┊243┊   ┊        if (!receiverId) {
+┊   ┊343┊        if (!receiver) {
 ┊244┊344┊          throw new Error(`Cannot find receiver's user.`);
 ┊245┊345┊        }
 ┊246┊346┊
-┊247┊   ┊        if (!chat.listingMemberIds.find(listingId => listingId === receiverId)) {
-┊248┊   ┊          // Chat is not listed for the receiver user. Add him to the listingMemberIds
-┊249┊   ┊          chat.listingMemberIds = chat.listingMemberIds.concat(receiverId);
+┊   ┊347┊        if (!chat.listingMembers.find(listingMember => listingMember.id === receiver.id)) {
+┊   ┊348┊          // Chat is not listed for the receiver user. Add him to the listingIds
+┊   ┊349┊          chat.listingMembers.push(receiver);
 ┊250┊350┊
-┊251┊   ┊          holderIds = chat.listingMemberIds;
+┊   ┊351┊          await connection.getRepository(Chat).save(chat);
 ┊252┊352┊
 ┊253┊353┊          pubsub.publish('chatAdded', {
 ┊254┊354┊            creatorId: currentUser.id,
 ┊255┊355┊            chatAdded: chat,
 ┊256┊356┊          });
 ┊257┊357┊        }
+┊   ┊358┊
+┊   ┊359┊        holders = chat.listingMembers;
 ┊258┊360┊      } else {
 ┊259┊361┊        // Group
-┊260┊   ┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser.id)) {
+┊   ┊362┊        if (!chat.actualGroupMembers || !chat.actualGroupMembers.find(actualGroupMember => actualGroupMember.id === currentUser.id)) {
 ┊261┊363┊          throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
 ┊262┊364┊        }
 ┊263┊365┊
-┊264┊   ┊        holderIds = chat.actualGroupMemberIds!;
+┊   ┊366┊        holders = chat.actualGroupMembers;
 ┊265┊367┊      }
 ┊266┊368┊
-┊267┊   ┊      const id = (chat.messages.length && chat.messages[chat.messages.length - 1].id + 1) || 1;
-┊268┊   ┊
-┊269┊   ┊      let recipients: RecipientDb[] = [];
-┊270┊   ┊
-┊271┊   ┊      holderIds.forEach(holderId => {
-┊272┊   ┊        if (holderId !== currentUser.id) {
-┊273┊   ┊          recipients.push({
-┊274┊   ┊            userId: holderId,
-┊275┊   ┊            messageId: id,
-┊276┊   ┊            chatId: Number(chatId),
-┊277┊   ┊            receivedAt: null,
-┊278┊   ┊            readAt: null,
-┊279┊   ┊          });
-┊280┊   ┊        }
-┊281┊   ┊      });
-┊282┊   ┊
-┊283┊   ┊      const message: MessageDb = {
-┊284┊   ┊        id,
-┊285┊   ┊        chatId: Number(chatId),
-┊286┊   ┊        senderId: currentUser.id,
+┊   ┊369┊      const message = await connection.getRepository(Message).save(new Message({
+┊   ┊370┊        chat,
+┊   ┊371┊        sender: currentUser,
 ┊287┊372┊        content,
-┊288┊   ┊        createdAt: moment().toDate(),
 ┊289┊373┊        type: MessageType.TEXT,
-┊290┊   ┊        recipients,
-┊291┊   ┊        holderIds,
-┊292┊   ┊      };
-┊293┊   ┊
-┊294┊   ┊      chats = chats.map(chat => {
-┊295┊   ┊        if (chat.id === Number(chatId)) {
-┊296┊   ┊          chat = {...chat, messages: chat.messages.concat(message)}
-┊297┊   ┊        }
-┊298┊   ┊        return chat;
-┊299┊   ┊      });
+┊   ┊374┊        holders,
+┊   ┊375┊        recipients: holders.reduce<Recipient[]>((filtered, holder) => {
+┊   ┊376┊          if (holder.id !== currentUser.id) {
+┊   ┊377┊            filtered.push(new Recipient({
+┊   ┊378┊              user: holder,
+┊   ┊379┊            }));
+┊   ┊380┊          }
+┊   ┊381┊          return filtered;
+┊   ┊382┊        }, []),
+┊   ┊383┊      }));
 ┊300┊384┊
 ┊301┊385┊      pubsub.publish('messageAdded', {
 ┊302┊386┊        messageAdded: message,
 ┊303┊387┊      });
 ┊304┊388┊
-┊305┊   ┊      return message;
+┊   ┊389┊      return message || null;
 ┊306┊390┊    },
-┊307┊   ┊    removeMessages: (obj, {chatId, messageIds, all}, {currentUser}) => {
-┊308┊   ┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊391┊    removeMessages: async (obj, {chatId, messageIds, all}, {currentUser, connection}) => {
+┊   ┊392┊      const chat = await connection
+┊   ┊393┊        .createQueryBuilder(Chat, "chat")
+┊   ┊394┊        .whereInIds(chatId)
+┊   ┊395┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊396┊        .innerJoinAndSelect('chat.messages', 'messages')
+┊   ┊397┊        .innerJoinAndSelect('messages.holders', 'holders')
+┊   ┊398┊        .getOne();
 ┊309┊399┊
 ┊310┊400┊      if (!chat) {
 ┊311┊401┊        throw new Error(`Cannot find chat ${chatId}.`);
 ┊312┊402┊      }
 ┊313┊403┊
-┊314┊   ┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊404┊      if (!chat.listingMembers.find(user => user.id === currentUser.id)) {
 ┊315┊405┊        throw new Error(`The chat/group ${chatId} is not listed for the current user, so there is nothing to delete.`);
 ┊316┊406┊      }
 ┊317┊407┊
```
```diff
@@ -319,109 +409,228 @@
 ┊319┊409┊        throw new Error(`Cannot specify both 'all' and 'messageIds'.`);
 ┊320┊410┊      }
 ┊321┊411┊
+┊   ┊412┊      if (!all && !(messageIds && messageIds.length)) {
+┊   ┊413┊        throw new Error(`'all' and 'messageIds' cannot be both null`);
+┊   ┊414┊      }
+┊   ┊415┊
 ┊322┊416┊      let deletedIds: string[] = [];
-┊323┊   ┊      chats = chats.map(chat => {
-┊324┊   ┊        if (chat.id === Number(chatId)) {
-┊325┊   ┊          // Instead of chaining map and filter we can loop once using reduce
-┊326┊   ┊          const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
-┊327┊   ┊            if (all || messageIds!.map(Number).includes(message.id)) {
-┊328┊   ┊              deletedIds.push(String(message.id));
-┊329┊   ┊              // Remove the current user from the message holders
-┊330┊   ┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
-┊331┊   ┊            }
+┊   ┊417┊      // Instead of chaining map and filter we can loop once using reduce
+┊   ┊418┊      chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊419┊        const filtered = await filtered$;
 ┊332┊420┊
-┊333┊   ┊            if (message.holderIds.length !== 0) {
-┊334┊   ┊              filtered.push(message);
-┊335┊   ┊            } // else discard the message
+┊   ┊421┊        if (all || messageIds!.map(Number).includes(message.id)) {
+┊   ┊422┊          deletedIds.push(String(message.id));
+┊   ┊423┊          // Remove the current user from the message holders
+┊   ┊424┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
 ┊336┊425┊
-┊337┊   ┊            return filtered;
-┊338┊   ┊          }, []);
-┊339┊   ┊          chat = {...chat, messages};
 ┊340┊426┊        }
-┊341┊   ┊        return chat;
-┊342┊   ┊      });
+┊   ┊427┊
+┊   ┊428┊        if (message.holders.length !== 0) {
+┊   ┊429┊          // Remove the current user from the message holders
+┊   ┊430┊          await connection.getRepository(Message).save(message);
+┊   ┊431┊          filtered.push(message);
+┊   ┊432┊        } else {
+┊   ┊433┊          // Simply remove the message
+┊   ┊434┊          const recipients = await connection
+┊   ┊435┊            .createQueryBuilder(Recipient, "recipient")
+┊   ┊436┊            .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊437┊            .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊438┊            .getMany();
+┊   ┊439┊          for (let recipient of recipients) {
+┊   ┊440┊            await connection.getRepository(Recipient).remove(recipient);
+┊   ┊441┊          }
+┊   ┊442┊          await connection.getRepository(Message).remove(message);
+┊   ┊443┊        }
+┊   ┊444┊
+┊   ┊445┊        return filtered;
+┊   ┊446┊      }, Promise.resolve([]));
+┊   ┊447┊
+┊   ┊448┊      await connection.getRepository(Chat).save(chat);
+┊   ┊449┊
 ┊343┊450┊      return deletedIds;
 ┊344┊451┊    },
 ┊345┊452┊  },
 ┊346┊453┊  Subscription: {
 ┊347┊454┊    messageAdded: {
 ┊348┊455┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
-┊349┊   ┊        ({messageAdded}: {messageAdded: MessageDb & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊456┊        ({messageAdded}: {messageAdded: Message}, {chatId}: MessageAddedSubscriptionArgs, {currentUser}: { currentUser: User }) => {
 ┊350┊457┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
-┊351┊   ┊            messageAdded.recipients.some(recipient => recipient.userId === currentUser.id);
+┊   ┊458┊            messageAdded.recipients.some(recipient => recipient.user.id === currentUser.id);
 ┊352┊459┊        }),
 ┊353┊460┊    },
 ┊354┊461┊    chatAdded: {
 ┊355┊462┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
-┊356┊   ┊        ({creatorId, chatAdded}: {creatorId: number, chatAdded: ChatDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
-┊357┊   ┊          return creatorId !== currentUser.id && chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊463┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables, {currentUser}: { currentUser: User }) => {
+┊   ┊464┊          return Number(creatorId) !== currentUser.id &&
+┊   ┊465┊            chatAdded.listingMembers.some(user => user.id === currentUser.id);
 ┊358┊466┊        }),
 ┊359┊467┊    },
 ┊360┊468┊    chatUpdated: {
 ┊361┊469┊      subscribe: withFilter(() => pubsub.asyncIterator('chatUpdated'),
-┊362┊   ┊        ({updaterId, chatUpdated}: {updaterId: number, chatUpdated: ChatDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
-┊363┊   ┊          return updaterId !== currentUser.id && chatUpdated.listingMemberIds.includes(currentUser.id);
+┊   ┊470┊        ({updaterId, chatUpdated}: {updaterId: number, chatUpdated: Chat}, variables: any, {currentUser}: { currentUser: User }) => {
+┊   ┊471┊          return updaterId !== currentUser.id && chatUpdated.listingMembers.some(user => user.id === currentUser.id);
 ┊364┊472┊        }),
 ┊365┊473┊    },
 ┊366┊474┊    userUpdated: {
 ┊367┊475┊      subscribe: withFilter(() => pubsub.asyncIterator('userUpdated'),
-┊368┊   ┊        ({userUpdated}: {userUpdated: UserDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊476┊        ({userUpdated}: {userUpdated: User}, variables: any, {currentUser}: { currentUser: User }) => {
 ┊369┊477┊          return userUpdated.id !== currentUser.id;
 ┊370┊478┊        }),
 ┊371┊479┊    },
 ┊372┊480┊    userAdded: {
 ┊373┊481┊      subscribe: withFilter(() => pubsub.asyncIterator('userAdded'),
-┊374┊   ┊        ({userAdded}: {userAdded: UserDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊482┊        ({userAdded}: {userAdded: User}, variables: any, {currentUser}: { currentUser: User }) => {
 ┊375┊483┊          return userAdded.id !== currentUser.id;
 ┊376┊484┊        }),
 ┊377┊485┊    },
 ┊378┊486┊  },
 ┊379┊487┊  Chat: {
-┊380┊   ┊    name: (chat, args, {currentUser}) => chat.name ? chat.name : users
-┊381┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
-┊382┊   ┊    picture: (chat, args, {currentUser}) => chat.name ? chat.picture : users
-┊383┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.picture,
-┊384┊   ┊    allTimeMembers: (chat) => users.filter(user => chat.allTimeMemberIds.includes(user.id)),
-┊385┊   ┊    listingMembers: (chat) => users.filter(user => chat.listingMemberIds.includes(user.id)),
-┊386┊   ┊    actualGroupMembers: (chat) => users.filter(user => chat.actualGroupMemberIds && chat.actualGroupMemberIds.includes(user.id)),
-┊387┊   ┊    admins: (chat) => users.filter(user => chat.adminIds && chat.adminIds.includes(user.id)),
-┊388┊   ┊    owner: (chat) => users.find(user => chat.ownerId === user.id) || null,
+┊   ┊488┊    name: async (chat, args, {currentUser, connection}) => {
+┊   ┊489┊      if (chat.name) {
+┊   ┊490┊        return chat.name;
+┊   ┊491┊      }
+┊   ┊492┊      const user = await connection
+┊   ┊493┊        .createQueryBuilder(User, "user")
+┊   ┊494┊        .where('user.id != :userId', {userId: currentUser.id})
+┊   ┊495┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊496┊        .getOne();
+┊   ┊497┊      return user && user.name || null;
+┊   ┊498┊    },
+┊   ┊499┊    picture: async (chat, args, {currentUser, connection}) => {
+┊   ┊500┊      if (chat.name) {
+┊   ┊501┊        return chat.picture;
+┊   ┊502┊      }
+┊   ┊503┊      const user = await connection
+┊   ┊504┊        .createQueryBuilder(User, "user")
+┊   ┊505┊        .where('user.id != :userId', {userId: currentUser.id})
+┊   ┊506┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊507┊        .getOne();
+┊   ┊508┊      return user ? user.picture : null;
+┊   ┊509┊    },
+┊   ┊510┊    allTimeMembers: async (chat, args, {currentUser, connection}) => {
+┊   ┊511┊      return await connection
+┊   ┊512┊        .createQueryBuilder(User, "user")
+┊   ┊513┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊514┊        .getMany();
+┊   ┊515┊    },
+┊   ┊516┊    listingMembers: async (chat, args, {currentUser, connection}) => {
+┊   ┊517┊      return await connection
+┊   ┊518┊        .createQueryBuilder(User, "user")
+┊   ┊519┊        .innerJoin('user.listingMemberChats', 'listingMemberChats', 'listingMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊520┊        .getMany();
+┊   ┊521┊    },
+┊   ┊522┊    actualGroupMembers: async (chat, args, {currentUser, connection}) => {
+┊   ┊523┊      return await connection
+┊   ┊524┊        .createQueryBuilder(User, "user")
+┊   ┊525┊        .innerJoin('user.actualGroupMemberChats', 'actualGroupMemberChats', 'actualGroupMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊526┊        .getMany();
+┊   ┊527┊    },
+┊   ┊528┊    admins: async (chat, args, {currentUser, connection}) => {
+┊   ┊529┊      return await connection
+┊   ┊530┊        .createQueryBuilder(User, "user")
+┊   ┊531┊        .innerJoin('user.adminChats', 'adminChats', 'adminChats.id = :chatId', {chatId: chat.id})
+┊   ┊532┊        .getMany();
+┊   ┊533┊    },
+┊   ┊534┊    owner: async (chat, args, {currentUser, connection}) => {
+┊   ┊535┊      return await connection
+┊   ┊536┊        .createQueryBuilder(User, "user")
+┊   ┊537┊        .innerJoin('user.ownerChats', 'ownerChats', 'ownerChats.id = :chatId', {chatId: chat.id})
+┊   ┊538┊        .getOne() || null;
+┊   ┊539┊    },
 ┊389┊540┊    isGroup: (chat) => !!chat.name,
-┊390┊   ┊    messages: (chat, {amount = 0}, {currentUser}) => {
-┊391┊   ┊      const messages = chat.messages
-┊392┊   ┊      .filter(message => message.holderIds.includes(currentUser.id))
-┊393┊   ┊      .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf()) || [];
-┊394┊   ┊      return (amount ? messages.slice(0, amount) : messages).reverse();
+┊   ┊541┊    messages: async (chat, {amount = 0}, {currentUser, connection}) => {
+┊   ┊542┊      if (chat.messages) {
+┊   ┊543┊        return amount ? chat.messages.slice(-amount) : chat.messages;
+┊   ┊544┊      }
+┊   ┊545┊
+┊   ┊546┊      let query = connection
+┊   ┊547┊        .createQueryBuilder(Message, "message")
+┊   ┊548┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', { chatId: chat.id })
+┊   ┊549┊        .innerJoin('message.holders', 'holders', 'holders.id = :userId', {
+┊   ┊550┊          userId: currentUser.id,
+┊   ┊551┊        })
+┊   ┊552┊        .orderBy({ 'message.createdAt': { order: 'DESC', nulls: 'NULLS LAST' } });
+┊   ┊553┊
+┊   ┊554┊      if (amount) {
+┊   ┊555┊        query = query.take(amount);
+┊   ┊556┊      }
+┊   ┊557┊
+┊   ┊558┊      return (await query.getMany()).reverse();
 ┊395┊559┊    },
-┊396┊   ┊    lastMessage: (chat, args, {currentUser}) => {
-┊397┊   ┊      return chat.messages
-┊398┊   ┊        .filter(message => message.holderIds.includes(currentUser.id))
-┊399┊   ┊        .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf())[0] || null;
+┊   ┊560┊    lastMessage: async (chat, args, {currentUser, connection}) => {
+┊   ┊561┊      if (chat.messages) {
+┊   ┊562┊        return chat.messages.length ? chat.messages[chat.messages.length - 1] : null;
+┊   ┊563┊      }
+┊   ┊564┊
+┊   ┊565┊      return await connection
+┊   ┊566┊        .createQueryBuilder(Message, "message")
+┊   ┊567┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', { chatId: chat.id })
+┊   ┊568┊        .innerJoin('message.holders', 'holders', 'holders.id = :userId', {
+┊   ┊569┊          userId: currentUser.id,
+┊   ┊570┊        })
+┊   ┊571┊        .orderBy({ 'message.createdAt': { order: 'DESC', nulls: 'NULLS LAST' } })
+┊   ┊572┊        .take(1)
+┊   ┊573┊        .getOne() || null;
 ┊400┊574┊    },
-┊401┊   ┊    updatedAt: (chat, args, {currentUser}) => {
-┊402┊   ┊      const lastMessage = chat.messages
-┊403┊   ┊        .filter(message => message.holderIds.includes(currentUser.id))
-┊404┊   ┊        .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf())[0];
+┊   ┊575┊    updatedAt: async (chat, args, {currentUser, connection}) => {
+┊   ┊576┊      if (chat.messages) {
+┊   ┊577┊        return chat.messages.length ? chat.messages[0].createdAt : null;
+┊   ┊578┊      }
+┊   ┊579┊
+┊   ┊580┊      const latestMessage = await connection
+┊   ┊581┊        .createQueryBuilder(Message, "message")
+┊   ┊582┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', { chatId: chat.id })
+┊   ┊583┊        .innerJoin('message.holders', 'holders', 'holders.id = :userId', {
+┊   ┊584┊          userId: currentUser.id,
+┊   ┊585┊        })
+┊   ┊586┊        .orderBy({ 'message.createdAt': 'DESC' })
+┊   ┊587┊        .getOne();
 ┊405┊588┊
-┊406┊   ┊      return lastMessage ? lastMessage.createdAt : chat.createdAt;
+┊   ┊589┊      return latestMessage ? latestMessage.createdAt : null;
+┊   ┊590┊    },
+┊   ┊591┊    unreadMessages: async (chat, args, {currentUser, connection}) => {
+┊   ┊592┊      return await connection
+┊   ┊593┊        .createQueryBuilder(Message, "message")
+┊   ┊594┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', {chatId: chat.id})
+┊   ┊595┊        .innerJoin('message.recipients', 'recipients', 'recipients.user.id = :userId AND recipients.readAt IS NULL', {
+┊   ┊596┊          userId: currentUser.id
+┊   ┊597┊        })
+┊   ┊598┊        .getCount();
 ┊407┊599┊    },
-┊408┊   ┊    unreadMessages: (chat, args, {currentUser}) => chat.messages
-┊409┊   ┊      .filter(message => message.holderIds.includes(currentUser.id) &&
-┊410┊   ┊        message.recipients.find(recipient => recipient.userId === currentUser.id && !recipient.readAt))
-┊411┊   ┊      .length,
 ┊412┊600┊  },
 ┊413┊601┊  Message: {
-┊414┊   ┊    chat: (message) => chats.find(chat => message.chatId === chat.id)!,
-┊415┊   ┊    sender: (message) => users.find(user => user.id === message.senderId)!,
-┊416┊   ┊    holders: (message) => users.filter(user => message.holderIds.includes(user.id)),
-┊417┊   ┊    ownership: (message, args, {currentUser}) => message.senderId === currentUser.id,
-┊418┊   ┊  },
-┊419┊   ┊  Recipient: {
-┊420┊   ┊    user: (recipient) => users.find(user => recipient.userId === user.id)!,
-┊421┊   ┊    message: (recipient) => {
-┊422┊   ┊      const chat = chats.find(chat => recipient.chatId === chat.id)!;
-┊423┊   ┊      return chat.messages.find(message => recipient.messageId === message.id)!;
+┊   ┊602┊    sender: async (message: Message, args, {currentUser, connection}) => {
+┊   ┊603┊      return (await connection
+┊   ┊604┊        .createQueryBuilder(User, "user")
+┊   ┊605┊        .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {messageId: message.id})
+┊   ┊606┊        .getOne())!;
+┊   ┊607┊    },
+┊   ┊608┊    ownership: async (message: Message, args, {currentUser, connection}) => {
+┊   ┊609┊      return !!(await connection
+┊   ┊610┊        .createQueryBuilder(User, "user")
+┊   ┊611┊        .whereInIds(currentUser.id)
+┊   ┊612┊        .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {messageId: message.id})
+┊   ┊613┊        .getCount());
+┊   ┊614┊    },
+┊   ┊615┊    recipients: async (message: Message, args, {currentUser, connection}) => {
+┊   ┊616┊      return await connection
+┊   ┊617┊        .createQueryBuilder(Recipient, "recipient")
+┊   ┊618┊        .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊619┊        .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊620┊        .innerJoinAndSelect('recipient.chat', 'chat')
+┊   ┊621┊        .getMany();
+┊   ┊622┊    },
+┊   ┊623┊    holders: async (message: Message, args, {currentUser, connection}) => {
+┊   ┊624┊      return await connection
+┊   ┊625┊        .createQueryBuilder(User, "user")
+┊   ┊626┊        .innerJoin('user.holderMessages', 'holderMessages', 'holderMessages.id = :messageId', {messageId: message.id})
+┊   ┊627┊        .getMany();
+┊   ┊628┊    },
+┊   ┊629┊    chat: async (message: Message, args, {currentUser, connection})=> {
+┊   ┊630┊      return (await connection
+┊   ┊631┊        .createQueryBuilder(Chat, "chat")
+┊   ┊632┊        .innerJoin('chat.messages', 'messages', 'messages.id = :messageId', {messageId: message.id})
+┊   ┊633┊        .getOne())!;
 ┊424┊634┊    },
-┊425┊   ┊    chat: (recipient) => chats.find(chat => recipient.chatId === chat.id)!,
 ┊426┊635┊  },
 ┊427┊636┊};
```

[}]: #

`QueryBuilder` is one of the most powerful features of `TypeORM` - it allows you to build SQL queries using elegant and convenient syntax, execute them and get automatically transformed entities.

You can find more informations on `TypeORM` on http://typeorm.io

The best part is that you won't have to do anything on the client, everything will be completely transparent to it, even if migrated from NoSQL-like db structure to a relational one!
Of course, you could remove the custom normalization for the messages because now they have their own table and they are no longer embedded (so they have unique IDs), but we could leave it as well in order to be free to use any kind of backend.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step11.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step13.md) |
|:--------------------------------|--------------------------------:|

[}]: #
