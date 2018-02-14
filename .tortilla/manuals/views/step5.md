# Step 5: Chats listing

[//]: # (head-end)


## Server

After the planning phase it's finally time to start writing some real code!
We'll start with the server, so let's install a couple of packages first:

    $ npm install apollo-server-express body-parser cors express graphql
    $ npm install --save-dev @types/body-parser @types/cors @types/express @types/graphql

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
@@ -0,0 +1,10 @@
+┊  ┊ 1┊import { makeExecutableSchema } from 'graphql-tools';
+┊  ┊ 2┊import { typeDefs } from "./typeDefs";
+┊  ┊ 3┊import { resolvers } from "./resolvers";
+┊  ┊ 4┊import { IExecutableSchemaDefinition } from "graphql-tools/dist/Interfaces";
+┊  ┊ 5┊import { GraphQLSchema } from "graphql";
+┊  ┊ 6┊
+┊  ┊ 7┊export const schema: GraphQLSchema = makeExecutableSchema(<IExecutableSchemaDefinition>{
+┊  ┊ 8┊  typeDefs,
+┊  ┊ 9┊  resolvers,
+┊  ┊10┊});🚫↵
```

##### Added schema&#x2F;resolvers.ts
```diff
@@ -0,0 +1,6 @@
+┊ ┊1┊import { IResolvers } from "graphql-tools/dist/Interfaces";
+┊ ┊2┊
+┊ ┊3┊export const resolvers: IResolvers = {
+┊ ┊4┊  Query: {
+┊ ┊5┊  },
+┊ ┊6┊};
```

##### Added schema&#x2F;typeDefs.ts
```diff
@@ -0,0 +1,4 @@
+┊ ┊1┊import { ITypeDefinitions } from "graphql-tools/dist/Interfaces";
+┊ ┊2┊
+┊ ┊3┊export const typeDefs: ITypeDefinitions = `
+┊ ┊4┊`;
```

[}]: #

Time to create our index:

[{]: <helper> (diffStep "1.1" files="^index.ts" module="server")

#### Step 1.1: Create empty Apollo server

##### Added index.ts
```diff
@@ -0,0 +1,23 @@
+┊  ┊ 1┊import { schema } from "./schema";
+┊  ┊ 2┊import * as bodyParser from "body-parser";
+┊  ┊ 3┊import * as cors from 'cors';
+┊  ┊ 4┊import * as express from 'express';
+┊  ┊ 5┊import { graphiqlExpress, graphqlExpress } from "apollo-server-express";
+┊  ┊ 6┊
+┊  ┊ 7┊const PORT = 3000;
+┊  ┊ 8┊
+┊  ┊ 9┊const app = express();
+┊  ┊10┊
+┊  ┊11┊app.use(cors());
+┊  ┊12┊app.use(bodyParser.json());
+┊  ┊13┊
+┊  ┊14┊app.use('/graphql', graphqlExpress(req => ({
+┊  ┊15┊  schema: schema,
+┊  ┊16┊  context: req,
+┊  ┊17┊})));
+┊  ┊18┊
+┊  ┊19┊app.use('/graphiql', graphiqlExpress({
+┊  ┊20┊  endpointURL: '/graphql',
+┊  ┊21┊}));
+┊  ┊22┊
+┊  ┊23┊app.listen(PORT);
```

[}]: #

Now we want to feed our graphql server with some data. Soon we will need `moment`, so let's install it:

    $ npm install moment

Now we can create a fake db:

[{]: <helper> (diffStep "1.2" files="db.ts" module="server")

#### Step 1.2: Add fake db

##### Added db.ts
```diff
@@ -0,0 +1,438 @@
+┊   ┊  1┊import * as moment from 'moment';
+┊   ┊  2┊
+┊   ┊  3┊export enum MessageType {
+┊   ┊  4┊  PICTURE,
+┊   ┊  5┊  TEXT,
+┊   ┊  6┊  LOCATION,
+┊   ┊  7┊}
+┊   ┊  8┊
+┊   ┊  9┊export interface User {
+┊   ┊ 10┊  id: number,
+┊   ┊ 11┊  username: string,
+┊   ┊ 12┊  password: string,
+┊   ┊ 13┊  name: string,
+┊   ┊ 14┊  picture?: string | null,
+┊   ┊ 15┊  phone?: string | null,
+┊   ┊ 16┊}
+┊   ┊ 17┊
+┊   ┊ 18┊export interface Chat {
+┊   ┊ 19┊  id: number,
+┊   ┊ 20┊  name?: string | null,
+┊   ┊ 21┊  picture?: string | null,
+┊   ┊ 22┊  // All members, current and past ones.
+┊   ┊ 23┊  allTimeMemberIds: number[],
+┊   ┊ 24┊  // Whoever gets the chat listed. For groups includes past members who still didn't delete the group.
+┊   ┊ 25┊  listingMemberIds: number[],
+┊   ┊ 26┊  // Actual members of the group (they are not the only ones who get the group listed). Null for chats.
+┊   ┊ 27┊  actualGroupMemberIds?: number[] | null,
+┊   ┊ 28┊  adminIds?: number[] | null,
+┊   ┊ 29┊  ownerId?: number | null,
+┊   ┊ 30┊  messages: Message[],
+┊   ┊ 31┊}
+┊   ┊ 32┊
+┊   ┊ 33┊export interface Message {
+┊   ┊ 34┊  id: number,
+┊   ┊ 35┊  chatId: number,
+┊   ┊ 36┊  senderId: number,
+┊   ┊ 37┊  content: string,
+┊   ┊ 38┊  createdAt: number,
+┊   ┊ 39┊  type: MessageType,
+┊   ┊ 40┊  recipients: Recipient[],
+┊   ┊ 41┊  holderIds: number[],
+┊   ┊ 42┊}
+┊   ┊ 43┊
+┊   ┊ 44┊export interface Recipient {
+┊   ┊ 45┊  userId: number,
+┊   ┊ 46┊  messageId: number,
+┊   ┊ 47┊  chatId: number,
+┊   ┊ 48┊  receivedAt: number | null,
+┊   ┊ 49┊  readAt: number | null,
+┊   ┊ 50┊}
+┊   ┊ 51┊
+┊   ┊ 52┊const users: User[] = [
+┊   ┊ 53┊  {
+┊   ┊ 54┊    id: 1,
+┊   ┊ 55┊    username: 'ethan',
+┊   ┊ 56┊    password: '$2a$08$NO9tkFLCoSqX1c5wk3s7z.JfxaVMKA.m7zUDdDwEquo4rvzimQeJm', // 111
+┊   ┊ 57┊    name: 'Ethan Gonzalez',
+┊   ┊ 58┊    picture: 'https://randomuser.me/api/portraits/thumb/men/1.jpg',
+┊   ┊ 59┊    phone: '+391234567890',
+┊   ┊ 60┊  },
+┊   ┊ 61┊  {
+┊   ┊ 62┊    id: 2,
+┊   ┊ 63┊    username: 'bryan',
+┊   ┊ 64┊    password: '$2a$08$xE4FuCi/ifxjL2S8CzKAmuKLwv18ktksSN.F3XYEnpmcKtpbpeZgO', // 222
+┊   ┊ 65┊    name: 'Bryan Wallace',
+┊   ┊ 66┊    picture: 'https://randomuser.me/api/portraits/thumb/men/2.jpg',
+┊   ┊ 67┊    phone: '+391234567891',
+┊   ┊ 68┊  },
+┊   ┊ 69┊  {
+┊   ┊ 70┊    id: 3,
+┊   ┊ 71┊    username: 'avery',
+┊   ┊ 72┊    password: '$2a$08$UHgH7J8G6z1mGQn2qx2kdeWv0jvgHItyAsL9hpEUI3KJmhVW5Q1d.', // 333
+┊   ┊ 73┊    name: 'Avery Stewart',
+┊   ┊ 74┊    picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 75┊    phone: '+391234567892',
+┊   ┊ 76┊  },
+┊   ┊ 77┊  {
+┊   ┊ 78┊    id: 4,
+┊   ┊ 79┊    username: 'katie',
+┊   ┊ 80┊    password: '$2a$08$wR1k5Q3T9FC7fUgB7Gdb9Os/GV7dGBBf4PLlWT7HERMFhmFDt47xi', // 444
+┊   ┊ 81┊    name: 'Katie Peterson',
+┊   ┊ 82┊    picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
+┊   ┊ 83┊    phone: '+391234567893',
+┊   ┊ 84┊  },
+┊   ┊ 85┊  {
+┊   ┊ 86┊    id: 5,
+┊   ┊ 87┊    username: 'ray',
+┊   ┊ 88┊    password: '$2a$08$6.mbXqsDX82ZZ7q5d8Osb..JrGSsNp4R3IKj7mxgF6YGT0OmMw242', // 555
+┊   ┊ 89┊    name: 'Ray Edwards',
+┊   ┊ 90┊    picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
+┊   ┊ 91┊    phone: '+391234567894',
+┊   ┊ 92┊  },
+┊   ┊ 93┊  {
+┊   ┊ 94┊    id: 6,
+┊   ┊ 95┊    username: 'niko',
+┊   ┊ 96┊    password: '$2a$08$fL5lZR.Rwf9FWWe8XwwlceiPBBim8n9aFtaem.INQhiKT4.Ux3Uq.', // 666
+┊   ┊ 97┊    name: 'Niccolò Belli',
+┊   ┊ 98┊    picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
+┊   ┊ 99┊    phone: '+391234567895',
+┊   ┊100┊  },
+┊   ┊101┊  {
+┊   ┊102┊    id: 7,
+┊   ┊103┊    username: 'mario',
+┊   ┊104┊    password: '$2a$08$nDHDmWcVxDnH5DDT3HMMC.psqcnu6wBiOgkmJUy9IH..qxa3R6YrO', // 777
+┊   ┊105┊    name: 'Mario Rossi',
+┊   ┊106┊    picture: 'https://randomuser.me/api/portraits/thumb/men/5.jpg',
+┊   ┊107┊    phone: '+391234567896',
+┊   ┊108┊  },
+┊   ┊109┊];
+┊   ┊110┊
+┊   ┊111┊const chats: Chat[] = [
+┊   ┊112┊  {
+┊   ┊113┊    id: 1,
+┊   ┊114┊    name: null,
+┊   ┊115┊    picture: null,
+┊   ┊116┊    allTimeMemberIds: [1, 3],
+┊   ┊117┊    listingMemberIds: [1, 3],
+┊   ┊118┊    adminIds: null,
+┊   ┊119┊    ownerId: null,
+┊   ┊120┊    messages: [
+┊   ┊121┊      {
+┊   ┊122┊        id: 1,
+┊   ┊123┊        chatId: 1,
+┊   ┊124┊        senderId: 1,
+┊   ┊125┊        content: 'You on your way?',
+┊   ┊126┊        createdAt: moment().subtract(1, 'hours').unix(),
+┊   ┊127┊        type: MessageType.TEXT,
+┊   ┊128┊        recipients: [
+┊   ┊129┊          {
+┊   ┊130┊            userId: 3,
+┊   ┊131┊            messageId: 1,
+┊   ┊132┊            chatId: 1,
+┊   ┊133┊            receivedAt: null,
+┊   ┊134┊            readAt: null,
+┊   ┊135┊          },
+┊   ┊136┊        ],
+┊   ┊137┊        holderIds: [1, 3],
+┊   ┊138┊      },
+┊   ┊139┊      {
+┊   ┊140┊        id: 2,
+┊   ┊141┊        chatId: 1,
+┊   ┊142┊        senderId: 3,
+┊   ┊143┊        content: 'Yep!',
+┊   ┊144┊        createdAt: moment().subtract(1, 'hours').add(5, 'minutes').unix(),
+┊   ┊145┊        type: MessageType.TEXT,
+┊   ┊146┊        recipients: [
+┊   ┊147┊          {
+┊   ┊148┊            userId: 1,
+┊   ┊149┊            messageId: 2,
+┊   ┊150┊            chatId: 1,
+┊   ┊151┊            receivedAt: null,
+┊   ┊152┊            readAt: null,
+┊   ┊153┊          },
+┊   ┊154┊        ],
+┊   ┊155┊        holderIds: [3, 1],
+┊   ┊156┊      },
+┊   ┊157┊    ],
+┊   ┊158┊  },
+┊   ┊159┊  {
+┊   ┊160┊    id: 2,
+┊   ┊161┊    name: null,
+┊   ┊162┊    picture: null,
+┊   ┊163┊    allTimeMemberIds: [1, 4],
+┊   ┊164┊    listingMemberIds: [1, 4],
+┊   ┊165┊    adminIds: null,
+┊   ┊166┊    ownerId: null,
+┊   ┊167┊    messages: [
+┊   ┊168┊      {
+┊   ┊169┊        id: 1,
+┊   ┊170┊        chatId: 2,
+┊   ┊171┊        senderId: 1,
+┊   ┊172┊        content: 'Hey, it\'s me',
+┊   ┊173┊        createdAt: moment().subtract(2, 'hours').unix(),
+┊   ┊174┊        type: MessageType.TEXT,
+┊   ┊175┊        recipients: [
+┊   ┊176┊          {
+┊   ┊177┊            userId: 4,
+┊   ┊178┊            messageId: 1,
+┊   ┊179┊            chatId: 2,
+┊   ┊180┊            receivedAt: null,
+┊   ┊181┊            readAt: null,
+┊   ┊182┊          },
+┊   ┊183┊        ],
+┊   ┊184┊        holderIds: [1, 4],
+┊   ┊185┊      },
+┊   ┊186┊    ],
+┊   ┊187┊  },
+┊   ┊188┊  {
+┊   ┊189┊    id: 3,
+┊   ┊190┊    name: null,
+┊   ┊191┊    picture: null,
+┊   ┊192┊    allTimeMemberIds: [1, 5],
+┊   ┊193┊    listingMemberIds: [1, 5],
+┊   ┊194┊    adminIds: null,
+┊   ┊195┊    ownerId: null,
+┊   ┊196┊    messages: [
+┊   ┊197┊      {
+┊   ┊198┊        id: 1,
+┊   ┊199┊        chatId: 3,
+┊   ┊200┊        senderId: 1,
+┊   ┊201┊        content: 'I should buy a boat',
+┊   ┊202┊        createdAt: moment().subtract(1, 'days').unix(),
+┊   ┊203┊        type: MessageType.TEXT,
+┊   ┊204┊        recipients: [
+┊   ┊205┊          {
+┊   ┊206┊            userId: 5,
+┊   ┊207┊            messageId: 1,
+┊   ┊208┊            chatId: 3,
+┊   ┊209┊            receivedAt: null,
+┊   ┊210┊            readAt: null,
+┊   ┊211┊          },
+┊   ┊212┊        ],
+┊   ┊213┊        holderIds: [1, 5],
+┊   ┊214┊      },
+┊   ┊215┊      {
+┊   ┊216┊        id: 2,
+┊   ┊217┊        chatId: 3,
+┊   ┊218┊        senderId: 1,
+┊   ┊219┊        content: 'You still there?',
+┊   ┊220┊        createdAt: moment().subtract(1, 'days').add(16, 'hours').unix(),
+┊   ┊221┊        type: MessageType.TEXT,
+┊   ┊222┊        recipients: [
+┊   ┊223┊          {
+┊   ┊224┊            userId: 5,
+┊   ┊225┊            messageId: 2,
+┊   ┊226┊            chatId: 3,
+┊   ┊227┊            receivedAt: null,
+┊   ┊228┊            readAt: null,
+┊   ┊229┊          },
+┊   ┊230┊        ],
+┊   ┊231┊        holderIds: [1, 5],
+┊   ┊232┊      },
+┊   ┊233┊    ],
+┊   ┊234┊  },
+┊   ┊235┊  {
+┊   ┊236┊    id: 4,
+┊   ┊237┊    name: null,
+┊   ┊238┊    picture: null,
+┊   ┊239┊    allTimeMemberIds: [3, 4],
+┊   ┊240┊    listingMemberIds: [3, 4],
+┊   ┊241┊    adminIds: null,
+┊   ┊242┊    ownerId: null,
+┊   ┊243┊    messages: [
+┊   ┊244┊      {
+┊   ┊245┊        id: 1,
+┊   ┊246┊        chatId: 4,
+┊   ┊247┊        senderId: 3,
+┊   ┊248┊        content: 'Look at my mukluks!',
+┊   ┊249┊        createdAt: moment().subtract(4, 'days').unix(),
+┊   ┊250┊        type: MessageType.TEXT,
+┊   ┊251┊        recipients: [
+┊   ┊252┊          {
+┊   ┊253┊            userId: 4,
+┊   ┊254┊            messageId: 1,
+┊   ┊255┊            chatId: 4,
+┊   ┊256┊            receivedAt: null,
+┊   ┊257┊            readAt: null,
+┊   ┊258┊          },
+┊   ┊259┊        ],
+┊   ┊260┊        holderIds: [3, 4],
+┊   ┊261┊      },
+┊   ┊262┊    ],
+┊   ┊263┊  },
+┊   ┊264┊  {
+┊   ┊265┊    id: 5,
+┊   ┊266┊    name: null,
+┊   ┊267┊    picture: null,
+┊   ┊268┊    allTimeMemberIds: [2, 5],
+┊   ┊269┊    listingMemberIds: [2, 5],
+┊   ┊270┊    adminIds: null,
+┊   ┊271┊    ownerId: null,
+┊   ┊272┊    messages: [
+┊   ┊273┊      {
+┊   ┊274┊        id: 1,
+┊   ┊275┊        chatId: 5,
+┊   ┊276┊        senderId: 2,
+┊   ┊277┊        content: 'This is wicked good ice cream.',
+┊   ┊278┊        createdAt: moment().subtract(2, 'weeks').unix(),
+┊   ┊279┊        type: MessageType.TEXT,
+┊   ┊280┊        recipients: [
+┊   ┊281┊          {
+┊   ┊282┊            userId: 5,
+┊   ┊283┊            messageId: 1,
+┊   ┊284┊            chatId: 5,
+┊   ┊285┊            receivedAt: null,
+┊   ┊286┊            readAt: null,
+┊   ┊287┊          },
+┊   ┊288┊        ],
+┊   ┊289┊        holderIds: [2, 5],
+┊   ┊290┊      },
+┊   ┊291┊      {
+┊   ┊292┊        id: 2,
+┊   ┊293┊        chatId: 6,
+┊   ┊294┊        senderId: 5,
+┊   ┊295┊        content: 'Love it!',
+┊   ┊296┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').unix(),
+┊   ┊297┊        type: MessageType.TEXT,
+┊   ┊298┊        recipients: [
+┊   ┊299┊          {
+┊   ┊300┊            userId: 2,
+┊   ┊301┊            messageId: 2,
+┊   ┊302┊            chatId: 5,
+┊   ┊303┊            receivedAt: null,
+┊   ┊304┊            readAt: null,
+┊   ┊305┊          },
+┊   ┊306┊        ],
+┊   ┊307┊        holderIds: [5, 2],
+┊   ┊308┊      },
+┊   ┊309┊    ],
+┊   ┊310┊  },
+┊   ┊311┊  {
+┊   ┊312┊    id: 6,
+┊   ┊313┊    name: null,
+┊   ┊314┊    picture: null,
+┊   ┊315┊    allTimeMemberIds: [1, 6],
+┊   ┊316┊    listingMemberIds: [1],
+┊   ┊317┊    adminIds: null,
+┊   ┊318┊    ownerId: null,
+┊   ┊319┊    messages: [],
+┊   ┊320┊  },
+┊   ┊321┊  {
+┊   ┊322┊    id: 7,
+┊   ┊323┊    name: null,
+┊   ┊324┊    picture: null,
+┊   ┊325┊    allTimeMemberIds: [2, 1],
+┊   ┊326┊    listingMemberIds: [2],
+┊   ┊327┊    adminIds: null,
+┊   ┊328┊    ownerId: null,
+┊   ┊329┊    messages: [],
+┊   ┊330┊  },
+┊   ┊331┊  {
+┊   ┊332┊    id: 8,
+┊   ┊333┊    name: 'A user 0 group',
+┊   ┊334┊    picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊335┊    allTimeMemberIds: [1, 3, 4, 6],
+┊   ┊336┊    listingMemberIds: [1, 3, 4, 6],
+┊   ┊337┊    actualGroupMemberIds: [1, 4, 6],
+┊   ┊338┊    adminIds: [1, 6],
+┊   ┊339┊    ownerId: 1,
+┊   ┊340┊    messages: [
+┊   ┊341┊      {
+┊   ┊342┊        id: 1,
+┊   ┊343┊        chatId: 8,
+┊   ┊344┊        senderId: 1,
+┊   ┊345┊        content: 'I made a group',
+┊   ┊346┊        createdAt: moment().subtract(2, 'weeks').unix(),
+┊   ┊347┊        type: MessageType.TEXT,
+┊   ┊348┊        recipients: [
+┊   ┊349┊          {
+┊   ┊350┊            userId: 3,
+┊   ┊351┊            messageId: 1,
+┊   ┊352┊            chatId: 8,
+┊   ┊353┊            receivedAt: null,
+┊   ┊354┊            readAt: null,
+┊   ┊355┊          },
+┊   ┊356┊          {
+┊   ┊357┊            userId: 4,
+┊   ┊358┊            messageId: 1,
+┊   ┊359┊            chatId: 8,
+┊   ┊360┊            receivedAt: moment().subtract(2, 'weeks').add(1, 'minutes').unix(),
+┊   ┊361┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').unix(),
+┊   ┊362┊          },
+┊   ┊363┊          {
+┊   ┊364┊            userId: 6,
+┊   ┊365┊            messageId: 1,
+┊   ┊366┊            chatId: 8,
+┊   ┊367┊            receivedAt: null,
+┊   ┊368┊            readAt: null,
+┊   ┊369┊          },
+┊   ┊370┊        ],
+┊   ┊371┊        holderIds: [1, 3, 4, 6],
+┊   ┊372┊      },
+┊   ┊373┊      {
+┊   ┊374┊        id: 2,
+┊   ┊375┊        chatId: 8,
+┊   ┊376┊        senderId: 1,
+┊   ┊377┊        content: 'Ops, user 3 was not supposed to be here',
+┊   ┊378┊        createdAt: moment().subtract(2, 'weeks').add(2, 'minutes').unix(),
+┊   ┊379┊        type: MessageType.TEXT,
+┊   ┊380┊        recipients: [
+┊   ┊381┊          {
+┊   ┊382┊            userId: 4,
+┊   ┊383┊            messageId: 2,
+┊   ┊384┊            chatId: 8,
+┊   ┊385┊            receivedAt: moment().subtract(2, 'weeks').add(3, 'minutes').unix(),
+┊   ┊386┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').unix(),
+┊   ┊387┊          },
+┊   ┊388┊          {
+┊   ┊389┊            userId: 6,
+┊   ┊390┊            messageId: 2,
+┊   ┊391┊            chatId: 8,
+┊   ┊392┊            receivedAt: null,
+┊   ┊393┊            readAt: null,
+┊   ┊394┊          },
+┊   ┊395┊        ],
+┊   ┊396┊        holderIds: [1, 4, 6],
+┊   ┊397┊      },
+┊   ┊398┊      {
+┊   ┊399┊        id: 3,
+┊   ┊400┊        chatId: 8,
+┊   ┊401┊        senderId: 4,
+┊   ┊402┊        content: 'Awesome!',
+┊   ┊403┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').unix(),
+┊   ┊404┊        type: MessageType.TEXT,
+┊   ┊405┊        recipients: [
+┊   ┊406┊          {
+┊   ┊407┊            userId: 1,
+┊   ┊408┊            messageId: 3,
+┊   ┊409┊            chatId: 8,
+┊   ┊410┊            receivedAt: null,
+┊   ┊411┊            readAt: null,
+┊   ┊412┊          },
+┊   ┊413┊          {
+┊   ┊414┊            userId: 6,
+┊   ┊415┊            messageId: 3,
+┊   ┊416┊            chatId: 8,
+┊   ┊417┊            receivedAt: null,
+┊   ┊418┊            readAt: null,
+┊   ┊419┊          },
+┊   ┊420┊        ],
+┊   ┊421┊        holderIds: [1, 4, 6],
+┊   ┊422┊      },
+┊   ┊423┊    ],
+┊   ┊424┊  },
+┊   ┊425┊  {
+┊   ┊426┊    id: 9,
+┊   ┊427┊    name: 'A user 5 group',
+┊   ┊428┊    picture: null,
+┊   ┊429┊    allTimeMemberIds: [6, 3],
+┊   ┊430┊    listingMemberIds: [6, 3],
+┊   ┊431┊    actualGroupMemberIds: [6, 3],
+┊   ┊432┊    adminIds: [6],
+┊   ┊433┊    ownerId: 6,
+┊   ┊434┊    messages: [],
+┊   ┊435┊  },
+┊   ┊436┊];
+┊   ┊437┊
+┊   ┊438┊export const db = {users, chats};
```

[}]: #

Its' finally time to create our schema and our resolvers:

[{]: <helper> (diffStep "1.3" module="server")

#### Step 1.3: Add resolvers and schema

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,6 +1,51 @@
+┊  ┊ 1┊import { Chat, db, Message, Recipient, User } from "../db";
 ┊ 1┊ 2┊import { IResolvers } from "graphql-tools/dist/Interfaces";
 ┊ 2┊ 3┊
+┊  ┊ 4┊let users = db.users;
+┊  ┊ 5┊let chats = db.chats;
+┊  ┊ 6┊const currentUser = 1;
+┊  ┊ 7┊
 ┊ 3┊ 8┊export const resolvers: IResolvers = {
 ┊ 4┊ 9┊  Query: {
+┊  ┊10┊    // Show all users for the moment.
+┊  ┊11┊    users: (): User[] => users.filter(user => user.id !== currentUser),
+┊  ┊12┊    chats: (): Chat[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser)),
+┊  ┊13┊    chat: (obj: any, {chatId}): Chat | null => chats.find(chat => chat.id === chatId) || null,
+┊  ┊14┊  },
+┊  ┊15┊  Chat: {
+┊  ┊16┊    name: (chat: Chat): string => chat.name ? chat.name : users
+┊  ┊17┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser))!.name,
+┊  ┊18┊    picture: (chat: Chat) => chat.name ? chat.picture : users
+┊  ┊19┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser))!.picture,
+┊  ┊20┊    allTimeMembers: (chat: Chat): User[] => users.filter(user => chat.allTimeMemberIds.includes(user.id)),
+┊  ┊21┊    listingMembers: (chat: Chat): User[] => users.filter(user => chat.listingMemberIds.includes(user.id)),
+┊  ┊22┊    actualGroupMembers: (chat: Chat): User[] => users.filter(user => chat.actualGroupMemberIds && chat.actualGroupMemberIds.includes(user.id)),
+┊  ┊23┊    admins: (chat: Chat): User[] => users.filter(user => chat.adminIds && chat.adminIds.includes(user.id)),
+┊  ┊24┊    owner: (chat: Chat): User | null => users.find(user => chat.ownerId === user.id) || null,
+┊  ┊25┊    messages: (chat: Chat, {amount = null}: {amount: number}): Message[] => {
+┊  ┊26┊      const messages = chat.messages
+┊  ┊27┊      .filter(message => message.holderIds.includes(currentUser))
+┊  ┊28┊      .sort((a, b) => b.createdAt - a.createdAt) || <Message[]>[];
+┊  ┊29┊      return (amount ? messages.slice(0, amount) : messages).reverse();
+┊  ┊30┊    },
+┊  ┊31┊    unreadMessages: (chat: Chat): number => chat.messages
+┊  ┊32┊      .filter(message => message.holderIds.includes(currentUser) &&
+┊  ┊33┊        message.recipients.find(recipient => recipient.userId === currentUser && !recipient.readAt))
+┊  ┊34┊      .length,
+┊  ┊35┊    isGroup: (chat: Chat): boolean => !!chat.name,
+┊  ┊36┊  },
+┊  ┊37┊  Message: {
+┊  ┊38┊    chat: (message: Message): Chat | null => chats.find(chat => message.chatId === chat.id) || null,
+┊  ┊39┊    sender: (message: Message): User | null => users.find(user => user.id === message.senderId) || null,
+┊  ┊40┊    holders: (message: Message): User[] => users.filter(user => message.holderIds.includes(user.id)),
+┊  ┊41┊    ownership: (message: Message): boolean => message.senderId === currentUser,
+┊  ┊42┊  },
+┊  ┊43┊  Recipient: {
+┊  ┊44┊    user: (recipient: Recipient): User | null => users.find(user => recipient.userId === user.id) || null,
+┊  ┊45┊    message: (recipient: Recipient): Message | null => {
+┊  ┊46┊      const chat = chats.find(chat => recipient.chatId === chat.id);
+┊  ┊47┊      return chat ? chat.messages.find(message => recipient.messageId === message.id) || null : null;
+┊  ┊48┊    },
+┊  ┊49┊    chat: (recipient: Recipient): Chat | null => chats.find(chat => recipient.chatId === chat.id) || null,
 ┊ 5┊50┊  },
 ┊ 6┊51┊};
```

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -1,4 +1,70 @@
 ┊ 1┊ 1┊import { ITypeDefinitions } from "graphql-tools/dist/Interfaces";
 ┊ 2┊ 2┊
 ┊ 3┊ 3┊export const typeDefs: ITypeDefinitions = `
+┊  ┊ 4┊  type Query {
+┊  ┊ 5┊    users: [User!]
+┊  ┊ 6┊    chats: [Chat!]
+┊  ┊ 7┊    chat(chatId: ID!): Chat
+┊  ┊ 8┊  }
+┊  ┊ 9┊
+┊  ┊10┊  enum MessageType {
+┊  ┊11┊    LOCATION
+┊  ┊12┊    TEXT
+┊  ┊13┊    PICTURE
+┊  ┊14┊  }
+┊  ┊15┊  
+┊  ┊16┊  type Chat {
+┊  ┊17┊    #May be a chat or a group
+┊  ┊18┊    id: ID!
+┊  ┊19┊    #Computed for chats
+┊  ┊20┊    name: String
+┊  ┊21┊    #Computed for chats
+┊  ┊22┊    picture: String
+┊  ┊23┊    #All members, current and past ones.
+┊  ┊24┊    allTimeMembers: [User!]!
+┊  ┊25┊    #Whoever gets the chat listed. For groups includes past members who still didn't delete the group.
+┊  ┊26┊    listingMembers: [User!]!
+┊  ┊27┊    #Actual members of the group (they are not the only ones who get the group listed). Null for chats.
+┊  ┊28┊    actualGroupMembers: [User!]!
+┊  ┊29┊    #Null for chats
+┊  ┊30┊    admins: [User!]
+┊  ┊31┊    #If null the group is read-only. Null for chats.
+┊  ┊32┊    owner: User
+┊  ┊33┊    messages(amount: Int): [Message]!
+┊  ┊34┊    #Computed property
+┊  ┊35┊    unreadMessages: Int!
+┊  ┊36┊    #Computed property
+┊  ┊37┊    isGroup: Boolean!
+┊  ┊38┊  }
+┊  ┊39┊
+┊  ┊40┊  type Message {
+┊  ┊41┊    id: ID!
+┊  ┊42┊    sender: User!
+┊  ┊43┊    chat: Chat!
+┊  ┊44┊    content: String!
+┊  ┊45┊    createdAt: String!
+┊  ┊46┊    #FIXME: should return MessageType
+┊  ┊47┊    type: Int!
+┊  ┊48┊    #Whoever received the message
+┊  ┊49┊    recipients: [Recipient!]!
+┊  ┊50┊    #Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise
+┊  ┊51┊    holders: [User!]!
+┊  ┊52┊    #Computed property
+┊  ┊53┊    ownership: Boolean!
+┊  ┊54┊  }
+┊  ┊55┊  
+┊  ┊56┊  type Recipient {
+┊  ┊57┊    user: User!
+┊  ┊58┊    message: Message!
+┊  ┊59┊    chat: Chat!
+┊  ┊60┊    receivedAt: String
+┊  ┊61┊    readAt: String
+┊  ┊62┊  }
+┊  ┊63┊
+┊  ┊64┊  type User {
+┊  ┊65┊    id: ID!
+┊  ┊66┊    name: String
+┊  ┊67┊    picture: String
+┊  ┊68┊    phone: String
+┊  ┊69┊  }
 ┊ 4┊70┊`;
```

[}]: #

Out basic server is already done and working. We still have no way to do any kind of mutation, but we already set up several queries to return a list of users or chats.
In particular we can choose if we want to return all the chats (and how many messages we want to return for each chat) or if we want to return a single chat. We can also choose which and how many properties we want to return for each query.
We can start the server by simply running:

    $ npm start

## Client

Now we can concentrate on the client and bootstrap it using Angular-CLI.
First you will need to install it globally with:

    $ npm install -g @angular/cli

Then we can create a new project from scratch:

    $ ng new client --style scss

Time to install a couple of packages:

    $ ng add apollo-angular
    $ npm install --save-dev @types/graphql

Apollo's Schematics sets everything for us. The only thing missing is to fill the `url`.

[{]: <helper> (diffStep "1.1" files="src/app/graphql.module.ts" module="client")

#### Step 1.1: Add angular-apollo to app module



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
@@ -0,0 +1,26 @@
+┊  ┊ 1┊import {ApolloQueryResult, WatchQueryOptions} from 'apollo-client';
+┊  ┊ 2┊import {map} from 'rxjs/operators';
+┊  ┊ 3┊import {Apollo} from 'apollo-angular';
+┊  ┊ 4┊import {Injectable} from '@angular/core';
+┊  ┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
+┊  ┊ 6┊
+┊  ┊ 7┊@Injectable()
+┊  ┊ 8┊export class ChatsService {
+┊  ┊ 9┊  messagesAmount = 3;
+┊  ┊10┊
+┊  ┊11┊  constructor(private apollo: Apollo) {}
+┊  ┊12┊
+┊  ┊13┊  getChats() {
+┊  ┊14┊    const query = this.apollo.watchQuery<any>(<WatchQueryOptions>{
+┊  ┊15┊      query: getChatsQuery,
+┊  ┊16┊      variables: {
+┊  ┊17┊        amount: this.messagesAmount,
+┊  ┊18┊      },
+┊  ┊19┊    });
+┊  ┊20┊    const chats$ = query.valueChanges.pipe(
+┊  ┊21┊      map((result: ApolloQueryResult<any>) => result.data.chats)
+┊  ┊22┊    );
+┊  ┊23┊
+┊  ┊24┊    return {query, chats$};
+┊  ┊25┊  }
+┊  ┊26┊}
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
@@ -0,0 +1,26 @@
+┊  ┊ 1┊import {ApolloQueryResult, WatchQueryOptions} from 'apollo-client';
+┊  ┊ 2┊import {map} from 'rxjs/operators';
+┊  ┊ 3┊import {Apollo} from 'apollo-angular';
+┊  ┊ 4┊import {Injectable} from '@angular/core';
+┊  ┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
+┊  ┊ 6┊
+┊  ┊ 7┊@Injectable()
+┊  ┊ 8┊export class ChatsService {
+┊  ┊ 9┊  messagesAmount = 3;
+┊  ┊10┊
+┊  ┊11┊  constructor(private apollo: Apollo) {}
+┊  ┊12┊
+┊  ┊13┊  getChats() {
+┊  ┊14┊    const query = this.apollo.watchQuery<any>(<WatchQueryOptions>{
+┊  ┊15┊      query: getChatsQuery,
+┊  ┊16┊      variables: {
+┊  ┊17┊        amount: this.messagesAmount,
+┊  ┊18┊      },
+┊  ┊19┊    });
+┊  ┊20┊    const chats$ = query.valueChanges.pipe(
+┊  ┊21┊      map((result: ApolloQueryResult<any>) => result.data.chats)
+┊  ┊22┊    );
+┊  ┊23┊
+┊  ┊24┊    return {query, chats$};
+┊  ┊25┊  }
+┊  ┊26┊}
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

    $ npm install @angular/cdk @angular/material hammerjs ng2-truncate

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
@@ -1 +1,8 @@
 ┊1┊1┊/* You can add global styles to this file, and also import other style files */
+┊ ┊2┊
+┊ ┊3┊/* Meterial theme */
+┊ ┊4┊@import "~@angular/material/prebuilt-themes/indigo-pink.css";
+┊ ┊5┊
+┊ ┊6┊body {
+┊ ┊7┊  margin: 0;
+┊ ┊8┊}
```

[}]: #

