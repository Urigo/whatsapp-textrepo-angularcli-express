# Step 13: GraphQL Modules

[//]: # (head-end)


So far we've built a fully functional application, with a database and lots of cool features.
But it we look at our codebase we feel that we could separate the GraphQL server into smaller, reusable, feature based parts.
As our application grew, its code and schematic relationship became bigger and more complex, making the schema maintenance harder and harder to work with.
Some old school MVC frameworks solve this issue adding layers after layer, but most of them just implement separation based technical layers: controllers, models, etc.
There is a better approach: you should separate your GraphQL schema by modules, or features, and include everything related to a specific part of the app under a module.
This is where `GraphQL Modules` comes into our help, providing:
- Reusable modules: Modules are defined by their GraphQL schema (Schema first design). They're completely independent and can be shared between apps.
- Scalable structure: Allows to manage multiple teams and features, multiple micro-services and servers.
- Gradual growth: A clear, gradual path from a very simple and fast, single-file modules, to scalable multi-file, multi-teams, multi-repo, multi-server modules.
- Testable architecture: A rich toolset around testing, mocking and separation.

First things first, let's install it:

    yarn add @graphql-modules/core @graphql-modules/sonar

The first question we should ask ourselves is: how do we divide the app into modules?

First, let's tackle the most obvious ones: the `App Module` and the `Auth Module`.

## App Module and Auth Module

We obviously need authentication, so we need to create a module to handle it.
But we also need to create an "`App Module`", which allows us to import every other module. This is main module of our app.
First, let's delete the whole `/schema` directory, we won't need it anymore.
Instead, let's create a `/modules` directory and create `app.module.ts`:

[{]: <helper> (diffStep "8.2" files="modules/app.module.ts" module="server")

#### [Step 8.2: Auth Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/671a634)

##### Added modules&#x2F;app.module.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import { GraphQLModule } from '@graphql-modules/core';
+┊  ┊ 2┊import { Connection } from 'typeorm';
+┊  ┊ 3┊import { Express } from 'express';
+┊  ┊ 4┊import { AuthModule } from './auth';
+┊  ┊ 5┊
+┊  ┊ 6┊export interface IAppModuleConfig {
+┊  ┊ 7┊  connection: Connection,
+┊  ┊ 8┊  app: Express;
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊export const AppModule = new GraphQLModule<IAppModuleConfig>({
+┊  ┊12┊  name: 'App',
+┊  ┊13┊  imports: ({config: {connection, app}}) => [
+┊  ┊14┊    AuthModule.forRoot({
+┊  ┊15┊      connection,
+┊  ┊16┊      app,
+┊  ┊17┊    }),
+┊  ┊18┊  ],
+┊  ┊19┊  configRequired: true,
+┊  ┊20┊});
```

[}]: #

`imports` allows to import other modules (like the `Auth Module` which we are going to create soon), but it also allows us to access the `config` property which contains all the options we are going to pass to the `App Module` when we instantiate it.
We are going to pass those options to the child modules, as configuration parameters.

Since we are going to move all the middleware for authentication into the `Auth Module` itself, it's time to clean up our index file:

[{]: <helper> (diffStep "8.2" files="^index.ts" module="server")

#### [Step 8.2: Auth Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/671a634)

##### Changed index.ts
```diff
@@ -1,144 +1,29 @@
-┊  1┊   ┊/// <reference path="./cloudinary.d.ts" />
 ┊  2┊  1┊import "reflect-metadata"; // For TypeORM
 ┊  3┊  2┊require('dotenv').config();
-┊  4┊   ┊import { schema } from "./schema";
 ┊  5┊  3┊import bodyParser from "body-parser";
 ┊  6┊  4┊import cors from 'cors';
 ┊  7┊  5┊import express from 'express';
 ┊  8┊  6┊import { ApolloServer } from "apollo-server-express";
-┊  9┊   ┊import cloudinary from 'cloudinary';
-┊ 10┊   ┊import multer from 'multer';
-┊ 11┊   ┊import tmp from 'tmp';
-┊ 12┊   ┊import passport from "passport";
-┊ 13┊   ┊import basicStrategy from 'passport-http';
-┊ 14┊   ┊import bcrypt from 'bcrypt-nodejs';
 ┊ 15┊  7┊import { createServer } from "http";
-┊ 16┊   ┊import { pubsub } from "./schema/resolvers";
 ┊ 17┊  8┊import { createConnection } from "typeorm";
-┊ 18┊   ┊import { User } from "./entity/User";
 ┊ 19┊  9┊import { addSampleData } from "./db";
-┊ 20┊   ┊
-┊ 21┊   ┊function generateHash(password: string) {
-┊ 22┊   ┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
-┊ 23┊   ┊}
-┊ 24┊   ┊
-┊ 25┊   ┊function validPassword(password: string, localPassword: string) {
-┊ 26┊   ┊  return bcrypt.compareSync(password, localPassword);
-┊ 27┊   ┊}
+┊   ┊ 10┊import { AppModule } from "./modules/app.module";
 ┊ 28┊ 11┊
 ┊ 29┊ 12┊createConnection().then(async connection => {
 ┊ 30┊ 13┊  if (process.argv.includes('--add-sample-data')) {
 ┊ 31┊ 14┊    addSampleData(connection);
 ┊ 32┊ 15┊  }
 ┊ 33┊ 16┊
-┊ 34┊   ┊  passport.use('basic-signin', new basicStrategy.BasicStrategy(
-┊ 35┊   ┊    async function (username: string, password: string, done: any) {
-┊ 36┊   ┊      const user = await connection.getRepository(User).findOne({where: { username }});
-┊ 37┊   ┊      if (user && validPassword(password, user.password)) {
-┊ 38┊   ┊        return done(null, user);
-┊ 39┊   ┊      }
-┊ 40┊   ┊      return done(null, false);
-┊ 41┊   ┊    }
-┊ 42┊   ┊  ));
-┊ 43┊   ┊
-┊ 44┊   ┊  passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
-┊ 45┊   ┊    async function (req: any, username: string, password: string, done: any) {
-┊ 46┊   ┊      const userExists = !!(await connection.getRepository(User).findOne({where: { username }}));
-┊ 47┊   ┊      if (!userExists && password && req.body.name) {
-┊ 48┊   ┊        const user = await connection.manager.save(new User({
-┊ 49┊   ┊          username,
-┊ 50┊   ┊          password: generateHash(password),
-┊ 51┊   ┊          name: req.body.name,
-┊ 52┊   ┊        }));
-┊ 53┊   ┊
-┊ 54┊   ┊        pubsub.publish('userAdded', {
-┊ 55┊   ┊          userAdded: user,
-┊ 56┊   ┊        });
-┊ 57┊   ┊
-┊ 58┊   ┊        return done(null, user);
-┊ 59┊   ┊      }
-┊ 60┊   ┊      return done(null, false);
-┊ 61┊   ┊    }
-┊ 62┊   ┊  ));
-┊ 63┊   ┊
 ┊ 64┊ 17┊  const PORT = 4000;
-┊ 65┊   ┊  const CLOUDINARY_URL = process.env.CLOUDINARY_URL || '';
 ┊ 66┊ 18┊
 ┊ 67┊ 19┊  const app = express();
 ┊ 68┊ 20┊
 ┊ 69┊ 21┊  app.use(cors());
 ┊ 70┊ 22┊  app.use(bodyParser.json());
-┊ 71┊   ┊  app.use(passport.initialize());
-┊ 72┊   ┊
-┊ 73┊   ┊  app.post('/signup',
-┊ 74┊   ┊    passport.authenticate('basic-signup', {session: false}),
-┊ 75┊   ┊    function (req: any, res) {
-┊ 76┊   ┊      res.json(req.user);
-┊ 77┊   ┊    });
-┊ 78┊   ┊
-┊ 79┊   ┊  app.use(passport.authenticate('basic-signin', {session: false}));
-┊ 80┊   ┊
-┊ 81┊   ┊  app.post('/signin', function (req, res) {
-┊ 82┊   ┊    res.json(req.user);
-┊ 83┊   ┊  });
-┊ 84┊   ┊
-┊ 85┊   ┊  const match = CLOUDINARY_URL.match(/cloudinary:\/\/(\d+):(\w+)@(\.+)/);
 ┊ 86┊ 23┊
-┊ 87┊   ┊  if (match) {
-┊ 88┊   ┊    const [api_key, api_secret, cloud_name] = match.slice(1);
-┊ 89┊   ┊    cloudinary.config({ api_key, api_secret, cloud_name });
-┊ 90┊   ┊  }
-┊ 91┊   ┊
-┊ 92┊   ┊  const upload = multer({
-┊ 93┊   ┊    dest: tmp.dirSync({ unsafeCleanup: true }).name,
-┊ 94┊   ┊  });
+┊   ┊ 24┊  const { schema, context, subscriptions } = AppModule.forRoot({ connection, app, });
 ┊ 95┊ 25┊
-┊ 96┊   ┊  app.post('/upload-profile-pic', upload.single('file'), async (req: any, res, done) => {
-┊ 97┊   ┊    try {
-┊ 98┊   ┊      res.json(await new Promise((resolve, reject) => {
-┊ 99┊   ┊        cloudinary.v2.uploader.upload(req.file.path, (error, result) => {
-┊100┊   ┊          if (error) {
-┊101┊   ┊            reject(error);
-┊102┊   ┊          } else {
-┊103┊   ┊            resolve(result);
-┊104┊   ┊          }
-┊105┊   ┊        })
-┊106┊   ┊      }));
-┊107┊   ┊    } catch (e) {
-┊108┊   ┊      done(e);
-┊109┊   ┊    }
-┊110┊   ┊  });
-┊111┊   ┊
-┊112┊   ┊  const apollo = new ApolloServer({
-┊113┊   ┊    schema,
-┊114┊   ┊    context(received: any) {
-┊115┊   ┊      return {
-┊116┊   ┊        currentUser: received.connection ? received.connection.context.currentUser : received.req!['user'],
-┊117┊   ┊        connection,
-┊118┊   ┊      }
-┊119┊   ┊    },
-┊120┊   ┊    subscriptions: {
-┊121┊   ┊      onConnect: async (connectionParams: any, webSocket: any) => {
-┊122┊   ┊        if (connectionParams.authToken) {
-┊123┊   ┊          // Create a buffer and tell it the data coming in is base64
-┊124┊   ┊          const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
-┊125┊   ┊          // Read it back out as a string
-┊126┊   ┊          const [username, password]: string[] = buf.toString().split(':');
-┊127┊   ┊          if (username && password) {
-┊128┊   ┊            const currentUser = await connection.getRepository(User).findOne({where: { username }});
-┊129┊   ┊
-┊130┊   ┊            if (currentUser && validPassword(password, currentUser.password)) {
-┊131┊   ┊              // Set context for the WebSocket
-┊132┊   ┊              return {currentUser};
-┊133┊   ┊            } else {
-┊134┊   ┊              throw new Error('Wrong credentials!');
-┊135┊   ┊            }
-┊136┊   ┊          }
-┊137┊   ┊        }
-┊138┊   ┊        throw new Error('Missing auth token!');
-┊139┊   ┊      }
-┊140┊   ┊    }
-┊141┊   ┊  });
+┊   ┊ 26┊  const apollo = new ApolloServer({ schema, context, subscriptions });
 ┊142┊ 27┊
 ┊143┊ 28┊  apollo.applyMiddleware({
 ┊144┊ 29┊    app,
```

[}]: #

We also need to adjust our `GraphQL Code Generator` configuration in order to get the schema from our main `App Module`:


[{]: <helper> (diffStep "8.2" files="codegen.yml, schema.ts" module="server")

#### [Step 8.2: Auth Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/671a634)

##### Changed codegen.yml
```diff
@@ -1,19 +1,18 @@
 ┊ 1┊ 1┊overwrite: true
-┊ 2┊  ┊schema: "./schema/typeDefs.ts"
-┊ 3┊  ┊documents: null
-┊ 4┊  ┊require:
-┊ 5┊  ┊  - ts-node/register
+┊  ┊ 2┊schema: ./schema.ts
+┊  ┊ 3┊require: ts-node/register/transpile-only
 ┊ 6┊ 4┊generates:
 ┊ 7┊ 5┊  ./types.d.ts:
+┊  ┊ 6┊    plugins:
+┊  ┊ 7┊      - typescript-common
+┊  ┊ 8┊      - typescript-resolvers
 ┊ 8┊ 9┊    config:
 ┊ 9┊10┊      optionalType: undefined | null
-┊10┊  ┊      contextType: ./schema/types#AppContext
+┊  ┊11┊      contextType: "@graphql-modules/core#ModuleContext"
 ┊11┊12┊      mappers:
 ┊12┊13┊        Chat: ./entity/Chat#Chat
 ┊13┊14┊        Message: ./entity/Message#Message
 ┊14┊15┊        Recipient: ./entity/Recipient#Recipient
 ┊15┊16┊        User: ./entity/User#User
-┊16┊  ┊    plugins:
-┊17┊  ┊      - "typescript-common"
-┊18┊  ┊      - "typescript-server"
-┊19┊  ┊      - "typescript-resolvers"
+┊  ┊17┊      scalars:
+┊  ┊18┊        Date: Date
```

##### Added schema.ts
```diff
@@ -0,0 +1,4 @@
+┊ ┊1┊import 'reflect-metadata';
+┊ ┊2┊import { AppModule } from './modules/app.module';
+┊ ┊3┊
+┊ ┊4┊export default AppModule.forRoot({} as any).typeDefs;
```

[}]: #

Now it's finally time to create the `Auth Module`:

[{]: <helper> (diffStep "8.2" files="modules/auth/index.ts, modules/app.symbols.ts" module="server")

#### [Step 8.2: Auth Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/671a634)

##### Added modules&#x2F;app.symbols.ts
```diff
@@ -0,0 +1 @@
+┊ ┊1┊export const APP = Symbol.for('APP');
```

##### Added modules&#x2F;auth&#x2F;index.ts
```diff
@@ -0,0 +1,55 @@
+┊  ┊ 1┊import { GraphQLModule } from '@graphql-modules/core';
+┊  ┊ 2┊import { loadResolversFiles, loadSchemaFiles } from '@graphql-modules/sonar';
+┊  ┊ 3┊import { Express } from 'express';
+┊  ┊ 4┊import { Connection } from 'typeorm';
+┊  ┊ 5┊import { AuthProvider } from './providers/auth.provider';
+┊  ┊ 6┊import { APP } from '../app.symbols';
+┊  ┊ 7┊import { PubSub } from 'apollo-server-express';
+┊  ┊ 8┊import passport from 'passport';
+┊  ┊ 9┊import basicStrategy from 'passport-http';
+┊  ┊10┊import { InjectFunction } from '@graphql-modules/di';
+┊  ┊11┊
+┊  ┊12┊export interface IAppModuleConfig {
+┊  ┊13┊  connection: Connection,
+┊  ┊14┊  app: Express;
+┊  ┊15┊}
+┊  ┊16┊
+┊  ┊17┊export const AuthModule = new GraphQLModule<IAppModuleConfig>({
+┊  ┊18┊  name: "Auth",
+┊  ┊19┊  providers: ({config: {connection, app}}) => [
+┊  ┊20┊    {provide: Connection, useValue: connection},
+┊  ┊21┊    {provide: APP, useValue: app},
+┊  ┊22┊    PubSub,
+┊  ┊23┊    AuthProvider,
+┊  ┊24┊  ],
+┊  ┊25┊  typeDefs: loadSchemaFiles(__dirname + '/schema/'),
+┊  ┊26┊  resolvers: loadResolversFiles(__dirname + '/resolvers/'),
+┊  ┊27┊  configRequired: true,
+┊  ┊28┊  middleware: InjectFunction(AuthProvider, APP)((authProvider, app: Express) => {
+┊  ┊29┊    passport.use('basic-signin', new basicStrategy.BasicStrategy(
+┊  ┊30┊      async (username: string, password: string, done: any) => {
+┊  ┊31┊        done(
+┊  ┊32┊          null,
+┊  ┊33┊          await authProvider.signIn(username, password)
+┊  ┊34┊        );
+┊  ┊35┊      }
+┊  ┊36┊    ));
+┊  ┊37┊
+┊  ┊38┊    passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
+┊  ┊39┊      async (req: Express.Request & { body: { name?: string }}, username: string, password: string, done: any) => {
+┊  ┊40┊        const name = req.body.name;
+┊  ┊41┊        return done(null, !!name && await authProvider.signUp(username, password, name));
+┊  ┊42┊      }
+┊  ┊43┊    ));
+┊  ┊44┊
+┊  ┊45┊    app.post('/signup',
+┊  ┊46┊      passport.authenticate('basic-signup', {session: false}),
+┊  ┊47┊        (req, res) => res.json(req.user)
+┊  ┊48┊    );
+┊  ┊49┊
+┊  ┊50┊    app.use(passport.authenticate('basic-signin', {session: false}));
+┊  ┊51┊
+┊  ┊52┊    app.post('/signin', (req, res) => res.json(req.user));
+┊  ┊53┊    return {};
+┊  ┊54┊  }),
+┊  ┊55┊});
```

[}]: #

Notice the `middleware` section, where we put all our Express middlewares.

Also note `providers`, where we define all our `Providers` (they're basically services) where we store all the logic.
It gets `config` as parameter, allowing to access the Module configuration: we created a `Connection` provider out of the `connection` configuration option and the same happened for `app`.
Once a provider gets defined it can be injected almost everywhere.
`GraphQL Modules` features a strong encapsulation, so if you want to access another Module's provider you will need to import that Module first.

[{]: <helper> (diffStep "8.2" files="modules/auth/providers/auth.provider.ts" module="server")

#### [Step 8.2: Auth Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/671a634)

##### Added modules&#x2F;auth&#x2F;providers&#x2F;auth.provider.ts
```diff
@@ -0,0 +1,82 @@
+┊  ┊ 1┊import { Injectable, ProviderScope } from '@graphql-modules/di';
+┊  ┊ 2┊import { ModuleSessionInfo, OnRequest, OnConnect } from '@graphql-modules/core';
+┊  ┊ 3┊import { Connection } from 'typeorm';
+┊  ┊ 4┊import { User } from '../../../entity/User';
+┊  ┊ 5┊import bcrypt from 'bcrypt-nodejs';
+┊  ┊ 6┊import { PubSub } from 'apollo-server-express'
+┊  ┊ 7┊
+┊  ┊ 8┊@Injectable({
+┊  ┊ 9┊  scope: ProviderScope.Session
+┊  ┊10┊})
+┊  ┊11┊export class AuthProvider implements OnRequest, OnConnect {
+┊  ┊12┊
+┊  ┊13┊  currentUser: User;
+┊  ┊14┊  constructor(
+┊  ┊15┊    private connection: Connection,
+┊  ┊16┊    private pubsub: PubSub,
+┊  ┊17┊  ) {}
+┊  ┊18┊
+┊  ┊19┊  onRequest({ session }: ModuleSessionInfo) {
+┊  ┊20┊    if ('req' in session) {
+┊  ┊21┊      this.currentUser = session.req.user;
+┊  ┊22┊    }
+┊  ┊23┊  }
+┊  ┊24┊
+┊  ┊25┊  async onConnect(connectionParams: { authToken?: string }) {
+┊  ┊26┊    if (connectionParams.authToken) {
+┊  ┊27┊      // Create a buffer and tell it the data coming in is base64
+┊  ┊28┊      const buf = Buffer.from(connectionParams.authToken.split(' ')[1], 'base64');
+┊  ┊29┊      // Read it back out as a string
+┊  ┊30┊      const [username, password]: string[] = buf.toString().split(':');
+┊  ┊31┊      const user = await this.signIn(username, password);
+┊  ┊32┊      if (user) {
+┊  ┊33┊        // Set context for the WebSocket
+┊  ┊34┊        this.currentUser = user;
+┊  ┊35┊      } else {
+┊  ┊36┊        throw new Error('Wrong credentials!');
+┊  ┊37┊      }
+┊  ┊38┊    } else {
+┊  ┊39┊      throw new Error('Missing auth token!');
+┊  ┊40┊    }
+┊  ┊41┊  }
+┊  ┊42┊
+┊  ┊43┊  getUserByUsername(username: string) {
+┊  ┊44┊    return this.connection.getRepository(User).findOne({where: { username }});
+┊  ┊45┊  }
+┊  ┊46┊
+┊  ┊47┊  async signIn(username: string, password: string): Promise<User | false> {
+┊  ┊48┊    const user = await this.getUserByUsername(username);
+┊  ┊49┊    if (user && this.validPassword(password, user.password)) {
+┊  ┊50┊      return user;
+┊  ┊51┊    } else {
+┊  ┊52┊      return false;
+┊  ┊53┊    }
+┊  ┊54┊  }
+┊  ┊55┊
+┊  ┊56┊  async signUp(username: string, password: string, name: string): Promise<User | false> {
+┊  ┊57┊    const userExists = !!(await this.getUserByUsername(username));
+┊  ┊58┊    if (!userExists) {
+┊  ┊59┊      const user = this.connection.manager.save(
+┊  ┊60┊        new User({
+┊  ┊61┊          username,
+┊  ┊62┊          password: this.generateHash(password),
+┊  ┊63┊          name,
+┊  ┊64┊        })
+┊  ┊65┊      );
+┊  ┊66┊      this.pubsub.publish('userAdded', {
+┊  ┊67┊        userAdded: user,
+┊  ┊68┊      });
+┊  ┊69┊      return user;
+┊  ┊70┊    } else {
+┊  ┊71┊      return false;
+┊  ┊72┊    }
+┊  ┊73┊  }
+┊  ┊74┊
+┊  ┊75┊  generateHash(password: string) {
+┊  ┊76┊    return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
+┊  ┊77┊  }
+┊  ┊78┊
+┊  ┊79┊  validPassword(password: string, localPassword: string) {
+┊  ┊80┊    return bcrypt.compareSync(password, localPassword);
+┊  ┊81┊  }
+┊  ┊82┊}
```

[}]: #

Take a look at the `Auth Provider` and you will notice the `ProviderScope.Session` scope.
That means that a new instance of the `Auth Provider` gets created for each session.
That allows us to get rid of the GraphQL context and simply save the `currentUser` into its own class property.
Every time we want to access the `currentUser` we simply need to access the `Auth Provider` and look at the `currentUser` property.
This is very useful because each Module's context will be different depending on which modules it imports, thus we will need to generate per-module typings.
Instead we preferred getting rid of the context altogether and simply use session-scoped providers instead (the default is application-scoped).

[{]: <helper> (diffStep "8.2" files="modules/auth/resolvers, modules/auth/schema" module="server")

#### [Step 8.2: Auth Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/671a634)

##### Added modules&#x2F;auth&#x2F;resolvers&#x2F;resolvers.ts
```diff
@@ -0,0 +1,6 @@
+┊ ┊1┊import { GraphQLDateTime } from 'graphql-iso-date';
+┊ ┊2┊import { IResolvers } from '../../../types';
+┊ ┊3┊
+┊ ┊4┊export default {
+┊ ┊5┊  Date: GraphQLDateTime,
+┊ ┊6┊} as IResolvers;
```

##### Added modules&#x2F;auth&#x2F;schema&#x2F;typeDefs.graphql
```diff
@@ -0,0 +1 @@
+┊ ┊1┊scalar Date
```

[}]: #

Notice that the `Auth Module` doesn't implement any GraphQL schema except for the scalar `Date` and its resolver.
This is because the authentication happens on the REST endpoints and not through GraphQL (yet, in a later chapter we will cover `Accounts JS`).
Yet for simplicity we decided to put `GraphQLDateTime` into the `Auth Module` instead of creating its own.
Ideally you would want to create a `GraphQLDateTime Module` which takes care of it and you will have to import it in all the modules which makes use of it.

## User Module

[{]: <helper> (diffStep "8.3" files="^\(?!types.d.ts$\).*" module="server")

#### [Step 8.3: User Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/f4ff15b)

##### Changed modules&#x2F;app.module.ts
```diff
@@ -2,6 +2,7 @@
 ┊2┊2┊import { Connection } from 'typeorm';
 ┊3┊3┊import { Express } from 'express';
 ┊4┊4┊import { AuthModule } from './auth';
+┊ ┊5┊import { UserModule } from './user';
 ┊5┊6┊
 ┊6┊7┊export interface IAppModuleConfig {
 ┊7┊8┊  connection: Connection,
```
```diff
@@ -15,6 +16,7 @@
 ┊15┊16┊      connection,
 ┊16┊17┊      app,
 ┊17┊18┊    }),
+┊  ┊19┊    UserModule,
 ┊18┊20┊  ],
 ┊19┊21┊  configRequired: true,
 ┊20┊22┊});
