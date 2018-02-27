# Step 16: TypeORM with PostgreSQL

[//]: # (head-end)


## Server

First of all you will have to install PostgreSQL on your operating system. Since there so many options (different Linux distributions, MacOS X, Windows...) I will assume that you already know how to install a software in your OS and take that part for granted.

Then you will have to install a couple of packages:

    yarn add pg reflect-metadata typeorm
    yarn add -D @types/pg

We aren't going to use plain SQL, instead we will use an Object-relational mapping framework (ORM) called `TypeORM`.
`TypeORM` takes advantage of Typescript classes and type declarations in order to infer the db structure.

We will need to enable support for experimental decorators, emit type metadata for decorators and disable strict property initialization:

[{]: <helper> (diffStep "6.1" files="tsconfig.json" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed tsconfig.json
```diff
@@ -22,11 +22,11 @@
 ┊22┊22┊    // "isolatedModules": true,               /* Transpile each file as a separate module (similar to 'ts.transpileModule'). */
 ┊23┊23┊
 ┊24┊24┊    /* Strict Type-Checking Options */
-┊25┊  ┊    "strict": true,                            /* Enable all strict type-checking options. */
+┊  ┊25┊    "strict": true,                           /* Enable all strict type-checking options. */
 ┊26┊26┊    // "noImplicitAny": true,                 /* Raise error on expressions and declarations with an implied 'any' type. */
 ┊27┊27┊    // "strictNullChecks": true,              /* Enable strict null checks. */
 ┊28┊28┊    // See https://github.com/DefinitelyTyped/DefinitelyTyped/issues/21359
-┊29┊  ┊    "strictFunctionTypes": false              /* Enable strict checking of function types. */
+┊  ┊29┊    "strictFunctionTypes": false,             /* Enable strict checking of function types. */
 ┊30┊30┊    // "noImplicitThis": true,                /* Raise error on 'this' expressions with an implied 'any' type. */
 ┊31┊31┊    // "alwaysStrict": true,                  /* Parse in strict mode and emit "use strict" for each source file. */
 ┊32┊32┊
```
```diff
@@ -53,7 +53,8 @@
 ┊53┊53┊    // "inlineSources": true,                 /* Emit the source alongside the sourcemaps within a single file; requires '--inlineSourceMap' or '--sourceMap' to be set. */
 ┊54┊54┊
 ┊55┊55┊    /* Experimental Options */
-┊56┊  ┊    // "experimentalDecorators": true,        /* Enables experimental support for ES7 decorators. */
-┊57┊  ┊    // "emitDecoratorMetadata": true,         /* Enables experimental support for emitting type metadata for decorators. */
+┊  ┊56┊    "experimentalDecorators": true,           /* Enables experimental support for ES7 decorators. */
+┊  ┊57┊    "emitDecoratorMetadata": true,            /* Enables experimental support for emitting type metadata for decorators. */
+┊  ┊58┊    "strictPropertyInitialization": false
 ┊58┊59┊  }
 ┊59┊60┊}🚫↵
```

[}]: #

The next step is to create Entities. An Entity is a class that maps to a database table. You can create a entity by defining a new class and mark it with @Entity():

[{]: <helper> (diffStep "6.1" files="entity" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Added entity&#x2F;Chat.ts
```diff
@@ -0,0 +1,79 @@
+┊  ┊ 1┊import { Entity, Column, PrimaryGeneratedColumn, OneToMany, JoinTable, ManyToMany, ManyToOne } from "typeorm";
+┊  ┊ 2┊import { Message } from "./Message";
+┊  ┊ 3┊import { User } from "./User";
+┊  ┊ 4┊import { Recipient } from "./Recipient";
+┊  ┊ 5┊
+┊  ┊ 6┊interface ChatConstructor {
+┊  ┊ 7┊  name?: string;
+┊  ┊ 8┊  picture?: string;
+┊  ┊ 9┊  allTimeMembers?: User[];
+┊  ┊10┊  listingMembers?: User[];
+┊  ┊11┊  actualGroupMembers?: User[];
+┊  ┊12┊  admins?: User[];
+┊  ┊13┊  owner?: User;
+┊  ┊14┊  messages?: Message[];
+┊  ┊15┊}
+┊  ┊16┊
+┊  ┊17┊@Entity()
+┊  ┊18┊export class Chat {
+┊  ┊19┊  @PrimaryGeneratedColumn()
+┊  ┊20┊  id: number;
+┊  ┊21┊
+┊  ┊22┊  @Column({nullable: true})
+┊  ┊23┊  name: string;
+┊  ┊24┊
+┊  ┊25┊  @Column({nullable: true})
+┊  ┊26┊  picture: string;
+┊  ┊27┊
+┊  ┊28┊  @ManyToMany(type => User, user => user.allTimeMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊29┊  @JoinTable()
+┊  ┊30┊  allTimeMembers: User[];
+┊  ┊31┊
+┊  ┊32┊  @ManyToMany(type => User, user => user.listingMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊33┊  @JoinTable()
+┊  ┊34┊  listingMembers: User[];
+┊  ┊35┊
+┊  ┊36┊  @ManyToMany(type => User, user => user.actualGroupMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊37┊  @JoinTable()
+┊  ┊38┊  actualGroupMembers?: User[];
+┊  ┊39┊
+┊  ┊40┊  @ManyToMany(type => User, user => user.adminChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊41┊  @JoinTable()
+┊  ┊42┊  admins?: User[];
+┊  ┊43┊
+┊  ┊44┊  @ManyToOne(type => User, user => user.ownerChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊45┊  owner?: User | null;
+┊  ┊46┊
+┊  ┊47┊  @OneToMany(type => Message, message => message.chat, {cascade: ["insert", "update"], eager: true})
+┊  ┊48┊  messages: Message[];
+┊  ┊49┊
+┊  ┊50┊  @OneToMany(type => Recipient, recipient => recipient.chat)
+┊  ┊51┊  recipients: Recipient[];
+┊  ┊52┊
+┊  ┊53┊  constructor({name, picture, allTimeMembers, listingMembers, actualGroupMembers, admins, owner, messages}: ChatConstructor = {}) {
+┊  ┊54┊    if (name) {
+┊  ┊55┊      this.name = name;
+┊  ┊56┊    }
+┊  ┊57┊    if (picture) {
+┊  ┊58┊      this.picture = picture;
+┊  ┊59┊    }
+┊  ┊60┊    if (allTimeMembers) {
+┊  ┊61┊      this.allTimeMembers = allTimeMembers;
+┊  ┊62┊    }
+┊  ┊63┊    if (listingMembers) {
+┊  ┊64┊      this.listingMembers = listingMembers;
+┊  ┊65┊    }
+┊  ┊66┊    if (actualGroupMembers) {
+┊  ┊67┊      this.actualGroupMembers = actualGroupMembers;
+┊  ┊68┊    }
+┊  ┊69┊    if (admins) {
+┊  ┊70┊      this.admins = admins;
+┊  ┊71┊    }
+┊  ┊72┊    if (owner) {
+┊  ┊73┊      this.owner = owner;
+┊  ┊74┊    }
+┊  ┊75┊    if (messages) {
+┊  ┊76┊      this.messages = messages;
+┊  ┊77┊    }
+┊  ┊78┊  }
+┊  ┊79┊}
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

[{]: <helper> (diffStep "6.1" files="ormconfig.json" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Added ormconfig.json
```diff
@@ -0,0 +1,24 @@
+┊  ┊ 1┊{
+┊  ┊ 2┊   "type": "postgres",
+┊  ┊ 3┊   "host": "localhost",
+┊  ┊ 4┊   "port": 5432,
+┊  ┊ 5┊   "username": "test",
+┊  ┊ 6┊   "password": "",
+┊  ┊ 7┊   "database": "test",
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
+┊  ┊24┊}🚫↵
```

[}]: #

Since database table consist of columns your entities must consist of columns too. Each entity class property you marked with @Column will be mapped to a database table column.
Each entity must have at least one primary column. There are several types of primary columns, but in our case `@PrimaryGeneratedColumn()` creates a primary column which value will be automatically generated with an auto-increment value.
`@CreateDateColumn` is a special column that is automatically set to the entity's insertion date. You don't need set this column - it will be automatically set.
For the Recipient Entity we use a composite primary key that consists of two foreign keys.

The next thing to do is to create a connection with the database before firing up the web server:

[{]: <helper> (diffStep "6.1" files="index.ts" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed index.ts
```diff
@@ -1,3 +1,5 @@
+┊ ┊1┊// For TypeORM
+┊ ┊2┊import "reflect-metadata";
 ┊1┊3┊import { schema } from "./schema";
 ┊2┊4┊import * as bodyParser from "body-parser";
 ┊3┊5┊import * as cors from 'cors';
```
```diff
@@ -6,12 +8,10 @@
 ┊ 6┊ 8┊import * as passport from "passport";
 ┊ 7┊ 9┊import * as basicStrategy from 'passport-http';
 ┊ 8┊10┊import * as bcrypt from 'bcrypt-nodejs';
-┊ 9┊  ┊import { db, User } from "./db";
 ┊10┊11┊import { createServer } from "http";
-┊11┊  ┊
-┊12┊  ┊let users = db.users;
-┊13┊  ┊
-┊14┊  ┊console.log(users);
+┊  ┊12┊import { createConnection } from "typeorm";
+┊  ┊13┊import { User } from "./entity/User";
+┊  ┊14┊import { addSampleData } from "./db";
 ┊15┊15┊
 ┊16┊16┊function generateHash(password: string) {
 ┊17┊17┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
```
```diff
@@ -21,93 +21,96 @@
 ┊ 21┊ 21┊  return bcrypt.compareSync(password, localPassword);
 ┊ 22┊ 22┊}
 ┊ 23┊ 23┊
-┊ 24┊   ┊passport.use('basic-signin', new basicStrategy.BasicStrategy(
-┊ 25┊   ┊  function (username: string, password: string, done: any) {
-┊ 26┊   ┊    const user = users.find(user => user.username == username);
-┊ 27┊   ┊    if (user && validPassword(password, user.password)) {
-┊ 28┊   ┊      return done(null, user);
+┊   ┊ 24┊createConnection().then(async connection => {
+┊   ┊ 25┊  await addSampleData(connection);
+┊   ┊ 26┊
+┊   ┊ 27┊  passport.use('basic-signin', new basicStrategy.BasicStrategy(
+┊   ┊ 28┊    async function (username: string, password: string, done: any) {
+┊   ┊ 29┊      const user = await connection.getRepository(User).findOne({where: { username }});
+┊   ┊ 30┊      if (user && validPassword(password, user.password)) {
+┊   ┊ 31┊        return done(null, user);
+┊   ┊ 32┊      }
+┊   ┊ 33┊      return done(null, false);
 ┊ 29┊ 34┊    }
-┊ 30┊   ┊    return done(null, false);
-┊ 31┊   ┊  }
-┊ 32┊   ┊));
-┊ 33┊   ┊
-┊ 34┊   ┊passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
-┊ 35┊   ┊  function (req: any, username: any, password: any, done: any) {
-┊ 36┊   ┊    const userExists = !!users.find(user => user.username === username);
-┊ 37┊   ┊    if (!userExists && password && req.body.name) {
-┊ 38┊   ┊      const user: User = {
-┊ 39┊   ┊        id: (users.length && users[users.length - 1].id + 1) || 1,
-┊ 40┊   ┊        username,
-┊ 41┊   ┊        password: generateHash(password),
-┊ 42┊   ┊        name: req.body.name,
-┊ 43┊   ┊      };
-┊ 44┊   ┊      users.push(user);
-┊ 45┊   ┊      return done(null, user);
+┊   ┊ 35┊  ));
+┊   ┊ 36┊
+┊   ┊ 37┊  passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
+┊   ┊ 38┊    async function (req: any, username: string, password: string, done: any) {
+┊   ┊ 39┊      const userExists = !!(await connection.getRepository(User).findOne({where: { username }}));
+┊   ┊ 40┊      if (!userExists && password && req.body.name) {
+┊   ┊ 41┊        const user = await connection.manager.save(new User({
+┊   ┊ 42┊          username,
+┊   ┊ 43┊          password: generateHash(password),
+┊   ┊ 44┊          name: req.body.name,
+┊   ┊ 45┊        }));
+┊   ┊ 46┊        return done(null, user);
+┊   ┊ 47┊      }
+┊   ┊ 48┊      return done(null, false);
 ┊ 46┊ 49┊    }
-┊ 47┊   ┊    return done(null, false);
-┊ 48┊   ┊  }
-┊ 49┊   ┊));
+┊   ┊ 50┊  ));
 ┊ 50┊ 51┊
-┊ 51┊   ┊const PORT = 3000;
+┊   ┊ 52┊  const PORT = 3000;
 ┊ 52┊ 53┊
-┊ 53┊   ┊const app = express();
+┊   ┊ 54┊  const app = express();
 ┊ 54┊ 55┊
-┊ 55┊   ┊app.use(cors());
-┊ 56┊   ┊app.use(bodyParser.json());
-┊ 57┊   ┊app.use(passport.initialize());
+┊   ┊ 56┊  app.use(cors());
+┊   ┊ 57┊  app.use(bodyParser.json());
+┊   ┊ 58┊  app.use(passport.initialize());
 ┊ 58┊ 59┊
-┊ 59┊   ┊app.post('/signup',
-┊ 60┊   ┊  passport.authenticate('basic-signup', {session: false}),
-┊ 61┊   ┊  function (req: any, res) {
-┊ 62┊   ┊    res.json(req.user);
-┊ 63┊   ┊  });
+┊   ┊ 60┊  app.post('/signup',
+┊   ┊ 61┊    passport.authenticate('basic-signup', {session: false}),
+┊   ┊ 62┊    function (req: any, res) {
+┊   ┊ 63┊      res.json(req.user);
+┊   ┊ 64┊    });
 ┊ 64┊ 65┊
-┊ 65┊   ┊app.use(passport.authenticate('basic-signin', {session: false}));
+┊   ┊ 66┊  app.use(passport.authenticate('basic-signin', {session: false}));
 ┊ 66┊ 67┊
-┊ 67┊   ┊app.post('/signin', function (req: any, res) {
-┊ 68┊   ┊  res.json(req.user);
-┊ 69┊   ┊});
+┊   ┊ 68┊  app.post('/signin', function (req, res) {
+┊   ┊ 69┊    res.json(req.user);
+┊   ┊ 70┊  });
 ┊ 70┊ 71┊
-┊ 71┊   ┊const apollo = new ApolloServer({
-┊ 72┊   ┊  schema,
-┊ 73┊   ┊  context(received: any) {
-┊ 74┊   ┊    return {
-┊ 75┊   ┊      user: received.connection ? received.connection.context.user : received.req!['user'],
-┊ 76┊   ┊    }
-┊ 77┊   ┊  },
-┊ 78┊   ┊  subscriptions: {
-┊ 79┊   ┊    onConnect: (connectionParams: any, webSocket: any) => {
-┊ 80┊   ┊      if (connectionParams.authToken) {
-┊ 81┊   ┊        // create a buffer and tell it the data coming in is base64
-┊ 82┊   ┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
-┊ 83┊   ┊        // read it back out as a string
-┊ 84┊   ┊        const [username, password]: string[] = buf.toString().split(':');
-┊ 85┊   ┊        if (username && password) {
-┊ 86┊   ┊          const user = users.find(user => user.username == username);
-┊ 87┊   ┊
-┊ 88┊   ┊          if (user && validPassword(password, user.password)) {
-┊ 89┊   ┊            // Set context for the WebSocket
-┊ 90┊   ┊            return {user};
-┊ 91┊   ┊          } else {
-┊ 92┊   ┊            throw new Error('Wrong credentials!');
+┊   ┊ 72┊  const apollo = new ApolloServer({
+┊   ┊ 73┊    schema,
+┊   ┊ 74┊    context(received: any) {
+┊   ┊ 75┊      return {
+┊   ┊ 76┊        user: received.connection ? received.connection.context.user : received.req!['user'],
+┊   ┊ 77┊        connection,
+┊   ┊ 78┊      }
+┊   ┊ 79┊    },
+┊   ┊ 80┊    subscriptions: {
+┊   ┊ 81┊      onConnect: async (connectionParams: any, webSocket: any) => {
+┊   ┊ 82┊        if (connectionParams.authToken) {
+┊   ┊ 83┊          // Create a buffer and tell it the data coming in is base64
+┊   ┊ 84┊          const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊   ┊ 85┊          // Read it back out as a string
+┊   ┊ 86┊          const [username, password]: string[] = buf.toString().split(':');
+┊   ┊ 87┊          if (username && password) {
+┊   ┊ 88┊            const user = await connection.getRepository(User).findOne({where: { username }});
+┊   ┊ 89┊
+┊   ┊ 90┊            if (user && validPassword(password, user.password)) {
+┊   ┊ 91┊              // Set context for the WebSocket
+┊   ┊ 92┊              return {user};
+┊   ┊ 93┊            } else {
+┊   ┊ 94┊              throw new Error('Wrong credentials!');
+┊   ┊ 95┊            }
 ┊ 93┊ 96┊          }
 ┊ 94┊ 97┊        }
+┊   ┊ 98┊        throw new Error('Missing auth token!');
 ┊ 95┊ 99┊      }
-┊ 96┊   ┊      throw new Error('Missing auth token!');
 ┊ 97┊100┊    }
-┊ 98┊   ┊  }
-┊ 99┊   ┊});
+┊   ┊101┊  });
 ┊100┊102┊
-┊101┊   ┊apollo.applyMiddleware({
-┊102┊   ┊  app,
-┊103┊   ┊  path: '/graphql'
-┊104┊   ┊});
+┊   ┊103┊  apollo.applyMiddleware({
+┊   ┊104┊    app,
+┊   ┊105┊    path: '/graphql'
+┊   ┊106┊  });
 ┊105┊107┊
-┊106┊   ┊// Wrap the Express server
-┊107┊   ┊const ws = createServer(app);
+┊   ┊108┊  // Wrap the Express server
+┊   ┊109┊  const ws = createServer(app);
 ┊108┊110┊
-┊109┊   ┊apollo.installSubscriptionHandlers(ws);
+┊   ┊111┊  apollo.installSubscriptionHandlers(ws);
 ┊110┊112┊
-┊111┊   ┊ws.listen(PORT, () => {
-┊112┊   ┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊113┊  ws.listen(PORT, () => {
+┊   ┊114┊    console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊115┊  });
 ┊113┊116┊});
```

[}]: #

We will also remove our fake db and replace it with some real data:

[{]: <helper> (diffStep "6.1" files="db.ts" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed db.ts
```diff
@@ -1,4 +1,11 @@
+┊  ┊ 1┊// For TypeORM
+┊  ┊ 2┊import "reflect-metadata";
+┊  ┊ 3┊import { Chat } from "./entity/Chat";
+┊  ┊ 4┊import { Recipient } from "./entity/Recipient";
 ┊ 1┊ 5┊import * as moment from 'moment';
+┊  ┊ 6┊import { Message } from "./entity/Message";
+┊  ┊ 7┊import { User } from "./entity/User";
+┊  ┊ 8┊import { Connection } from "typeorm";
 ┊ 2┊ 9┊
 ┊ 3┊10┊export enum MessageType {
 ┊ 4┊11┊  PICTURE,
```
```diff
@@ -6,433 +13,280 @@
 ┊  6┊ 13┊  LOCATION,
 ┊  7┊ 14┊}
 ┊  8┊ 15┊
-┊  9┊   ┊export interface User {
-┊ 10┊   ┊  id: number,
-┊ 11┊   ┊  username: string,
-┊ 12┊   ┊  password: string,
-┊ 13┊   ┊  name: string,
-┊ 14┊   ┊  picture?: string | null,
-┊ 15┊   ┊  phone?: string | null,
-┊ 16┊   ┊}
-┊ 17┊   ┊
-┊ 18┊   ┊export interface Chat {
-┊ 19┊   ┊  id: number,
-┊ 20┊   ┊  name?: string | null,
-┊ 21┊   ┊  picture?: string | null,
-┊ 22┊   ┊  // All members, current and past ones.
-┊ 23┊   ┊  allTimeMemberIds: number[],
-┊ 24┊   ┊  // Whoever gets the chat listed. For groups includes past members who still didn't delete the group.
-┊ 25┊   ┊  listingMemberIds: number[],
-┊ 26┊   ┊  // Actual members of the group (they are not the only ones who get the group listed). Null for chats.
-┊ 27┊   ┊  actualGroupMemberIds?: number[] | null,
-┊ 28┊   ┊  adminIds?: number[] | null,
-┊ 29┊   ┊  ownerId?: number | null,
-┊ 30┊   ┊  messages: Message[],
-┊ 31┊   ┊}
-┊ 32┊   ┊
-┊ 33┊   ┊export interface Message {
-┊ 34┊   ┊  id: number,
-┊ 35┊   ┊  chatId: number,
-┊ 36┊   ┊  senderId: number,
-┊ 37┊   ┊  content: string,
-┊ 38┊   ┊  createdAt: number,
-┊ 39┊   ┊  type: MessageType,
-┊ 40┊   ┊  recipients: Recipient[],
-┊ 41┊   ┊  holderIds: number[],
-┊ 42┊   ┊}
-┊ 43┊   ┊
-┊ 44┊   ┊export interface Recipient {
-┊ 45┊   ┊  userId: number,
-┊ 46┊   ┊  messageId: number,
-┊ 47┊   ┊  chatId: number,
-┊ 48┊   ┊  receivedAt: number | null,
-┊ 49┊   ┊  readAt: number | null,
-┊ 50┊   ┊}
-┊ 51┊   ┊
-┊ 52┊   ┊const users: User[] = [
-┊ 53┊   ┊  {
-┊ 54┊   ┊    id: 1,
+┊   ┊ 16┊export async function addSampleData(connection: Connection) {
+┊   ┊ 17┊  const user1 = new User({
 ┊ 55┊ 18┊    username: 'ethan',
 ┊ 56┊ 19┊    password: '$2a$08$NO9tkFLCoSqX1c5wk3s7z.JfxaVMKA.m7zUDdDwEquo4rvzimQeJm', // 111
 ┊ 57┊ 20┊    name: 'Ethan Gonzalez',
 ┊ 58┊ 21┊    picture: 'https://randomuser.me/api/portraits/thumb/men/1.jpg',
 ┊ 59┊ 22┊    phone: '+391234567890',
-┊ 60┊   ┊  },
-┊ 61┊   ┊  {
-┊ 62┊   ┊    id: 2,
+┊   ┊ 23┊  });
+┊   ┊ 24┊  await connection.manager.save(user1);
+┊   ┊ 25┊
+┊   ┊ 26┊  const user2 = new User({
 ┊ 63┊ 27┊    username: 'bryan',
 ┊ 64┊ 28┊    password: '$2a$08$xE4FuCi/ifxjL2S8CzKAmuKLwv18ktksSN.F3XYEnpmcKtpbpeZgO', // 222
 ┊ 65┊ 29┊    name: 'Bryan Wallace',
 ┊ 66┊ 30┊    picture: 'https://randomuser.me/api/portraits/thumb/men/2.jpg',
 ┊ 67┊ 31┊    phone: '+391234567891',
-┊ 68┊   ┊  },
-┊ 69┊   ┊  {
-┊ 70┊   ┊    id: 3,
+┊   ┊ 32┊  });
+┊   ┊ 33┊  await connection.manager.save(user2);
+┊   ┊ 34┊
+┊   ┊ 35┊  const user3 = new User({
 ┊ 71┊ 36┊    username: 'avery',
 ┊ 72┊ 37┊    password: '$2a$08$UHgH7J8G6z1mGQn2qx2kdeWv0jvgHItyAsL9hpEUI3KJmhVW5Q1d.', // 333
 ┊ 73┊ 38┊    name: 'Avery Stewart',
 ┊ 74┊ 39┊    picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
 ┊ 75┊ 40┊    phone: '+391234567892',
-┊ 76┊   ┊  },
-┊ 77┊   ┊  {
-┊ 78┊   ┊    id: 4,
+┊   ┊ 41┊  });
+┊   ┊ 42┊  await connection.manager.save(user3);
+┊   ┊ 43┊
+┊   ┊ 44┊  const user4 = new User({
 ┊ 79┊ 45┊    username: 'katie',
 ┊ 80┊ 46┊    password: '$2a$08$wR1k5Q3T9FC7fUgB7Gdb9Os/GV7dGBBf4PLlWT7HERMFhmFDt47xi', // 444
 ┊ 81┊ 47┊    name: 'Katie Peterson',
 ┊ 82┊ 48┊    picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
 ┊ 83┊ 49┊    phone: '+391234567893',
-┊ 84┊   ┊  },
-┊ 85┊   ┊  {
-┊ 86┊   ┊    id: 5,
+┊   ┊ 50┊  });
+┊   ┊ 51┊  await connection.manager.save(user4);
+┊   ┊ 52┊
+┊   ┊ 53┊  const user5 = new User({
 ┊ 87┊ 54┊    username: 'ray',
 ┊ 88┊ 55┊    password: '$2a$08$6.mbXqsDX82ZZ7q5d8Osb..JrGSsNp4R3IKj7mxgF6YGT0OmMw242', // 555
 ┊ 89┊ 56┊    name: 'Ray Edwards',
 ┊ 90┊ 57┊    picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
 ┊ 91┊ 58┊    phone: '+391234567894',
-┊ 92┊   ┊  },
-┊ 93┊   ┊  {
-┊ 94┊   ┊    id: 6,
+┊   ┊ 59┊  });
+┊   ┊ 60┊  await connection.manager.save(user5);
+┊   ┊ 61┊
+┊   ┊ 62┊  const user6 = new User({
 ┊ 95┊ 63┊    username: 'niko',
 ┊ 96┊ 64┊    password: '$2a$08$fL5lZR.Rwf9FWWe8XwwlceiPBBim8n9aFtaem.INQhiKT4.Ux3Uq.', // 666
 ┊ 97┊ 65┊    name: 'Niccolò Belli',
 ┊ 98┊ 66┊    picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
 ┊ 99┊ 67┊    phone: '+391234567895',
-┊100┊   ┊  },
-┊101┊   ┊  {
-┊102┊   ┊    id: 7,
+┊   ┊ 68┊  });
+┊   ┊ 69┊  await connection.manager.save(user6);
+┊   ┊ 70┊
+┊   ┊ 71┊  const user7 = new User({
 ┊103┊ 72┊    username: 'mario',
 ┊104┊ 73┊    password: '$2a$08$nDHDmWcVxDnH5DDT3HMMC.psqcnu6wBiOgkmJUy9IH..qxa3R6YrO', // 777
 ┊105┊ 74┊    name: 'Mario Rossi',
 ┊106┊ 75┊    picture: 'https://randomuser.me/api/portraits/thumb/men/5.jpg',
 ┊107┊ 76┊    phone: '+391234567896',
-┊108┊   ┊  },
-┊109┊   ┊];
+┊   ┊ 77┊  });
+┊   ┊ 78┊  await connection.manager.save(user7);
+┊   ┊ 79┊
+┊   ┊ 80┊
 ┊110┊ 81┊
-┊111┊   ┊const chats: Chat[] = [
-┊112┊   ┊  {
-┊113┊   ┊    id: 1,
-┊114┊   ┊    name: null,
-┊115┊   ┊    picture: null,
-┊116┊   ┊    allTimeMemberIds: [1, 3],
-┊117┊   ┊    listingMemberIds: [1, 3],
-┊118┊   ┊    adminIds: null,
-┊119┊   ┊    ownerId: null,
+┊   ┊ 82┊
+┊   ┊ 83┊  await connection.manager.save(new Chat({
+┊   ┊ 84┊    allTimeMembers: [user1, user3],
+┊   ┊ 85┊    listingMembers: [user1, user3],
 ┊120┊ 86┊    messages: [
-┊121┊   ┊      {
-┊122┊   ┊        id: 1,
-┊123┊   ┊        chatId: 1,
-┊124┊   ┊        senderId: 1,
+┊   ┊ 87┊      new Message({
+┊   ┊ 88┊        sender: user1,
 ┊125┊ 89┊        content: 'You on your way?',
-┊126┊   ┊        createdAt: moment().subtract(1, 'hours').unix(),
+┊   ┊ 90┊        createdAt: moment().subtract(1, 'hours').toDate(),
 ┊127┊ 91┊        type: MessageType.TEXT,
+┊   ┊ 92┊        holders: [user1, user3],
 ┊128┊ 93┊        recipients: [
-┊129┊   ┊          {
-┊130┊   ┊            userId: 3,
-┊131┊   ┊            messageId: 1,
-┊132┊   ┊            chatId: 1,
-┊133┊   ┊            receivedAt: null,
-┊134┊   ┊            readAt: null,
-┊135┊   ┊          },
+┊   ┊ 94┊          new Recipient({
+┊   ┊ 95┊            user: user3,
+┊   ┊ 96┊          }),
 ┊136┊ 97┊        ],
-┊137┊   ┊        holderIds: [1, 3],
-┊138┊   ┊      },
-┊139┊   ┊      {
-┊140┊   ┊        id: 2,
-┊141┊   ┊        chatId: 1,
-┊142┊   ┊        senderId: 3,
+┊   ┊ 98┊      }),
+┊   ┊ 99┊      new Message({
+┊   ┊100┊        sender: user3,
 ┊143┊101┊        content: 'Yep!',
-┊144┊   ┊        createdAt: moment().subtract(1, 'hours').add(5, 'minutes').unix(),
+┊   ┊102┊        createdAt: moment().subtract(1, 'hours').add(5, 'minutes').toDate(),
 ┊145┊103┊        type: MessageType.TEXT,
+┊   ┊104┊        holders: [user1, user3],
 ┊146┊105┊        recipients: [
-┊147┊   ┊          {
-┊148┊   ┊            userId: 1,
-┊149┊   ┊            messageId: 2,
-┊150┊   ┊            chatId: 1,
-┊151┊   ┊            receivedAt: null,
-┊152┊   ┊            readAt: null,
-┊153┊   ┊          },
+┊   ┊106┊          new Recipient({
+┊   ┊107┊            user: user1,
+┊   ┊108┊          }),
 ┊154┊109┊        ],
-┊155┊   ┊        holderIds: [3, 1],
-┊156┊   ┊      },
+┊   ┊110┊      }),
 ┊157┊111┊    ],
-┊158┊   ┊  },
-┊159┊   ┊  {
-┊160┊   ┊    id: 2,
-┊161┊   ┊    name: null,
-┊162┊   ┊    picture: null,
-┊163┊   ┊    allTimeMemberIds: [1, 4],
-┊164┊   ┊    listingMemberIds: [1, 4],
-┊165┊   ┊    adminIds: null,
-┊166┊   ┊    ownerId: null,
+┊   ┊112┊  }));
+┊   ┊113┊
+┊   ┊114┊  await connection.manager.save(new Chat({
+┊   ┊115┊    allTimeMembers: [user1, user4],
+┊   ┊116┊    listingMembers: [user1, user4],
 ┊167┊117┊    messages: [
-┊168┊   ┊      {
-┊169┊   ┊        id: 1,
-┊170┊   ┊        chatId: 2,
-┊171┊   ┊        senderId: 1,
+┊   ┊118┊      new Message({
+┊   ┊119┊        sender: user1,
 ┊172┊120┊        content: 'Hey, it\'s me',
-┊173┊   ┊        createdAt: moment().subtract(2, 'hours').unix(),
+┊   ┊121┊        createdAt: moment().subtract(2, 'hours').toDate(),
 ┊174┊122┊        type: MessageType.TEXT,
+┊   ┊123┊        holders: [user1, user4],
 ┊175┊124┊        recipients: [
-┊176┊   ┊          {
-┊177┊   ┊            userId: 4,
-┊178┊   ┊            messageId: 1,
-┊179┊   ┊            chatId: 2,
-┊180┊   ┊            receivedAt: null,
-┊181┊   ┊            readAt: null,
-┊182┊   ┊          },
+┊   ┊125┊          new Recipient({
+┊   ┊126┊            user: user4,
+┊   ┊127┊          }),
 ┊183┊128┊        ],
-┊184┊   ┊        holderIds: [1, 4],
-┊185┊   ┊      },
+┊   ┊129┊      }),
 ┊186┊130┊    ],
-┊187┊   ┊  },
-┊188┊   ┊  {
-┊189┊   ┊    id: 3,
-┊190┊   ┊    name: null,
-┊191┊   ┊    picture: null,
-┊192┊   ┊    allTimeMemberIds: [1, 5],
-┊193┊   ┊    listingMemberIds: [1, 5],
-┊194┊   ┊    adminIds: null,
-┊195┊   ┊    ownerId: null,
+┊   ┊131┊  }));
+┊   ┊132┊
+┊   ┊133┊  await connection.manager.save(new Chat({
+┊   ┊134┊    allTimeMembers: [user1, user5],
+┊   ┊135┊    listingMembers: [user1, user5],
 ┊196┊136┊    messages: [
-┊197┊   ┊      {
-┊198┊   ┊        id: 1,
-┊199┊   ┊        chatId: 3,
-┊200┊   ┊        senderId: 1,
+┊   ┊137┊      new Message({
+┊   ┊138┊        sender: user1,
 ┊201┊139┊        content: 'I should buy a boat',
-┊202┊   ┊        createdAt: moment().subtract(1, 'days').unix(),
+┊   ┊140┊        createdAt: moment().subtract(1, 'days').toDate(),
 ┊203┊141┊        type: MessageType.TEXT,
+┊   ┊142┊        holders: [user1, user5],
 ┊204┊143┊        recipients: [
-┊205┊   ┊          {
-┊206┊   ┊            userId: 5,
-┊207┊   ┊            messageId: 1,
-┊208┊   ┊            chatId: 3,
-┊209┊   ┊            receivedAt: null,
-┊210┊   ┊            readAt: null,
-┊211┊   ┊          },
+┊   ┊144┊          new Recipient({
+┊   ┊145┊            user: user5,
+┊   ┊146┊          }),
 ┊212┊147┊        ],
-┊213┊   ┊        holderIds: [1, 5],
-┊214┊   ┊      },
-┊215┊   ┊      {
-┊216┊   ┊        id: 2,
-┊217┊   ┊        chatId: 3,
-┊218┊   ┊        senderId: 1,
+┊   ┊148┊      }),
+┊   ┊149┊      new Message({
+┊   ┊150┊        sender: user1,
 ┊219┊151┊        content: 'You still there?',
-┊220┊   ┊        createdAt: moment().subtract(1, 'days').add(16, 'hours').unix(),
+┊   ┊152┊        createdAt: moment().subtract(1, 'days').add(16, 'hours').toDate(),
 ┊221┊153┊        type: MessageType.TEXT,
+┊   ┊154┊        holders: [user1, user5],
 ┊222┊155┊        recipients: [
-┊223┊   ┊          {
-┊224┊   ┊            userId: 5,
-┊225┊   ┊            messageId: 2,
-┊226┊   ┊            chatId: 3,
-┊227┊   ┊            receivedAt: null,
-┊228┊   ┊            readAt: null,
-┊229┊   ┊          },
+┊   ┊156┊          new Recipient({
+┊   ┊157┊            user: user5,
+┊   ┊158┊          }),
 ┊230┊159┊        ],
-┊231┊   ┊        holderIds: [1, 5],
-┊232┊   ┊      },
+┊   ┊160┊      }),
 ┊233┊161┊    ],
-┊234┊   ┊  },
-┊235┊   ┊  {
-┊236┊   ┊    id: 4,
-┊237┊   ┊    name: null,
-┊238┊   ┊    picture: null,
-┊239┊   ┊    allTimeMemberIds: [3, 4],
-┊240┊   ┊    listingMemberIds: [3, 4],
-┊241┊   ┊    adminIds: null,
-┊242┊   ┊    ownerId: null,
+┊   ┊162┊  }));
+┊   ┊163┊
+┊   ┊164┊  await connection.manager.save(new Chat({
+┊   ┊165┊    allTimeMembers: [user3, user4],
+┊   ┊166┊    listingMembers: [user3, user4],
 ┊243┊167┊    messages: [
-┊244┊   ┊      {
-┊245┊   ┊        id: 1,
-┊246┊   ┊        chatId: 4,
-┊247┊   ┊        senderId: 3,
+┊   ┊168┊      new Message({
+┊   ┊169┊        sender: user3,
 ┊248┊170┊        content: 'Look at my mukluks!',
-┊249┊   ┊        createdAt: moment().subtract(4, 'days').unix(),
+┊   ┊171┊        createdAt: moment().subtract(4, 'days').toDate(),
 ┊250┊172┊        type: MessageType.TEXT,
+┊   ┊173┊        holders: [user3, user4],
 ┊251┊174┊        recipients: [
-┊252┊   ┊          {
-┊253┊   ┊            userId: 4,
-┊254┊   ┊            messageId: 1,
-┊255┊   ┊            chatId: 4,
-┊256┊   ┊            receivedAt: null,
-┊257┊   ┊            readAt: null,
-┊258┊   ┊          },
+┊   ┊175┊          new Recipient({
+┊   ┊176┊            user: user4,
+┊   ┊177┊          }),
 ┊259┊178┊        ],
-┊260┊   ┊        holderIds: [3, 4],
-┊261┊   ┊      },
+┊   ┊179┊      }),
 ┊262┊180┊    ],
-┊263┊   ┊  },
-┊264┊   ┊  {
-┊265┊   ┊    id: 5,
-┊266┊   ┊    name: null,
-┊267┊   ┊    picture: null,
-┊268┊   ┊    allTimeMemberIds: [2, 5],
-┊269┊   ┊    listingMemberIds: [2, 5],
-┊270┊   ┊    adminIds: null,
-┊271┊   ┊    ownerId: null,
+┊   ┊181┊  }));
+┊   ┊182┊
+┊   ┊183┊  await connection.manager.save(new Chat({
+┊   ┊184┊    allTimeMembers: [user2, user5],
+┊   ┊185┊    listingMembers: [user2, user5],
 ┊272┊186┊    messages: [
-┊273┊   ┊      {
-┊274┊   ┊        id: 1,
-┊275┊   ┊        chatId: 5,
-┊276┊   ┊        senderId: 2,
+┊   ┊187┊      new Message({
+┊   ┊188┊        sender: user2,
 ┊277┊189┊        content: 'This is wicked good ice cream.',
-┊278┊   ┊        createdAt: moment().subtract(2, 'weeks').unix(),
+┊   ┊190┊        createdAt: moment().subtract(2, 'weeks').toDate(),
 ┊279┊191┊        type: MessageType.TEXT,
+┊   ┊192┊        holders: [user2, user5],
 ┊280┊193┊        recipients: [
-┊281┊   ┊          {
-┊282┊   ┊            userId: 5,
-┊283┊   ┊            messageId: 1,
-┊284┊   ┊            chatId: 5,
-┊285┊   ┊            receivedAt: null,
-┊286┊   ┊            readAt: null,
-┊287┊   ┊          },
+┊   ┊194┊          new Recipient({
+┊   ┊195┊            user: user5,
+┊   ┊196┊          }),
 ┊288┊197┊        ],
-┊289┊   ┊        holderIds: [2, 5],
-┊290┊   ┊      },
-┊291┊   ┊      {
-┊292┊   ┊        id: 2,
-┊293┊   ┊        chatId: 6,
-┊294┊   ┊        senderId: 5,
+┊   ┊198┊      }),
+┊   ┊199┊      new Message({
+┊   ┊200┊        sender: user5,
 ┊295┊201┊        content: 'Love it!',
-┊296┊   ┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').unix(),
+┊   ┊202┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
 ┊297┊203┊        type: MessageType.TEXT,
+┊   ┊204┊        holders: [user2, user5],
 ┊298┊205┊        recipients: [
-┊299┊   ┊          {
-┊300┊   ┊            userId: 2,
-┊301┊   ┊            messageId: 2,
-┊302┊   ┊            chatId: 5,
-┊303┊   ┊            receivedAt: null,
-┊304┊   ┊            readAt: null,
-┊305┊   ┊          },
+┊   ┊206┊          new Recipient({
+┊   ┊207┊            user: user2,
+┊   ┊208┊          }),
 ┊306┊209┊        ],
-┊307┊   ┊        holderIds: [5, 2],
-┊308┊   ┊      },
+┊   ┊210┊      }),
 ┊309┊211┊    ],
-┊310┊   ┊  },
-┊311┊   ┊  {
-┊312┊   ┊    id: 6,
-┊313┊   ┊    name: null,
-┊314┊   ┊    picture: null,
-┊315┊   ┊    allTimeMemberIds: [1, 6],
-┊316┊   ┊    listingMemberIds: [1],
-┊317┊   ┊    adminIds: null,
-┊318┊   ┊    ownerId: null,
-┊319┊   ┊    messages: [],
-┊320┊   ┊  },
-┊321┊   ┊  {
-┊322┊   ┊    id: 7,
-┊323┊   ┊    name: null,
-┊324┊   ┊    picture: null,
-┊325┊   ┊    allTimeMemberIds: [2, 1],
-┊326┊   ┊    listingMemberIds: [2],
-┊327┊   ┊    adminIds: null,
-┊328┊   ┊    ownerId: null,
-┊329┊   ┊    messages: [],
-┊330┊   ┊  },
-┊331┊   ┊  {
-┊332┊   ┊    id: 8,
-┊333┊   ┊    name: 'A user 0 group',
+┊   ┊212┊  }));
+┊   ┊213┊
+┊   ┊214┊  await connection.manager.save(new Chat({
+┊   ┊215┊    allTimeMembers: [user1, user6],
+┊   ┊216┊    listingMembers: [user1],
+┊   ┊217┊  }));
+┊   ┊218┊
+┊   ┊219┊  await connection.manager.save(new Chat({
+┊   ┊220┊    allTimeMembers: [user2, user1],
+┊   ┊221┊    listingMembers: [user2],
+┊   ┊222┊  }));
+┊   ┊223┊
+┊   ┊224┊  await connection.manager.save(new Chat({
+┊   ┊225┊    name: 'Ethan\'s group',
 ┊334┊226┊    picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
-┊335┊   ┊    allTimeMemberIds: [1, 3, 4, 6],
-┊336┊   ┊    listingMemberIds: [1, 3, 4, 6],
-┊337┊   ┊    actualGroupMemberIds: [1, 4, 6],
-┊338┊   ┊    adminIds: [1, 6],
-┊339┊   ┊    ownerId: 1,
+┊   ┊227┊    allTimeMembers: [user1, user3, user4, user6],
+┊   ┊228┊    listingMembers: [user1, user3, user4, user6],
+┊   ┊229┊    actualGroupMembers: [user1, user4, user6],
+┊   ┊230┊    admins: [user1, user6],
+┊   ┊231┊    owner: user1,
 ┊340┊232┊    messages: [
-┊341┊   ┊      {
-┊342┊   ┊        id: 1,
-┊343┊   ┊        chatId: 8,
-┊344┊   ┊        senderId: 1,
+┊   ┊233┊      new Message({
+┊   ┊234┊        sender: user1,
 ┊345┊235┊        content: 'I made a group',
-┊346┊   ┊        createdAt: moment().subtract(2, 'weeks').unix(),
+┊   ┊236┊        createdAt: moment().subtract(2, 'weeks').toDate(),
 ┊347┊237┊        type: MessageType.TEXT,
+┊   ┊238┊        holders: [user1, user3, user4, user6],
 ┊348┊239┊        recipients: [
-┊349┊   ┊          {
-┊350┊   ┊            userId: 3,
-┊351┊   ┊            messageId: 1,
-┊352┊   ┊            chatId: 8,
-┊353┊   ┊            receivedAt: null,
-┊354┊   ┊            readAt: null,
-┊355┊   ┊          },
-┊356┊   ┊          {
-┊357┊   ┊            userId: 4,
-┊358┊   ┊            messageId: 1,
-┊359┊   ┊            chatId: 8,
-┊360┊   ┊            receivedAt: moment().subtract(2, 'weeks').add(1, 'minutes').unix(),
-┊361┊   ┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').unix(),
-┊362┊   ┊          },
-┊363┊   ┊          {
-┊364┊   ┊            userId: 6,
-┊365┊   ┊            messageId: 1,
-┊366┊   ┊            chatId: 8,
-┊367┊   ┊            receivedAt: null,
-┊368┊   ┊            readAt: null,
-┊369┊   ┊          },
+┊   ┊240┊          new Recipient({
+┊   ┊241┊            user: user3,
+┊   ┊242┊          }),
+┊   ┊243┊          new Recipient({
+┊   ┊244┊            user: user4,
+┊   ┊245┊          }),
+┊   ┊246┊          new Recipient({
+┊   ┊247┊            user: user6,
+┊   ┊248┊          }),
 ┊370┊249┊        ],
-┊371┊   ┊        holderIds: [1, 3, 4, 6],
-┊372┊   ┊      },
-┊373┊   ┊      {
-┊374┊   ┊        id: 2,
-┊375┊   ┊        chatId: 8,
-┊376┊   ┊        senderId: 1,
-┊377┊   ┊        content: 'Ops, user 3 was not supposed to be here',
-┊378┊   ┊        createdAt: moment().subtract(2, 'weeks').add(2, 'minutes').unix(),
+┊   ┊250┊      }),
+┊   ┊251┊      new Message({
+┊   ┊252┊        sender: user1,
+┊   ┊253┊        content: 'Ops, Avery was not supposed to be here',
+┊   ┊254┊        createdAt: moment().subtract(2, 'weeks').add(2, 'minutes').toDate(),
 ┊379┊255┊        type: MessageType.TEXT,
+┊   ┊256┊        holders: [user1, user4, user6],
 ┊380┊257┊        recipients: [
-┊381┊   ┊          {
-┊382┊   ┊            userId: 4,
-┊383┊   ┊            messageId: 2,
-┊384┊   ┊            chatId: 8,
-┊385┊   ┊            receivedAt: moment().subtract(2, 'weeks').add(3, 'minutes').unix(),
-┊386┊   ┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').unix(),
-┊387┊   ┊          },
-┊388┊   ┊          {
-┊389┊   ┊            userId: 6,
-┊390┊   ┊            messageId: 2,
-┊391┊   ┊            chatId: 8,
-┊392┊   ┊            receivedAt: null,
-┊393┊   ┊            readAt: null,
-┊394┊   ┊          },
+┊   ┊258┊          new Recipient({
+┊   ┊259┊            user: user4,
+┊   ┊260┊          }),
+┊   ┊261┊          new Recipient({
+┊   ┊262┊            user: user6,
+┊   ┊263┊          }),
 ┊395┊264┊        ],
-┊396┊   ┊        holderIds: [1, 4, 6],
-┊397┊   ┊      },
-┊398┊   ┊      {
-┊399┊   ┊        id: 3,
-┊400┊   ┊        chatId: 8,
-┊401┊   ┊        senderId: 4,
+┊   ┊265┊      }),
+┊   ┊266┊      new Message({
+┊   ┊267┊        sender: user4,
 ┊402┊268┊        content: 'Awesome!',
-┊403┊   ┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').unix(),
+┊   ┊269┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
 ┊404┊270┊        type: MessageType.TEXT,
+┊   ┊271┊        holders: [user1, user4, user6],
 ┊405┊272┊        recipients: [
-┊406┊   ┊          {
-┊407┊   ┊            userId: 1,
-┊408┊   ┊            messageId: 3,
-┊409┊   ┊            chatId: 8,
-┊410┊   ┊            receivedAt: null,
-┊411┊   ┊            readAt: null,
-┊412┊   ┊          },
-┊413┊   ┊          {
-┊414┊   ┊            userId: 6,
-┊415┊   ┊            messageId: 3,
-┊416┊   ┊            chatId: 8,
-┊417┊   ┊            receivedAt: null,
-┊418┊   ┊            readAt: null,
-┊419┊   ┊          },
+┊   ┊273┊          new Recipient({
+┊   ┊274┊            user: user1,
+┊   ┊275┊          }),
+┊   ┊276┊          new Recipient({
+┊   ┊277┊            user: user6,
+┊   ┊278┊          }),
 ┊420┊279┊        ],
-┊421┊   ┊        holderIds: [1, 4, 6],
-┊422┊   ┊      },
+┊   ┊280┊      }),
 ┊423┊281┊    ],
-┊424┊   ┊  },
-┊425┊   ┊  {
-┊426┊   ┊    id: 9,
-┊427┊   ┊    name: 'A user 5 group',
-┊428┊   ┊    picture: null,
-┊429┊   ┊    allTimeMemberIds: [6, 3],
-┊430┊   ┊    listingMemberIds: [6, 3],
-┊431┊   ┊    actualGroupMemberIds: [6, 3],
-┊432┊   ┊    adminIds: [6],
-┊433┊   ┊    ownerId: 6,
-┊434┊   ┊    messages: [],
-┊435┊   ┊  },
-┊436┊   ┊];
+┊   ┊282┊  }));
 ┊437┊283┊
-┊438┊   ┊export const db = {users, chats};
+┊   ┊284┊  await connection.manager.save(new Chat({
+┊   ┊285┊    name: 'Ray\'s group',
+┊   ┊286┊    allTimeMembers: [user3, user6],
+┊   ┊287┊    listingMembers: [user3, user6],
+┊   ┊288┊    actualGroupMembers: [user3, user6],
+┊   ┊289┊    admins: [user6],
+┊   ┊290┊    owner: user6,
+┊   ┊291┊  }));
+┊   ┊292┊}
```

[}]: #

It's time to deal with resolvers:

[{]: <helper> (diffStep "6.1" files="schema/resolvers.ts" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,91 +1,127 @@
 ┊  1┊  1┊import { PubSub, withFilter, IResolvers } from 'apollo-server-express';
-┊  2┊   ┊import { Chat, db, Message, MessageType, Recipient, User } from "../db";
+┊   ┊  2┊import { MessageType } from "../db";
 ┊  3┊  3┊import {
 ┊  4┊  4┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs, MessageAddedSubscriptionArgs,
 ┊  5┊  5┊  RemoveChatMutationArgs, RemoveMessagesMutationArgs
 ┊  6┊  6┊} from "../types";
 ┊  7┊  7┊import * as moment from "moment";
-┊  8┊   ┊
-┊  9┊   ┊let users = db.users;
-┊ 10┊   ┊let chats = db.chats;
+┊   ┊  8┊import { User } from "../entity/User";
+┊   ┊  9┊import { Chat } from "../entity/Chat";
+┊   ┊ 10┊import { Message } from "../entity/Message";
+┊   ┊ 11┊import { Recipient } from "../entity/Recipient";
+┊   ┊ 12┊import { Connection } from "typeorm";
 ┊ 11┊ 13┊
 ┊ 12┊ 14┊export const pubsub = new PubSub();
 ┊ 13┊ 15┊
 ┊ 14┊ 16┊export const resolvers: IResolvers = {
 ┊ 15┊ 17┊  Query: {
 ┊ 16┊ 18┊    // Show all users for the moment.
-┊ 17┊   ┊    users: (obj: any, args: any, {user: currentUser}: {user: User}): User[] => users.filter(user => user.id !== currentUser.id),
-┊ 18┊   ┊    chats: (obj: any, args: any, {user: currentUser}: {user: User}): Chat[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
-┊ 19┊   ┊    chat: (obj: any, {chatId}: ChatQueryArgs): Chat | null => chats.find(chat => chat.id === Number(chatId)) || null,
+┊   ┊ 19┊    users: async (obj: any, args: any, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<User[]> => {
+┊   ┊ 20┊      return await connection
+┊   ┊ 21┊        .createQueryBuilder(User, "user")
+┊   ┊ 22┊        .where('user.id != :id', {id: currentUser.id})
+┊   ┊ 23┊        .getMany();
+┊   ┊ 24┊    },
+┊   ┊ 25┊    chats: async (obj: any, args: any, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<any[]> => {
+┊   ┊ 26┊      return await connection
+┊   ┊ 27┊        .createQueryBuilder(Chat, "chat")
+┊   ┊ 28┊        .leftJoin('chat.listingMembers', 'listingMembers')
+┊   ┊ 29┊        .where('listingMembers.id = :id', {id: currentUser.id})
+┊   ┊ 30┊        .getMany();
+┊   ┊ 31┊    },
+┊   ┊ 32┊    chat: async (obj: any, {chatId}: ChatQueryArgs, {connection}: { user: User, connection: Connection }): Promise<any> => {
+┊   ┊ 33┊      return await connection
+┊   ┊ 34┊        .createQueryBuilder(Chat, "chat")
+┊   ┊ 35┊        .whereInIds(chatId)
+┊   ┊ 36┊        .getOne();
+┊   ┊ 37┊    },
 ┊ 20┊ 38┊  },
 ┊ 21┊ 39┊  Mutation: {
-┊ 22┊   ┊    addChat: (obj: any, {recipientId}: AddChatMutationArgs, {user: currentUser}: {user: User}): Chat => {
-┊ 23┊   ┊      if (!users.find(user => user.id === Number(recipientId))) {
+┊   ┊ 40┊    addChat: async (obj: any, {recipientId}: AddChatMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<Chat | null> => {
+┊   ┊ 41┊      const recipient = await connection
+┊   ┊ 42┊        .createQueryBuilder(User, "user")
+┊   ┊ 43┊        .whereInIds(recipientId)
+┊   ┊ 44┊        .getOne();
+┊   ┊ 45┊
+┊   ┊ 46┊      if (!recipient) {
 ┊ 24┊ 47┊        throw new Error(`Recipient ${recipientId} doesn't exist.`);
 ┊ 25┊ 48┊      }
 ┊ 26┊ 49┊
-┊ 27┊   ┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser.id) && chat.allTimeMemberIds.includes(Number(recipientId)));
+┊   ┊ 50┊      let chat = await connection
+┊   ┊ 51┊        .createQueryBuilder(Chat, "chat")
+┊   ┊ 52┊        .where('chat.name IS NULL')
+┊   ┊ 53┊        .innerJoin('chat.allTimeMembers', 'allTimeMembers1', 'allTimeMembers1.id = :currentUserId', {currentUserId: currentUser.id})
+┊   ┊ 54┊        .innerJoin('chat.allTimeMembers', 'allTimeMembers2', 'allTimeMembers2.id = :recipientId', {recipientId})
+┊   ┊ 55┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊ 56┊        .getOne();
+┊   ┊ 57┊
 ┊ 28┊ 58┊      if (chat) {
-┊ 29┊   ┊        // Chat already exists. Both users are already in the allTimeMemberIds array
-┊ 30┊   ┊        const chatId = chat.id;
-┊ 31┊   ┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
+┊   ┊ 59┊        // Chat already exists. Both users are already in the userIds array
+┊   ┊ 60┊        const listingMembers = await connection
+┊   ┊ 61┊          .createQueryBuilder(User, "user")
+┊   ┊ 62┊          .innerJoin('user.listingMemberChats', 'listingMemberChats', 'listingMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊ 63┊          .getMany();
+┊   ┊ 64┊
+┊   ┊ 65┊        if (!listingMembers.find(user => user.id === currentUser.id)) {
 ┊ 32┊ 66┊          // The chat isn't listed for the current user. Add him to the memberIds
-┊ 33┊   ┊          chat.listingMemberIds.push(currentUser.id);
-┊ 34┊   ┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser.id);
-┊ 35┊   ┊          return chat;
+┊   ┊ 67┊          chat.listingMembers.push(currentUser);
+┊   ┊ 68┊          chat = await connection.getRepository(Chat).save(chat);
+┊   ┊ 69┊
+┊   ┊ 70┊          return chat || null;
 ┊ 36┊ 71┊        } else {
 ┊ 37┊ 72┊          throw new Error(`Chat already exists.`);
 ┊ 38┊ 73┊        }
 ┊ 39┊ 74┊      } else {
 ┊ 40┊ 75┊        // Create the chat
-┊ 41┊   ┊        const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
-┊ 42┊   ┊        const chat: Chat = {
-┊ 43┊   ┊          id,
-┊ 44┊   ┊          name: null,
-┊ 45┊   ┊          picture: null,
-┊ 46┊   ┊          adminIds: null,
-┊ 47┊   ┊          ownerId: null,
-┊ 48┊   ┊          allTimeMemberIds: [currentUser.id, Number(recipientId)],
+┊   ┊ 76┊        chat = await connection.getRepository(Chat).save(new Chat({
+┊   ┊ 77┊          allTimeMembers: [currentUser, recipient],
 ┊ 49┊ 78┊          // Chat will not be listed to the other user until the first message gets written
-┊ 50┊   ┊          listingMemberIds: [currentUser.id],
-┊ 51┊   ┊          actualGroupMemberIds: null,
-┊ 52┊   ┊          messages: [],
-┊ 53┊   ┊        };
-┊ 54┊   ┊        chats.push(chat);
+┊   ┊ 79┊          listingMembers: [currentUser],
+┊   ┊ 80┊        }));
 ┊ 55┊ 81┊
-┊ 56┊   ┊        return chat;
+┊   ┊ 82┊        return chat || null;
 ┊ 57┊ 83┊      }
 ┊ 58┊ 84┊    },
-┊ 59┊   ┊    addGroup: (obj: any, {recipientIds, groupName}: AddGroupMutationArgs, {user: currentUser}: {user: User}): Chat => {
-┊ 60┊   ┊      recipientIds.forEach(recipientId => {
-┊ 61┊   ┊        if (!users.find(user => user.id === Number(recipientId))) {
+┊   ┊ 85┊    addGroup: async (obj: any, {recipientIds, groupName}: AddGroupMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<Chat | null> => {
+┊   ┊ 86┊      let recipients: User[] = [];
+┊   ┊ 87┊      for (let recipientId of recipientIds) {
+┊   ┊ 88┊        const recipient = await connection
+┊   ┊ 89┊          .createQueryBuilder(User, "user")
+┊   ┊ 90┊          .whereInIds(recipientId)
+┊   ┊ 91┊          .getOne();
+┊   ┊ 92┊        if (!recipient) {
 ┊ 62┊ 93┊          throw new Error(`Recipient ${recipientId} doesn't exist.`);
 ┊ 63┊ 94┊        }
-┊ 64┊   ┊      });
+┊   ┊ 95┊        recipients.push(recipient);
+┊   ┊ 96┊      }
 ┊ 65┊ 97┊
-┊ 66┊   ┊      const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
-┊ 67┊   ┊      const chat: Chat = {
-┊ 68┊   ┊        id,
+┊   ┊ 98┊      const chat = await connection.getRepository(Chat).save(new Chat({
 ┊ 69┊ 99┊        name: groupName,
-┊ 70┊   ┊        picture: null,
-┊ 71┊   ┊        adminIds: [currentUser.id],
-┊ 72┊   ┊        ownerId: currentUser.id,
-┊ 73┊   ┊        allTimeMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
-┊ 74┊   ┊        listingMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
-┊ 75┊   ┊        actualGroupMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
-┊ 76┊   ┊        messages: [],
-┊ 77┊   ┊      };
-┊ 78┊   ┊      chats.push(chat);
+┊   ┊100┊        admins: [currentUser],
+┊   ┊101┊        owner: currentUser,
+┊   ┊102┊        allTimeMembers: [...recipients, currentUser],
+┊   ┊103┊        listingMembers: [...recipients, currentUser],
+┊   ┊104┊        actualGroupMembers: [...recipients, currentUser],
+┊   ┊105┊      }));
 ┊ 79┊106┊
 ┊ 80┊107┊      pubsub.publish('chatAdded', {
 ┊ 81┊108┊        creatorId: currentUser.id,
 ┊ 82┊109┊        chatAdded: chat,
 ┊ 83┊110┊      });
 ┊ 84┊111┊
-┊ 85┊   ┊      return chat;
+┊   ┊112┊      return chat || null;
 ┊ 86┊113┊    },
-┊ 87┊   ┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser}: {user: User}): number => {
-┊ 88┊   ┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊114┊    removeChat: async (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }) => {
+┊   ┊115┊      const chat = await connection
+┊   ┊116┊        .createQueryBuilder(Chat, "chat")
+┊   ┊117┊        .whereInIds(Number(chatId))
+┊   ┊118┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊119┊        .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊120┊        .leftJoinAndSelect('chat.admins', 'admins')
+┊   ┊121┊        .leftJoinAndSelect('chat.owner', 'owner')
+┊   ┊122┊        .leftJoinAndSelect('chat.messages', 'messages')
+┊   ┊123┊        .leftJoinAndSelect('messages.holders', 'holders')
+┊   ┊124┊        .getOne();
 ┊ 89┊125┊
 ┊ 90┊126┊      if (!chat) {
 ┊ 91┊127┊        throw new Error(`The chat ${chatId} doesn't exist.`);
```
```diff
@@ -93,186 +129,188 @@
 ┊ 93┊129┊
 ┊ 94┊130┊      if (!chat.name) {
 ┊ 95┊131┊        // Chat
-┊ 96┊   ┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
-┊ 97┊   ┊          throw new Error(`The user is not a member of the chat ${chatId}.`);
+┊   ┊132┊        if (!chat.listingMembers.find(user => user.id === currentUser.id)) {
+┊   ┊133┊          throw new Error(`The user is not a listing member of the chat ${chatId}.`);
 ┊ 98┊134┊        }
 ┊ 99┊135┊
 ┊100┊136┊        // Instead of chaining map and filter we can loop once using reduce
-┊101┊   ┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
-┊102┊   ┊          // Remove the current user from the message holders
-┊103┊   ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊137┊        chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊138┊          const filtered = await filtered$;
 ┊104┊139┊
-┊105┊   ┊          if (message.holderIds.length !== 0) {
+┊   ┊140┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
+┊   ┊141┊
+┊   ┊142┊          if (message.holders.length !== 0) {
+┊   ┊143┊            // Remove the current user from the message holders
+┊   ┊144┊            await connection.getRepository(Message).save(message);
 ┊106┊145┊            filtered.push(message);
-┊107┊   ┊          } // else discard the message
+┊   ┊146┊          } else {
+┊   ┊147┊            // Simply remove the message
+┊   ┊148┊            const recipients = await connection
+┊   ┊149┊              .createQueryBuilder(Recipient, "recipient")
+┊   ┊150┊              .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊151┊              .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊152┊              .getMany();
+┊   ┊153┊            for (let recipient of recipients) {
+┊   ┊154┊              await connection.getRepository(Recipient).remove(recipient);
+┊   ┊155┊            }
+┊   ┊156┊            await connection.getRepository(Message).remove(message);
+┊   ┊157┊          }
 ┊108┊158┊
 ┊109┊159┊          return filtered;
-┊110┊   ┊        }, []);
+┊   ┊160┊        }, Promise.resolve([]));
 ┊111┊161┊
 ┊112┊162┊        // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
-┊113┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊163┊        chat.listingMembers = chat.listingMembers.filter(user => user.id !== currentUser.id);
 ┊114┊164┊
 ┊115┊165┊        // Check how many members are left
-┊116┊   ┊        if (listingMemberIds.length === 0) {
+┊   ┊166┊        if (chat.listingMembers.length === 0) {
 ┊117┊167┊          // Delete the chat
-┊118┊   ┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊168┊          await connection.getRepository(Chat).remove(chat);
 ┊119┊169┊        } else {
 ┊120┊170┊          // Update the chat
-┊121┊   ┊          chats = chats.map(chat => {
-┊122┊   ┊            if (chat.id === Number(chatId)) {
-┊123┊   ┊              chat = {...chat, listingMemberIds, messages};
-┊124┊   ┊            }
-┊125┊   ┊            return chat;
-┊126┊   ┊          });
+┊   ┊171┊          await connection.getRepository(Chat).save(chat);
 ┊127┊172┊        }
-┊128┊   ┊        return Number(chatId);
+┊   ┊173┊        return chatId;
 ┊129┊174┊      } else {
 ┊130┊175┊        // Group
-┊131┊   ┊        if (chat.ownerId !== currentUser.id) {
-┊132┊   ┊          throw new Error(`Group ${chatId} is not owned by the user.`);
-┊133┊   ┊        }
 ┊134┊176┊
 ┊135┊177┊        // Instead of chaining map and filter we can loop once using reduce
-┊136┊   ┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
-┊137┊   ┊          // Remove the current user from the message holders
-┊138┊   ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊178┊        chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊179┊          const filtered = await filtered$;
+┊   ┊180┊
+┊   ┊181┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
 ┊139┊182┊
-┊140┊   ┊          if (message.holderIds.length !== 0) {
+┊   ┊183┊          if (message.holders.length !== 0) {
+┊   ┊184┊            // Remove the current user from the message holders
+┊   ┊185┊            await connection.getRepository(Message).save(message);
 ┊141┊186┊            filtered.push(message);
-┊142┊   ┊          } // else discard the message
+┊   ┊187┊          } else {
+┊   ┊188┊            // Simply remove the message
+┊   ┊189┊            const recipients = await connection
+┊   ┊190┊              .createQueryBuilder(Recipient, "recipient")
+┊   ┊191┊              .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊192┊              .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊193┊              .getMany();
+┊   ┊194┊            for (let recipient of recipients) {
+┊   ┊195┊              await connection.getRepository(Recipient).remove(recipient);
+┊   ┊196┊            }
+┊   ┊197┊            await connection.getRepository(Message).remove(message);
+┊   ┊198┊          }
 ┊143┊199┊
 ┊144┊200┊          return filtered;
-┊145┊   ┊        }, []);
+┊   ┊201┊        }, Promise.resolve([]));
 ┊146┊202┊
 ┊147┊203┊        // Remove the current user from who gets the group listed. The group will no longer appear in his list
-┊148┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊204┊        chat.listingMembers = chat.listingMembers.filter(user => user.id !== currentUser.id);
 ┊149┊205┊
 ┊150┊206┊        // Check how many members (including previous ones who can still access old messages) are left
-┊151┊   ┊        if (listingMemberIds.length === 0) {
+┊   ┊207┊        if (chat.listingMembers.length === 0) {
 ┊152┊208┊          // Remove the group
-┊153┊   ┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊209┊          await connection.getRepository(Chat).remove(chat);
 ┊154┊210┊        } else {
 ┊155┊211┊          // Update the group
 ┊156┊212┊
 ┊157┊213┊          // Remove the current user from the chat members. He is no longer a member of the group
-┊158┊   ┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser.id);
+┊   ┊214┊          chat.actualGroupMembers = chat.actualGroupMembers && chat.actualGroupMembers.filter(user => user.id !== currentUser.id);
 ┊159┊215┊          // Remove the current user from the chat admins
-┊160┊   ┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser.id);
-┊161┊   ┊          // Set the owner id to be null. A null owner means the group is read-only
-┊162┊   ┊          let ownerId: number | null = null;
-┊163┊   ┊
-┊164┊   ┊          // Check if there is any admin left
-┊165┊   ┊          if (adminIds!.length) {
-┊166┊   ┊            // Pick an admin as the new owner. The group is no longer read-only
-┊167┊   ┊            ownerId = chat.adminIds![0];
-┊168┊   ┊          }
+┊   ┊216┊          chat.admins = chat.admins && chat.admins.filter(user => user.id !== currentUser.id);
+┊   ┊217┊          // If there are no more admins left the group goes read only
+┊   ┊218┊          chat.owner = chat.admins && chat.admins[0] || null; // A null owner means the group is read-only
 ┊169┊219┊
-┊170┊   ┊          chats = chats.map(chat => {
-┊171┊   ┊            if (chat.id === Number(chatId)) {
-┊172┊   ┊              chat = {...chat, messages, listingMemberIds, actualGroupMemberIds, adminIds, ownerId};
-┊173┊   ┊            }
-┊174┊   ┊            return chat;
-┊175┊   ┊          });
+┊   ┊220┊          await connection.getRepository(Chat).save(chat);
 ┊176┊221┊        }