We're now creating a `shared` module where we will define our header component where we're going to project a different content from each component:

[{]: <helper> (diffStep "1.3" files="src/app/shared/*" module="client")

#### Step 1.3: List the chats

##### Added src&#x2F;app&#x2F;shared&#x2F;components&#x2F;toolbar&#x2F;toolbar.component.scss
```diff
@@ -0,0 +1,13 @@
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
+┊  ┊12┊  }
+┊  ┊13┊}
```

##### Added src&#x2F;app&#x2F;shared&#x2F;components&#x2F;toolbar&#x2F;toolbar.component.ts
```diff
@@ -0,0 +1,18 @@
+┊  ┊ 1┊import {Component} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-toolbar',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <mat-toolbar>
+┊  ┊ 7┊      <div class="left-block">
+┊  ┊ 8┊        <ng-content select=".navigation"></ng-content>
+┊  ┊ 9┊        <ng-content select=".title"></ng-content>
+┊  ┊10┊      </div>
+┊  ┊11┊      <ng-content select=".menu"></ng-content>
+┊  ┊12┊    </mat-toolbar>
+┊  ┊13┊  `,
+┊  ┊14┊  styleUrls: ['./toolbar.component.scss']
+┊  ┊15┊})
+┊  ┊16┊export class ToolbarComponent {
+┊  ┊17┊
+┊  ┊18┊}
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

