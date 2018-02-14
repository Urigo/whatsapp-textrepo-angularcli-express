# Step 8: Chat viewer

[//]: # (head-end)


At this point we have a module which lists all of our chats, but we still need to show a particular chat.
We're going to implement in in this chapter.

### ChatViewer module

First, let's create a `ChatViewer` module:

[{]: <helper> (diffStep "4.1" files="src/app/app.module.ts, src/app/chat-viewer/chat-viewer.module.ts" module="client")

#### Step 4.1: Chat Viewer

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

#### Step 4.1: Chat Viewer

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

#### Step 4.1: Chat Viewer

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

#### Step 4.1: Chat Viewer

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -1,11 +1,11 @@
-┊ 1┊  ┊import {Component, Input} from '@angular/core';
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
 ┊ 2┊ 2┊import {GetChats} from '../../../../graphql';
 ┊ 3┊ 3┊
 ┊ 4┊ 4┊@Component({
 ┊ 5┊ 5┊  selector: 'app-chat-item',
 ┊ 6┊ 6┊  template: `
 ┊ 7┊ 7┊    <div class="chat-row">
-┊ 8┊  ┊        <div class="chat-recipient">
+┊  ┊ 8┊        <div class="chat-recipient" (click)="selectChat()">
 ┊ 9┊ 9┊          <img *ngIf="chat.picture" [src]="chat.picture" width="48" height="48">
 ┊10┊10┊          <div>{{ chat.name }} [id: {{ chat.id }}]</div>
 ┊11┊11┊        </div>
```
```diff
@@ -18,4 +18,11 @@
 ┊18┊18┊  // tslint:disable-next-line:no-input-rename
 ┊19┊19┊  @Input('item')
 ┊20┊20┊  chat: GetChats.Chats;
+┊  ┊21┊
+┊  ┊22┊  @Output()
+┊  ┊23┊  select = new EventEmitter<string>();
+┊  ┊24┊
+┊  ┊25┊  selectChat() {
+┊  ┊26┊    this.select.emit(this.chat.id);
+┊  ┊27┊  }
 ┊21┊28┊}
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
 ┊32┊33┊    <button class="chat-button" mat-fab color="primary">
 ┊33┊34┊      <mat-icon aria-label="Icon-button with a + icon">add</mat-icon>
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

#### Step 4.1: Chat Viewer

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.scss
```diff
@@ -0,0 +1,10 @@
+┊  ┊ 1┊.container {
+┊  ┊ 2┊  display: flex;
+┊  ┊ 3┊  flex-flow: column;
+┊  ┊ 4┊  justify-content: space-between;
+┊  ┊ 5┊  height: calc(100vh - 8vh);
+┊  ┊ 6┊
+┊  ┊ 7┊  app-confirm-selection {
+┊  ┊ 8┊    bottom: 10vh;
+┊  ┊ 9┊  }
+┊  ┊10┊}
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -0,0 +1,45 @@
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
+┊  ┊11┊      <div class="title">{{ name }}</div>
+┊  ┊12┊    </app-toolbar>
+┊  ┊13┊    <div class="container">
+┊  ┊14┊      <app-messages-list [items]="messages" [isGroup]="isGroup"></app-messages-list>
+┊  ┊15┊      <app-new-message></app-new-message>
+┊  ┊16┊    </div>
+┊  ┊17┊  `,
+┊  ┊18┊  styleUrls: ['./chat.component.scss']
+┊  ┊19┊})
+┊  ┊20┊export class ChatComponent implements OnInit {
+┊  ┊21┊  chatId: string;
+┊  ┊22┊  messages: any[];
+┊  ┊23┊  name: string;
+┊  ┊24┊  isGroup: boolean;
+┊  ┊25┊
+┊  ┊26┊  constructor(private route: ActivatedRoute,
+┊  ┊27┊              private router: Router,
+┊  ┊28┊              private chatsService: ChatsService) {
+┊  ┊29┊  }
+┊  ┊30┊
+┊  ┊31┊  ngOnInit() {
+┊  ┊32┊    this.route.params.subscribe(({id: chatId}) => {
+┊  ┊33┊      this.chatId = chatId;
+┊  ┊34┊      this.chatsService.getChat(chatId).chat$.subscribe(chat => {
+┊  ┊35┊        this.messages = chat.messages;
+┊  ┊36┊        this.name = chat.name;
+┊  ┊37┊        this.isGroup = chat.isGroup;
+┊  ┊38┊      });
+┊  ┊39┊    });
+┊  ┊40┊  }
+┊  ┊41┊
+┊  ┊42┊  goToChats() {
+┊  ┊43┊    this.router.navigate(['/chats']);
+┊  ┊44┊  }
+┊  ┊45┊}
```

[}]: #

The `ChatComponent` component contains:

 - Toolbar with chat's name and a "go back" button
 - List of messages sorted from oldest to newest
 - Space to submit a new message

### List of messages

We now split the list of messages to the host:

[{]: <helper> (diffStep "4.1" files="src/app/chat-viewer/components/messages-list/messages-list.component.ts, src/app/chat-viewer/components/messages-list/messages-list.component.scss" module="client")

#### Step 4.1: Chat Viewer

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;messages-list&#x2F;messages-list.component.scss
```diff
@@ -0,0 +1,12 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  height: 100%;
+┊  ┊ 4┊  overflow-y: scroll;
+┊  ┊ 5┊  background-color: aliceblue;
+┊  ┊ 6┊}
+┊  ┊ 7┊
+┊  ┊ 8┊/*
+┊  ┊ 9┊:host::-webkit-scrollbar {
+┊  ┊10┊  display: none;
+┊  ┊11┊}
+┊  ┊12┊*/
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

#### Step 4.1: Chat Viewer

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;message-item&#x2F;message-item.component.scss
```diff
@@ -0,0 +1,18 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: flex;
+┊  ┊ 3┊  width: 100%;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊.message {
+┊  ┊ 7┊  max-width: 75%;
+┊  ┊ 8┊  background-color: lightgoldenrodyellow;
+┊  ┊ 9┊
+┊  ┊10┊  &.mine {
+┊  ┊11┊    background-color: lightcyan;
+┊  ┊12┊    margin-left: auto;
+┊  ┊13┊  }
+┊  ┊14┊
+┊  ┊15┊  .message-sender {
+┊  ┊16┊    font-size: small;
+┊  ┊17┊  }
+┊  ┊18┊}
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;message-item&#x2F;message-item.component.ts
```diff
@@ -0,0 +1,21 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-message-item',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <div class="message"
+┊  ┊ 7┊         [ngClass]="{'mine': message.ownership}">
+┊  ┊ 8┊      <div *ngIf="isGroup && !message.ownership" class="message-sender">{{ message.sender.name }}</div>
+┊  ┊ 9┊      <div>{{ message.content }}</div>
+┊  ┊10┊    </div>
+┊  ┊11┊  `,
+┊  ┊12┊  styleUrls: ['message-item.component.scss'],
+┊  ┊13┊})
+┊  ┊14┊export class MessageItemComponent {
+┊  ┊15┊  // tslint:disable-next-line:no-input-rename
+┊  ┊16┊  @Input('item')
+┊  ┊17┊  message: any;
+┊  ┊18┊
+┊  ┊19┊  @Input()
+┊  ┊20┊  isGroup: boolean;
+┊  ┊21┊}
```

[}]: #

As you see, we could easily decide which message comes from which user based on the `ownership` property.

### Submit a new message

The app shows the conversation and now we're going to focus on the last part, actually emitting a new message!

[{]: <helper> (diffStep "4.1" files="src/app/chat-viewer/components/new-message/new-message.component.scss, src/app/chat-viewer/components/new-message/new-message.component.ts" module="client")

#### Step 4.1: Chat Viewer

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;new-message&#x2F;new-message.component.scss
```diff
@@ -0,0 +1,13 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: flex;
+┊  ┊ 3┊  height: 8vh;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊input {
+┊  ┊ 7┊  width: 100%;
+┊  ┊ 8┊}
+┊  ┊ 9┊
+┊  ┊10┊button {
+┊  ┊11┊  width: 8vh;
+┊  ┊12┊  min-width: 56px;
+┊  ┊13┊}
```

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;new-message&#x2F;new-message.component.ts
```diff
@@ -0,0 +1,34 @@
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-new-message',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <input type="text" [(ngModel)]="message" (keyup)="onInputKeyup($event)"/>
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

| [< Previous Step](step7.md) | [Next Step >](step9.md) |
|:--------------------------------|--------------------------------:|

[}]: #