```

##### Added modules&#x2F;user&#x2F;index.ts
```diff
@@ -0,0 +1,46 @@
+┊  ┊ 1┊/// <reference path="../../cloudinary.d.ts" />
+┊  ┊ 2┊import { GraphQLModule } from '@graphql-modules/core';
+┊  ┊ 3┊import { loadResolversFiles, loadSchemaFiles } from '@graphql-modules/sonar';
+┊  ┊ 4┊import { UserProvider } from './providers/user.provider';
+┊  ┊ 5┊import { AuthModule } from '../auth';
+┊  ┊ 6┊import { Express } from 'express';
+┊  ┊ 7┊import multer from 'multer';
+┊  ┊ 8┊import tmp from 'tmp';
+┊  ┊ 9┊import cloudinary from 'cloudinary';
+┊  ┊10┊import { APP } from '../app.symbols';
+┊  ┊11┊import { InjectFunction, ProviderScope } from '@graphql-modules/di';
+┊  ┊12┊export const CLOUDINARY_URL = process.env.CLOUDINARY_URL || '';
+┊  ┊13┊
+┊  ┊14┊export const UserModule = new GraphQLModule({
+┊  ┊15┊  name: 'User',
+┊  ┊16┊  imports: [
+┊  ┊17┊    AuthModule,
+┊  ┊18┊  ],
+┊  ┊19┊  providers: [
+┊  ┊20┊    UserProvider,
+┊  ┊21┊  ],
+┊  ┊22┊  defaultProviderScope: ProviderScope.Session,
+┊  ┊23┊  typeDefs: loadSchemaFiles(__dirname + '/schema/'),
+┊  ┊24┊  resolvers: loadResolversFiles(__dirname + '/resolvers/'),
+┊  ┊25┊  middleware: InjectFunction(UserProvider, APP)((userProvider, app: Express) => {
+┊  ┊26┊    const match = CLOUDINARY_URL.match(/cloudinary:\/\/(\d+):(\w+)@(\.+)/);
+┊  ┊27┊
+┊  ┊28┊    if (match) {
+┊  ┊29┊      const [api_key, api_secret, cloud_name] = match.slice(1);
+┊  ┊30┊      cloudinary.config({ api_key, api_secret, cloud_name });
+┊  ┊31┊    }
+┊  ┊32┊
+┊  ┊33┊    const upload = multer({
+┊  ┊34┊      dest: tmp.dirSync({ unsafeCleanup: true }).name,
+┊  ┊35┊    });
+┊  ┊36┊
+┊  ┊37┊    app.post('/upload-profile-pic', upload.single('file'), async (req: any, res, done) => {
+┊  ┊38┊      try {
+┊  ┊39┊        res.json(await userProvider.uploadProfilePic(req.file.path));
+┊  ┊40┊      } catch (e) {
+┊  ┊41┊        done(e);
+┊  ┊42┊      }
+┊  ┊43┊    });
+┊  ┊44┊    return {};
+┊  ┊45┊  })
+┊  ┊46┊});
```

##### Added modules&#x2F;user&#x2F;providers&#x2F;user.provider.ts
```diff
@@ -0,0 +1,70 @@
+┊  ┊ 1┊import { Injectable } from '@graphql-modules/di'
+┊  ┊ 2┊import { PubSub } from 'apollo-server-express'
+┊  ┊ 3┊import { Connection } from 'typeorm'
+┊  ┊ 4┊import { User } from '../../../entity/User';
+┊  ┊ 5┊import { AuthProvider } from '../../auth/providers/auth.provider';
+┊  ┊ 6┊import cloudinary from 'cloudinary';
+┊  ┊ 7┊
+┊  ┊ 8┊@Injectable()
+┊  ┊ 9┊export class UserProvider {
+┊  ┊10┊
+┊  ┊11┊  constructor(
+┊  ┊12┊    private pubsub: PubSub,
+┊  ┊13┊    private connection: Connection,
+┊  ┊14┊    private authProvider: AuthProvider,
+┊  ┊15┊  ) { }
+┊  ┊16┊
+┊  ┊17┊  public repository = this.connection.getRepository(User);
+┊  ┊18┊  private currentUser = this.authProvider.currentUser;
+┊  ┊19┊
+┊  ┊20┊  createQueryBuilder() {
+┊  ┊21┊    return this.connection.createQueryBuilder(User, 'user');
+┊  ┊22┊  }
+┊  ┊23┊
+┊  ┊24┊  getMe() {
+┊  ┊25┊    return this.currentUser;
+┊  ┊26┊  }
+┊  ┊27┊
+┊  ┊28┊  getUsers() {
+┊  ┊29┊    return this.createQueryBuilder().where('user.id != :id', {id: this.currentUser.id}).getMany();
+┊  ┊30┊  }
+┊  ┊31┊
+┊  ┊32┊  async updateUser({
+┊  ┊33┊    name,
+┊  ┊34┊    picture,
+┊  ┊35┊  }: {
+┊  ┊36┊    name?: string,
+┊  ┊37┊    picture?: string,
+┊  ┊38┊  } = {}) {
+┊  ┊39┊    if (name === this.currentUser.name && picture === this.currentUser.picture) {
+┊  ┊40┊      return this.currentUser;
+┊  ┊41┊    }
+┊  ┊42┊
+┊  ┊43┊    this.currentUser.name = name || this.currentUser.name;
+┊  ┊44┊    this.currentUser.picture = picture || this.currentUser.picture;
+┊  ┊45┊
+┊  ┊46┊    await this.repository.save(this.currentUser);
+┊  ┊47┊
+┊  ┊48┊    this.pubsub.publish('userUpdated', {
+┊  ┊49┊      userUpdated: this.currentUser,
+┊  ┊50┊    });
+┊  ┊51┊
+┊  ┊52┊    return this.currentUser;
+┊  ┊53┊  }
+┊  ┊54┊
+┊  ┊55┊  filterUserAddedOrUpdated(userAddedOrUpdated: User) {
+┊  ┊56┊    return userAddedOrUpdated.id !== this.currentUser.id;
+┊  ┊57┊  }
+┊  ┊58┊
+┊  ┊59┊  uploadProfilePic(filePath: string) {
+┊  ┊60┊    return new Promise((resolve, reject) => {
+┊  ┊61┊      cloudinary.v2.uploader.upload(filePath, (error, result) => {
+┊  ┊62┊        if (error) {
+┊  ┊63┊          reject(error);
+┊  ┊64┊        } else {
+┊  ┊65┊          resolve(result);
+┊  ┊66┊        }
+┊  ┊67┊      })
+┊  ┊68┊    });
+┊  ┊69┊  }
+┊  ┊70┊}
```

##### Added modules&#x2F;user&#x2F;resolvers&#x2F;resolvers.ts
```diff
@@ -0,0 +1,33 @@
+┊  ┊ 1┊import { PubSub } from 'apollo-server-express';
+┊  ┊ 2┊import { withFilter } from 'apollo-server-express';
+┊  ┊ 3┊import { UserProvider } from '../providers/user.provider';
+┊  ┊ 4┊import { User } from '../../../entity/User';
+┊  ┊ 5┊import { IResolvers } from '../../../types';
+┊  ┊ 6┊import { ModuleContext } from '@graphql-modules/core';
+┊  ┊ 7┊
+┊  ┊ 8┊export default {
+┊  ┊ 9┊  Query: {
+┊  ┊10┊    me: (_, __, { injector }) => injector.get(UserProvider).getMe(),
+┊  ┊11┊    users: (obj, args, { injector }) => injector.get(UserProvider).getUsers(),
+┊  ┊12┊  },
+┊  ┊13┊  Mutation: {
+┊  ┊14┊    updateUser: (obj, {name, picture}, { injector }) => injector.get(UserProvider).updateUser({
+┊  ┊15┊        name: name || '',
+┊  ┊16┊        picture: picture || '',
+┊  ┊17┊      }),
+┊  ┊18┊  },
+┊  ┊19┊  Subscription: {
+┊  ┊20┊    userAdded: {
+┊  ┊21┊      subscribe: withFilter(
+┊  ┊22┊        (root, args, { injector }: ModuleContext) => injector.get(PubSub).asyncIterator('userAdded'),
+┊  ┊23┊        (data: { userAdded: User }, variables, { injector }: ModuleContext) => data && injector.get(UserProvider).filterUserAddedOrUpdated(data.userAdded),
+┊  ┊24┊      ),
+┊  ┊25┊    },
+┊  ┊26┊    userUpdated: {
+┊  ┊27┊      subscribe: withFilter(
+┊  ┊28┊        (root, args, { injector }: ModuleContext) => injector.get(PubSub).asyncIterator('userUpdated'),
+┊  ┊29┊        (data: { userUpdated: User }, variables, { injector }: ModuleContext) => data && injector.get(UserProvider).filterUserAddedOrUpdated(data.userUpdated)
+┊  ┊30┊      ),
+┊  ┊31┊    },
+┊  ┊32┊  },
+┊  ┊33┊} as IResolvers;
```

##### Added modules&#x2F;user&#x2F;schema&#x2F;typeDefs.graphql
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊type Query {
+┊  ┊ 2┊  me: User
+┊  ┊ 3┊  users: [User!]
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊type Mutation {
+┊  ┊ 7┊  updateUser(name: String, picture: String): User!
+┊  ┊ 8┊}
+┊  ┊ 9┊
+┊  ┊10┊type Subscription {
+┊  ┊11┊  userAdded: User
+┊  ┊12┊  userUpdated: User
+┊  ┊13┊}
+┊  ┊14┊
+┊  ┊15┊type User {
+┊  ┊16┊  id: ID!
+┊  ┊17┊  name: String
+┊  ┊18┊  picture: String
+┊  ┊19┊  phone: String
+┊  ┊20┊}
```

