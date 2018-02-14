# Step 13: Zero latency on slow 3g networks

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

[{]: <helper> (diffStep "9.1" files="^\(?!package.json$\).*" module="client")

#### Step 9.1: Optimistic updates

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.ts
```diff
@@ -51,7 +51,7 @@
 ┊51┊51┊      // Chat is already listed for the current user
 ┊52┊52┊      this.router.navigate(['/chat', chatId]);
 ┊53┊53┊    } else {
-┊54┊  ┊      this.chatsService.addChat(recipientId).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
+┊  ┊54┊      this.chatsService.addChat(recipientId, this.users).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
 ┊55┊55┊        this.router.navigate(['/chat', id]);
 ┊56┊56┊      });
 ┊57┊57┊    }
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
@@ -70,6 +77,24 @@
 ┊ 70┊ 77┊      chatId,
 ┊ 71┊ 78┊      content,
 ┊ 72┊ 79┊    }, {
+┊   ┊ 80┊      optimisticResponse: {
+┊   ┊ 81┊        __typename: 'Mutation',
+┊   ┊ 82┊        addMessage: {
+┊   ┊ 83┊          id: ChatsService.getRandomId(),
+┊   ┊ 84┊          __typename: 'Message',
+┊   ┊ 85┊          senderId: currentUserId,
+┊   ┊ 86┊          sender: {
+┊   ┊ 87┊            id: currentUserId,
+┊   ┊ 88┊            __typename: 'User',
+┊   ┊ 89┊            name: currentUserName,
+┊   ┊ 90┊          },
+┊   ┊ 91┊          content,
+┊   ┊ 92┊          createdAt: moment().unix(),
+┊   ┊ 93┊          type: 0,
+┊   ┊ 94┊          recipients: [],
+┊   ┊ 95┊          ownership: true,
+┊   ┊ 96┊        },
+┊   ┊ 97┊      },
 ┊ 73┊ 98┊      update: (store, { data: { addMessage } }: {data: AddMessage.Mutation}) => {
 ┊ 74┊ 99┊        // Update the messages cache
 ┊ 75┊100┊        {
```
```diff
@@ -121,6 +146,10 @@
 ┊121┊146┊      {
 ┊122┊147┊        chatId,
 ┊123┊148┊      }, {
+┊   ┊149┊        optimisticResponse: {
+┊   ┊150┊          __typename: 'Mutation',
+┊   ┊151┊          removeChat: chatId,
+┊   ┊152┊        },
 ┊124┊153┊        update: (store, { data: { removeChat } }) => {
 ┊125┊154┊          // Read the data from our cache for this query.
 ┊126┊155┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
```
```diff
@@ -154,6 +183,10 @@
 ┊154┊183┊    let ids: string[] = [];
 ┊155┊184┊
 ┊156┊185┊    const options = {
+┊   ┊186┊      optimisticResponse: () => ({
+┊   ┊187┊        __typename: 'Mutation',
+┊   ┊188┊        removeMessages: ids,
+┊   ┊189┊      }),
 ┊157┊190┊      update: (store: DataProxy, { data: { removeMessages } }: {data: RemoveMessages.Mutation | RemoveAllMessages.Mutation}) => {
 ┊158┊191┊        // Update the messages cache
 ┊159┊192┊        {
```
```diff
@@ -242,11 +275,33 @@
 ┊242┊275┊    return _chat ? _chat.id : false;
 ┊243┊276┊  }
 ┊244┊277┊
-┊245┊   ┊  addChat(recipientId: string) {
+┊   ┊278┊  addChat(recipientId: string, users: GetUsers.Users[]) {
 ┊246┊279┊    return this.addChatGQL.mutate(
 ┊247┊280┊      {
 ┊248┊281┊        recipientId,
 ┊249┊282┊      }, {
+┊   ┊283┊        optimisticResponse: {
+┊   ┊284┊          __typename: 'Mutation',
+┊   ┊285┊          addChat: {
+┊   ┊286┊            id: ChatsService.getRandomId(),
+┊   ┊287┊            __typename: 'Chat',
+┊   ┊288┊            name: users.find(user => user.id === recipientId).name,
+┊   ┊289┊            picture: users.find(user => user.id === recipientId).picture,
+┊   ┊290┊            allTimeMembers: [
+┊   ┊291┊              {
+┊   ┊292┊                id: currentUserId,
+┊   ┊293┊                __typename: 'User',
+┊   ┊294┊              },
+┊   ┊295┊              {
+┊   ┊296┊                id: recipientId,
+┊   ┊297┊                __typename: 'User',
+┊   ┊298┊              }
+┊   ┊299┊            ],
+┊   ┊300┊            unreadMessages: 0,
+┊   ┊301┊            messages: [],
+┊   ┊302┊            isGroup: false,
+┊   ┊303┊          },
+┊   ┊304┊        },
 ┊250┊305┊        update: (store, { data: { addChat } }) => {
 ┊251┊306┊          // Read the data from our cache for this query.
 ┊252┊307┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
```
```diff
@@ -278,6 +333,26 @@
 ┊278┊333┊        recipientIds,
 ┊279┊334┊        groupName,
 ┊280┊335┊      }, {
+┊   ┊336┊        optimisticResponse: {
+┊   ┊337┊          __typename: 'Mutation',
+┊   ┊338┊          addGroup: {
+┊   ┊339┊            id: ChatsService.getRandomId(),
+┊   ┊340┊            __typename: 'Chat',
+┊   ┊341┊            name: groupName,
+┊   ┊342┊            picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊343┊            userIds: [currentUserId, recipientIds],
+┊   ┊344┊            allTimeMembers: [
+┊   ┊345┊              {
+┊   ┊346┊                id: currentUserId,
+┊   ┊347┊                __typename: 'User',
+┊   ┊348┊              },
+┊   ┊349┊              ...recipientIds.map(id => ({id, __typename: 'User'})),
+┊   ┊350┊            ],
+┊   ┊351┊            unreadMessages: 0,
+┊   ┊352┊            messages: [],
+┊   ┊353┊            isGroup: true,
+┊   ┊354┊          },
+┊   ┊355┊        },
 ┊281┊356┊        update: (store, { data: { addGroup } }) => {
 ┊282┊357┊          // Read the data from our cache for this query.
 ┊283┊358┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
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
@@ -2,6 +2,8 @@
 ┊2┊2┊import {ActivatedRoute, Router} from '@angular/router';
 ┊3┊3┊import {ChatsService} from '../../../services/chats.service';
 ┊4┊4┊import {GetChat} from '../../../../graphql';
+┊ ┊5┊import {combineLatest} from 'rxjs';
+┊ ┊6┊import {Location} from '@angular/common';
 ┊5┊7┊
 ┊6┊8┊@Component({
 ┊7┊9┊  template: `
```
```diff
@@ -26,21 +28,40 @@
 ┊26┊28┊  messages: GetChat.Messages[];
 ┊27┊29┊  name: string;
 ┊28┊30┊  isGroup: boolean;
+┊  ┊31┊  optimisticUI: boolean;
 ┊29┊32┊
 ┊30┊33┊  constructor(private route: ActivatedRoute,
 ┊31┊34┊              private router: Router,
+┊  ┊35┊              private location: Location,
 ┊32┊36┊              private chatsService: ChatsService) {
 ┊33┊37┊  }
 ┊34┊38┊
 ┊35┊39┊  ngOnInit() {
-┊36┊  ┊    this.route.params.subscribe(({id: chatId}) => {
-┊37┊  ┊      this.chatId = chatId;
-┊38┊  ┊      this.chatsService.getChat(chatId).chat$.subscribe(chat => {
-┊39┊  ┊        this.messages = chat.messages;
-┊40┊  ┊        this.name = chat.name;
-┊41┊  ┊        this.isGroup = chat.isGroup;
+┊  ┊40┊    combineLatest(this.route.params, this.route.queryParams,
+┊  ┊41┊      (params: { id: string }, queryParams: { oui?: boolean }) => ({params, queryParams}))
+┊  ┊42┊      .subscribe(({params: {id: chatId}, queryParams: {oui}}) => {
+┊  ┊43┊        this.chatId = chatId;
+┊  ┊44┊
+┊  ┊45┊        this.optimisticUI = oui;
+┊  ┊46┊
+┊  ┊47┊        if (this.optimisticUI) {
+┊  ┊48┊          // We are using fake IDs generated by the Optimistic UI
+┊  ┊49┊          this.chatsService.addChat$.subscribe(({data: {addChat, addGroup}}) => {
+┊  ┊50┊            this.chatId = addChat ? addChat.id : addGroup.id;
+┊  ┊51┊            console.log(`Switching from the Optimistic UI id ${chatId} to ${this.chatId}`);
+┊  ┊52┊            // Rewrite the URL
+┊  ┊53┊            this.location.go(`chat/${this.chatId}`);
+┊  ┊54┊            // Optimistic UI no more
+┊  ┊55┊            this.optimisticUI = false;
+┊  ┊56┊          });
+┊  ┊57┊        }
+┊  ┊58┊
+┊  ┊59┊        this.chatsService.getChat(chatId, this.optimisticUI).chat$.subscribe(chat => {
+┊  ┊60┊          this.messages = chat.messages;
+┊  ┊61┊          this.name = chat.name;
+┊  ┊62┊          this.isGroup = chat.isGroup;
+┊  ┊63┊        });
 ┊42┊64┊      });
-┊43┊  ┊    });
 ┊44┊65┊  }
 ┊45┊66┊
 ┊46┊67┊  goToChats() {
```

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.ts
```diff
@@ -51,9 +51,10 @@
 ┊51┊51┊      // Chat is already listed for the current user
 ┊52┊52┊      this.router.navigate(['/chat', chatId]);
 ┊53┊53┊    } else {
-┊54┊  ┊      this.chatsService.addChat(recipientId, this.users).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
-┊55┊  ┊        this.router.navigate(['/chat', id]);
-┊56┊  ┊      });
+┊  ┊54┊      // Generate id for Optimistic UI
+┊  ┊55┊      const ouiId = ChatsService.getRandomId();
+┊  ┊56┊      this.chatsService.addChat(recipientId, this.users, ouiId).subscribe();
+┊  ┊57┊      this.router.navigate(['/chat', ouiId], {queryParams: {oui: true}, skipLocationChange: true});
 ┊57┊58┊    }
 ┊58┊59┊  }
 ┊59┊60┊}
