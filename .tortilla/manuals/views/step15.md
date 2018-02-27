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
@@ -7,6 +7,9 @@
 ┊ 7┊ 7┊import * as basicStrategy from 'passport-http';
 ┊ 8┊ 8┊import * as bcrypt from 'bcrypt-nodejs';
 ┊ 9┊ 9┊import { db, User } from "./db";
+┊  ┊10┊import { createServer } from "http";
+┊  ┊11┊import { SubscriptionServer } from "subscriptions-transport-ws";
+┊  ┊12┊import { execute, subscribe } from "graphql";
 ┊10┊13┊
 ┊11┊14┊let users = db.users;
 ┊12┊15┊
```
```diff
@@ -76,4 +79,36 @@
 ┊ 76┊ 79┊  endpointURL: '/graphql',
 ┊ 77┊ 80┊}));
 ┊ 78┊ 81┊
-┊ 79┊   ┊app.listen(PORT);
+┊   ┊ 82┊// Wrap the Express server
+┊   ┊ 83┊const ws = createServer(app);
+┊   ┊ 84┊ws.listen(PORT, () => {
+┊   ┊ 85┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊ 86┊  // Set up the WebSocket for handling GraphQL subscriptions
+┊   ┊ 87┊  new SubscriptionServer({
+┊   ┊ 88┊    onConnect: (connectionParams: any, webSocket: any) => {
+┊   ┊ 89┊      if (connectionParams.authToken) {
+┊   ┊ 90┊        // create a buffer and tell it the data coming in is base64
+┊   ┊ 91┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊   ┊ 92┊        // read it back out as a string
+┊   ┊ 93┊        const [username, password]: string[] = buf.toString().split(':');
+┊   ┊ 94┊        if (username && password) {
+┊   ┊ 95┊          const user = users.find(user => user.username == username);
+┊   ┊ 96┊
+┊   ┊ 97┊          if (user && validPassword(password, user.password)) {
+┊   ┊ 98┊            // Set context for the WebSocket
+┊   ┊ 99┊            return {user};
+┊   ┊100┊          } else {
+┊   ┊101┊            throw new Error('Wrong credentials!');
+┊   ┊102┊          }
+┊   ┊103┊        }
+┊   ┊104┊      }
+┊   ┊105┊      throw new Error('Missing auth token!');
+┊   ┊106┊    },
+┊   ┊107┊    execute,
+┊   ┊108┊    subscribe,
+┊   ┊109┊    schema
+┊   ┊110┊  }, {
+┊   ┊111┊    server: ws,
+┊   ┊112┊    path: '/subscriptions',
+┊   ┊113┊  });
+┊   ┊114┊});
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
@@ -7,6 +7,11 @@
 ┊ 7┊ 7┊    chat(chatId: ID!): Chat
 ┊ 8┊ 8┊  }
 ┊ 9┊ 9┊
