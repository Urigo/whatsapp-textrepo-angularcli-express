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
 ┊10┊11┊    "@types/body-parser": "1.17.0",
```
```diff
@@ -22,6 +23,7 @@
 ┊22┊23┊    "cors": "2.8.4",
 ┊23┊24┊    "express": "4.16.3",
 ┊24┊25┊    "graphql": "0.13.2",
+┊  ┊26┊    "graphql-code-generator": "0.9.1",
 ┊25┊27┊    "graphql-tools": "3.0.1",
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

##### Changed package.json
```diff
@@ -5,7 +5,7 @@
 ┊ 5┊ 5┊  "scripts": {
 ┊ 6┊ 6┊    "start": "npm run build:live",
 ┊ 7┊ 7┊    "build:live": "nodemon --exec ./node_modules/.bin/ts-node -- ./index.ts",
-┊ 8┊  ┊    "generator": "gql-gen --url http://localhost:3000/graphql --template ts --out ./types.d.ts"
+┊  ┊ 8┊    "generator": "gql-gen --schema http://localhost:3000/graphql --template ts --out ./types.d.ts"
 ┊ 9┊ 9┊  },
 ┊10┊10┊  "devDependencies": {
 ┊11┊11┊    "@types/body-parser": "1.17.0",
```
```diff
@@ -24,6 +24,7 @@
 ┊24┊24┊    "express": "4.16.3",
 ┊25┊25┊    "graphql": "0.13.2",
 ┊26┊26┊    "graphql-code-generator": "0.9.1",
+┊  ┊27┊    "graphql-codegen-typescript-template": "0.9.1",
 ┊27┊28┊    "graphql-tools": "3.0.1",
 ┊28┊29┊    "moment": "2.22.1"
 ┊29┊30┊  }
```

##### Added types.d.ts
```diff
@@ -0,0 +1,60 @@
+┊  ┊ 1┊/* tslint:disable */
+┊  ┊ 2┊
+┊  ┊ 3┊export interface Query {
+┊  ┊ 4┊  users?: User[] | null;
+┊  ┊ 5┊  chats?: Chat[] | null;
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
+┊  ┊17┊  id: string /** May be a chat or a group */;
+┊  ┊18┊  name?: string | null /** Computed for chats */;
+┊  ┊19┊  picture?: string | null /** Computed for chats */;
+┊  ┊20┊  allTimeMembers: User[] /** All members, current and past ones. */;
+┊  ┊21┊  listingMembers: User[] /** Whoever gets the chat listed. For groups includes past members who still didn't delete the group. */;
+┊  ┊22┊  actualGroupMembers: User[] /** Actual members of the group (they are not the only ones who get the group listed). Null for chats. */;
+┊  ┊23┊  admins?: User[] | null /** Null for chats */;
+┊  ┊24┊  owner?: User | null /** If null the group is read-only. Null for chats. */;
+┊  ┊25┊  messages: (Message | null)[];
+┊  ┊26┊  unreadMessages: number /** Computed property */;
+┊  ┊27┊  isGroup: boolean /** Computed property */;
+┊  ┊28┊}
+┊  ┊29┊
+┊  ┊30┊export interface Message {
+┊  ┊31┊  id: string;
+┊  ┊32┊  sender: User;
+┊  ┊33┊  chat: Chat;
+┊  ┊34┊  content: string;
+┊  ┊35┊  createdAt: string;
+┊  ┊36┊  type: number /** FIXME: should return MessageType */;
+┊  ┊37┊  recipients: Recipient[] /** Whoever received the message */;
+┊  ┊38┊  holders: User[] /** Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise */;
+┊  ┊39┊  ownership: boolean /** Computed property */;
+┊  ┊40┊}
+┊  ┊41┊
+┊  ┊42┊export interface Recipient {
+┊  ┊43┊  user: User;
+┊  ┊44┊  message: Message;
+┊  ┊45┊  chat: Chat;
+┊  ┊46┊  receivedAt?: string | null;
+┊  ┊47┊  readAt?: string | null;
+┊  ┊48┊}
+┊  ┊49┊export interface ChatQueryArgs {
+┊  ┊50┊  chatId: string;
+┊  ┊51┊}
+┊  ┊52┊export interface MessagesChatArgs {
+┊  ┊53┊  amount?: number | null;
+┊  ┊54┊}
+┊  ┊55┊
+┊  ┊56┊export enum MessageType {
+┊  ┊57┊  LOCATION = "LOCATION",
+┊  ┊58┊  TEXT = "TEXT",
+┊  ┊59┊  PICTURE = "PICTURE"
+┊  ┊60┊}
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
@@ -7,7 +7,8 @@
 ┊ 7┊ 7┊    "build": "ng build",
 ┊ 8┊ 8┊    "test": "ng test",
 ┊ 9┊ 9┊    "lint": "ng lint",
-┊10┊  ┊    "e2e": "ng e2e"
+┊  ┊10┊    "e2e": "ng e2e",
+┊  ┊11┊    "generator": "gql-gen --url http://localhost:3000/graphql --template ts --out ./src/types.d.ts \"./src/graphql/**/*.ts\""
 ┊11┊12┊  },
 ┊12┊13┊  "private": true,
 ┊13┊14┊  "dependencies": {
```
```diff
@@ -45,6 +46,7 @@
 ┊45┊46┊    "@types/jasminewd2": "2.0.3",
 ┊46┊47┊    "@types/node": "8.9.5",
 ┊47┊48┊    "codelyzer": "4.2.1",
+┊  ┊49┊    "graphql-code-generator": "0.9.1",
 ┊48┊50┊    "jasmine-core": "2.99.1",
 ┊49┊51┊    "jasmine-spec-reporter": "4.2.1",
 ┊50┊52┊    "karma": "1.7.1",
```

