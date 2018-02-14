# Step 8: Chat viewer

[//]: # (head-end)


We created a module which lists all of our chats, but we still need to show a particular chat.
Let's create the `chat-viewer` module! We're going to create a container component called `ChatComponent` and a couple of presentational components.

[{]: <helper> (diffStep "4.1" module="client")

#### Step 4.1: Chat Viewer

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -8,6 +8,7 @@
 ┊ 8┊ 8┊import {defaultDataIdFromObject, InMemoryCache} from 'apollo-cache-inmemory';
 ┊ 9┊ 9┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
 ┊10┊10┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊11┊import {ChatViewerModule} from './chat-viewer/chat-viewer.module';
 ┊11┊12┊const routes: Routes = [];
 ┊12┊13┊
 ┊13┊14┊@NgModule({
```
```diff
@@ -24,6 +25,7 @@
 ┊24┊25┊    RouterModule.forRoot(routes),
 ┊25┊26┊    // Feature modules
 ┊26┊27┊    ChatsListerModule,
+┊  ┊28┊    ChatViewerModule,
 ┊27┊29┊  ],
 ┊28┊30┊  providers: [],
 ┊29┊31┊  bootstrap: [AppComponent]
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

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -1,11 +1,11 @@
-┊ 1┊  ┊import {Component, Input} from '@angular/core';
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
 ┊ 2┊ 2┊import {GetChats} from '../../../../types';
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
 ┊2┊2┊import {GetChats} from '../../../../types';
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
 ┊4┊4┊import {GetChats} from '../../../../types';
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

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -4,6 +4,7 @@
 ┊ 4┊ 4┊import {Injectable} from '@angular/core';
 ┊ 5┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
 ┊ 6┊ 6┊import {GetChats} from '../../types';
+┊  ┊ 7┊import {getChatQuery} from '../../graphql/getChat.query';
 ┊ 7┊ 8┊
 ┊ 8┊ 9┊@Injectable()
 ┊ 9┊10┊export class ChatsService {
```
```diff
@@ -24,4 +25,19 @@
 ┊24┊25┊
 ┊25┊26┊    return {query, chats$};
 ┊26┊27┊  }
+┊  ┊28┊
+┊  ┊29┊  getChat(chatId: string) {
+┊  ┊30┊    const query = this.apollo.watchQuery<any>({
+┊  ┊31┊      query: getChatQuery,
+┊  ┊32┊      variables: {
+┊  ┊33┊        chatId: chatId,
+┊  ┊34┊      }
+┊  ┊35┊    });
+┊  ┊36┊
+┊  ┊37┊    const chat$ = query.valueChanges.pipe(
+┊  ┊38┊      map((result: ApolloQueryResult<any>) => result.data.chat)
+┊  ┊39┊    );
+┊  ┊40┊
+┊  ┊41┊    return {query, chat$};
+┊  ┊42┊  }
 ┊27┊43┊}
```

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

It's time to generate our types:

    $ npm run generator

And use them:

[{]: <helper> (diffStep "4.2" files="^\(?!src/types.d.ts$\).*" module="client")

#### Step 4.2: Add generated types

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;message-item&#x2F;message-item.component.ts
```diff
@@ -1,4 +1,5 @@
 ┊1┊1┊import {Component, Input} from '@angular/core';
+┊ ┊2┊import {GetChat} from '../../../../types';
 ┊2┊3┊
 ┊3┊4┊@Component({
 ┊4┊5┊  selector: 'app-message-item',
```
```diff
@@ -14,7 +15,7 @@
 ┊14┊15┊export class MessageItemComponent {
 ┊15┊16┊  // tslint:disable-next-line:no-input-rename
 ┊16┊17┊  @Input('item')
-┊17┊  ┊  message: any;
+┊  ┊18┊  message: GetChat.Messages;
 ┊18┊19┊
 ┊19┊20┊  @Input()
 ┊20┊21┊  isGroup: boolean;
```

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;messages-list&#x2F;messages-list.component.ts
```diff
@@ -1,4 +1,5 @@
 ┊1┊1┊import {Component, Input} from '@angular/core';
+┊ ┊2┊import {GetChat} from '../../../../types';
 ┊2┊3┊
 ┊3┊4┊@Component({
 ┊4┊5┊  selector: 'app-messages-list',
```
```diff
@@ -14,7 +15,7 @@
 ┊14┊15┊export class MessagesListComponent {
 ┊15┊16┊  // tslint:disable-next-line:no-input-rename
 ┊16┊17┊  @Input('items')
-┊17┊  ┊  messages: any[];
+┊  ┊18┊  messages: GetChat.Messages[];
 ┊18┊19┊
 ┊19┊20┊  @Input()
 ┊20┊21┊  isGroup: boolean;
```

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -1,6 +1,7 @@
 ┊1┊1┊import {Component, OnInit} from '@angular/core';
 ┊2┊2┊import {ActivatedRoute, Router} from '@angular/router';
 ┊3┊3┊import {ChatsService} from '../../../services/chats.service';
+┊ ┊4┊import {GetChat} from '../../../../types';
 ┊4┊5┊
 ┊5┊6┊@Component({
 ┊6┊7┊  template: `
```
```diff
@@ -19,7 +20,7 @@
 ┊19┊20┊})
 ┊20┊21┊export class ChatComponent implements OnInit {
 ┊21┊22┊  chatId: string;
-┊22┊  ┊  messages: any[];
+┊  ┊23┊  messages: GetChat.Messages[];
 ┊23┊24┊  name: string;
 ┊24┊25┊  isGroup: boolean;
 ┊25┊26┊
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -3,7 +3,7 @@
 ┊3┊3┊import {Apollo} from 'apollo-angular';
 ┊4┊4┊import {Injectable} from '@angular/core';
 ┊5┊5┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊6┊ ┊import {GetChats} from '../../types';
+┊ ┊6┊import {GetChat, GetChats} from '../../types';
 ┊7┊7┊import {getChatQuery} from '../../graphql/getChat.query';
 ┊8┊8┊
 ┊9┊9┊@Injectable()
```
```diff
@@ -27,7 +27,7 @@
 ┊27┊27┊  }
 ┊28┊28┊
 ┊29┊29┊  getChat(chatId: string) {
-┊30┊  ┊    const query = this.apollo.watchQuery<any>({
+┊  ┊30┊    const query = this.apollo.watchQuery<GetChat.Query>({
 ┊31┊31┊      query: getChatQuery,
 ┊32┊32┊      variables: {
 ┊33┊33┊        chatId: chatId,
```
```diff
@@ -35,7 +35,7 @@
 ┊35┊35┊    });
 ┊36┊36┊
 ┊37┊37┊    const chat$ = query.valueChanges.pipe(
-┊38┊  ┊      map((result: ApolloQueryResult<any>) => result.data.chat)
+┊  ┊38┊      map((result: ApolloQueryResult<GetChat.Query>) => result.data.chat)
 ┊39┊39┊    );
 ┊40┊40┊
 ┊41┊41┊    return {query, chat$};
```

[}]: #

We will also create some more tests for the newly created Chat container component:

[{]: <helper> (diffStep "4.3" module="client")

#### Step 4.3: Testing

##### Added src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -0,0 +1,170 @@
+┊   ┊  1┊import { async, ComponentFixture, TestBed } from '@angular/core/testing';
+┊   ┊  2┊
+┊   ┊  3┊import { ChatComponent } from './chat.component';
+┊   ┊  4┊import {DebugElement, NO_ERRORS_SCHEMA} from '@angular/core';
+┊   ┊  5┊import {MatButtonModule, MatGridListModule, MatIconModule, MatListModule, MatMenuModule, MatToolbarModule} from '@angular/material';
+┊   ┊  6┊import {ChatsService} from '../../../services/chats.service';
+┊   ┊  7┊import {Apollo} from 'apollo-angular';
+┊   ┊  8┊import {HttpClientTestingModule, HttpTestingController} from '@angular/common/http/testing';
+┊   ┊  9┊import {HttpLink, HttpLinkModule, Options} from 'apollo-angular-link-http';
+┊   ┊ 10┊import {defaultDataIdFromObject, InMemoryCache} from 'apollo-cache-inmemory';
+┊   ┊ 11┊import {RouterTestingModule} from '@angular/router/testing';
+┊   ┊ 12┊import {ActivatedRoute} from '@angular/router';
+┊   ┊ 13┊import {of} from 'rxjs';
+┊   ┊ 14┊import {By} from '@angular/platform-browser';
+┊   ┊ 15┊import {FormsModule} from '@angular/forms';
+┊   ┊ 16┊import {SharedModule} from '../../../shared/shared.module';
+┊   ┊ 17┊import {NewMessageComponent} from '../../components/new-message/new-message.component';
+┊   ┊ 18┊import {MessagesListComponent} from '../../components/messages-list/messages-list.component';
+┊   ┊ 19┊import {MessageItemComponent} from '../../components/message-item/message-item.component';
+┊   ┊ 20┊
+┊   ┊ 21┊describe('ChatComponent', () => {
+┊   ┊ 22┊  let component: ChatComponent;
+┊   ┊ 23┊  let fixture: ComponentFixture<ChatComponent>;
+┊   ┊ 24┊  let el: DebugElement;
+┊   ┊ 25┊
+┊   ┊ 26┊  let httpMock: HttpTestingController;
+┊   ┊ 27┊  let httpLink: HttpLink;
+┊   ┊ 28┊  let apollo: Apollo;
+┊   ┊ 29┊
+┊   ┊ 30┊  const chat: any = {
+┊   ┊ 31┊    id: '1',
+┊   ┊ 32┊    __typename: 'Chat',
+┊   ┊ 33┊    name: 'Avery Stewart',
+┊   ┊ 34┊    picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 35┊    allTimeMembers: [
+┊   ┊ 36┊      {
+┊   ┊ 37┊        id: '1',
+┊   ┊ 38┊        __typename: 'User',
+┊   ┊ 39┊      },
+┊   ┊ 40┊      {
+┊   ┊ 41┊        id: '3',
+┊   ┊ 42┊        __typename: 'User',
+┊   ┊ 43┊      }
+┊   ┊ 44┊    ],
+┊   ┊ 45┊    unreadMessages: 1,
+┊   ┊ 46┊    isGroup: false,
+┊   ┊ 47┊    messages: [
+┊   ┊ 48┊      {
+┊   ┊ 49┊        id: '1',
+┊   ┊ 50┊        chat: {
+┊   ┊ 51┊          id: '1',
+┊   ┊ 52┊          __typename: 'Chat',
+┊   ┊ 53┊        },
+┊   ┊ 54┊        __typename: 'Message',
+┊   ┊ 55┊        sender: {
+┊   ┊ 56┊          id: '3',
+┊   ┊ 57┊          __typename: 'User',
+┊   ┊ 58┊          name: 'Avery Stewart'
+┊   ┊ 59┊        },
+┊   ┊ 60┊        content: 'Yep!',
+┊   ┊ 61┊        createdAt: '1514035700',
+┊   ┊ 62┊        type: 0,
+┊   ┊ 63┊        recipients: [
+┊   ┊ 64┊          {
+┊   ┊ 65┊            user: {
+┊   ┊ 66┊              id: '1',
+┊   ┊ 67┊              __typename: 'User',
+┊   ┊ 68┊            },
+┊   ┊ 69┊            message: {
+┊   ┊ 70┊              id: '1',
+┊   ┊ 71┊              __typename: 'Message',
+┊   ┊ 72┊              chat: {
+┊   ┊ 73┊                id: '1',
+┊   ┊ 74┊                __typename: 'Chat',
+┊   ┊ 75┊              },
+┊   ┊ 76┊            },
+┊   ┊ 77┊            __typename: 'Recipient',
+┊   ┊ 78┊            chat: {
+┊   ┊ 79┊              id: '1',
+┊   ┊ 80┊              __typename: 'Chat',
+┊   ┊ 81┊            },
+┊   ┊ 82┊            receivedAt: null,
+┊   ┊ 83┊            readAt: null
+┊   ┊ 84┊          }
+┊   ┊ 85┊        ],
+┊   ┊ 86┊        ownership: false
+┊   ┊ 87┊      }
+┊   ┊ 88┊    ],
+┊   ┊ 89┊  };
+┊   ┊ 90┊
+┊   ┊ 91┊  beforeEach(async(() => {
+┊   ┊ 92┊    TestBed.configureTestingModule({
+┊   ┊ 93┊      declarations: [
+┊   ┊ 94┊        ChatComponent,
+┊   ┊ 95┊        MessagesListComponent,
+┊   ┊ 96┊        MessageItemComponent,
+┊   ┊ 97┊        NewMessageComponent,
+┊   ┊ 98┊      ],
+┊   ┊ 99┊      imports: [
+┊   ┊100┊        MatToolbarModule,
+┊   ┊101┊        MatMenuModule,
+┊   ┊102┊        MatIconModule,
+┊   ┊103┊        MatButtonModule,
+┊   ┊104┊        MatListModule,
+┊   ┊105┊        MatGridListModule,
+┊   ┊106┊        FormsModule,
+┊   ┊107┊        SharedModule,
+┊   ┊108┊        HttpLinkModule,
+┊   ┊109┊        HttpClientTestingModule,
+┊   ┊110┊        RouterTestingModule
+┊   ┊111┊      ],
+┊   ┊112┊      providers: [
+┊   ┊113┊        ChatsService,
+┊   ┊114┊        Apollo,
+┊   ┊115┊        {
+┊   ┊116┊          provide: ActivatedRoute,
+┊   ┊117┊          useValue: { params: of({ id: chat.id }) }
+┊   ┊118┊        }
+┊   ┊119┊      ],
+┊   ┊120┊      schemas: [NO_ERRORS_SCHEMA]
+┊   ┊121┊    })
+┊   ┊122┊      .compileComponents();
+┊   ┊123┊
+┊   ┊124┊    httpMock = TestBed.get(HttpTestingController);
+┊   ┊125┊    httpLink = TestBed.get(HttpLink);
+┊   ┊126┊    apollo = TestBed.get(Apollo);
+┊   ┊127┊
+┊   ┊128┊    apollo.create({
+┊   ┊129┊      link: httpLink.create(<Options>{ uri: 'http://localhost:3000/graphql' }),
+┊   ┊130┊      cache: new InMemoryCache({
+┊   ┊131┊        dataIdFromObject: (object: any) => {
+┊   ┊132┊          switch (object.__typename) {
+┊   ┊133┊            case 'Message': return `${object.chat.id}:${object.id}`; // use `chatId` prefix and `messageId` as the primary key
+┊   ┊134┊            default: return defaultDataIdFromObject(object); // fall back to default handling
+┊   ┊135┊          }
+┊   ┊136┊        }
+┊   ┊137┊      }),
+┊   ┊138┊    });
+┊   ┊139┊  }));
+┊   ┊140┊
+┊   ┊141┊  beforeEach(() => {
+┊   ┊142┊    fixture = TestBed.createComponent(ChatComponent);
+┊   ┊143┊    component = fixture.componentInstance;
+┊   ┊144┊    fixture.detectChanges();
+┊   ┊145┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊146┊    req.flush({
+┊   ┊147┊      data: {
+┊   ┊148┊        chat
+┊   ┊149┊      }
+┊   ┊150┊    });
+┊   ┊151┊  });
+┊   ┊152┊
+┊   ┊153┊  it('should create', () => {
+┊   ┊154┊    expect(component).toBeTruthy();
+┊   ┊155┊  });
+┊   ┊156┊
+┊   ┊157┊  it('should display the chat', () => {
+┊   ┊158┊    fixture.whenStable().then(() => {
+┊   ┊159┊      fixture.detectChanges();
+┊   ┊160┊      el = fixture.debugElement;
+┊   ┊161┊      expect(el.query(By.css(`app-toolbar > mat-toolbar > div > div`)).nativeElement.textContent).toContain(chat.name);
+┊   ┊162┊      for (let i = 0; i < chat.messages.length; i++) {
+┊   ┊163┊        expect(el.query(By.css(`app-messages-list > mat-list > mat-list-item:nth-child(${i + 1}) > div > app-message-item > div`))
+┊   ┊164┊          .nativeElement.textContent).toContain(chat.messages[i].content);
+┊   ┊165┊      }
+┊   ┊166┊    });
+┊   ┊167┊
+┊   ┊168┊    httpMock.verify();
+┊   ┊169┊  });
+┊   ┊170┊});
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step7.md) | [Next Step >](step9.md) |
|:--------------------------------|--------------------------------:|

[}]: #
