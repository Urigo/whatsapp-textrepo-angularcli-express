# Step 9: Zero latency on slow 3g networks

[//]: # (head-end)


## Client

Now let's start our client in production mode:

    yarn start --prod

Now open the **Chrome Developers Tools** and, in the **Network tab**, select `Slow 3G Network` and `Disable cache`.
Then refresh the page and look at the `DOMContentLoaded` time and at the transferred size. You'll notice that our bundle size is quite small and so the loads time.

Now let's click on a specific chat. It will take some time to load the data and then add a new message.
Once again it will take some time to load the data.
We could also create a new chat and the result would be the same.
The whole app doesn't feel as snappier as the real Whatsapp on a slow 3G Network.

"That's normal, it's a web application with a remote db while Whatsapp is a native app with a local database..."
That's just an excuse, because we can do as good as Whatsapp thanks to Apollo!

Let's install `moment`, we will soon need it:

    yarn add moment

Start by making our UI optimistic. We can predict most of the response we will get from our server, except for a few things like `id` of newly created messages. But since we don't really need that id, we can simply generate a fake one
which will be later overridden once we get the response from the server:

[{]: <helper> (diffStep "9.1" files="^\(?!package.json$|yarn.lock$\).*" module="client")

#### Step 9.1: Optimistic updates

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.ts
```diff
@@ -47,7 +47,7 @@
 ┊47┊47┊      // Chat is already listed for the current user
 ┊48┊48┊      this.router.navigate(['/chat', chatId]);
 ┊49┊49┊    } else {
-┊50┊  ┊      this.chatsService.addChat(userId).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
+┊  ┊50┊      this.chatsService.addChat(userId, this.users).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
 ┊51┊51┊        this.router.navigate(['/chat', id]);
 ┊52┊52┊      });
 ┊53┊53┊    }
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -2,6 +2,7 @@
 ┊2┊2┊import {Injectable} from '@angular/core';
 ┊3┊3┊import {Observable} from 'rxjs';
 ┊4┊4┊import {QueryRef} from 'apollo-angular';
+┊ ┊5┊import * as moment from 'moment';
 ┊5┊6┊import {
 ┊6┊7┊  GetChatsGQL,
 ┊7┊8┊  GetChatGQL,
```
```diff
@@ -17,10 +18,12 @@
 ┊17┊18┊  GetChat,
 ┊18┊19┊  RemoveMessages,
 ┊19┊20┊  RemoveAllMessages,
+┊  ┊21┊  GetUsers,
 ┊20┊22┊} from '../../graphql';
 ┊21┊23┊import { DataProxy } from 'apollo-cache';
 ┊22┊24┊
 ┊23┊25┊const currentUserId = '1';
+┊  ┊26┊const currentUserName = 'Ethan Gonzalez';
 ┊24┊27┊
 ┊25┊28┊@Injectable()
 ┊26┊29┊export class ChatsService {
```
```diff
@@ -49,6 +52,10 @@
 ┊49┊52┊    this.chats$.subscribe(chats => this.chats = chats);
 ┊50┊53┊  }
 ┊51┊54┊
+┊  ┊55┊  static getRandomId() {
+┊  ┊56┊    return String(Math.round(Math.random() * 1000000000000));
+┊  ┊57┊  }
+┊  ┊58┊
 ┊52┊59┊  getChats() {
 ┊53┊60┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊54┊61┊  }
```
```diff
@@ -70,6 +77,27 @@
 ┊ 70┊ 77┊      chatId,
 ┊ 71┊ 78┊      content,
 ┊ 72┊ 79┊    }, {
+┊   ┊ 80┊      optimisticResponse: {
+┊   ┊ 81┊        __typename: 'Mutation',
+┊   ┊ 82┊        addMessage: {
+┊   ┊ 83┊          __typename: 'Message',
+┊   ┊ 84┊          id: ChatsService.getRandomId(),
+┊   ┊ 85┊          chat: {
+┊   ┊ 86┊            __typename: 'Chat',
+┊   ┊ 87┊            id: chatId,
+┊   ┊ 88┊          },
+┊   ┊ 89┊          sender: {
+┊   ┊ 90┊            __typename: 'User',
+┊   ┊ 91┊            id: currentUserId,
+┊   ┊ 92┊            name: currentUserName,
+┊   ┊ 93┊          },
+┊   ┊ 94┊          content,
+┊   ┊ 95┊          createdAt: moment().unix(),
+┊   ┊ 96┊          type: 1,
+┊   ┊ 97┊          recipients: [],
+┊   ┊ 98┊          ownership: true,
+┊   ┊ 99┊        },
+┊   ┊100┊      },
 ┊ 73┊101┊      update: (store, { data: { addMessage } }: {data: AddMessage.Mutation}) => {
 ┊ 74┊102┊        // Update the messages cache
 ┊ 75┊103┊        {
```
```diff
@@ -121,6 +149,10 @@
 ┊121┊149┊      {
 ┊122┊150┊        chatId,
 ┊123┊151┊      }, {
+┊   ┊152┊        optimisticResponse: {
+┊   ┊153┊          __typename: 'Mutation',
+┊   ┊154┊          removeChat: chatId,
+┊   ┊155┊        },
 ┊124┊156┊        update: (store, { data: { removeChat } }) => {
 ┊125┊157┊          // Read the data from our cache for this query.
 ┊126┊158┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
```
```diff
@@ -154,6 +186,10 @@
 ┊154┊186┊    let ids: string[] = [];
 ┊155┊187┊
 ┊156┊188┊    const options = {
+┊   ┊189┊      optimisticResponse: () => ({
+┊   ┊190┊        __typename: 'Mutation',
+┊   ┊191┊        removeMessages: ids,
+┊   ┊192┊      }),
 ┊157┊193┊      update: (store: DataProxy, { data: { removeMessages } }: {data: RemoveMessages.Mutation | RemoveAllMessages.Mutation}) => {
 ┊158┊194┊        // Update the messages cache
 ┊159┊195┊        {
```
```diff
@@ -241,11 +277,33 @@
 ┊241┊277┊    return _chat ? _chat.id : false;
 ┊242┊278┊  }
 ┊243┊279┊
-┊244┊   ┊  addChat(userId: string) {
+┊   ┊280┊  addChat(userId: string, users: GetUsers.Users[]) {
 ┊245┊281┊    return this.addChatGQL.mutate(
 ┊246┊282┊      {
 ┊247┊283┊        userId,
 ┊248┊284┊      }, {
+┊   ┊285┊        optimisticResponse: {
+┊   ┊286┊          __typename: 'Mutation',
+┊   ┊287┊          addChat: {
+┊   ┊288┊            __typename: 'Chat',
+┊   ┊289┊            id: ChatsService.getRandomId(),
+┊   ┊290┊            name: users.find(user => user.id === userId).name,
+┊   ┊291┊            picture: users.find(user => user.id === userId).picture,
+┊   ┊292┊            allTimeMembers: [
+┊   ┊293┊              {
+┊   ┊294┊                id: currentUserId,
+┊   ┊295┊                __typename: 'User',
+┊   ┊296┊              },
+┊   ┊297┊              {
+┊   ┊298┊                id: userId,
+┊   ┊299┊                __typename: 'User',
+┊   ┊300┊              }
+┊   ┊301┊            ],
+┊   ┊302┊            unreadMessages: 0,
+┊   ┊303┊            messages: [],
+┊   ┊304┊            isGroup: false,
+┊   ┊305┊          },
+┊   ┊306┊        },
 ┊249┊307┊        update: (store, { data: { addChat } }) => {
 ┊250┊308┊          // Read the data from our cache for this query.
 ┊251┊309┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
```
```diff
@@ -277,6 +335,26 @@
 ┊277┊335┊        userIds,
 ┊278┊336┊        groupName,
 ┊279┊337┊      }, {
+┊   ┊338┊        optimisticResponse: {
+┊   ┊339┊          __typename: 'Mutation',
+┊   ┊340┊          addGroup: {
+┊   ┊341┊            __typename: 'Chat',
+┊   ┊342┊            id: ChatsService.getRandomId(),
+┊   ┊343┊            name: groupName,
+┊   ┊344┊            picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊345┊            userIds: [currentUserId, userIds],
+┊   ┊346┊            allTimeMembers: [
+┊   ┊347┊              {
+┊   ┊348┊                id: currentUserId,
+┊   ┊349┊                __typename: 'User',
+┊   ┊350┊              },
+┊   ┊351┊              ...userIds.map(id => ({id, __typename: 'User'})),
+┊   ┊352┊            ],
+┊   ┊353┊            unreadMessages: 0,
+┊   ┊354┊            messages: [],
+┊   ┊355┊            isGroup: true,
+┊   ┊356┊          },
+┊   ┊357┊        },
 ┊280┊358┊        update: (store, { data: { addGroup } }) => {
 ┊281┊359┊          // Read the data from our cache for this query.
 ┊282┊360┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
```

[}]: #

When we open a specific chat we can also preload some data from our chats list cache while waiting for the server response. We will initially be able to show only the chat name, the last message or the last few messages and a few more informations instead of the whole content from the server, but that would be more than enough to entertain the user while waiting for the server response:

[{]: <helper> (diffStep "9.2" module="client")

#### Step 9.2: Get chat data from chats cache while waiting for server response

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,6 +1,6 @@
-┊1┊ ┊import {map} from 'rxjs/operators';
+┊ ┊1┊import {concat, map} from 'rxjs/operators';
 ┊2┊2┊import {Injectable} from '@angular/core';
-┊3┊ ┊import {Observable} from 'rxjs';
+┊ ┊3┊import {Observable, AsyncSubject, of} from 'rxjs';
 ┊4┊4┊import {QueryRef} from 'apollo-angular';
 ┊5┊5┊import * as moment from 'moment';
 ┊6┊6┊import {
```
```diff
@@ -31,6 +31,7 @@
 ┊31┊31┊  getChatsWq: QueryRef<GetChats.Query, GetChats.Variables>;
 ┊32┊32┊  chats$: Observable<GetChats.Chats[]>;
 ┊33┊33┊  chats: GetChats.Chats[];
+┊  ┊34┊  getChatWqSubject: AsyncSubject<QueryRef<GetChat.Query>>;
 ┊34┊35┊
 ┊35┊36┊  constructor(
 ┊36┊37┊    private getChatsGQL: GetChatsGQL,
```
```diff
@@ -61,15 +62,34 @@
 ┊61┊62┊  }
 ┊62┊63┊
 ┊63┊64┊  getChat(chatId: string) {
+┊  ┊65┊    const _chat = this.chats && this.chats.find(chat => chat.id === chatId) || {
+┊  ┊66┊      id: chatId,
+┊  ┊67┊      name: '',
+┊  ┊68┊      picture: null,
+┊  ┊69┊      allTimeMembers: [],
+┊  ┊70┊      unreadMessages: 0,
+┊  ┊71┊      isGroup: false,
+┊  ┊72┊      messages: [],
+┊  ┊73┊    };
+┊  ┊74┊    const chat$FromCache = of<GetChat.Chat>(_chat);
+┊  ┊75┊
 ┊64┊76┊    const query = this.getChatGQL.watch({
 ┊65┊77┊      chatId: chatId,
 ┊66┊78┊    });
 ┊67┊79┊
-┊68┊  ┊    const chat$ = query.valueChanges.pipe(
-┊69┊  ┊      map((result) => result.data.chat)
+┊  ┊80┊    const chat$ = chat$FromCache.pipe(
+┊  ┊81┊      concat(
+┊  ┊82┊        query.valueChanges.pipe(
+┊  ┊83┊          map((result) => result.data.chat)
+┊  ┊84┊        )
+┊  ┊85┊      )
 ┊70┊86┊    );
 ┊71┊87┊
-┊72┊  ┊    return {query, chat$};
+┊  ┊88┊    this.getChatWqSubject = new AsyncSubject();
+┊  ┊89┊    this.getChatWqSubject.next(query);
+┊  ┊90┊    this.getChatWqSubject.complete();
+┊  ┊91┊
+┊  ┊92┊    return {query$: this.getChatWqSubject.asObservable(), chat$};
 ┊73┊93┊  }
 ┊74┊94┊
 ┊75┊95┊  addMessage(chatId: string, content: string) {
```

[}]: #

Now let's deal with the most difficult part, chats creation.

We cannot predict the `id` of the new chat and so we cannot navigate to the chat page because it contains the chat id in the url. We could simply navigate to the "optimistic" id, but then the user wouldn't be able to reach that url if he refreshes the page or bookmarks it. That's a problem we care about.

How to solve it? We're going to create a landing page and we will later override the url once we get the response from the server!

[{]: <helper> (diffStep "9.3" module="client")

#### Step 9.3: Landing page for new chats/groups

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -129,7 +129,10 @@
 ┊129┊129┊        },
 ┊130┊130┊        {
 ┊131┊131┊          provide: ActivatedRoute,
-┊132┊   ┊          useValue: { params: of({ id: chat.id }) },
+┊   ┊132┊          useValue: {
+┊   ┊133┊            params: of({ id: chat.id }),
+┊   ┊134┊            queryParams: of({}),
+┊   ┊135┊          },
 ┊133┊136┊        },
 ┊134┊137┊      ],
 ┊135┊138┊      schemas: [NO_ERRORS_SCHEMA],
```

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -1,7 +1,9 @@
 ┊1┊1┊import {Component, OnInit} from '@angular/core';
 ┊2┊2┊import {ActivatedRoute, Router} from '@angular/router';
 ┊3┊3┊import {ChatsService} from '../../../services/chats.service';
-┊4┊ ┊import {GetChat} from '../../../../graphql';
+┊ ┊4┊import {AddChat, AddGroup, GetChat} from '../../../../graphql';
+┊ ┊5┊import {combineLatest} from 'rxjs';
+┊ ┊6┊import {Location} from '@angular/common';
 ┊5┊7┊
 ┊6┊8┊@Component({
 ┊7┊9┊  template: `
```
```diff
@@ -27,21 +29,40 @@
 ┊27┊29┊  messages: GetChat.Messages[];
 ┊28┊30┊  name: string;
 ┊29┊31┊  isGroup: boolean;
+┊  ┊32┊  optimisticUI: boolean;
 ┊30┊33┊
 ┊31┊34┊  constructor(private route: ActivatedRoute,
 ┊32┊35┊              private router: Router,
+┊  ┊36┊              private location: Location,
 ┊33┊37┊              private chatsService: ChatsService) {
 ┊34┊38┊  }
 ┊35┊39┊
 ┊36┊40┊  ngOnInit() {
-┊37┊  ┊    this.route.params.subscribe(({id: chatId}) => {
-┊38┊  ┊      this.chatId = chatId;
-┊39┊  ┊      this.chatsService.getChat(chatId).chat$.subscribe(chat => {
-┊40┊  ┊        this.messages = chat.messages;
-┊41┊  ┊        this.name = chat.name;
-┊42┊  ┊        this.isGroup = chat.isGroup;
+┊  ┊41┊    combineLatest(this.route.params, this.route.queryParams,
+┊  ┊42┊      (params: { id: string }, queryParams: { oui?: boolean }) => ({params, queryParams}))
+┊  ┊43┊      .subscribe(({params: {id: chatId}, queryParams: {oui}}) => {
+┊  ┊44┊        this.chatId = chatId;
+┊  ┊45┊
+┊  ┊46┊        this.optimisticUI = oui;
+┊  ┊47┊
+┊  ┊48┊        if (this.optimisticUI) {
+┊  ┊49┊          // We are using fake IDs generated by the Optimistic UI
+┊  ┊50┊          this.chatsService.addChat$.subscribe(({data}) => {
+┊  ┊51┊            this.chatId = (<AddChat.Mutation>data).addChat ? (<AddChat.Mutation>data).addChat.id : (<AddGroup.Mutation>data).addGroup.id;
+┊  ┊52┊            console.log(`Switching from the Optimistic UI id ${chatId} to ${this.chatId}`);
+┊  ┊53┊            // Rewrite the URL
+┊  ┊54┊            this.location.go(`chat/${this.chatId}`);
+┊  ┊55┊            // Optimistic UI no more
+┊  ┊56┊            this.optimisticUI = false;
+┊  ┊57┊          });
+┊  ┊58┊        }
+┊  ┊59┊
+┊  ┊60┊        this.chatsService.getChat(chatId, this.optimisticUI).chat$.subscribe(chat => {
+┊  ┊61┊          this.messages = chat.messages;
+┊  ┊62┊          this.name = chat.name;
+┊  ┊63┊          this.isGroup = chat.isGroup;
+┊  ┊64┊        });
 ┊43┊65┊      });
-┊44┊  ┊    });
 ┊45┊66┊  }
 ┊46┊67┊
 ┊47┊68┊  goToChats() {
```

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.ts
```diff
@@ -47,9 +47,10 @@
 ┊47┊47┊      // Chat is already listed for the current user
 ┊48┊48┊      this.router.navigate(['/chat', chatId]);
 ┊49┊49┊    } else {
-┊50┊  ┊      this.chatsService.addChat(userId, this.users).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
-┊51┊  ┊        this.router.navigate(['/chat', id]);
-┊52┊  ┊      });
+┊  ┊50┊      // Generate id for Optimistic UI
+┊  ┊51┊      const ouiId = ChatsService.getRandomId();
+┊  ┊52┊      this.chatsService.addChat(userId, this.users, ouiId).subscribe();
+┊  ┊53┊      this.router.navigate(['/chat', ouiId], {queryParams: {oui: true}, skipLocationChange: true});
 ┊53┊54┊    }
 ┊54┊55┊  }
 ┊55┊56┊}
```

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-group&#x2F;new-group.component.ts
```diff
@@ -52,9 +52,9 @@
 ┊52┊52┊
 ┊53┊53┊  addGroup(groupName: string) {
 ┊54┊54┊    if (groupName && this.userIds.length) {
-┊55┊  ┊      this.chatsService.addGroup(this.userIds, groupName).subscribe(({data: {addGroup: {id}}}: { data: AddGroup.Mutation }) => {
-┊56┊  ┊        this.router.navigate(['/chat', id]);
-┊57┊  ┊      });
+┊  ┊55┊      const ouiId = ChatsService.getRandomId();
+┊  ┊56┊      this.chatsService.addGroup(this.userIds, groupName, ouiId).subscribe();
+┊  ┊57┊      this.router.navigate(['/chat', ouiId], {queryParams: {oui: true}, skipLocationChange: true});
 ┊58┊58┊    }
 ┊59┊59┊  }
 ┊60┊60┊}
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,4 +1,4 @@
-┊1┊ ┊import {concat, map} from 'rxjs/operators';
+┊ ┊1┊import {concat, map, share, switchMap} from 'rxjs/operators';
 ┊2┊2┊import {Injectable} from '@angular/core';
 ┊3┊3┊import {Observable, AsyncSubject, of} from 'rxjs';
 ┊4┊4┊import {QueryRef} from 'apollo-angular';
```
```diff
@@ -19,8 +19,11 @@
 ┊19┊19┊  RemoveMessages,
 ┊20┊20┊  RemoveAllMessages,
 ┊21┊21┊  GetUsers,
+┊  ┊22┊  AddChat,
+┊  ┊23┊  AddGroup,
 ┊22┊24┊} from '../../graphql';
 ┊23┊25┊import { DataProxy } from 'apollo-cache';
+┊  ┊26┊import { FetchResult } from 'apollo-link';
 ┊24┊27┊
 ┊25┊28┊const currentUserId = '1';
 ┊26┊29┊const currentUserName = 'Ethan Gonzalez';
```
```diff
@@ -32,6 +35,7 @@
 ┊32┊35┊  chats$: Observable<GetChats.Chats[]>;
 ┊33┊36┊  chats: GetChats.Chats[];
 ┊34┊37┊  getChatWqSubject: AsyncSubject<QueryRef<GetChat.Query>>;
+┊  ┊38┊  addChat$: Observable<FetchResult<AddChat.Mutation | AddGroup.Mutation>>;
 ┊35┊39┊
 ┊36┊40┊  constructor(
 ┊37┊41┊    private getChatsGQL: GetChatsGQL,
```
```diff
@@ -61,7 +65,7 @@
 ┊61┊65┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊62┊66┊  }
 ┊63┊67┊
-┊64┊  ┊  getChat(chatId: string) {
+┊  ┊68┊  getChat(chatId: string, oui?: boolean) {
 ┊65┊69┊    const _chat = this.chats && this.chats.find(chat => chat.id === chatId) || {
 ┊66┊70┊      id: chatId,
 ┊67┊71┊      name: '',
```
```diff
@@ -73,21 +77,44 @@
 ┊ 73┊ 77┊    };
 ┊ 74┊ 78┊    const chat$FromCache = of<GetChat.Chat>(_chat);
 ┊ 75┊ 79┊
-┊ 76┊   ┊    const query = this.getChatGQL.watch({
-┊ 77┊   ┊      chatId: chatId,
-┊ 78┊   ┊    });
-┊ 79┊   ┊
-┊ 80┊   ┊    const chat$ = chat$FromCache.pipe(
-┊ 81┊   ┊      concat(
-┊ 82┊   ┊        query.valueChanges.pipe(
-┊ 83┊   ┊          map((result) => result.data.chat)
-┊ 84┊   ┊        )
-┊ 85┊   ┊      )
-┊ 86┊   ┊    );
+┊   ┊ 80┊    const getApolloWatchQuery = (id: string) => {
+┊   ┊ 81┊      return this.getChatGQL.watch({
+┊   ┊ 82┊        chatId: id,
+┊   ┊ 83┊      });
+┊   ┊ 84┊    };
 ┊ 87┊ 85┊
+┊   ┊ 86┊    let chat$: Observable<GetChat.Chat>;
 ┊ 88┊ 87┊    this.getChatWqSubject = new AsyncSubject();
-┊ 89┊   ┊    this.getChatWqSubject.next(query);
-┊ 90┊   ┊    this.getChatWqSubject.complete();
+┊   ┊ 88┊
+┊   ┊ 89┊    if (oui) {
+┊   ┊ 90┊      chat$ = chat$FromCache.pipe(
+┊   ┊ 91┊        concat(this.addChat$.pipe(
+┊   ┊ 92┊          switchMap(({ data }) => {
+┊   ┊ 93┊            const id = (<AddChat.Mutation>data).addChat ? (<AddChat.Mutation>data).addChat.id : (<AddGroup.Mutation>data).addGroup.id;
+┊   ┊ 94┊            const query = getApolloWatchQuery(id);
+┊   ┊ 95┊
+┊   ┊ 96┊            this.getChatWqSubject.next(query);
+┊   ┊ 97┊            this.getChatWqSubject.complete();
+┊   ┊ 98┊
+┊   ┊ 99┊            return query.valueChanges.pipe(
+┊   ┊100┊              map((result) => result.data.chat)
+┊   ┊101┊            );
+┊   ┊102┊          }))
+┊   ┊103┊        ));
+┊   ┊104┊    } else {
+┊   ┊105┊      const query = getApolloWatchQuery(chatId);
+┊   ┊106┊
+┊   ┊107┊      this.getChatWqSubject.next(query);
+┊   ┊108┊      this.getChatWqSubject.complete();
+┊   ┊109┊
+┊   ┊110┊      chat$ = chat$FromCache.pipe(
+┊   ┊111┊        concat(
+┊   ┊112┊          query.valueChanges.pipe(
+┊   ┊113┊            map((result) => result.data.chat)
+┊   ┊114┊          )
+┊   ┊115┊        )
+┊   ┊116┊      );
+┊   ┊117┊    }
 ┊ 91┊118┊
 ┊ 92┊119┊    return {query$: this.getChatWqSubject.asObservable(), chat$};
 ┊ 93┊120┊  }
```
```diff
@@ -297,8 +324,8 @@
 ┊297┊324┊    return _chat ? _chat.id : false;
 ┊298┊325┊  }
 ┊299┊326┊
-┊300┊   ┊  addChat(userId: string, users: GetUsers.Users[]) {
-┊301┊   ┊    return this.addChatGQL.mutate(
+┊   ┊327┊  addChat(userId: string, users: GetUsers.Users[], ouiId: string) {
+┊   ┊328┊    this.addChat$ = this.addChatGQL.mutate(
 ┊302┊329┊      {
 ┊303┊330┊        userId,
 ┊304┊331┊      }, {
```
```diff
@@ -306,7 +333,7 @@
 ┊306┊333┊          __typename: 'Mutation',
 ┊307┊334┊          addChat: {
 ┊308┊335┊            __typename: 'Chat',
-┊309┊   ┊            id: ChatsService.getRandomId(),
+┊   ┊336┊            id: ouiId,
 ┊310┊337┊            name: users.find(user => user.id === userId).name,
 ┊311┊338┊            picture: users.find(user => user.id === userId).picture,
 ┊312┊339┊            allTimeMembers: [
```
```diff
@@ -346,11 +373,12 @@
 ┊346┊373┊          });
 ┊347┊374┊        },
 ┊348┊375┊      }
-┊349┊   ┊    );
+┊   ┊376┊    ).pipe(share());
+┊   ┊377┊    return this.addChat$;
 ┊350┊378┊  }
 ┊351┊379┊
-┊352┊   ┊  addGroup(userIds: string[], groupName: string) {
-┊353┊   ┊    return this.addGroupGQL.mutate(
+┊   ┊380┊  addGroup(userIds: string[], groupName: string, ouiId: string) {
+┊   ┊381┊    this.addChat$ = this.addGroupGQL.mutate(
 ┊354┊382┊      {
 ┊355┊383┊        userIds,
 ┊356┊384┊        groupName,
```
```diff
@@ -359,7 +387,7 @@
 ┊359┊387┊          __typename: 'Mutation',
 ┊360┊388┊          addGroup: {
 ┊361┊389┊            __typename: 'Chat',
-┊362┊   ┊            id: ChatsService.getRandomId(),
+┊   ┊390┊            id: ouiId,
 ┊363┊391┊            name: groupName,
 ┊364┊392┊            picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
 ┊365┊393┊            userIds: [currentUserId, userIds],
```
```diff
@@ -397,6 +425,7 @@
 ┊397┊425┊          });
 ┊398┊426┊        },
 ┊399┊427┊      }
-┊400┊   ┊    );
+┊   ┊428┊    ).pipe(share());
+┊   ┊429┊    return this.addChat$;
 ┊401┊430┊  }
 ┊402┊431┊}
```

[}]: #

Poof, now our Whatsapp clone feels no more like a clone it has the same native feel.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step8.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step10.md) |
|:--------------------------------|--------------------------------:|

[}]: #
