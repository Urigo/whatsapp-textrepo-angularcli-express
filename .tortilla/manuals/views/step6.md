# Step 6: graphql-code-generator

[//]: # (head-end)


## Code Generation

GraphQL entities are defined as static and typed, which means they can be analyzed and use as a base for generating everything.

There are many tools related to that topic, but we're going to focus on The **GraphQL Coge Generator**.

The GraphQL Coge Generator can generate any code for any language  —  including type definitions, data models, query builder, resolvers, ORM code, complete full stack platforms.

The tool is based on the concept of templates. It has few official templates to solve most required features. The GraphQL Code Gen allows to create your own custom codegen templates in 10 minutes, that fit exactly your needs.

We will use TypeScript template to generate typings.

### Generating types for server-side code

First, let's install `graphql-code-generator` with the TypeScript template and add it to the run scripts:

    yarn add graphql-code-generator graphql-codegen-typescript-template

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
+┊  ┊ 8┊    "generator": "gql-gen -r ts-node/register —schema schema/typeDefs.ts --template ts --out ./types.d.ts"
 ┊ 8┊ 9┊  },
 ┊ 9┊10┊  "devDependencies": {
 ┊10┊11┊    "@types/body-parser": "1.17.0",
```
```diff
@@ -22,6 +23,7 @@
 ┊22┊23┊    "cors": "2.8.4",
 ┊23┊24┊    "express": "4.16.3",
 ┊24┊25┊    "graphql": "14.0.2",
+┊  ┊26┊    "graphql-code-generator": "0.12.6",
 ┊25┊27┊    "moment": "2.22.1"
 ┊26┊28┊  }
 ┊27┊29┊}
```

[}]: #

Few things here:

- `--require ts-node/register` - makes the ts-node compile TypeScript files
- `--schema` - points to a file that exports the GraphQL Schema object or a string (it also accepts an url)
- `--template` - tells about the template we want to use
- `--out` - the location of a generated file

Now let's run the generator:

    yarn generator

Please note that the server doesn't have to be running in background because we import schema through a file.

Next, let's use the generated types:

[{]: <helper> (diffStep "2.3" module="server")

#### Step 2.3: Use our types

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,5 +1,6 @@
 ┊1┊1┊import { IResolvers } from 'apollo-server-express';
 ┊2┊2┊import { Chat, db, Message, Recipient, User } from "../db";
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

### Generating types for client-side code

Let's do the same on the client:

    yarn add graphql-code-generator graphql-codegen-typescript-template

Please note that in this case, the server must be started before running the generator.

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
+┊  ┊11┊    "generator": "gql-gen --schema http://localhost:3000/graphql --template graphql-codegen-typescript-template --out ./src/graphql.ts ./src/graphql/**/*.ts"
 ┊11┊12┊  },
 ┊12┊13┊  "private": true,
 ┊13┊14┊  "dependencies": {
```
```diff
@@ -45,6 +46,7 @@
 ┊45┊46┊    "@types/jasminewd2": "2.0.3",
 ┊46┊47┊    "@types/node": "8.9.5",
 ┊47┊48┊    "codelyzer": "4.2.1",
+┊  ┊49┊    "graphql-code-generator": "0.12.6",
 ┊48┊50┊    "jasmine-core": "2.99.1",
 ┊49┊51┊    "jasmine-spec-reporter": "4.2.1",
 ┊50┊52┊    "karma": "1.7.1",
```

[}]: #

We saw how to use them on the server but let's see how easy it is to take advantage of them in Apollo.

First thing you should know is most of the methods of Apollo service accepts generic types.

#### What are Generic Types

A major part of software engineering is building components that not only have well-defined and consistent APIs, but are also reusable. Components that are capable of working on the data of today as well as the data of tomorrow will give you the most flexible capabilities for building up large software systems.

We like to work on an code samples so let's do some functional programming and create the `map` method that accepts a transform function and return a new result.

Without generics, we would have to use a specific type or `any`:

```typescript
function map(mapFn: (value: any) => any) {
  return function(source: any): any {
    return mapFn(source);
  };
}
```

Imagine, we want to use `map` function to pick user's id from an object:

```typescript
interface User {
  id: number;
}
const me = {
  id: 1,
};

const pickUserId = map(user => user.id);

console.log(pickUserId(me)); // outputs: 1
```

Great, it works, we get the expected result but it could be done way better.

What is the main issue here?
It accepts an object of any type and get `any` in return. It's not very useful because we know it return a `number` and receive a `User` but TypeScript doesn't know that and later on, in other part of the code, we end up with no information about the result.

That's where Generic Types jumps in!

With generics, it could look like this:

```typescript
function map<T, R>(mapFn: (value: T) => R) {
  return function(source: T): R {
    return mapFn(source);
  };
}
```

There's... a lot! So let's break it:

- `map<T, R>` - the map function accepts two generic types
- `(mapFn: (value: T) => R)` - the only argument is a function that accepts an object of type `T` and returns value of type `R`
- `function (source: T): R => {...}` - it's a higher order function that accepts a source of type `T` and transforms to be `R`.

Back to our example, now with generic types:

```typescript
interface User {
  id: number;
}
const me = {
  id: 1,
};

const pickUserId = map<User, number>(user => user.id);

console.log(pickUserId(me)); // outputs: 1
```

Our `map` and `pickUserId` function are strongly typed now so when you try to provide an object that has no `id` field or the field is a `string` you'll receive a proper information from TypeScript (and your IDE).

#### Make client-side code strongly typed

With all that knowledge let's use how to use those generated typed with `Apollo` service:

