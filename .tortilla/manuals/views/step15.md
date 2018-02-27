# Step 15: Subscriptions

[//]: # (head-end)


## Server

In order to use WebSockets we don't need to install any package, it comes for free with Apollo Server 2.0.

Our GraphQL server will use WebSockets only for subscriptions, while using HTTP for everything else. That means that we will have to add subscriptions on a specific path.
We're using `connectionParams` for the authentication over WebSockets, that means that we won't be using the `Passport` framework at all. Instead we will use the `onConnect` hook to manually validate the parameters provided by the user to either validate the WebSocket connection or throw an error.
We will also return the user object we retrieved from the db, to let the resolvers know who is the current user.

[{]: <helper> (diffStep "5.1" files="index.ts" module="server")

#### Step 5.1: Subscriptions

##### Changed index.ts
```diff
@@ -7,9 +7,12 @@
 ┊ 7┊ 7┊import * as basicStrategy from 'passport-http';
 ┊ 8┊ 8┊import * as bcrypt from 'bcrypt-nodejs';
 ┊ 9┊ 9┊import { db, User } from "./db";
+┊  ┊10┊import { createServer } from "http";
 ┊10┊11┊
 ┊11┊12┊let users = db.users;
 ┊12┊13┊
+┊  ┊14┊console.log(users);
+┊  ┊15┊
 ┊13┊16┊function generateHash(password: string) {
 ┊14┊17┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
 ┊15┊18┊}
```
```diff
@@ -69,9 +72,30 @@
 ┊ 69┊ 72┊  schema,
 ┊ 70┊ 73┊  context(received: any) {
 ┊ 71┊ 74┊    return {
-┊ 72┊   ┊      user: received.req!['user'],
+┊   ┊ 75┊      user: received.connection ? received.connection.context.user : received.req!['user'],
 ┊ 73┊ 76┊    }
 ┊ 74┊ 77┊  },
+┊   ┊ 78┊  subscriptions: {
+┊   ┊ 79┊    onConnect: (connectionParams: any, webSocket: any) => {
+┊   ┊ 80┊      if (connectionParams.authToken) {
+┊   ┊ 81┊        // create a buffer and tell it the data coming in is base64
+┊   ┊ 82┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊   ┊ 83┊        // read it back out as a string
+┊   ┊ 84┊        const [username, password]: string[] = buf.toString().split(':');
+┊   ┊ 85┊        if (username && password) {
+┊   ┊ 86┊          const user = users.find(user => user.username == username);
+┊   ┊ 87┊
+┊   ┊ 88┊          if (user && validPassword(password, user.password)) {
+┊   ┊ 89┊            // Set context for the WebSocket
+┊   ┊ 90┊            return {user};
+┊   ┊ 91┊          } else {
+┊   ┊ 92┊            throw new Error('Wrong credentials!');
+┊   ┊ 93┊          }
+┊   ┊ 94┊        }
+┊   ┊ 95┊      }
+┊   ┊ 96┊      throw new Error('Missing auth token!');
+┊   ┊ 97┊    }
+┊   ┊ 98┊  }
 ┊ 75┊ 99┊});
 ┊ 76┊100┊
 ┊ 77┊101┊apollo.applyMiddleware({
```
```diff
@@ -79,4 +103,11 @@
 ┊ 79┊103┊  path: '/graphql'
 ┊ 80┊104┊});
 ┊ 81┊105┊
-┊ 82┊   ┊app.listen(PORT);
+┊   ┊106┊// Wrap the Express server
+┊   ┊107┊const ws = createServer(app);
+┊   ┊108┊
+┊   ┊109┊apollo.installSubscriptionHandlers(ws);
+┊   ┊110┊
+┊   ┊111┊ws.listen(PORT, () => {
+┊   ┊112┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊113┊});
```

[}]: #

We will use the `PubSub` implementation from `apollo-server-express`, and publish the data using with Apollo Server.

The process of setting up a GraphQL subscriptions server consist of the following steps:

1. Declaring subscriptions in the GraphQL schema
2. Setup a PubSub instance that our server will publish new events to
3. Hook together `PubSub` event and GraphQL subscription.
4. Setting up `ApolloServer`

[{]: <helper> (diffStep "5.1" files="schema/typeDefs.ts" module="server")

#### Step 5.1: Subscriptions

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

#### Step 5.1: Subscriptions

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,7 +1,7 @@
-┊1┊ ┊import { IResolvers } from 'apollo-server-express';
+┊ ┊1┊import { PubSub, withFilter, IResolvers } from 'apollo-server-express';
 ┊2┊2┊import { Chat, db, Message, MessageType, Recipient, User } from "../db";
 ┊3┊3┊import {
-┊4┊ ┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs,
+┊ ┊4┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs, MessageAddedSubscriptionArgs,
 ┊5┊5┊  RemoveChatMutationArgs, RemoveMessagesMutationArgs
 ┊6┊6┊} from "../types";
 ┊7┊7┊import * as moment from "moment";
```
```diff
@@ -9,6 +9,8 @@
 ┊ 9┊ 9┊let users = db.users;
 ┊10┊10┊let chats = db.chats;
 ┊11┊11┊
+┊  ┊12┊export const pubsub = new PubSub();
+┊  ┊13┊
 ┊12┊14┊export const resolvers: IResolvers = {
 ┊13┊15┊  Query: {
 ┊14┊16┊    // Show all users for the moment.
```
```diff
@@ -50,6 +52,7 @@
 ┊50┊52┊          messages: [],
 ┊51┊53┊        };
 ┊52┊54┊        chats.push(chat);
+┊  ┊55┊
 ┊53┊56┊        return chat;
 ┊54┊57┊      }
 ┊55┊58┊    },
```
```diff
@@ -73,6 +76,12 @@
 ┊73┊76┊        messages: [],
 ┊74┊77┊      };
 ┊75┊78┊      chats.push(chat);