+┊  ┊10┊  type Subscription {
+┊  ┊11┊    messageAdded(chatId: ID): Message
+┊  ┊12┊    chatAdded: Chat
+┊  ┊13┊  }
+┊  ┊14┊
 ┊10┊15┊  enum MessageType {
 ┊11┊16┊    LOCATION
 ┊12┊17┊    TEXT
```

[}]: #

We created two subscriptions: one to notify for new chats and one to notify for new messages.

[{]: <helper> (diffStep "5.1" files="schema/resolvers.ts" module="server")

#### Step 5.1: Subscriptions

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,14 +1,17 @@
 ┊ 1┊ 1┊import { Chat, db, Message, MessageType, Recipient, User } from "../db";
 ┊ 2┊ 2┊import { IResolvers } from "graphql-tools/dist/Interfaces";
 ┊ 3┊ 3┊import {
-┊ 4┊  ┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs,
+┊  ┊ 4┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs, MessageAddedSubscriptionArgs,
 ┊ 5┊ 5┊  RemoveChatMutationArgs, RemoveMessagesMutationArgs
 ┊ 6┊ 6┊} from "../types";
 ┊ 7┊ 7┊import * as moment from "moment";
+┊  ┊ 8┊import { PubSub, withFilter } from "graphql-subscriptions";
 ┊ 8┊ 9┊
 ┊ 9┊10┊let users = db.users;
 ┊10┊11┊let chats = db.chats;
 ┊11┊12┊
+┊  ┊13┊export const pubsub = new PubSub();
+┊  ┊14┊
 ┊12┊15┊export const resolvers: IResolvers = {
 ┊13┊16┊  Query: {
 ┊14┊17┊    // Show all users for the moment.
```
```diff
@@ -50,6 +53,7 @@
 ┊50┊53┊          messages: [],
 ┊51┊54┊        };
 ┊52┊55┊        chats.push(chat);
+┊  ┊56┊
 ┊53┊57┊        return chat;
 ┊54┊58┊      }
 ┊55┊59┊    },
```
```diff
@@ -73,6 +77,12 @@
 ┊73┊77┊        messages: [],
 ┊74┊78┊      };
 ┊75┊79┊      chats.push(chat);
+┊  ┊80┊
+┊  ┊81┊      pubsub.publish('chatAdded', {
+┊  ┊82┊        creatorId: currentUser.id,
+┊  ┊83┊        chatAdded: chat,
+┊  ┊84┊      });
+┊  ┊85┊
 ┊76┊86┊      return chat;
 ┊77┊87┊    },
 ┊78┊88┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser}: {user: User}): number => {
```
```diff
@@ -201,6 +211,11 @@
 ┊201┊211┊          });
 ┊202┊212┊
 ┊203┊213┊          holderIds = listingMemberIds;
+┊   ┊214┊
+┊   ┊215┊          pubsub.publish('chatAdded', {
+┊   ┊216┊            creatorId: currentUser.id,
+┊   ┊217┊            chatAdded: chat,
+┊   ┊218┊          });
 ┊204┊219┊        }
 ┊205┊220┊      } else {
 ┊206┊221┊        // Group
```
```diff
@@ -245,6 +260,10 @@
 ┊245┊260┊        return chat;
 ┊246┊261┊      });
 ┊247┊262┊
+┊   ┊263┊      pubsub.publish('messageAdded', {
+┊   ┊264┊        messageAdded: message,
+┊   ┊265┊      });
+┊   ┊266┊
 ┊248┊267┊      return message;
 ┊249┊268┊    },
 ┊250┊269┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser}: {user: User}): number[] => {
```
```diff
@@ -286,6 +305,21 @@
 ┊286┊305┊      return deletedIds;
 ┊287┊306┊    },
 ┊288┊307┊  },
+┊   ┊308┊  Subscription: {
+┊   ┊309┊    messageAdded: {
+┊   ┊310┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
+┊   ┊311┊        ({messageAdded}: {messageAdded: Message & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: User }) => {
+┊   ┊312┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
+┊   ┊313┊            !!messageAdded.recipients.find((recipient: Recipient) => recipient.userId === currentUser.id);
+┊   ┊314┊        }),
+┊   ┊315┊    },
+┊   ┊316┊    chatAdded: {
+┊   ┊317┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
+┊   ┊318┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables: any, {user: currentUser}: { user: User }) => {
+┊   ┊319┊          return Number(creatorId) !== currentUser.id && !chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊320┊        }),
+┊   ┊321┊    }
+┊   ┊322┊  },
 ┊289┊323┊  Chat: {
 ┊290┊324┊    name: (chat: Chat, args: any, {user: currentUser}: {user: User}): string => chat.name ? chat.name : users
 ┊291┊325┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
```

[}]: #

We will publish a message to the `messageAdded` subscription every time that a user sends a message, then we will filter them according to the current user (we don't want to send someone else's messages).
The `chatAdded` subscription is similar: we will publish each time that a group gets created, but not when chats get created. This is because when a user creates a chat the chat doesn't appear to the other peer until he writes the first message. That's why we also publish when new messages get added (we first look if the other peer already gets the chat listed).

## Client

In order to use WebSockets we will need to install a couple of dependencies:

    $ npm install apollo-link-ws apollo-utilities subscriptions-transport-ws

First let's create the queries for the GraphQL Subscriptions:

[{]: <helper> (diffStep "11.1" files="src/graphql" module="client")

#### Step 11.1: Subscriptions

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

    $ npm run generator

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
@@ -3,7 +3,10 @@
 ┊ 3┊ 3┊import {Apollo, QueryRef} from 'apollo-angular';
 ┊ 4┊ 4┊import {Injectable} from '@angular/core';
 ┊ 5┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊ 6┊  ┊import {AddChat, AddGroup, AddMessage, GetChat, GetChats, GetUsers, RemoveAllMessages, RemoveChat, RemoveMessages} from '../../types';
+┊  ┊ 6┊import {
+┊  ┊ 7┊  AddChat, AddGroup, AddMessage, GetChat, GetChats, GetUsers, MessageAdded, RemoveAllMessages, RemoveChat,
+┊  ┊ 8┊  RemoveMessages
+┊  ┊ 9┊} from '../../types';
 ┊ 7┊10┊import {getChatQuery} from '../../graphql/getChat.query';
 ┊ 8┊11┊import {addMessageMutation} from '../../graphql/addMessage.mutation';
 ┊ 9┊12┊import {removeChatMutation} from '../../graphql/removeChat.mutation';
```
```diff
@@ -17,6 +20,8 @@
 ┊17┊20┊import * as moment from 'moment';
 ┊18┊21┊import {FetchResult} from 'apollo-link';
 ┊19┊22┊import {LoginService} from '../login/services/login.service';
+┊  ┊23┊import {chatAddedSubscription} from '../../graphql/chatAdded.subscription';
+┊  ┊24┊import {messageAddedSubscription} from '../../graphql/messageAdded.subscription';
 ┊20┊25┊
 ┊21┊26┊@Injectable()
 ┊22┊27┊export class ChatsService {
```
```diff
@@ -35,6 +40,55 @@
 ┊35┊40┊        amount: this.messagesAmount,
 ┊36┊41┊      },
 ┊37┊42┊    });