[{]: <helper> (diffStep "2.3" module="client")

#### Step 2.3: Use the generated types

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -1,4 +1,5 @@
 ┊1┊1┊import {Component, Input} from '@angular/core';
+┊ ┊2┊import {GetChats} from '../../../../graphql';
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
+┊ ┊2┊import {GetChats} from '../../../../graphql';
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
+┊ ┊4┊import {GetChats} from '../../../../graphql';
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
@@ -2,6 +2,7 @@
 ┊2┊2┊import {Apollo} from 'apollo-angular';
 ┊3┊3┊import {Injectable} from '@angular/core';
 ┊4┊4┊import {getChatsQuery} from '../../graphql/getChats.query';
+┊ ┊5┊import {GetChats} from '../../graphql';
 ┊5┊6┊
 ┊6┊7┊@Injectable()
 ┊7┊8┊export class ChatsService {
```
```diff
@@ -10,7 +11,7 @@
 ┊10┊11┊  constructor(private apollo: Apollo) {}
 ┊11┊12┊
 ┊12┊13┊  getChats() {
-┊13┊  ┊    const query = this.apollo.watchQuery<any>({
+┊  ┊14┊    const query = this.apollo.watchQuery<GetChats.Query>({
 ┊14┊15┊      query: getChatsQuery,
 ┊15┊16┊      variables: {
 ┊16┊17┊        amount: this.messagesAmount,
```

[}]: #

As you can probably tell by now, methods like `watchQuery` and `mutate` (and others) accepts two generic types, first one describes the result and the second one the variables.

```typescript
export class ChatService {
  constructor(private apollo: Apollo) {}

  getChat(chatId: string, amount: number): GetChat.Chat {
    return this.apollo
      .watchQuery<GetChat.Query, GetChat.Variables>({
        query: getChatQuery,
        variables: {
          chatId,
          amount,
        },
      })
      .pipe(map(result => result.data.chat));
  }
}
```

Few things here.

- An object of `GetChat.Query` shape lives under the `result.data` because `watchQuery` returns an object of type `ApolloQueryResult<T>` where `T` is our `GetChat.Query`.

```typescript
export type ApolloQueryResult<T> = {
  data: T;
  // ... other fields
};
```

You get the same result with Mutation or Subscription.

- Thanks to the second argument and `GetChat.Variables` we as well as TypeScript (and an IDE) know that `chatId` is a string and `amount` accepts only a number. Whenever we try to pass a value of different kind an Error will pop out.

### Take full advantage of Codegen and generate ready to use Services!

I've got a fantasctic news, you don't have to manually provide those generated types to each type Apollo service is used. Because GraphQL Documents are statically analyzable, we prepared a codegen template specific for Apollo Angular users.

First, let's install `graphql-codegen-apollo-angular` with the Apollo Angular templatea and update the template param of `generator` script

    yarn add graphql-code-generator graphql-codegen-typescript-template

[{]: <helper> (diffStep "2.4" file="package.json" module="client")

#### Step 2.4: Use auto-generated GQL services

##### Changed package.json
```diff
@@ -8,7 +8,7 @@
 ┊ 8┊ 8┊    "test": "ng test",
 ┊ 9┊ 9┊    "lint": "ng lint",
 ┊10┊10┊    "e2e": "ng e2e",
-┊11┊  ┊    "generator": "gql-gen --schema http://localhost:3000/graphql --template graphql-codegen-typescript-template --out ./src/graphql.ts ./src/graphql/**/*.ts"
+┊  ┊11┊    "generator": "gql-gen --schema http://localhost:3000/graphql --template graphql-codegen-apollo-angular-template --out ./src/graphql.ts ./src/graphql/**/*.ts"
 ┊12┊12┊  },
 ┊13┊13┊  "private": true,
 ┊14┊14┊  "dependencies": {
```
```diff
@@ -48,6 +48,7 @@
 ┊48┊48┊    "codelyzer": "4.2.1",
 ┊49┊49┊    "graphql-code-generator": "0.12.6",
 ┊50┊50┊    "graphql-codegen-typescript-template": "0.12.6",
+┊  ┊51┊    "graphql-codegen-apollo-angular-template": "0.12.6",
 ┊51┊52┊    "jasmine-core": "2.99.1",
 ┊52┊53┊    "jasmine-spec-reporter": "4.2.1",
 ┊53┊54┊    "karma": "1.7.1",
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,21 +1,18 @@
 ┊ 1┊ 1┊import {map} from 'rxjs/operators';
-┊ 2┊  ┊import {Apollo} from 'apollo-angular';
 ┊ 3┊ 2┊import {Injectable} from '@angular/core';
-┊ 4┊  ┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊ 5┊  ┊import {GetChats} from '../../graphql';
+┊  ┊ 3┊import {GetChatsGQL} from '../../graphql';
 ┊ 6┊ 4┊
 ┊ 7┊ 5┊@Injectable()
 ┊ 8┊ 6┊export class ChatsService {
 ┊ 9┊ 7┊  messagesAmount = 3;
 ┊10┊ 8┊
-┊11┊  ┊  constructor(private apollo: Apollo) {}
+┊  ┊ 9┊  constructor(
+┊  ┊10┊    private getChatsGQL: GetChatsGQL
+┊  ┊11┊  ) {}
 ┊12┊12┊
 ┊13┊13┊  getChats() {
-┊14┊  ┊    const query = this.apollo.watchQuery<GetChats.Query>({
-┊15┊  ┊      query: getChatsQuery,
-┊16┊  ┊      variables: {
-┊17┊  ┊        amount: this.messagesAmount,
-┊18┊  ┊      },
+┊  ┊14┊    const query = this.getChatsGQL.watch({
+┊  ┊15┊      amount: this.messagesAmount,
 ┊19┊16┊    });
 ┊20┊17┊    const chats$ = query.valueChanges.pipe(
 ┊21┊18┊      map((result) => result.data.chats)
```

##### Changed src&#x2F;graphql.ts
```diff
@@ -28,6 +28,8 @@
 ┊28┊28┊  ): R | Result | Promise<R | Result>;
 ┊29┊29┊};
 ┊30┊30┊
+┊  ┊31┊export type Date = any;
+┊  ┊32┊
 ┊31┊33┊export interface Query {
 ┊32┊34┊  users?: User[] | null;
 ┊33┊35┊  chats?: Chat[] | null;
```
```diff
@@ -42,17 +44,18 @@
 ┊42┊44┊}
 ┊43┊45┊
 ┊44┊46┊export interface Chat {
-┊45┊  ┊  id: string /** May be a chat or a group */;
-┊46┊  ┊  name?: string | null /** Computed for chats */;
-┊47┊  ┊  picture?: string | null /** Computed for chats */;
-┊48┊  ┊  allTimeMembers: User[] /** All members, current and past ones. */;
-┊49┊  ┊  listingMembers: User[] /** Whoever gets the chat listed. For groups includes past members who still didn't delete the group. */;
-┊50┊  ┊  actualGroupMembers: User[] /** Actual members of the group (they are not the only ones who get the group listed). Null for chats. */;
-┊51┊  ┊  admins?: User[] | null /** Null for chats */;
-┊52┊  ┊  owner?: User | null /** If null the group is read-only. Null for chats. */;
+┊  ┊47┊  id: string;
+┊  ┊48┊  name?: string | null;
+┊  ┊49┊  picture?: string | null;
+┊  ┊50┊  allTimeMembers: User[];
+┊  ┊51┊  listingMembers: User[];
+┊  ┊52┊  actualGroupMembers: User[];
+┊  ┊53┊  admins?: User[] | null;
+┊  ┊54┊  owner?: User | null;
 ┊53┊55┊  messages: (Message | null)[];
-┊54┊  ┊  unreadMessages: number /** Computed property */;
-┊55┊  ┊  isGroup: boolean /** Computed property */;
+┊  ┊56┊  messageFeed?: MessageFeed | null;
+┊  ┊57┊  unreadMessages: number;
+┊  ┊58┊  isGroup: boolean;
 ┊56┊59┊}
 ┊57┊60┊
 ┊58┊61┊export interface Message {
```
```diff
@@ -60,25 +63,107 @@
 ┊ 60┊ 63┊  sender: User;
 ┊ 61┊ 64┊  chat: Chat;
 ┊ 62┊ 65┊  content: string;
-┊ 63┊   ┊  createdAt: string;
-┊ 64┊   ┊  type: number /** FIXME: should return MessageType */;
-┊ 65┊   ┊  recipients: Recipient[] /** Whoever received the message */;
-┊ 66┊   ┊  holders: User[] /** Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise */;
-┊ 67┊   ┊  ownership: boolean /** Computed property */;
+┊   ┊ 66┊  createdAt: Date;
+┊   ┊ 67┊  type: number;
+┊   ┊ 68┊  recipients: Recipient[];
+┊   ┊ 69┊  holders: User[];
+┊   ┊ 70┊  ownership: boolean;
 ┊ 68┊ 71┊}
 ┊ 69┊ 72┊
 ┊ 70┊ 73┊export interface Recipient {
 ┊ 71┊ 74┊  user: User;
 ┊ 72┊ 75┊  message: Message;
 ┊ 73┊ 76┊  chat: Chat;
-┊ 74┊   ┊  receivedAt?: string | null;
-┊ 75┊   ┊  readAt?: string | null;
+┊   ┊ 77┊  receivedAt?: Date | null;
+┊   ┊ 78┊  readAt?: Date | null;
+┊   ┊ 79┊}
+┊   ┊ 80┊
+┊   ┊ 81┊export interface MessageFeed {
+┊   ┊ 82┊  hasNextPage: boolean;
+┊   ┊ 83┊  cursor?: string | null;
+┊   ┊ 84┊  messages: (Message | null)[];
+┊   ┊ 85┊}
+┊   ┊ 86┊
+┊   ┊ 87┊export interface Mutation {
+┊   ┊ 88┊  addChat?: Chat | null;
+┊   ┊ 89┊  addGroup?: Chat | null;
+┊   ┊ 90┊  removeChat?: string | null;
+┊   ┊ 91┊  addMessage?: Message | null;
+┊   ┊ 92┊  removeMessages?: (string | null)[] | null;
+┊   ┊ 93┊  addMembers?: (string | null)[] | null;
+┊   ┊ 94┊  removeMembers?: (string | null)[] | null;
+┊   ┊ 95┊  addAdmins?: (string | null)[] | null;
+┊   ┊ 96┊  removeAdmins?: (string | null)[] | null;
+┊   ┊ 97┊  setGroupName?: string | null;
+┊   ┊ 98┊  setGroupPicture?: string | null;
+┊   ┊ 99┊  markAsReceived?: boolean | null;
+┊   ┊100┊  markAsRead?: boolean | null;
+┊   ┊101┊}
+┊   ┊102┊
+┊   ┊103┊export interface Subscription {
+┊   ┊104┊  messageAdded?: Message | null;
+┊   ┊105┊  chatAdded?: Chat | null;
 ┊ 76┊106┊}
 ┊ 77┊107┊export interface ChatQueryArgs {
 ┊ 78┊108┊  chatId: string;
 ┊ 79┊109┊}
 ┊ 80┊110┊export interface MessagesChatArgs {
 ┊ 81┊111┊  amount?: number | null;
+┊   ┊112┊  before?: string | null;
+┊   ┊113┊}
+┊   ┊114┊export interface MessageFeedChatArgs {
+┊   ┊115┊  amount?: number | null;
+┊   ┊116┊  before?: string | null;
+┊   ┊117┊}
+┊   ┊118┊export interface AddChatMutationArgs {
+┊   ┊119┊  recipientId: string;
+┊   ┊120┊}
+┊   ┊121┊export interface AddGroupMutationArgs {
+┊   ┊122┊  recipientIds: string[];
+┊   ┊123┊  groupName: string;
+┊   ┊124┊}
+┊   ┊125┊export interface RemoveChatMutationArgs {
+┊   ┊126┊  chatId: string;
+┊   ┊127┊}
+┊   ┊128┊export interface AddMessageMutationArgs {
+┊   ┊129┊  chatId: string;
+┊   ┊130┊  content: string;
+┊   ┊131┊}
+┊   ┊132┊export interface RemoveMessagesMutationArgs {
+┊   ┊133┊  chatId: string;
+┊   ┊134┊  messageIds?: (string | null)[] | null;
+┊   ┊135┊  all?: boolean | null;
+┊   ┊136┊}
+┊   ┊137┊export interface AddMembersMutationArgs {
+┊   ┊138┊  groupId: string;
+┊   ┊139┊  userIds: string[];
+┊   ┊140┊}
+┊   ┊141┊export interface RemoveMembersMutationArgs {
+┊   ┊142┊  groupId: string;
+┊   ┊143┊  userIds: string[];
+┊   ┊144┊}
+┊   ┊145┊export interface AddAdminsMutationArgs {
+┊   ┊146┊  groupId: string;
+┊   ┊147┊  userIds: string[];
+┊   ┊148┊}
+┊   ┊149┊export interface RemoveAdminsMutationArgs {
+┊   ┊150┊  groupId: string;
+┊   ┊151┊  userIds: string[];
+┊   ┊152┊}
+┊   ┊153┊export interface SetGroupNameMutationArgs {
+┊   ┊154┊  groupId: string;
+┊   ┊155┊}
+┊   ┊156┊export interface SetGroupPictureMutationArgs {
+┊   ┊157┊  groupId: string;
+┊   ┊158┊}
+┊   ┊159┊export interface MarkAsReceivedMutationArgs {
+┊   ┊160┊  chatId: string;
+┊   ┊161┊}
+┊   ┊162┊export interface MarkAsReadMutationArgs {
+┊   ┊163┊  chatId: string;
+┊   ┊164┊}
+┊   ┊165┊export interface MessageAddedSubscriptionArgs {
+┊   ┊166┊  chatId?: string | null;
 ┊ 82┊167┊}
 ┊ 83┊168┊
 ┊ 84┊169┊export enum MessageType {
```
```diff
@@ -146,41 +231,18 @@
 ┊146┊231┊
 ┊147┊232┊export namespace ChatResolvers {
 ┊148┊233┊  export interface Resolvers<Context = any> {
-┊149┊   ┊    id?: IdResolver<string, any, Context> /** May be a chat or a group */;
-┊150┊   ┊    name?: NameResolver<string | null, any, Context> /** Computed for chats */;
-┊151┊   ┊    picture?: PictureResolver<
-┊152┊   ┊      string | null,
-┊153┊   ┊      any,
-┊154┊   ┊      Context
-┊155┊   ┊    > /** Computed for chats */;
-┊156┊   ┊    allTimeMembers?: AllTimeMembersResolver<
-┊157┊   ┊      User[],
-┊158┊   ┊      any,
-┊159┊   ┊      Context
-┊160┊   ┊    > /** All members, current and past ones. */;
-┊161┊   ┊    listingMembers?: ListingMembersResolver<
-┊162┊   ┊      User[],
-┊163┊   ┊      any,
-┊164┊   ┊      Context
-┊165┊   ┊    > /** Whoever gets the chat listed. For groups includes past members who still didn't delete the group. */;
-┊166┊   ┊    actualGroupMembers?: ActualGroupMembersResolver<
-┊167┊   ┊      User[],
-┊168┊   ┊      any,
-┊169┊   ┊      Context
-┊170┊   ┊    > /** Actual members of the group (they are not the only ones who get the group listed). Null for chats. */;
-┊171┊   ┊    admins?: AdminsResolver<User[] | null, any, Context> /** Null for chats */;
-┊172┊   ┊    owner?: OwnerResolver<
-┊173┊   ┊      User | null,
-┊174┊   ┊      any,
-┊175┊   ┊      Context
-┊176┊   ┊    > /** If null the group is read-only. Null for chats. */;
+┊   ┊234┊    id?: IdResolver<string, any, Context>;
+┊   ┊235┊    name?: NameResolver<string | null, any, Context>;
+┊   ┊236┊    picture?: PictureResolver<string | null, any, Context>;
+┊   ┊237┊    allTimeMembers?: AllTimeMembersResolver<User[], any, Context>;
+┊   ┊238┊    listingMembers?: ListingMembersResolver<User[], any, Context>;
+┊   ┊239┊    actualGroupMembers?: ActualGroupMembersResolver<User[], any, Context>;
+┊   ┊240┊    admins?: AdminsResolver<User[] | null, any, Context>;
+┊   ┊241┊    owner?: OwnerResolver<User | null, any, Context>;
 ┊177┊242┊    messages?: MessagesResolver<(Message | null)[], any, Context>;
-┊178┊   ┊    unreadMessages?: UnreadMessagesResolver<
-┊179┊   ┊      number,
-┊180┊   ┊      any,
-┊181┊   ┊      Context
-┊182┊   ┊    > /** Computed property */;
-┊183┊   ┊    isGroup?: IsGroupResolver<boolean, any, Context> /** Computed property */;
+┊   ┊243┊    messageFeed?: MessageFeedResolver<MessageFeed | null, any, Context>;
+┊   ┊244┊    unreadMessages?: UnreadMessagesResolver<number, any, Context>;
+┊   ┊245┊    isGroup?: IsGroupResolver<boolean, any, Context>;
 ┊184┊246┊  }
 ┊185┊247┊
 ┊186┊248┊  export type IdResolver<R = string, Parent = any, Context = any> = Resolver<
```
```diff
@@ -230,6 +292,17 @@
 ┊230┊292┊  > = Resolver<R, Parent, Context, MessagesArgs>;
 ┊231┊293┊  export interface MessagesArgs {
 ┊232┊294┊    amount?: number | null;
+┊   ┊295┊    before?: string | null;
+┊   ┊296┊  }
+┊   ┊297┊
+┊   ┊298┊  export type MessageFeedResolver<
+┊   ┊299┊    R = MessageFeed | null,
+┊   ┊300┊    Parent = any,
+┊   ┊301┊    Context = any
+┊   ┊302┊  > = Resolver<R, Parent, Context, MessageFeedArgs>;
+┊   ┊303┊  export interface MessageFeedArgs {
+┊   ┊304┊    amount?: number | null;
+┊   ┊305┊    before?: string | null;
 ┊233┊306┊  }
 ┊234┊307┊
 ┊235┊308┊  export type UnreadMessagesResolver<
```
```diff
@@ -250,27 +323,11 @@
 ┊250┊323┊    sender?: SenderResolver<User, any, Context>;
 ┊251┊324┊    chat?: ChatResolver<Chat, any, Context>;
 ┊252┊325┊    content?: ContentResolver<string, any, Context>;
-┊253┊   ┊    createdAt?: CreatedAtResolver<string, any, Context>;
-┊254┊   ┊    type?: TypeResolver<
-┊255┊   ┊      number,
-┊256┊   ┊      any,
-┊257┊   ┊      Context
-┊258┊   ┊    > /** FIXME: should return MessageType */;
-┊259┊   ┊    recipients?: RecipientsResolver<
-┊260┊   ┊      Recipient[],
-┊261┊   ┊      any,
-┊262┊   ┊      Context
-┊263┊   ┊    > /** Whoever received the message */;
-┊264┊   ┊    holders?: HoldersResolver<
-┊265┊   ┊      User[],
-┊266┊   ┊      any,
-┊267┊   ┊      Context
-┊268┊   ┊    > /** Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise */;
-┊269┊   ┊    ownership?: OwnershipResolver<
-┊270┊   ┊      boolean,
-┊271┊   ┊      any,
-┊272┊   ┊      Context
-┊273┊   ┊    > /** Computed property */;
+┊   ┊326┊    createdAt?: CreatedAtResolver<Date, any, Context>;
+┊   ┊327┊    type?: TypeResolver<number, any, Context>;
+┊   ┊328┊    recipients?: RecipientsResolver<Recipient[], any, Context>;
+┊   ┊329┊    holders?: HoldersResolver<User[], any, Context>;
+┊   ┊330┊    ownership?: OwnershipResolver<boolean, any, Context>;
 ┊274┊331┊  }
 ┊275┊332┊
 ┊276┊333┊  export type IdResolver<R = string, Parent = any, Context = any> = Resolver<
```
```diff
@@ -294,7 +351,7 @@
 ┊294┊351┊    Context = any
 ┊295┊352┊  > = Resolver<R, Parent, Context>;
 ┊296┊353┊  export type CreatedAtResolver<
-┊297┊   ┊    R = string,
+┊   ┊354┊    R = Date,
 ┊298┊355┊    Parent = any,
 ┊299┊356┊    Context = any
 ┊300┊357┊  > = Resolver<R, Parent, Context>;
```
```diff
@@ -325,8 +382,8 @@
 ┊325┊382┊    user?: UserResolver<User, any, Context>;
 ┊326┊383┊    message?: MessageResolver<Message, any, Context>;
 ┊327┊384┊    chat?: ChatResolver<Chat, any, Context>;
-┊328┊   ┊    receivedAt?: ReceivedAtResolver<string | null, any, Context>;
-┊329┊   ┊    readAt?: ReadAtResolver<string | null, any, Context>;
+┊   ┊385┊    receivedAt?: ReceivedAtResolver<Date | null, any, Context>;
+┊   ┊386┊    readAt?: ReadAtResolver<Date | null, any, Context>;
 ┊330┊387┊  }
 ┊331┊388┊
 ┊332┊389┊  export type UserResolver<R = User, Parent = any, Context = any> = Resolver<
```
```diff
@@ -345,14 +402,211 @@
 ┊345┊402┊    Context
 ┊346┊403┊  >;
 ┊347┊404┊  export type ReceivedAtResolver<
-┊348┊   ┊    R = string | null,
+┊   ┊405┊    R = Date | null,
 ┊349┊406┊    Parent = any,
 ┊350┊407┊    Context = any
 ┊351┊408┊  > = Resolver<R, Parent, Context>;
 ┊352┊409┊  export type ReadAtResolver<
+┊   ┊410┊    R = Date | null,
+┊   ┊411┊    Parent = any,
+┊   ┊412┊    Context = any
+┊   ┊413┊  > = Resolver<R, Parent, Context>;
+┊   ┊414┊}
+┊   ┊415┊
+┊   ┊416┊export namespace MessageFeedResolvers {
+┊   ┊417┊  export interface Resolvers<Context = any> {
+┊   ┊418┊    hasNextPage?: HasNextPageResolver<boolean, any, Context>;
+┊   ┊419┊    cursor?: CursorResolver<string | null, any, Context>;
+┊   ┊420┊    messages?: MessagesResolver<(Message | null)[], any, Context>;
+┊   ┊421┊  }
+┊   ┊422┊
+┊   ┊423┊  export type HasNextPageResolver<
+┊   ┊424┊    R = boolean,
+┊   ┊425┊    Parent = any,
+┊   ┊426┊    Context = any
+┊   ┊427┊  > = Resolver<R, Parent, Context>;
+┊   ┊428┊  export type CursorResolver<
+┊   ┊429┊    R = string | null,
+┊   ┊430┊    Parent = any,
+┊   ┊431┊    Context = any
+┊   ┊432┊  > = Resolver<R, Parent, Context>;
+┊   ┊433┊  export type MessagesResolver<
+┊   ┊434┊    R = (Message | null)[],
+┊   ┊435┊    Parent = any,
+┊   ┊436┊    Context = any
+┊   ┊437┊  > = Resolver<R, Parent, Context>;
+┊   ┊438┊}
+┊   ┊439┊
+┊   ┊440┊export namespace MutationResolvers {
+┊   ┊441┊  export interface Resolvers<Context = any> {
+┊   ┊442┊    addChat?: AddChatResolver<Chat | null, any, Context>;
+┊   ┊443┊    addGroup?: AddGroupResolver<Chat | null, any, Context>;
+┊   ┊444┊    removeChat?: RemoveChatResolver<string | null, any, Context>;
+┊   ┊445┊    addMessage?: AddMessageResolver<Message | null, any, Context>;
+┊   ┊446┊    removeMessages?: RemoveMessagesResolver<
+┊   ┊447┊      (string | null)[] | null,
+┊   ┊448┊      any,
+┊   ┊449┊      Context
+┊   ┊450┊    >;
+┊   ┊451┊    addMembers?: AddMembersResolver<(string | null)[] | null, any, Context>;
+┊   ┊452┊    removeMembers?: RemoveMembersResolver<
+┊   ┊453┊      (string | null)[] | null,
+┊   ┊454┊      any,
+┊   ┊455┊      Context
+┊   ┊456┊    >;
+┊   ┊457┊    addAdmins?: AddAdminsResolver<(string | null)[] | null, any, Context>;
+┊   ┊458┊    removeAdmins?: RemoveAdminsResolver<(string | null)[] | null, any, Context>;
+┊   ┊459┊    setGroupName?: SetGroupNameResolver<string | null, any, Context>;
+┊   ┊460┊    setGroupPicture?: SetGroupPictureResolver<string | null, any, Context>;
+┊   ┊461┊    markAsReceived?: MarkAsReceivedResolver<boolean | null, any, Context>;
+┊   ┊462┊    markAsRead?: MarkAsReadResolver<boolean | null, any, Context>;
+┊   ┊463┊  }
+┊   ┊464┊
+┊   ┊465┊  export type AddChatResolver<
+┊   ┊466┊    R = Chat | null,
+┊   ┊467┊    Parent = any,
+┊   ┊468┊    Context = any
+┊   ┊469┊  > = Resolver<R, Parent, Context, AddChatArgs>;
+┊   ┊470┊  export interface AddChatArgs {
+┊   ┊471┊    recipientId: string;
+┊   ┊472┊  }
+┊   ┊473┊
+┊   ┊474┊  export type AddGroupResolver<
+┊   ┊475┊    R = Chat | null,
+┊   ┊476┊    Parent = any,
+┊   ┊477┊    Context = any
+┊   ┊478┊  > = Resolver<R, Parent, Context, AddGroupArgs>;
+┊   ┊479┊  export interface AddGroupArgs {
+┊   ┊480┊    recipientIds: string[];
+┊   ┊481┊    groupName: string;
+┊   ┊482┊  }
+┊   ┊483┊
+┊   ┊484┊  export type RemoveChatResolver<
+┊   ┊485┊    R = string | null,
+┊   ┊486┊    Parent = any,
+┊   ┊487┊    Context = any
+┊   ┊488┊  > = Resolver<R, Parent, Context, RemoveChatArgs>;
+┊   ┊489┊  export interface RemoveChatArgs {
+┊   ┊490┊    chatId: string;
+┊   ┊491┊  }
+┊   ┊492┊
+┊   ┊493┊  export type AddMessageResolver<
+┊   ┊494┊    R = Message | null,
+┊   ┊495┊    Parent = any,
+┊   ┊496┊    Context = any
+┊   ┊497┊  > = Resolver<R, Parent, Context, AddMessageArgs>;
+┊   ┊498┊  export interface AddMessageArgs {
+┊   ┊499┊    chatId: string;
+┊   ┊500┊    content: string;
+┊   ┊501┊  }
+┊   ┊502┊
+┊   ┊503┊  export type RemoveMessagesResolver<
+┊   ┊504┊    R = (string | null)[] | null,
+┊   ┊505┊    Parent = any,
+┊   ┊506┊    Context = any
+┊   ┊507┊  > = Resolver<R, Parent, Context, RemoveMessagesArgs>;
+┊   ┊508┊  export interface RemoveMessagesArgs {
+┊   ┊509┊    chatId: string;
+┊   ┊510┊    messageIds?: (string | null)[] | null;
+┊   ┊511┊    all?: boolean | null;
+┊   ┊512┊  }
+┊   ┊513┊
+┊   ┊514┊  export type AddMembersResolver<
+┊   ┊515┊    R = (string | null)[] | null,
+┊   ┊516┊    Parent = any,
+┊   ┊517┊    Context = any
+┊   ┊518┊  > = Resolver<R, Parent, Context, AddMembersArgs>;
+┊   ┊519┊  export interface AddMembersArgs {
+┊   ┊520┊    groupId: string;
+┊   ┊521┊    userIds: string[];
+┊   ┊522┊  }
+┊   ┊523┊
+┊   ┊524┊  export type RemoveMembersResolver<
+┊   ┊525┊    R = (string | null)[] | null,
+┊   ┊526┊    Parent = any,
+┊   ┊527┊    Context = any
+┊   ┊528┊  > = Resolver<R, Parent, Context, RemoveMembersArgs>;
+┊   ┊529┊  export interface RemoveMembersArgs {
+┊   ┊530┊    groupId: string;
+┊   ┊531┊    userIds: string[];
+┊   ┊532┊  }
+┊   ┊533┊
+┊   ┊534┊  export type AddAdminsResolver<
+┊   ┊535┊    R = (string | null)[] | null,
+┊   ┊536┊    Parent = any,
+┊   ┊537┊    Context = any
+┊   ┊538┊  > = Resolver<R, Parent, Context, AddAdminsArgs>;
+┊   ┊539┊  export interface AddAdminsArgs {
+┊   ┊540┊    groupId: string;
+┊   ┊541┊    userIds: string[];
+┊   ┊542┊  }
+┊   ┊543┊
+┊   ┊544┊  export type RemoveAdminsResolver<
+┊   ┊545┊    R = (string | null)[] | null,
+┊   ┊546┊    Parent = any,
+┊   ┊547┊    Context = any
+┊   ┊548┊  > = Resolver<R, Parent, Context, RemoveAdminsArgs>;
+┊   ┊549┊  export interface RemoveAdminsArgs {
+┊   ┊550┊    groupId: string;
+┊   ┊551┊    userIds: string[];
+┊   ┊552┊  }
+┊   ┊553┊
+┊   ┊554┊  export type SetGroupNameResolver<
+┊   ┊555┊    R = string | null,
+┊   ┊556┊    Parent = any,
+┊   ┊557┊    Context = any
+┊   ┊558┊  > = Resolver<R, Parent, Context, SetGroupNameArgs>;
+┊   ┊559┊  export interface SetGroupNameArgs {
+┊   ┊560┊    groupId: string;
+┊   ┊561┊  }
+┊   ┊562┊
+┊   ┊563┊  export type SetGroupPictureResolver<
 ┊353┊564┊    R = string | null,
 ┊354┊565┊    Parent = any,
 ┊355┊566┊    Context = any
+┊   ┊567┊  > = Resolver<R, Parent, Context, SetGroupPictureArgs>;
+┊   ┊568┊  export interface SetGroupPictureArgs {
+┊   ┊569┊    groupId: string;
+┊   ┊570┊  }
+┊   ┊571┊
+┊   ┊572┊  export type MarkAsReceivedResolver<
+┊   ┊573┊    R = boolean | null,
+┊   ┊574┊    Parent = any,
+┊   ┊575┊    Context = any
+┊   ┊576┊  > = Resolver<R, Parent, Context, MarkAsReceivedArgs>;
+┊   ┊577┊  export interface MarkAsReceivedArgs {
+┊   ┊578┊    chatId: string;
+┊   ┊579┊  }
+┊   ┊580┊
+┊   ┊581┊  export type MarkAsReadResolver<
+┊   ┊582┊    R = boolean | null,
+┊   ┊583┊    Parent = any,
+┊   ┊584┊    Context = any
+┊   ┊585┊  > = Resolver<R, Parent, Context, MarkAsReadArgs>;
+┊   ┊586┊  export interface MarkAsReadArgs {
+┊   ┊587┊    chatId: string;
+┊   ┊588┊  }
+┊   ┊589┊}
+┊   ┊590┊
+┊   ┊591┊export namespace SubscriptionResolvers {
+┊   ┊592┊  export interface Resolvers<Context = any> {
+┊   ┊593┊    messageAdded?: MessageAddedResolver<Message | null, any, Context>;
+┊   ┊594┊    chatAdded?: ChatAddedResolver<Chat | null, any, Context>;
+┊   ┊595┊  }
+┊   ┊596┊
+┊   ┊597┊  export type MessageAddedResolver<
+┊   ┊598┊    R = Message | null,
+┊   ┊599┊    Parent = any,
+┊   ┊600┊    Context = any
+┊   ┊601┊  > = Resolver<R, Parent, Context, MessageAddedArgs>;
+┊   ┊602┊  export interface MessageAddedArgs {
+┊   ┊603┊    chatId?: string | null;
+┊   ┊604┊  }
+┊   ┊605┊
+┊   ┊606┊  export type ChatAddedResolver<
+┊   ┊607┊    R = Chat | null,
+┊   ┊608┊    Parent = any,
+┊   ┊609┊    Context = any
 ┊356┊610┊  > = Resolver<R, Parent, Context>;
 ┊357┊611┊}
 ┊358┊612┊
```
```diff
@@ -398,7 +652,7 @@
 ┊398┊652┊    chat: Chat;
 ┊399┊653┊    sender: Sender;
 ┊400┊654┊    content: string;
-┊401┊   ┊    createdAt: string;
+┊   ┊655┊    createdAt: Date;
 ┊402┊656┊    type: number;
 ┊403┊657┊    recipients: Recipients[];
 ┊404┊658┊    ownership: boolean;
```
```diff
@@ -420,8 +674,8 @@
 ┊420┊674┊    user: User;
 ┊421┊675┊    message: Message;
 ┊422┊676┊    chat: __Chat;
-┊423┊   ┊    receivedAt?: string | null;
-┊424┊   ┊    readAt?: string | null;
+┊   ┊677┊    receivedAt?: Date | null;
+┊   ┊678┊    readAt?: Date | null;
 ┊425┊679┊  };
 ┊426┊680┊
 ┊427┊681┊  export type User = {
```
```diff
@@ -445,3 +699,77 @@
 ┊445┊699┊    id: string;
 ┊446┊700┊  };
 ┊447┊701┊}
+┊   ┊702┊
+┊   ┊703┊import { Injectable } from "@angular/core";
+┊   ┊704┊
+┊   ┊705┊import * as Apollo from "apollo-angular";
+┊   ┊706┊
+┊   ┊707┊import gql from "graphql-tag";
+┊   ┊708┊
+┊   ┊709┊const ChatWithoutMessagesFragment = gql`
+┊   ┊710┊  fragment ChatWithoutMessages on Chat {
+┊   ┊711┊    id
+┊   ┊712┊    name
+┊   ┊713┊    picture
+┊   ┊714┊    allTimeMembers {
+┊   ┊715┊      id
+┊   ┊716┊    }
+┊   ┊717┊    unreadMessages
+┊   ┊718┊    isGroup
+┊   ┊719┊  }
+┊   ┊720┊`;
+┊   ┊721┊
+┊   ┊722┊const MessageFragment = gql`
+┊   ┊723┊  fragment Message on Message {
+┊   ┊724┊    id
+┊   ┊725┊    chat {
+┊   ┊726┊      id
+┊   ┊727┊    }
+┊   ┊728┊    sender {
+┊   ┊729┊      id
+┊   ┊730┊      name
+┊   ┊731┊    }
+┊   ┊732┊    content
+┊   ┊733┊    createdAt
+┊   ┊734┊    type
+┊   ┊735┊    recipients {
+┊   ┊736┊      user {
+┊   ┊737┊        id
+┊   ┊738┊      }
+┊   ┊739┊      message {
+┊   ┊740┊        id
+┊   ┊741┊        chat {
+┊   ┊742┊          id
+┊   ┊743┊        }
+┊   ┊744┊      }
+┊   ┊745┊      chat {
+┊   ┊746┊        id
+┊   ┊747┊      }
+┊   ┊748┊      receivedAt
+┊   ┊749┊      readAt
+┊   ┊750┊    }
+┊   ┊751┊    ownership
+┊   ┊752┊  }
+┊   ┊753┊`;
+┊   ┊754┊
+┊   ┊755┊@Injectable({
+┊   ┊756┊  providedIn: "root"
+┊   ┊757┊})
+┊   ┊758┊export class GetChatsGQL extends Apollo.Query<
+┊   ┊759┊  GetChats.Query,
+┊   ┊760┊  GetChats.Variables
+┊   ┊761┊> {
+┊   ┊762┊  document: any = gql`
+┊   ┊763┊    query GetChats($amount: Int) {
+┊   ┊764┊      chats {
+┊   ┊765┊        ...ChatWithoutMessages
+┊   ┊766┊        messages(amount: $amount) {
+┊   ┊767┊          ...Message
+┊   ┊768┊        }
+┊   ┊769┊      }
+┊   ┊770┊    }
+┊   ┊771┊
+┊   ┊772┊    ${ChatWithoutMessagesFragment}
+┊   ┊773┊    ${MessageFragment}
+┊   ┊774┊  `;
+┊   ┊775┊}
```

[}]: #

The Apollo Angular template generates a ready to use in your component, strongly typed Angular services, for every defined query, mutation or subscription.

It's possible thanks to the new API of Apollo Angular. More on that in ["Query, Mutation, Subscription services"](http://apollographql.com/docs/angular/basics/services.html?_ga=2.227615197.1327014552.1538988114-793224955.1532981447) chapter of the documentation.

Given an example:

```graphql
query GetChats($amount: Int) {
  chats {
    ...ChatWithoutMessages
    messages(amount: $amount) {
      ...Message
    }
  }
}
```

The Apollo Angular template, after you run `yarn generate`, outputs a service called `GetChatsGQL`:

```typescript
import { GetChatsGQL, GetChats } from '../graphql';

export class AppComponent {
  constructor(private getChatsGQL: GetChatsGQL) {}

  getChats(): GetChats.Chats[] {
    return this.getChatsGQL
      .watch({
        amount: 10,
      })
      .valueChanges.pipe(map(result => result.data.chats));
  }
}
```

> Remember, every operation should has an unique name.

It might look a bit different then what you have already learnt but we promise, the API is even easier.

But first, let's dive into how those services look like under the hood.

```typescript
import { Query } from 'apollo-angular';

@Injectable({
  providedIn: 'root',
})
export class GetChatsGQL extends Query<GetChats.Query, GetChats.Variables> {
  document = gql`
    here goes the document
  `;
}
```

Okay, seems easy but what is the `Query` class, you might ask!

Apollo Angular exposes three classes for three kinds of the GraphQL operation: `Query`, `Mutation` and `Subscription`. Each of them has a different API.

- `Query` has `fetch` and `watch`. First one behave like `Apollo.query()`, second one like `Apollo.watchQuery()`
- `Mutation` has `mutate`
- `Subscription` has `subscribe`

Because the `document` is already defined in a class, they all accept two arguments (Subscription has third one). First argument defines variables, second shapes the options. Thats why we used `this.getChatsGQL.watch({ amount: 10 })`.

Let's stop talking about the API itself and see what benefits it brings us:

- **Less code to write** - no need to create a network call, no need to create Typescript typings, no need to create a dedicated Angular service
- **Strongly typed out of the box — all** types are being generated, no need to write any Typescript definitions and struggle to keep them updated
- _More pleasent API_ to work with
- **Full developer experience of tools and IDEs**  —  development time, autocomplete and error checking, not only across your frontend app but also with your API teams!
- **Tree-shakable** thanks to Angular 6

Most IDEs with the GraphQL support (built-in or thanks to extensions) fully handles `.graphql` files and helps you with features like auto-completion, validation but they strugle with `gql` tag. To fully enjoy GraphQL we highly recommend to use static `.graphql` files.

With all that knowledge, let's use GQL services in our application:

[{]: <helper> (diffStep "2.4" files="src/app/services/chats.service.ts" module="client")

#### Step 2.4: Use auto-generated GQL services

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,21 +1,18 @@
 ┊ 1┊ 1┊import {map} from 'rxjs/operators';
-┊ 2┊  ┊import {Apollo} from 'apollo-angular';
 ┊ 3┊ 2┊import {Injectable} from '@angular/core';
-┊ 4┊  ┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊ 5┊  ┊import {GetChats} from '../../graphql';
+┊  ┊ 3┊import {GetChatsGQL} from '../../graphql';
 ┊ 6┊ 4┊
 ┊ 7┊ 5┊@Injectable()
 ┊ 8┊ 6┊export class ChatsService {
 ┊ 9┊ 7┊  messagesAmount = 3;
 ┊10┊ 8┊
-┊11┊  ┊  constructor(private apollo: Apollo) {}
+┊  ┊ 9┊  constructor(
+┊  ┊10┊    private getChatsGQL: GetChatsGQL
+┊  ┊11┊  ) {}
 ┊12┊12┊
 ┊13┊13┊  getChats() {
-┊14┊  ┊    const query = this.apollo.watchQuery<GetChats.Query>({
-┊15┊  ┊      query: getChatsQuery,
-┊16┊  ┊      variables: {
-┊17┊  ┊        amount: this.messagesAmount,
-┊18┊  ┊      },
+┊  ┊14┊    const query = this.getChatsGQL.watch({
+┊  ┊15┊      amount: this.messagesAmount,
 ┊19┊16┊    });
 ┊20┊17┊    const chats$ = query.valueChanges.pipe(
 ┊21┊18┊      map((result) => result.data.chats)
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step5.md) | [Next Step >](step7.md) |
|:--------------------------------|--------------------------------:|

[}]: #