[}]: #

As you can notice we still have some middleware inside the `User Module`, because the image upload is done through a REST endpoint instead of a GraphQL one.
This will change with a future update.

Also notice how tidy the resolvers look: this is because all of our logic gone into Providers.

## Chat, Message and Recipient Modules

Now this is where the hard part begins, because we have to figure out how to divide the rest of our applications into Modules.
The first and most important concept to realize is that each Module should be able to work on its own (the module itself plus all of its dependencies).
That means that if we are going to remove all the other modules it should still keep working.
`GraphQL Modules` has strong encapsulation (it forbids access to other modules unless dependencies are explicitly stated), which forces us into this kind of behaviour.
This is great for team development and testability because each module can be worked and tested on its own (we could even import Modules directly from `npm`).
On the other side, sometimes figuring out relationships between Modules can be hard and we will soon realize why.
For this app we are going to create three more modules: `Chat Module`, `Message Module` and `Recipient Module`.
Let's focus on the first two for the moment.
It's obvious that the `Message Module` will have to depend on the `Chat Module`, because to create a new message you will need the chat id.
That means that `Chat Module` cannot depend on `Message Module`, because otherwise we will have circular dependencies.
Circular dependencies basically mean that those modules won't be able to work on their own: each one of them will require the other.
They are basically no modules at all: it's basically like splitting the code in two different files, but you still won't be able to develop and test them separately.
To avoid abuse of circular dependencies we recently removed that feature from `GraphQL Modules`.
At first look it doesn't look like there is any way to avoid a dependency on `Message Module`, because of the `messages` field inside `Chat`.
That's not a problem at all, because we are going to implement that field inside the `Message Module` itself (it depends on `Chat Module`, so it can extend it).
Now if you think there are no more dependencies towards `Message Module` you would be wrong, because in the `chats` resolver we order the chats based on the latest message date.
The fix is very easy: just return them in the natural order and override the `chats` resolver from the `Message Module` to order them according to messages.
That means that if someone is going to import the `Chat Module` as standalone the chats will be returned in no particular order, while if he imports the `Message Module` the order will be according to messages.
But that's not finished yet, because if you take a closer look at the `removeChat` mutation you will notice that it also removes all related messages.
That's a big problem, because if we are going to override the whole mutation we will end up duplicating lots of code.
To avoid that, we will get access to the `Chat Module` `removeChat` mutation from the `Message Module` (it's as easy as injecting its provider and calling it).
Once called we will further expand it to also remove the related messages.
It's done! But you could ask yourself, what's the point of a standalone `Chat Module` if we don't have messages? How can it be useful at all?
What's misleading here is the name: a better name would be the `Group Module`.
In fact what it does is simply creating groups of two or more users and it could be used for basically anything.

[{]: <helper> (diffStep "8.4" files="^\(?!types.d.ts$\).*" module="server")

#### [Step 8.4: Chat Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/1756642)

##### Changed modules&#x2F;app.module.ts
```diff
@@ -3,6 +3,7 @@
 ┊3┊3┊import { Express } from 'express';
 ┊4┊4┊import { AuthModule } from './auth';
 ┊5┊5┊import { UserModule } from './user';
+┊ ┊6┊import { ChatModule } from './chat';
 ┊6┊7┊
 ┊7┊8┊export interface IAppModuleConfig {
 ┊8┊9┊  connection: Connection,
```
```diff
@@ -17,6 +18,7 @@
 ┊17┊18┊      app,
 ┊18┊19┊    }),
 ┊19┊20┊    UserModule,
+┊  ┊21┊    ChatModule,
 ┊20┊22┊  ],
 ┊21┊23┊  configRequired: true,
 ┊22┊24┊});
```

##### Added modules&#x2F;chat&#x2F;index.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import { GraphQLModule } from '@graphql-modules/core';
+┊  ┊ 2┊import { loadResolversFiles, loadSchemaFiles } from '@graphql-modules/sonar';
+┊  ┊ 3┊import { UserModule } from '../user';
+┊  ┊ 4┊import { ChatProvider } from './providers/chat.provider';
+┊  ┊ 5┊import { AuthModule } from '../auth';
+┊  ┊ 6┊import { ProviderScope } from '@graphql-modules/di';
+┊  ┊ 7┊
+┊  ┊ 8┊export const ChatModule = new GraphQLModule({
+┊  ┊ 9┊  name: "Chat",
+┊  ┊10┊  imports: [
+┊  ┊11┊    AuthModule,
+┊  ┊12┊    UserModule,
+┊  ┊13┊  ],
+┊  ┊14┊  providers: [
+┊  ┊15┊    ChatProvider,
+┊  ┊16┊  ],
+┊  ┊17┊  defaultProviderScope: ProviderScope.Session,
+┊  ┊18┊  typeDefs: loadSchemaFiles(__dirname + '/schema/'),
+┊  ┊19┊  resolvers: loadResolversFiles(__dirname + '/resolvers/'),
+┊  ┊20┊});
```

##### Added modules&#x2F;chat&#x2F;providers&#x2F;chat.provider.ts
```diff
@@ -0,0 +1,389 @@
+┊   ┊  1┊import { Injectable } from '@graphql-modules/di'
+┊   ┊  2┊import { PubSub } from 'apollo-server-express'
+┊   ┊  3┊import { Connection } from 'typeorm'
+┊   ┊  4┊import { User } from '../../../entity/User';
+┊   ┊  5┊import { Chat } from '../../../entity/Chat';
+┊   ┊  6┊import { UserProvider } from '../../user/providers/user.provider';
+┊   ┊  7┊import { AuthProvider } from '../../auth/providers/auth.provider';
+┊   ┊  8┊
+┊   ┊  9┊@Injectable()
+┊   ┊ 10┊export class ChatProvider {
+┊   ┊ 11┊  constructor(
+┊   ┊ 12┊    private pubsub: PubSub,
+┊   ┊ 13┊    private connection: Connection,
+┊   ┊ 14┊    private userProvider: UserProvider,
+┊   ┊ 15┊    private authProvider: AuthProvider,
+┊   ┊ 16┊  ) {
+┊   ┊ 17┊  }
+┊   ┊ 18┊
+┊   ┊ 19┊  repository = this.connection.getRepository(Chat);
+┊   ┊ 20┊  currentUser = this.authProvider.currentUser;
+┊   ┊ 21┊
+┊   ┊ 22┊  createQueryBuilder() {
+┊   ┊ 23┊    return this.connection.createQueryBuilder(Chat, 'chat');
+┊   ┊ 24┊  }
+┊   ┊ 25┊
+┊   ┊ 26┊  async getChats() {
+┊   ┊ 27┊    return this
+┊   ┊ 28┊      .createQueryBuilder()
+┊   ┊ 29┊      .leftJoin('chat.listingMembers', 'listingMembers')
+┊   ┊ 30┊      .where('listingMembers.id = :id', { id: this.currentUser.id })
+┊   ┊ 31┊      .orderBy('chat.createdAt', 'DESC')
+┊   ┊ 32┊      .getMany();
+┊   ┊ 33┊  }
+┊   ┊ 34┊
+┊   ┊ 35┊  async getChat(chatId: string) {
+┊   ┊ 36┊    const chat = await this
+┊   ┊ 37┊      .createQueryBuilder()
+┊   ┊ 38┊      .whereInIds(chatId)
+┊   ┊ 39┊      .getOne();
+┊   ┊ 40┊
+┊   ┊ 41┊    return chat || null;
+┊   ┊ 42┊  }
+┊   ┊ 43┊
+┊   ┊ 44┊  async addChat(userId: string) {
+┊   ┊ 45┊    const user = await this.userProvider
+┊   ┊ 46┊      .createQueryBuilder()
+┊   ┊ 47┊      .whereInIds(userId)
+┊   ┊ 48┊      .getOne();
+┊   ┊ 49┊
+┊   ┊ 50┊    if (!user) {
+┊   ┊ 51┊      throw new Error(`User ${userId} doesn't exist.`);
+┊   ┊ 52┊    }
+┊   ┊ 53┊
+┊   ┊ 54┊    let chat = await this
+┊   ┊ 55┊      .createQueryBuilder()
+┊   ┊ 56┊      .where('chat.name IS NULL')
+┊   ┊ 57┊      .innerJoin('chat.allTimeMembers', 'allTimeMembers1', 'allTimeMembers1.id = :currentUserId', {
+┊   ┊ 58┊        currentUserId: this.currentUser.id,
+┊   ┊ 59┊      })
+┊   ┊ 60┊      .innerJoin('chat.allTimeMembers', 'allTimeMembers2', 'allTimeMembers2.id = :userId', {
+┊   ┊ 61┊        userId: userId,
+┊   ┊ 62┊      })
+┊   ┊ 63┊      .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊ 64┊      .getOne();
+┊   ┊ 65┊
+┊   ┊ 66┊    if (chat) {
+┊   ┊ 67┊      // Chat already exists. Both users are already in the userIds array
+┊   ┊ 68┊      const listingMembers = await this.userProvider
+┊   ┊ 69┊        .createQueryBuilder()
+┊   ┊ 70┊        .innerJoin(
+┊   ┊ 71┊          'user.listingMemberChats',
+┊   ┊ 72┊          'listingMemberChats',
+┊   ┊ 73┊          'listingMemberChats.id = :chatId',
+┊   ┊ 74┊          { chatId: chat.id },
+┊   ┊ 75┊        )
+┊   ┊ 76┊        .getMany();
+┊   ┊ 77┊
+┊   ┊ 78┊      if (!listingMembers.find(user => user.id === this.currentUser.id)) {
+┊   ┊ 79┊        // The chat isn't listed for the current user. Add him to the memberIds
+┊   ┊ 80┊        chat.listingMembers.push(this.currentUser);
+┊   ┊ 81┊        chat = await this.repository.save(chat);
+┊   ┊ 82┊
+┊   ┊ 83┊        return chat || null;
+┊   ┊ 84┊      } else {
+┊   ┊ 85┊        return chat;
+┊   ┊ 86┊      }
+┊   ┊ 87┊    } else {
+┊   ┊ 88┊      // Create the chat
+┊   ┊ 89┊      chat = await this.repository.save(
+┊   ┊ 90┊        new Chat({
+┊   ┊ 91┊          allTimeMembers: [this.currentUser, user],
+┊   ┊ 92┊          // Chat will not be listed to the other user until the first message gets written
+┊   ┊ 93┊          listingMembers: [this.currentUser],
+┊   ┊ 94┊        }),
+┊   ┊ 95┊      );
+┊   ┊ 96┊
+┊   ┊ 97┊      return chat || null;
+┊   ┊ 98┊    }
+┊   ┊ 99┊  }
+┊   ┊100┊
+┊   ┊101┊  async addGroup(
+┊   ┊102┊    userIds: string[],
+┊   ┊103┊    {
+┊   ┊104┊      groupName,
+┊   ┊105┊      groupPicture,
+┊   ┊106┊    }: {
+┊   ┊107┊      groupName?: string
+┊   ┊108┊      groupPicture?: string
+┊   ┊109┊    } = {},
+┊   ┊110┊  ) {
+┊   ┊111┊    let users: User[] = [];
+┊   ┊112┊    for (let userId of userIds) {
+┊   ┊113┊      const user = await this.userProvider
+┊   ┊114┊        .createQueryBuilder()
+┊   ┊115┊        .whereInIds(userId)
+┊   ┊116┊        .getOne();
+┊   ┊117┊
+┊   ┊118┊      if (!user) {
+┊   ┊119┊        throw new Error(`User ${userId} doesn't exist.`);
+┊   ┊120┊      }
+┊   ┊121┊
+┊   ┊122┊      users.push(user);
+┊   ┊123┊    }
+┊   ┊124┊
+┊   ┊125┊    const chat = await this.repository.save(
+┊   ┊126┊      new Chat({
+┊   ┊127┊        name: groupName,
+┊   ┊128┊        admins: [this.currentUser],
+┊   ┊129┊        picture: groupPicture || undefined,
+┊   ┊130┊        owner: this.currentUser,
+┊   ┊131┊        allTimeMembers: [...users, this.currentUser],
+┊   ┊132┊        listingMembers: [...users, this.currentUser],
+┊   ┊133┊        actualGroupMembers: [...users, this.currentUser],
+┊   ┊134┊      }),
+┊   ┊135┊    );
+┊   ┊136┊
+┊   ┊137┊    this.pubsub.publish('chatAdded', {
+┊   ┊138┊      creatorId: this.currentUser.id,
+┊   ┊139┊      chatAdded: chat,
+┊   ┊140┊    });
+┊   ┊141┊
+┊   ┊142┊    return chat || null;
+┊   ┊143┊  }
+┊   ┊144┊
+┊   ┊145┊  async updateGroup(
+┊   ┊146┊    chatId: string,
+┊   ┊147┊    {
+┊   ┊148┊      groupName,
+┊   ┊149┊      groupPicture,
+┊   ┊150┊    }: {
+┊   ┊151┊      groupName?: string
+┊   ┊152┊      groupPicture?: string
+┊   ┊153┊    } = {},
+┊   ┊154┊  ) {
+┊   ┊155┊    const chat = await this.createQueryBuilder()
+┊   ┊156┊      .whereInIds(chatId)
+┊   ┊157┊      .getOne();
+┊   ┊158┊
+┊   ┊159┊    if (!chat) {
+┊   ┊160┊      throw new Error(`The chat ${chatId} doesn't exist.`);
+┊   ┊161┊    }
+┊   ┊162┊
+┊   ┊163┊    if (!chat.name) {
+┊   ┊164┊      throw new Error(`The chat ${chatId} is not a group.`);
+┊   ┊165┊    }
+┊   ┊166┊
+┊   ┊167┊    chat.name = groupName || chat.name;
+┊   ┊168┊    chat.picture = groupPicture || chat.picture;
+┊   ┊169┊
+┊   ┊170┊    // Update the chat
+┊   ┊171┊    await this.repository.save(chat);
+┊   ┊172┊
+┊   ┊173┊    this.pubsub.publish('chatUpdated', {
+┊   ┊174┊      updaterId: this.currentUser.id,
+┊   ┊175┊      chatUpdated: chat,
+┊   ┊176┊    });
+┊   ┊177┊
+┊   ┊178┊    return chat;
+┊   ┊179┊  }
+┊   ┊180┊
+┊   ┊181┊  async removeChat(chatId: string) {
+┊   ┊182┊    const chat = await this.createQueryBuilder()
+┊   ┊183┊      .whereInIds(Number(chatId))
+┊   ┊184┊      .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊185┊      .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊186┊      .leftJoinAndSelect('chat.admins', 'admins')
+┊   ┊187┊      .leftJoinAndSelect('chat.owner', 'owner')
+┊   ┊188┊      .getOne();
+┊   ┊189┊
+┊   ┊190┊    if (!chat) {
+┊   ┊191┊      throw new Error(`The chat ${chatId} doesn't exist.`)
+┊   ┊192┊    }
+┊   ┊193┊
+┊   ┊194┊    if (!chat.name) {
+┊   ┊195┊      // Chat
+┊   ┊196┊      if (!chat.listingMembers.find(user => user.id === this.currentUser.id)) {
+┊   ┊197┊        throw new Error(`The user is not a listing member of the chat ${chatId}.`)
+┊   ┊198┊      }
+┊   ┊199┊
+┊   ┊200┊      // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
+┊   ┊201┊      chat.listingMembers = chat.listingMembers.filter(user => user.id !== this.currentUser.id);
+┊   ┊202┊
+┊   ┊203┊      // Check how many members are left
+┊   ┊204┊      if (chat.listingMembers.length === 0) {
+┊   ┊205┊        // Delete the chat
+┊   ┊206┊        await this.repository.remove(chat);
+┊   ┊207┊      } else {
+┊   ┊208┊        // Update the chat
+┊   ┊209┊        await this.repository.save(chat);
+┊   ┊210┊      }
+┊   ┊211┊
+┊   ┊212┊      return chatId;
+┊   ┊213┊    } else {
+┊   ┊214┊      // Group
+┊   ┊215┊
+┊   ┊216┊      // Remove the current user from who gets the group listed. The group will no longer appear in his list
+┊   ┊217┊      chat.listingMembers = chat.listingMembers.filter(user => user.id !== this.currentUser.id);
+┊   ┊218┊
+┊   ┊219┊      // Check how many members (including previous ones who can still access old messages) are left
+┊   ┊220┊      if (chat.listingMembers.length === 0) {
+┊   ┊221┊        // Remove the group
+┊   ┊222┊        await this.repository.remove(chat);
+┊   ┊223┊      } else {
+┊   ┊224┊        // Update the group
+┊   ┊225┊
+┊   ┊226┊        // Remove the current user from the chat members. He is no longer a member of the group
+┊   ┊227┊        chat.actualGroupMembers = chat.actualGroupMembers && chat.actualGroupMembers.filter(user =>
+┊   ┊228┊          user.id !== this.currentUser.id
+┊   ┊229┊        );
+┊   ┊230┊        // Remove the current user from the chat admins
+┊   ┊231┊        chat.admins = chat.admins && chat.admins.filter(user => user.id !== this.currentUser.id);
+┊   ┊232┊        // If there are no more admins left the group goes read only
+┊   ┊233┊        // A null owner means the group is read-only
+┊   ┊234┊        chat.owner = chat.admins && chat.admins[0] || null;
+┊   ┊235┊
+┊   ┊236┊        await this.repository.save(chat);
+┊   ┊237┊      }
+┊   ┊238┊
+┊   ┊239┊      return chatId;
+┊   ┊240┊    }
+┊   ┊241┊  }
+┊   ┊242┊
+┊   ┊243┊  async getChatName(chat: Chat) {
+┊   ┊244┊    if (chat.name) {
+┊   ┊245┊      return chat.name;
+┊   ┊246┊    }
+┊   ┊247┊
+┊   ┊248┊    const user = await this.userProvider
+┊   ┊249┊      .createQueryBuilder()
+┊   ┊250┊      .where('user.id != :userId', { userId: this.currentUser.id })
+┊   ┊251┊      .innerJoin(
+┊   ┊252┊        'user.allTimeMemberChats',
+┊   ┊253┊        'allTimeMemberChats',
+┊   ┊254┊        'allTimeMemberChats.id = :chatId',
+┊   ┊255┊        { chatId: chat.id },
+┊   ┊256┊      )
+┊   ┊257┊      .getOne();
+┊   ┊258┊
+┊   ┊259┊    return (user && user.name) || null;
+┊   ┊260┊  }
+┊   ┊261┊
+┊   ┊262┊  async getChatPicture(chat: Chat) {
+┊   ┊263┊
+┊   ┊264┊    if (chat.name) {
+┊   ┊265┊      return chat.picture;
+┊   ┊266┊    }
+┊   ┊267┊
+┊   ┊268┊    const user = await this.userProvider
+┊   ┊269┊      .createQueryBuilder()
+┊   ┊270┊      .where('user.id != :userId', { userId: this.currentUser.id })
+┊   ┊271┊      .innerJoin(
+┊   ┊272┊        'user.allTimeMemberChats',
+┊   ┊273┊        'allTimeMemberChats',
+┊   ┊274┊        'allTimeMemberChats.id = :chatId',
+┊   ┊275┊        { chatId: chat.id },
+┊   ┊276┊      )
+┊   ┊277┊      .getOne();
+┊   ┊278┊
+┊   ┊279┊    return user ? user.picture : null;
+┊   ┊280┊  }
+┊   ┊281┊
+┊   ┊282┊  getChatAllTimeMembers(chat: Chat) {
+┊   ┊283┊    return this.userProvider
+┊   ┊284┊      .createQueryBuilder()
+┊   ┊285┊      .innerJoin(
+┊   ┊286┊        'user.listingMemberChats',
+┊   ┊287┊        'listingMemberChats',
+┊   ┊288┊        'listingMemberChats.id = :chatId',
+┊   ┊289┊        { chatId: chat.id },
+┊   ┊290┊      )
+┊   ┊291┊      .getMany()
+┊   ┊292┊  }
+┊   ┊293┊
+┊   ┊294┊  getChatListingMembers(chat: Chat) {
+┊   ┊295┊    return this.userProvider
+┊   ┊296┊      .createQueryBuilder()
+┊   ┊297┊      .innerJoin(
+┊   ┊298┊        'user.listingMemberChats',
+┊   ┊299┊        'listingMemberChats',
+┊   ┊300┊        'listingMemberChats.id = :chatId',
+┊   ┊301┊        { chatId: chat.id },
+┊   ┊302┊      )
+┊   ┊303┊      .getMany();
+┊   ┊304┊  }
+┊   ┊305┊
+┊   ┊306┊  getChatActualGroupMembers(chat: Chat) {
+┊   ┊307┊    return this.userProvider
+┊   ┊308┊      .createQueryBuilder()
+┊   ┊309┊      .innerJoin(
+┊   ┊310┊        'user.actualGroupMemberChats',
+┊   ┊311┊        'actualGroupMemberChats',
+┊   ┊312┊        'actualGroupMemberChats.id = :chatId',
+┊   ┊313┊        { chatId: chat.id },
+┊   ┊314┊      )
+┊   ┊315┊      .getMany();
+┊   ┊316┊  }
+┊   ┊317┊
+┊   ┊318┊  getChatAdmins(chat: Chat) {
+┊   ┊319┊    return this.userProvider
+┊   ┊320┊      .createQueryBuilder()
+┊   ┊321┊      .innerJoin('user.adminChats', 'adminChats', 'adminChats.id = :chatId', {
+┊   ┊322┊        chatId: chat.id,
+┊   ┊323┊      })
+┊   ┊324┊      .getMany();
+┊   ┊325┊  }
+┊   ┊326┊
+┊   ┊327┊  async getChatOwner(chat: Chat) {
+┊   ┊328┊    const owner = await this.userProvider
+┊   ┊329┊      .createQueryBuilder()
+┊   ┊330┊      .innerJoin('user.ownerChats', 'ownerChats', 'ownerChats.id = :chatId', {
+┊   ┊331┊        chatId: chat.id,
+┊   ┊332┊      })
+┊   ┊333┊      .getOne();
+┊   ┊334┊
+┊   ┊335┊    return owner || null;
+┊   ┊336┊  }
+┊   ┊337┊
+┊   ┊338┊  async isChatGroup(chat: Chat) {
+┊   ┊339┊    return !!chat.name;
+┊   ┊340┊  }
+┊   ┊341┊
+┊   ┊342┊  async filterChatAddedOrUpdated(chatAddedOrUpdated: Chat, creatorOrUpdaterId: number) {
+┊   ┊343┊
+┊   ┊344┊    return Number(creatorOrUpdaterId) !== this.currentUser.id &&
+┊   ┊345┊      chatAddedOrUpdated.listingMembers.some((user: User) => user.id === this.currentUser.id);
+┊   ┊346┊  }
+┊   ┊347┊
+┊   ┊348┊  async updateUser({
+┊   ┊349┊    name,
+┊   ┊350┊    picture,
+┊   ┊351┊  }: {
+┊   ┊352┊    name?: string,
+┊   ┊353┊    picture?: string,
+┊   ┊354┊  } = {}) {
+┊   ┊355┊    await this.userProvider.updateUser({ name, picture });
+┊   ┊356┊
+┊   ┊357┊
+┊   ┊358┊    const data = await this.connection
+┊   ┊359┊      .createQueryBuilder(User, 'user')
+┊   ┊360┊      .where('user.id = :id', { id: this.currentUser.id })
+┊   ┊361┊      // Get a list of the chats who have/had currentUser involved
+┊   ┊362┊      .innerJoinAndSelect(
+┊   ┊363┊        'user.allTimeMemberChats',
+┊   ┊364┊        'allTimeMemberChats',
+┊   ┊365┊        // Groups are unaffected
+┊   ┊366┊        'allTimeMemberChats.name IS NULL',
+┊   ┊367┊      )
+┊   ┊368┊      // We need to notify only those who get the chat listed (except currentUser of course)
+┊   ┊369┊      .innerJoin(
+┊   ┊370┊        'allTimeMemberChats.listingMembers',
+┊   ┊371┊        'listingMembers',
+┊   ┊372┊        'listingMembers.id != :currentUserId',
+┊   ┊373┊        {
+┊   ┊374┊          currentUserId: this.currentUser.id,
+┊   ┊375┊        })
+┊   ┊376┊      .getOne();
+┊   ┊377┊
+┊   ┊378┊    const chatsAffected = data && data.allTimeMemberChats || [];
+┊   ┊379┊
+┊   ┊380┊    chatsAffected.forEach(chat => {
+┊   ┊381┊      this.pubsub.publish('chatUpdated', {
+┊   ┊382┊        updaterId: this.currentUser.id,
+┊   ┊383┊        chatUpdated: chat,
+┊   ┊384┊      })
+┊   ┊385┊    });
+┊   ┊386┊
+┊   ┊387┊    return this.currentUser;
+┊   ┊388┊  }
+┊   ┊389┊}
```

##### Added modules&#x2F;chat&#x2F;resolvers&#x2F;resolvers.ts
```diff
@@ -0,0 +1,53 @@
+┊  ┊ 1┊import { PubSub, withFilter } from 'apollo-server-express';
+┊  ┊ 2┊import { Chat } from '../../../entity/Chat';
+┊  ┊ 3┊import { IResolvers } from '../../../types';
+┊  ┊ 4┊import { ChatProvider } from '../providers/chat.provider';
+┊  ┊ 5┊import { ModuleContext } from '@graphql-modules/core';
+┊  ┊ 6┊
+┊  ┊ 7┊export default {
+┊  ┊ 8┊  Query: {
+┊  ┊ 9┊    chats: (obj, args, { injector }) => injector.get(ChatProvider).getChats(),
+┊  ┊10┊    chat: (obj, { chatId }, { injector }) => injector.get(ChatProvider).getChat(chatId),
+┊  ┊11┊  },
+┊  ┊12┊  Mutation: {
+┊  ┊13┊    addChat: (obj, { userId }, { injector }) => injector.get(ChatProvider).addChat(userId),
+┊  ┊14┊    addGroup: (obj, { userIds, groupName, groupPicture }, { injector }) =>
+┊  ┊15┊      injector.get(ChatProvider).addGroup(userIds, {
+┊  ┊16┊        groupName: groupName || '',
+┊  ┊17┊        groupPicture: groupPicture || '',
+┊  ┊18┊      }),
+┊  ┊19┊    updateGroup: (obj, { chatId, groupName, groupPicture }, { injector }) => injector.get(ChatProvider).updateGroup(chatId, {
+┊  ┊20┊      groupName: groupName || '',
+┊  ┊21┊      groupPicture: groupPicture || '',
+┊  ┊22┊    }),
+┊  ┊23┊    removeChat: (obj, { chatId }, { injector }) => injector.get(ChatProvider).removeChat(chatId),
+┊  ┊24┊    updateUser: (obj, { name, picture }, { injector }) => injector.get(ChatProvider).updateUser({
+┊  ┊25┊      name: name || '',
+┊  ┊26┊      picture: picture || '',
+┊  ┊27┊    }),
+┊  ┊28┊  },
+┊  ┊29┊  Subscription: {
+┊  ┊30┊    chatAdded: {
+┊  ┊31┊      subscribe: withFilter((root, args, { injector }: ModuleContext) => injector.get(PubSub).asyncIterator('chatAdded'),
+┊  ┊32┊        (data: { chatAdded: Chat, creatorId: number }, variables, { injector }: ModuleContext) =>
+┊  ┊33┊          data && injector.get(ChatProvider).filterChatAddedOrUpdated(data.chatAdded, data.creatorId)
+┊  ┊34┊      ),
+┊  ┊35┊    },
+┊  ┊36┊    chatUpdated: {
+┊  ┊37┊      subscribe: withFilter((root, args, { injector }: ModuleContext) => injector.get(PubSub).asyncIterator('chatUpdated'),
+┊  ┊38┊        (data: { chatUpdated: Chat, updaterId: number }, variables, { injector }: ModuleContext) =>
+┊  ┊39┊          data && injector.get(ChatProvider).filterChatAddedOrUpdated(data.chatUpdated, data.updaterId)
+┊  ┊40┊      ),
+┊  ┊41┊    },
+┊  ┊42┊  },
+┊  ┊43┊  Chat: {
+┊  ┊44┊    name: (chat, args, { injector }) => injector.get(ChatProvider).getChatName(chat),
+┊  ┊45┊    picture: (chat, args, { injector }) => injector.get(ChatProvider).getChatPicture(chat),
+┊  ┊46┊    allTimeMembers: (chat, args, { injector }) => injector.get(ChatProvider).getChatAllTimeMembers(chat),
+┊  ┊47┊    listingMembers: (chat, args, { injector }) => injector.get(ChatProvider).getChatListingMembers(chat),
+┊  ┊48┊    actualGroupMembers: (chat, args, { injector }) => injector.get(ChatProvider).getChatActualGroupMembers(chat),
+┊  ┊49┊    admins: (chat, args, { injector }) => injector.get(ChatProvider).getChatAdmins(chat),
+┊  ┊50┊    owner: (chat, args, { injector }) => injector.get(ChatProvider).getChatOwner(chat),
+┊  ┊51┊    isGroup: (chat, args, { injector }) => injector.get(ChatProvider).isChatGroup(chat),
+┊  ┊52┊  },
+┊  ┊53┊} as IResolvers;
```

##### Added modules&#x2F;chat&#x2F;schema&#x2F;typeDefs.graphql
```diff
@@ -0,0 +1,42 @@
+┊  ┊ 1┊type Query {
+┊  ┊ 2┊  chats: [Chat!]!
+┊  ┊ 3┊  chat(chatId: ID!): Chat
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊type Subscription {
+┊  ┊ 7┊  chatAdded: Chat
+┊  ┊ 8┊  chatUpdated: Chat
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊type Chat {
+┊  ┊12┊  #May be a chat or a group
+┊  ┊13┊  id: ID!
+┊  ┊14┊  createdAt: Date!
+┊  ┊15┊  #Computed for chats
+┊  ┊16┊  name: String
+┊  ┊17┊  #Computed for chats
+┊  ┊18┊  picture: String
+┊  ┊19┊  #All members, current and past ones. Includes users who still didn't get the chat listed.
+┊  ┊20┊  allTimeMembers: [User!]!
+┊  ┊21┊  #Whoever gets the chat listed. For groups includes past members who still didn't delete the group. For chats they are the only ones who can send messages.
+┊  ┊22┊  listingMembers: [User!]!
+┊  ┊23┊  #Actual members of the group. Null for chats. For groups they are the only ones who can send messages. They aren't the only ones who get the group listed.
+┊  ┊24┊  actualGroupMembers: [User!]
+┊  ┊25┊  #Null for chats
+┊  ┊26┊  admins: [User!]
+┊  ┊27┊  #If null the group is read-only. Null for chats.
+┊  ┊28┊  owner: User
+┊  ┊29┊  #Computed property
+┊  ┊30┊  isGroup: Boolean!
+┊  ┊31┊}
+┊  ┊32┊
+┊  ┊33┊type Mutation {
+┊  ┊34┊  addChat(userId: ID!): Chat
+┊  ┊35┊  addGroup(userIds: [ID!]!, groupName: String!, groupPicture: String): Chat
+┊  ┊36┊  updateGroup(chatId: ID!, groupName: String, groupPicture: String): Chat
+┊  ┊37┊  removeChat(chatId: ID!): ID
+┊  ┊38┊  addAdmins(groupId: ID!, userIds: [ID!]!): [ID]!
+┊  ┊39┊  removeAdmins(groupId: ID!, userIds: [ID!]!): [ID]!
+┊  ┊40┊  addMembers(groupId: ID!, userIds: [ID!]!): [ID]!
+┊  ┊41┊  removeMembers(groupId: ID!, userIds: [ID!]!): [ID]!
+┊  ┊42┊}
```

[}]: #

Notice tha inside the `Chat Provider` we injected the `User Provider` in order to access the `currentUser`.

[{]: <helper> (diffStep "8.5" files="^\(?!types.d.ts$\).*" module="server")

#### [Step 8.5: Message Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/ab48690)

##### Changed modules&#x2F;app.module.ts
```diff
@@ -4,6 +4,7 @@
 ┊ 4┊ 4┊import { AuthModule } from './auth';
 ┊ 5┊ 5┊import { UserModule } from './user';
 ┊ 6┊ 6┊import { ChatModule } from './chat';
+┊  ┊ 7┊import { MessageModule } from './message';
 ┊ 7┊ 8┊
 ┊ 8┊ 9┊export interface IAppModuleConfig {
 ┊ 9┊10┊  connection: Connection,
```
```diff
@@ -19,6 +20,7 @@
 ┊19┊20┊    }),
 ┊20┊21┊    UserModule,
 ┊21┊22┊    ChatModule,
+┊  ┊23┊    MessageModule,
 ┊22┊24┊  ],
 ┊23┊25┊  configRequired: true,
 ┊24┊26┊});
