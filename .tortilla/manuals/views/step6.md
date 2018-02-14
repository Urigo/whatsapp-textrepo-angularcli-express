# Step 6: graphql-code-generator

[//]: # (head-end)


## Server

The GraphQL codegen library can generate any code for any language — including type definitions, data models, query builder, resolvers, ORM code, complete full stack platforms.
You can create your own custom GraphQL codegen templates in 10 minutes, that fit exactly your needs. We will use it to generate `Typescript` typings.

First, let's install `graphql-code-generator`  in our server and add it to the run scripts:

    $ npm install graphql-code-generator

[{]: <helper> (diffStep "2.1" module="server")

#### Step 2.1: Install graphql-code-generator

##### Changed package.json
```diff
@@ -4,7 +4,8 @@
 ┊ 4┊ 4┊  "private": true,
 ┊ 5┊ 5┊  "scripts": {
 ┊ 6┊ 6┊    "start": "npm run build:live",
-┊ 7┊  ┊    "build:live": "nodemon --exec ./node_modules/.bin/ts-node -- ./index.ts"
+┊  ┊ 7┊    "build:live": "nodemon --exec ./node_modules/.bin/ts-node -- ./index.ts",
+┊  ┊ 8┊    "generator": "gql-gen --url http://localhost:3000/graphql --template ts --out ./types.d.ts"
 ┊ 8┊ 9┊  },
 ┊ 9┊10┊  "devDependencies": {
 ┊10┊11┊    "@types/body-parser": "1.16.8",
```
```diff
@@ -22,6 +23,7 @@
 ┊22┊23┊    "cors": "2.8.4",
 ┊23┊24┊    "express": "4.16.3",
 ┊24┊25┊    "graphql": "0.12.3",
+┊  ┊26┊    "graphql-code-generator": "0.8.21",
 ┊25┊27┊    "graphql-tools": "2.24.0",
 ┊26┊28┊    "moment": "2.22.1"
 ┊27┊29┊  }
```

[}]: #

Now let's run the generator (the server must be running in the background):

    $ npm run generator

Please note that the server must be started before running the generator.

Those are the types created with `npm run generator`:

[{]: <helper> (diffStep "2.2" module="server")

#### Step 2.2: Create types with generator

##### Added types.d.ts
```diff
@@ -0,0 +1,56 @@
+┊  ┊ 1┊/* tslint:disable */
+┊  ┊ 2┊
+┊  ┊ 3┊export interface Query {
+┊  ┊ 4┊  users: User[]; 
+┊  ┊ 5┊  chats: Chat[]; 
+┊  ┊ 6┊  chat?: Chat | null; 
+┊  ┊ 7┊}
+┊  ┊ 8┊
+┊  ┊ 9┊export interface User {
+┊  ┊10┊  id: string; 
+┊  ┊11┊  name?: string | null; 
+┊  ┊12┊  picture?: string | null; 
+┊  ┊13┊  phone?: string | null; 
+┊  ┊14┊}
+┊  ┊15┊
+┊  ┊16┊export interface Chat {
+┊  ┊17┊  id: string; /* May be a chat or a group */
+┊  ┊18┊  name?: string | null; /* Computed for chats */
+┊  ┊19┊  picture?: string | null; /* Computed for chats */
+┊  ┊20┊  allTimeMembers: User[]; /* All members, current and past ones. */
+┊  ┊21┊  listingMembers: User[]; /* Whoever gets the chat listed. For groups includes past members who still didn&#x27;t delete the group. */
+┊  ┊22┊  actualGroupMembers: User[]; /* Actual members of the group (they are not the only ones who get the group listed). Null for chats. */
+┊  ┊23┊  admins: User[]; /* Null for chats */
+┊  ┊24┊  owner?: User | null; /* If null the group is read-only. Null for chats. */
+┊  ┊25┊  messages: Message[]; 
+┊  ┊26┊  unreadMessages: number; /* Computed property */
+┊  ┊27┊  isGroup: boolean; /* Computed property */
+┊  ┊28┊}
+┊  ┊29┊
+┊  ┊30┊export interface Message {
+┊  ┊31┊  id: string; 
+┊  ┊32┊  sender: User; 
+┊  ┊33┊  chat: Chat; 
+┊  ┊34┊  content: string; 
+┊  ┊35┊  createdAt: string; 
+┊  ┊36┊  type: number; /* FIXME: should return MessageType */
+┊  ┊37┊  recipients: Recipient[]; /* Whoever received the message */
+┊  ┊38┊  holders: User[]; /* Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise */
+┊  ┊39┊  ownership: boolean; /* Computed property */
+┊  ┊40┊}
+┊  ┊41┊
+┊  ┊42┊export interface Recipient {
+┊  ┊43┊  user: User; 
+┊  ┊44┊  message: Message; 
+┊  ┊45┊  receivedAt?: string | null; 
+┊  ┊46┊  readAt?: string | null; 
+┊  ┊47┊}
+┊  ┊48┊export interface ChatQueryArgs {
+┊  ┊49┊  chatId: string; 
+┊  ┊50┊}
+┊  ┊51┊export interface MessagesChatArgs {
+┊  ┊52┊  amount?: number | null; 
+┊  ┊53┊}
+┊  ┊54┊
+┊  ┊55┊export type MessageType = "LOCATION" | "TEXT" | "PICTURE";
+┊  ┊56┊
```

[}]: #

Now let's use them:

[{]: <helper> (diffStep "2.3" module="server")

#### Step 2.3: Use our types

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,5 +1,6 @@
 ┊1┊1┊import { Chat, db, Message, Recipient, User } from "../db";
 ┊2┊2┊import { IResolvers } from "graphql-tools/dist/Interfaces";
+┊ ┊3┊import { ChatQueryArgs } from "../types";
 ┊3┊4┊
 ┊4┊5┊let users = db.users;
 ┊5┊6┊let chats = db.chats;
```
```diff
@@ -10,7 +11,7 @@
 ┊10┊11┊    // Show all users for the moment.
 ┊11┊12┊    users: (): User[] => users.filter(user => user.id !== currentUser),
 ┊12┊13┊    chats: (): Chat[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser)),
-┊13┊  ┊    chat: (obj: any, {chatId}): Chat | null => chats.find(chat => chat.id === chatId) || null,
+┊  ┊14┊    chat: (obj: any, {chatId}: ChatQueryArgs): Chat | null => chats.find(chat => chat.id === Number(chatId)) || null,
 ┊14┊15┊  },
 ┊15┊16┊  Chat: {
 ┊16┊17┊    name: (chat: Chat): string => chat.name ? chat.name : users
```

[}]: #

Don't worry, they will be much more useful when we will write our first mutation.

## Client

Let's do the same on the client:

    $ npm install graphql-code-generator

Please note that the server must be started before running the generator.

[{]: <helper> (diffStep "2.1" module="client")

#### Step 2.1: Install graphql-code-generator

##### Changed package.json
```diff
@@ -8,7 +8,8 @@
 ┊ 8┊ 8┊    "build": "ng build --prod",
 ┊ 9┊ 9┊    "test": "ng test",
 ┊10┊10┊    "lint": "ng lint",
-┊11┊  ┊    "e2e": "ng e2e"
+┊  ┊11┊    "e2e": "ng e2e",
+┊  ┊12┊    "generator": "gql-gen --url http://localhost:3000/graphql --template ts --out ./src/types.d.ts \"./src/graphql/**/*.ts\""
 ┊12┊13┊  },
 ┊13┊14┊  "private": true,
 ┊14┊15┊  "dependencies": {
```
```diff
@@ -46,6 +47,7 @@
 ┊46┊47┊    "@types/jasminewd2": "2.0.3",
 ┊47┊48┊    "@types/node": "6.0.101",
 ┊48┊49┊    "codelyzer": "4.1.0",
+┊  ┊50┊    "graphql-code-generator": "0.8.14",
 ┊49┊51┊    "jasmine-core": "2.8.0",
 ┊50┊52┊    "jasmine-spec-reporter": "4.2.1",
 ┊51┊53┊    "karma": "2.0.0",
```

[}]: #

Those are our generated types:

[{]: <helper> (diffStep "2.2" module="client")

#### Step 2.2: Run generator

##### Added src&#x2F;types.d.ts
```diff
@@ -0,0 +1,118 @@
+┊   ┊  1┊/* tslint:disable */
+┊   ┊  2┊
+┊   ┊  3┊export interface Query {
+┊   ┊  4┊  users: User[]; 
+┊   ┊  5┊  chats: Chat[]; 
+┊   ┊  6┊  chat?: Chat | null; 
+┊   ┊  7┊}
+┊   ┊  8┊
+┊   ┊  9┊export interface User {
+┊   ┊ 10┊  id: string; 
+┊   ┊ 11┊  name?: string | null; 
+┊   ┊ 12┊  picture?: string | null; 
+┊   ┊ 13┊  phone?: string | null; 
+┊   ┊ 14┊}
+┊   ┊ 15┊
+┊   ┊ 16┊export interface Chat {
+┊   ┊ 17┊  id: string; /* May be a chat or a group */
+┊   ┊ 18┊  name?: string | null; /* Computed for chats */
+┊   ┊ 19┊  picture?: string | null; /* Computed for chats */
+┊   ┊ 20┊  allTimeMembers: User[]; /* All members, current and past ones. */
+┊   ┊ 21┊  listingMembers: User[]; /* Whoever gets the chat listed. For groups includes past members who still didn&#x27;t delete the group. */
+┊   ┊ 22┊  actualGroupMembers: User[]; /* Actual members of the group (they are not the only ones who get the group listed). Null for chats. */
+┊   ┊ 23┊  admins: User[]; /* Null for chats */
+┊   ┊ 24┊  owner?: User | null; /* If null the group is read-only. Null for chats. */
+┊   ┊ 25┊  messages: Message[]; 
+┊   ┊ 26┊  unreadMessages: number; /* Computed property */
+┊   ┊ 27┊  isGroup: boolean; /* Computed property */
+┊   ┊ 28┊}
+┊   ┊ 29┊
+┊   ┊ 30┊export interface Message {
+┊   ┊ 31┊  id: string; 
+┊   ┊ 32┊  sender: User; 
+┊   ┊ 33┊  chat: Chat; 
+┊   ┊ 34┊  content: string; 
+┊   ┊ 35┊  createdAt: string; 
+┊   ┊ 36┊  type: number; /* FIXME: should return MessageType */
+┊   ┊ 37┊  recipients: Recipient[]; /* Whoever received the message */
+┊   ┊ 38┊  holders: User[]; /* Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise */
+┊   ┊ 39┊  ownership: boolean; /* Computed property */
+┊   ┊ 40┊}
+┊   ┊ 41┊
+┊   ┊ 42┊export interface Recipient {
+┊   ┊ 43┊  user: User; 
+┊   ┊ 44┊  message: Message; 
+┊   ┊ 45┊  receivedAt?: string | null; 
+┊   ┊ 46┊  readAt?: string | null; 
+┊   ┊ 47┊}
+┊   ┊ 48┊export interface ChatQueryArgs {
+┊   ┊ 49┊  chatId: string; 
+┊   ┊ 50┊}
+┊   ┊ 51┊export interface MessagesChatArgs {
+┊   ┊ 52┊  amount?: number | null; 
+┊   ┊ 53┊}
+┊   ┊ 54┊
+┊   ┊ 55┊export type MessageType = "LOCATION" | "TEXT" | "PICTURE";
+┊   ┊ 56┊
+┊   ┊ 57┊export namespace GetChats {
+┊   ┊ 58┊  export type Variables = {
+┊   ┊ 59┊    amount?: number | null;
+┊   ┊ 60┊  }
+┊   ┊ 61┊
+┊   ┊ 62┊  export type Query = {
+┊   ┊ 63┊    chats: Chats[]; 
+┊   ┊ 64┊  } 
+┊   ┊ 65┊
+┊   ┊ 66┊  export type Chats = {
+┊   ┊ 67┊    messages: Messages[]; 
+┊   ┊ 68┊  } & ChatWithoutMessages.Fragment
+┊   ┊ 69┊
+┊   ┊ 70┊  export type Messages = Message.Fragment
+┊   ┊ 71┊}
+┊   ┊ 72┊
+┊   ┊ 73┊export namespace ChatWithoutMessages {
+┊   ┊ 74┊  export type Fragment = {
+┊   ┊ 75┊    id: string; 
+┊   ┊ 76┊    name?: string | null; 
+┊   ┊ 77┊    picture?: string | null; 
+┊   ┊ 78┊    allTimeMembers: AllTimeMembers[]; 
+┊   ┊ 79┊    unreadMessages: number; 
+┊   ┊ 80┊    isGroup: boolean; 
+┊   ┊ 81┊  } 
+┊   ┊ 82┊
+┊   ┊ 83┊  export type AllTimeMembers = {
+┊   ┊ 84┊    id: string; 
+┊   ┊ 85┊  } 
+┊   ┊ 86┊}
+┊   ┊ 87┊
+┊   ┊ 88┊export namespace Message {
+┊   ┊ 89┊  export type Fragment = {
+┊   ┊ 90┊    id: string; 
+┊   ┊ 91┊    sender: Sender; 
+┊   ┊ 92┊    content: string; 
+┊   ┊ 93┊    createdAt: string; 
+┊   ┊ 94┊    type: number; 
+┊   ┊ 95┊    recipients: Recipients[]; 
+┊   ┊ 96┊    ownership: boolean; 
+┊   ┊ 97┊  } 
+┊   ┊ 98┊
+┊   ┊ 99┊  export type Sender = {
+┊   ┊100┊    id: string; 
+┊   ┊101┊    name?: string | null; 
+┊   ┊102┊  } 
+┊   ┊103┊
+┊   ┊104┊  export type Recipients = {
+┊   ┊105┊    user: User; 
+┊   ┊106┊    message: Message; 
+┊   ┊107┊    receivedAt?: string | null; 
+┊   ┊108┊    readAt?: string | null; 
+┊   ┊109┊  } 
+┊   ┊110┊
+┊   ┊111┊  export type User = {
+┊   ┊112┊    id: string; 
+┊   ┊113┊  } 
+┊   ┊114┊
+┊   ┊115┊  export type Message = {
+┊   ┊116┊    id: string; 
+┊   ┊117┊  } 
+┊   ┊118┊}
```

[}]: #

Let's use them:

[{]: <helper> (diffStep "2.3" module="client")

#### Step 2.3: Use the generated types

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -1,4 +1,5 @@
 ┊1┊1┊import {Component, Input} from '@angular/core';
+┊ ┊2┊import {GetChats} from '../../../../types';
 ┊2┊3┊
 ┊3┊4┊@Component({
 ┊4┊5┊  selector: 'app-chat-item',
```
```diff
@@ -16,5 +17,5 @@
 ┊16┊17┊export class ChatItemComponent {
 ┊17┊18┊  // tslint:disable-next-line:no-input-rename
 ┊18┊19┊  @Input('item')
-┊19┊  ┊  chat: any;
+┊  ┊20┊  chat: GetChats.Chats;
 ┊20┊21┊}
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chats-list&#x2F;chats-list.component.ts
```diff
@@ -1,4 +1,5 @@
 ┊1┊1┊import {Component, Input} from '@angular/core';
+┊ ┊2┊import {GetChats} from '../../../../types';
 ┊2┊3┊
 ┊3┊4┊@Component({
 ┊4┊5┊  selector: 'app-chats-list',
```
```diff
@@ -14,7 +15,7 @@
 ┊14┊15┊export class ChatsListComponent {
 ┊15┊16┊  // tslint:disable-next-line:no-input-rename
 ┊16┊17┊  @Input('items')
-┊17┊  ┊  chats: any[];
+┊  ┊18┊  chats: GetChats.Chats[];
 ┊18┊19┊
 ┊19┊20┊  constructor() {}
 ┊20┊21┊}
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.ts
```diff
@@ -1,6 +1,7 @@
 ┊1┊1┊import {Component, OnInit} from '@angular/core';
 ┊2┊2┊import {ChatsService} from '../../../services/chats.service';
 ┊3┊3┊import {Observable} from 'rxjs/Observable';
+┊ ┊4┊import {GetChats} from '../../../../types';
 ┊4┊5┊
 ┊5┊6┊@Component({
 ┊6┊7┊  template: `
```
```diff
@@ -35,7 +36,7 @@
 ┊35┊36┊  styleUrls: ['./chats.component.scss'],
 ┊36┊37┊})
 ┊37┊38┊export class ChatsComponent implements OnInit {
-┊38┊  ┊  chats$: Observable<any[]>;
+┊  ┊39┊  chats$: Observable<GetChats.Chats[]>;
 ┊39┊40┊
 ┊40┊41┊  constructor(private chatsService: ChatsService) {
 ┊41┊42┊  }
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -3,6 +3,7 @@
 ┊3┊3┊import {Apollo} from 'apollo-angular';
 ┊4┊4┊import {Injectable} from '@angular/core';
 ┊5┊5┊import {getChatsQuery} from '../../graphql/getChats.query';
+┊ ┊6┊import {GetChats} from '../../types';
 ┊6┊7┊
 ┊7┊8┊@Injectable()
 ┊8┊9┊export class ChatsService {
```
```diff
@@ -11,14 +12,14 @@
 ┊11┊12┊  constructor(private apollo: Apollo) {}
 ┊12┊13┊
 ┊13┊14┊  getChats() {
-┊14┊  ┊    const query = this.apollo.watchQuery<any>(<WatchQueryOptions>{
+┊  ┊15┊    const query = this.apollo.watchQuery<GetChats.Query>(<WatchQueryOptions>{
 ┊15┊16┊      query: getChatsQuery,
 ┊16┊17┊      variables: {
 ┊17┊18┊        amount: this.messagesAmount,
 ┊18┊19┊      },
 ┊19┊20┊    });
 ┊20┊21┊    const chats$ = query.valueChanges.pipe(
-┊21┊  ┊      map((result: ApolloQueryResult<any>) => result.data.chats)
+┊  ┊22┊      map((result: ApolloQueryResult<GetChats.Query>) => result.data.chats)
 ┊22┊23┊    );
 ┊23┊24┊
 ┊24┊25┊    return {query, chats$};
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step5.md) | [Next Step >](step7.md) |
|:--------------------------------|--------------------------------:|

[}]: #