-┊177┊   ┊        return Number(chatId);
+┊   ┊222┊        return chatId;
 ┊178┊223┊      }
 ┊179┊224┊    },
-┊180┊   ┊    addMessage: (obj: any, {chatId, content}: AddMessageMutationArgs, {user: currentUser}: {user: User}): Message => {
+┊   ┊225┊    addMessage: async (obj: any, {chatId, content}: AddMessageMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<Message | null> => {
 ┊181┊226┊      if (content === null || content === '') {
 ┊182┊227┊        throw new Error(`Cannot add empty or null messages.`);
 ┊183┊228┊      }
 ┊184┊229┊
-┊185┊   ┊      let chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊230┊      let chat = await connection
+┊   ┊231┊        .createQueryBuilder(Chat, "chat")
+┊   ┊232┊        .whereInIds(chatId)
+┊   ┊233┊        .innerJoinAndSelect('chat.allTimeMembers', 'allTimeMembers')
+┊   ┊234┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊235┊        .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊236┊        .getOne();
 ┊186┊237┊
 ┊187┊238┊      if (!chat) {
 ┊188┊239┊        throw new Error(`Cannot find chat ${chatId}.`);
 ┊189┊240┊      }
 ┊190┊241┊
-┊191┊   ┊      let holderIds = chat.listingMemberIds;
+┊   ┊242┊      let holders: User[];
 ┊192┊243┊
 ┊193┊244┊      if (!chat.name) {
 ┊194┊245┊        // Chat
-┊195┊   ┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊246┊        if (!chat.listingMembers.map(user => user.id).includes(currentUser.id)) {
 ┊196┊247┊          throw new Error(`The chat ${chatId} must be listed for the current user before adding a message.`);
 ┊197┊248┊        }
 ┊198┊249┊
-┊199┊   ┊        const recipientId = chat.allTimeMemberIds.filter(userId => userId !== currentUser.id)[0];
+┊   ┊250┊        const recipientUser = chat.allTimeMembers.find(user => user.id !== currentUser.id);
 ┊200┊251┊
-┊201┊   ┊        if (!chat.listingMemberIds.find(listingId => listingId === recipientId)) {
-┊202┊   ┊          // Chat is not listed for the recipient. Add him to the listingMemberIds
-┊203┊   ┊          const listingMemberIds = chat.listingMemberIds.concat(recipientId);
+┊   ┊252┊        if (!recipientUser) {
+┊   ┊253┊          throw new Error(`Cannot find recipient user.`);
+┊   ┊254┊        }
 ┊204┊255┊
-┊205┊   ┊          chats = chats.map(chat => {
-┊206┊   ┊            if (chat.id === Number(chatId)) {
-┊207┊   ┊              chat = {...chat, listingMemberIds};
-┊208┊   ┊            }
-┊209┊   ┊            return chat;
-┊210┊   ┊          });
+┊   ┊256┊        if (!chat.listingMembers.find(user => user.id === recipientUser.id)) {
+┊   ┊257┊          // Chat is not listed for the recipient. Add him to the listingIds
+┊   ┊258┊          chat.listingMembers.push(recipientUser);
 ┊211┊259┊
-┊212┊   ┊          holderIds = listingMemberIds;
+┊   ┊260┊          await connection.getRepository(Chat).save(chat);
 ┊213┊261┊
 ┊214┊262┊          pubsub.publish('chatAdded', {
 ┊215┊263┊            creatorId: currentUser.id,
 ┊216┊264┊            chatAdded: chat,
 ┊217┊265┊          });
 ┊218┊266┊        }
+┊   ┊267┊
+┊   ┊268┊        holders = chat.listingMembers;
 ┊219┊269┊      } else {
 ┊220┊270┊        // Group
-┊221┊   ┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser.id)) {
+┊   ┊271┊        if (!chat.actualGroupMembers || !chat.actualGroupMembers.find(user => user.id === currentUser.id)) {
 ┊222┊272┊          throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
 ┊223┊273┊        }
 ┊224┊274┊
-┊225┊   ┊        holderIds = chat.actualGroupMemberIds!;
+┊   ┊275┊        holders = chat.actualGroupMembers;
 ┊226┊276┊      }
 ┊227┊277┊
-┊228┊   ┊      const id = (chat.messages.length && chat.messages[chat.messages.length - 1].id + 1) || 1;
-┊229┊   ┊
-┊230┊   ┊      let recipients: Recipient[] = [];
-┊231┊   ┊
-┊232┊   ┊      holderIds.forEach(holderId => {
-┊233┊   ┊        if (holderId !== currentUser.id) {
-┊234┊   ┊          recipients.push({
-┊235┊   ┊            userId: holderId,
-┊236┊   ┊            messageId: id,
-┊237┊   ┊            chatId: Number(chatId),
-┊238┊   ┊            receivedAt: null,
-┊239┊   ┊            readAt: null,
-┊240┊   ┊          });
-┊241┊   ┊        }
-┊242┊   ┊      });
-┊243┊   ┊
-┊244┊   ┊      const message: Message = {
-┊245┊   ┊        id,
-┊246┊   ┊        chatId: Number(chatId),
-┊247┊   ┊        senderId: currentUser.id,
+┊   ┊278┊      const message = await connection.getRepository(Message).save(new Message({
+┊   ┊279┊        chat: chat,
+┊   ┊280┊        sender: currentUser,
 ┊248┊281┊        content,
-┊249┊   ┊        createdAt: moment().unix(),
 ┊250┊282┊        type: MessageType.TEXT,
-┊251┊   ┊        recipients,
-┊252┊   ┊        holderIds,
-┊253┊   ┊      };
-┊254┊   ┊
-┊255┊   ┊      chats = chats.map(chat => {
-┊256┊   ┊        if (chat.id === Number(chatId)) {
-┊257┊   ┊          chat = {...chat, messages: chat.messages.concat(message)}
-┊258┊   ┊        }
-┊259┊   ┊        return chat;
-┊260┊   ┊      });
+┊   ┊283┊        holders,
+┊   ┊284┊        recipients: holders.reduce<Recipient[]>((filtered, user) => {
+┊   ┊285┊          if (user.id !== currentUser.id) {
+┊   ┊286┊            filtered.push(new Recipient({
+┊   ┊287┊              user,
+┊   ┊288┊            }));
+┊   ┊289┊          }
+┊   ┊290┊          return filtered;
+┊   ┊291┊        }, []),
+┊   ┊292┊      }));
 ┊261┊293┊
 ┊262┊294┊      pubsub.publish('messageAdded', {
 ┊263┊295┊        messageAdded: message,
 ┊264┊296┊      });
 ┊265┊297┊
-┊266┊   ┊      return message;
+┊   ┊298┊      return message || null;
 ┊267┊299┊    },
-┊268┊   ┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser}: {user: User}): number[] => {
-┊269┊   ┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊300┊    removeMessages: async (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }) => {
+┊   ┊301┊      const chat = await connection
+┊   ┊302┊        .createQueryBuilder(Chat, "chat")
+┊   ┊303┊        .whereInIds(chatId)
+┊   ┊304┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊305┊        .innerJoinAndSelect('chat.messages', 'messages')
+┊   ┊306┊        .innerJoinAndSelect('messages.holders', 'holders')
+┊   ┊307┊        .getOne();
 ┊270┊308┊
 ┊271┊309┊      if (!chat) {
 ┊272┊310┊        throw new Error(`Cannot find chat ${chatId}.`);
 ┊273┊311┊      }
 ┊274┊312┊
-┊275┊   ┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊313┊      if (!chat.listingMembers.find(user => user.id === currentUser.id)) {
 ┊276┊314┊        throw new Error(`The chat/group ${chatId} is not listed for the current user, so there is nothing to delete.`);
 ┊277┊315┊      }
 ┊278┊316┊
```
```diff
@@ -280,79 +318,166 @@
 ┊280┊318┊        throw new Error(`Cannot specify both 'all' and 'messageIds'.`);
 ┊281┊319┊      }
 ┊282┊320┊