```

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-group&#x2F;new-group.component.ts
```diff
@@ -52,9 +52,9 @@
 ┊52┊52┊
 ┊53┊53┊  addGroup(groupName: string) {
 ┊54┊54┊    if (groupName && this.recipientIds.length) {
-┊55┊  ┊      this.chatsService.addGroup(this.recipientIds, groupName).subscribe(({data: {addGroup: {id}}}: { data: AddGroup.Mutation }) => {
-┊56┊  ┊        this.router.navigate(['/chat', id]);
-┊57┊  ┊      });
+┊  ┊55┊      const ouiId = ChatsService.getRandomId();
+┊  ┊56┊      this.chatsService.addGroup(this.recipientIds, groupName, ouiId).subscribe();
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
@@ -73,21 +77,43 @@
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
+┊   ┊ 92┊          switchMap(({ data: { addChat, addGroup } }) => {
+┊   ┊ 93┊            const query = getApolloWatchQuery(addChat ? addChat.id : addGroup.id);
+┊   ┊ 94┊
+┊   ┊ 95┊            this.getChatWqSubject.next(query);
+┊   ┊ 96┊            this.getChatWqSubject.complete();
+┊   ┊ 97┊
+┊   ┊ 98┊            return query.valueChanges.pipe(
+┊   ┊ 99┊              map((result) => result.data.chat)
+┊   ┊100┊            );
+┊   ┊101┊          }))
+┊   ┊102┊        ));
+┊   ┊103┊    } else {
+┊   ┊104┊      const query = getApolloWatchQuery(chatId);
+┊   ┊105┊
+┊   ┊106┊      this.getChatWqSubject.next(query);
+┊   ┊107┊      this.getChatWqSubject.complete();
+┊   ┊108┊
+┊   ┊109┊      chat$ = chat$FromCache.pipe(
+┊   ┊110┊        concat(
+┊   ┊111┊          query.valueChanges.pipe(
+┊   ┊112┊            map((result) => result.data.chat)
+┊   ┊113┊          )
+┊   ┊114┊        )
+┊   ┊115┊      );
+┊   ┊116┊    }
 ┊ 91┊117┊
 ┊ 92┊118┊    return {query$: this.getChatWqSubject.asObservable(), chat$};
 ┊ 93┊119┊  }