Now we want to create the `chats-lister` module, with a container component called `ChatsComponent` and a couple of presentational components.

[{]: <helper> (diffStep "1.3" files="src/app/chats-lister/*" module="client")

#### Step 1.3: List the chats

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;chats-lister.module.ts
```diff
@@ -0,0 +1,49 @@
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
+┊  ┊12┊import {TruncateModule} from 'ng2-truncate';
+┊  ┊13┊import {SharedModule} from '../shared/shared.module';
+┊  ┊14┊
+┊  ┊15┊const routes: Routes = [
+┊  ┊16┊  {path: '', redirectTo: 'chats', pathMatch: 'full'},
+┊  ┊17┊  {path: 'chats', component: ChatsComponent},
+┊  ┊18┊];
+┊  ┊19┊
+┊  ┊20┊@NgModule({
+┊  ┊21┊  declarations: [
+┊  ┊22┊    ChatsComponent,
+┊  ┊23┊    ChatsListComponent,
+┊  ┊24┊    ChatItemComponent,
+┊  ┊25┊  ],
+┊  ┊26┊  imports: [
+┊  ┊27┊    BrowserModule,
+┊  ┊28┊    // Material
+┊  ┊29┊    MatMenuModule,
+┊  ┊30┊    MatIconModule,
+┊  ┊31┊    MatButtonModule,
+┊  ┊32┊    MatListModule,
+┊  ┊33┊    // Animations
+┊  ┊34┊    BrowserAnimationsModule,
+┊  ┊35┊    // Routing
+┊  ┊36┊    RouterModule.forChild(routes),
+┊  ┊37┊    // Forms
+┊  ┊38┊    FormsModule,
+┊  ┊39┊    // Truncate Pipe
+┊  ┊40┊    TruncateModule,
+┊  ┊41┊    // Feature modules
+┊  ┊42┊    SharedModule,
+┊  ┊43┊  ],
+┊  ┊44┊  providers: [
+┊  ┊45┊    ChatsService,
+┊  ┊46┊  ],
+┊  ┊47┊})
+┊  ┊48┊export class ChatsListerModule {
+┊  ┊49┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.scss
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  width: 100%;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊.chat-row {
+┊  ┊ 7┊  padding: 0;
+┊  ┊ 8┊  display: flex;
+┊  ┊ 9┊  width: 100%;
+┊  ┊10┊  justify-content: space-between;
+┊  ┊11┊  align-items: center;
+┊  ┊12┊
+┊  ┊13┊  .chat-recipient {
+┊  ┊14┊    display: flex;
+┊  ┊15┊    width: 60%;
+┊  ┊16┊  }
+┊  ┊17┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-chat-item',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <div class="chat-row">
+┊  ┊ 7┊        <div class="chat-recipient">
+┊  ┊ 8┊          <img *ngIf="chat.picture" [src]="chat.picture" width="48" height="48">
+┊  ┊ 9┊          <div>{{ chat.name }} [id: {{ chat.id }}]</div>
+┊  ┊10┊        </div>
+┊  ┊11┊        <div class="chat-content">{{ chat.messages[chat.messages.length - 1]?.content | truncate : 20 : '...' }}</div>
+┊  ┊12┊    </div>
+┊  ┊13┊  `,
+┊  ┊14┊  styleUrls: ['chat-item.component.scss'],
+┊  ┊15┊})
+┊  ┊16┊export class ChatItemComponent {
+┊  ┊17┊  // tslint:disable-next-line:no-input-rename
+┊  ┊18┊  @Input('item')
+┊  ┊19┊  chat: any;
+┊  ┊20┊}
```

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chats-list&#x2F;chats-list.component.scss
```diff
@@ -0,0 +1,3 @@
+┊ ┊1┊:host {
+┊ ┊2┊  display: block;
+┊ ┊3┊}
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
+┊  ┊31┊    <button class="chat-button" mat-fab color="primary">
+┊  ┊32┊      <mat-icon aria-label="Icon-button with a + icon">add</mat-icon>
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
@@ -5,7 +5,10 @@
 ┊ 5┊ 5┊import {HttpClientModule} from '@angular/common/http';
 ┊ 6┊ 6┊import {HttpLink, HttpLinkModule, Options} from 'apollo-angular-link-http';
 ┊ 7┊ 7┊import {Apollo, ApolloModule} from 'apollo-angular';