[}]: #

Those are our generated types:

[{]: <helper> (diffStep "2.2" module="client")

#### Step 2.2: Run generator

##### Changed package.json
```diff
@@ -8,7 +8,7 @@
 ┊ 8┊ 8┊    "test": "ng test",
 ┊ 9┊ 9┊    "lint": "ng lint",
 ┊10┊10┊    "e2e": "ng e2e",
-┊11┊  ┊    "generator": "gql-gen --url http://localhost:3000/graphql --template ts --out ./src/types.d.ts \"./src/graphql/**/*.ts\""
+┊  ┊11┊    "generator": "gql-gen --schema http://localhost:3000/graphql --template ts --out ./src/types.d.ts \"./src/graphql/**/*.ts\""
 ┊12┊12┊  },
 ┊13┊13┊  "private": true,
 ┊14┊14┊  "dependencies": {
```
```diff
@@ -47,6 +47,7 @@
 ┊47┊47┊    "@types/node": "8.9.5",
 ┊48┊48┊    "codelyzer": "4.2.1",
 ┊49┊49┊    "graphql-code-generator": "0.9.1",
+┊  ┊50┊    "graphql-codegen-typescript-template": "0.9.1",
 ┊50┊51┊    "jasmine-core": "2.99.1",
 ┊51┊52┊    "jasmine-spec-reporter": "4.2.1",
 ┊52┊53┊    "karma": "1.7.1",
```

##### Added src&#x2F;types.d.ts
```diff
@@ -0,0 +1,149 @@
+┊   ┊  1┊/* tslint:disable */
+┊   ┊  2┊
+┊   ┊  3┊export interface Query {
+┊   ┊  4┊  users?: User[] | null;
+┊   ┊  5┊  chats?: Chat[] | null;
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
+┊   ┊ 17┊  id: string /** May be a chat or a group */;
+┊   ┊ 18┊  name?: string | null /** Computed for chats */;
+┊   ┊ 19┊  picture?: string | null /** Computed for chats */;
+┊   ┊ 20┊  allTimeMembers: User[] /** All members, current and past ones. */;
+┊   ┊ 21┊  listingMembers: User[] /** Whoever gets the chat listed. For groups includes past members who still didn't delete the group. */;
+┊   ┊ 22┊  actualGroupMembers: User[] /** Actual members of the group (they are not the only ones who get the group listed). Null for chats. */;
+┊   ┊ 23┊  admins?: User[] | null /** Null for chats */;
+┊   ┊ 24┊  owner?: User | null /** If null the group is read-only. Null for chats. */;
+┊   ┊ 25┊  messages: (Message | null)[];
+┊   ┊ 26┊  unreadMessages: number /** Computed property */;
+┊   ┊ 27┊  isGroup: boolean /** Computed property */;
+┊   ┊ 28┊}
+┊   ┊ 29┊
+┊   ┊ 30┊export interface Message {
+┊   ┊ 31┊  id: string;
+┊   ┊ 32┊  sender: User;
+┊   ┊ 33┊  chat: Chat;
+┊   ┊ 34┊  content: string;
+┊   ┊ 35┊  createdAt: string;
+┊   ┊ 36┊  type: number /** FIXME: should return MessageType */;
+┊   ┊ 37┊  recipients: Recipient[] /** Whoever received the message */;
+┊   ┊ 38┊  holders: User[] /** Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise */;
+┊   ┊ 39┊  ownership: boolean /** Computed property */;
+┊   ┊ 40┊}
+┊   ┊ 41┊
+┊   ┊ 42┊export interface Recipient {
+┊   ┊ 43┊  user: User;
+┊   ┊ 44┊  message: Message;
+┊   ┊ 45┊  chat: Chat;
+┊   ┊ 46┊  receivedAt?: string | null;
+┊   ┊ 47┊  readAt?: string | null;
+┊   ┊ 48┊}
+┊   ┊ 49┊export interface ChatQueryArgs {
+┊   ┊ 50┊  chatId: string;
+┊   ┊ 51┊}
+┊   ┊ 52┊export interface MessagesChatArgs {
+┊   ┊ 53┊  amount?: number | null;
+┊   ┊ 54┊}
+┊   ┊ 55┊
+┊   ┊ 56┊export enum MessageType {
+┊   ┊ 57┊  LOCATION = "LOCATION",
+┊   ┊ 58┊  TEXT = "TEXT",
+┊   ┊ 59┊  PICTURE = "PICTURE"
+┊   ┊ 60┊}
+┊   ┊ 61┊export namespace GetChats {
+┊   ┊ 62┊  export type Variables = {
+┊   ┊ 63┊    amount?: number | null;
+┊   ┊ 64┊  };
+┊   ┊ 65┊
+┊   ┊ 66┊  export type Query = {
+┊   ┊ 67┊    __typename?: "Query";
+┊   ┊ 68┊    chats?: Chats[] | null;
+┊   ┊ 69┊  };
+┊   ┊ 70┊
+┊   ┊ 71┊  export type Chats = {
+┊   ┊ 72┊    __typename?: "Chat";
+┊   ┊ 73┊    messages: (Messages | null)[];
+┊   ┊ 74┊  } & ChatWithoutMessages.Fragment;
+┊   ┊ 75┊
+┊   ┊ 76┊  export type Messages = Message.Fragment;
+┊   ┊ 77┊}
+┊   ┊ 78┊
+┊   ┊ 79┊export namespace ChatWithoutMessages {
+┊   ┊ 80┊  export type Fragment = {
+┊   ┊ 81┊    __typename?: "Chat";
+┊   ┊ 82┊    id: string;
+┊   ┊ 83┊    name?: string | null;
+┊   ┊ 84┊    picture?: string | null;
+┊   ┊ 85┊    allTimeMembers: AllTimeMembers[];
+┊   ┊ 86┊    unreadMessages: number;
+┊   ┊ 87┊    isGroup: boolean;
+┊   ┊ 88┊  };
+┊   ┊ 89┊
+┊   ┊ 90┊  export type AllTimeMembers = {
+┊   ┊ 91┊    __typename?: "User";
+┊   ┊ 92┊    id: string;
+┊   ┊ 93┊  };
+┊   ┊ 94┊}
+┊   ┊ 95┊
+┊   ┊ 96┊export namespace Message {
+┊   ┊ 97┊  export type Fragment = {
+┊   ┊ 98┊    __typename?: "Message";
+┊   ┊ 99┊    id: string;
+┊   ┊100┊    chat: Chat;
+┊   ┊101┊    sender: Sender;
+┊   ┊102┊    content: string;
+┊   ┊103┊    createdAt: string;
+┊   ┊104┊    type: number;
+┊   ┊105┊    recipients: Recipients[];
+┊   ┊106┊    ownership: boolean;
+┊   ┊107┊  };
+┊   ┊108┊
+┊   ┊109┊  export type Chat = {
+┊   ┊110┊    __typename?: "Chat";
+┊   ┊111┊    id: string;
+┊   ┊112┊  };
+┊   ┊113┊
+┊   ┊114┊  export type Sender = {
+┊   ┊115┊    __typename?: "User";
+┊   ┊116┊    id: string;
+┊   ┊117┊    name?: string | null;
+┊   ┊118┊  };
+┊   ┊119┊
+┊   ┊120┊  export type Recipients = {
+┊   ┊121┊    __typename?: "Recipient";
+┊   ┊122┊    user: User;
+┊   ┊123┊    message: Message;
+┊   ┊124┊    chat: __Chat;
+┊   ┊125┊    receivedAt?: string | null;
+┊   ┊126┊    readAt?: string | null;
+┊   ┊127┊  };
+┊   ┊128┊
+┊   ┊129┊  export type User = {
+┊   ┊130┊    __typename?: "User";
+┊   ┊131┊    id: string;
+┊   ┊132┊  };
+┊   ┊133┊
+┊   ┊134┊  export type Message = {
+┊   ┊135┊    __typename?: "Message";
+┊   ┊136┊    id: string;
+┊   ┊137┊    chat: _Chat;
+┊   ┊138┊  };
+┊   ┊139┊
+┊   ┊140┊  export type _Chat = {
+┊   ┊141┊    __typename?: "Chat";
+┊   ┊142┊    id: string;
+┊   ┊143┊  };
+┊   ┊144┊
+┊   ┊145┊  export type __Chat = {
+┊   ┊146┊    __typename?: "Chat";
+┊   ┊147┊    id: string;
+┊   ┊148┊  };
+┊   ┊149┊}
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
 ┊3┊3┊import {Observable} from 'rxjs';
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
