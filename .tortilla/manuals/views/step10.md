# Step 10: Authentication

[//]: # (head-end)


## Authentication on server

Authentication is an hot topic in the GraphQL world and there are some projects which aim at authenticating through GraphQL.
Since often you will be required to use a specific auth framework (because of a feature you need or because of an existing authorization infrastructure) I will show you how to use a classic REST API framework within your GraphQL application.
This approach is completely fine and in line with the official GraphQL best practices.
We will use `Passport` for the authentication and `BasicAuth` as the auth mechanism:

    yarn add bcrypt-nodejs passport passport-http
    yarn add -D @types/bcrypt-nodejs @types/passport @types/passport-http

`BasicAuth` basically involves to send username e password in an Authorization Header together with each request and is fully supported by any browser (meaning that we will be able to use `Graphiql` simply by proving username and password in the login window provided by the browser itself).
It's the most simple auth mechanism but it's completely fine for our needs. Later we could decide to use something more complicated like `JWT`, but it's outside of the scope of this tutorial.

[{]: <helper> (diffStep "5.1" files="index.ts" module="server")

#### Step 5.1: Authentication

##### Changed index.ts
```diff
@@ -8,6 +8,47 @@
 ┊ 8┊ 8┊import cloudinary from 'cloudinary';
 ┊ 9┊ 9┊import multer from 'multer';
 ┊10┊10┊import tmp from 'tmp';
+┊  ┊11┊import passport from "passport";
+┊  ┊12┊import basicStrategy from 'passport-http';
+┊  ┊13┊import bcrypt from 'bcrypt-nodejs';
+┊  ┊14┊import { db, UserDb } from "./db";
+┊  ┊15┊
+┊  ┊16┊let users = db.users;
+┊  ┊17┊
+┊  ┊18┊function generateHash(password: string) {
+┊  ┊19┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
+┊  ┊20┊}
+┊  ┊21┊
+┊  ┊22┊function validPassword(password: string, localPassword: string) {
+┊  ┊23┊  return bcrypt.compareSync(password, localPassword);
+┊  ┊24┊}
+┊  ┊25┊
+┊  ┊26┊passport.use('basic-signin', new basicStrategy.BasicStrategy(
+┊  ┊27┊  function (username: string, password: string, done: any) {
+┊  ┊28┊    const user = users.find(user => user.username == username);
+┊  ┊29┊    if (user && validPassword(password, user.password)) {
+┊  ┊30┊      return done(null, user);
+┊  ┊31┊    }
+┊  ┊32┊    return done(null, false);
+┊  ┊33┊  }
+┊  ┊34┊));
+┊  ┊35┊
+┊  ┊36┊passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
+┊  ┊37┊  function (req: any, username: any, password: any, done: any) {
+┊  ┊38┊    const userExists = !!users.find(user => user.username === username);
+┊  ┊39┊    if (!userExists && password && req.body.name) {
+┊  ┊40┊      const user: UserDb = {
+┊  ┊41┊        id: (users.length && users[users.length - 1].id + 1) || 1,
+┊  ┊42┊        username,
+┊  ┊43┊        password: generateHash(password),
+┊  ┊44┊        name: req.body.name,
+┊  ┊45┊      };
+┊  ┊46┊      users.push(user);
+┊  ┊47┊      return done(null, user);
+┊  ┊48┊    }
+┊  ┊49┊    return done(null, false);
+┊  ┊50┊  }
+┊  ┊51┊));
 ┊11┊52┊
 ┊12┊53┊const PORT = 4000;
 ┊13┊54┊const CLOUDINARY_URL = process.env.CLOUDINARY_URL || '';
```
```diff
@@ -16,6 +57,19 @@
 ┊16┊57┊
 ┊17┊58┊app.use(cors());
 ┊18┊59┊app.use(bodyParser.json());
+┊  ┊60┊app.use(passport.initialize());
+┊  ┊61┊
+┊  ┊62┊app.post('/signup',
+┊  ┊63┊  passport.authenticate('basic-signup', {session: false}),
+┊  ┊64┊  function (req: any, res) {
+┊  ┊65┊    res.json(req.user);
+┊  ┊66┊  });
+┊  ┊67┊
+┊  ┊68┊app.use(passport.authenticate('basic-signin', {session: false}));
+┊  ┊69┊
+┊  ┊70┊app.post('/signin', function (req: any, res) {
+┊  ┊71┊  res.json(req.user);
+┊  ┊72┊});
 ┊19┊73┊
 ┊20┊74┊const match = CLOUDINARY_URL.match(/cloudinary:\/\/(\d+):(\w+)@(\.+)/);
 ┊21┊75┊
```
```diff
@@ -45,7 +99,12 @@
 ┊ 45┊ 99┊});
 ┊ 46┊100┊
 ┊ 47┊101┊const apollo = new ApolloServer({
-┊ 48┊   ┊  schema
+┊   ┊102┊  schema,
+┊   ┊103┊  context(received: any) {
+┊   ┊104┊    return {
+┊   ┊105┊      currentUser: received.req!['user'],
+┊   ┊106┊    }
+┊   ┊107┊  },
 ┊ 49┊108┊});
 ┊ 50┊109┊
 ┊ 51┊110┊apollo.applyMiddleware({
```

[}]: #

We are going to store hashes instead of plain passwords, that's why we're using `bcrypt-nodejs`.
With `passport.use('basic-signin')` and `passport.use('basic-signup')` we define how the auth framework deals with our database (well, our JSON file for the moment).
`app.post('/signup')` is the endpoint for creating new accounts, so we left it out of the authentication middleware (`app.use(passport.authenticate('basic-signin')`).
What's of particular interest is that we're passing the user object to the GraphQL context.

[{]: <helper> (diffStep "5.1" files="schema/types.ts" module="server")

#### Step 5.1: Authentication

##### Added schema&#x2F;types.ts
```diff
@@ -0,0 +1,5 @@
+┊ ┊1┊import { UserDb } from "../db";
+┊ ┊2┊
+┊ ┊3┊export interface AppContext {
+┊ ┊4┊  currentUser: UserDb;
+┊ ┊5┊}
```

[}]: #
[{]: <helper> (diffStep "5.1" files="codegen.yml" module="server")

#### Step 5.1: Authentication

##### Changed codegen.yml
```diff
@@ -7,6 +7,7 @@
 ┊ 7┊ 7┊  ./types.d.ts:
 ┊ 8┊ 8┊    config:
 ┊ 9┊ 9┊      optionalType: undefined | null
+┊  ┊10┊      contextType: ./schema/types#AppContext
 ┊10┊11┊      mappers:
 ┊11┊12┊        Chat: ./db#ChatDb
 ┊12┊13┊        Message: ./db#MessageDb
```

[}]: #

As you can see, we defined the `contextType` and `AppContext`. Let me explain it.

```yaml
# contextType: ./relative/path/to/module#Interface
contextType: ./schema/types#AppContext
```

It means we want to use `AppContext` interface from `./schema/types.ts` module as a Context of every resolver. This is very helpful to not repeat the same interface over and over again, in each resolver function. The path should be relative to the output file.

[{]: <helper> (diffStep "5.1" files="schema/resolvers.ts" module="server")

#### Step 5.1: Authentication

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -5,24 +5,23 @@
 ┊ 5┊ 5┊
 ┊ 6┊ 6┊let users = db.users;
 ┊ 7┊ 7┊let chats = db.chats;
-┊ 8┊  ┊const currentUser = users[0];
 ┊ 9┊ 8┊
 ┊10┊ 9┊export const resolvers: IResolvers = {
 ┊11┊10┊  Date: GraphQLDateTime,
 ┊12┊11┊  Query: {
-┊13┊  ┊    me: () => currentUser,
-┊14┊  ┊    users: () => users.filter(user => user.id !== currentUser.id),
-┊15┊  ┊    chats: () => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
-┊16┊  ┊    chat: (obj, {chatId}) => chats.find(chat => chat.id === Number(chatId)),
+┊  ┊12┊    me: (obj, args, {currentUser}) => currentUser,
+┊  ┊13┊    users: (obj, args, {currentUser}) => users.filter(user => user.id !== currentUser.id),
+┊  ┊14┊    chats: (obj, args, {currentUser}) => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
+┊  ┊15┊    chat: (obj, {chatId}) => chats.find(chat => chat.id === Number(chatId)) || null,
 ┊17┊16┊  },
 ┊18┊17┊  Mutation: {
-┊19┊  ┊    updateUser: (obj, {name, picture}) => {
+┊  ┊18┊    updateUser: (obj, {name, picture}, {currentUser}) => {
 ┊20┊19┊      currentUser.name = name || currentUser.name;
 ┊21┊20┊      currentUser.picture = picture || currentUser.picture;
 ┊22┊21┊
 ┊23┊22┊      return currentUser;
 ┊24┊23┊    },
-┊25┊  ┊    addChat: (obj, {userId}) => {
+┊  ┊24┊    addChat: (obj, {userId}, {currentUser}) => {
 ┊26┊25┊      if (!users.find(user => user.id === Number(userId))) {
 ┊27┊26┊        throw new Error(`User ${userId} doesn't exist.`);
 ┊28┊27┊      }
```
```diff
@@ -59,7 +58,7 @@
 ┊59┊58┊        return chat;
 ┊60┊59┊      }
 ┊61┊60┊    },
-┊62┊  ┊    addGroup: (obj, {userIds, groupName, groupPicture}) => {
+┊  ┊61┊    addGroup: (obj, {userIds, groupName, groupPicture}, {currentUser}) => {
 ┊63┊62┊      userIds.forEach(userId => {
 ┊64┊63┊        if (!users.find(user => user.id === Number(userId))) {
 ┊65┊64┊          throw new Error(`User ${userId} doesn't exist.`);
```
```diff
@@ -82,7 +81,7 @@
 ┊82┊81┊      chats.push(chat);
 ┊83┊82┊      return chat;
 ┊84┊83┊    },
-┊85┊  ┊    updateGroup: (obj, {chatId, groupName, groupPicture}) => {
+┊  ┊84┊    updateGroup: (obj, {chatId, groupName, groupPicture}, {currentUser}) => {
 ┊86┊85┊      const chat = chats.find(chat => chat.id === Number(chatId));
 ┊87┊86┊
 ┊88┊87┊      if (!chat) {
```
```diff
@@ -98,7 +97,7 @@
 ┊ 98┊ 97┊
 ┊ 99┊ 98┊      return chat;
 ┊100┊ 99┊    },
-┊101┊   ┊    removeChat: (obj, {chatId}) => {
+┊   ┊100┊    removeChat: (obj, {chatId}, {currentUser}) => {
 ┊102┊101┊      const chat = chats.find(chat => chat.id === Number(chatId));
 ┊103┊102┊
 ┊104┊103┊      if (!chat) {
```
```diff
@@ -191,7 +190,7 @@
 ┊191┊190┊        return chatId;
 ┊192┊191┊      }
 ┊193┊192┊    },
-┊194┊   ┊    addMessage: (obj, {chatId, content}) => {
+┊   ┊193┊    addMessage: (obj, {chatId, content}, {currentUser}) => {
 ┊195┊194┊      if (content === null || content === '') {
 ┊196┊195┊        throw new Error(`Cannot add empty or null messages.`);
 ┊197┊196┊      }
```
```diff
@@ -268,7 +267,7 @@
 ┊268┊267┊
 ┊269┊268┊      return message;
 ┊270┊269┊    },
-┊271┊   ┊    removeMessages: (obj, {chatId, messageIds, all}) => {
+┊   ┊270┊    removeMessages: (obj, {chatId, messageIds, all}, {currentUser}) => {
 ┊272┊271┊      const chat = chats.find(chat => chat.id === Number(chatId));
 ┊273┊272┊
 ┊274┊273┊      if (!chat) {
```
```diff
@@ -308,9 +307,9 @@
 ┊308┊307┊    },
 ┊309┊308┊  },
 ┊310┊309┊  Chat: {
-┊311┊   ┊    name: (chat) => chat.name ? chat.name : users
+┊   ┊310┊    name: (chat, args, {currentUser}) => chat.name ? chat.name : users
 ┊312┊311┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
-┊313┊   ┊    picture: (chat) => chat.name ? chat.picture : users
+┊   ┊312┊    picture: (chat, args, {currentUser}) => chat.name ? chat.picture : users
 ┊314┊313┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.picture,
 ┊315┊314┊    allTimeMembers: (chat) => users.filter(user => chat.allTimeMemberIds.includes(user.id)),
 ┊316┊315┊    listingMembers: (chat) => users.filter(user => chat.listingMemberIds.includes(user.id)),
```
```diff
@@ -318,25 +317,25 @@
 ┊318┊317┊    admins: (chat) => users.filter(user => chat.adminIds && chat.adminIds.includes(user.id)),
 ┊319┊318┊    owner: (chat) => users.find(user => chat.ownerId === user.id) || null,
 ┊320┊319┊    isGroup: (chat) => !!chat.name,
-┊321┊   ┊    messages: (chat, {amount = 0}) => {
+┊   ┊320┊    messages: (chat, {amount = 0}, {currentUser}) => {
 ┊322┊321┊      const messages = chat.messages
 ┊323┊322┊      .filter(message => message.holderIds.includes(currentUser.id))
 ┊324┊323┊      .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf()) || [];
 ┊325┊324┊      return (amount ? messages.slice(0, amount) : messages).reverse();
 ┊326┊325┊    },
-┊327┊   ┊    lastMessage: (chat) => {
+┊   ┊326┊    lastMessage: (chat, args, {currentUser}) => {
 ┊328┊327┊      return chat.messages
 ┊329┊328┊        .filter(message => message.holderIds.includes(currentUser.id))
 ┊330┊329┊        .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf())[0] || null;
 ┊331┊330┊    },
-┊332┊   ┊    updatedAt: (chat) => {
+┊   ┊331┊    updatedAt: (chat, args, {currentUser}) => {
 ┊333┊332┊      const lastMessage = chat.messages
 ┊334┊333┊        .filter(message => message.holderIds.includes(currentUser.id))
 ┊335┊334┊        .sort((a, b) => b.createdAt.valueOf() - a.createdAt.valueOf())[0];
 ┊336┊335┊
 ┊337┊336┊      return lastMessage ? lastMessage.createdAt : chat.createdAt;
 ┊338┊337┊    },
-┊339┊   ┊    unreadMessages: (chat) => chat.messages
+┊   ┊338┊    unreadMessages: (chat, args, {currentUser}) => chat.messages
 ┊340┊339┊      .filter(message => message.holderIds.includes(currentUser.id) &&
 ┊341┊340┊        message.recipients.find(recipient => recipient.userId === currentUser.id && !recipient.readAt))
 ┊342┊341┊      .length,
```
```diff
@@ -345,7 +344,7 @@
 ┊345┊344┊    chat: (message) => chats.find(chat => message.chatId === chat.id)!,
 ┊346┊345┊    sender: (message) => users.find(user => user.id === message.senderId)!,
 ┊347┊346┊    holders: (message) => users.filter(user => message.holderIds.includes(user.id)),
-┊348┊   ┊    ownership: (message) => message.senderId === currentUser.id,
+┊   ┊347┊    ownership: (message, args, {currentUser}) => message.senderId === currentUser.id,
 ┊349┊348┊  },
 ┊350┊349┊  Recipient: {
 ┊351┊350┊    user: (recipient) => users.find(user => recipient.userId === user.id)!,
```

[}]: #

In the resolvers we're basically making use of the user object taken from the context.

## Authentication on client

Let's start installing `@angular/flex-layout`, because we will use it later:

    yarn add @angular/flex-layout


First of all we need to create an `HTTP Interceptor`, which will intercept every HTTP request and will add authentication headers.
For the moment we still don't have those headers, but we are going to store them in the `LocalStorage` later.
We are also creating an `AuthGuard`, which we will use to stop the user from reaching unauthorized Routes.

The `AuthGuard` will simply look for the presence of the `Authentication` Header, but will not guarantee that the header is authentic.
This is no problem, because client side AuthGuards are not safe by design and the real authentication will be done server side anyway.
AuthGuards are here just to redirect the user to the Login page.

The service we are going to create will contain some auth methods we are going to use across the app.

[{]: <helper> (diffStep "10.1" files="src/app/login/services" module="client")

#### Step 10.1: Authentication

##### Added src&#x2F;app&#x2F;login&#x2F;services&#x2F;auth.guard.ts
```diff
@@ -0,0 +1,18 @@
+┊  ┊ 1┊import {Injectable} from '@angular/core';
+┊  ┊ 2┊import {CanActivate, Router} from '@angular/router';
+┊  ┊ 3┊import {LoginService} from './login.service';
+┊  ┊ 4┊
+┊  ┊ 5┊@Injectable()
+┊  ┊ 6┊export class AuthGuard implements CanActivate {
+┊  ┊ 7┊  constructor(private router: Router,
+┊  ┊ 8┊              private loginService: LoginService) {}
+┊  ┊ 9┊
+┊  ┊10┊  canActivate() {
+┊  ┊11┊    if (this.loginService.getAuthHeader()) {
+┊  ┊12┊      return true;
+┊  ┊13┊    } else {
+┊  ┊14┊      this.router.navigate(['/login']);
+┊  ┊15┊      return false;
+┊  ┊16┊    }
+┊  ┊17┊  }
+┊  ┊18┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;services&#x2F;auth.interceptor.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import {Injectable} from '@angular/core';
+┊  ┊ 2┊import {HttpEvent, HttpHandler, HttpInterceptor, HttpRequest} from '@angular/common/http';
+┊  ┊ 3┊import {Observable} from 'rxjs';
+┊  ┊ 4┊import {LoginService} from './login.service';
+┊  ┊ 5┊
+┊  ┊ 6┊@Injectable()
+┊  ┊ 7┊export class AuthInterceptor implements HttpInterceptor {
+┊  ┊ 8┊  constructor(private loginService: LoginService) {}
+┊  ┊ 9┊  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
+┊  ┊10┊    const auth = this.loginService.getAuthHeader();
+┊  ┊11┊    if (auth) {
+┊  ┊12┊      request = request.clone({
+┊  ┊13┊        setHeaders: {
+┊  ┊14┊          Authorization: auth,
+┊  ┊15┊        }
+┊  ┊16┊      });
+┊  ┊17┊    }
+┊  ┊18┊    return next.handle(request);
+┊  ┊19┊  }
+┊  ┊20┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;services&#x2F;error.interceptor.ts
```diff
@@ -0,0 +1,26 @@
+┊  ┊ 1┊import { Injectable } from '@angular/core';
+┊  ┊ 2┊import { HttpRequest, HttpHandler, HttpEvent, HttpInterceptor } from '@angular/common/http';
+┊  ┊ 3┊import { Observable, throwError } from 'rxjs';
+┊  ┊ 4┊import { catchError } from 'rxjs/operators';
+┊  ┊ 5┊import { LoginService } from './login.service';
+┊  ┊ 6┊
+┊  ┊ 7┊@Injectable({
+┊  ┊ 8┊  providedIn: 'root'
+┊  ┊ 9┊})
+┊  ┊10┊export class ErrorInterceptor implements HttpInterceptor {
+┊  ┊11┊  constructor(private loginService: LoginService) {}
+┊  ┊12┊
+┊  ┊13┊  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
+┊  ┊14┊    return next.handle(request).pipe(catchError(err => {
+┊  ┊15┊      if (err.status === 401) {
+┊  ┊16┊        // auto logout if 401 response returned from api
+┊  ┊17┊        this.loginService.removeAuthHeader();
+┊  ┊18┊        this.loginService.removeUser();
+┊  ┊19┊        location.reload(true);
+┊  ┊20┊      }
+┊  ┊21┊
+┊  ┊22┊      const error = err.error.message || err.statusText;
+┊  ┊23┊      return throwError(error);
+┊  ┊24┊    }));
+┊  ┊25┊  }
+┊  ┊26┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;services&#x2F;login.service.ts
```diff
@@ -0,0 +1,36 @@
+┊  ┊ 1┊import { Injectable } from '@angular/core';
+┊  ┊ 2┊import {User} from '../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Injectable()
+┊  ┊ 5┊export class LoginService {
+┊  ┊ 6┊
+┊  ┊ 7┊  constructor() { }
+┊  ┊ 8┊
+┊  ┊ 9┊  storeAuthHeader(auth: string) {
+┊  ┊10┊    localStorage.setItem('Authorization', auth);
+┊  ┊11┊  }
+┊  ┊12┊
+┊  ┊13┊  getAuthHeader(): string {
+┊  ┊14┊    return localStorage.getItem('Authorization');
+┊  ┊15┊  }
+┊  ┊16┊
+┊  ┊17┊  removeAuthHeader() {
+┊  ┊18┊    localStorage.removeItem('Authorization');
+┊  ┊19┊  }
+┊  ┊20┊
+┊  ┊21┊  storeUser(user: User) {
+┊  ┊22┊    localStorage.setItem('user', JSON.stringify(user));
+┊  ┊23┊  }
+┊  ┊24┊
+┊  ┊25┊  getUser(): User {
+┊  ┊26┊    return JSON.parse(localStorage.getItem('user'));
+┊  ┊27┊  }
+┊  ┊28┊
+┊  ┊29┊  removeUser() {
+┊  ┊30┊    localStorage.removeItem('user');
+┊  ┊31┊  }
+┊  ┊32┊
+┊  ┊33┊  createBase64Auth(username: string, password: string): string {
+┊  ┊34┊    return `Basic ${btoa(`${username}:${password}`)}`;
+┊  ┊35┊  }
+┊  ┊36┊}
```

[}]: #

Now it's time to create a `SignIn`/`SignUp` component. Since we use Passport in the server we are going to make REST calls for the authentication, instead of using GraphQL.
Since we use `Basic Auth` we will simply combine the username and the password together to create the authentication header.
We will also store the response from the server, which will contain the user information like the ID, etc. which we are going to need later.

[{]: <helper> (diffStep "10.1" files="src/app/login/containers, src/app/login/login.module.ts" module="client")

#### Step 10.1: Authentication

##### Added src&#x2F;app&#x2F;login&#x2F;containers&#x2F;login.component.scss
```diff
@@ -0,0 +1,96 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  height: 100vh;
+┊  ┊ 3┊  display: block;
+┊  ┊ 4┊  background: radial-gradient(rgb(34,65,67), rgb(17,48,50)), url(/assets/chat-background.jpg) no-repeat;
+┊  ┊ 5┊  background-size: cover;
+┊  ┊ 6┊  background-blend-mode: multiply;
+┊  ┊ 7┊
+┊  ┊ 8┊  > img {
+┊  ┊ 9┊    width: 35%;
+┊  ┊10┊    height: auto;
+┊  ┊11┊    margin-left: auto;
+┊  ┊12┊    margin-right: auto;
+┊  ┊13┊    padding-top: 70px;
+┊  ┊14┊    display: block;
+┊  ┊15┊  }
+┊  ┊16┊
+┊  ┊17┊  > h2 {
+┊  ┊18┊    width: 100%;
+┊  ┊19┊    text-align: center;
+┊  ┊20┊    color: white;
+┊  ┊21┊  }
+┊  ┊22┊}
+┊  ┊23┊
+┊  ┊24┊form {
+┊  ┊25┊  padding: 20px;
+┊  ┊26┊
+┊  ┊27┊  > div {
+┊  ┊28┊    padding-top: 20px;
+┊  ┊29┊    padding-bottom: 20px;
+┊  ┊30┊  }
+┊  ┊31┊}
+┊  ┊32┊
+┊  ┊33┊mat-form-field ::ng-deep {
+┊  ┊34┊  width: 100%;
+┊  ┊35┊  background-color: transparent !important;
+┊  ┊36┊
+┊  ┊37┊  mat-label {
+┊  ┊38┊    color: white !important ;
+┊  ┊39┊  }
+┊  ┊40┊
+┊  ┊41┊  .mat-form-field-underline {
+┊  ┊42┊    background-color: white !important;
+┊  ┊43┊  }
+┊  ┊44┊
+┊  ┊45┊  .mat-focused .mat-form-field-label {
+┊  ┊46┊    color: white !important;
+┊  ┊47┊  }
+┊  ┊48┊
+┊  ┊49┊  .mat-form-field-ripple {
+┊  ┊50┊    background-color: var(--primary) !important;
+┊  ┊51┊  }
+┊  ┊52┊
+┊  ┊53┊  .mat-input-element {
+┊  ┊54┊    caret-color: white !important;
+┊  ┊55┊  }
+┊  ┊56┊
+┊  ┊57┊  .mat-input-element::placeholder {
+┊  ┊58┊    color: var(--primary);
+┊  ┊59┊  }
+┊  ┊60┊}
+┊  ┊61┊
+┊  ┊62┊legend {
+┊  ┊63┊  font-weight: bold;
+┊  ┊64┊  color: white;
+┊  ┊65┊}
+┊  ┊66┊
+┊  ┊67┊label {
+┊  ┊68┊  display: block;
+┊  ┊69┊  margin-top: 4px;
+┊  ┊70┊  margin-bottom: 4px;
+┊  ┊71┊}
+┊  ┊72┊
+┊  ┊73┊button {
+┊  ┊74┊  width: 100px;
+┊  ┊75┊  display: block;
+┊  ┊76┊  margin-left: auto;
+┊  ┊77┊  margin-right: auto;
+┊  ┊78┊}
+┊  ┊79┊
+┊  ┊80┊.error {
+┊  ┊81┊  color: red;
+┊  ┊82┊  font-size: 13px;
+┊  ┊83┊  margin-top: -15.5px;
+┊  ┊84┊}
+┊  ┊85┊
+┊  ┊86┊.alternative {
+┊  ┊87┊  position: fixed;
+┊  ┊88┊  left: 10px;
+┊  ┊89┊  bottom: 10px;
+┊  ┊90┊  color: white;
+┊  ┊91┊  font-size: 14px;
+┊  ┊92┊
+┊  ┊93┊  a {
+┊  ┊94┊    color: var(--secondary);
+┊  ┊95┊  }
+┊  ┊96┊}🚫↵
```

##### Added src&#x2F;app&#x2F;login&#x2F;containers&#x2F;login.component.ts
```diff
@@ -0,0 +1,136 @@
+┊   ┊  1┊import {Component} from '@angular/core';
+┊   ┊  2┊import {HttpClient} from '@angular/common/http';
+┊   ┊  3┊import {FormBuilder, Validators} from '@angular/forms';
+┊   ┊  4┊// import {matchOtherValidator} from '@moebius/ng-validators';
+┊   ┊  5┊import {Router} from '@angular/router';
+┊   ┊  6┊import {User} from '../../../graphql';
+┊   ┊  7┊import {LoginService} from '../services/login.service';
+┊   ┊  8┊
+┊   ┊  9┊@Component({
+┊   ┊ 10┊  selector: 'app-login',
+┊   ┊ 11┊  template: `
+┊   ┊ 12┊    <img src="assets/whatsapp-icon.png" />
+┊   ┊ 13┊    <h2>WhatsApp Clone</h2>
+┊   ┊ 14┊    <form *ngIf="signingIn" (ngSubmit)="signIn()" [formGroup]="signInForm" novalidate>
+┊   ┊ 15┊      <legend>Sign in</legend>
+┊   ┊ 16┊      <div style="width: 100%">
+┊   ┊ 17┊        <mat-form-field>
+┊   ┊ 18┊          <mat-label>Username</mat-label>
+┊   ┊ 19┊          <input matInput autocomplete="username" formControlName="username" type="text" placeholder="Enter your username" />
+┊   ┊ 20┊        </mat-form-field>
+┊   ┊ 21┊        <div class="error" *ngIf="signInForm.get('username').hasError('required') && signInForm.get('username').touched">
+┊   ┊ 22┊          Username is required
+┊   ┊ 23┊        </div>
+┊   ┊ 24┊        <mat-form-field>
+┊   ┊ 25┊          <mat-label>Password</mat-label>
+┊   ┊ 26┊          <input matInput autocomplete="current-password" formControlName="password" type="password" placeholder="Enter your password" />
+┊   ┊ 27┊        </mat-form-field>
+┊   ┊ 28┊        <div class="error" *ngIf="signInForm.get('password').hasError('required') && signInForm.get('password').touched">
+┊   ┊ 29┊          Password is required
+┊   ┊ 30┊        </div>
+┊   ┊ 31┊      </div>
+┊   ┊ 32┊      <button mat-button type="submit" color="secondary" [disabled]="signInForm.invalid">Sign in</button>
+┊   ┊ 33┊      <span class="alternative">Don't have an account yet? <a (click)="signingIn = false">Sign up!</a></span>
+┊   ┊ 34┊    </form>
+┊   ┊ 35┊    <form *ngIf="!signingIn" (ngSubmit)="signUp()" [formGroup]="signUpForm" novalidate>
+┊   ┊ 36┊      <legend>Sign up</legend>
+┊   ┊ 37┊      <div style="float: left; width: calc(50% - 10px); padding-right: 10px;">
+┊   ┊ 38┊        <mat-form-field>
+┊   ┊ 39┊          <mat-label>Name</mat-label>
+┊   ┊ 40┊          <input matInput autocomplete="name" formControlName="name" type="text" placeholder="Enter your name" />
+┊   ┊ 41┊        </mat-form-field>
+┊   ┊ 42┊        <mat-form-field>
+┊   ┊ 43┊          <mat-label>Username</mat-label>
+┊   ┊ 44┊          <input matInput autocomplete="username" formControlName="username" type="text" placeholder="Enter your username" />
+┊   ┊ 45┊        </mat-form-field>
+┊   ┊ 46┊        <div class="error" *ngIf="signUpForm.get('username').hasError('required') && signUpForm.get('username').touched">
+┊   ┊ 47┊          Username is required
+┊   ┊ 48┊        </div>
+┊   ┊ 49┊      </div>
+┊   ┊ 50┊      <div style="float: right; width: calc(50% - 10px); padding-left: 10px;">
+┊   ┊ 51┊        <mat-form-field>
+┊   ┊ 52┊          <mat-label>Password</mat-label>
+┊   ┊ 53┊          <input matInput autocomplete="new-password" formControlName="newPassword" type="password" placeholder="Enter your password" />
+┊   ┊ 54┊        </mat-form-field>
+┊   ┊ 55┊        <div class="error" *ngIf="signUpForm.get('newPassword').hasError('required') && signUpForm.get('newPassword').touched">
+┊   ┊ 56┊          Password is required
+┊   ┊ 57┊        </div>
+┊   ┊ 58┊        <mat-form-field>
+┊   ┊ 59┊          <mat-label>Confirm Password</mat-label>
+┊   ┊ 60┊          <input matInput autocomplete="new-password" formControlName="confirmPassword" type="password" placeholder="Confirm password" />
+┊   ┊ 61┊        </mat-form-field>
+┊   ┊ 62┊        <div class="error" *ngIf="signUpForm.get('confirmPassword').hasError('required') && signUpForm.get('confirmPassword').touched">
+┊   ┊ 63┊          Passwords must match
+┊   ┊ 64┊        </div>
+┊   ┊ 65┊      </div>
+┊   ┊ 66┊      <button mat-button type="submit" color="secondary" [disabled]="signUpForm.invalid">Sign up</button>
+┊   ┊ 67┊      <span class="alternative">Already have an account? <a (click)="signingIn = true">Sign in!</a></span>
+┊   ┊ 68┊    </form>
+┊   ┊ 69┊  `,
+┊   ┊ 70┊  styleUrls: ['./login.component.scss'],
+┊   ┊ 71┊})
+┊   ┊ 72┊export class LoginComponent {
+┊   ┊ 73┊  signInForm = this.fb.group({
+┊   ┊ 74┊    username: [null, [
+┊   ┊ 75┊      Validators.required,
+┊   ┊ 76┊    ]],
+┊   ┊ 77┊    password: [null, [
+┊   ┊ 78┊      Validators.required,
+┊   ┊ 79┊    ]],
+┊   ┊ 80┊  });
+┊   ┊ 81┊
+┊   ┊ 82┊  signUpForm = this.fb.group({
+┊   ┊ 83┊    name: [null, [
+┊   ┊ 84┊      Validators.required,
+┊   ┊ 85┊    ]],
+┊   ┊ 86┊    username: [null, [
+┊   ┊ 87┊      Validators.required,
+┊   ┊ 88┊    ]],
+┊   ┊ 89┊    newPassword: [null, [
+┊   ┊ 90┊      Validators.required,
+┊   ┊ 91┊    ]],
+┊   ┊ 92┊    confirmPassword: [null, [
+┊   ┊ 93┊      Validators.required,
+┊   ┊ 94┊      // matchOtherValidator('newPassword'),
+┊   ┊ 95┊    ]],
+┊   ┊ 96┊  });
+┊   ┊ 97┊
+┊   ┊ 98┊  public signingIn = true;
+┊   ┊ 99┊
+┊   ┊100┊  constructor(private http: HttpClient,
+┊   ┊101┊              private fb: FormBuilder,
+┊   ┊102┊              private router: Router,
+┊   ┊103┊              private loginService: LoginService) {}
+┊   ┊104┊
+┊   ┊105┊  signIn() {
+┊   ┊106┊    const {username, password} = this.signInForm.value;
+┊   ┊107┊    const auth = `Basic ${btoa(`${username}:${password}`)}`;
+┊   ┊108┊    this.http.post('http://localhost:4000/signin', null, {
+┊   ┊109┊      headers: {
+┊   ┊110┊        Authorization: auth,
+┊   ┊111┊      }
+┊   ┊112┊    }).subscribe((user: User) => {
+┊   ┊113┊      this.loginService.storeAuthHeader(auth);
+┊   ┊114┊      this.loginService.storeUser(user);
+┊   ┊115┊      this.router.navigate(['/chats']);
+┊   ┊116┊    }, err => console.error(err));
+┊   ┊117┊  }
+┊   ┊118┊
+┊   ┊119┊  signUp() {
+┊   ┊120┊    const {username, newPassword: password, name} = this.signUpForm.value;
+┊   ┊121┊    const auth = this.loginService.createBase64Auth(username, password);
+┊   ┊122┊    this.http.post('http://localhost:4000/signup', {
+┊   ┊123┊      name,
+┊   ┊124┊      username,
+┊   ┊125┊      password,
+┊   ┊126┊    }, {
+┊   ┊127┊      headers: {
+┊   ┊128┊        Authorization: auth,
+┊   ┊129┊      }
+┊   ┊130┊    }).subscribe((user: User) => {
+┊   ┊131┊      this.loginService.storeAuthHeader(auth);
+┊   ┊132┊      this.loginService.storeUser(user);
+┊   ┊133┊      this.router.navigate(['/chats']);
+┊   ┊134┊    }, err => console.error(err));
+┊   ┊135┊  }
+┊   ┊136┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;login.module.ts
```diff
@@ -0,0 +1,53 @@
+┊  ┊ 1┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊ 2┊import {NgModule} from '@angular/core';
+┊  ┊ 3┊import {MatButtonModule, MatIconModule, MatListModule, MatMenuModule, MatFormFieldModule, MatInputModule} from '@angular/material';
+┊  ┊ 4┊import {SharedModule} from '../shared/shared.module';
+┊  ┊ 5┊import {BrowserModule} from '@angular/platform-browser';
+┊  ┊ 6┊import {FormsModule, ReactiveFormsModule} from '@angular/forms';
+┊  ┊ 7┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 8┊import {LoginComponent} from './containers/login.component';
+┊  ┊ 9┊import {FlexLayoutModule} from '@angular/flex-layout';
+┊  ┊10┊import {AuthInterceptor} from './services/auth.interceptor';
+┊  ┊11┊import {AuthGuard} from './services/auth.guard';
+┊  ┊12┊import {LoginService} from './services/login.service';
+┊  ┊13┊import { ErrorInterceptor } from './services/error.interceptor';
+┊  ┊14┊
+┊  ┊15┊
+┊  ┊16┊const routes: Routes = [
+┊  ┊17┊  {path: 'login', component: LoginComponent},
+┊  ┊18┊];
+┊  ┊19┊
+┊  ┊20┊@NgModule({
+┊  ┊21┊  declarations: [
+┊  ┊22┊    LoginComponent,
+┊  ┊23┊  ],
+┊  ┊24┊  imports: [
+┊  ┊25┊    BrowserModule,
+┊  ┊26┊    // Material
+┊  ┊27┊    MatInputModule,
+┊  ┊28┊    MatFormFieldModule,
+┊  ┊29┊    MatMenuModule,
+┊  ┊30┊    MatIconModule,
+┊  ┊31┊    MatButtonModule,
+┊  ┊32┊    MatListModule,
+┊  ┊33┊    // Animations
+┊  ┊34┊    BrowserAnimationsModule,
+┊  ┊35┊    // Flex layout
+┊  ┊36┊    FlexLayoutModule,
+┊  ┊37┊    // Routing
+┊  ┊38┊    RouterModule.forChild(routes),
+┊  ┊39┊    // Forms
+┊  ┊40┊    FormsModule,
+┊  ┊41┊    ReactiveFormsModule,
+┊  ┊42┊    // Feature modules
+┊  ┊43┊    SharedModule,
+┊  ┊44┊  ],
+┊  ┊45┊  providers: [
+┊  ┊46┊    LoginService,
+┊  ┊47┊    AuthInterceptor,
+┊  ┊48┊    ErrorInterceptor,
+┊  ┊49┊    AuthGuard,
+┊  ┊50┊  ],
+┊  ┊51┊})
+┊  ┊52┊export class LoginModule {
+┊  ┊53┊}
```

[}]: #

Now it's time use the Interceptor we just created:

[{]: <helper> (diffStep "10.1" files="src/app/app.module.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -2,12 +2,15 @@
 ┊ 2┊ 2┊import { NgModule } from '@angular/core';
 ┊ 3┊ 3┊
 ┊ 4┊ 4┊import { AppComponent } from './app.component';
-┊ 5┊  ┊import { HttpClientModule } from '@angular/common/http';
+┊  ┊ 5┊import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';
 ┊ 6┊ 6┊import { GraphQLModule } from './graphql.module';
 ┊ 7┊ 7┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
 ┊ 8┊ 8┊import {RouterModule, Routes} from '@angular/router';
 ┊ 9┊ 9┊import {ChatViewerModule} from './chat-viewer/chat-viewer.module';
 ┊10┊10┊import {ChatsCreationModule} from './chats-creation/chats-creation.module';
+┊  ┊11┊import {LoginModule} from './login/login.module';
+┊  ┊12┊import {AuthInterceptor} from './login/services/auth.interceptor';
+┊  ┊13┊import { ErrorInterceptor } from './login/services/error.interceptor';
 ┊11┊14┊const routes: Routes = [];
 ┊12┊15┊
 ┊13┊16┊@NgModule({
```
```diff
@@ -24,8 +27,20 @@
 ┊24┊27┊    ChatsListerModule,
 ┊25┊28┊    ChatViewerModule,
 ┊26┊29┊    ChatsCreationModule,
+┊  ┊30┊    LoginModule,
+┊  ┊31┊  ],
+┊  ┊32┊  providers: [
+┊  ┊33┊    {
+┊  ┊34┊      provide: HTTP_INTERCEPTORS,
+┊  ┊35┊      useClass: AuthInterceptor,
+┊  ┊36┊      multi: true,
+┊  ┊37┊    },
+┊  ┊38┊    {
+┊  ┊39┊      provide: HTTP_INTERCEPTORS,
+┊  ┊40┊      useClass: ErrorInterceptor,
+┊  ┊41┊      multi: true,
+┊  ┊42┊    },
 ┊27┊43┊  ],
-┊28┊  ┊  providers: [],
 ┊29┊44┊  bootstrap: [AppComponent]
 ┊30┊45┊})
 ┊31┊46┊export class AppModule {}
```

[}]: #

As well as the `AuthGuard`:

[{]: <helper> (diffStep "10.1" files="src/app/chat-viewer/chat-viewer.module.ts, src/app/chats-creation/chats-creation.module.ts, src/app/chats-lister/chats-lister.module.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;chat-viewer.module.ts
```diff
@@ -12,11 +12,12 @@
 ┊12┊12┊import {NewMessageComponent} from './components/new-message/new-message.component';
 ┊13┊13┊import {SharedModule} from '../shared/shared.module';
 ┊14┊14┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊15┊import {AuthGuard} from '../login/services/auth.guard';
 ┊15┊16┊
 ┊16┊17┊const routes: Routes = [
 ┊17┊18┊  {
 ┊18┊19┊    path: 'chat', children: [
-┊19┊  ┊      {path: ':id', component: ChatComponent},
+┊  ┊20┊      {path: ':id', canActivate: [AuthGuard], component: ChatComponent},
 ┊20┊21┊    ],
 ┊21┊22┊  },
 ┊22┊23┊];
```

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;chats-creation.module.ts
```diff
@@ -16,10 +16,11 @@
 ┊16┊16┊import {NewGroupDetailsComponent} from './components/new-group-details/new-group-details.component';
 ┊17┊17┊import {SharedModule} from '../shared/shared.module';
 ┊18┊18┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊19┊import {AuthGuard} from '../login/services/auth.guard';
 ┊19┊20┊
 ┊20┊21┊const routes: Routes = [
-┊21┊  ┊  {path: 'new-chat', component: NewChatComponent},
-┊22┊  ┊  {path: 'new-group', component: NewGroupComponent},
+┊  ┊22┊  {path: 'new-chat', canActivate: [AuthGuard], component: NewChatComponent},
+┊  ┊23┊  {path: 'new-group', canActivate: [AuthGuard], component: NewGroupComponent},
 ┊23┊24┊];
 ┊24┊25┊
 ┊25┊26┊@NgModule({
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;chats-lister.module.ts
```diff
@@ -11,10 +11,11 @@
 ┊11┊11┊import {ChatsListComponent} from './components/chats-list/chats-list.component';
 ┊12┊12┊import {SharedModule} from '../shared/shared.module';
 ┊13┊13┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊14┊import {AuthGuard} from '../login/services/auth.guard';
 ┊14┊15┊
 ┊15┊16┊const routes: Routes = [
 ┊16┊17┊  {path: '', redirectTo: 'chats', pathMatch: 'full'},
-┊17┊  ┊  {path: 'chats', component: ChatsComponent},
+┊  ┊18┊  {path: 'chats', canActivate: [AuthGuard], component: ChatsComponent},
 ┊18┊19┊];
 ┊19┊20┊
 ┊20┊21┊@NgModule({
```

[}]: #

Last but not the least we need to fix our main service in order to not use the hardcoded user anymore. Instead we will use our Login service to read the user info from the `LocalStorage`.

[{]: <helper> (diffStep "10.1" files="src/app/services/chats.service.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -24,6 +24,7 @@
 ┊24┊24┊} from '../../graphql';
 ┊25┊25┊import { DataProxy } from 'apollo-cache';
 ┊26┊26┊import { FetchResult } from 'apollo-link';
+┊  ┊27┊import {LoginService} from '../login/services/login.service';
 ┊27┊28┊
 ┊28┊29┊const currentUserId = '1';
 ┊29┊30┊const currentUserName = 'Ethan Gonzalez';
```
```diff
@@ -46,7 +47,8 @@
 ┊46┊47┊    private removeAllMessagesGQL: RemoveAllMessagesGQL,
 ┊47┊48┊    private getUsersGQL: GetUsersGQL,
 ┊48┊49┊    private addChatGQL: AddChatGQL,
-┊49┊  ┊    private addGroupGQL: AddGroupGQL
+┊  ┊50┊    private addGroupGQL: AddGroupGQL,
+┊  ┊51┊    private loginService: LoginService
 ┊50┊52┊  ) {
 ┊51┊53┊    this.getChatsWq = this.getChatsGQL.watch({
 ┊52┊54┊      amount: this.messagesAmount,
```
```diff
@@ -135,8 +137,8 @@
 ┊135┊137┊          },
 ┊136┊138┊          sender: {
 ┊137┊139┊            __typename: 'User',
-┊138┊   ┊            id: currentUserId,
-┊139┊   ┊            name: currentUserName,
+┊   ┊140┊            id: this.loginService.getUser().id,
+┊   ┊141┊            name: this.loginService.getUser().name,
 ┊140┊142┊          },
 ┊141┊143┊          content,
 ┊142┊144┊          createdAt: moment().unix(),
```
```diff
@@ -318,7 +320,7 @@
 ┊318┊320┊  // Checks if the chat is listed for the current user and returns the id
 ┊319┊321┊  getChatId(userId: string) {
 ┊320┊322┊    const _chat = this.chats.find(chat => {
-┊321┊   ┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === currentUserId) &&
+┊   ┊323┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === this.loginService.getUser().id) &&
 ┊322┊324┊        !!chat.allTimeMembers.find(user => user.id === userId);
 ┊323┊325┊    });
 ┊324┊326┊    return _chat ? _chat.id : false;
```
```diff
@@ -338,7 +340,7 @@
 ┊338┊340┊            picture: users.find(user => user.id === userId).picture,
 ┊339┊341┊            allTimeMembers: [
 ┊340┊342┊              {
-┊341┊   ┊                id: currentUserId,
+┊   ┊343┊                id: this.loginService.getUser().id,
 ┊342┊344┊                __typename: 'User',
 ┊343┊345┊              },
 ┊344┊346┊              {
```
```diff
@@ -390,10 +392,10 @@
 ┊390┊392┊            id: ouiId,
 ┊391┊393┊            name: groupName,
 ┊392┊394┊            picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
-┊393┊   ┊            userIds: [currentUserId, userIds],
+┊   ┊395┊            userIds: [this.loginService.getUser().id, userIds],
 ┊394┊396┊            allTimeMembers: [
 ┊395┊397┊              {
-┊396┊   ┊                id: currentUserId,
+┊   ┊398┊                id: this.loginService.getUser().id,
 ┊397┊399┊                __typename: 'User',
 ┊398┊400┊              },
 ┊399┊401┊              ...userIds.map(id => ({id, __typename: 'User'})),
```

[}]: #

We also need to fix our tests:

[{]: <helper> (diffStep "10.1" files="src/app/chat-viewer/containers/chat/chat.component.spec.ts, src/app/chats-lister/containers/chats/chats.component.spec.ts, src/app/services/chats.service.spec.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -29,6 +29,7 @@
 ┊29┊29┊import { NewMessageComponent } from '../../components/new-message/new-message.component';
 ┊30┊30┊import { MessagesListComponent } from '../../components/messages-list/messages-list.component';
 ┊31┊31┊import { MessageItemComponent } from '../../components/message-item/message-item.component';
+┊  ┊32┊import { LoginService } from '../../../login/services/login.service';
 ┊32┊33┊
 ┊33┊34┊describe('ChatComponent', () => {
 ┊34┊35┊  let component: ChatComponent;
```
```diff
@@ -134,6 +135,7 @@
 ┊134┊135┊            queryParams: of({}),
 ┊135┊136┊          },
 ┊136┊137┊        },
+┊   ┊138┊        LoginService,
 ┊137┊139┊      ],
 ┊138┊140┊      schemas: [NO_ERRORS_SCHEMA],
 ┊139┊141┊    }).compileComponents();
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -23,6 +23,7 @@
 ┊23┊23┊import { ChatsListComponent } from '../../components/chats-list/chats-list.component';
 ┊24┊24┊import { ChatItemComponent } from '../../components/chat-item/chat-item.component';
 ┊25┊25┊import { ChatsService } from '../../../services/chats.service';
+┊  ┊26┊import { LoginService } from '../../../login/services/login.service';
 ┊26┊27┊
 ┊27┊28┊describe('ChatsComponent', () => {
 ┊28┊29┊  let component: ChatsComponent;
```
```diff
@@ -343,6 +344,7 @@
 ┊343┊344┊            return new InMemoryCache({ dataIdFromObject });
 ┊344┊345┊          },
 ┊345┊346┊        },
+┊   ┊347┊        LoginService,
 ┊346┊348┊      ],
 ┊347┊349┊      schemas: [NO_ERRORS_SCHEMA],
 ┊348┊350┊    }).compileComponents();
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.spec.ts
```diff
@@ -11,6 +11,7 @@
 ┊11┊11┊import { GetChats } from '../../graphql';
 ┊12┊12┊import { dataIdFromObject } from '../graphql.module';
 ┊13┊13┊import { ChatsService } from './chats.service';
+┊  ┊14┊import {LoginService} from '../login/services/login.service';
 ┊14┊15┊
 ┊15┊16┊describe('ChatsService', () => {
 ┊16┊17┊  let controller: ApolloTestingController;
```
```diff
@@ -312,6 +313,7 @@
 ┊312┊313┊      imports: [ApolloTestingModule],
 ┊313┊314┊      providers: [
 ┊314┊315┊        ChatsService,
+┊   ┊316┊        LoginService,
 ┊315┊317┊        {
 ┊316┊318┊          provide: APOLLO_TESTING_CACHE,
 ┊317┊319┊          useFactory() {
```

[}]: #


### GraphQL Server behind a firewall

There's still one thing remaining. Our GraphQL Code Generator setup is broken now. That's because every made request to the server has to have proper rights and currently the codegen's request introspection query is not authenticated. In short, we need to attach an `Authorization` header.

We're going to implement the following scenario:

1. User runs `yarn generator`
1. He is asked for a _username_ and a _password_
1. Our script generates an _Authorization_ header
1. GraphQL Code Generator get the GraphQL Schema of our server and outputs the types

To achieve the goal, we need to change the way we interact with the codegen.
Right now we use GraphQL Code Generator's CLI but the tool allows to use it programatically. So one issue solved.

What about getting user's name and password part? We will use in-terminal prompt so user can type those. There's a package for that:

    yarn add -D prompt

Next, create a `src/codegen.ts` file and write our script there:

[{]: <helper> (diffStep "10.1" files="src/codegen.ts" module="client")

#### Step 10.1: Authentication

##### Added src&#x2F;codegen.ts
```diff
@@ -0,0 +1,60 @@
+┊  ┊ 1┊import * as prompt from 'prompt';
+┊  ┊ 2┊import { generate } from 'graphql-code-generator';
+┊  ┊ 3┊
+┊  ┊ 4┊interface Credentials {
+┊  ┊ 5┊  username: string;
+┊  ┊ 6┊  password: string;
+┊  ┊ 7┊}
+┊  ┊ 8┊
+┊  ┊ 9┊async function getCredentials(): Promise<Credentials> {
+┊  ┊10┊  return new Promise<Credentials>(resolve => {
+┊  ┊11┊    prompt.start();
+┊  ┊12┊    prompt.get(['username', 'password'], (_err, result) => {
+┊  ┊13┊      resolve({
+┊  ┊14┊        username: result.username,
+┊  ┊15┊        password: result.password,
+┊  ┊16┊      });
+┊  ┊17┊    });
+┊  ┊18┊  });
+┊  ┊19┊}
+┊  ┊20┊
+┊  ┊21┊function generateHeaders({
+┊  ┊22┊  username,
+┊  ┊23┊  password,
+┊  ┊24┊}: Credentials): Record<string, any> {
+┊  ┊25┊  const Authorization = `Basic ${Buffer.from(
+┊  ┊26┊    `${username}:${password}`,
+┊  ┊27┊  ).toString('base64')}`;
+┊  ┊28┊
+┊  ┊29┊  return { Authorization };
+┊  ┊30┊}
+┊  ┊31┊
+┊  ┊32┊async function main() {
+┊  ┊33┊  const credentials = await getCredentials();
+┊  ┊34┊  const headers = generateHeaders(credentials);
+┊  ┊35┊
+┊  ┊36┊  await generate(
+┊  ┊37┊    {
+┊  ┊38┊      schema: {
+┊  ┊39┊        'http://localhost:4000/graphql': {
+┊  ┊40┊          headers,
+┊  ┊41┊        },
+┊  ┊42┊      },
+┊  ┊43┊      documents: './src/graphql/**/*.ts',
+┊  ┊44┊      generates: {
+┊  ┊45┊        './src/graphql.ts': {
+┊  ┊46┊          plugins: [
+┊  ┊47┊            'typescript-common',
+┊  ┊48┊            'typescript-client',
+┊  ┊49┊            'typescript-server',
+┊  ┊50┊            'typescript-apollo-angular',
+┊  ┊51┊          ],
+┊  ┊52┊        },
+┊  ┊53┊      },
+┊  ┊54┊      overwrite: true,
+┊  ┊55┊    },
+┊  ┊56┊    true,
+┊  ┊57┊  );
+┊  ┊58┊}
+┊  ┊59┊
+┊  ┊60┊main();
```

[}]: #

Let's take a closer look what we did there:

- We asks user for a _username_ and a _password_ with `prompt`
- Based on that we generate a header
- We put the header in `schema` option
- there's a `true` value as the second argument, it tells codegen to write the output to a file

Finally we need to change our `generator` script in `package.json` to `"ts-node src/codegen.ts"` and to tweak a couple of configs:

[{]: <helper> (diffStep "10.1" files="tsconfig.json, src/tsconfig.app.json" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;tsconfig.app.json
```diff
@@ -2,7 +2,7 @@
 ┊2┊2┊  "extends": "../tsconfig.json",
 ┊3┊3┊  "compilerOptions": {
 ┊4┊4┊    "outDir": "../out-tsc/app",
-┊5┊ ┊    "types": []
+┊ ┊5┊    "types": [ "node" ]
 ┊6┊6┊  },
 ┊7┊7┊  "exclude": [
 ┊8┊8┊    "test.ts",
```

##### Changed tsconfig.json
```diff
@@ -5,7 +5,6 @@
 ┊ 5┊ 5┊    "outDir": "./dist/out-tsc",
 ┊ 6┊ 6┊    "sourceMap": true,
 ┊ 7┊ 7┊    "declaration": false,
-┊ 8┊  ┊    "module": "es2015",
 ┊ 9┊ 8┊    "moduleResolution": "node",
 ┊10┊ 9┊    "emitDecoratorMetadata": true,
 ┊11┊10┊    "experimentalDecorators": true,
```

[}]: #

With all of that, whenever we run `yarn generator`, we access the GraphQL server as an authenticated user.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step9.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step11.md) |
|:--------------------------------|--------------------------------:|

[}]: #