+┊   ┊321┊      if (!all && !(messageIds && messageIds.length)) {
+┊   ┊322┊        throw new Error(`'all' and 'messageIds' cannot be both null`);
+┊   ┊323┊      }
+┊   ┊324┊
 ┊283┊325┊      let deletedIds: number[] = [];
-┊284┊   ┊      chats = chats.map(chat => {
-┊285┊   ┊        if (chat.id === Number(chatId)) {
-┊286┊   ┊          // Instead of chaining map and filter we can loop once using reduce
-┊287┊   ┊          const messages = chat.messages.reduce<Message[]>((filtered, message) => {
-┊288┊   ┊            if (all || messageIds!.includes(String(message.id))) {
-┊289┊   ┊              deletedIds.push(message.id);
-┊290┊   ┊              // Remove the current user from the message holders
-┊291┊   ┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
-┊292┊   ┊            }
+┊   ┊326┊      // Instead of chaining map and filter we can loop once using reduce
+┊   ┊327┊      chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊328┊        const filtered = await filtered$;
 ┊293┊329┊
-┊294┊   ┊            if (message.holderIds.length !== 0) {
-┊295┊   ┊              filtered.push(message);
-┊296┊   ┊            } // else discard the message
+┊   ┊330┊        if (all || messageIds!.includes(String(message.id))) {
+┊   ┊331┊          deletedIds.push(message.id);
+┊   ┊332┊          // Remove the current user from the message holders
+┊   ┊333┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
 ┊297┊334┊
-┊298┊   ┊            return filtered;
-┊299┊   ┊          }, []);
-┊300┊   ┊          chat = {...chat, messages};
 ┊301┊335┊        }
-┊302┊   ┊        return chat;
-┊303┊   ┊      });
+┊   ┊336┊
+┊   ┊337┊        if (message.holders.length !== 0) {
+┊   ┊338┊          // Remove the current user from the message holders
+┊   ┊339┊          await connection.getRepository(Message).save(message);
+┊   ┊340┊          filtered.push(message);
+┊   ┊341┊        } else {
+┊   ┊342┊          // Simply remove the message
+┊   ┊343┊          const recipients = await connection
+┊   ┊344┊            .createQueryBuilder(Recipient, "recipient")
+┊   ┊345┊            .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊346┊            .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊347┊            .getMany();
+┊   ┊348┊          for (let recipient of recipients) {
+┊   ┊349┊            await connection.getRepository(Recipient).remove(recipient);
+┊   ┊350┊          }
+┊   ┊351┊          await connection.getRepository(Message).remove(message);
+┊   ┊352┊        }
+┊   ┊353┊
+┊   ┊354┊        return filtered;
+┊   ┊355┊      }, Promise.resolve([]));
+┊   ┊356┊
+┊   ┊357┊      await connection.getRepository(Chat).save(chat);
+┊   ┊358┊
 ┊304┊359┊      return deletedIds;
 ┊305┊360┊    },
 ┊306┊361┊  },
 ┊307┊362┊  Subscription: {
 ┊308┊363┊    messageAdded: {
 ┊309┊364┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
-┊310┊   ┊        ({messageAdded}: {messageAdded: Message & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: User }) => {
+┊   ┊365┊        ({messageAdded}: {messageAdded: Message}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: User }) => {
 ┊311┊366┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
-┊312┊   ┊            !!messageAdded.recipients.find((recipient: Recipient) => recipient.userId === currentUser.id);
+┊   ┊367┊            !!messageAdded.recipients.find((recipient: Recipient) => recipient.user.id === currentUser.id);
 ┊313┊368┊        }),
 ┊314┊369┊    },
 ┊315┊370┊    chatAdded: {
 ┊316┊371┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
-┊317┊   ┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables: any, {user: currentUser}: { user: User }) => {
-┊318┊   ┊          return Number(creatorId) !== currentUser.id && !chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊372┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables, {user: currentUser}: { user: User }) => {
+┊   ┊373┊          return Number(creatorId) !== currentUser.id &&
+┊   ┊374┊            !!chatAdded.listingMembers.find((user: User) => user.id === currentUser.id);
 ┊319┊375┊        }),
 ┊320┊376┊    }
 ┊321┊377┊  },
 ┊322┊378┊  Chat: {
-┊323┊   ┊    name: (chat: Chat, args: any, {user: currentUser}: {user: User}): string => chat.name ? chat.name : users
-┊324┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
-┊325┊   ┊    picture: (chat: Chat, args: any, {user: currentUser}: {user: User}) => chat.name ? chat.picture : users
-┊326┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.picture,
-┊327┊   ┊    allTimeMembers: (chat: Chat): User[] => users.filter(user => chat.allTimeMemberIds.includes(user.id)),
-┊328┊   ┊    listingMembers: (chat: Chat): User[] => users.filter(user => chat.listingMemberIds.includes(user.id)),
-┊329┊   ┊    actualGroupMembers: (chat: Chat): User[] => users.filter(user => chat.actualGroupMemberIds && chat.actualGroupMemberIds.includes(user.id)),
-┊330┊   ┊    admins: (chat: Chat): User[] => users.filter(user => chat.adminIds && chat.adminIds.includes(user.id)),
-┊331┊   ┊    owner: (chat: Chat): User | null => users.find(user => chat.ownerId === user.id) || null,
-┊332┊   ┊    messages: (chat: Chat, {amount = 0}: {amount: number}, {user: currentUser}: {user: User}): Message[] => {
-┊333┊   ┊      const messages = chat.messages
-┊334┊   ┊      .filter(message => message.holderIds.includes(currentUser.id))
-┊335┊   ┊      .sort((a, b) => b.createdAt - a.createdAt) || <Message[]>[];
-┊336┊   ┊      return (amount ? messages.slice(0, amount) : messages).reverse();
+┊   ┊379┊    name: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<string | null> => {
+┊   ┊380┊      if (chat.name) {
+┊   ┊381┊        return chat.name;
+┊   ┊382┊      }
+┊   ┊383┊      const user = await connection
+┊   ┊384┊        .createQueryBuilder(User, "user")
+┊   ┊385┊        .where('user.id != :userId', {userId: currentUser.id})
+┊   ┊386┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊387┊        .getOne();
+┊   ┊388┊      return user && user.name || null;
+┊   ┊389┊    },
+┊   ┊390┊    picture: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<string | null> => {
+┊   ┊391┊      if (chat.name) {
+┊   ┊392┊        return chat.picture;
+┊   ┊393┊      }
+┊   ┊394┊      const user = await connection
+┊   ┊395┊        .createQueryBuilder(User, "user")
+┊   ┊396┊        .where('user.id != :userId', {userId: currentUser.id})
+┊   ┊397┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊398┊        .getOne();
+┊   ┊399┊      return user ? user.picture : null;
+┊   ┊400┊    },
+┊   ┊401┊    allTimeMembers: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊402┊      return await connection
+┊   ┊403┊        .createQueryBuilder(User, "user")
+┊   ┊404┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊405┊        .getMany();
+┊   ┊406┊    },
+┊   ┊407┊    listingMembers: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊408┊      return await connection
+┊   ┊409┊        .createQueryBuilder(User, "user")
+┊   ┊410┊        .innerJoin('user.listingMemberChats', 'listingMemberChats', 'listingMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊411┊        .getMany();
+┊   ┊412┊    },
+┊   ┊413┊    actualGroupMembers: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊414┊      return await connection
+┊   ┊415┊        .createQueryBuilder(User, "user")
+┊   ┊416┊        .innerJoin('user.actualGroupMemberChats', 'actualGroupMemberChats', 'actualGroupMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊417┊        .getMany();
+┊   ┊418┊    },
+┊   ┊419┊    admins: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊420┊      return await connection
+┊   ┊421┊        .createQueryBuilder(User, "user")
+┊   ┊422┊        .innerJoin('user.adminChats', 'adminChats', 'adminChats.id = :chatId', {chatId: chat.id})
+┊   ┊423┊        .getMany();
 ┊337┊424┊    },