```
```diff
@@ -295,15 +321,15 @@
 ┊295┊321┊    return _chat ? _chat.id : false;
 ┊296┊322┊  }
 ┊297┊323┊
-┊298┊   ┊  addChat(recipientId: string, users: GetUsers.Users[]) {
-┊299┊   ┊    return this.addChatGQL.mutate(
+┊   ┊324┊  addChat(recipientId: string, users: GetUsers.Users[], ouiId: string) {
+┊   ┊325┊    this.addChat$ = this.addChatGQL.mutate(
 ┊300┊326┊      {
 ┊301┊327┊        recipientId,
 ┊302┊328┊      }, {
 ┊303┊329┊        optimisticResponse: {
 ┊304┊330┊          __typename: 'Mutation',
 ┊305┊331┊          addChat: {
-┊306┊   ┊            id: ChatsService.getRandomId(),
+┊   ┊332┊            id: ouiId,
 ┊307┊333┊            __typename: 'Chat',
 ┊308┊334┊            name: users.find(user => user.id === recipientId).name,
 ┊309┊335┊            picture: users.find(user => user.id === recipientId).picture,
```
```diff
@@ -344,11 +370,12 @@
 ┊344┊370┊          });
 ┊345┊371┊        },
 ┊346┊372┊      }
-┊347┊   ┊    );
+┊   ┊373┊    ).pipe(share());
+┊   ┊374┊    return this.addChat$;
 ┊348┊375┊  }
 ┊349┊376┊