+┊  ┊79┊
+┊  ┊80┊      pubsub.publish('chatAdded', {
+┊  ┊81┊        creatorId: currentUser.id,
+┊  ┊82┊        chatAdded: chat,
+┊  ┊83┊      });
+┊  ┊84┊
 ┊76┊85┊      return chat;
 ┊77┊86┊    },
 ┊78┊87┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser}: {user: User}): number => {
```
```diff
@@ -201,6 +210,11 @@
 ┊201┊210┊          });
 ┊202┊211┊
 ┊203┊212┊          holderIds = listingMemberIds;
+┊   ┊213┊
+┊   ┊214┊          pubsub.publish('chatAdded', {
+┊   ┊215┊            creatorId: currentUser.id,
+┊   ┊216┊            chatAdded: chat,
+┊   ┊217┊          });
 ┊204┊218┊        }
 ┊205┊219┊      } else {
 ┊206┊220┊        // Group
```
```diff
@@ -245,6 +259,10 @@
 ┊245┊259┊        return chat;
 ┊246┊260┊      });
 ┊247┊261┊
+┊   ┊262┊      pubsub.publish('messageAdded', {
+┊   ┊263┊        messageAdded: message,
+┊   ┊264┊      });
+┊   ┊265┊
 ┊248┊266┊      return message;
 ┊249┊267┊    },
 ┊250┊268┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser}: {user: User}): number[] => {
```
```diff
@@ -286,6 +304,21 @@
 ┊286┊304┊      return deletedIds;
 ┊287┊305┊    },
 ┊288┊306┊  },