-┊ 8┊  ┊import {InMemoryCache} from 'apollo-cache-inmemory';
+┊  ┊ 8┊import {defaultDataIdFromObject, InMemoryCache} from 'apollo-cache-inmemory';
+┊  ┊ 9┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
+┊  ┊10┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊11┊const routes: Routes = [];
 ┊ 9┊12┊
 ┊10┊13┊@NgModule({
 ┊11┊14┊  declarations: [
```
```diff
@@ -17,6 +20,10 @@
 ┊17┊20┊    ApolloModule,
 ┊18┊21┊    HttpLinkModule,
 ┊19┊22┊    HttpClientModule,
+┊  ┊23┊    // Routing
+┊  ┊24┊    RouterModule.forRoot(routes),
+┊  ┊25┊    // Feature modules
+┊  ┊26┊    ChatsListerModule,
 ┊20┊27┊  ],
 ┊21┊28┊  providers: [],
 ┊22┊29┊  bootstrap: [AppComponent]
```

[}]: #

If you will try to run the frontend you will notice that several messages seems like duplicated, why does it happen?

Since we use NoSQL-like structure in our backend, messages are stored as an array inside each chat so their incremental identifiers are not unique across different chats. We need to normalize them in a way that takes into account both the message id and the chat id:

[{]: <helper> (diffStep "1.4" module="client")

#### Step 1.4: Better normalize messages

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -35,7 +35,14 @@
 ┊35┊35┊  ) {
 ┊36┊36┊    apollo.create({
 ┊37┊37┊      link: httpLink.create(<Options>{uri: 'http://localhost:3000/graphql'}),
-┊38┊  ┊      cache: new InMemoryCache(),
+┊  ┊38┊      cache: new InMemoryCache({
+┊  ┊39┊        dataIdFromObject: (object: any) => {
+┊  ┊40┊          switch (object.__typename) {
+┊  ┊41┊            case 'Message': return `${object.chat.id}:${object.id}`; // use `chatId` prefix and `messageId` as the primary key
+┊  ┊42┊            default: return defaultDataIdFromObject(object); // fall back to default handling
+┊  ┊43┊          }
+┊  ┊44┊        }
+┊  ┊45┊      }),
 ┊39┊46┊    });
 ┊40┊47┊  }
 ┊41┊48┊}
```

[}]: #

That way our application will work even if the backend is a NoSQL. What's even more interesting is that our application will keep working as well even when we will switch our backend to PostgreSQL.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step4.md) | [Next Step >](step6.md) |
|:--------------------------------|--------------------------------:|

[}]: #