-┊338┊   ┊    unreadMessages: (chat: Chat, args: any, {user: currentUser}: {user: User}): number => chat.messages
-┊339┊   ┊      .filter(message => message.holderIds.includes(currentUser.id) &&
-┊340┊   ┊        message.recipients.find(recipient => recipient.userId === currentUser.id && !recipient.readAt))
-┊341┊   ┊      .length,
-┊342┊   ┊    isGroup: (chat: Chat): boolean => !!chat.name,
+┊   ┊425┊    owner: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User | null> => {
+┊   ┊426┊      return await connection
+┊   ┊427┊        .createQueryBuilder(User, "user")
+┊   ┊428┊        .innerJoin('user.ownerChats', 'ownerChats', 'ownerChats.id = :chatId', {chatId: chat.id})
+┊   ┊429┊        .getOne() || null;
+┊   ┊430┊    },
+┊   ┊431┊    messages: async (chat: Chat, {amount = 0}: {amount: number}, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<Message[]> => {
+┊   ┊432┊      const query = connection
+┊   ┊433┊        .createQueryBuilder(Message, "message")
+┊   ┊434┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', {chatId: chat.id})
+┊   ┊435┊        .innerJoin('message.holders', 'holders', 'holders.id = :userId', {userId: currentUser.id})
+┊   ┊436┊        .orderBy({"message.createdAt": "DESC"});
+┊   ┊437┊      return (amount ? await query.take(amount).getMany() : await query.getMany()).reverse();
+┊   ┊438┊    },
+┊   ┊439┊    unreadMessages: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<number> => {
+┊   ┊440┊      return await connection
+┊   ┊441┊        .createQueryBuilder(Message, "message")
+┊   ┊442┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', {chatId: chat.id})
+┊   ┊443┊        .innerJoin('message.recipients', 'recipients', 'recipients.user.id = :userId AND recipients.readAt IS NULL', {userId: currentUser.id})
+┊   ┊444┊        .getCount();
+┊   ┊445┊    },
+┊   ┊446┊    isGroup: (chat: Chat) => !!chat.name,
 ┊343┊447┊  },
 ┊344┊448┊  Message: {
-┊345┊   ┊    chat: (message: Message): Chat | null => chats.find(chat => message.chatId === chat.id) || null,
-┊346┊   ┊    sender: (message: Message): User | null => users.find(user => user.id === message.senderId) || null,
-┊347┊   ┊    holders: (message: Message): User[] => users.filter(user => message.holderIds.includes(user.id)),
-┊348┊   ┊    ownership: (message: Message, args: any, {user: currentUser}: {user: User}): boolean => message.senderId === currentUser.id,
-┊349┊   ┊  },
-┊350┊   ┊  Recipient: {
-┊351┊   ┊    user: (recipient: Recipient): User | null => users.find(user => recipient.userId === user.id) || null,
-┊352┊   ┊    message: (recipient: Recipient): Message | null => {
-┊353┊   ┊      const chat = chats.find(chat => recipient.chatId === chat.id);
-┊354┊   ┊      return chat ? chat.messages.find(message => recipient.messageId === message.id) || null : null;
+┊   ┊449┊    sender: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User | null> => {
+┊   ┊450┊      return (await connection
+┊   ┊451┊        .createQueryBuilder(User, "user")
+┊   ┊452┊        .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {messageId: message.id})
+┊   ┊453┊        .getOne()) || null;
+┊   ┊454┊    },
+┊   ┊455┊    ownership: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<boolean> => {
+┊   ┊456┊      return !!(await connection
+┊   ┊457┊        .createQueryBuilder(User, "user")
+┊   ┊458┊        .whereInIds(currentUser.id)
+┊   ┊459┊        .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {messageId: message.id})
+┊   ┊460┊        .getCount());
+┊   ┊461┊    },
+┊   ┊462┊    recipients: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<Recipient[]> => {
+┊   ┊463┊      return await connection
+┊   ┊464┊        .createQueryBuilder(Recipient, "recipient")
+┊   ┊465┊        .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊466┊        .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊467┊        .innerJoinAndSelect('recipient.chat', 'chat')
+┊   ┊468┊        .getMany();
+┊   ┊469┊    },
+┊   ┊470┊    holders: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊471┊      return await connection
+┊   ┊472┊        .createQueryBuilder(User, "user")
+┊   ┊473┊        .innerJoin('user.holderMessages', 'holderMessages', 'holderMessages.id = :messageId', {messageId: message.id})
+┊   ┊474┊        .getMany();
+┊   ┊475┊    },
+┊   ┊476┊    chat: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<Chat | null> => {
+┊   ┊477┊      return (await connection
+┊   ┊478┊        .createQueryBuilder(Chat, "chat")
+┊   ┊479┊        .innerJoin('chat.messages', 'messages', 'messages.id = :messageId', {messageId: message.id})
+┊   ┊480┊        .getOne()) || null;
 ┊355┊481┊    },
-┊356┊   ┊    chat: (recipient: Recipient): Chat | null => chats.find(chat => recipient.chatId === chat.id) || null,
 ┊357┊482┊  },
 ┊358┊483┊};
```

[}]: #

`QueryBuilder` is one of the most powerful features of `TypeORM` - it allows you to build SQL queries using elegant and convenient syntax, execute them and get automatically transformed entities.

You can find more informations on `TypeORM` on http://typeorm.io

The best part is that you won't have to do anything on the client, everything will be completely transparent to it, even if migrated from NoSQL-like db structure to a relational one!
Of course, you could remove the custom normalization for the messages because now they have their own table and they are no longer embedded (so they have unique IDs), but we could leave it as well in order to be free to use any kind of backend.

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step15.md) |
|:----------------------|

[}]: #