+┊   ┊307┊  Subscription: {
+┊   ┊308┊    messageAdded: {
+┊   ┊309┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
+┊   ┊310┊        ({messageAdded}: {messageAdded: Message & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: User }) => {
+┊   ┊311┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
+┊   ┊312┊            !!messageAdded.recipients.find((recipient: Recipient) => recipient.userId === currentUser.id);
+┊   ┊313┊        }),
+┊   ┊314┊    },
+┊   ┊315┊    chatAdded: {
+┊   ┊316┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
+┊   ┊317┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables: any, {user: currentUser}: { user: User }) => {
+┊   ┊318┊          return Number(creatorId) !== currentUser.id && !chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊319┊        }),
+┊   ┊320┊    }
+┊   ┊321┊  },
 ┊289┊322┊  Chat: {
 ┊290┊323┊    name: (chat: Chat, args: any, {user: currentUser}: {user: User}): string => chat.name ? chat.name : users
 ┊291┊324┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
```

[}]: #

We will publish a message to the `messageAdded` subscription every time that a user sends a message, then we will filter them according to the current user (we don't want to send someone else's messages).
The `chatAdded` subscription is similar: we will publish each time that a group gets created, but not when chats get created. This is because when a user creates a chat the chat doesn't appear to the other peer until he writes the first message. That's why we also publish when new messages get added (we first look if the other peer already gets the chat listed).

## Client

In order to use WebSockets we will need to install a couple of dependencies:

    yarn add apollo-link-ws apollo-utilities subscriptions-transport-ws

First let's create the queries for the GraphQL Subscriptions:

[{]: <helper> (diffStep "11.1" files="src/graphql" module="client")

#### Step 11.1: Subscriptions

##### Changed src&#x2F;graphql.ts
```diff
@@ -651,6 +651,22 @@
 ┊651┊651┊  export type AddMessage = Message.Fragment;
 ┊652┊652┊}
 ┊653┊653┊
+┊   ┊654┊export namespace ChatAdded {
+┊   ┊655┊  export type Variables = {};
+┊   ┊656┊
+┊   ┊657┊  export type Subscription = {
+┊   ┊658┊    __typename?: "Subscription";
+┊   ┊659┊    chatAdded?: ChatAdded | null;
+┊   ┊660┊  };
+┊   ┊661┊
+┊   ┊662┊  export type ChatAdded = {
+┊   ┊663┊    __typename?: "Chat";
+┊   ┊664┊    messages: (Messages | null)[];
+┊   ┊665┊  } & ChatWithoutMessages.Fragment;
+┊   ┊666┊
+┊   ┊667┊  export type Messages = Message.Fragment;
+┊   ┊668┊}
+┊   ┊669┊
 ┊654┊670┊export namespace GetChat {
 ┊655┊671┊  export type Variables = {
 ┊656┊672┊    chatId: string;
```
```diff
@@ -703,6 +719,27 @@
 ┊703┊719┊  };
 ┊704┊720┊}
 ┊705┊721┊