```

##### Added modules&#x2F;message&#x2F;index.ts
```diff
@@ -0,0 +1,22 @@
+┊  ┊ 1┊import { GraphQLModule } from '@graphql-modules/core';
+┊  ┊ 2┊import { loadResolversFiles, loadSchemaFiles } from '@graphql-modules/sonar';
+┊  ┊ 3┊import { UserModule } from '../user';
+┊  ┊ 4┊import { ChatModule } from '../chat';
+┊  ┊ 5┊import { MessageProvider } from './providers/message.provider';
+┊  ┊ 6┊import { AuthModule } from '../auth';
+┊  ┊ 7┊import { ProviderScope } from '@graphql-modules/di';
+┊  ┊ 8┊
+┊  ┊ 9┊export const MessageModule = new GraphQLModule({
+┊  ┊10┊  name: "Message",
+┊  ┊11┊  imports: [
+┊  ┊12┊    AuthModule,
+┊  ┊13┊    UserModule,
+┊  ┊14┊    ChatModule,
+┊  ┊15┊  ],
+┊  ┊16┊  providers: [
+┊  ┊17┊    MessageProvider,
+┊  ┊18┊  ],
+┊  ┊19┊  defaultProviderScope: ProviderScope.Session,
+┊  ┊20┊  typeDefs: loadSchemaFiles(__dirname + '/schema/'),
+┊  ┊21┊  resolvers: loadResolversFiles(__dirname + '/resolvers/'),
+┊  ┊22┊});
```

##### Added modules&#x2F;message&#x2F;providers&#x2F;message.provider.ts
```diff
@@ -0,0 +1,347 @@
+┊   ┊  1┊import { Injectable } from '@graphql-modules/di'
+┊   ┊  2┊import { PubSub } from 'apollo-server-express'
+┊   ┊  3┊import { Connection } from 'typeorm'
+┊   ┊  4┊import { User } from '../../../entity/User';
+┊   ┊  5┊import { Chat } from '../../../entity/Chat';
+┊   ┊  6┊import { ChatProvider } from '../../chat/providers/chat.provider';
+┊   ┊  7┊import { Message } from '../../../entity/Message';
+┊   ┊  8┊import { MessageType } from '../../../db';
+┊   ┊  9┊import { AuthProvider } from '../../auth/providers/auth.provider';
+┊   ┊ 10┊import { UserProvider } from '../../user/providers/user.provider';
+┊   ┊ 11┊
+┊   ┊ 12┊@Injectable()
+┊   ┊ 13┊export class MessageProvider {
+┊   ┊ 14┊  constructor(
+┊   ┊ 15┊    private pubsub: PubSub,
+┊   ┊ 16┊    private connection: Connection,
+┊   ┊ 17┊    private chatProvider: ChatProvider,
+┊   ┊ 18┊    private authProvider: AuthProvider,
+┊   ┊ 19┊    private userProvider: UserProvider,
+┊   ┊ 20┊  ) { }
+┊   ┊ 21┊
+┊   ┊ 22┊  repository = this.connection.getRepository(Message);
+┊   ┊ 23┊  currentUser = this.authProvider.currentUser;
+┊   ┊ 24┊
+┊   ┊ 25┊  createQueryBuilder() {
+┊   ┊ 26┊    return this.connection.createQueryBuilder(Message, 'message');
+┊   ┊ 27┊  }
+┊   ┊ 28┊
+┊   ┊ 29┊  async addMessage(chatId: string, content: string) {
+┊   ┊ 30┊    if (content === null || content === '') {
+┊   ┊ 31┊      throw new Error(`Cannot add empty or null messages.`);
+┊   ┊ 32┊    }
+┊   ┊ 33┊
+┊   ┊ 34┊    let chat = await this.chatProvider
+┊   ┊ 35┊      .createQueryBuilder()
+┊   ┊ 36┊      .whereInIds(chatId)
+┊   ┊ 37┊      .innerJoinAndSelect('chat.allTimeMembers', 'allTimeMembers')
+┊   ┊ 38┊      .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊ 39┊      .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊ 40┊      .getOne();
+┊   ┊ 41┊
+┊   ┊ 42┊    if (!chat) {
+┊   ┊ 43┊      throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊ 44┊    }
+┊   ┊ 45┊
+┊   ┊ 46┊    let holders: User[];
+┊   ┊ 47┊
+┊   ┊ 48┊    if (!chat.name) {
+┊   ┊ 49┊      // Chat
+┊   ┊ 50┊      if (!chat.listingMembers.map(user => user.id).includes(this.currentUser.id)) {
+┊   ┊ 51┊        throw new Error(`The chat ${chatId} must be listed for the current user in order to add a message.`);
+┊   ┊ 52┊      }
+┊   ┊ 53┊
+┊   ┊ 54┊      // Receiver's user
+┊   ┊ 55┊      const receiver = chat.allTimeMembers.find(user => user.id !== this.currentUser.id);
+┊   ┊ 56┊
+┊   ┊ 57┊      if (!receiver) {
+┊   ┊ 58┊        throw new Error(`Cannot find receiver's user.`);
+┊   ┊ 59┊      }
+┊   ┊ 60┊
+┊   ┊ 61┊      if (!chat.listingMembers.find(listingMember => listingMember.id === receiver.id)) {
+┊   ┊ 62┊        // Chat is not listed for the receiver user. Add him to the listingIds
+┊   ┊ 63┊        chat.listingMembers.push(receiver);
+┊   ┊ 64┊
+┊   ┊ 65┊        await this.chatProvider.repository.save(chat);
+┊   ┊ 66┊
+┊   ┊ 67┊        this.pubsub.publish('chatAdded', {
+┊   ┊ 68┊          creatorId: this.currentUser.id,
+┊   ┊ 69┊          chatAdded: chat,
+┊   ┊ 70┊        });
+┊   ┊ 71┊      }
+┊   ┊ 72┊
+┊   ┊ 73┊      holders = chat.listingMembers;
+┊   ┊ 74┊    } else {
+┊   ┊ 75┊      // Group
+┊   ┊ 76┊      if (!chat.actualGroupMembers || !chat.actualGroupMembers.find(actualGroupMember => actualGroupMember.id === this.currentUser.id)) {
+┊   ┊ 77┊        throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
+┊   ┊ 78┊      }
+┊   ┊ 79┊
+┊   ┊ 80┊      holders = chat.actualGroupMembers;
+┊   ┊ 81┊    }
+┊   ┊ 82┊
+┊   ┊ 83┊    const message = await this.repository.save(new Message({
+┊   ┊ 84┊      chat,
+┊   ┊ 85┊      sender: this.currentUser,
+┊   ┊ 86┊      content,
+┊   ┊ 87┊      type: MessageType.TEXT,
+┊   ┊ 88┊      holders,
+┊   ┊ 89┊    }));
+┊   ┊ 90┊
+┊   ┊ 91┊    this.pubsub.publish('messageAdded', {
+┊   ┊ 92┊      messageAdded: message,
+┊   ┊ 93┊    });
+┊   ┊ 94┊
+┊   ┊ 95┊    return message || null;
+┊   ┊ 96┊  }
+┊   ┊ 97┊
+┊   ┊ 98┊  async _removeMessages(
+┊   ┊ 99┊
+┊   ┊100┊    chatId: string,
+┊   ┊101┊    {
+┊   ┊102┊      messageIds,
+┊   ┊103┊      all,
+┊   ┊104┊    }: {
+┊   ┊105┊      messageIds?: string[]
+┊   ┊106┊      all?: boolean
+┊   ┊107┊    } = {},
+┊   ┊108┊  ) {
+┊   ┊109┊    const chat = await this.chatProvider
+┊   ┊110┊      .createQueryBuilder()
+┊   ┊111┊      .whereInIds(chatId)
+┊   ┊112┊      .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊113┊      .innerJoinAndSelect('chat.messages', 'messages')
+┊   ┊114┊      .innerJoinAndSelect('messages.holders', 'holders')
+┊   ┊115┊      .getOne();
+┊   ┊116┊
+┊   ┊117┊    if (!chat) {
+┊   ┊118┊      throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊119┊    }
+┊   ┊120┊
+┊   ┊121┊    if (!chat.listingMembers.find(user => user.id === this.currentUser.id)) {
+┊   ┊122┊      throw new Error(`The chat/group ${chatId} is not listed for the current user so there is nothing to delete.`);
+┊   ┊123┊    }
+┊   ┊124┊
+┊   ┊125┊    if (all && messageIds) {
+┊   ┊126┊      throw new Error(`Cannot specify both 'all' and 'messageIds'.`)
+┊   ┊127┊    }
+┊   ┊128┊
+┊   ┊129┊    if (!all && !(messageIds && messageIds.length)) {
+┊   ┊130┊      throw new Error(`'all' and 'messageIds' cannot be both null`)
+┊   ┊131┊    }
+┊   ┊132┊
+┊   ┊133┊    let deletedIds: string[] = [];
+┊   ┊134┊    let removedMessages: Message[] = [];
+┊   ┊135┊    // Instead of chaining map and filter we can loop once using reduce
+┊   ┊136┊    chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊137┊      const filtered = await filtered$;
+┊   ┊138┊
+┊   ┊139┊      if (all || messageIds!.map(Number).includes(message.id)) {
+┊   ┊140┊        deletedIds.push(String(message.id));
+┊   ┊141┊        // Remove the current user from the message holders
+┊   ┊142┊        message.holders = message.holders.filter(user => user.id !== this.currentUser.id);
+┊   ┊143┊
+┊   ┊144┊      }
+┊   ┊145┊
+┊   ┊146┊      if (message.holders.length !== 0) {
+┊   ┊147┊        // Remove the current user from the message holders
+┊   ┊148┊        await this.repository.save(message);
+┊   ┊149┊        filtered.push(message);
+┊   ┊150┊      } else {
+┊   ┊151┊        // Message is flagged for removal
+┊   ┊152┊        removedMessages.push(message);
+┊   ┊153┊      }
+┊   ┊154┊
+┊   ┊155┊      return filtered;
+┊   ┊156┊    }, Promise.resolve([]));
+┊   ┊157┊
+┊   ┊158┊    return { deletedIds, removedMessages };
+┊   ┊159┊  }
+┊   ┊160┊
+┊   ┊161┊  async removeMessages(
+┊   ┊162┊
+┊   ┊163┊    chatId: string,
+┊   ┊164┊    {
+┊   ┊165┊      messageIds,
+┊   ┊166┊      all,
+┊   ┊167┊    }: {
+┊   ┊168┊      messageIds?: string[]
+┊   ┊169┊      all?: boolean
+┊   ┊170┊    } = {},
+┊   ┊171┊  ) {
+┊   ┊172┊    const { deletedIds, removedMessages } = await this._removeMessages(chatId, { messageIds, all });
+┊   ┊173┊
+┊   ┊174┊    for (let message of removedMessages) {
+┊   ┊175┊      await this.repository.remove(message);
+┊   ┊176┊    }
+┊   ┊177┊
+┊   ┊178┊    return deletedIds;
+┊   ┊179┊  }
+┊   ┊180┊
+┊   ┊181┊  async _removeChatGetMessages(chatId: string) {
+┊   ┊182┊    let messages = await this.createQueryBuilder()
+┊   ┊183┊      .innerJoin('message.chat', 'chat', 'chat.id = :chatId', { chatId })
+┊   ┊184┊      .leftJoinAndSelect('message.holders', 'holders')
+┊   ┊185┊      .getMany();
+┊   ┊186┊
+┊   ┊187┊    messages = messages.map(message => ({
+┊   ┊188┊      ...message,
+┊   ┊189┊      holders: message.holders.filter(user => user.id !== this.currentUser.id),
+┊   ┊190┊    }));
+┊   ┊191┊
+┊   ┊192┊    return messages;
+┊   ┊193┊  }
+┊   ┊194┊
+┊   ┊195┊  async removeChat(chatId: string, messages?: Message[]) {
+┊   ┊196┊    if (!messages) {
+┊   ┊197┊      messages = await this._removeChatGetMessages(chatId);
+┊   ┊198┊    }
+┊   ┊199┊
+┊   ┊200┊    for (let message of messages) {
+┊   ┊201┊      message.holders = message.holders.filter(user => user.id !== this.currentUser.id);
+┊   ┊202┊
+┊   ┊203┊      if (message.holders.length !== 0) {
+┊   ┊204┊        // Remove the current user from the message holders
+┊   ┊205┊        await this.repository.save(message);
+┊   ┊206┊      } else {
+┊   ┊207┊        // Simply remove the message
+┊   ┊208┊        await this.repository.remove(message);
+┊   ┊209┊      }
+┊   ┊210┊    }
+┊   ┊211┊
+┊   ┊212┊    return await this.chatProvider.removeChat(chatId);
+┊   ┊213┊  }
+┊   ┊214┊
+┊   ┊215┊  async getMessageSender(message: Message) {
+┊   ┊216┊    const sender = await this.userProvider
+┊   ┊217┊      .createQueryBuilder()
+┊   ┊218┊      .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {
+┊   ┊219┊        messageId: message.id,
+┊   ┊220┊      })
+┊   ┊221┊      .getOne();
+┊   ┊222┊
+┊   ┊223┊    if (!sender) {
+┊   ┊224┊      throw new Error(`Message must have a sender.`);
+┊   ┊225┊    }
+┊   ┊226┊
+┊   ┊227┊    return sender;
+┊   ┊228┊  }
+┊   ┊229┊
+┊   ┊230┊  async getMessageOwnership(message: Message) {
+┊   ┊231┊
+┊   ┊232┊    return !!(await this.userProvider
+┊   ┊233┊      .createQueryBuilder()
+┊   ┊234┊      .whereInIds(this.currentUser.id)
+┊   ┊235┊      .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {
+┊   ┊236┊        messageId: message.id,
+┊   ┊237┊      })
+┊   ┊238┊      .getCount());
+┊   ┊239┊  }
+┊   ┊240┊
+┊   ┊241┊  async getMessageHolders(message: Message) {
+┊   ┊242┊    return await this.userProvider
+┊   ┊243┊      .createQueryBuilder()
+┊   ┊244┊      .innerJoin('user.holderMessages', 'holderMessages', 'holderMessages.id = :messageId', {
+┊   ┊245┊        messageId: message.id,
+┊   ┊246┊      })
+┊   ┊247┊      .getMany();
+┊   ┊248┊  }
+┊   ┊249┊
+┊   ┊250┊  async getMessageChat(message: Message) {
+┊   ┊251┊    const chat = await this.chatProvider
+┊   ┊252┊      .createQueryBuilder()
+┊   ┊253┊      .innerJoin('chat.messages', 'messages', 'messages.id = :messageId', {
+┊   ┊254┊        messageId: message.id
+┊   ┊255┊      })
+┊   ┊256┊      .getOne();
+┊   ┊257┊
+┊   ┊258┊    if (!chat) {
+┊   ┊259┊      throw new Error(`Message must have a chat.`);
+┊   ┊260┊    }
+┊   ┊261┊
+┊   ┊262┊    return chat;
+┊   ┊263┊  }
+┊   ┊264┊
+┊   ┊265┊  async getChats() {
+┊   ┊266┊    // TODO: make a proper query instead of this mess (see https://github.com/typeorm/typeorm/issues/2132)
+┊   ┊267┊    const chats = await this.chatProvider
+┊   ┊268┊      .createQueryBuilder()
+┊   ┊269┊      .leftJoin('chat.listingMembers', 'listingMembers')
+┊   ┊270┊      .where('listingMembers.id = :id', { id: this.currentUser.id })
+┊   ┊271┊      .getMany();
+┊   ┊272┊
+┊   ┊273┊    for (let chat of chats) {
+┊   ┊274┊      chat.messages = await this.getChatMessages(chat);
+┊   ┊275┊    }
+┊   ┊276┊
+┊   ┊277┊    return chats.sort((chatA, chatB) => {
+┊   ┊278┊      const dateA = chatA.messages.length ? chatA.messages[chatA.messages.length - 1].createdAt : chatA.createdAt;
+┊   ┊279┊      const dateB = chatB.messages.length ? chatB.messages[chatB.messages.length - 1].createdAt : chatB.createdAt;
+┊   ┊280┊      return dateB.valueOf() - dateA.valueOf();
+┊   ┊281┊    });
+┊   ┊282┊  }
+┊   ┊283┊
+┊   ┊284┊  async getChatMessages(chat: Chat, amount?: number) {
+┊   ┊285┊    if (chat.messages) {
+┊   ┊286┊      return amount ? chat.messages.slice(-amount) : chat.messages;
+┊   ┊287┊    }
+┊   ┊288┊
+┊   ┊289┊    let query = this.createQueryBuilder()
+┊   ┊290┊      .innerJoin('message.chat', 'chat', 'chat.id = :chatId', { chatId: chat.id })
+┊   ┊291┊      .innerJoin('message.holders', 'holders', 'holders.id = :userId', {
+┊   ┊292┊        userId: this.currentUser.id,
+┊   ┊293┊      })
+┊   ┊294┊      .orderBy({ 'message.createdAt': { order: 'DESC', nulls: 'NULLS LAST' } });
+┊   ┊295┊
+┊   ┊296┊    if (amount) {
+┊   ┊297┊      query = query.take(amount);
+┊   ┊298┊    }
+┊   ┊299┊
+┊   ┊300┊    return (await query.getMany()).reverse();
+┊   ┊301┊  }
+┊   ┊302┊
+┊   ┊303┊  async getChatLastMessage(chat: Chat) {
+┊   ┊304┊    const messages = await this.getChatMessages(chat, 1);
+┊   ┊305┊
+┊   ┊306┊    return messages && messages.length ? messages[0] : null;
+┊   ┊307┊  }
+┊   ┊308┊
+┊   ┊309┊  async getChatUpdatedAt(chat: Chat) {
+┊   ┊310┊    const latestMessage = await this.getChatLastMessage(chat);
+┊   ┊311┊
+┊   ┊312┊    return latestMessage ? latestMessage.createdAt : null;
+┊   ┊313┊  }
+┊   ┊314┊
+┊   ┊315┊  async filterMessageAdded(messageAdded: Message) {
+┊   ┊316┊
+┊   ┊317┊    let relevantUsers: User[];
+┊   ┊318┊
+┊   ┊319┊    if (!messageAdded.chat.name) {
+┊   ┊320┊      // Chat
+┊   ┊321┊      relevantUsers = (await this.userProvider
+┊   ┊322┊        .createQueryBuilder()
+┊   ┊323┊        .innerJoin(
+┊   ┊324┊          'user.listingMemberChats',
+┊   ┊325┊          'listingMemberChats',
+┊   ┊326┊          'listingMemberChats.id = :chatId',
+┊   ┊327┊          { chatId: messageAdded.chat.id },
+┊   ┊328┊        )
+┊   ┊329┊        .getMany())
+┊   ┊330┊        .filter(user => user.id != messageAdded.sender.id);
+┊   ┊331┊    } else {
+┊   ┊332┊      // Group
+┊   ┊333┊      relevantUsers = (await this.userProvider
+┊   ┊334┊        .createQueryBuilder()
+┊   ┊335┊        .innerJoin(
+┊   ┊336┊          'user.actualGroupMemberChats',
+┊   ┊337┊          'actualGroupMemberChats',
+┊   ┊338┊          'actualGroupMemberChats.id = :chatId',
+┊   ┊339┊          { chatId: messageAdded.chat.id },
+┊   ┊340┊        )
+┊   ┊341┊        .getMany())
+┊   ┊342┊        .filter(user => user.id != messageAdded.sender.id);
+┊   ┊343┊    }
+┊   ┊344┊
+┊   ┊345┊    return relevantUsers.some(user => user.id === this.currentUser.id);
+┊   ┊346┊  }
+┊   ┊347┊}
```

##### Added modules&#x2F;message&#x2F;resolvers&#x2F;resolvers.ts
```diff
@@ -0,0 +1,47 @@
+┊  ┊ 1┊import { PubSub } from 'apollo-server-express';
+┊  ┊ 2┊import { withFilter } from 'apollo-server-express';
+┊  ┊ 3┊import { Message } from '../../../entity/Message';
+┊  ┊ 4┊import { IResolvers } from '../../../types';
+┊  ┊ 5┊import { MessageProvider } from '../providers/message.provider';
+┊  ┊ 6┊import { ModuleContext } from '@graphql-modules/core';
+┊  ┊ 7┊
+┊  ┊ 8┊export default {
+┊  ┊ 9┊  Query: {
+┊  ┊10┊    // The ordering depends on the messages
+┊  ┊11┊    chats: (obj, args, { injector }) => injector.get(MessageProvider).getChats(),
+┊  ┊12┊  },
+┊  ┊13┊  Mutation: {
+┊  ┊14┊    addMessage: async (obj, { chatId, content }, { injector }) =>
+┊  ┊15┊      injector.get(MessageProvider).addMessage(chatId, content),
+┊  ┊16┊    removeMessages: async (obj, { chatId, messageIds, all }, { injector }) =>
+┊  ┊17┊      injector.get(MessageProvider).removeMessages(chatId, {
+┊  ┊18┊        messageIds: messageIds || undefined,
+┊  ┊19┊        all: all || false,
+┊  ┊20┊      }),
+┊  ┊21┊    // We may need to also remove the messages
+┊  ┊22┊    removeChat: async (obj, { chatId }, { injector }) => injector.get(MessageProvider).removeChat(chatId),
+┊  ┊23┊  },
+┊  ┊24┊  Subscription: {
+┊  ┊25┊    messageAdded: {
+┊  ┊26┊      subscribe: withFilter((root, args, { injector }: ModuleContext) => injector.get(PubSub).asyncIterator('messageAdded'),
+┊  ┊27┊        (data: { messageAdded: Message }, variables, { injector }: ModuleContext) => data && injector.get(MessageProvider).filterMessageAdded(data.messageAdded)
+┊  ┊28┊      ),
+┊  ┊29┊    },
+┊  ┊30┊  },
+┊  ┊31┊  Chat: {
+┊  ┊32┊    messages: async (chat, { amount }, { injector }) =>
+┊  ┊33┊      injector.get(MessageProvider).getChatMessages(chat, amount || 0),
+┊  ┊34┊    lastMessage: async (chat, args, { injector }) =>
+┊  ┊35┊      injector.get(MessageProvider).getChatLastMessage(chat),
+┊  ┊36┊    updatedAt: async (chat, args, { injector }) => injector.get(MessageProvider).getChatUpdatedAt(chat),
+┊  ┊37┊  },
+┊  ┊38┊  Message: {
+┊  ┊39┊    sender: async (message, args, { injector }) =>
+┊  ┊40┊      injector.get(MessageProvider).getMessageSender(message),
+┊  ┊41┊    ownership: async (message, args, { injector }) =>
+┊  ┊42┊      injector.get(MessageProvider).getMessageOwnership(message),
+┊  ┊43┊    holders: async (message, args, { injector }) =>
+┊  ┊44┊      injector.get(MessageProvider).getMessageHolders(message),
+┊  ┊45┊    chat: async (message, args, { injector }) => injector.get(MessageProvider).getMessageChat(message),
+┊  ┊46┊  },
+┊  ┊47┊} as IResolvers;
```

##### Added modules&#x2F;message&#x2F;schema&#x2F;typeDefs.graphql
```diff
@@ -0,0 +1,35 @@
+┊  ┊ 1┊type Subscription {
+┊  ┊ 2┊  messageAdded: Message
+┊  ┊ 3┊}
+┊  ┊ 4┊
+┊  ┊ 5┊enum MessageType {
+┊  ┊ 6┊  LOCATION
+┊  ┊ 7┊  TEXT
+┊  ┊ 8┊  PICTURE
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊extend type Chat {
+┊  ┊12┊  messages(amount: Int): [Message]!
+┊  ┊13┊  lastMessage: Message
+┊  ┊14┊  updatedAt: Date!
+┊  ┊15┊}
+┊  ┊16┊
+┊  ┊17┊type Message {
+┊  ┊18┊  id: ID!
+┊  ┊19┊  sender: User!
+┊  ┊20┊  chat: Chat!
+┊  ┊21┊  content: String!
+┊  ┊22┊  createdAt: Date!
+┊  ┊23┊  #FIXME: should return MessageType
+┊  ┊24┊  type: Int!
+┊  ┊25┊  #Whoever still holds a copy of the message. Cannot be null because the message gets deleted otherwise
+┊  ┊26┊  holders: [User!]!
+┊  ┊27┊  #Computed property
+┊  ┊28┊  ownership: Boolean!
+┊  ┊29┊}
+┊  ┊30┊
+┊  ┊31┊type Mutation {
+┊  ┊32┊  addMessage(chatId: ID!, content: String!): Message
+┊  ┊33┊  removeMessages(chatId: ID!, messageIds: [ID!], all: Boolean): [ID]!
+┊  ┊34┊}
+┊  ┊35┊
```

[}]: #

Next is the `Recipient Module`. What does it do? It extends the `Message Module` in order to implement the infamous Whatsapp ticks (single, double and blue).
Right now our client doesn't support this functionality, but soon we will update it in order to make use of it.
Once again notice that the Recipient module extends both the `Chat` and the `Message` types, while also extending several mutations like `removeChat`, `addMessage` and `removeMessages`.

[{]: <helper> (diffStep "8.6" files="^\(?!types.d.ts$\).*" module="server")

#### [Step 8.6: Recipient Module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/562bfb2)

##### Changed modules&#x2F;app.module.ts
```diff
@@ -5,6 +5,7 @@
 ┊ 5┊ 5┊import { UserModule } from './user';
 ┊ 6┊ 6┊import { ChatModule } from './chat';
 ┊ 7┊ 7┊import { MessageModule } from './message';
+┊  ┊ 8┊import { RecipientModule } from './recipient';
 ┊ 8┊ 9┊
 ┊ 9┊10┊export interface IAppModuleConfig {
 ┊10┊11┊  connection: Connection,
```
```diff
@@ -21,6 +22,7 @@
 ┊21┊22┊    UserModule,
 ┊22┊23┊    ChatModule,
 ┊23┊24┊    MessageModule,
+┊  ┊25┊    RecipientModule,
 ┊24┊26┊  ],
 ┊25┊27┊  configRequired: true,
 ┊26┊28┊});