+┊  ┊43┊
+┊  ┊44┊    this.getChatsWq.subscribeToMore({
+┊  ┊45┊      document: chatAddedSubscription,
+┊  ┊46┊      updateQuery: (prev: GetChats.Query, { subscriptionData }) => {
+┊  ┊47┊        if (!subscriptionData.data) {
+┊  ┊48┊          return prev;
+┊  ┊49┊        }
+┊  ┊50┊
+┊  ┊51┊        const newChat: GetChats.Chats = subscriptionData.data.chatAdded;
+┊  ┊52┊
+┊  ┊53┊        return Object.assign({}, prev, {
+┊  ┊54┊          chats: [...prev.chats, newChat]
+┊  ┊55┊        });
+┊  ┊56┊      }
+┊  ┊57┊    });
+┊  ┊58┊
+┊  ┊59┊    this.getChatsWq.subscribeToMore({
+┊  ┊60┊      document: messageAddedSubscription,
+┊  ┊61┊      updateQuery: (prev: GetChats.Query, { subscriptionData }) => {
+┊  ┊62┊        if (!subscriptionData.data) {
+┊  ┊63┊          return prev;
+┊  ┊64┊        }
+┊  ┊65┊
+┊  ┊66┊        const newMessage: MessageAdded.MessageAdded = subscriptionData.data.messageAdded;
+┊  ┊67┊
+┊  ┊68┊        // We need to update the cache for both Chat and Chats. The following updates the cache for Chat.
+┊  ┊69┊        try {
+┊  ┊70┊          // Read the data from our cache for this query.
+┊  ┊71┊          const {chat}: GetChat.Query = this.apollo.getClient().readQuery({
+┊  ┊72┊            query: getChatQuery, variables: {
+┊  ┊73┊              chatId: newMessage.chat.id,
+┊  ┊74┊            }
+┊  ┊75┊          });
+┊  ┊76┊
+┊  ┊77┊          // Add our message from the mutation to the end.
+┊  ┊78┊          chat.messages.push(newMessage);
+┊  ┊79┊          // Write our data back to the cache.
+┊  ┊80┊          this.apollo.getClient().writeQuery({ query: getChatQuery, data: {chat} });
+┊  ┊81┊        } catch {
+┊  ┊82┊          console.error('The chat we received an update for does not exist in the store');
+┊  ┊83┊        }
+┊  ┊84┊
+┊  ┊85┊        return Object.assign({}, prev, {
+┊  ┊86┊          chats: [...prev.chats.map(_chat =>
+┊  ┊87┊            _chat.id === newMessage.chat.id ? {..._chat, messages: [..._chat.messages, newMessage]} : _chat)]
+┊  ┊88┊        });
+┊  ┊89┊      }
+┊  ┊90┊    });
+┊  ┊91┊
 ┊38┊92┊    this.chats$ = this.getChatsWq.valueChanges.pipe(
 ┊39┊93┊      map((result: ApolloQueryResult<GetChats.Query>) => result.data.chats)
 ┊40┊94┊    );
