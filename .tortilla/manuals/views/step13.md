# Step 13: Zero latency on slow 3g networks

[//]: # (head-end)


## Client

Now let's start our client in production mode:

    $ ng serve --prod

Now open the Chrome Developers Tools and, in the Network tab, select 'Slow 3G Network' and 'Disable cache'.
Now refresh the page and look at the DOMContentLoaded time and at the transferred size. You'll notice that our bundle size is quite small and so the loads time.
Now let's click on a specific chat. It will take some time to load the data. Now let's add a new message. Once again it will take some time to load the data. We could also create a new chat and the result would be the same. The whole app doesn't
feel as snappier as the real Whatsapp on a slow 3G Network.
"That's normal, it's a web application with a remote db while Whatsapp is a native app with a local database..."
That's just an excuse, because we can do as good as Whatsapp thanks to Apollo!

Let's install `moment`, we will soon need it:

    $ npm install moment

Let's start by making our UI optimistic. We can predict most of the response we will get from our server, except for a few things like `id` of newly created messages. But since we don't really need that id, we can simply generate a fake one
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
@@ -14,8 +14,10 @@
 ┊14┊14┊import {Observable} from 'rxjs/Observable';
 ┊15┊15┊import {addChatMutation} from '../../graphql/addChat.mutation';
 ┊16┊16┊import {addGroupMutation} from '../../graphql/addGroup.mutation';
+┊  ┊17┊import * as moment from 'moment';
 ┊17┊18┊
 ┊18┊19┊const currentUserId = '1';
+┊  ┊20┊const currentUserName = 'Ethan Gonzalez';
 ┊19┊21┊
 ┊20┊22┊@Injectable()
 ┊21┊23┊export class ChatsService {
```
```diff
@@ -37,6 +39,10 @@
 ┊37┊39┊    this.chats$.subscribe(chats => this.chats = chats);
 ┊38┊40┊  }
 ┊39┊41┊
+┊  ┊42┊  static getRandomId() {
+┊  ┊43┊    return String(Math.round(Math.random() * 1000000000000));
+┊  ┊44┊  }
+┊  ┊45┊
 ┊40┊46┊  getChats() {
 ┊41┊47┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊42┊48┊  }
```
```diff
@@ -63,6 +69,24 @@
 ┊63┊69┊        chatId,
 ┊64┊70┊        content,
 ┊65┊71┊      },
