# Step 11: Subscriptions

[//]: # (head-end)


## Server

In order to use WebSockets we don't need to install any package, it comes for free with Apollo Server 2.0.

Our GraphQL server will use WebSockets only for subscriptions, while using HTTP for everything else. That means that we will have to add subscriptions on a specific path.
We're using `connectionParams` for the authentication over WebSockets, that means that we won't be using the `Passport` framework at all. Instead we will use the `onConnect` hook to manually validate the parameters provided by the user to either validate the WebSocket connection or throw an error.
We will also return the user object we retrieved from the db, to let the resolvers know who is the current user.

[{]: <helper> (diffStep "6.1" files="index.ts" module="server")

#### [Step 6.1: Subscriptions](https://github.com/Urigo/WhatsApp-Clone-Server/commit/e40c1e7)

##### Changed index.ts
```diff
@@ -12,6 +12,8 @@
 ┊12┊12┊import basicStrategy from 'passport-http';
 ┊13┊13┊import bcrypt from 'bcrypt-nodejs';
 ┊14┊14┊import { db, UserDb } from "./db";
+┊  ┊15┊import { createServer } from "http";
+┊  ┊16┊import { pubsub } from "./schema/resolvers";
 ┊15┊17┊
 ┊16┊18┊let users = db.users;
 ┊17┊19┊
```
```diff
@@ -44,6 +46,11 @@
 ┊44┊46┊        name: req.body.name,
 ┊45┊47┊      };
 ┊46┊48┊      users.push(user);
+┊  ┊49┊
+┊  ┊50┊      pubsub.publish('userAdded', {
+┊  ┊51┊        userAdded: user,
+┊  ┊52┊      });
+┊  ┊53┊
 ┊47┊54┊      return done(null, user);
 ┊48┊55┊    }
 ┊49┊56┊    return done(null, false);
```
```diff
@@ -102,9 +109,30 @@
 ┊102┊109┊  schema,
 ┊103┊110┊  context(received: any) {
 ┊104┊111┊    return {
-┊105┊   ┊      currentUser: received.req!['user'],
+┊   ┊112┊      currentUser: received.connection ? received.connection.context.currentUser : received.req!['user'],
 ┊106┊113┊    }
 ┊107┊114┊  },
+┊   ┊115┊  subscriptions: {
+┊   ┊116┊    onConnect: (connectionParams: any, webSocket: any) => {
+┊   ┊117┊      if (connectionParams.authToken) {
+┊   ┊118┊        // create a buffer and tell it the data coming in is base64
+┊   ┊119┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊   ┊120┊        // read it back out as a string
+┊   ┊121┊        const [username, password]: string[] = buf.toString().split(':');
+┊   ┊122┊        if (username && password) {
+┊   ┊123┊          const currentUser = users.find(user => user.username == username);
+┊   ┊124┊
+┊   ┊125┊          if (currentUser && validPassword(password, currentUser.password)) {
+┊   ┊126┊            // Set context for the WebSocket
+┊   ┊127┊            return {currentUser};
+┊   ┊128┊          } else {
+┊   ┊129┊            throw new Error('Wrong credentials!');
+┊   ┊130┊          }
+┊   ┊131┊        }
+┊   ┊132┊      }
+┊   ┊133┊      throw new Error('Missing auth token!');
+┊   ┊134┊    }
+┊   ┊135┊  }
 ┊108┊136┊});
 ┊109┊137┊
 ┊110┊138┊apollo.applyMiddleware({
```
```diff
@@ -112,4 +140,11 @@
 ┊112┊140┊  path: '/graphql'
 ┊113┊141┊});
 ┊114┊142┊
-┊115┊   ┊app.listen(PORT);
+┊   ┊143┊// Wrap the Express server
+┊   ┊144┊const ws = createServer(app);
+┊   ┊145┊
+┊   ┊146┊apollo.installSubscriptionHandlers(ws);
+┊   ┊147┊
+┊   ┊148┊ws.listen(PORT, () => {
+┊   ┊149┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊150┊});
```

[}]: #

We will use the `PubSub` implementation from `apollo-server-express`, and publish the data using with Apollo Server.

The process of setting up a GraphQL subscriptions server consist of the following steps:

1. Declaring subscriptions in the GraphQL schema
2. Setup a PubSub instance that our server will publish new events to
3. Hook together `PubSub` event and GraphQL subscription.
4. Setting up `ApolloServer`

[{]: <helper> (diffStep "6.1" files="schema/typeDefs.ts" module="server")

#### [Step 6.1: Subscriptions](https://github.com/Urigo/WhatsApp-Clone-Server/commit/e40c1e7)

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -8,6 +8,14 @@
 ┊ 8┊ 8┊    chat(chatId: ID!): Chat
 ┊ 9┊ 9┊  }
 ┊10┊10┊
+┊  ┊11┊  type Subscription {
+┊  ┊12┊    messageAdded(chatId: ID): Message
+┊  ┊13┊    chatAdded: Chat
+┊  ┊14┊    chatUpdated: Chat
+┊  ┊15┊    userUpdated: User
+┊  ┊16┊    userAdded: User
+┊  ┊17┊  }
+┊  ┊18┊
 ┊11┊19┊  enum MessageType {
 ┊12┊20┊    LOCATION
 ┊13┊21┊    TEXT
```

[}]: #

We created two subscriptions: one to notify for new chats and one to notify for new messages.

[{]: <helper> (diffStep "6.1" files="schema/resolvers.ts" module="server")

#### [Step 6.1: Subscriptions](https://github.com/Urigo/WhatsApp-Clone-Server/commit/e40c1e7)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,11 +1,14 @@
-┊ 1┊  ┊import { IResolvers } from "../types";
+┊  ┊ 1┊import { PubSub, withFilter } from 'apollo-server-express';
 ┊ 2┊ 2┊import { GraphQLDateTime } from 'graphql-iso-date';
-┊ 3┊  ┊import { ChatDb, db, MessageDb, MessageType, RecipientDb } from "../db";
+┊  ┊ 3┊import { ChatDb, db, MessageDb, MessageType, RecipientDb, UserDb } from "../db";
+┊  ┊ 4┊import { IResolvers, MessageAddedSubscriptionArgs } from "../types";
 ┊ 4┊ 5┊import moment from "moment";
 ┊ 5┊ 6┊
 ┊ 6┊ 7┊let users = db.users;
 ┊ 7┊ 8┊let chats = db.chats;
 ┊ 8┊ 9┊
+┊  ┊10┊export const pubsub = new PubSub();
+┊  ┊11┊
 ┊ 9┊12┊export const resolvers: IResolvers = {
 ┊10┊13┊  Date: GraphQLDateTime,
 ┊11┊14┊  Query: {
```
```diff
@@ -19,6 +22,20 @@
 ┊19┊22┊      currentUser.name = name || currentUser.name;
 ┊20┊23┊      currentUser.picture = picture || currentUser.picture;
 ┊21┊24┊
+┊  ┊25┊      pubsub.publish('userUpdated', {
+┊  ┊26┊        userUpdated: currentUser,
+┊  ┊27┊      });
+┊  ┊28┊
+┊  ┊29┊      // Get a list of the chats who have/had currentUser involved
+┊  ┊30┊      const chatsAffected = chats.filter(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser.id));
+┊  ┊31┊
+┊  ┊32┊      chatsAffected.forEach(chat => {
+┊  ┊33┊        pubsub.publish('chatUpdated', {
+┊  ┊34┊          updaterId: currentUser.id,
+┊  ┊35┊          chatUpdated: chat,
+┊  ┊36┊        })
+┊  ┊37┊      });
+┊  ┊38┊
 ┊22┊39┊      return currentUser;
 ┊23┊40┊    },
 ┊24┊41┊    addChat: (obj, {userId}, {currentUser}) => {
```
```diff
@@ -79,6 +96,12 @@
 ┊ 79┊ 96┊        messages: [],
 ┊ 80┊ 97┊      };
 ┊ 81┊ 98┊      chats.push(chat);
+┊   ┊ 99┊
+┊   ┊100┊      pubsub.publish('chatAdded', {
+┊   ┊101┊        creatorId: currentUser.id,
+┊   ┊102┊        chatAdded: chat,
+┊   ┊103┊      });
+┊   ┊104┊
 ┊ 82┊105┊      return chat;
 ┊ 83┊106┊    },
 ┊ 84┊107┊    updateGroup: (obj, {chatId, groupName, groupPicture}, {currentUser}) => {
```
```diff
@@ -95,6 +118,11 @@
 ┊ 95┊118┊      chat.name = groupName || chat.name;
 ┊ 96┊119┊      chat.picture = groupPicture || chat.picture;
 ┊ 97┊120┊
+┊   ┊121┊      pubsub.publish('chatUpdated', {
+┊   ┊122┊        updaterId: currentUser.id,
+┊   ┊123┊        chatUpdated: chat,
+┊   ┊124┊      });
+┊   ┊125┊
 ┊ 98┊126┊      return chat;
 ┊ 99┊127┊    },
 ┊100┊128┊    removeChat: (obj, {chatId}, {currentUser}) => {
```
```diff
@@ -221,6 +249,11 @@
 ┊221┊249┊          chat.listingMemberIds = chat.listingMemberIds.concat(receiverId);
 ┊222┊250┊
 ┊223┊251┊          holderIds = chat.listingMemberIds;
+┊   ┊252┊
+┊   ┊253┊          pubsub.publish('chatAdded', {
+┊   ┊254┊            creatorId: currentUser.id,
+┊   ┊255┊            chatAdded: chat,
+┊   ┊256┊          });
 ┊224┊257┊        }
 ┊225┊258┊      } else {
 ┊226┊259┊        // Group
```
```diff
@@ -265,6 +298,10 @@
 ┊265┊298┊        return chat;
 ┊266┊299┊      });
 ┊267┊300┊
+┊   ┊301┊      pubsub.publish('messageAdded', {
+┊   ┊302┊        messageAdded: message,
+┊   ┊303┊      });
+┊   ┊304┊
 ┊268┊305┊      return message;
 ┊269┊306┊    },
 ┊270┊307┊    removeMessages: (obj, {chatId, messageIds, all}, {currentUser}) => {
```
```diff
@@ -306,6 +343,39 @@
 ┊306┊343┊      return deletedIds;
 ┊307┊344┊    },
 ┊308┊345┊  },
+┊   ┊346┊  Subscription: {
+┊   ┊347┊    messageAdded: {
+┊   ┊348┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
+┊   ┊349┊        ({messageAdded}: {messageAdded: MessageDb & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊350┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
+┊   ┊351┊            messageAdded.recipients.some(recipient => recipient.userId === currentUser.id);
+┊   ┊352┊        }),
+┊   ┊353┊    },
+┊   ┊354┊    chatAdded: {
+┊   ┊355┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
+┊   ┊356┊        ({creatorId, chatAdded}: {creatorId: number, chatAdded: ChatDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊357┊          return creatorId !== currentUser.id && chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊358┊        }),
+┊   ┊359┊    },
+┊   ┊360┊    chatUpdated: {
+┊   ┊361┊      subscribe: withFilter(() => pubsub.asyncIterator('chatUpdated'),
+┊   ┊362┊        ({updaterId, chatUpdated}: {updaterId: number, chatUpdated: ChatDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊363┊          return updaterId !== currentUser.id && chatUpdated.listingMemberIds.includes(currentUser.id);
+┊   ┊364┊        }),
+┊   ┊365┊    },
+┊   ┊366┊    userUpdated: {
+┊   ┊367┊      subscribe: withFilter(() => pubsub.asyncIterator('userUpdated'),
+┊   ┊368┊        ({userUpdated}: {userUpdated: UserDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊369┊          return userUpdated.id !== currentUser.id;
+┊   ┊370┊        }),
+┊   ┊371┊    },
+┊   ┊372┊    userAdded: {
+┊   ┊373┊      subscribe: withFilter(() => pubsub.asyncIterator('userAdded'),
+┊   ┊374┊        ({userAdded}: {userAdded: UserDb}, variables: any, {currentUser}: { currentUser: UserDb }) => {
+┊   ┊375┊          return userAdded.id !== currentUser.id;
+┊   ┊376┊        }),
+┊   ┊377┊    },
+┊   ┊378┊  },
 ┊309┊379┊  Chat: {
 ┊310┊380┊    name: (chat, args, {currentUser}) => chat.name ? chat.name : users
 ┊311┊381┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
```

[}]: #

We will publish a message to the `messageAdded` subscription every time that a user sends a message, then we will filter them according to the current user (we don't want to send someone else's messages).
The `chatAdded` subscription is similar: we will publish each time that a group gets created, but not when chats get created. This is because when a user creates a chat the chat doesn't appear to the other peer until he writes the first message. That's why we also publish when new messages get added (we first look if the other peer already gets the chat listed).

## Client

In order to use WebSockets we will need to install a couple of dependencies:

    yarn add apollo-link-ws apollo-utilities subscriptions-transport-ws

First let's create the queries for the GraphQL Subscriptions:

[{]: <helper> (diffStep "11.1" files="src/graphql" module="client")

#### [Step 11.1: Subscriptions](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc07f78)

##### Changed src&#x2F;graphql.ts
```diff
@@ -69,6 +69,24 @@
 ┊69┊69┊  export type AddMessage = Message.Fragment;
 ┊70┊70┊}
 ┊71┊71┊
+┊  ┊72┊export namespace ChatAdded {
+┊  ┊73┊  export type Variables = {};
+┊  ┊74┊
+┊  ┊75┊  export type Subscription = {
+┊  ┊76┊    __typename?: "Subscription";
+┊  ┊77┊
+┊  ┊78┊    chatAdded: Maybe<ChatAdded>;
+┊  ┊79┊  };
+┊  ┊80┊
+┊  ┊81┊  export type ChatAdded = {
+┊  ┊82┊    __typename?: "Chat";
+┊  ┊83┊
+┊  ┊84┊    messages: (Maybe<Messages>)[];
+┊  ┊85┊  } & ChatWithoutMessages.Fragment;
+┊  ┊86┊
+┊  ┊87┊  export type Messages = Message.Fragment;
+┊  ┊88┊}
+┊  ┊89┊
 ┊72┊90┊export namespace GetChat {
 ┊73┊91┊  export type Variables = {
 ┊74┊92┊    chatId: string;
```
```diff
@@ -129,6 +147,28 @@
 ┊129┊147┊  };
 ┊130┊148┊}
 ┊131┊149┊
+┊   ┊150┊export namespace MessageAdded {
+┊   ┊151┊  export type Variables = {};
+┊   ┊152┊
+┊   ┊153┊  export type Subscription = {
+┊   ┊154┊    __typename?: "Subscription";
+┊   ┊155┊
+┊   ┊156┊    messageAdded: Maybe<MessageAdded>;
+┊   ┊157┊  };
+┊   ┊158┊
+┊   ┊159┊  export type MessageAdded = {
+┊   ┊160┊    __typename?: "Message";
+┊   ┊161┊
+┊   ┊162┊    chat: Chat;
+┊   ┊163┊  } & Message.Fragment;
+┊   ┊164┊
+┊   ┊165┊  export type Chat = {
+┊   ┊166┊    __typename?: "Chat";
+┊   ┊167┊
+┊   ┊168┊    id: string;
+┊   ┊169┊  };
+┊   ┊170┊}
+┊   ┊171┊
 ┊132┊172┊export namespace RemoveAllMessages {
 ┊133┊173┊  export type Variables = {
 ┊134┊174┊    chatId: string;
```
```diff
@@ -390,6 +430,18 @@
 ┊390┊430┊  markAsRead?: Maybe<boolean>;
 ┊391┊431┊}
 ┊392┊432┊
+┊   ┊433┊export interface Subscription {
+┊   ┊434┊  messageAdded?: Maybe<Message>;
+┊   ┊435┊
+┊   ┊436┊  chatAdded?: Maybe<Chat>;
+┊   ┊437┊
+┊   ┊438┊  chatUpdated?: Maybe<Chat>;
+┊   ┊439┊
+┊   ┊440┊  userUpdated?: Maybe<User>;
+┊   ┊441┊
+┊   ┊442┊  userAdded?: Maybe<User>;
+┊   ┊443┊}
+┊   ┊444┊
 ┊393┊445┊// ====================================================
 ┊394┊446┊// Arguments
 ┊395┊447┊// ====================================================
```
```diff
@@ -469,6 +521,9 @@
 ┊469┊521┊export interface MarkAsReadMutationArgs {
 ┊470┊522┊  chatId: string;
 ┊471┊523┊}
+┊   ┊524┊export interface MessageAddedSubscriptionArgs {
+┊   ┊525┊  chatId?: Maybe<string>;
+┊   ┊526┊}
 ┊472┊527┊
 ┊473┊528┊// ====================================================
 ┊474┊529┊// START: Apollo Angular template
```
```diff
@@ -595,6 +650,27 @@
 ┊595┊650┊@Injectable({
 ┊596┊651┊  providedIn: "root"
 ┊597┊652┊})
+┊   ┊653┊export class ChatAddedGQL extends Apollo.Subscription<
+┊   ┊654┊  ChatAdded.Subscription,
+┊   ┊655┊  ChatAdded.Variables
+┊   ┊656┊> {
+┊   ┊657┊  document: any = gql`
+┊   ┊658┊    subscription chatAdded {
+┊   ┊659┊      chatAdded {
+┊   ┊660┊        ...ChatWithoutMessages
+┊   ┊661┊        messages {
+┊   ┊662┊          ...Message
+┊   ┊663┊        }
+┊   ┊664┊      }
+┊   ┊665┊    }
+┊   ┊666┊
+┊   ┊667┊    ${ChatWithoutMessagesFragment}
+┊   ┊668┊    ${MessageFragment}
+┊   ┊669┊  `;
+┊   ┊670┊}
+┊   ┊671┊@Injectable({
+┊   ┊672┊  providedIn: "root"
+┊   ┊673┊})
 ┊598┊674┊export class GetChatGQL extends Apollo.Query<GetChat.Query, GetChat.Variables> {
 ┊599┊675┊  document: any = gql`
 ┊600┊676┊    query GetChat($chatId: ID!) {
```
```diff
@@ -651,6 +727,26 @@
 ┊651┊727┊@Injectable({
 ┊652┊728┊  providedIn: "root"
 ┊653┊729┊})
+┊   ┊730┊export class MessageAddedGQL extends Apollo.Subscription<
+┊   ┊731┊  MessageAdded.Subscription,
+┊   ┊732┊  MessageAdded.Variables
+┊   ┊733┊> {
+┊   ┊734┊  document: any = gql`
+┊   ┊735┊    subscription messageAdded {
+┊   ┊736┊      messageAdded {
+┊   ┊737┊        ...Message
+┊   ┊738┊        chat {
+┊   ┊739┊          id
+┊   ┊740┊        }
+┊   ┊741┊      }
+┊   ┊742┊    }
+┊   ┊743┊
+┊   ┊744┊    ${MessageFragment}
+┊   ┊745┊  `;
+┊   ┊746┊}
+┊   ┊747┊@Injectable({
+┊   ┊748┊  providedIn: "root"
+┊   ┊749┊})
 ┊654┊750┊export class RemoveAllMessagesGQL extends Apollo.Mutation<
 ┊655┊751┊  RemoveAllMessages.Mutation,
 ┊656┊752┊  RemoveAllMessages.Variables
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
+┊  ┊ 6┊  subscription messageAdded {
+┊  ┊ 7┊    messageAdded {
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

#### [Step 11.1: Subscriptions](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc07f78)

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

#### [Step 11.1: Subscriptions](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc07f78)

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
 ┊ 6┊11┊const uri = 'http://localhost:4000/graphql';
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

#### [Step 11.1: Subscriptions](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc07f78)

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

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step10.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step12.md) |
|:--------------------------------|--------------------------------:|

[}]: #