```

##### Added modules&#x2F;recipient&#x2F;index.ts
```diff
@@ -0,0 +1,22 @@
+┊  ┊ 1┊import { GraphQLModule } from '@graphql-modules/core';
+┊  ┊ 2┊import { loadResolversFiles, loadSchemaFiles } from '@graphql-modules/sonar';
+┊  ┊ 3┊import { UserModule } from '../user';
+┊  ┊ 4┊import { MessageModule } from '../message';
+┊  ┊ 5┊import { ChatModule } from '../chat';
+┊  ┊ 6┊import { RecipientProvider } from './providers/recipient.provider';
+┊  ┊ 7┊import { AuthModule } from '../auth';
+┊  ┊ 8┊
+┊  ┊ 9┊export const RecipientModule = new GraphQLModule({
+┊  ┊10┊  name: "Recipient",
+┊  ┊11┊  imports: [
+┊  ┊12┊    AuthModule,
+┊  ┊13┊    UserModule,
+┊  ┊14┊    ChatModule,
+┊  ┊15┊    MessageModule,
+┊  ┊16┊  ],
+┊  ┊17┊  providers: [
+┊  ┊18┊    RecipientProvider,
+┊  ┊19┊  ],
+┊  ┊20┊  typeDefs: loadSchemaFiles(__dirname + '/schema/'),
+┊  ┊21┊  resolvers: loadResolversFiles(__dirname + '/resolvers/'),
+┊  ┊22┊});
```

##### Added modules&#x2F;recipient&#x2F;providers&#x2F;recipient.provider.ts
```diff
@@ -0,0 +1,111 @@
+┊   ┊  1┊import { Injectable, ProviderScope } from '@graphql-modules/di'
+┊   ┊  2┊import { Connection } from 'typeorm'
+┊   ┊  3┊import { MessageProvider } from '../../message/providers/message.provider';
+┊   ┊  4┊import { Chat } from '../../../entity/Chat';
+┊   ┊  5┊import { Message } from '../../../entity/Message';
+┊   ┊  6┊import { Recipient } from '../../../entity/Recipient';
+┊   ┊  7┊import { AuthProvider } from '../../auth/providers/auth.provider';
+┊   ┊  8┊
+┊   ┊  9┊@Injectable({
+┊   ┊ 10┊  scope: ProviderScope.Session,
+┊   ┊ 11┊})
+┊   ┊ 12┊export class RecipientProvider {
+┊   ┊ 13┊  constructor(
+┊   ┊ 14┊    private authProvider: AuthProvider,
+┊   ┊ 15┊    private connection: Connection,
+┊   ┊ 16┊    private messageProvider: MessageProvider,
+┊   ┊ 17┊  ) { }
+┊   ┊ 18┊
+┊   ┊ 19┊  public repository = this.connection.getRepository(Recipient);
+┊   ┊ 20┊  public currentUser = this.authProvider.currentUser;
+┊   ┊ 21┊
+┊   ┊ 22┊  createQueryBuilder() {
+┊   ┊ 23┊    return this.connection.createQueryBuilder(Recipient, 'recipient');
+┊   ┊ 24┊  }
+┊   ┊ 25┊
+┊   ┊ 26┊  getChatUnreadMessagesCount(chat: Chat) {
+┊   ┊ 27┊    return this.messageProvider
+┊   ┊ 28┊      .createQueryBuilder()
+┊   ┊ 29┊      .innerJoin('message.chat', 'chat', 'chat.id = :chatId', { chatId: chat.id })
+┊   ┊ 30┊      .innerJoin('message.recipients', 'recipients', 'recipients.user.id = :userId AND recipients.readAt IS NULL', {
+┊   ┊ 31┊        userId: this.currentUser.id
+┊   ┊ 32┊      })
+┊   ┊ 33┊      .getCount();
+┊   ┊ 34┊  }
+┊   ┊ 35┊
+┊   ┊ 36┊  getMessageRecipients(message: Message) {
+┊   ┊ 37┊    return this.createQueryBuilder()
+┊   ┊ 38┊      .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {
+┊   ┊ 39┊        messageId: message.id,
+┊   ┊ 40┊      })
+┊   ┊ 41┊      .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊ 42┊      .getMany();
+┊   ┊ 43┊  }
+┊   ┊ 44┊
+┊   ┊ 45┊  async getRecipientChat(recipient: Recipient) {
+┊   ┊ 46┊    if (recipient.message.chat) {
+┊   ┊ 47┊      return recipient.message.chat;
+┊   ┊ 48┊    }
+┊   ┊ 49┊
+┊   ┊ 50┊    return this.messageProvider.getMessageChat(recipient.message);
+┊   ┊ 51┊  }
+┊   ┊ 52┊
+┊   ┊ 53┊  async removeChat(chatId: string) {
+┊   ┊ 54┊    const messages = await this.messageProvider._removeChatGetMessages(chatId);
+┊   ┊ 55┊
+┊   ┊ 56┊    for (let message of messages) {
+┊   ┊ 57┊      if (message.holders.length === 0) {
+┊   ┊ 58┊        const recipients = await this.createQueryBuilder()
+┊   ┊ 59┊          .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', { messageId: message.id })
+┊   ┊ 60┊          .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊ 61┊          .getMany();
+┊   ┊ 62┊
+┊   ┊ 63┊        for (let recipient of recipients) {
+┊   ┊ 64┊          await this.repository.remove(recipient);
+┊   ┊ 65┊        }
+┊   ┊ 66┊      }
+┊   ┊ 67┊    }
+┊   ┊ 68┊
+┊   ┊ 69┊    return await this.messageProvider.removeChat(chatId, messages);
+┊   ┊ 70┊  }
+┊   ┊ 71┊
+┊   ┊ 72┊  async addMessage(chatId: string, content: string) {
+┊   ┊ 73┊    const message = await this.messageProvider.addMessage(chatId, content);
+┊   ┊ 74┊
+┊   ┊ 75┊    for (let user of message.holders) {
+┊   ┊ 76┊      if (user.id !== this.currentUser.id) {
+┊   ┊ 77┊        await this.repository.save(new Recipient({ user, message }));
+┊   ┊ 78┊      }
+┊   ┊ 79┊    }
+┊   ┊ 80┊
+┊   ┊ 81┊    return message;
+┊   ┊ 82┊  }
+┊   ┊ 83┊
+┊   ┊ 84┊  async removeMessages(
+┊   ┊ 85┊    chatId: string,
+┊   ┊ 86┊    {
+┊   ┊ 87┊      messageIds,
+┊   ┊ 88┊      all,
+┊   ┊ 89┊    }: {
+┊   ┊ 90┊      messageIds?: string[]
+┊   ┊ 91┊      all?: boolean
+┊   ┊ 92┊    } = {},
+┊   ┊ 93┊  ) {
+┊   ┊ 94┊    const { deletedIds, removedMessages } = await this.messageProvider._removeMessages(chatId, { messageIds, all });
+┊   ┊ 95┊
+┊   ┊ 96┊    for (let message of removedMessages) {
+┊   ┊ 97┊      const recipients = await this.createQueryBuilder()
+┊   ┊ 98┊        .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', { messageId: message.id })
+┊   ┊ 99┊        .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊100┊        .getMany();
+┊   ┊101┊
+┊   ┊102┊      for (let recipient of recipients) {
+┊   ┊103┊        await this.repository.remove(recipient);
+┊   ┊104┊      }
+┊   ┊105┊
+┊   ┊106┊      await this.messageProvider.repository.remove(message);
+┊   ┊107┊    }
+┊   ┊108┊
+┊   ┊109┊    return deletedIds;
+┊   ┊110┊  }
+┊   ┊111┊}
```

##### Added modules&#x2F;recipient&#x2F;resolvers&#x2F;resolvers.ts
```diff
@@ -0,0 +1,29 @@
+┊  ┊ 1┊import { IResolvers } from '../../../types';
+┊  ┊ 2┊import { RecipientProvider } from '../providers/recipient.provider';
+┊  ┊ 3┊
+┊  ┊ 4┊export default {
+┊  ┊ 5┊  Mutation: {
+┊  ┊ 6┊    // TODO: implement me
+┊  ┊ 7┊    markAsReceived: async (obj, { chatId }) => false,
+┊  ┊ 8┊    // TODO: implement me
+┊  ┊ 9┊    markAsRead: async (obj, { chatId }) => false,
+┊  ┊10┊    // We may also need to remove the recipients
+┊  ┊11┊    removeChat: async (obj, { chatId }, { injector }) => injector.get(RecipientProvider).removeChat(chatId),
+┊  ┊12┊    // We also need to create the recipients
+┊  ┊13┊    addMessage: async (obj, { chatId, content }, { injector }) => injector.get(RecipientProvider).addMessage(chatId, content),
+┊  ┊14┊    // We may also need to remove the recipients
+┊  ┊15┊    removeMessages: async (obj, { chatId, messageIds, all }, { injector }) => injector.get(RecipientProvider).removeMessages(chatId, {
+┊  ┊16┊      messageIds: messageIds || undefined,
+┊  ┊17┊      all: all || false,
+┊  ┊18┊    }),
+┊  ┊19┊  },
+┊  ┊20┊  Chat: {
+┊  ┊21┊    unreadMessages: async (chat, args, { injector }) => injector.get(RecipientProvider).getChatUnreadMessagesCount(chat),
+┊  ┊22┊  },
+┊  ┊23┊  Message: {
+┊  ┊24┊    recipients: async (message, args, { injector }) => injector.get(RecipientProvider).getMessageRecipients(message),
+┊  ┊25┊  },
+┊  ┊26┊  Recipient: {
+┊  ┊27┊    chat: async (recipient, args, { injector }) => injector.get(RecipientProvider).getRecipientChat(recipient),
+┊  ┊28┊  },
+┊  ┊29┊} as IResolvers;
```

##### Added modules&#x2F;recipient&#x2F;schema&#x2F;typeDefs.graphql
```diff
@@ -0,0 +1,22 @@
+┊  ┊ 1┊extend type Chat {
+┊  ┊ 2┊  #Computed property
+┊  ┊ 3┊  unreadMessages: Int!
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊extend type Message {
+┊  ┊ 7┊  #Whoever received the message
+┊  ┊ 8┊  recipients: [Recipient!]!
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊type Recipient {
+┊  ┊12┊  user: User!
+┊  ┊13┊  message: Message!
+┊  ┊14┊  chat: Chat!
+┊  ┊15┊  receivedAt: Date
+┊  ┊16┊  readAt: Date
+┊  ┊17┊}
+┊  ┊18┊
+┊  ┊19┊type Mutation {
+┊  ┊20┊  markAsReceived(chatId: ID!): Boolean
+┊  ┊21┊  markAsRead(chatId: ID!): Boolean
+┊  ┊22┊}
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step12.md) |
|:----------------------|

[}]: #
