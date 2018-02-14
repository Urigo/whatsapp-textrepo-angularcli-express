# Step 1: Chats listing

[//]: # (head-end)


Our goal for this chapter will be very simple: we want to simply retrieve and display a list of chats from our own server.

## Server

We'll start with the server, so let's install a couple of packages first:

    yarn add apollo-server-express body-parser cors express graphql
    yarn add -D @types/body-parser @types/cors @types/express @types/graphql

Express is a fast, unopinionated, minimalist web framework for node.
Cross-Origin Resource Sharing (CORS) is a mechanism that uses additional HTTP headers to let a user agent gain permission to access selected resources from a server on a different origin (domain) than the site currently in use. A user agent makes a cross-origin HTTP request when it requests a resource from a different domain, protocol, or port than the one from which the current document originated.
We will need CORS because Webpack's development server used in the client will make use of a different port than the Express server, thus configuring a different origin.
GraphQL is a query language for APIs and a runtime for fulfilling those queries with your existing data. GraphQL provides a complete and understandable description of the data in your API, gives clients the power to ask for exactly what they need and nothing more, makes it easier to evolve APIs over time, and enables powerful developer tools.
Apollo Server is a community-maintained open-source GraphQL server. It works with pretty much all Node.js HTTP server frameworks. Apollo Server works with any GraphQL schema built with GraphQL.js or with a convenience library such as graphql-tools.

The GraphQL query language is basically about selecting fields on objects. Because the shape of a GraphQL query closely matches the result, you can predict what the query will return without knowing that much about the server. But it's useful to have an exact description of the data we can ask for - what fields can we select? What kinds of objects might they return? What fields are available on those sub-objects? That's where the schema comes in.
Every GraphQL service defines a set of types which completely describe the set of possible data you can query on that service. Then, when queries come in, they are validated and executed against that schema.

For the moment let's create some empty schemas and resolvers:

[{]: <helper> (diffStep "1.1" files="schema/*" module="server")

#### Step 1.1: Create empty Apollo server

##### Added schema&#x2F;index.ts
```diff
@@ -0,0 +1,8 @@
+┊ ┊1┊import { makeExecutableSchema } from 'apollo-server-express';
+┊ ┊2┊import typeDefs from './typeDefs';
+┊ ┊3┊import { resolvers } from './resolvers';
+┊ ┊4┊
+┊ ┊5┊export const schema = makeExecutableSchema({
+┊ ┊6┊  typeDefs,
+┊ ┊7┊  resolvers,
+┊ ┊8┊});
```

##### Added schema&#x2F;resolvers.ts
```diff
@@ -0,0 +1,5 @@
+┊ ┊1┊import { IResolvers } from 'apollo-server-express';
+┊ ┊2┊
+┊ ┊3┊export const resolvers: IResolvers = {
+┊ ┊4┊  Query: {},
+┊ ┊5┊};
```

##### Added schema&#x2F;typeDefs.ts
```diff
@@ -0,0 +1,2 @@
+┊ ┊1┊export default `
+┊ ┊2┊`;
```

[}]: #

Time to create our index:

[{]: <helper> (diffStep "1.1" files="^index.ts" module="server")

#### Step 1.1: Create empty Apollo server

##### Added index.ts
```diff
@@ -0,0 +1,23 @@
+┊  ┊ 1┊import { schema } from "./schema";
+┊  ┊ 2┊import bodyParser from "body-parser";
+┊  ┊ 3┊import cors from 'cors';
+┊  ┊ 4┊import express from 'express';
+┊  ┊ 5┊import { ApolloServer } from "apollo-server-express";
+┊  ┊ 6┊
+┊  ┊ 7┊const PORT = 4000;
+┊  ┊ 8┊
+┊  ┊ 9┊const app = express();
+┊  ┊10┊
+┊  ┊11┊app.use(cors());
+┊  ┊12┊app.use(bodyParser.json());
+┊  ┊13┊
+┊  ┊14┊const apollo = new ApolloServer({
+┊  ┊15┊  schema
+┊  ┊16┊});
+┊  ┊17┊
+┊  ┊18┊apollo.applyMiddleware({
+┊  ┊19┊  app,
+┊  ┊20┊  path: '/graphql'
+┊  ┊21┊});
+┊  ┊22┊
+┊  ┊23┊app.listen(PORT);
```

[}]: #

Now we want to feed our graphql server with some data. Soon we will need `moment`, so let's install it:

    yarn add moment

Now we can create a fake db:

[{]: <helper> (diffStep "1.2" files="db.ts" module="server")

#### Step 1.2: Add fake db

##### Added db.ts
```diff
@@ -0,0 +1,448 @@
+┊   ┊  1┊import moment from 'moment';
+┊   ┊  2┊
+┊   ┊  3┊export enum MessageType {
+┊   ┊  4┊  PICTURE,
+┊   ┊  5┊  TEXT,
+┊   ┊  6┊  LOCATION,
+┊   ┊  7┊}
+┊   ┊  8┊
+┊   ┊  9┊export interface UserDb {
+┊   ┊ 10┊  id: number,
+┊   ┊ 11┊  username: string,
+┊   ┊ 12┊  password: string,
+┊   ┊ 13┊  name: string,
+┊   ┊ 14┊  picture?: string | null,
+┊   ┊ 15┊  phone?: string | null,
+┊   ┊ 16┊}
+┊   ┊ 17┊
+┊   ┊ 18┊export interface ChatDb {
+┊   ┊ 19┊  id: number,
+┊   ┊ 20┊  createdAt: Date,
+┊   ┊ 21┊  name?: string | null,
+┊   ┊ 22┊  picture?: string | null,
+┊   ┊ 23┊  // All members, current and past ones.
+┊   ┊ 24┊  allTimeMemberIds: number[],
+┊   ┊ 25┊  // Whoever gets the chat listed. For groups includes past members who still didn't delete the group.
+┊   ┊ 26┊  listingMemberIds: number[],
+┊   ┊ 27┊  // Actual members of the group (they are not the only ones who get the group listed). Null for chats.
+┊   ┊ 28┊  actualGroupMemberIds?: number[] | null,
+┊   ┊ 29┊  adminIds?: number[] | null,
+┊   ┊ 30┊  ownerId?: number | null,
+┊   ┊ 31┊  messages: MessageDb[],
+┊   ┊ 32┊}
+┊   ┊ 33┊
+┊   ┊ 34┊export interface MessageDb {
+┊   ┊ 35┊  id: number,
+┊   ┊ 36┊  chatId: number,
+┊   ┊ 37┊  senderId: number,
+┊   ┊ 38┊  content: string,
+┊   ┊ 39┊  createdAt: Date,
+┊   ┊ 40┊  type: MessageType,
+┊   ┊ 41┊  recipients: RecipientDb[],
+┊   ┊ 42┊  holderIds: number[],
+┊   ┊ 43┊}
+┊   ┊ 44┊
+┊   ┊ 45┊export interface RecipientDb {
+┊   ┊ 46┊  userId: number,
+┊   ┊ 47┊  messageId: number,
+┊   ┊ 48┊  chatId: number,
+┊   ┊ 49┊  receivedAt: Date | null,
+┊   ┊ 50┊  readAt: Date | null,
+┊   ┊ 51┊}
+┊   ┊ 52┊
+┊   ┊ 53┊const users: UserDb[] = [
+┊   ┊ 54┊  {
+┊   ┊ 55┊    id: 1,
+┊   ┊ 56┊    username: 'ethan',
+┊   ┊ 57┊    password: '$2a$08$NO9tkFLCoSqX1c5wk3s7z.JfxaVMKA.m7zUDdDwEquo4rvzimQeJm', // 111
+┊   ┊ 58┊    name: 'Ethan Gonzalez',
+┊   ┊ 59┊    picture: 'https://randomuser.me/api/portraits/thumb/men/1.jpg',
+┊   ┊ 60┊    phone: '+391234567890',
+┊   ┊ 61┊  },
+┊   ┊ 62┊  {
+┊   ┊ 63┊    id: 2,
+┊   ┊ 64┊    username: 'bryan',
+┊   ┊ 65┊    password: '$2a$08$xE4FuCi/ifxjL2S8CzKAmuKLwv18ktksSN.F3XYEnpmcKtpbpeZgO', // 222
+┊   ┊ 66┊    name: 'Bryan Wallace',
+┊   ┊ 67┊    picture: 'https://randomuser.me/api/portraits/thumb/men/2.jpg',
+┊   ┊ 68┊    phone: '+391234567891',
+┊   ┊ 69┊  },
+┊   ┊ 70┊  {
+┊   ┊ 71┊    id: 3,
+┊   ┊ 72┊    username: 'avery',
+┊   ┊ 73┊    password: '$2a$08$UHgH7J8G6z1mGQn2qx2kdeWv0jvgHItyAsL9hpEUI3KJmhVW5Q1d.', // 333
+┊   ┊ 74┊    name: 'Avery Stewart',
+┊   ┊ 75┊    picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 76┊    phone: '+391234567892',
+┊   ┊ 77┊  },
+┊   ┊ 78┊  {
+┊   ┊ 79┊    id: 4,
+┊   ┊ 80┊    username: 'katie',
+┊   ┊ 81┊    password: '$2a$08$wR1k5Q3T9FC7fUgB7Gdb9Os/GV7dGBBf4PLlWT7HERMFhmFDt47xi', // 444
+┊   ┊ 82┊    name: 'Katie Peterson',
+┊   ┊ 83┊    picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
+┊   ┊ 84┊    phone: '+391234567893',
+┊   ┊ 85┊  },
+┊   ┊ 86┊  {
+┊   ┊ 87┊    id: 5,
+┊   ┊ 88┊    username: 'ray',
+┊   ┊ 89┊    password: '$2a$08$6.mbXqsDX82ZZ7q5d8Osb..JrGSsNp4R3IKj7mxgF6YGT0OmMw242', // 555
+┊   ┊ 90┊    name: 'Ray Edwards',
+┊   ┊ 91┊    picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
+┊   ┊ 92┊    phone: '+391234567894',
+┊   ┊ 93┊  },
+┊   ┊ 94┊  {
+┊   ┊ 95┊    id: 6,
+┊   ┊ 96┊    username: 'niko',
+┊   ┊ 97┊    password: '$2a$08$fL5lZR.Rwf9FWWe8XwwlceiPBBim8n9aFtaem.INQhiKT4.Ux3Uq.', // 666
+┊   ┊ 98┊    name: 'Niccolò Belli',
+┊   ┊ 99┊    picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
+┊   ┊100┊    phone: '+391234567895',
+┊   ┊101┊  },
+┊   ┊102┊  {
+┊   ┊103┊    id: 7,
+┊   ┊104┊    username: 'mario',
+┊   ┊105┊    password: '$2a$08$nDHDmWcVxDnH5DDT3HMMC.psqcnu6wBiOgkmJUy9IH..qxa3R6YrO', // 777
+┊   ┊106┊    name: 'Mario Rossi',
+┊   ┊107┊    picture: 'https://randomuser.me/api/portraits/thumb/men/5.jpg',
+┊   ┊108┊    phone: '+391234567896',
+┊   ┊109┊  },
+┊   ┊110┊];
+┊   ┊111┊
+┊   ┊112┊const chats: ChatDb[] = [
+┊   ┊113┊  {
+┊   ┊114┊    id: 1,
+┊   ┊115┊    createdAt: moment().subtract(10, 'weeks').toDate(),
+┊   ┊116┊    name: null,
+┊   ┊117┊    picture: null,
+┊   ┊118┊    allTimeMemberIds: [1, 3],
+┊   ┊119┊    listingMemberIds: [1, 3],
+┊   ┊120┊    adminIds: null,
+┊   ┊121┊    ownerId: null,
+┊   ┊122┊    messages: [
+┊   ┊123┊      {
+┊   ┊124┊        id: 1,
+┊   ┊125┊        chatId: 1,
+┊   ┊126┊        senderId: 1,
+┊   ┊127┊        content: 'You on your way?',
+┊   ┊128┊        createdAt: moment().subtract(1, 'hours').toDate(),
+┊   ┊129┊        type: MessageType.TEXT,
+┊   ┊130┊        recipients: [
+┊   ┊131┊          {
+┊   ┊132┊            userId: 3,
+┊   ┊133┊            messageId: 1,
+┊   ┊134┊            chatId: 1,
+┊   ┊135┊            receivedAt: null,
+┊   ┊136┊            readAt: null,
+┊   ┊137┊          },
+┊   ┊138┊        ],
+┊   ┊139┊        holderIds: [1, 3],
+┊   ┊140┊      },
+┊   ┊141┊      {
+┊   ┊142┊        id: 2,
+┊   ┊143┊        chatId: 1,
+┊   ┊144┊        senderId: 3,
+┊   ┊145┊        content: 'Yep!',
+┊   ┊146┊        createdAt: moment().subtract(1, 'hours').add(5, 'minutes').toDate(),
+┊   ┊147┊        type: MessageType.TEXT,
+┊   ┊148┊        recipients: [
+┊   ┊149┊          {
+┊   ┊150┊            userId: 1,
+┊   ┊151┊            messageId: 2,
+┊   ┊152┊            chatId: 1,
+┊   ┊153┊            receivedAt: null,
+┊   ┊154┊            readAt: null,
+┊   ┊155┊          },
+┊   ┊156┊        ],
+┊   ┊157┊        holderIds: [3, 1],
+┊   ┊158┊      },
+┊   ┊159┊    ],
+┊   ┊160┊  },
+┊   ┊161┊  {
+┊   ┊162┊    id: 2,
+┊   ┊163┊    createdAt: moment().subtract(9, 'weeks').toDate(),
+┊   ┊164┊    name: null,
+┊   ┊165┊    picture: null,
+┊   ┊166┊    allTimeMemberIds: [1, 4],
+┊   ┊167┊    listingMemberIds: [1, 4],
+┊   ┊168┊    adminIds: null,
+┊   ┊169┊    ownerId: null,
+┊   ┊170┊    messages: [
+┊   ┊171┊      {
+┊   ┊172┊        id: 1,
+┊   ┊173┊        chatId: 2,
+┊   ┊174┊        senderId: 1,
+┊   ┊175┊        content: 'Hey, it\'s me',
+┊   ┊176┊        createdAt: moment().subtract(2, 'hours').toDate(),
+┊   ┊177┊        type: MessageType.TEXT,
+┊   ┊178┊        recipients: [
+┊   ┊179┊          {
+┊   ┊180┊            userId: 4,
+┊   ┊181┊            messageId: 1,
+┊   ┊182┊            chatId: 2,
+┊   ┊183┊            receivedAt: null,
+┊   ┊184┊            readAt: null,
+┊   ┊185┊          },
+┊   ┊186┊        ],
+┊   ┊187┊        holderIds: [1, 4],
+┊   ┊188┊      },
+┊   ┊189┊    ],
+┊   ┊190┊  },
+┊   ┊191┊  {
+┊   ┊192┊    id: 3,
+┊   ┊193┊    createdAt: moment().subtract(8, 'weeks').toDate(),
+┊   ┊194┊    name: null,
+┊   ┊195┊    picture: null,
+┊   ┊196┊    allTimeMemberIds: [1, 5],
+┊   ┊197┊    listingMemberIds: [1, 5],
+┊   ┊198┊    adminIds: null,
+┊   ┊199┊    ownerId: null,
+┊   ┊200┊    messages: [
+┊   ┊201┊      {
+┊   ┊202┊        id: 1,
+┊   ┊203┊        chatId: 3,
+┊   ┊204┊        senderId: 1,
+┊   ┊205┊        content: 'I should buy a boat',
+┊   ┊206┊        createdAt: moment().subtract(1, 'days').toDate(),
+┊   ┊207┊        type: MessageType.TEXT,
+┊   ┊208┊        recipients: [
+┊   ┊209┊          {
+┊   ┊210┊            userId: 5,
+┊   ┊211┊            messageId: 1,
+┊   ┊212┊            chatId: 3,
+┊   ┊213┊            receivedAt: null,
+┊   ┊214┊            readAt: null,
+┊   ┊215┊          },
+┊   ┊216┊        ],
+┊   ┊217┊        holderIds: [1, 5],
+┊   ┊218┊      },
+┊   ┊219┊      {
+┊   ┊220┊        id: 2,
+┊   ┊221┊        chatId: 3,
+┊   ┊222┊        senderId: 1,
+┊   ┊223┊        content: 'You still there?',
+┊   ┊224┊        createdAt: moment().subtract(1, 'days').add(16, 'hours').toDate(),
+┊   ┊225┊        type: MessageType.TEXT,
+┊   ┊226┊        recipients: [
+┊   ┊227┊          {
+┊   ┊228┊            userId: 5,
+┊   ┊229┊            messageId: 2,
+┊   ┊230┊            chatId: 3,
+┊   ┊231┊            receivedAt: null,
+┊   ┊232┊            readAt: null,
+┊   ┊233┊          },
+┊   ┊234┊        ],
+┊   ┊235┊        holderIds: [1, 5],
+┊   ┊236┊      },
+┊   ┊237┊    ],
+┊   ┊238┊  },
+┊   ┊239┊  {
+┊   ┊240┊    id: 4,
+┊   ┊241┊    createdAt: moment().subtract(7, 'weeks').toDate(),
+┊   ┊242┊    name: null,
+┊   ┊243┊    picture: null,
+┊   ┊244┊    allTimeMemberIds: [3, 4],
+┊   ┊245┊    listingMemberIds: [3, 4],
+┊   ┊246┊    adminIds: null,
+┊   ┊247┊    ownerId: null,
+┊   ┊248┊    messages: [
+┊   ┊249┊      {
+┊   ┊250┊        id: 1,
+┊   ┊251┊        chatId: 4,
+┊   ┊252┊        senderId: 3,
+┊   ┊253┊        content: 'Look at my mukluks!',
+┊   ┊254┊        createdAt: moment().subtract(4, 'days').toDate(),
+┊   ┊255┊        type: MessageType.TEXT,
+┊   ┊256┊        recipients: [
+┊   ┊257┊          {
+┊   ┊258┊            userId: 4,
+┊   ┊259┊            messageId: 1,
+┊   ┊260┊            chatId: 4,
+┊   ┊261┊            receivedAt: null,
+┊   ┊262┊            readAt: null,
+┊   ┊263┊          },
+┊   ┊264┊        ],
+┊   ┊265┊        holderIds: [3, 4],
+┊   ┊266┊      },
+┊   ┊267┊    ],
+┊   ┊268┊  },
+┊   ┊269┊  {
+┊   ┊270┊    id: 5,
+┊   ┊271┊    createdAt: moment().subtract(6, 'weeks').toDate(),
+┊   ┊272┊    name: null,
+┊   ┊273┊    picture: null,
+┊   ┊274┊    allTimeMemberIds: [2, 5],
+┊   ┊275┊    listingMemberIds: [2, 5],
+┊   ┊276┊    adminIds: null,
+┊   ┊277┊    ownerId: null,
+┊   ┊278┊    messages: [
+┊   ┊279┊      {
+┊   ┊280┊        id: 1,
+┊   ┊281┊        chatId: 5,
+┊   ┊282┊        senderId: 2,
+┊   ┊283┊        content: 'This is wicked good ice cream.',
+┊   ┊284┊        createdAt: moment().subtract(2, 'weeks').toDate(),
+┊   ┊285┊        type: MessageType.TEXT,
+┊   ┊286┊        recipients: [
+┊   ┊287┊          {
+┊   ┊288┊            userId: 5,
+┊   ┊289┊            messageId: 1,
+┊   ┊290┊            chatId: 5,
+┊   ┊291┊            receivedAt: null,
+┊   ┊292┊            readAt: null,
+┊   ┊293┊          },
+┊   ┊294┊        ],
+┊   ┊295┊        holderIds: [2, 5],
+┊   ┊296┊      },
+┊   ┊297┊      {
+┊   ┊298┊        id: 2,
+┊   ┊299┊        chatId: 6,
+┊   ┊300┊        senderId: 5,
+┊   ┊301┊        content: 'Love it!',
+┊   ┊302┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
+┊   ┊303┊        type: MessageType.TEXT,
+┊   ┊304┊        recipients: [
+┊   ┊305┊          {
+┊   ┊306┊            userId: 2,
+┊   ┊307┊            messageId: 2,
+┊   ┊308┊            chatId: 5,
+┊   ┊309┊            receivedAt: null,
+┊   ┊310┊            readAt: null,
+┊   ┊311┊          },
+┊   ┊312┊        ],
+┊   ┊313┊        holderIds: [5, 2],
+┊   ┊314┊      },
+┊   ┊315┊    ],
+┊   ┊316┊  },
+┊   ┊317┊  {
+┊   ┊318┊    id: 6,
+┊   ┊319┊    createdAt: moment().subtract(5, 'weeks').toDate(),
+┊   ┊320┊    name: null,
+┊   ┊321┊    picture: null,
+┊   ┊322┊    allTimeMemberIds: [1, 6],
+┊   ┊323┊    listingMemberIds: [1],
+┊   ┊324┊    adminIds: null,
+┊   ┊325┊    ownerId: null,
+┊   ┊326┊    messages: [],
+┊   ┊327┊  },
+┊   ┊328┊  {
+┊   ┊329┊    id: 7,
+┊   ┊330┊    createdAt: moment().subtract(4, 'weeks').toDate(),
+┊   ┊331┊    name: null,
+┊   ┊332┊    picture: null,
+┊   ┊333┊    allTimeMemberIds: [2, 1],
+┊   ┊334┊    listingMemberIds: [2],
+┊   ┊335┊    adminIds: null,
+┊   ┊336┊    ownerId: null,
+┊   ┊337┊    messages: [],
+┊   ┊338┊  },
+┊   ┊339┊  {
+┊   ┊340┊    id: 8,
+┊   ┊341┊    createdAt: moment().subtract(3, 'weeks').toDate(),
+┊   ┊342┊    name: 'A user 0 group',
+┊   ┊343┊    picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊344┊    allTimeMemberIds: [1, 3, 4, 6],
+┊   ┊345┊    listingMemberIds: [1, 3, 4, 6],
+┊   ┊346┊    actualGroupMemberIds: [1, 4, 6],
+┊   ┊347┊    adminIds: [1, 6],
+┊   ┊348┊    ownerId: 1,
+┊   ┊349┊    messages: [
+┊   ┊350┊      {
+┊   ┊351┊        id: 1,
+┊   ┊352┊        chatId: 8,
+┊   ┊353┊        senderId: 1,
+┊   ┊354┊        content: 'I made a group',
+┊   ┊355┊        createdAt: moment().subtract(2, 'weeks').toDate(),
+┊   ┊356┊        type: MessageType.TEXT,
+┊   ┊357┊        recipients: [
+┊   ┊358┊          {
+┊   ┊359┊            userId: 3,
+┊   ┊360┊            messageId: 1,
+┊   ┊361┊            chatId: 8,
+┊   ┊362┊            receivedAt: null,
+┊   ┊363┊            readAt: null,
+┊   ┊364┊          },
+┊   ┊365┊          {
+┊   ┊366┊            userId: 4,
+┊   ┊367┊            messageId: 1,
+┊   ┊368┊            chatId: 8,
+┊   ┊369┊            receivedAt: moment().subtract(2, 'weeks').add(1, 'minutes').toDate(),
+┊   ┊370┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').toDate(),
+┊   ┊371┊          },
+┊   ┊372┊          {
+┊   ┊373┊            userId: 6,
+┊   ┊374┊            messageId: 1,
+┊   ┊375┊            chatId: 8,
+┊   ┊376┊            receivedAt: null,
+┊   ┊377┊            readAt: null,
+┊   ┊378┊          },
+┊   ┊379┊        ],
+┊   ┊380┊        holderIds: [1, 3, 4, 6],
+┊   ┊381┊      },
+┊   ┊382┊      {
+┊   ┊383┊        id: 2,
+┊   ┊384┊        chatId: 8,
+┊   ┊385┊        senderId: 1,
+┊   ┊386┊        content: 'Ops, user 3 was not supposed to be here',
+┊   ┊387┊        createdAt: moment().subtract(2, 'weeks').add(2, 'minutes').toDate(),
+┊   ┊388┊        type: MessageType.TEXT,
+┊   ┊389┊        recipients: [
+┊   ┊390┊          {
+┊   ┊391┊            userId: 4,
+┊   ┊392┊            messageId: 2,
+┊   ┊393┊            chatId: 8,
+┊   ┊394┊            receivedAt: moment().subtract(2, 'weeks').add(3, 'minutes').toDate(),
+┊   ┊395┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').toDate(),
+┊   ┊396┊          },
+┊   ┊397┊          {
+┊   ┊398┊            userId: 6,
+┊   ┊399┊            messageId: 2,
+┊   ┊400┊            chatId: 8,
+┊   ┊401┊            receivedAt: null,
+┊   ┊402┊            readAt: null,
+┊   ┊403┊          },
+┊   ┊404┊        ],
+┊   ┊405┊        holderIds: [1, 4, 6],
+┊   ┊406┊      },
+┊   ┊407┊      {
+┊   ┊408┊        id: 3,
+┊   ┊409┊        chatId: 8,
+┊   ┊410┊        senderId: 4,
+┊   ┊411┊        content: 'Awesome!',
+┊   ┊412┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
+┊   ┊413┊        type: MessageType.TEXT,
+┊   ┊414┊        recipients: [
+┊   ┊415┊          {
+┊   ┊416┊            userId: 1,
+┊   ┊417┊            messageId: 3,
+┊   ┊418┊            chatId: 8,
+┊   ┊419┊            receivedAt: null,
+┊   ┊420┊            readAt: null,
+┊   ┊421┊          },
+┊   ┊422┊          {
+┊   ┊423┊            userId: 6,
+┊   ┊424┊            messageId: 3,
+┊   ┊425┊            chatId: 8,
+┊   ┊426┊            receivedAt: null,
+┊   ┊427┊            readAt: null,
+┊   ┊428┊          },
+┊   ┊429┊        ],
+┊   ┊430┊        holderIds: [1, 4, 6],
+┊   ┊431┊      },
+┊   ┊432┊    ],
+┊   ┊433┊  },
+┊   ┊434┊  {
+┊   ┊435┊    id: 9,
+┊   ┊436┊    createdAt: moment().subtract(2, 'weeks').toDate(),
+┊   ┊437┊    name: 'A user 5 group',
+┊   ┊438┊    picture: null,
+┊   ┊439┊    allTimeMemberIds: [6, 3],
+┊   ┊440┊    listingMemberIds: [6, 3],
+┊   ┊441┊    actualGroupMemberIds: [6, 3],
+┊   ┊442┊    adminIds: [6],
+┊   ┊443┊    ownerId: 6,
+┊   ┊444┊    messages: [],
+┊   ┊445┊  },
+┊   ┊446┊];
+┊   ┊447┊
+┊   ┊448┊export const db = {users, chats};
```

[}]: #

Its' finally time to create our schema

[{]: <helper> (diffStep "1.3" files="schema/typeDefs.ts" module="server")

#### Step 1.3: Add resolvers and schema

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -1,2 +1,76 @@
 ┊ 1┊ 1┊export default `
+┊  ┊ 2┊  scalar Date
+┊  ┊ 3┊
+┊  ┊ 4┊  type Query {
+┊  ┊ 5┊    me: User
+┊  ┊ 6┊    users: [User!]
+┊  ┊ 7┊    chats: [Chat!]!
+┊  ┊ 8┊    chat(chatId: ID!): Chat
+┊  ┊ 9┊  }
+┊  ┊10┊
+┊  ┊11┊  enum MessageType {
+┊  ┊12┊    LOCATION
+┊  ┊13┊    TEXT
+┊  ┊14┊    PICTURE
+┊  ┊15┊  }
+┊  ┊16┊
+┊  ┊17┊  type Chat {
+┊  ┊18┊    #May be a chat or a group
+┊  ┊19┊    id: ID!
+┊  ┊20┊    createdAt: Date!
+┊  ┊21┊    #Computed for chats
+┊  ┊22┊    name: String
+┊  ┊23┊    #Computed for chats
+┊  ┊24┊    picture: String
+┊  ┊25┊    #All members, current and past ones. Includes users who still didn't get the chat listed.
+┊  ┊26┊    allTimeMembers: [User!]!
+┊  ┊27┊    #Whoever gets the chat listed. For groups includes past members who still didn't delete the group. For chats they are the only ones who can send messages.
+┊  ┊28┊    listingMembers: [User!]!
+┊  ┊29┊    #Actual members of the group. Null for chats. For groups they are the only ones who can send messages. They aren't the only ones who get the group listed.
+┊  ┊30┊    actualGroupMembers: [User!]
+┊  ┊31┊    #Null for chats
+┊  ┊32┊    admins: [User!]
+┊  ┊33┊    #If null the group is read-only. Null for chats.
+┊  ┊34┊    owner: User
+┊  ┊35┊    #Computed property
+┊  ┊36┊    isGroup: Boolean!
+┊  ┊37┊    messages(amount: Int): [Message]!
+┊  ┊38┊    #Computed property
+┊  ┊39┊    lastMessage: Message
+┊  ┊40┊    #Computed property
+┊  ┊41┊    updatedAt: Date!
+┊  ┊42┊    #Computed property
+┊  ┊43┊    unreadMessages: Int!
+┊  ┊44┊  }
+┊  ┊45┊
+┊  ┊46┊  type Message {
+┊  ┊47┊    id: ID!
+┊  ┊48┊    sender: User!
+┊  ┊49┊    chat: Chat!
+┊  ┊50┊    content: String!
+┊  ┊51┊    createdAt: Date!
+┊  ┊52┊    #FIXME: should return MessageType
+┊  ┊53┊    type: Int!
+┊  ┊54┊    #Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise
+┊  ┊55┊    holders: [User!]!
+┊  ┊56┊    #Computed property
+┊  ┊57┊    ownership: Boolean!
+┊  ┊58┊    #Whoever received the message
+┊  ┊59┊    recipients: [Recipient!]!
+┊  ┊60┊  }
+┊  ┊61┊
+┊  ┊62┊  type Recipient {
+┊  ┊63┊    user: User!
+┊  ┊64┊    message: Message!
+┊  ┊65┊    chat: Chat!
+┊  ┊66┊    receivedAt: Date
+┊  ┊67┊    readAt: Date
+┊  ┊68┊  }
+┊  ┊69┊
+┊  ┊70┊  type User {
+┊  ┊71┊    id: ID!
+┊  ┊72┊    name: String
+┊  ┊73┊    picture: String
+┊  ┊74┊    phone: String
+┊  ┊75┊  }
 ┊ 2┊76┊`;
```

[}]: #

and our resolvers

[{]: <helper> (diffStep "1.3" files="schema/resolvers.ts" module="server")

#### Step 1.3: Add resolvers and schema

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,5 +1,65 @@
 ┊ 1┊ 1┊import { IResolvers } from 'apollo-server-express';
+┊  ┊ 2┊import { GraphQLDateTime } from 'graphql-iso-date';
+┊  ┊ 3┊import { ChatDb, db, MessageDb, RecipientDb, UserDb } from "../db";
+┊  ┊ 4┊
+┊  ┊ 5┊let users = db.users;
+┊  ┊ 6┊let chats = db.chats;
+┊  ┊ 7┊const currentUser = users[0];
 ┊ 2┊ 8┊
 ┊ 3┊ 9┊export const resolvers: IResolvers = {
-┊ 4┊  ┊  Query: {},
+┊  ┊10┊  Date: GraphQLDateTime,
+┊  ┊11┊  Query: {
+┊  ┊12┊    me: (): UserDb => currentUser,
+┊  ┊13┊    users: (): UserDb[] => users.filter(user => user.id !== currentUser.id),
+┊  ┊14┊    chats: (): ChatDb[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
+┊  ┊15┊    chat: (obj: any, {chatId}): ChatDb | null => chats.find(chat => chat.id === chatId) || null,
+┊  ┊16┊  },
+┊  ┊17┊  Chat: {
+┊  ┊18┊    name: (chat: ChatDb): string => chat.name ? chat.name : users
+┊  ┊19┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
+┊  ┊20┊    picture: (chat: ChatDb) => chat.name ? chat.picture : users
+┊  ┊21┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.picture,
+┊  ┊22┊    allTimeMembers: (chat: ChatDb): UserDb[] => users.filter(user => chat.allTimeMemberIds.includes(user.id)),
+┊  ┊23┊    listingMembers: (chat: ChatDb): UserDb[] => users.filter(user => chat.listingMemberIds.includes(user.id)),
+┊  ┊24┊    actualGroupMembers: (chat: ChatDb): UserDb[] => users.filter(user => chat.actualGroupMemberIds && chat.actualGroupMemberIds.includes(user.id)),
+┊  ┊25┊    admins: (chat: ChatDb): UserDb[] => users.filter(user => chat.adminIds && chat.adminIds.includes(user.id)),
+┊  ┊26┊    owner: (chat: ChatDb): UserDb | null => users.find(user => chat.ownerId === user.id) || null,
+┊  ┊27┊    isGroup: (chat: ChatDb): boolean => !!chat.name,
+┊  ┊28┊    messages: (chat: ChatDb, {amount = 0}: {amount: number}): MessageDb[] => {
+┊  ┊29┊      const messages = chat.messages
+┊  ┊30┊      .filter(message => message.holderIds.includes(currentUser.id))
+┊  ┊31┊      .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf()) || <MessageDb[]>[];
+┊  ┊32┊      return (amount ? messages.slice(0, amount) : messages).reverse();
+┊  ┊33┊    },
+┊  ┊34┊    lastMessage: (chat: ChatDb): MessageDb => {
+┊  ┊35┊      return chat.messages
+┊  ┊36┊        .filter(message => message.holderIds.includes(currentUser.id))
+┊  ┊37┊        .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf())[0] || null;
+┊  ┊38┊    },
+┊  ┊39┊    updatedAt: (chat: ChatDb): Date => {
+┊  ┊40┊      const lastMessage = chat.messages
+┊  ┊41┊        .filter(message => message.holderIds.includes(currentUser.id))
+┊  ┊42┊        .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf())[0];
+┊  ┊43┊
+┊  ┊44┊      return lastMessage ? lastMessage.createdAt : chat.createdAt;
+┊  ┊45┊    },
+┊  ┊46┊    unreadMessages: (chat: ChatDb): number => chat.messages
+┊  ┊47┊      .filter(message => message.holderIds.includes(currentUser.id) &&
+┊  ┊48┊        message.recipients.find(recipient => recipient.userId === currentUser.id && !recipient.readAt))
+┊  ┊49┊      .length,
+┊  ┊50┊  },
+┊  ┊51┊  Message: {
+┊  ┊52┊    chat: (message: MessageDb): ChatDb | null => chats.find(chat => message.chatId === chat.id) || null,
+┊  ┊53┊    sender: (message: MessageDb): UserDb | null => users.find(user => user.id === message.senderId) || null,
+┊  ┊54┊    holders: (message: MessageDb): UserDb[] => users.filter(user => message.holderIds.includes(user.id)),
+┊  ┊55┊    ownership: (message: MessageDb): boolean => message.senderId === currentUser.id,
+┊  ┊56┊  },
+┊  ┊57┊  Recipient: {
+┊  ┊58┊    user: (recipient: RecipientDb): UserDb | null => users.find(user => recipient.userId === user.id) || null,
+┊  ┊59┊    message: (recipient: RecipientDb): MessageDb | null => {
+┊  ┊60┊      const chat = chats.find(chat => recipient.chatId === chat.id);
+┊  ┊61┊      return chat ? chat.messages.find(message => recipient.messageId === message.id) || null : null;
+┊  ┊62┊    },
+┊  ┊63┊    chat: (recipient: RecipientDb): ChatDb | null => chats.find(chat => recipient.chatId === chat.id) || null,
+┊  ┊64┊  },
 ┊ 5┊65┊};
```

[}]: #

Out basic server is already done and working. We still have no way to do any kind of mutation, but we already set up several queries to return a list of users or chats.
In particular we can choose if we want to return all the chats (and how many messages we want to return for each chat) or if we want to return a single chat. We can also choose which and how many properties we want to return for each query.
We can start the server by simply running:

    yarn start

## Client

Now we can concentrate on the client and bootstrap it using Angular-CLI.
First you will need to install it globally with:

    yarn global add @angular/cli

Then we can create a new project from scratch:

    ng new client --style scss

Time to install a couple of packages:

    ng add apollo-angular
    yarn add -D @types/graphql

Apollo's Schematics sets everything for us. The only thing missing is to fill the `url`.

[{]: <helper> (diffStep "1.1" files="src/app/graphql.module.ts" module="client")

#### Step 1.1: Add angular-apollo to app module

##### Added src&#x2F;app&#x2F;graphql.module.ts
```diff
@@ -0,0 +1,25 @@
+┊  ┊ 1┊import { NgModule } from '@angular/core';
+┊  ┊ 2┊import { ApolloModule, APOLLO_OPTIONS } from 'apollo-angular';
+┊  ┊ 3┊import { HttpLinkModule, HttpLink } from 'apollo-angular-link-http';
+┊  ┊ 4┊import { InMemoryCache } from 'apollo-cache-inmemory';
+┊  ┊ 5┊
+┊  ┊ 6┊const uri = 'http://localhost:4000/graphql';
+┊  ┊ 7┊
+┊  ┊ 8┊export function createApollo(httpLink: HttpLink) {
+┊  ┊ 9┊  return {
+┊  ┊10┊    link: httpLink.create({uri}),
+┊  ┊11┊    cache: new InMemoryCache(),
+┊  ┊12┊  };
+┊  ┊13┊}
+┊  ┊14┊
+┊  ┊15┊@NgModule({
+┊  ┊16┊  exports: [ApolloModule, HttpLinkModule],
+┊  ┊17┊  providers: [
+┊  ┊18┊    {
+┊  ┊19┊      provide: APOLLO_OPTIONS,
+┊  ┊20┊      useFactory: createApollo,
+┊  ┊21┊      deps: [HttpLink],
+┊  ┊22┊    },
+┊  ┊23┊  ],
+┊  ┊24┊})
+┊  ┊25┊export class GraphQLModule {}
```

[}]: #

Let's take a closer look what Schematics prepared for us.

First, it created `graphql.module.ts` and used `ApolloModule` and `HttpLinkModule`.

- `ApolloModule` is the center of using GraphQL in your app! It includes all needed services that allows to use ApolloClient’s features.
- `HttpLinkModule` makes it easy to fetch data in Angular.

`HttpLinkModule` is optional, you can replace it with any other Link.
Its biggest advantage of all is that it uses Angular's `HttpClient` internally so it's possible to use it in `NativeScript` or in combination with any other `HttpClient` provider. By using `HttpLinkModule` you get Server-Side Rendering for free, without any additional work.

Next, we got `APOLLO_OPTIONS` provided to Angular.

We're using the `InMemoryCache` cache, but there are several options like `Redux`, `Hermes` and even `ngrx`.

- `apollo-cache-inmemory` is an official and recommended Apollo Cache.

### Cache and Links

These are two main parts of Apollo Stack, both are pluggable and customizable.

- **Cache** is responsible for storing data and keeping it consistent.
- **Links** describe how to access the source of data (usually a GraphQL endpoint).

Let's take a closer look on what cache means for Apollo, we're going to base it on the official `apollo-cache-inmemory`, one of many implementations.

But first, imagine, we write our own amazing Apollo Cache implementation.

```graphql
query getMessages {
  messages {
    id
    content
    likes
  }
}
```

When we execute a query like one above what we get in return is an array of all messages of logged in user:

```json
{
  "data": {
    "messages": [
      {
        "id": 0,
        "content": "Hi",
        "likes": 0
      },
      {
        "id": 1,
        "content": "Hello",
        "likes": 0
      }
    ]
  }
}
```

Our cache stores the result in exact shape as above.

Later on we add more queries and one of them returns a list of messages from one conversation.

Now, a short story. We've got two users, Niccolo and Dotan.

1. Niccolo opens the app and Apollo fetches all of his messages (I know, it's stupid but stay with me)
1. One message is to Dotan, Niccolo said "Hi" to him.
1. Now Dotan opens the app and reacts to the message with a like.
1. After few minutes Niccolo asks for messages from a conversation with Dotan.
1. Apollo fetches those messages and there is one in which Niccolo says "Hi" and it's liked by Dotan.
1. Now we have the same message twice, one has different state that the other, in one place there is no like, in other, we have +1.

As you see, our cache is not smart enough to figure out one query might be a subset of another.
That's because it doesn't get enough informations.

Let's go back to InMemoryCache and see how it describes itself:

> Thanks to InMemoryCache, it's possible for the results of a query or mutation to update the UI in all the right places. In many cases it's possible for that to happen automatically, whereas in others you need to help the client out a little in doing so.

It means it could solve the issue we have with duplicated messages with different state.

It's very important to understand one concept that InMemoryCache is heavily based on, it's called normalization.

The InMemoryCache splits the result into individual objects, creates a unique identifier for each of them and flattens everything up before saving it to the store.
Might seem complicated at first, but as InMemoryCache, let's break everything into smaller parts.

First, "breaking the result into pieces" part. The InMemoryCache tells ApolloClient to add `__typename` field to your query before passing it to the server. I used server for simplicity, it actually passes it to Apollo Links and they are responsible to make a HTTP request. Every Apollo Cache is able to transform a document, that's as a side note.

```graphql
query getMessages {
  messages {
    id
    content
    likes
    __typename
  }
}
```

The `__typename` field tells about the type of the object. It's compatible with every GraphQL server as it's part of the specification.

```graphql
type Message {
  id: ID
  content: String
  likes: Int
}

type Query {
  messages: [Message]
}
```

In our case, `__typename` returns `Message`. So now InMemoryCache knows that a message is a Message. Computers are stupid, we have to tell them everything!

Next, let's take a look at "creates a unique identifier" part.

The InMemoryCache iterates through every object and looks for the `__typename` and two fields, `id` and `_id` that usually are used as identifiers. Taking our example, it turns every Message object into `Message:1`, `Message:2`, `Message:3` and so on. You get the idea.

It's the default behaviour but you can change it as you wish:

```typescript
const cache = new InMemoryCache({
  dataIdFromObject(result) {
    if (result.__typename) {
      const id = result.uid || result.id || result._id;

      if (id !== undefined) {
        return `${result.__typename}:${id}`;
      }
    }
    return null;
  },
});
```

Great, we covered two out of three things InMemoryCache does. Now we're going to explore "flattens everything up".

We're able to understand the type of each individual object and it's unique identifier. It still has to be stored.

What InMemoryCache does is it's saves every element on root level in a plain object where the key contains a unique identifier and the value has a body.

```json
{
  "Message:1": {
    "id": 1,
    "content": "Hi",
    "likes": 1,
    "__typename": "Message"
  },
  "Message:2": {
    "id": 2,
    "content": "Hello",
    "likes": 0,
    "__typename": "Message"
  },
  "ROOT_QUERY": {
    "messages": [
      {
        "id": "Message:1",
        "typename": "Message"
      },
      {
        "id": "Message:2",
        "typename": "Message"
      }
    ]
  }
}
```

Wait, `ROOT_QUERY`? Yes, it has to somehow store the actual result of every executed query so next time you ask for the same thing, it matches it with the query in store and resolves right away.

> You should be aware that the InMemoryCache stringifies variables of a query too, which means the same queries but with different variables are stored separately. Even the order of those variables matters.

As you can see, each element in `ROOT_QUERY.message` is kind of a reference to a message record.

Let's bring back our short story, a scenario.
When Niccolo asks for all messages, the whole three part process is started, each message is saved to the store.
When he asks for history of a conversation with Dotan, InMemoryCache starts the process again and if it sees that there is already a massage with the same unique identifier, it knows they both are the same so it merged them.
The InMemoryCache can tell what changed and emits the new object to the UI.
Thanks to that we no longer have the same message in two places where one has a like and other doesn't.

Uff, Apollo Cache part is done. Thanks for being with us that long, I hope you didn't get bored because there's a lot ahead of us.

Let's talk Apollo Link.

As I mentioned it's kind of a network stack of Apollo. You execute a document with some metadata like variables or a context, Apollo Client receives it and passes it to the chain of Links.

The ApolloLink is based on an Observable (not RxJS) and if you know `RxJS` you're familiar with `pipe`. Each Link is basically a function that receives a requested operation and a function that runs the next Link. Just like a pipe!

```typescript
import { ApolloLink } from 'apollo-link';

const log = new ApolloLink((operation, next) => {
  console.log('Name', operation.operationName);
  return next(operation);
});
```

Now when we add it in `graphql.module.ts` we see the name of each requested operation.

We can even intercept the result.

```typescript
const intercept = new ApolloLink((operation, next) => {
    return forward(operation).map((data) => {
        // data from a previous link
        console.log('data', data);
        return data;
    });
});
```

As you can see below, by network we don't always mean making HTTP requests. It could be Http Link at the end of Links chain or WebSocket Link or even something that works on in-memory data.

```typescript
import { ApolloLink, Observable } from 'apollo-link';

const inmemory = new ApolloLink(operation => {
    return Observable(observer => {
        // function that executes the operation and returns data
        const result = resolveOperation(operation);

        observer.next(result);
        observer.complete();
    });
});
```

You can find tons of helpful Links [on npm](https://www.npmjs.com/search?q=apollo-link-) and [on Apollo Link documentation](https://www.apollographql.com/docs/link/links/community.html).

We highly recommend to take a look at Angular related Links [on Apollo Angular documentation](https://www.apollographql.com/docs/angular/guides/tools-and-packages.html).

### GQL Tag

The `gql` template tag is what you use to define GraphQL queries in your Apollo apps. It parses your GraphQL query into the `GraphQL.js AST format` which may then be consumed by Apollo methods. Whenever Apollo is asking for a GraphQL query you will always want to wrap it in a `gql` template tag.

You can embed a GraphQL document containing only fragments inside of another GraphQL document using template string interpolation. This allows you to use fragments defined in one part of your codebase inside of a query defined in a completely different file.

[{]: <helper> (diffStep "1.2" file="src/graphql/fragment.ts" module="client")

#### Step 1.2: Add chats service

##### Added src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -0,0 +1,25 @@
+┊  ┊ 1┊import {map} from 'rxjs/operators';
+┊  ┊ 2┊import {Apollo} from 'apollo-angular';
+┊  ┊ 3┊import {Injectable} from '@angular/core';
+┊  ┊ 4┊import {getChatsQuery} from '../../graphql/getChats.query';
+┊  ┊ 5┊
+┊  ┊ 6┊@Injectable()
+┊  ┊ 7┊export class ChatsService {
+┊  ┊ 8┊  messagesAmount = 3;
+┊  ┊ 9┊
+┊  ┊10┊  constructor(private apollo: Apollo) {}
+┊  ┊11┊
+┊  ┊12┊  getChats() {
+┊  ┊13┊    const query = this.apollo.watchQuery<any>({
+┊  ┊14┊      query: getChatsQuery,
+┊  ┊15┊      variables: {
+┊  ┊16┊        amount: this.messagesAmount,
+┊  ┊17┊      },
+┊  ┊18┊    });
+┊  ┊19┊    const chats$ = query.valueChanges.pipe(
+┊  ┊20┊      map((result) => result.data.chats)
+┊  ┊21┊    );
+┊  ┊22┊
+┊  ┊23┊    return {query, chats$};
+┊  ┊24┊  }
+┊  ┊25┊}
```

##### Added src&#x2F;graphql&#x2F;fragment.ts
```diff
@@ -0,0 +1,51 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {DocumentNode} from 'graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊export const fragments: {
+┊  ┊ 5┊  [key: string]: DocumentNode
+┊  ┊ 6┊} = {
+┊  ┊ 7┊  chatWithoutMessages: gql`
+┊  ┊ 8┊    fragment ChatWithoutMessages on Chat {
+┊  ┊ 9┊      id
+┊  ┊10┊      name
+┊  ┊11┊      picture
+┊  ┊12┊      allTimeMembers {
+┊  ┊13┊        id
+┊  ┊14┊      }
+┊  ┊15┊      unreadMessages
+┊  ┊16┊      isGroup
+┊  ┊17┊    }
+┊  ┊18┊  `,
+┊  ┊19┊  message: gql`
+┊  ┊20┊    fragment Message on Message {
+┊  ┊21┊      id
+┊  ┊22┊      chat {
+┊  ┊23┊        id
+┊  ┊24┊      }
+┊  ┊25┊      sender {
+┊  ┊26┊        id
+┊  ┊27┊        name
+┊  ┊28┊      }
+┊  ┊29┊      content
+┊  ┊30┊      createdAt
+┊  ┊31┊      type
+┊  ┊32┊      recipients {
+┊  ┊33┊        user {
+┊  ┊34┊          id
+┊  ┊35┊        }
+┊  ┊36┊        message {
+┊  ┊37┊          id
+┊  ┊38┊          chat {
+┊  ┊39┊            id
+┊  ┊40┊          }
+┊  ┊41┊        }
+┊  ┊42┊        chat {
+┊  ┊43┊          id
+┊  ┊44┊        }
+┊  ┊45┊        receivedAt
+┊  ┊46┊        readAt
+┊  ┊47┊      }
+┊  ┊48┊      ownership
+┊  ┊49┊    }
+┊  ┊50┊  `,
+┊  ┊51┊};
```

##### Added src&#x2F;graphql&#x2F;getChats.query.ts
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const getChatsQuery = gql`
+┊  ┊ 6┊  query GetChats($amount: Int) {
+┊  ┊ 7┊    chats {
+┊  ┊ 8┊      ...ChatWithoutMessages
+┊  ┊ 9┊      messages(amount: $amount) {
+┊  ┊10┊        ...Message
+┊  ┊11┊      }
+┊  ┊12┊    }
+┊  ┊13┊  }
+┊  ┊14┊
+┊  ┊15┊  ${fragments['chatWithoutMessages']}
+┊  ┊16┊  ${fragments['message']}
+┊  ┊17┊`;
```

[}]: #

### Use Apollo service

Let's create a simple service to query the chats from our just created server:

[{]: <helper> (diffStep "1.2" files="src/app/services" module="client")

#### Step 1.2: Add chats service

##### Added src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -0,0 +1,25 @@
+┊  ┊ 1┊import {map} from 'rxjs/operators';
+┊  ┊ 2┊import {Apollo} from 'apollo-angular';
+┊  ┊ 3┊import {Injectable} from '@angular/core';
+┊  ┊ 4┊import {getChatsQuery} from '../../graphql/getChats.query';
+┊  ┊ 5┊
+┊  ┊ 6┊@Injectable()
+┊  ┊ 7┊export class ChatsService {
+┊  ┊ 8┊  messagesAmount = 3;
+┊  ┊ 9┊
+┊  ┊10┊  constructor(private apollo: Apollo) {}
+┊  ┊11┊
+┊  ┊12┊  getChats() {
+┊  ┊13┊    const query = this.apollo.watchQuery<any>({
+┊  ┊14┊      query: getChatsQuery,
+┊  ┊15┊      variables: {
+┊  ┊16┊        amount: this.messagesAmount,
+┊  ┊17┊      },
+┊  ┊18┊    });
+┊  ┊19┊    const chats$ = query.valueChanges.pipe(
+┊  ┊20┊      map((result) => result.data.chats)
+┊  ┊21┊    );
+┊  ┊22┊
+┊  ┊23┊    return {query, chats$};
+┊  ┊24┊  }
+┊  ┊25┊}
```

[}]: #

We just learned how to use Apollo to attach GraphQL query results to the Angular UI.

The `watchQuery` method returns a `QueryRef` object which has the `valueChanges` property that is an `Observable`.

Every fetched data is stored in Apollo Client’s cache, so if some other query fetches new information about the chats, this component will update to remain consistent.

It’s also possible to fetch data only once. You simply use `query` method instead of `watchQuery`. The query method of Apollo service returns an Observable too that also resolves with the same result as above.

#### But what is a `QueryRef`?

As you know, `query` method returns an Observable that emits a result, just once, then the observable completes. `watchQuery` works a bit different. It fetches data then stays open and listens to the Cache for updates.

So why `watchQuery` can not simply expose an Observable and it serves `QueryRef` instead?

Because Apollo Angular is a bridge between Apollo Client (core library) and Angular framework it has to operate on RxJS based Observables. RxJS Observable has a simple API that cannot be easily extended and it's in general a bad practice.

Why we're talking about those things? Apollo Client exposes an Observable that has bunch of custom method on it. They are super useful and their purpose is to talk to Apollo through them.

Since we have to turn Apollo's Observable to something that is compatible with RxJS we decided to expose an object that has all those methods and `valueChanges` property that is a regular RxJS Observable.

#### Make our app pretty

We will use Materials for the UI, so let's install it:

    yarn add @angular/cdk @angular/material hammerjs ng2-truncate

Let's configure Material:

[{]: <helper> (diffStep "1.3" files="src/index.ts, src/main.ts, src/styles.scss" module="client")

#### Step 1.3: List the chats

##### Changed src&#x2F;main.ts
```diff
@@ -4,6 +4,9 @@
 ┊ 4┊ 4┊import { AppModule } from './app/app.module';
 ┊ 5┊ 5┊import { environment } from './environments/environment';
 ┊ 6┊ 6┊
+┊  ┊ 7┊// Material gestures
+┊  ┊ 8┊import 'hammerjs';
+┊  ┊ 9┊
 ┊ 7┊10┊if (environment.production) {
 ┊ 8┊11┊  enableProdMode();
 ┊ 9┊12┊}
```

##### Changed src&#x2F;styles.scss
```diff
@@ -1 +1,56 @@
 ┊ 1┊ 1┊/* You can add global styles to this file, and also import other style files */
+┊  ┊ 2┊
+┊  ┊ 3┊/* Meterial theme */
+┊  ┊ 4┊@import "~@angular/material/prebuilt-themes/indigo-pink.css";
+┊  ┊ 5┊
+┊  ┊ 6┊:root {
+┊  ┊ 7┊  --primary: #2c6157;
+┊  ┊ 8┊  --secondary: #6fd056;
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊body {
+┊  ┊12┊  margin: 0;
+┊  ┊13┊  font-family: Roboto, "Helvetica Neue", sans-serif;
+┊  ┊14┊}
+┊  ┊15┊
+┊  ┊16┊.mat-primary {
+┊  ┊17┊  background-color: var(--primary) !important;
+┊  ┊18┊  color: white;
+┊  ┊19┊}
+┊  ┊20┊
+┊  ┊21┊.mat-secondary {
+┊  ┊22┊  background-color: var(--secondary) !important;
+┊  ┊23┊  color: white;
+┊  ┊24┊}
+┊  ┊25┊
+┊  ┊26┊.mat-primary-negative {
+┊  ┊27┊  background-color: white !important;
+┊  ┊28┊  color: var(--primary);
+┊  ┊29┊}
+┊  ┊30┊
+┊  ┊31┊.mat-secondary-negative {
+┊  ┊32┊  background-color: white !important;
+┊  ┊33┊  color: var(--secondary);
+┊  ┊34┊}
+┊  ┊35┊
+┊  ┊36┊.mat-default {
+┊  ┊37┊  background-color: transparent !important;
+┊  ┊38┊  color: black;
+┊  ┊39┊}
+┊  ┊40┊
+┊  ┊41┊.mat-toolbar {
+┊  ┊42┊  color: white;
+┊  ┊43┊  background-color: var(--primary) !important;
+┊  ┊44┊}
+┊  ┊45┊
+┊  ┊46┊.mat-focused .mat-form-field-label {
+┊  ┊47┊  color: var(--primary) !important;
+┊  ┊48┊}
+┊  ┊49┊
+┊  ┊50┊.mat-form-field-ripple {
+┊  ┊51┊  background-color: var(--primary) !important;
+┊  ┊52┊}
+┊  ┊53┊
+┊  ┊54┊.mat-input-element {
+┊  ┊55┊  caret-color: var(--primary) !important;
+┊  ┊56┊}🚫↵
```

[}]: #

We're now creating a `shared` module where we will define our header component where we're going to project a different content from each component:

[{]: <helper> (diffStep "1.3" files="src/app/shared/*" module="client")

#### Step 1.3: List the chats

##### Added src&#x2F;app&#x2F;shared&#x2F;components&#x2F;toolbar&#x2F;toolbar.component.scss
```diff
@@ -0,0 +1,15 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  height: 8vh;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊.mat-toolbar {
+┊  ┊ 7┊  justify-content: space-between;
+┊  ┊ 8┊  height: 100%;
+┊  ┊ 9┊
+┊  ┊10┊  .left-block {
+┊  ┊11┊    display: flex;
+┊  ┊12┊    height: 8vh;
+┊  ┊13┊    line-height: 8vh;
+┊  ┊14┊  }
+┊  ┊15┊}
```

##### Added src&#x2F;app&#x2F;shared&#x2F;components&#x2F;toolbar&#x2F;toolbar.component.ts
```diff
@@ -0,0 +1,19 @@
+┊  ┊ 1┊import {Component} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-toolbar',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <mat-toolbar>
+┊  ┊ 7┊      <div class="left-block">
+┊  ┊ 8┊        <ng-content select=".navigation"></ng-content>
+┊  ┊ 9┊        <ng-content select=".profile-pic"></ng-content>
+┊  ┊10┊        <ng-content select=".title"></ng-content>
+┊  ┊11┊      </div>
+┊  ┊12┊      <ng-content select=".menu"></ng-content>
+┊  ┊13┊    </mat-toolbar>
+┊  ┊14┊  `,
+┊  ┊15┊  styleUrls: ['./toolbar.component.scss']
+┊  ┊16┊})
+┊  ┊17┊export class ToolbarComponent {
+┊  ┊18┊
+┊  ┊19┊}
```

##### Added src&#x2F;app&#x2F;shared&#x2F;shared.module.ts
```diff
@@ -0,0 +1,28 @@
+┊  ┊ 1┊import {BrowserModule} from '@angular/platform-browser';
+┊  ┊ 2┊import {NgModule} from '@angular/core';
+┊  ┊ 3┊
+┊  ┊ 4┊import {MatToolbarModule} from '@angular/material';
+┊  ┊ 5┊import {ToolbarComponent} from './components/toolbar/toolbar.component';
+┊  ┊ 6┊import {FormsModule} from '@angular/forms';
+┊  ┊ 7┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 8┊
+┊  ┊ 9┊@NgModule({
+┊  ┊10┊  declarations: [
+┊  ┊11┊    ToolbarComponent,
+┊  ┊12┊  ],
+┊  ┊13┊  imports: [
+┊  ┊14┊    BrowserModule,
+┊  ┊15┊    // Material
+┊  ┊16┊    MatToolbarModule,
+┊  ┊17┊    // Animations
+┊  ┊18┊    BrowserAnimationsModule,
+┊  ┊19┊    // Forms
+┊  ┊20┊    FormsModule,
+┊  ┊21┊  ],
+┊  ┊22┊  providers: [],
+┊  ┊23┊  exports: [
+┊  ┊24┊    ToolbarComponent,
+┊  ┊25┊  ],
+┊  ┊26┊})
+┊  ┊27┊export class SharedModule {
+┊  ┊28┊}
```

[}]: #

Now we want to create the `chats-lister` module, with a container component called `ChatsComponent` and a couple of presentational components. Each and every chat item would be presented with a default profile picture as a placeholder, so before we proceed we will have to download it to the `src/assets` directory:

    $ wget https://raw.githubusercontent.com/Urigo/whatsapp-client-angularcli-material/4f4497df6187c6cf42bb01d7c516db7f9f08e32c/src/assets/default-profile-pic.jpg src/assets

And now we can proceed to creating the relevant components and their style sheets:

[{]: <helper> (diffStep "1.3" files="src/app/chats-lister/*" module="client")

#### Step 1.3: List the chats

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;chats-lister.module.ts
```diff
@@ -0,0 +1,46 @@
+┊  ┊ 1┊import { BrowserModule } from '@angular/platform-browser';
+┊  ┊ 2┊import { NgModule } from '@angular/core';
+┊  ┊ 3┊
+┊  ┊ 4┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 5┊import {MatButtonModule, MatIconModule, MatListModule, MatMenuModule} from '@angular/material';
+┊  ┊ 6┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊ 7┊import {FormsModule} from '@angular/forms';
+┊  ┊ 8┊import {ChatsService} from '../services/chats.service';
+┊  ┊ 9┊import {ChatItemComponent} from './components/chat-item/chat-item.component';
+┊  ┊10┊import {ChatsComponent} from './containers/chats/chats.component';
+┊  ┊11┊import {ChatsListComponent} from './components/chats-list/chats-list.component';
+┊  ┊12┊import {SharedModule} from '../shared/shared.module';
+┊  ┊13┊
+┊  ┊14┊const routes: Routes = [
+┊  ┊15┊  {path: '', redirectTo: 'chats', pathMatch: 'full'},
+┊  ┊16┊  {path: 'chats', component: ChatsComponent},
+┊  ┊17┊];
+┊  ┊18┊
+┊  ┊19┊@NgModule({
+┊  ┊20┊  declarations: [
+┊  ┊21┊    ChatsComponent,
+┊  ┊22┊    ChatsListComponent,
+┊  ┊23┊    ChatItemComponent,
+┊  ┊24┊  ],
+┊  ┊25┊  imports: [
+┊  ┊26┊    BrowserModule,
+┊  ┊27┊    // Material
+┊  ┊28┊    MatMenuModule,
+┊  ┊29┊    MatIconModule,
+┊  ┊30┊    MatButtonModule,
+┊  ┊31┊    MatListModule,
+┊  ┊32┊    // Animations
+┊  ┊33┊    BrowserAnimationsModule,
+┊  ┊34┊    // Routing
+┊  ┊35┊    RouterModule.forChild(routes),
+┊  ┊36┊    // Forms
+┊  ┊37┊    FormsModule,
+┊  ┊38┊    // Feature modules
+┊  ┊39┊    SharedModule,
+┊  ┊40┊  ],
+┊  ┊41┊  providers: [
+┊  ┊42┊    ChatsService,
+┊  ┊43┊  ],
+┊  ┊44┊})
+┊  ┊45┊export class ChatsListerModule {
+┊  ┊46┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.scss
```diff
@@ -0,0 +1,45 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  width: 100%;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊.chat-row {
+┊  ┊ 7┊  margin: 10px 0 0 10px;
+┊  ┊ 8┊  height: 60px;
+┊  ┊ 9┊  display: flex;
+┊  ┊10┊
+┊  ┊11┊  .chat-pic {
+┊  ┊12┊    height: 50px;
+┊  ┊13┊    width: auto;
+┊  ┊14┊    border-radius: 50%;
+┊  ┊15┊  }
+┊  ┊16┊
+┊  ┊17┊  .chat-info {
+┊  ┊18┊    width: calc(100% - 60px);
+┊  ┊19┊    height: 100%;
+┊  ┊20┊    margin-left: 10px;
+┊  ┊21┊    border-bottom: .5px solid silver;
+┊  ┊22┊    position: relative;
+┊  ┊23┊
+┊  ┊24┊    .chat-name {
+┊  ┊25┊      margin-top: 5px;
+┊  ┊26┊    }
+┊  ┊27┊
+┊  ┊28┊    .chat-last-message {
+┊  ┊29┊      color: gray;
+┊  ┊30┊      font-size: 15px;
+┊  ┊31┊      margin-top: 5px;
+┊  ┊32┊      white-space: nowrap;
+┊  ┊33┊      text-overflow: ellipsis;
+┊  ┊34┊      overflow: hidden;
+┊  ┊35┊    }
+┊  ┊36┊
+┊  ┊37┊    .chat-timestamp {
+┊  ┊38┊      position: absolute;
+┊  ┊39┊      color: gray;
+┊  ┊40┊      top: 5px;
+┊  ┊41┊      right: 5px;
+┊  ┊42┊      font-size: 13px;
+┊  ┊43┊    }
+┊  ┊44┊  }
+┊  ┊45┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -0,0 +1,21 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-chat-item',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <div class="chat-row">
+┊  ┊ 7┊      <img class="chat-pic" [src]="chat.picture || 'assets/default-profile-pic.jpg'">
+┊  ┊ 8┊      <div class="chat-info">
+┊  ┊ 9┊        <div class="chat-name">{{ chat.name }}</div>
+┊  ┊10┊        <div class="chat-last-message">{{ chat.messages[chat.messages.length - 1]?.content }}</div>
+┊  ┊11┊        <div class="chat-timestamp">00:00</div>
+┊  ┊12┊      </div>
+┊  ┊13┊    </div>
+┊  ┊14┊  `,
+┊  ┊15┊  styleUrls: ['chat-item.component.scss'],
+┊  ┊16┊})
+┊  ┊17┊export class ChatItemComponent {
+┊  ┊18┊  // tslint:disable-next-line:no-input-rename
+┊  ┊19┊  @Input('item')
+┊  ┊20┊  chat: any;
+┊  ┊21┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chats-list&#x2F;chats-list.component.scss
```diff
@@ -0,0 +1,13 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  height: 60px;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊:host ::ng-deep .mat-list-item-content:first-child {
+┊  ┊ 7┊  padding-left: 0 !important;
+┊  ┊ 8┊  display: flow-root !important;
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊mat-list-item {
+┊  ┊12┊  height: 75px !important;
+┊  ┊13┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chats-list&#x2F;chats-list.component.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-chats-list',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <mat-list>
+┊  ┊ 7┊      <mat-list-item *ngFor="let chat of chats">
+┊  ┊ 8┊        <app-chat-item [item]="chat"></app-chat-item>
+┊  ┊ 9┊      </mat-list-item>
+┊  ┊10┊    </mat-list>
+┊  ┊11┊  `,
+┊  ┊12┊  styleUrls: ['chats-list.component.scss'],
+┊  ┊13┊})
+┊  ┊14┊export class ChatsListComponent {
+┊  ┊15┊  // tslint:disable-next-line:no-input-rename
+┊  ┊16┊  @Input('items')
+┊  ┊17┊  chats: any[];
+┊  ┊18┊
+┊  ┊19┊  constructor() {}
+┊  ┊20┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.scss
```diff
@@ -0,0 +1,5 @@
+┊ ┊1┊.chat-button {
+┊ ┊2┊  position: absolute;
+┊ ┊3┊  bottom: 5vw;
+┊ ┊4┊  right: 5vw;
+┊ ┊5┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.ts
```diff
@@ -0,0 +1,46 @@
+┊  ┊ 1┊import {Component, OnInit} from '@angular/core';
+┊  ┊ 2┊import {ChatsService} from '../../../services/chats.service';
+┊  ┊ 3┊import {Observable} from 'rxjs';
+┊  ┊ 4┊
+┊  ┊ 5┊@Component({
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <app-toolbar>
+┊  ┊ 8┊      <div class="title">Whatsapp Clone</div>
+┊  ┊ 9┊      <button mat-icon-button [matMenuTriggerFor]="menu" class="menu">
+┊  ┊10┊        <mat-icon>more_vert</mat-icon>
+┊  ┊11┊      </button>
+┊  ┊12┊    </app-toolbar>
+┊  ┊13┊
+┊  ┊14┊    <mat-menu #menu="matMenu">
+┊  ┊15┊      <button mat-menu-item>
+┊  ┊16┊        <mat-icon>dialpad</mat-icon>
+┊  ┊17┊        <span>Redial</span>
+┊  ┊18┊      </button>
+┊  ┊19┊      <button mat-menu-item disabled>
+┊  ┊20┊        <mat-icon>voicemail</mat-icon>
+┊  ┊21┊        <span>Check voicemail</span>
+┊  ┊22┊      </button>
+┊  ┊23┊      <button mat-menu-item>
+┊  ┊24┊        <mat-icon>notifications_off</mat-icon>
+┊  ┊25┊        <span>Disable alerts</span>
+┊  ┊26┊      </button>
+┊  ┊27┊    </mat-menu>
+┊  ┊28┊
+┊  ┊29┊    <app-chats-list [items]="chats$ | async"></app-chats-list>
+┊  ┊30┊
+┊  ┊31┊    <button class="chat-button" mat-fab color="secondary">
+┊  ┊32┊      <mat-icon aria-label="Icon-button with a + icon">chat</mat-icon>
+┊  ┊33┊    </button>
+┊  ┊34┊  `,
+┊  ┊35┊  styleUrls: ['./chats.component.scss'],
+┊  ┊36┊})
+┊  ┊37┊export class ChatsComponent implements OnInit {
+┊  ┊38┊  chats$: Observable<any[]>;
+┊  ┊39┊
+┊  ┊40┊  constructor(private chatsService: ChatsService) {
+┊  ┊41┊  }
+┊  ┊42┊
+┊  ┊43┊  ngOnInit() {
+┊  ┊44┊    this.chats$ = this.chatsService.getChats().chats$;
+┊  ┊45┊  }
+┊  ┊46┊}
```

[}]: #

Finally let's wire everything up to the main module:

[{]: <helper> (diffStep "1.3" files="src/app/app.component.ts, src/app/app.module.ts" module="client")

#### Step 1.3: List the chats

##### Changed src&#x2F;app&#x2F;app.component.ts
```diff
@@ -2,7 +2,9 @@
 ┊ 2┊ 2┊
 ┊ 3┊ 3┊@Component({
 ┊ 4┊ 4┊  selector: 'app-root',
-┊ 5┊  ┊  templateUrl: './app.component.html',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <router-outlet></router-outlet>
+┊  ┊ 7┊  `,
 ┊ 6┊ 8┊  styleUrls: ['./app.component.scss']
 ┊ 7┊ 9┊})
 ┊ 8┊10┊export class AppComponent {
```

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -4,6 +4,9 @@
 ┊ 4┊ 4┊import { AppComponent } from './app.component';
 ┊ 5┊ 5┊import { HttpClientModule } from '@angular/common/http';
 ┊ 6┊ 6┊import { GraphQLModule } from './graphql.module';
+┊  ┊ 7┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
+┊  ┊ 8┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊ 9┊const routes: Routes = [];
 ┊ 7┊10┊
 ┊ 8┊11┊@NgModule({
 ┊ 9┊12┊  declarations: [
```
```diff
@@ -13,6 +16,10 @@
 ┊13┊16┊    BrowserModule,
 ┊14┊17┊    HttpClientModule,
 ┊15┊18┊    GraphQLModule,
+┊  ┊19┊    // Routing
+┊  ┊20┊    RouterModule.forRoot(routes),
+┊  ┊21┊    // Feature modules
+┊  ┊22┊    ChatsListerModule,
 ┊16┊23┊  ],
 ┊17┊24┊  providers: [],
 ┊18┊25┊  bootstrap: [AppComponent]
```

[}]: #

If you will try to run the frontend you will notice that several messages seems like duplicated, why does it happen?

Since we use NoSQL-like structure in our backend, messages are stored as an array inside each chat so their incremental identifiers are not unique across different chats. We need to normalize them in a way that takes into account both the message id and the chat id:

[{]: <helper> (diffStep "1.4" module="client")

#### Step 1.4: Better normalize messages

##### Changed src&#x2F;app&#x2F;graphql.module.ts
```diff
@@ -1,14 +1,21 @@
 ┊ 1┊ 1┊import { NgModule } from '@angular/core';
 ┊ 2┊ 2┊import { ApolloModule, APOLLO_OPTIONS } from 'apollo-angular';
 ┊ 3┊ 3┊import { HttpLinkModule, HttpLink } from 'apollo-angular-link-http';
-┊ 4┊  ┊import { InMemoryCache } from 'apollo-cache-inmemory';
+┊  ┊ 4┊import { InMemoryCache, defaultDataIdFromObject } from 'apollo-cache-inmemory';
 ┊ 5┊ 5┊
 ┊ 6┊ 6┊const uri = 'http://localhost:4000/graphql';
 ┊ 7┊ 7┊
 ┊ 8┊ 8┊export function createApollo(httpLink: HttpLink) {
 ┊ 9┊ 9┊  return {
 ┊10┊10┊    link: httpLink.create({uri}),
-┊11┊  ┊    cache: new InMemoryCache(),
+┊  ┊11┊    cache: new InMemoryCache({
+┊  ┊12┊      dataIdFromObject: (object: any) => {
+┊  ┊13┊        switch (object.__typename) {
+┊  ┊14┊          case 'Message': return `${object.chat.id}:${object.id}`; // use `chatId` prefix and `messageId` as the primary key
+┊  ┊15┊          default: return defaultDataIdFromObject(object); // fall back to default handling
+┊  ┊16┊        }
+┊  ┊17┊      }
+┊  ┊18┊    }),
 ┊12┊19┊  };
 ┊13┊20┊}
```

[}]: #

That way our application will work even if the backend is a NoSQL. What's even more interesting is that our application will keep working as well even when we will switch our backend to PostgreSQL.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Intro](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/README.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step2.md) |
|:--------------------------------|--------------------------------:|

[}]: #
