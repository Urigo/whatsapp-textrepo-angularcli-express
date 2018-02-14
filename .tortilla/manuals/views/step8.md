# Step 8: Chat viewer

[//]: # (head-end)


At this point we have a module which lists all of our chats, but we still need to show a particular chat.
We're going to implement in in this chapter.

### ChatViewer module

First, let's create a `ChatViewer` module:

[{]: <helper> (diffStep "4.1" files="src/app/app.module.ts, src/app/chat-viewer/chat-viewer.module.ts" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -6,6 +6,7 @@
 ┊ 6┊ 6┊import { GraphQLModule } from './graphql.module';
 ┊ 7┊ 7┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
 ┊ 8┊ 8┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊ 9┊import {ChatViewerModule} from './chat-viewer/chat-viewer.module';
 ┊ 9┊10┊const routes: Routes = [];
 ┊10┊11┊
 ┊11┊12┊@NgModule({
```
```diff
@@ -20,6 +21,7 @@
 ┊20┊21┊    RouterModule.forRoot(routes),
 ┊21┊22┊    // Feature modules
 ┊22┊23┊    ChatsListerModule,
+┊  ┊24┊    ChatViewerModule,
 ┊23┊25┊  ],
 ┊24┊26┊  providers: [],
 ┊25┊27┊  bootstrap: [AppComponent]
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;chat-viewer.module.ts
```diff
@@ -0,0 +1,53 @@
+┊  ┊ 1┊import { BrowserModule } from '@angular/platform-browser';
+┊  ┊ 2┊import { NgModule } from '@angular/core';
+┊  ┊ 3┊
+┊  ┊ 4┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 5┊import {MatButtonModule, MatGridListModule, MatIconModule, MatListModule, MatMenuModule, MatToolbarModule} from '@angular/material';
+┊  ┊ 6┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊ 7┊import {FormsModule} from '@angular/forms';
+┊  ┊ 8┊import {ChatsService} from '../services/chats.service';
+┊  ┊ 9┊import {ChatComponent} from './containers/chat/chat.component';
+┊  ┊10┊import {MessagesListComponent} from './components/messages-list/messages-list.component';
+┊  ┊11┊import {MessageItemComponent} from './components/message-item/message-item.component';
+┊  ┊12┊import {NewMessageComponent} from './components/new-message/new-message.component';
+┊  ┊13┊import {SharedModule} from '../shared/shared.module';
+┊  ┊14┊
+┊  ┊15┊const routes: Routes = [
+┊  ┊16┊  {
+┊  ┊17┊    path: 'chat', children: [
+┊  ┊18┊      {path: ':id', component: ChatComponent},
+┊  ┊19┊    ],
+┊  ┊20┊  },
+┊  ┊21┊];
+┊  ┊22┊
+┊  ┊23┊@NgModule({
+┊  ┊24┊  declarations: [
+┊  ┊25┊    ChatComponent,
+┊  ┊26┊    MessagesListComponent,
+┊  ┊27┊    MessageItemComponent,
+┊  ┊28┊    NewMessageComponent,
+┊  ┊29┊  ],
+┊  ┊30┊  imports: [
+┊  ┊31┊    BrowserModule,
+┊  ┊32┊    // Material
+┊  ┊33┊    MatToolbarModule,
+┊  ┊34┊    MatMenuModule,
+┊  ┊35┊    MatIconModule,
+┊  ┊36┊    MatButtonModule,
+┊  ┊37┊    MatListModule,
+┊  ┊38┊    MatGridListModule,
+┊  ┊39┊    // Animations
+┊  ┊40┊    BrowserAnimationsModule,
+┊  ┊41┊    // Routing
+┊  ┊42┊    RouterModule.forChild(routes),
+┊  ┊43┊    // Forms
+┊  ┊44┊    FormsModule,
+┊  ┊45┊    // Feature modules
+┊  ┊46┊    SharedModule,
+┊  ┊47┊  ],
+┊  ┊48┊  providers: [
+┊  ┊49┊    ChatsService,
+┊  ┊50┊  ],
+┊  ┊51┊})
+┊  ┊52┊export class ChatViewerModule {
+┊  ┊53┊}
```

[}]: #

As you can see, it has already everything that we might need.

### GetChat operation

Components need data, so we create the `GetChat` operation and run `yarn generator`:

[{]: <helper> (diffStep "4.1" files="src/graphql/getChat.query.ts" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Added src&#x2F;graphql&#x2F;getChat.query.ts
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const getChatQuery = gql`
+┊  ┊ 6┊  query GetChat($chatId: ID!) {
+┊  ┊ 7┊    chat(chatId: $chatId) {
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

[}]: #

The query can be now implemented in `ChatsService`:

[{]: <helper> (diffStep "4.1" files="src/app/services/chats.service.ts" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,13 +1,14 @@
 ┊ 1┊ 1┊import {map} from 'rxjs/operators';
 ┊ 2┊ 2┊import {Injectable} from '@angular/core';
-┊ 3┊  ┊import {GetChatsGQL} from '../../graphql';
+┊  ┊ 3┊import {GetChatsGQL, GetChatGQL} from '../../graphql';
 ┊ 4┊ 4┊
 ┊ 5┊ 5┊@Injectable()
 ┊ 6┊ 6┊export class ChatsService {
 ┊ 7┊ 7┊  messagesAmount = 3;
 ┊ 8┊ 8┊
 ┊ 9┊ 9┊  constructor(
-┊10┊  ┊    private getChatsGQL: GetChatsGQL
+┊  ┊10┊    private getChatsGQL: GetChatsGQL,
+┊  ┊11┊    private getChatGQL: GetChatGQL
 ┊11┊12┊  ) {}
 ┊12┊13┊
 ┊13┊14┊  getChats() {
```
```diff
@@ -20,4 +21,16 @@
 ┊20┊21┊
 ┊21┊22┊    return {query, chats$};
 ┊22┊23┊  }
+┊  ┊24┊
+┊  ┊25┊  getChat(chatId: string) {
+┊  ┊26┊    const query = this.getChatGQL.watch({
+┊  ┊27┊      chatId: chatId,
+┊  ┊28┊    });
+┊  ┊29┊
+┊  ┊30┊    const chat$ = query.valueChanges.pipe(
+┊  ┊31┊      map((result) => result.data.chat)
+┊  ┊32┊    );
+┊  ┊33┊
+┊  ┊34┊    return {query, chat$};
+┊  ┊35┊  }
 ┊23┊36┊}
```

[}]: #

Great!

### Chat view

Now we've got data but a user can't still access the chat view.
There is one place where we're able to pick the chat, it's the list:

[{]: <helper> (diffStep "4.1" files="src/app/chats-lister/components/chat-item/chat-item.component.ts, src/app/chats-lister/components/chats-list/chats-list.component.ts, src/app/chats-lister/containers/chats/chats.component.ts" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -1,10 +1,10 @@
-┊ 1┊  ┊import {Component, Input} from '@angular/core';
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
 ┊ 2┊ 2┊import {GetChats} from '../../../../graphql';
 ┊ 3┊ 3┊
 ┊ 4┊ 4┊@Component({
 ┊ 5┊ 5┊  selector: 'app-chat-item',
 ┊ 6┊ 6┊  template: `
-┊ 7┊  ┊    <div class="chat-row">
+┊  ┊ 7┊    <div class="chat-row" (click)="selectChat()">
 ┊ 8┊ 8┊      <img class="chat-pic" [src]="chat.picture || 'assets/default-profile-pic.jpg'">
 ┊ 9┊ 9┊      <div class="chat-info">
 ┊10┊10┊        <div class="chat-name">{{ chat.name }}</div>
```
```diff
@@ -19,4 +19,11 @@
 ┊19┊19┊  // tslint:disable-next-line:no-input-rename
 ┊20┊20┊  @Input('item')
 ┊21┊21┊  chat: GetChats.Chats;
+┊  ┊22┊
+┊  ┊23┊  @Output()
+┊  ┊24┊  select = new EventEmitter<string>();
+┊  ┊25┊
+┊  ┊26┊  selectChat() {
+┊  ┊27┊    this.select.emit(this.chat.id);
+┊  ┊28┊  }
 ┊22┊29┊}
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chats-list&#x2F;chats-list.component.ts
```diff
@@ -1,4 +1,4 @@
-┊1┊ ┊import {Component, Input} from '@angular/core';
+┊ ┊1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
 ┊2┊2┊import {GetChats} from '../../../../graphql';
 ┊3┊3┊
 ┊4┊4┊@Component({
```
```diff
@@ -6,7 +6,7 @@
 ┊ 6┊ 6┊  template: `
 ┊ 7┊ 7┊    <mat-list>
 ┊ 8┊ 8┊      <mat-list-item *ngFor="let chat of chats">
-┊ 9┊  ┊        <app-chat-item [item]="chat"></app-chat-item>
+┊  ┊ 9┊        <app-chat-item [item]="chat" (select)="selectChat($event)"></app-chat-item>
 ┊10┊10┊      </mat-list-item>
 ┊11┊11┊    </mat-list>
 ┊12┊12┊  `,
```
```diff
@@ -17,5 +17,12 @@
 ┊17┊17┊  @Input('items')
 ┊18┊18┊  chats: GetChats.Chats[];
 ┊19┊19┊
+┊  ┊20┊  @Output()
+┊  ┊21┊  select = new EventEmitter<string>();
+┊  ┊22┊
 ┊20┊23┊  constructor() {}
+┊  ┊24┊
+┊  ┊25┊  selectChat(id: string) {
+┊  ┊26┊    this.select.emit(id);
+┊  ┊27┊  }
 ┊21┊28┊}
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.ts
```diff
@@ -2,6 +2,7 @@
 ┊2┊2┊import {ChatsService} from '../../../services/chats.service';
 ┊3┊3┊import {Observable} from 'rxjs';
 ┊4┊4┊import {GetChats} from '../../../../graphql';
+┊ ┊5┊import {Router} from '@angular/router';
 ┊5┊6┊
 ┊6┊7┊@Component({
 ┊7┊8┊  template: `
```
```diff
@@ -27,7 +28,7 @@
 ┊27┊28┊      </button>
 ┊28┊29┊    </mat-menu>
 ┊29┊30┊
-┊30┊  ┊    <app-chats-list [items]="chats$ | async"></app-chats-list>
+┊  ┊31┊    <app-chats-list [items]="chats$ | async" (select)="goToChat($event)"></app-chats-list>
 ┊31┊32┊
 ┊32┊33┊    <button class="chat-button" mat-fab color="secondary">
 ┊33┊34┊      <mat-icon aria-label="Icon-button with a + icon">chat</mat-icon>
```
```diff
@@ -38,10 +39,15 @@
 ┊38┊39┊export class ChatsComponent implements OnInit {
 ┊39┊40┊  chats$: Observable<GetChats.Chats[]>;
 ┊40┊41┊
-┊41┊  ┊  constructor(private chatsService: ChatsService) {
+┊  ┊42┊  constructor(private chatsService: ChatsService,
+┊  ┊43┊              private router: Router) {
 ┊42┊44┊  }
 ┊43┊45┊
 ┊44┊46┊  ngOnInit() {
 ┊45┊47┊    this.chats$ = this.chatsService.getChats().chats$;
 ┊46┊48┊  }
+┊  ┊49┊
+┊  ┊50┊  goToChat(chatId: string) {
+┊  ┊51┊    this.router.navigate(['/chat', chatId]);
+┊  ┊52┊  }
 ┊47┊53┊}
```

[}]: #

Time for a next step, an actual implementation of the chat view.

[{]: <helper> (diffStep "4.1" files="src/app/chat-viewer/containers/chat/chat.component.ts, src/app/chat-viewer/containers/chat/chat.component.scss" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.scss
```diff
@@ -0,0 +1,33 @@
+┊  ┊ 1┊app-toolbar {
+┊  ┊ 2┊  .navigation {
+┊  ┊ 3┊    padding: 0;
+┊  ┊ 4┊    display: inherit;
+┊  ┊ 5┊    margin-right: -40px;
+┊  ┊ 6┊  }
+┊  ┊ 7┊
+┊  ┊ 8┊  .title {
+┊  ┊ 9┊    line-height: 8vh;
+┊  ┊10┊  }
+┊  ┊11┊
+┊  ┊12┊  .profile-pic {
+┊  ┊13┊    height: calc(100% - 10px);
+┊  ┊14┊    width: auto;
+┊  ┊15┊    padding: 5px;
+┊  ┊16┊    border-radius: 50%;
+┊  ┊17┊  }
+┊  ┊18┊}
+┊  ┊19┊
+┊  ┊20┊.container {
+┊  ┊21┊  display: flex;
+┊  ┊22┊  flex-flow: column;
+┊  ┊23┊  justify-content: space-between;
+┊  ┊24┊  height: calc(100vh - 8vh);
+┊  ┊25┊  background-image: url(/assets/chat-background.jpg);
+┊  ┊26┊  background-color: #E0DAD6;
+┊  ┊27┊  background-repeat: no-repeat;
+┊  ┊28┊  background-size: cover;
+┊  ┊29┊
+┊  ┊30┊  app-confirm-selection {
+┊  ┊31┊    bottom: 10vh;
+┊  ┊32┊  }
+┊  ┊33┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -0,0 +1,46 @@
+┊  ┊ 1┊import {Component, OnInit} from '@angular/core';
+┊  ┊ 2┊import {ActivatedRoute, Router} from '@angular/router';
+┊  ┊ 3┊import {ChatsService} from '../../../services/chats.service';
+┊  ┊ 4┊
+┊  ┊ 5┊@Component({
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <app-toolbar>
+┊  ┊ 8┊      <button class="navigation" mat-button (click)="goToChats()">
+┊  ┊ 9┊        <mat-icon aria-label="Icon-button with an arrow back icon">arrow_back</mat-icon>
+┊  ┊10┊      </button>
+┊  ┊11┊      <img class="profile-pic" src="assets/default-profile-pic.jpg">
+┊  ┊12┊      <div class="title">{{ name }}</div>
+┊  ┊13┊    </app-toolbar>
+┊  ┊14┊    <div class="container">
+┊  ┊15┊      <app-messages-list [items]="messages" [isGroup]="isGroup"></app-messages-list>
+┊  ┊16┊      <app-new-message></app-new-message>
+┊  ┊17┊    </div>
+┊  ┊18┊  `,
+┊  ┊19┊  styleUrls: ['./chat.component.scss']
+┊  ┊20┊})
+┊  ┊21┊export class ChatComponent implements OnInit {
+┊  ┊22┊  chatId: string;
+┊  ┊23┊  messages: any[];
+┊  ┊24┊  name: string;
+┊  ┊25┊  isGroup: boolean;
+┊  ┊26┊
+┊  ┊27┊  constructor(private route: ActivatedRoute,
+┊  ┊28┊              private router: Router,
+┊  ┊29┊              private chatsService: ChatsService) {
+┊  ┊30┊  }
+┊  ┊31┊
+┊  ┊32┊  ngOnInit() {
+┊  ┊33┊    this.route.params.subscribe(({id: chatId}) => {
+┊  ┊34┊      this.chatId = chatId;
+┊  ┊35┊      this.chatsService.getChat(chatId).chat$.subscribe(chat => {
+┊  ┊36┊        this.messages = chat.messages;
+┊  ┊37┊        this.name = chat.name;
+┊  ┊38┊        this.isGroup = chat.isGroup;
+┊  ┊39┊      });
+┊  ┊40┊    });
+┊  ┊41┊  }
+┊  ┊42┊
+┊  ┊43┊  goToChats() {
+┊  ┊44┊    this.router.navigate(['/chats']);
+┊  ┊45┊  }
+┊  ┊46┊}
```

[}]: #

The `ChatComponent` component contains:

 - Toolbar with chat's name and a "go back" button
 - List of messages sorted from oldest to newest
 - Space to submit a new message

### List of messages

First we'll download and a couple of images which will help us create a "bubble" effect for received messages:

    $ wget https://raw.githubusercontent.com/Urigo/whatsapp-client-angularcli-material/4f4497df6187c6cf42bb01d7c516db7f9f08e32c/src/assets/chat-background.jpg src/assets
    $ wget https://raw.githubusercontent.com/Urigo/whatsapp-client-angularcli-material/4f4497df6187c6cf42bb01d7c516db7f9f08e32c/src/assets/message-mine.png src/assets
    $ wget https://raw.githubusercontent.com/Urigo/whatsapp-client-angularcli-material/4f4497df6187c6cf42bb01d7c516db7f9f08e32c/src/assets/message-other.png src/assets

We'll now split the list of messages to the host:

[{]: <helper> (diffStep "4.1" files="src/app/chat-viewer/components/messages-list/messages-list.component.ts, src/app/chat-viewer/components/messages-list/messages-list.component.scss" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;messages-list&#x2F;messages-list.component.scss
```diff
@@ -0,0 +1,15 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  height: 100%;
+┊  ┊ 4┊  overflow-y: overlay;
+┊  ┊ 5┊}
+┊  ┊ 6┊
+┊  ┊ 7┊::ng-deep .mat-list-item-content {
+┊  ┊ 8┊  display: block !important;
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊/*
+┊  ┊12┊:host::-webkit-scrollbar {
+┊  ┊13┊  display: none;
+┊  ┊14┊}
+┊  ┊15┊*/🚫↵
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;messages-list&#x2F;messages-list.component.ts
```diff
@@ -0,0 +1,23 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-messages-list',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <mat-list>
+┊  ┊ 7┊      <mat-list-item *ngFor="let message of messages">
+┊  ┊ 8┊        <app-message-item [item]="message" [isGroup]="isGroup"></app-message-item>
+┊  ┊ 9┊      </mat-list-item>
+┊  ┊10┊    </mat-list>
+┊  ┊11┊  `,
+┊  ┊12┊  styleUrls: ['messages-list.component.scss'],
+┊  ┊13┊})
+┊  ┊14┊export class MessagesListComponent {
+┊  ┊15┊  // tslint:disable-next-line:no-input-rename
+┊  ┊16┊  @Input('items')
+┊  ┊17┊  messages: any[];
+┊  ┊18┊
+┊  ┊19┊  @Input()
+┊  ┊20┊  isGroup: boolean;
+┊  ┊21┊
+┊  ┊22┊  constructor() {}
+┊  ┊23┊}
```

[}]: #

and a reused component for each message:

[{]: <helper> (diffStep "4.1" files="src/app/chat-viewer/components/message-item/message-item.component.ts, src/app/chat-viewer/components/message-item/message-item.component.scss" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;message-item&#x2F;message-item.component.scss
```diff
@@ -0,0 +1,74 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  margin-bottom: 9px;
+┊  ┊ 3┊
+┊  ┊ 4┊  &::after {
+┊  ┊ 5┊    content: "";
+┊  ┊ 6┊    display: table;
+┊  ┊ 7┊    clear: both;
+┊  ┊ 8┊  }
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊.message {
+┊  ┊12┊  display: inline-block;
+┊  ┊13┊  position: relative;
+┊  ┊14┊  max-width: 100%;
+┊  ┊15┊  border-radius: 7px;
+┊  ┊16┊  box-shadow: 0 1px 2px rgba(0, 0, 0, .15);
+┊  ┊17┊  margin-bottom: 20px;
+┊  ┊18┊  clear: both;
+┊  ┊19┊
+┊  ┊20┊  &.message-mine {
+┊  ┊21┊    float: right;
+┊  ┊22┊    background-color: #DCF8C6;
+┊  ┊23┊
+┊  ┊24┊    &::before {
+┊  ┊25┊      right: -11px;
+┊  ┊26┊      background-image: url(/assets/message-mine.png)
+┊  ┊27┊    }
+┊  ┊28┊  }
+┊  ┊29┊
+┊  ┊30┊  &.message-other {
+┊  ┊31┊    float: left;
+┊  ┊32┊    background-color: #FFF;
+┊  ┊33┊
+┊  ┊34┊    &::before {
+┊  ┊35┊      left: -11px;
+┊  ┊36┊      background-image: url(/assets/message-other.png)
+┊  ┊37┊    }
+┊  ┊38┊  }
+┊  ┊39┊
+┊  ┊40┊  &.message-other::before, &.message-mine::before {
+┊  ┊41┊    content: "";
+┊  ┊42┊    position: absolute;
+┊  ┊43┊    bottom: 3px;
+┊  ┊44┊    width: 12px;
+┊  ┊45┊    height: 19px;
+┊  ┊46┊    background-position: 50% 50%;
+┊  ┊47┊    background-repeat: no-repeat;
+┊  ┊48┊    background-size: contain;
+┊  ┊49┊  }
+┊  ┊50┊
+┊  ┊51┊  .message-content {
+┊  ┊52┊    padding: 5px 7px;
+┊  ┊53┊    word-wrap: break-word;
+┊  ┊54┊
+┊  ┊55┊    &::after {
+┊  ┊56┊      content: " \00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0\00a0";
+┊  ┊57┊      display: inline;
+┊  ┊58┊    }
+┊  ┊59┊  }
+┊  ┊60┊
+┊  ┊61┊  .message-sender {
+┊  ┊62┊    font-weight: bold;
+┊  ┊63┊    font-size: 0.9em;
+┊  ┊64┊    padding: 5px;
+┊  ┊65┊  }
+┊  ┊66┊
+┊  ┊67┊  .message-timestamp {
+┊  ┊68┊    position: absolute;
+┊  ┊69┊    bottom: 2px;
+┊  ┊70┊    right: 7px;
+┊  ┊71┊    color: gray;
+┊  ┊72┊    font-size: 12px;
+┊  ┊73┊  }
+┊  ┊74┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;message-item&#x2F;message-item.component.ts
```diff
@@ -0,0 +1,22 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-message-item',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <div class="message"
+┊  ┊ 7┊         [ngClass]="message.ownership ? 'message-mine' : 'message-other'">
+┊  ┊ 8┊      <div *ngIf="isGroup && !message.ownership" class="message-sender">{{ message.sender.name }}</div>
+┊  ┊ 9┊      <div class="message-content">{{ message.content }}</div>
+┊  ┊10┊      <span class="message-timestamp">00:00</span>
+┊  ┊11┊    </div>
+┊  ┊12┊  `,
+┊  ┊13┊  styleUrls: ['message-item.component.scss'],
+┊  ┊14┊})
+┊  ┊15┊export class MessageItemComponent {
+┊  ┊16┊  // tslint:disable-next-line:no-input-rename
+┊  ┊17┊  @Input('item')
+┊  ┊18┊  message: any;
+┊  ┊19┊
+┊  ┊20┊  @Input()
+┊  ┊21┊  isGroup: boolean;
+┊  ┊22┊}
```

[}]: #

As you see, we could easily decide which message comes from which user based on the `ownership` property.

### Submit a new message

The app shows the conversation and now we're going to focus on the last part, actually emitting a new message!

[{]: <helper> (diffStep "4.1" files="src/app/chat-viewer/components/new-message/new-message.component.scss, src/app/chat-viewer/components/new-message/new-message.component.ts" module="client")

#### [Step 4.1: Chat Viewer](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/9b6e918)

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;new-message&#x2F;new-message.component.scss
```diff
@@ -0,0 +1,32 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: flex;
+┊  ┊ 3┊  height: 50px;
+┊  ┊ 4┊  padding: 5px;
+┊  ┊ 5┊}
+┊  ┊ 6┊
+┊  ┊ 7┊input {
+┊  ┊ 8┊  width: 100%;
+┊  ┊ 9┊  border: none;
+┊  ┊10┊  border-radius: 999px;
+┊  ┊11┊  padding: 10px;
+┊  ┊12┊  padding-left: 20px;
+┊  ┊13┊  padding-right: 20px;
+┊  ┊14┊  font-size: 15px;
+┊  ┊15┊  outline: none;
+┊  ┊16┊  box-shadow: 0 1px silver;
+┊  ┊17┊  font-size: 18px;
+┊  ┊18┊  line-height: 45px;
+┊  ┊19┊}
+┊  ┊20┊
+┊  ┊21┊button {
+┊  ┊22┊  min-width: 45px;
+┊  ┊23┊  width: 45px;
+┊  ┊24┊  border-radius: 999px;
+┊  ┊25┊  background-color: var(--primary);
+┊  ┊26┊  margin-left: 5px;
+┊  ┊27┊  color: white;
+┊  ┊28┊
+┊  ┊29┊  mat-icon {
+┊  ┊30┊    margin-left: -3px;
+┊  ┊31┊  }
+┊  ┊32┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;new-message&#x2F;new-message.component.ts
```diff
@@ -0,0 +1,34 @@
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-new-message',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <input type="text" placeholder="Type a message" [(ngModel)]="message" (keyup)="onInputKeyup($event)"/>
+┊  ┊ 7┊    <button mat-button (click)="emitMessage()" [disabled]="disabled">
+┊  ┊ 8┊      <mat-icon aria-label="Icon-button with a send icon">send</mat-icon>
+┊  ┊ 9┊    </button>
+┊  ┊10┊  `,
+┊  ┊11┊  styleUrls: ['new-message.component.scss'],
+┊  ┊12┊})
+┊  ┊13┊export class NewMessageComponent {
+┊  ┊14┊  @Input()
+┊  ┊15┊  disabled: boolean;
+┊  ┊16┊
+┊  ┊17┊  @Output()
+┊  ┊18┊  newMessage = new EventEmitter<string>();
+┊  ┊19┊
+┊  ┊20┊  message = '';
+┊  ┊21┊
+┊  ┊22┊  onInputKeyup({ keyCode }: KeyboardEvent) {
+┊  ┊23┊    if (keyCode === 13) {
+┊  ┊24┊      this.emitMessage();
+┊  ┊25┊    }
+┊  ┊26┊  }
+┊  ┊27┊
+┊  ┊28┊  emitMessage() {
+┊  ┊29┊    if (this.message && !this.disabled) {
+┊  ┊30┊      this.newMessage.emit(this.message);
+┊  ┊31┊      this.message = '';
+┊  ┊32┊    }
+┊  ┊33┊  }
+┊  ┊34┊}
```

[}]: #

It's not yet fully functional, we receive the message in the `ChatComponent` but we still need to send it to the server. We'll cover that in next few steps!


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.3.1/.tortilla/manuals/views/step7.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.3.1/.tortilla/manuals/views/step9.md) |
|:--------------------------------|--------------------------------:|

[}]: #
