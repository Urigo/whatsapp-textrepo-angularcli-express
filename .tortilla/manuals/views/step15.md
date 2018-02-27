# Step 15: Subscriptions

[//]: # (head-end)


## Server

In order to use WebSockets we don't need to install any package, it comes for free with Apollo Server 2.0.

Our GraphQL server will use WebSockets only for subscriptions, while using HTTP for everything else. That means that we will have to add subscriptions on a specific path.
We're using `connectionParams` for the authentication over WebSockets, that means that we won't be using the `Passport` framework at all. Instead we will use the `onConnect` hook to manually validate the parameters provided by the user to either validate the WebSocket connection or throw an error.
We will also return the user object we retrieved from the db, to let the resolvers know who is the current user.

[{]: <helper> (diffStep "5.1" files="index.ts" module="server")

#### [Step 5.1: Subscriptions](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/fdb010e)

##### Changed index.ts
```diff
@@ -7,6 +7,7 @@
 ┊ 7┊ 7┊import basicStrategy from 'passport-http';
 ┊ 8┊ 8┊import bcrypt from 'bcrypt-nodejs';
 ┊ 9┊ 9┊import { db, UserDb } from "./db";
+┊  ┊10┊import { createServer } from "http";
 ┊10┊11┊
 ┊11┊12┊let users = db.users;
 ┊12┊13┊
```
```diff
@@ -69,9 +70,30 @@
 ┊69┊70┊  schema,
 ┊70┊71┊  context(received: any) {
 ┊71┊72┊    return {
-┊72┊  ┊      user: received.req!['user'],
+┊  ┊73┊      user: received.connection ? received.connection.context.user : received.req!['user'],
 ┊73┊74┊    }
 ┊74┊75┊  },
+┊  ┊76┊  subscriptions: {
+┊  ┊77┊    onConnect: (connectionParams: any, webSocket: any) => {
+┊  ┊78┊      if (connectionParams.authToken) {
+┊  ┊79┊        // create a buffer and tell it the data coming in is base64
+┊  ┊80┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊  ┊81┊        // read it back out as a string
+┊  ┊82┊        const [username, password]: string[] = buf.toString().split(':');
+┊  ┊83┊        if (username && password) {
+┊  ┊84┊          const user = users.find(user => user.username == username);
+┊  ┊85┊
+┊  ┊86┊          if (user && validPassword(password, user.password)) {
+┊  ┊87┊            // Set context for the WebSocket
+┊  ┊88┊            return {user};
+┊  ┊89┊          } else {
+┊  ┊90┊            throw new Error('Wrong credentials!');
+┊  ┊91┊          }
+┊  ┊92┊        }
+┊  ┊93┊      }
+┊  ┊94┊      throw new Error('Missing auth token!');
+┊  ┊95┊    }
+┊  ┊96┊  }
 ┊75┊97┊});
 ┊76┊98┊
 ┊77┊99┊apollo.applyMiddleware({
```
```diff
@@ -79,4 +101,11 @@
 ┊ 79┊101┊  path: '/graphql'
 ┊ 80┊102┊});
 ┊ 81┊103┊
-┊ 82┊   ┊app.listen(PORT);
+┊   ┊104┊// Wrap the Express server
+┊   ┊105┊const ws = createServer(app);
+┊   ┊106┊
+┊   ┊107┊apollo.installSubscriptionHandlers(ws);
+┊   ┊108┊
+┊   ┊109┊ws.listen(PORT, () => {
+┊   ┊110┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊111┊});
```

[}]: #

We will use the `PubSub` implementation from `apollo-server-express`, and publish the data using with Apollo Server.

The process of setting up a GraphQL subscriptions server consist of the following steps:

1. Declaring subscriptions in the GraphQL schema
2. Setup a PubSub instance that our server will publish new events to
3. Hook together `PubSub` event and GraphQL subscription.
4. Setting up `ApolloServer`

[{]: <helper> (diffStep "5.1" files="schema/typeDefs.ts" module="server")

#### [Step 5.1: Subscriptions](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/fdb010e)

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -5,6 +5,11 @@
 ┊ 5┊ 5┊    chat(chatId: ID!): Chat
 ┊ 6┊ 6┊  }
 ┊ 7┊ 7┊
+┊  ┊ 8┊  type Subscription {
+┊  ┊ 9┊    messageAdded(chatId: ID): Message
+┊  ┊10┊    chatAdded: Chat
+┊  ┊11┊  }
+┊  ┊12┊
 ┊ 8┊13┊  enum MessageType {
 ┊ 9┊14┊    LOCATION
 ┊10┊15┊    TEXT
```

[}]: #

We created two subscriptions: one to notify for new chats and one to notify for new messages.

[{]: <helper> (diffStep "5.1" files="schema/resolvers.ts" module="server")

#### [Step 5.1: Subscriptions](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/fdb010e)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,10 +1,13 @@
-┊ 1┊  ┊import { IResolvers } from "../types";
-┊ 2┊  ┊import { ChatDb, db, MessageDb, MessageType, RecipientDb } from "../db";
+┊  ┊ 1┊import { PubSub, withFilter } from 'apollo-server-express';
+┊  ┊ 2┊import { ChatDb, db, MessageDb, MessageType, RecipientDb, UserDb } from "../db";
+┊  ┊ 3┊import { IResolvers, MessageAddedSubscriptionArgs } from "../types";
 ┊ 3┊ 4┊import moment from "moment";
 ┊ 4┊ 5┊
 ┊ 5┊ 6┊let users = db.users;
 ┊ 6┊ 7┊let chats = db.chats;
 ┊ 7┊ 8┊
+┊  ┊ 9┊export const pubsub = new PubSub();
+┊  ┊10┊
 ┊ 8┊11┊export const resolvers: IResolvers = {
 ┊ 9┊12┊  Query: {
 ┊10┊13┊    // Show all users for the moment.
```
```diff
@@ -46,6 +49,7 @@
 ┊46┊49┊          messages: [],
 ┊47┊50┊        };
 ┊48┊51┊        chats.push(chat);
+┊  ┊52┊
 ┊49┊53┊        return chat;
 ┊50┊54┊      }
 ┊51┊55┊    },
```
```diff
@@ -69,6 +73,12 @@
 ┊69┊73┊        messages: [],
 ┊70┊74┊      };
 ┊71┊75┊      chats.push(chat);
+┊  ┊76┊
+┊  ┊77┊      pubsub.publish('chatAdded', {
+┊  ┊78┊        creatorId: currentUser.id,
+┊  ┊79┊        chatAdded: chat,
+┊  ┊80┊      });
+┊  ┊81┊
 ┊72┊82┊      return chat;
 ┊73┊83┊    },
 ┊74┊84┊    removeChat: (obj, {chatId}, {user: currentUser}) => {
```
```diff
@@ -197,6 +207,11 @@
 ┊197┊207┊          });
 ┊198┊208┊
 ┊199┊209┊          holderIds = listingMemberIds;
+┊   ┊210┊
+┊   ┊211┊          pubsub.publish('chatAdded', {
+┊   ┊212┊            creatorId: currentUser.id,
+┊   ┊213┊            chatAdded: chat,
+┊   ┊214┊          });
 ┊200┊215┊        }
 ┊201┊216┊      } else {
 ┊202┊217┊        // Group
```
```diff
@@ -241,6 +256,10 @@
 ┊241┊256┊        return chat;
 ┊242┊257┊      });
 ┊243┊258┊
+┊   ┊259┊      pubsub.publish('messageAdded', {
+┊   ┊260┊        messageAdded: message,
+┊   ┊261┊      });
+┊   ┊262┊
 ┊244┊263┊      return message;
 ┊245┊264┊    },
 ┊246┊265┊    removeMessages: (obj, {chatId, messageIds, all}, {user: currentUser}) => {
```
```diff
@@ -282,6 +301,21 @@
 ┊282┊301┊      return deletedIds;
 ┊283┊302┊    },
 ┊284┊303┊  },
+┊   ┊304┊  Subscription: {
+┊   ┊305┊    messageAdded: {
+┊   ┊306┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
+┊   ┊307┊        ({messageAdded}: {messageAdded: MessageDb & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: UserDb }) => {
+┊   ┊308┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
+┊   ┊309┊            !!messageAdded.recipients.find((recipient: RecipientDb) => recipient.userId === currentUser.id);
+┊   ┊310┊        }),
+┊   ┊311┊    },
+┊   ┊312┊    chatAdded: {
+┊   ┊313┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
+┊   ┊314┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: ChatDb}, variables: any, {user: currentUser}: { user: UserDb }) => {
+┊   ┊315┊          return Number(creatorId) !== currentUser.id && !chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊316┊        }),
+┊   ┊317┊    }
+┊   ┊318┊  },
 ┊285┊319┊  Chat: {
 ┊286┊320┊    name: (chat, args, {user: currentUser}) => chat.name ? chat.name : users
 ┊287┊321┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
```

[}]: #

We will publish a message to the `messageAdded` subscription every time that a user sends a message, then we will filter them according to the current user (we don't want to send someone else's messages).
The `chatAdded` subscription is similar: we will publish each time that a group gets created, but not when chats get created. This is because when a user creates a chat the chat doesn't appear to the other peer until he writes the first message. That's why we also publish when new messages get added (we first look if the other peer already gets the chat listed).

## Client

In order to use WebSockets we will need to install a couple of dependencies:

    yarn add apollo-link-ws apollo-utilities subscriptions-transport-ws

First let's create the queries for the GraphQL Subscriptions:

[{]: <helper> (diffStep "11.1" files="src/graphql" module="client")

#### [Step 11.1: Subscriptions](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/94d6817)

##### Changed src&#x2F;graphql.ts
```diff
@@ -66,6 +66,24 @@
 ┊66┊66┊  export type AddMessage = Message.Fragment;
 ┊67┊67┊}
 ┊68┊68┊
+┊  ┊69┊export namespace ChatAdded {
+┊  ┊70┊  export type Variables = {};
+┊  ┊71┊
+┊  ┊72┊  export type Subscription = {
+┊  ┊73┊    __typename?: "Subscription";
+┊  ┊74┊
+┊  ┊75┊    chatAdded: Maybe<ChatAdded>;
+┊  ┊76┊  };
+┊  ┊77┊
+┊  ┊78┊  export type ChatAdded = {
+┊  ┊79┊    __typename?: "Chat";
+┊  ┊80┊
+┊  ┊81┊    messages: (Maybe<Messages>)[];
+┊  ┊82┊  } & ChatWithoutMessages.Fragment;
+┊  ┊83┊
+┊  ┊84┊  export type Messages = Message.Fragment;
+┊  ┊85┊}
+┊  ┊86┊
 ┊69┊87┊export namespace GetChat {
 ┊70┊88┊  export type Variables = {
 ┊71┊89┊    chatId: string;
```
```diff
@@ -126,6 +144,30 @@
 ┊126┊144┊  };
 ┊127┊145┊}
 ┊128┊146┊
+┊   ┊147┊export namespace MessageAdded {
+┊   ┊148┊  export type Variables = {
+┊   ┊149┊    chatId?: Maybe<string>;
+┊   ┊150┊  };
+┊   ┊151┊
+┊   ┊152┊  export type Subscription = {
+┊   ┊153┊    __typename?: "Subscription";
+┊   ┊154┊
+┊   ┊155┊    messageAdded: Maybe<MessageAdded>;
+┊   ┊156┊  };
+┊   ┊157┊
+┊   ┊158┊  export type MessageAdded = {
+┊   ┊159┊    __typename?: "Message";
+┊   ┊160┊
+┊   ┊161┊    chat: Chat;
+┊   ┊162┊  } & Message.Fragment;
+┊   ┊163┊
+┊   ┊164┊  export type Chat = {
+┊   ┊165┊    __typename?: "Chat";
+┊   ┊166┊
+┊   ┊167┊    id: string;
+┊   ┊168┊  };
+┊   ┊169┊}
+┊   ┊170┊
 ┊129┊171┊export namespace RemoveAllMessages {
 ┊130┊172┊  export type Variables = {
 ┊131┊173┊    chatId: string;
```
```diff
@@ -371,6 +413,12 @@
 ┊371┊413┊  markAsRead?: Maybe<boolean>;
 ┊372┊414┊}
 ┊373┊415┊
+┊   ┊416┊export interface Subscription {
+┊   ┊417┊  messageAdded?: Maybe<Message>;
+┊   ┊418┊
+┊   ┊419┊  chatAdded?: Maybe<Chat>;
+┊   ┊420┊}
+┊   ┊421┊
 ┊374┊422┊// ====================================================
 ┊375┊423┊// Arguments
 ┊376┊424┊// ====================================================
```
```diff
@@ -436,6 +484,9 @@
 ┊436┊484┊export interface MarkAsReadMutationArgs {
 ┊437┊485┊  chatId: string;
 ┊438┊486┊}
+┊   ┊487┊export interface MessageAddedSubscriptionArgs {
+┊   ┊488┊  chatId?: Maybe<string>;
+┊   ┊489┊}
 ┊439┊490┊
 ┊440┊491┊// ====================================================
 ┊441┊492┊// START: Apollo Angular template
```
```diff
@@ -562,6 +613,27 @@
 ┊562┊613┊@Injectable({
 ┊563┊614┊  providedIn: "root"
 ┊564┊615┊})
+┊   ┊616┊export class ChatAddedGQL extends Apollo.Subscription<
+┊   ┊617┊  ChatAdded.Subscription,
+┊   ┊618┊  ChatAdded.Variables
+┊   ┊619┊> {
+┊   ┊620┊  document: any = gql`
+┊   ┊621┊    subscription chatAdded {
+┊   ┊622┊      chatAdded {
+┊   ┊623┊        ...ChatWithoutMessages
+┊   ┊624┊        messages {
+┊   ┊625┊          ...Message
+┊   ┊626┊        }
+┊   ┊627┊      }
+┊   ┊628┊    }
+┊   ┊629┊
+┊   ┊630┊    ${ChatWithoutMessagesFragment}
+┊   ┊631┊    ${MessageFragment}
+┊   ┊632┊  `;
+┊   ┊633┊}
+┊   ┊634┊@Injectable({
+┊   ┊635┊  providedIn: "root"
+┊   ┊636┊})
 ┊565┊637┊export class GetChatGQL extends Apollo.Query<GetChat.Query, GetChat.Variables> {
 ┊566┊638┊  document: any = gql`
 ┊567┊639┊    query GetChat($chatId: ID!) {
```
```diff
@@ -618,6 +690,26 @@
 ┊618┊690┊@Injectable({
 ┊619┊691┊  providedIn: "root"
 ┊620┊692┊})
+┊   ┊693┊export class MessageAddedGQL extends Apollo.Subscription<
+┊   ┊694┊  MessageAdded.Subscription,
+┊   ┊695┊  MessageAdded.Variables
+┊   ┊696┊> {
+┊   ┊697┊  document: any = gql`
+┊   ┊698┊    subscription messageAdded($chatId: ID) {
+┊   ┊699┊      messageAdded(chatId: $chatId) {
+┊   ┊700┊        ...Message
+┊   ┊701┊        chat {
+┊   ┊702┊          id
+┊   ┊703┊        }
+┊   ┊704┊      }
+┊   ┊705┊    }
+┊   ┊706┊
+┊   ┊707┊    ${MessageFragment}
+┊   ┊708┊  `;
+┊   ┊709┊}
+┊   ┊710┊@Injectable({
+┊   ┊711┊  providedIn: "root"
+┊   ┊712┊})
 ┊621┊713┊export class RemoveAllMessagesGQL extends Apollo.Mutation<
 ┊622┊714┊  RemoveAllMessages.Mutation,
 ┊623┊715┊  RemoveAllMessages.Variables
```

##### Added src&#x2F;graphql&#x2F;chatAdded.subscription.ts
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const chatAddedSubscription = gql`
+┊  ┊ 6┊  subscription chatAdded {
+┊  ┊ 7┊    chatAdded {
+┊  ┊ 8┊      ...ChatWithoutMessages
+┊  ┊ 9┊      messages {
+┊  ┊10┊        ...Message
+┊  ┊11┊      }
+┊  ┊12┊    }
+┊  ┊13┊  }
+┊  ┊14┊
+┊  ┊15┊  ${fragments['chatWithoutMessages']}
+┊  ┊16┊  ${fragments['message']}
+┊  ┊17┊`;
```

##### Added src&#x2F;graphql&#x2F;messageAdded.subscription.ts
```diff
@@ -0,0 +1,16 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const messageAddedSubscription = gql`
+┊  ┊ 6┊  subscription messageAdded($chatId: ID) {
+┊  ┊ 7┊    messageAdded(chatId: $chatId) {
+┊  ┊ 8┊      ...Message
+┊  ┊ 9┊      chat {
+┊  ┊10┊        id,
+┊  ┊11┊      },
+┊  ┊12┊    }
+┊  ┊13┊  }
+┊  ┊14┊
+┊  ┊15┊  ${fragments['message']}
+┊  ┊16┊`;
```

[}]: #

Then we need to run `graphql-code-generator` to generate the types:

    yarn generator

Now we can update the chats service to update the getChats query every time that we receive a new chat from the subscription.
With GraphQL subscriptions your client will be alerted on push from the server and you should choose the pattern that fits your application the most:

- Use it as a notification and run any logic you want when it fires, for example alerting the user or refetching data
- Use the data sent along with the notification and merge it directly into the store (existing queries are automatically notified)

With subscribeToMore, you can easily do the latter. We will manipulate the store to add the newly created chat.

We will do to do the same for the newMessage subscription, but this time we will have to update two different queries in the store: getChats and getChat.

[{]: <helper> (diffStep "11.1" files="src/app/services/chats.service.ts" module="client")

#### [Step 11.1: Subscriptions](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/94d6817)

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,7 +1,7 @@
 ┊1┊1┊import {concat, map, share, switchMap} from 'rxjs/operators';
 ┊2┊2┊import {Injectable} from '@angular/core';
 ┊3┊3┊import {Observable, AsyncSubject, of} from 'rxjs';
-┊4┊ ┊import {QueryRef} from 'apollo-angular';
+┊ ┊4┊import {Apollo, QueryRef} from 'apollo-angular';
 ┊5┊5┊import * as moment from 'moment';
 ┊6┊6┊import {
 ┊7┊7┊  GetChatsGQL,
```
```diff
@@ -13,6 +13,8 @@
 ┊13┊13┊  GetUsersGQL,
 ┊14┊14┊  AddChatGQL,
 ┊15┊15┊  AddGroupGQL,
+┊  ┊16┊  ChatAddedGQL,
+┊  ┊17┊  MessageAddedGQL,
 ┊16┊18┊  AddMessage,
 ┊17┊19┊  GetChats,
 ┊18┊20┊  GetChat,
```
```diff
@@ -21,6 +23,7 @@
 ┊21┊23┊  GetUsers,
 ┊22┊24┊  AddChat,
 ┊23┊25┊  AddGroup,
+┊  ┊26┊  MessageAdded,
 ┊24┊27┊} from '../../graphql';
 ┊25┊28┊import { DataProxy } from 'apollo-cache';
 ┊26┊29┊import { FetchResult } from 'apollo-link';
```
```diff
@@ -48,11 +51,69 @@
 ┊ 48┊ 51┊    private getUsersGQL: GetUsersGQL,
 ┊ 49┊ 52┊    private addChatGQL: AddChatGQL,
 ┊ 50┊ 53┊    private addGroupGQL: AddGroupGQL,
+┊   ┊ 54┊    private chatAddedGQL: ChatAddedGQL,
+┊   ┊ 55┊    private messageAddedGQL: MessageAddedGQL,
+┊   ┊ 56┊    private apollo: Apollo,
 ┊ 51┊ 57┊    private loginService: LoginService
 ┊ 52┊ 58┊  ) {
 ┊ 53┊ 59┊    this.getChatsWq = this.getChatsGQL.watch({
 ┊ 54┊ 60┊      amount: this.messagesAmount,
 ┊ 55┊ 61┊    });
+┊   ┊ 62┊
+┊   ┊ 63┊    this.getChatsWq.subscribeToMore({
+┊   ┊ 64┊      document: this.chatAddedGQL.document,
+┊   ┊ 65┊      updateQuery: (prev: GetChats.Query, { subscriptionData }) => {
+┊   ┊ 66┊        if (!subscriptionData.data) {
+┊   ┊ 67┊          return prev;
+┊   ┊ 68┊        }
+┊   ┊ 69┊
+┊   ┊ 70┊        const newChat: GetChats.Chats = (<any>subscriptionData).data.chatAdded;
+┊   ┊ 71┊
+┊   ┊ 72┊        return Object.assign({}, prev, {
+┊   ┊ 73┊          chats: [...prev.chats, newChat]
+┊   ┊ 74┊        });
+┊   ┊ 75┊      }
+┊   ┊ 76┊    });
+┊   ┊ 77┊
+┊   ┊ 78┊    this.getChatsWq.subscribeToMore({
+┊   ┊ 79┊      document: this.messageAddedGQL.document,
+┊   ┊ 80┊      updateQuery: (prev: GetChats.Query, { subscriptionData }) => {
+┊   ┊ 81┊        if (!subscriptionData.data) {
+┊   ┊ 82┊          return prev;
+┊   ┊ 83┊        }
+┊   ┊ 84┊
+┊   ┊ 85┊        const newMessage: MessageAdded.MessageAdded = (<any>subscriptionData).data.messageAdded;
+┊   ┊ 86┊
+┊   ┊ 87┊        // We need to update the cache for both Chat and Chats. The following updates the cache for Chat.
+┊   ┊ 88┊        try {
+┊   ┊ 89┊          // Read the data from our cache for this query.
+┊   ┊ 90┊          const {chat}: GetChat.Query = this.apollo.getClient().readQuery({
+┊   ┊ 91┊            query: this.getChatGQL.document,
+┊   ┊ 92┊            variables: {
+┊   ┊ 93┊              chatId: newMessage.chat.id,
+┊   ┊ 94┊            }
+┊   ┊ 95┊          });
+┊   ┊ 96┊
+┊   ┊ 97┊          // Add our message from the mutation to the end.
+┊   ┊ 98┊          chat.messages.push(newMessage);
+┊   ┊ 99┊          // Write our data back to the cache.
+┊   ┊100┊          this.apollo.getClient().writeQuery({
+┊   ┊101┊            query: this.getChatGQL.document,
+┊   ┊102┊            data: {
+┊   ┊103┊              chat
+┊   ┊104┊            }
+┊   ┊105┊          });
+┊   ┊106┊        } catch {
+┊   ┊107┊          console.error('The chat we received an update for does not exist in the store');
+┊   ┊108┊        }
+┊   ┊109┊
+┊   ┊110┊        return Object.assign({}, prev, {
+┊   ┊111┊          chats: [...prev.chats.map(_chat =>
+┊   ┊112┊            _chat.id === newMessage.chat.id ? {..._chat, messages: [..._chat.messages, newMessage]} : _chat)]
+┊   ┊113┊        });
+┊   ┊114┊      }
+┊   ┊115┊    });
+┊   ┊116┊
 ┊ 56┊117┊    this.chats$ = this.getChatsWq.valueChanges.pipe(
 ┊ 57┊118┊      map((result) => result.data.chats)
 ┊ 58┊119┊    );
```

[}]: #

We can finally configure the WebSocket in the GraphQL Module. Please notice that the WebSocket has its own authentication instead of using the `HttpInterceptor`, in fact we use `connectionParams` to send the authorization.
All queries will go through HTTP except the Subscriptions, which will use the WebSocket.

[{]: <helper> (diffStep "11.1" files="src/app/graphql.module.ts" module="client")

#### [Step 11.1: Subscriptions](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/94d6817)

##### Changed src&#x2F;app&#x2F;graphql.module.ts
```diff
@@ -2,6 +2,11 @@
 ┊ 2┊ 2┊import { ApolloModule, APOLLO_OPTIONS } from 'apollo-angular';
 ┊ 3┊ 3┊import { HttpLinkModule, HttpLink } from 'apollo-angular-link-http';
 ┊ 4┊ 4┊import { InMemoryCache, defaultDataIdFromObject } from 'apollo-cache-inmemory';
+┊  ┊ 5┊import {getMainDefinition} from 'apollo-utilities';
+┊  ┊ 6┊import {OperationDefinitionNode} from 'graphql';
+┊  ┊ 7┊import {split} from 'apollo-link';
+┊  ┊ 8┊import {WebSocketLink} from 'apollo-link-ws';
+┊  ┊ 9┊import {LoginService} from './login/services/login.service';
 ┊ 5┊10┊
 ┊ 6┊11┊const uri = 'http://localhost:3000/graphql';
 ┊ 7┊12┊
```
```diff
@@ -14,9 +19,28 @@
 ┊14┊19┊  }
 ┊15┊20┊};
 ┊16┊21┊
-┊17┊  ┊export function createApollo(httpLink: HttpLink) {
+┊  ┊22┊export function createApollo(httpLink: HttpLink, loginService: LoginService) {
+┊  ┊23┊  const subscriptionLink = new WebSocketLink({
+┊  ┊24┊    uri: uri.replace('http', 'ws'),
+┊  ┊25┊    options: {
+┊  ┊26┊      reconnect: true,
+┊  ┊27┊      connectionParams: () => ({
+┊  ┊28┊        authToken: loginService.getAuthHeader() || null
+┊  ┊29┊      })
+┊  ┊30┊    }
+┊  ┊31┊  });
+┊  ┊32┊
+┊  ┊33┊  const link = split(
+┊  ┊34┊    ({ query }) => {
+┊  ┊35┊      const { kind, operation } = getMainDefinition(query) as OperationDefinitionNode;
+┊  ┊36┊      return kind === 'OperationDefinition' && operation === 'subscription';
+┊  ┊37┊    },
+┊  ┊38┊    subscriptionLink,
+┊  ┊39┊    httpLink.create({uri})
+┊  ┊40┊  );
+┊  ┊41┊
 ┊18┊42┊  return {
-┊19┊  ┊    link: httpLink.create({ uri }),
+┊  ┊43┊    link,
 ┊20┊44┊    cache: new InMemoryCache({
 ┊21┊45┊      dataIdFromObject,
 ┊22┊46┊    }),
```
```diff
@@ -29,7 +53,7 @@
 ┊29┊53┊    {
 ┊30┊54┊      provide: APOLLO_OPTIONS,
 ┊31┊55┊      useFactory: createApollo,
-┊32┊  ┊      deps: [HttpLink],
+┊  ┊56┊      deps: [HttpLink, LoginService],
 ┊33┊57┊    },
 ┊34┊58┊  ],
 ┊35┊59┊})
```

[}]: #

We used `split` of ApolloLink to decide which path should a request get, through WebSocket or HTTP. We decided to push subscriptions over the WebSocket and the rest normally, just like before.

Finally, let's fix the tests:

[{]: <helper> (diffStep "11.1" files="src/app/chat-viewer/containers/chat/chat.component.spec.ts, src/app/chats-lister/containers/chats/chats.component.spec.ts, src/app/services/chats.service.spec.ts" module="client")

#### [Step 11.1: Subscriptions](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/94d6817)

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -148,6 +148,8 @@
 ┊148┊148┊    component = fixture.componentInstance;
 ┊149┊149┊    fixture.detectChanges();
 ┊150┊150┊
+┊   ┊151┊    controller.expectOne('chatAdded', 'call to chatAdded api');
+┊   ┊152┊    controller.expectOne('messageAdded', 'call to messageAdded api');
 ┊151┊153┊    controller.expectOne('GetChats', 'call to getChats api');
 ┊152┊154┊
 ┊153┊155┊    const req = controller.expectOne('GetChat', 'call to getChat api');
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -358,8 +358,10 @@
 ┊358┊358┊    component = fixture.componentInstance;
 ┊359┊359┊    fixture.detectChanges();
 ┊360┊360┊
-┊361┊   ┊    const req = controller.expectOne('GetChats', 'GetChats operation');
+┊   ┊361┊    controller.expectOne('chatAdded', 'call to chatAdded api');
+┊   ┊362┊    controller.expectOne('messageAdded', 'call to messageAdded api');
 ┊362┊363┊
+┊   ┊364┊    const req = controller.expectOne('GetChats', 'GetChats operation');
 ┊363┊365┊    req.flush({
 ┊364┊366┊      data: {
 ┊365┊367┊        chats,
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.spec.ts
```diff
@@ -11,7 +11,7 @@
 ┊11┊11┊import { GetChats } from '../../graphql';
 ┊12┊12┊import { dataIdFromObject } from '../graphql.module';
 ┊13┊13┊import { ChatsService } from './chats.service';
-┊14┊  ┊import {LoginService} from '../login/services/login.service';
+┊  ┊14┊import { LoginService } from '../login/services/login.service';
 ┊15┊15┊
 ┊16┊16┊describe('ChatsService', () => {
 ┊17┊17┊  let controller: ApolloTestingController;
```
```diff
@@ -341,6 +341,9 @@
 ┊341┊341┊      }
 ┊342┊342┊    });
 ┊343┊343┊
+┊   ┊344┊    controller.expectOne('chatAdded', 'call to chatAdded api');
+┊   ┊345┊    controller.expectOne('messageAdded', 'call to messageAdded api');
+┊   ┊346┊
 ┊344┊347┊    const req = controller.expectOne('GetChats', 'GetChats operation');
 ┊345┊348┊
 ┊346┊349┊    req.flush({
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step14.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step16.md) |
|:--------------------------------|--------------------------------:|

[}]: #