+┊  ┊72┊      optimisticResponse: {
+┊  ┊73┊        __typename: 'Mutation',
+┊  ┊74┊        addMessage: {
+┊  ┊75┊          id: ChatsService.getRandomId(),
+┊  ┊76┊          __typename: 'Message',
+┊  ┊77┊          senderId: currentUserId,
+┊  ┊78┊          sender: {
+┊  ┊79┊            id: currentUserId,
+┊  ┊80┊            __typename: 'User',
+┊  ┊81┊            name: currentUserName,
+┊  ┊82┊          },
+┊  ┊83┊          content,
+┊  ┊84┊          createdAt: moment().unix(),
+┊  ┊85┊          type: 0,
+┊  ┊86┊          recipients: [],
+┊  ┊87┊          ownership: true,
+┊  ┊88┊        },
+┊  ┊89┊      },
 ┊66┊90┊      update: (store, { data: { addMessage } }: {data: AddMessage.Mutation}) => {
 ┊67┊91┊        // Update the messages cache
 ┊68┊92┊        {
```
```diff
@@ -109,6 +133,10 @@
 ┊109┊133┊      variables: <RemoveChat.Variables>{
 ┊110┊134┊        chatId,
 ┊111┊135┊      },
+┊   ┊136┊      optimisticResponse: {
+┊   ┊137┊        __typename: 'Mutation',
+┊   ┊138┊        removeChat: chatId,
+┊   ┊139┊      },
 ┊112┊140┊      update: (store, { data: { removeChat } }) => {
 ┊113┊141┊        // Read the data from our cache for this query.
 ┊114┊142┊        const {chats}: GetChats.Query = store.readQuery({
```
```diff
@@ -155,6 +183,10 @@
 ┊155┊183┊    return this.apollo.mutate(<MutationOptions>{
 ┊156┊184┊      mutation,
 ┊157┊185┊      variables,
+┊   ┊186┊      optimisticResponse: {
+┊   ┊187┊        __typename: 'Mutation',
+┊   ┊188┊        removeMessages: ids,
+┊   ┊189┊      },
 ┊158┊190┊      update: (store, { data: { removeMessages } }: {data: RemoveMessages.Mutation | RemoveAllMessages.Mutation}) => {
 ┊159┊191┊        // Update the messages cache
 ┊160┊192┊        {
```
```diff
@@ -223,12 +255,34 @@
 ┊223┊255┊    return _chat ? _chat.id : false;
 ┊224┊256┊  }
 ┊225┊257┊
-┊226┊   ┊  addChat(recipientId: string) {
+┊   ┊258┊  addChat(recipientId: string, users: GetUsers.Users[]) {
 ┊227┊259┊    return this.apollo.mutate({
 ┊228┊260┊      mutation: addChatMutation,
 ┊229┊261┊      variables: <AddChat.Variables>{
 ┊230┊262┊        recipientId,
 ┊231┊263┊      },
+┊   ┊264┊      optimisticResponse: {
+┊   ┊265┊        __typename: 'Mutation',
+┊   ┊266┊        addChat: {
+┊   ┊267┊          id: ChatsService.getRandomId(),
+┊   ┊268┊          __typename: 'Chat',
+┊   ┊269┊          name: users.find(user => user.id === recipientId).name,
+┊   ┊270┊          picture: users.find(user => user.id === recipientId).picture,
+┊   ┊271┊          allTimeMembers: [
+┊   ┊272┊            {
+┊   ┊273┊              id: currentUserId,
+┊   ┊274┊              __typename: 'User',
+┊   ┊275┊            },
+┊   ┊276┊            {
+┊   ┊277┊              id: recipientId,
+┊   ┊278┊              __typename: 'User',
+┊   ┊279┊            }
+┊   ┊280┊          ],
+┊   ┊281┊          unreadMessages: 0,
+┊   ┊282┊          messages: [],
+┊   ┊283┊          isGroup: false,
+┊   ┊284┊        },
+┊   ┊285┊      },
 ┊232┊286┊      update: (store, { data: { addChat } }) => {
 ┊233┊287┊        // Read the data from our cache for this query.
 ┊234┊288┊        const {chats}: GetChats.Query = store.readQuery({
```
```diff
@@ -260,6 +314,26 @@
 ┊260┊314┊        recipientIds,
 ┊261┊315┊        groupName,
 ┊262┊316┊      },
+┊   ┊317┊      optimisticResponse: {
+┊   ┊318┊        __typename: 'Mutation',
+┊   ┊319┊        addGroup: {
+┊   ┊320┊          id: ChatsService.getRandomId(),
+┊   ┊321┊          __typename: 'Chat',
+┊   ┊322┊          name: groupName,
+┊   ┊323┊          picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊324┊          userIds: [currentUserId, recipientIds],
+┊   ┊325┊          allTimeMembers: [
+┊   ┊326┊            {
+┊   ┊327┊              id: currentUserId,
+┊   ┊328┊              __typename: 'User',
+┊   ┊329┊            },
+┊   ┊330┊            ...recipientIds.map(id => ({id, __typename: 'User'})),
+┊   ┊331┊          ],
+┊   ┊332┊          unreadMessages: 0,
+┊   ┊333┊          messages: [],
+┊   ┊334┊          isGroup: true,
+┊   ┊335┊        },
+┊   ┊336┊      },
 ┊263┊337┊      update: (store, { data: { addGroup } }) => {
 ┊264┊338┊        // Read the data from our cache for this query.
 ┊265┊339┊        const {chats}: GetChats.Query = store.readQuery({
```

[}]: #

When we open a specific chat we can also preload some data from our chats list cache while waiting for the server response. We will initially be able to show only the chat name, the last message or the last few messages and a few more informations instead of the whole content from the server, but that would be more than enough to entertain the user while waiting for the server response:

[{]: <helper> (diffStep "9.2" module="client")

#### Step 9.2: Get chat data from chats cache while waiting for server response

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,5 +1,5 @@
 ┊1┊1┊import {ApolloQueryResult, MutationOptions, WatchQueryOptions} from 'apollo-client';
-┊2┊ ┊import {map} from 'rxjs/operators';
+┊ ┊2┊import {concat, map} from 'rxjs/operators';
 ┊3┊3┊import {Apollo, QueryRef} from 'apollo-angular';
 ┊4┊4┊import {Injectable} from '@angular/core';
 ┊5┊5┊import {getChatsQuery} from '../../graphql/getChats.query';
```
```diff
@@ -15,6 +15,8 @@
 ┊15┊15┊import {addChatMutation} from '../../graphql/addChat.mutation';
 ┊16┊16┊import {addGroupMutation} from '../../graphql/addGroup.mutation';
 ┊17┊17┊import * as moment from 'moment';
+┊  ┊18┊import {AsyncSubject} from 'rxjs/AsyncSubject';
+┊  ┊19┊import {of} from 'rxjs/observable/of';
 ┊18┊20┊
 ┊19┊21┊const currentUserId = '1';
 ┊20┊22┊const currentUserName = 'Ethan Gonzalez';
```
```diff
@@ -25,6 +27,7 @@
 ┊25┊27┊  getChatsWq: QueryRef<GetChats.Query>;
 ┊26┊28┊  chats$: Observable<GetChats.Chats[]>;
 ┊27┊29┊  chats: GetChats.Chats[];
+┊  ┊30┊  getChatWqSubject: AsyncSubject<QueryRef<GetChat.Query>>;
 ┊28┊31┊
 ┊29┊32┊  constructor(private apollo: Apollo) {
 ┊30┊33┊    this.getChatsWq = this.apollo.watchQuery<GetChats.Query>(<WatchQueryOptions>{
```
```diff
@@ -48,18 +51,34 @@
 ┊48┊51┊  }
 ┊49┊52┊
 ┊50┊53┊  getChat(chatId: string) {
+┊  ┊54┊    const _chat = this.chats && this.chats.find(chat => chat.id === chatId) || {
+┊  ┊55┊      id: chatId,
+┊  ┊56┊      name: '',
+┊  ┊57┊      picture: null,
+┊  ┊58┊      allTimeMembers: [],
+┊  ┊59┊      unreadMessages: 0,
+┊  ┊60┊      isGroup: false,
+┊  ┊61┊      messages: [],
+┊  ┊62┊    };
+┊  ┊63┊    const chat$FromCache = of<GetChat.Chat>(_chat);
+┊  ┊64┊
 ┊51┊65┊    const query = this.apollo.watchQuery<GetChat.Query>({
 ┊52┊66┊      query: getChatQuery,
 ┊53┊67┊      variables: {
-┊54┊  ┊        chatId: chatId,
+┊  ┊68┊        chatId,
 ┊55┊69┊      }
 ┊56┊70┊    });
 ┊57┊71┊
-┊58┊  ┊    const chat$ = query.valueChanges.pipe(
-┊59┊  ┊      map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
-┊60┊  ┊    );
+┊  ┊72┊    const chat$ = chat$FromCache.pipe(
+┊  ┊73┊      concat(query.valueChanges.pipe(
+┊  ┊74┊        map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
+┊  ┊75┊      )));
+┊  ┊76┊
+┊  ┊77┊    this.getChatWqSubject = new AsyncSubject();
+┊  ┊78┊    this.getChatWqSubject.next(query);
+┊  ┊79┊    this.getChatWqSubject.complete();
 ┊61┊80┊
-┊62┊  ┊    return {query, chat$};
+┊  ┊81┊    return {query$: this.getChatWqSubject.asObservable(), chat$};
 ┊63┊82┊  }
 ┊64┊83┊
 ┊65┊84┊  addMessage(chatId: string, content: string) {
```

[}]: #

Now let's deal with the most difficult part: what about chats creation? We cannot predict the `id` of the new chat and so we cannot navigate to the chat page because it contains the chat id in the url. We could simply navigate to the "optimistic" id, but then the user wouldn't be able to reach that url if he refreshes the page or bookmarks it. That's a problem we care about. How to solve it? We're going to create a landing page and we will later override the url once we get the response from the server!

[{]: <helper> (diffStep "9.3" module="client")

#### Step 9.3: Landing page for new chats/groups

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -116,7 +116,10 @@
 ┊116┊116┊        Apollo,
 ┊117┊117┊        {
 ┊118┊118┊          provide: ActivatedRoute,
-┊119┊   ┊          useValue: { params: of({ id: chat.id }) }
+┊   ┊119┊          useValue: {
+┊   ┊120┊            params: of({ id: chat.id }),
+┊   ┊121┊            queryParams: of({}),
+┊   ┊122┊          }
 ┊120┊123┊        }
 ┊121┊124┊      ],
 ┊122┊125┊      schemas: [NO_ERRORS_SCHEMA]
```

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -2,6 +2,8 @@
 ┊2┊2┊import {ActivatedRoute, Router} from '@angular/router';
 ┊3┊3┊import {ChatsService} from '../../../services/chats.service';
 ┊4┊4┊import {GetChat} from '../../../../types';
+┊ ┊5┊import {combineLatest} from 'rxjs/observable/combineLatest';
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
@@ -1,5 +1,5 @@
 ┊1┊1┊import {ApolloQueryResult, MutationOptions, WatchQueryOptions} from 'apollo-client';
-┊2┊ ┊import {concat, map} from 'rxjs/operators';
+┊ ┊2┊import {concat, map, share, switchMap} from 'rxjs/operators';
 ┊3┊3┊import {Apollo, QueryRef} from 'apollo-angular';
 ┊4┊4┊import {Injectable} from '@angular/core';
 ┊5┊5┊import {getChatsQuery} from '../../graphql/getChats.query';
```
```diff
@@ -17,6 +17,7 @@
 ┊17┊17┊import * as moment from 'moment';
 ┊18┊18┊import {AsyncSubject} from 'rxjs/AsyncSubject';
 ┊19┊19┊import {of} from 'rxjs/observable/of';
+┊  ┊20┊import {FetchResult} from 'apollo-link';
 ┊20┊21┊
 ┊21┊22┊const currentUserId = '1';
 ┊22┊23┊const currentUserName = 'Ethan Gonzalez';
```
```diff
@@ -28,6 +29,7 @@
 ┊28┊29┊  chats$: Observable<GetChats.Chats[]>;
 ┊29┊30┊  chats: GetChats.Chats[];
 ┊30┊31┊  getChatWqSubject: AsyncSubject<QueryRef<GetChat.Query>>;
+┊  ┊32┊  addChat$: Observable<FetchResult<AddChat.Mutation | AddGroup.Mutation>>;
 ┊31┊33┊
 ┊32┊34┊  constructor(private apollo: Apollo) {
 ┊33┊35┊    this.getChatsWq = this.apollo.watchQuery<GetChats.Query>(<WatchQueryOptions>{
```
```diff
@@ -50,7 +52,7 @@
 ┊50┊52┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊51┊53┊  }
 ┊52┊54┊
-┊53┊  ┊  getChat(chatId: string) {
+┊  ┊55┊  getChat(chatId: string, oui?: boolean) {
 ┊54┊56┊    const _chat = this.chats && this.chats.find(chat => chat.id === chatId) || {
 ┊55┊57┊      id: chatId,
 ┊56┊58┊      name: '',
```
```diff
@@ -62,21 +64,39 @@
 ┊ 62┊ 64┊    };
 ┊ 63┊ 65┊    const chat$FromCache = of<GetChat.Chat>(_chat);
 ┊ 64┊ 66┊
-┊ 65┊   ┊    const query = this.apollo.watchQuery<GetChat.Query>({
-┊ 66┊   ┊      query: getChatQuery,
-┊ 67┊   ┊      variables: {
-┊ 68┊   ┊        chatId,
-┊ 69┊   ┊      }
-┊ 70┊   ┊    });
-┊ 71┊   ┊
-┊ 72┊   ┊    const chat$ = chat$FromCache.pipe(
-┊ 73┊   ┊      concat(query.valueChanges.pipe(
-┊ 74┊   ┊        map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
-┊ 75┊   ┊      )));
+┊   ┊ 67┊    const getApolloWatchQuery = (id: string) => {
+┊   ┊ 68┊      return this.apollo.watchQuery<GetChat.Query>({
+┊   ┊ 69┊        query: getChatQuery,
+┊   ┊ 70┊        variables: {
+┊   ┊ 71┊          chatId: id,
+┊   ┊ 72┊        }
+┊   ┊ 73┊      });
+┊   ┊ 74┊    };
 ┊ 76┊ 75┊
+┊   ┊ 76┊    let chat$: Observable<GetChat.Chat>;
 ┊ 77┊ 77┊    this.getChatWqSubject = new AsyncSubject();
-┊ 78┊   ┊    this.getChatWqSubject.next(query);
-┊ 79┊   ┊    this.getChatWqSubject.complete();
+┊   ┊ 78┊
+┊   ┊ 79┊    if (oui) {
+┊   ┊ 80┊      chat$ = chat$FromCache.pipe(
+┊   ┊ 81┊        concat(this.addChat$.pipe(
+┊   ┊ 82┊          switchMap(({ data: { addChat, addGroup } }) => {
+┊   ┊ 83┊            const query = getApolloWatchQuery(addChat ? addChat.id : addGroup.id);
+┊   ┊ 84┊            this.getChatWqSubject.next(query);
+┊   ┊ 85┊            this.getChatWqSubject.complete();
+┊   ┊ 86┊            return query.valueChanges.pipe(
+┊   ┊ 87┊              map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
+┊   ┊ 88┊            );
+┊   ┊ 89┊          }))
+┊   ┊ 90┊        ));
+┊   ┊ 91┊    } else {
+┊   ┊ 92┊      const query = getApolloWatchQuery(chatId);
+┊   ┊ 93┊      this.getChatWqSubject.next(query);
+┊   ┊ 94┊      this.getChatWqSubject.complete();
+┊   ┊ 95┊      chat$ = chat$FromCache.pipe(
+┊   ┊ 96┊        concat(query.valueChanges.pipe(
+┊   ┊ 97┊          map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
+┊   ┊ 98┊        )));
+┊   ┊ 99┊    }
 ┊ 80┊100┊
 ┊ 81┊101┊    return {query$: this.getChatWqSubject.asObservable(), chat$};
 ┊ 82┊102┊  }
```
```diff
@@ -274,8 +294,8 @@
 ┊274┊294┊    return _chat ? _chat.id : false;
 ┊275┊295┊  }
 ┊276┊296┊
-┊277┊   ┊  addChat(recipientId: string, users: GetUsers.Users[]) {
-┊278┊   ┊    return this.apollo.mutate({
+┊   ┊297┊  addChat(recipientId: string, users: GetUsers.Users[], ouiId: string) {
+┊   ┊298┊    this.addChat$ = this.apollo.mutate({
 ┊279┊299┊      mutation: addChatMutation,
 ┊280┊300┊      variables: <AddChat.Variables>{
 ┊281┊301┊        recipientId,
```
```diff
@@ -283,7 +303,7 @@
 ┊283┊303┊      optimisticResponse: {
 ┊284┊304┊        __typename: 'Mutation',
 ┊285┊305┊        addChat: {
-┊286┊   ┊          id: ChatsService.getRandomId(),
+┊   ┊306┊          id: ouiId,
 ┊287┊307┊          __typename: 'Chat',
 ┊288┊308┊          name: users.find(user => user.id === recipientId).name,
 ┊289┊309┊          picture: users.find(user => user.id === recipientId).picture,
```
```diff
@@ -323,11 +343,12 @@
 ┊323┊343┊          },
 ┊324┊344┊        });
 ┊325┊345┊      },
-┊326┊   ┊    });
+┊   ┊346┊    }).pipe(share());
+┊   ┊347┊    return this.addChat$;
 ┊327┊348┊  }
 ┊328┊349┊
-┊329┊   ┊  addGroup(recipientIds: string[], groupName: string) {
-┊330┊   ┊    return this.apollo.mutate({
+┊   ┊350┊  addGroup(recipientIds: string[], groupName: string, ouiId: string) {
+┊   ┊351┊    this.addChat$ = this.apollo.mutate({
 ┊331┊352┊      mutation: addGroupMutation,
 ┊332┊353┊      variables: <AddGroup.Variables>{
 ┊333┊354┊        recipientIds,
```
```diff
@@ -336,7 +357,7 @@
 ┊336┊357┊      optimisticResponse: {
 ┊337┊358┊        __typename: 'Mutation',
 ┊338┊359┊        addGroup: {
-┊339┊   ┊          id: ChatsService.getRandomId(),
+┊   ┊360┊          id: ouiId,
 ┊340┊361┊          __typename: 'Chat',
 ┊341┊362┊          name: groupName,
 ┊342┊363┊          picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
```
```diff
@@ -374,6 +395,7 @@
 ┊374┊395┊          },
 ┊375┊396┊        });
 ┊376┊397┊      },
-┊377┊   ┊    });
+┊   ┊398┊    }).pipe(share());
+┊   ┊399┊    return this.addChat$;
 ┊378┊400┊  }
 ┊379┊401┊}
```

[}]: #

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step12.md) | [Next Step >](step14.md) |
|:--------------------------------|--------------------------------:|

[}]: #