+┊   ┊722┊export namespace MessageAdded {
+┊   ┊723┊  export type Variables = {
+┊   ┊724┊    chatId?: string | null;
+┊   ┊725┊  };
+┊   ┊726┊
+┊   ┊727┊  export type Subscription = {
+┊   ┊728┊    __typename?: "Subscription";
+┊   ┊729┊    messageAdded?: MessageAdded | null;
+┊   ┊730┊  };
+┊   ┊731┊
+┊   ┊732┊  export type MessageAdded = {
+┊   ┊733┊    __typename?: "Message";
+┊   ┊734┊    chat: Chat;
+┊   ┊735┊  } & Message.Fragment;
+┊   ┊736┊
+┊   ┊737┊  export type Chat = {
+┊   ┊738┊    __typename?: "Chat";
+┊   ┊739┊    id: string;
+┊   ┊740┊  };
+┊   ┊741┊}
+┊   ┊742┊
 ┊706┊743┊export namespace RemoveAllMessages {
 ┊707┊744┊  export type Variables = {
 ┊708┊745┊    chatId: string;
```
```diff
@@ -924,6 +961,27 @@
 ┊924┊961┊@Injectable({
 ┊925┊962┊  providedIn: "root"
 ┊926┊963┊})
+┊   ┊964┊export class ChatAddedGQL extends Apollo.Subscription<
+┊   ┊965┊  ChatAdded.Subscription,
+┊   ┊966┊  ChatAdded.Variables
+┊   ┊967┊> {
+┊   ┊968┊  document: any = gql`
+┊   ┊969┊    subscription chatAdded {
+┊   ┊970┊      chatAdded {
+┊   ┊971┊        ...ChatWithoutMessages
+┊   ┊972┊        messages {
+┊   ┊973┊          ...Message
+┊   ┊974┊        }
+┊   ┊975┊      }
+┊   ┊976┊    }
+┊   ┊977┊
+┊   ┊978┊    ${ChatWithoutMessagesFragment}
+┊   ┊979┊    ${MessageFragment}
+┊   ┊980┊  `;
+┊   ┊981┊}
+┊   ┊982┊@Injectable({
+┊   ┊983┊  providedIn: "root"
+┊   ┊984┊})
 ┊927┊985┊export class GetChatGQL extends Apollo.Query<GetChat.Query, GetChat.Variables> {
 ┊928┊986┊  document: any = gql`
 ┊929┊987┊    query GetChat($chatId: ID!) {
```
```diff
@@ -980,6 +1038,26 @@
 ┊ 980┊1038┊@Injectable({
 ┊ 981┊1039┊  providedIn: "root"
 ┊ 982┊1040┊})
+┊    ┊1041┊export class MessageAddedGQL extends Apollo.Subscription<
+┊    ┊1042┊  MessageAdded.Subscription,
+┊    ┊1043┊  MessageAdded.Variables
+┊    ┊1044┊> {
+┊    ┊1045┊  document: any = gql`
+┊    ┊1046┊    subscription messageAdded($chatId: ID) {
+┊    ┊1047┊      messageAdded(chatId: $chatId) {
+┊    ┊1048┊        ...Message
+┊    ┊1049┊        chat {
+┊    ┊1050┊          id
+┊    ┊1051┊        }
+┊    ┊1052┊      }
+┊    ┊1053┊    }
+┊    ┊1054┊
+┊    ┊1055┊    ${MessageFragment}
+┊    ┊1056┊  `;
+┊    ┊1057┊}
+┊    ┊1058┊@Injectable({
+┊    ┊1059┊  providedIn: "root"
+┊    ┊1060┊})
 ┊ 983┊1061┊export class RemoveAllMessagesGQL extends Apollo.Mutation<
 ┊ 984┊1062┊  RemoveAllMessages.Mutation,
 ┊ 985┊1063┊  RemoveAllMessages.Variables
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

#### Step 11.1: Subscriptions

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
+┊   ┊ 70┊        const newChat: GetChats.Chats = subscriptionData.data.chatAdded;
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
+┊   ┊ 85┊        const newMessage: MessageAdded.MessageAdded = subscriptionData.data.messageAdded;
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
```diff
@@ -130,6 +191,10 @@
 ┊130┊191┊        addMessage: {
 ┊131┊192┊          id: ChatsService.getRandomId(),
 ┊132┊193┊          __typename: 'Message',
+┊   ┊194┊          chat: {
+┊   ┊195┊            id: chatId,
+┊   ┊196┊            __typename: 'Chat',
+┊   ┊197┊          },
 ┊133┊198┊          senderId: this.loginService.getUser().id,
 ┊134┊199┊          sender: {
 ┊135┊200┊            id: this.loginService.getUser().id,
```

[}]: #

We can finally configure the WebSocket in the GraphQL Module. Please notice that the WebSocket has its own authentication instead of using the `HttpInterceptor`, in fact we use `connectionParams` to send the authorization.
All queries will go through HTTP except the Subscriptions, which will use the WebSocket.

[{]: <helper> (diffStep "11.1" files="src/app/graphql.module.ts" module="client")

#### Step 11.1: Subscriptions

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

#### Step 11.1: Subscriptions

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
@@ -360,8 +360,10 @@
 ┊360┊360┊    component = fixture.componentInstance;
 ┊361┊361┊    fixture.detectChanges();
 ┊362┊362┊
-┊363┊   ┊    const req = controller.expectOne('GetChats', 'GetChats operation');
+┊   ┊363┊    controller.expectOne('chatAdded', 'call to chatAdded api');
+┊   ┊364┊    controller.expectOne('messageAdded', 'call to messageAdded api');
 ┊364┊365┊
+┊   ┊366┊    const req = controller.expectOne('GetChats', 'GetChats operation');
 ┊365┊367┊    req.flush({
 ┊366┊368┊      data: {
 ┊367┊369┊        chats,
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

| [< Previous Step](step14.md) | [Next Step >](step16.md) |
|:--------------------------------|--------------------------------:|

[}]: #
