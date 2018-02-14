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
 ┊14┊14┊import {Observable} from 'rxjs';
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
@@ -11,7 +11,7 @@
 ┊11┊11┊import {removeAllMessagesMutation} from '../../graphql/removeAllMessages.mutation';
 ┊12┊12┊import {removeMessagesMutation} from '../../graphql/removeMessages.mutation';
 ┊13┊13┊import {getUsersQuery} from '../../graphql/getUsers.query';
-┊14┊  ┊import {Observable} from 'rxjs';
+┊  ┊14┊import {Observable, AsyncSubject, of} from 'rxjs';
 ┊15┊15┊import {addChatMutation} from '../../graphql/addChat.mutation';
 ┊16┊16┊import {addGroupMutation} from '../../graphql/addGroup.mutation';
 ┊17┊17┊import * as moment from 'moment';
```
```diff
@@ -25,6 +25,7 @@
 ┊25┊25┊  getChatsWq: QueryRef<GetChats.Query>;
 ┊26┊26┊  chats$: Observable<GetChats.Chats[]>;
 ┊27┊27┊  chats: GetChats.Chats[];
+┊  ┊28┊  getChatWqSubject: AsyncSubject<QueryRef<GetChat.Query>>;
 ┊28┊29┊
 ┊29┊30┊  constructor(private apollo: Apollo) {
 ┊30┊31┊    this.getChatsWq = this.apollo.watchQuery<GetChats.Query>(<WatchQueryOptions>{
```
```diff
@@ -48,18 +49,34 @@
 ┊48┊49┊  }
 ┊49┊50┊
 ┊50┊51┊  getChat(chatId: string) {
+┊  ┊52┊    const _chat = this.chats && this.chats.find(chat => chat.id === chatId) || {
+┊  ┊53┊      id: chatId,
+┊  ┊54┊      name: '',
+┊  ┊55┊      picture: null,
+┊  ┊56┊      allTimeMembers: [],
+┊  ┊57┊      unreadMessages: 0,
+┊  ┊58┊      isGroup: false,
+┊  ┊59┊      messages: [],
+┊  ┊60┊    };
+┊  ┊61┊    const chat$FromCache = of<GetChat.Chat>(_chat);
+┊  ┊62┊
 ┊51┊63┊    const query = this.apollo.watchQuery<GetChat.Query>({
 ┊52┊64┊      query: getChatQuery,
 ┊53┊65┊      variables: {
-┊54┊  ┊        chatId: chatId,
+┊  ┊66┊        chatId,
 ┊55┊67┊      }
 ┊56┊68┊    });
 ┊57┊69┊
-┊58┊  ┊    const chat$ = query.valueChanges.pipe(
-┊59┊  ┊      map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
-┊60┊  ┊    );
+┊  ┊70┊    const chat$ = chat$FromCache.pipe(
+┊  ┊71┊      concat(query.valueChanges.pipe(
+┊  ┊72┊        map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
+┊  ┊73┊      )));
+┊  ┊74┊
+┊  ┊75┊    this.getChatWqSubject = new AsyncSubject();
+┊  ┊76┊    this.getChatWqSubject.next(query);
+┊  ┊77┊    this.getChatWqSubject.complete();
 ┊61┊78┊
-┊62┊  ┊    return {query, chat$};
+┊  ┊79┊    return {query$: this.getChatWqSubject.asObservable(), chat$};
 ┊63┊80┊  }
 ┊64┊81┊
 ┊65┊82┊  addMessage(chatId: string, content: string) {
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
@@ -1,5 +1,5 @@
 ┊1┊1┊import {ApolloQueryResult, MutationOptions, WatchQueryOptions} from 'apollo-client';
-┊2┊ ┊import {concat, map} from 'rxjs/operators';
+┊ ┊2┊import {concat, map, share, switchMap} from 'rxjs/operators';
 ┊3┊3┊import {Apollo, QueryRef} from 'apollo-angular';
 ┊4┊4┊import {Injectable} from '@angular/core';
 ┊5┊5┊import {getChatsQuery} from '../../graphql/getChats.query';
```
```diff
@@ -15,6 +15,7 @@
 ┊15┊15┊import {addChatMutation} from '../../graphql/addChat.mutation';
 ┊16┊16┊import {addGroupMutation} from '../../graphql/addGroup.mutation';
 ┊17┊17┊import * as moment from 'moment';
+┊  ┊18┊import {FetchResult} from 'apollo-link';
 ┊18┊19┊
 ┊19┊20┊const currentUserId = '1';
 ┊20┊21┊const currentUserName = 'Ethan Gonzalez';
```
```diff
@@ -26,6 +27,7 @@
 ┊26┊27┊  chats$: Observable<GetChats.Chats[]>;
 ┊27┊28┊  chats: GetChats.Chats[];
 ┊28┊29┊  getChatWqSubject: AsyncSubject<QueryRef<GetChat.Query>>;
+┊  ┊30┊  addChat$: Observable<FetchResult<AddChat.Mutation | AddGroup.Mutation>>;
 ┊29┊31┊
 ┊30┊32┊  constructor(private apollo: Apollo) {
 ┊31┊33┊    this.getChatsWq = this.apollo.watchQuery<GetChats.Query>(<WatchQueryOptions>{
```
```diff
@@ -48,7 +50,7 @@
 ┊48┊50┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊49┊51┊  }
 ┊50┊52┊
-┊51┊  ┊  getChat(chatId: string) {
+┊  ┊53┊  getChat(chatId: string, oui?: boolean) {
 ┊52┊54┊    const _chat = this.chats && this.chats.find(chat => chat.id === chatId) || {
 ┊53┊55┊      id: chatId,
 ┊54┊56┊      name: '',
```
```diff
@@ -60,21 +62,39 @@
 ┊ 60┊ 62┊    };
 ┊ 61┊ 63┊    const chat$FromCache = of<GetChat.Chat>(_chat);
 ┊ 62┊ 64┊
-┊ 63┊   ┊    const query = this.apollo.watchQuery<GetChat.Query>({
-┊ 64┊   ┊      query: getChatQuery,
-┊ 65┊   ┊      variables: {
-┊ 66┊   ┊        chatId,
-┊ 67┊   ┊      }
-┊ 68┊   ┊    });
-┊ 69┊   ┊
-┊ 70┊   ┊    const chat$ = chat$FromCache.pipe(
-┊ 71┊   ┊      concat(query.valueChanges.pipe(
-┊ 72┊   ┊        map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
-┊ 73┊   ┊      )));
+┊   ┊ 65┊    const getApolloWatchQuery = (id: string) => {
+┊   ┊ 66┊      return this.apollo.watchQuery<GetChat.Query>({
+┊   ┊ 67┊        query: getChatQuery,
+┊   ┊ 68┊        variables: {
+┊   ┊ 69┊          chatId: id,
+┊   ┊ 70┊        }
+┊   ┊ 71┊      });
+┊   ┊ 72┊    };
 ┊ 74┊ 73┊
+┊   ┊ 74┊    let chat$: Observable<GetChat.Chat>;
 ┊ 75┊ 75┊    this.getChatWqSubject = new AsyncSubject();
-┊ 76┊   ┊    this.getChatWqSubject.next(query);
-┊ 77┊   ┊    this.getChatWqSubject.complete();
+┊   ┊ 76┊
+┊   ┊ 77┊    if (oui) {
+┊   ┊ 78┊      chat$ = chat$FromCache.pipe(
+┊   ┊ 79┊        concat(this.addChat$.pipe(
+┊   ┊ 80┊          switchMap(({ data: { addChat, addGroup } }) => {
+┊   ┊ 81┊            const query = getApolloWatchQuery(addChat ? addChat.id : addGroup.id);
+┊   ┊ 82┊            this.getChatWqSubject.next(query);
+┊   ┊ 83┊            this.getChatWqSubject.complete();
+┊   ┊ 84┊            return query.valueChanges.pipe(
+┊   ┊ 85┊              map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
+┊   ┊ 86┊            );
+┊   ┊ 87┊          }))
+┊   ┊ 88┊        ));
+┊   ┊ 89┊    } else {
+┊   ┊ 90┊      const query = getApolloWatchQuery(chatId);
+┊   ┊ 91┊      this.getChatWqSubject.next(query);
+┊   ┊ 92┊      this.getChatWqSubject.complete();
+┊   ┊ 93┊      chat$ = chat$FromCache.pipe(
+┊   ┊ 94┊        concat(query.valueChanges.pipe(
+┊   ┊ 95┊          map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
+┊   ┊ 96┊        )));
+┊   ┊ 97┊    }
 ┊ 78┊ 98┊
 ┊ 79┊ 99┊    return {query$: this.getChatWqSubject.asObservable(), chat$};
 ┊ 80┊100┊  }
```
```diff
@@ -272,8 +292,8 @@
 ┊272┊292┊    return _chat ? _chat.id : false;
 ┊273┊293┊  }
 ┊274┊294┊
-┊275┊   ┊  addChat(recipientId: string, users: GetUsers.Users[]) {
-┊276┊   ┊    return this.apollo.mutate({
+┊   ┊295┊  addChat(recipientId: string, users: GetUsers.Users[], ouiId: string) {
+┊   ┊296┊    this.addChat$ = this.apollo.mutate({
 ┊277┊297┊      mutation: addChatMutation,
 ┊278┊298┊      variables: <AddChat.Variables>{
 ┊279┊299┊        recipientId,
```
```diff
@@ -281,7 +301,7 @@
 ┊281┊301┊      optimisticResponse: {
 ┊282┊302┊        __typename: 'Mutation',
 ┊283┊303┊        addChat: {
-┊284┊   ┊          id: ChatsService.getRandomId(),
+┊   ┊304┊          id: ouiId,
 ┊285┊305┊          __typename: 'Chat',
 ┊286┊306┊          name: users.find(user => user.id === recipientId).name,
 ┊287┊307┊          picture: users.find(user => user.id === recipientId).picture,
```
```diff
@@ -321,11 +341,12 @@
 ┊321┊341┊          },
 ┊322┊342┊        });
 ┊323┊343┊      },
-┊324┊   ┊    });
+┊   ┊344┊    }).pipe(share());
+┊   ┊345┊    return this.addChat$;
 ┊325┊346┊  }
 ┊326┊347┊
-┊327┊   ┊  addGroup(recipientIds: string[], groupName: string) {
-┊328┊   ┊    return this.apollo.mutate({
+┊   ┊348┊  addGroup(recipientIds: string[], groupName: string, ouiId: string) {
+┊   ┊349┊    this.addChat$ = this.apollo.mutate({
 ┊329┊350┊      mutation: addGroupMutation,
 ┊330┊351┊      variables: <AddGroup.Variables>{
 ┊331┊352┊        recipientIds,
```
```diff
@@ -334,7 +355,7 @@
 ┊334┊355┊      optimisticResponse: {
 ┊335┊356┊        __typename: 'Mutation',
 ┊336┊357┊        addGroup: {
-┊337┊   ┊          id: ChatsService.getRandomId(),
+┊   ┊358┊          id: ouiId,
 ┊338┊359┊          __typename: 'Chat',
 ┊339┊360┊          name: groupName,
 ┊340┊361┊          picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
```
```diff
@@ -372,6 +393,7 @@
 ┊372┊393┊          },
 ┊373┊394┊        });
 ┊374┊395┊      },
-┊375┊   ┊    });
+┊   ┊396┊    }).pipe(share());
+┊   ┊397┊    return this.addChat$;
 ┊376┊398┊  }
 ┊377┊399┊}
```

[}]: #

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step12.md) | [Next Step >](step14.md) |
|:--------------------------------|--------------------------------:|

[}]: #