-┊350┊   ┊  addGroup(recipientIds: string[], groupName: string) {
-┊351┊   ┊    return this.addGroupGQL.mutate(
+┊   ┊377┊  addGroup(recipientIds: string[], groupName: string, ouiId: string) {
+┊   ┊378┊    this.addChat$ = this.addGroupGQL.mutate(
 ┊352┊379┊      {
 ┊353┊380┊        recipientIds,
 ┊354┊381┊        groupName,
```
```diff
@@ -356,7 +383,7 @@
 ┊356┊383┊        optimisticResponse: {
 ┊357┊384┊          __typename: 'Mutation',
 ┊358┊385┊          addGroup: {
-┊359┊   ┊            id: ChatsService.getRandomId(),
+┊   ┊386┊            id: ouiId,
 ┊360┊387┊            __typename: 'Chat',
 ┊361┊388┊            name: groupName,
 ┊362┊389┊            picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
```
```diff
@@ -395,6 +422,7 @@
 ┊395┊422┊          });
 ┊396┊423┊        },
 ┊397┊424┊      }
-┊398┊   ┊    );
+┊   ┊425┊    ).pipe(share());
+┊   ┊426┊    return this.addChat$;
 ┊399┊427┊  }
 ┊400┊428┊}
```

[}]: #

Poof, now our Whatsapp clone feels no more like a clone it has the same native feel.

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step12.md) | [Next Step >](step14.md) |
|:--------------------------------|--------------------------------:|

[}]: #