```
```diff
@@ -110,6 +164,10 @@
 ┊110┊164┊        addMessage: {
 ┊111┊165┊          id: ChatsService.getRandomId(),
 ┊112┊166┊          __typename: 'Message',
+┊   ┊167┊          chat: {
+┊   ┊168┊            id: chatId,
+┊   ┊169┊            __typename: 'Chat',
+┊   ┊170┊          },
 ┊113┊171┊          senderId: this.loginService.getUser().id,
 ┊114┊172┊          sender: {
 ┊115┊173┊            id: this.loginService.getUser().id,
```

[}]: #

We can finally configure the WebSocket in the app module. Please notice that the WebSocket has its own authentication instead of using the HttpInterceptor, in fact we use `connectionParams` to send the authorization.
All queries will go through HTTP except the Subscriptions, which will use the WebSocket.

[{]: <helper> (diffStep "11.1" files="src/app/app.module.ts" module="client")

#### Step 11.1: Subscriptions

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -12,6 +12,11 @@
 ┊12┊12┊import {ChatsCreationModule} from './chats-creation/chats-creation.module';
 ┊13┊13┊import {LoginModule} from './login/login.module';
 ┊14┊14┊import {AuthInterceptor} from './login/services/auth.interceptor';
+┊  ┊15┊import {getMainDefinition} from 'apollo-utilities';
+┊  ┊16┊import {OperationDefinitionNode} from 'graphql';
+┊  ┊17┊import {split} from 'apollo-link';
+┊  ┊18┊import {WebSocketLink} from 'apollo-link-ws';
+┊  ┊19┊import {LoginService} from './login/services/login.service';
 ┊15┊20┊const routes: Routes = [];
 ┊16┊21┊
 ┊17┊22┊@NgModule({
```
```diff
@@ -45,9 +50,30 @@
 ┊45┊50┊  constructor(
 ┊46┊51┊    apollo: Apollo,
 ┊47┊52┊    httpLink: HttpLink,
+┊  ┊53┊    loginService: LoginService,
 ┊48┊54┊  ) {
+┊  ┊55┊    const subscriptionLink = new WebSocketLink({
+┊  ┊56┊      uri:
+┊  ┊57┊        'ws://localhost:3000/subscriptions',
+┊  ┊58┊      options: {
+┊  ┊59┊        reconnect: true,
+┊  ┊60┊        connectionParams: () => ({
+┊  ┊61┊          authToken: loginService.getAuthHeader() || null
+┊  ┊62┊        })
+┊  ┊63┊      }
+┊  ┊64┊    });
+┊  ┊65┊
+┊  ┊66┊    const link = split(
+┊  ┊67┊      ({ query }) => {
+┊  ┊68┊        const { kind, operation } = <OperationDefinitionNode>getMainDefinition(<any>query);
+┊  ┊69┊        return kind === 'OperationDefinition' && operation === 'subscription';
+┊  ┊70┊      },
+┊  ┊71┊      subscriptionLink,
+┊  ┊72┊      httpLink.create(<Options>{uri: 'http://localhost:3000/graphql'})
+┊  ┊73┊    );
+┊  ┊74┊
 ┊49┊75┊    apollo.create({
-┊50┊  ┊      link: httpLink.create(<Options>{uri: 'http://localhost:3000/graphql'}),
+┊  ┊76┊      link,
 ┊51┊77┊      cache: new InMemoryCache({
 ┊52┊78┊        dataIdFromObject: (object: any) => {
 ┊53┊79┊          switch (object.__typename) {
```

[}]: #

Finally, let's fix the tests:

[{]: <helper> (diffStep "11.1" files="src/app/chat-viewer/containers/chat/chat.component.spec.ts, src/app/chats-lister/containers/chats/chats.component.spec.ts, src/app/services/chats.service.spec.ts" module="client")

#### Step 11.1: Subscriptions

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -149,6 +149,8 @@
 ┊149┊149┊    fixture = TestBed.createComponent(ChatComponent);
 ┊150┊150┊    component = fixture.componentInstance;
 ┊151┊151┊    fixture.detectChanges();
+┊   ┊152┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'chatAdded', 'call to chatAdded api');
+┊   ┊153┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'messageAdded', 'call to messageAdded api');
 ┊152┊154┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChats', 'call to getChats api');
 ┊153┊155┊    const req = httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChat', 'call to getChat api');
 ┊154┊156┊    req.flush({
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -364,7 +364,9 @@
 ┊364┊364┊    fixture = TestBed.createComponent(ChatsComponent);
 ┊365┊365┊    component = fixture.componentInstance;
 ┊366┊366┊    fixture.detectChanges();
-┊367┊   ┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊367┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'chatAdded', 'call to chatAdded api');
+┊   ┊368┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'messageAdded', 'call to messageAdded api');
+┊   ┊369┊    const req = httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChats', 'call to getChats api');
 ┊368┊370┊    req.flush({
 ┊369┊371┊      data: {
 ┊370┊372┊        chats
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.spec.ts
```diff
@@ -346,7 +346,9 @@
 ┊346┊346┊      }
 ┊347┊347┊    });
 ┊348┊348┊
-┊349┊   ┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊349┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'chatAdded', 'call to chatAdded api');
+┊   ┊350┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'messageAdded', 'call to messageAdded api');
+┊   ┊351┊    const req = httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChats', 'call to getChats api');
 ┊350┊352┊    expect(req.request.method).toBe('POST');
 ┊351┊353┊    req.flush({
 ┊352┊354┊      data: {
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step14.md) | [Next Step >](step16.md) |
|:--------------------------------|--------------------------------:|

[}]: #
