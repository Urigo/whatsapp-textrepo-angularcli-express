# Step 11: Messages and chats removal

[//]: # (head-end)


## Client

Since we're now familiar with the way mutations work, it's time to add messages and chats removal to our list of features!
Since the most annoying part is going to be dealing with the user interface (because a multiple selection started by a press event is involved), I created an Angular directive to ease the process.
Let's start by installing it:

    $ npm install ngx-selectable-list

Now let's import it:

[{]: <helper> (diffStep "7.1" files="^\(?!package.json$\).*" module="client")

#### Step 7.1: Add SelectableListModule

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;chats-lister.module.ts
```diff
@@ -11,6 +11,7 @@
 ┊11┊11┊import {ChatsListComponent} from './components/chats-list/chats-list.component';
 ┊12┊12┊import {TruncateModule} from 'ng2-truncate';
 ┊13┊13┊import {SharedModule} from '../shared/shared.module';
+┊  ┊14┊import {NgxSelectableListModule} from 'ngx-selectable-list';
 ┊14┊15┊
 ┊15┊16┊const routes: Routes = [
 ┊16┊17┊  {path: '', redirectTo: 'chats', pathMatch: 'full'},
```
```diff
@@ -40,6 +41,7 @@
 ┊40┊41┊    TruncateModule,
 ┊41┊42┊    // Feature modules
 ┊42┊43┊    SharedModule,
+┊  ┊44┊    NgxSelectableListModule,
 ┊43┊45┊  ],
 ┊44┊46┊  providers: [
 ┊45┊47┊    ChatsService,
```

[}]: #

Let's create the mutations:

[{]: <helper> (diffStep "7.2" files="src/graphql" module="client")

#### Step 7.2: Remove messages and chats

##### Added src&#x2F;graphql&#x2F;removeAllMessages.mutation.ts
```diff
@@ -0,0 +1,8 @@
+┊ ┊1┊import gql from 'graphql-tag';
+┊ ┊2┊
+┊ ┊3┊// We use the gql tag to parse our query string into a query document
+┊ ┊4┊export const removeAllMessagesMutation = gql`
+┊ ┊5┊  mutation RemoveAllMessages($chatId: ID!, $all: Boolean) {
+┊ ┊6┊    removeMessages(chatId: $chatId, all: $all)
+┊ ┊7┊  }
+┊ ┊8┊`;
```

##### Added src&#x2F;graphql&#x2F;removeChat.mutation.ts
```diff
@@ -0,0 +1,8 @@
+┊ ┊1┊import gql from 'graphql-tag';
+┊ ┊2┊
+┊ ┊3┊// We use the gql tag to parse our query string into a query document
+┊ ┊4┊export const removeChatMutation = gql`
+┊ ┊5┊  mutation RemoveChat($chatId: ID!) {
+┊ ┊6┊    removeChat(chatId: $chatId)
+┊ ┊7┊  }
+┊ ┊8┊`;
```

##### Added src&#x2F;graphql&#x2F;removeMessages.mutation.ts
```diff
@@ -0,0 +1,8 @@
+┊ ┊1┊import gql from 'graphql-tag';
+┊ ┊2┊
+┊ ┊3┊// We use the gql tag to parse our query string into a query document
+┊ ┊4┊export const removeMessagesMutation = gql`
+┊ ┊5┊  mutation RemoveMessages($chatId: ID!, $messageIds: [ID]) {
+┊ ┊6┊    removeMessages(chatId: $chatId, messageIds: $messageIds)
+┊ ┊7┊  }
+┊ ┊8┊`;
```

[}]: #

Now let's update our service:

[{]: <helper> (diffStep "7.2" files="chats.service.ts" module="client")

#### Step 7.2: Remove messages and chats

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -3,9 +3,13 @@
 ┊ 3┊ 3┊import {Apollo} from 'apollo-angular';
 ┊ 4┊ 4┊import {Injectable} from '@angular/core';
 ┊ 5┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊ 6┊  ┊import {AddMessage, GetChat, GetChats} from '../../types';
+┊  ┊ 6┊import {AddMessage, GetChat, GetChats, RemoveAllMessages, RemoveChat, RemoveMessages} from '../../types';
 ┊ 7┊ 7┊import {getChatQuery} from '../../graphql/getChat.query';
 ┊ 8┊ 8┊import {addMessageMutation} from '../../graphql/addMessage.mutation';
+┊  ┊ 9┊import {removeChatMutation} from '../../graphql/removeChat.mutation';
+┊  ┊10┊import {DocumentNode} from 'graphql';
+┊  ┊11┊import {removeAllMessagesMutation} from '../../graphql/removeAllMessages.mutation';
+┊  ┊12┊import {removeMessagesMutation} from '../../graphql/removeMessages.mutation';
 ┊ 9┊13┊
 ┊10┊14┊@Injectable()
 ┊11┊15┊export class ChatsService {
```
```diff
@@ -88,4 +92,104 @@
 ┊ 88┊ 92┊      },
 ┊ 89┊ 93┊    });
 ┊ 90┊ 94┊  }
+┊   ┊ 95┊
+┊   ┊ 96┊  removeChat(chatId: string) {
+┊   ┊ 97┊    return this.apollo.mutate({
+┊   ┊ 98┊      mutation: removeChatMutation,
+┊   ┊ 99┊      variables: <RemoveChat.Variables>{
+┊   ┊100┊        chatId,
+┊   ┊101┊      },
+┊   ┊102┊      update: (store, { data: { removeChat } }) => {
+┊   ┊103┊        // Read the data from our cache for this query.
+┊   ┊104┊        const {chats}: GetChats.Query = store.readQuery({
+┊   ┊105┊          query: getChatsQuery,
+┊   ┊106┊          variables: <GetChats.Variables>{
+┊   ┊107┊            amount: this.messagesAmount,
+┊   ┊108┊          },
+┊   ┊109┊        });
+┊   ┊110┊        // Remove the chat (mutable)
+┊   ┊111┊        for (const index of chats.keys()) {
+┊   ┊112┊          if (chats[index].id === removeChat) {
+┊   ┊113┊            chats.splice(index, 1);
+┊   ┊114┊          }
+┊   ┊115┊        }
+┊   ┊116┊        // Write our data back to the cache.
+┊   ┊117┊        store.writeQuery({
+┊   ┊118┊          query: getChatsQuery,
+┊   ┊119┊          variables: <GetChats.Variables>{
+┊   ┊120┊            amount: this.messagesAmount,
+┊   ┊121┊          },
+┊   ┊122┊          data: {
+┊   ┊123┊            chats,
+┊   ┊124┊          },
+┊   ┊125┊        });
+┊   ┊126┊      },
+┊   ┊127┊    });
+┊   ┊128┊  }
+┊   ┊129┊
+┊   ┊130┊  removeMessages(chatId: string, messages: GetChat.Messages[], messageIdsOrAll: string[] | boolean) {
+┊   ┊131┊    let variables: RemoveMessages.Variables | RemoveAllMessages.Variables;
+┊   ┊132┊    let ids: string[] = [];
+┊   ┊133┊    let mutation: DocumentNode;
+┊   ┊134┊
+┊   ┊135┊    if (typeof messageIdsOrAll === 'boolean') {
+┊   ┊136┊      variables = {chatId, all: messageIdsOrAll} as RemoveAllMessages.Variables;
+┊   ┊137┊      ids = messages.map(message => message.id);
+┊   ┊138┊      mutation = removeAllMessagesMutation;
+┊   ┊139┊    } else {
+┊   ┊140┊      variables = {chatId, messageIds: messageIdsOrAll} as RemoveMessages.Variables;
+┊   ┊141┊      ids = messageIdsOrAll;
+┊   ┊142┊      mutation = removeMessagesMutation;
+┊   ┊143┊    }
+┊   ┊144┊
+┊   ┊145┊    return this.apollo.mutate(<MutationOptions>{
+┊   ┊146┊      mutation,
+┊   ┊147┊      variables,
+┊   ┊148┊      update: (store, { data: { removeMessages } }: {data: RemoveMessages.Mutation | RemoveAllMessages.Mutation}) => {
+┊   ┊149┊        // Update the messages cache
+┊   ┊150┊        {
+┊   ┊151┊          // Read the data from our cache for this query.
+┊   ┊152┊          const {chat}: GetChat.Query = store.readQuery({
+┊   ┊153┊            query: getChatQuery, variables: {
+┊   ┊154┊              chatId,
+┊   ┊155┊            }
+┊   ┊156┊          });
+┊   ┊157┊          // Remove the messages (mutable)
+┊   ┊158┊          removeMessages.forEach(messageId => {
+┊   ┊159┊            for (const index of chat.messages.keys()) {
+┊   ┊160┊              if (chat.messages[index].id === messageId) {
+┊   ┊161┊                chat.messages.splice(index, 1);
+┊   ┊162┊              }
+┊   ┊163┊            }
+┊   ┊164┊          });
+┊   ┊165┊          // Write our data back to the cache.
+┊   ┊166┊          store.writeQuery({ query: getChatQuery, data: {chat} });
+┊   ┊167┊        }
+┊   ┊168┊        // Update last message cache
+┊   ┊169┊        {
+┊   ┊170┊          // Read the data from our cache for this query.
+┊   ┊171┊          const {chats}: GetChats.Query = store.readQuery({
+┊   ┊172┊            query: getChatsQuery,
+┊   ┊173┊            variables: <GetChats.Variables>{
+┊   ┊174┊              amount: this.messagesAmount,
+┊   ┊175┊            },
+┊   ┊176┊          });
+┊   ┊177┊          // Fix last message
+┊   ┊178┊          chats.find(chat => chat.id === chatId).messages = messages
+┊   ┊179┊            .filter(message => !ids.includes(message.id))
+┊   ┊180┊            .sort((a, b) => Number(b.createdAt) - Number(a.createdAt)) || [];
+┊   ┊181┊          // Write our data back to the cache.
+┊   ┊182┊          store.writeQuery({
+┊   ┊183┊            query: getChatsQuery,
+┊   ┊184┊            variables: <GetChats.Variables>{
+┊   ┊185┊              amount: this.messagesAmount,
+┊   ┊186┊            },
+┊   ┊187┊            data: {
+┊   ┊188┊              chats,
+┊   ┊189┊            },
+┊   ┊190┊          });
+┊   ┊191┊        }
+┊   ┊192┊      },
+┊   ┊193┊    });
+┊   ┊194┊  }
 ┊ 91┊195┊}
```

[}]: #

As you can see every time that we remove a message we also have to update the last message in the chats list.

Finally we can wire everything up into our components:

[{]: <helper> (diffStep "7.2" files="src/app/chat-viewer, src/app/chats-lister, src/app/shared" module="client")

#### Step 7.2: Remove messages and chats

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;chat-viewer.module.ts
```diff
@@ -11,6 +11,7 @@
 ┊11┊11┊import {MessageItemComponent} from './components/message-item/message-item.component';
 ┊12┊12┊import {NewMessageComponent} from './components/new-message/new-message.component';
 ┊13┊13┊import {SharedModule} from '../shared/shared.module';
+┊  ┊14┊import {NgxSelectableListModule} from 'ngx-selectable-list';
 ┊14┊15┊
 ┊15┊16┊const routes: Routes = [
 ┊16┊17┊  {
```
```diff
@@ -44,6 +45,7 @@
 ┊44┊45┊    FormsModule,
 ┊45┊46┊    // Feature modules
 ┊46┊47┊    SharedModule,
+┊  ┊48┊    NgxSelectableListModule,
 ┊47┊49┊  ],
 ┊48┊50┊  providers: [
 ┊49┊51┊    ChatsService,
```

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;components&#x2F;messages-list&#x2F;messages-list.component.ts
```diff
@@ -1,14 +1,17 @@
 ┊ 1┊ 1┊import {Component, Input} from '@angular/core';
 ┊ 2┊ 2┊import {GetChat} from '../../../../types';
+┊  ┊ 3┊import {SelectableListDirective} from 'ngx-selectable-list';
 ┊ 3┊ 4┊
 ┊ 4┊ 5┊@Component({
 ┊ 5┊ 6┊  selector: 'app-messages-list',
 ┊ 6┊ 7┊  template: `
 ┊ 7┊ 8┊    <mat-list>
 ┊ 8┊ 9┊      <mat-list-item *ngFor="let message of messages">
-┊ 9┊  ┊        <app-message-item [item]="message" [isGroup]="isGroup"></app-message-item>
+┊  ┊10┊        <app-message-item [item]="message" [isGroup]="isGroup"
+┊  ┊11┊                          libSelectableItem></app-message-item>
 ┊10┊12┊      </mat-list-item>
 ┊11┊13┊    </mat-list>
+┊  ┊14┊    <ng-content *ngIf="selectableListDirective.selecting"></ng-content>
 ┊12┊15┊  `,
 ┊13┊16┊  styleUrls: ['messages-list.component.scss'],
 ┊14┊17┊})
```
```diff
@@ -20,5 +23,5 @@
 ┊20┊23┊  @Input()
 ┊21┊24┊  isGroup: boolean;
 ┊22┊25┊
-┊23┊  ┊  constructor() {}
+┊  ┊26┊  constructor(public selectableListDirective: SelectableListDirective) {}
 ┊24┊27┊}
```

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -17,6 +17,7 @@
 ┊17┊17┊import {NewMessageComponent} from '../../components/new-message/new-message.component';
 ┊18┊18┊import {MessagesListComponent} from '../../components/messages-list/messages-list.component';
 ┊19┊19┊import {MessageItemComponent} from '../../components/message-item/message-item.component';
+┊  ┊20┊import {NgxSelectableListModule} from 'ngx-selectable-list';
 ┊20┊21┊
 ┊21┊22┊describe('ChatComponent', () => {
 ┊22┊23┊  let component: ChatComponent;
```
```diff
@@ -107,7 +108,8 @@
 ┊107┊108┊        SharedModule,
 ┊108┊109┊        HttpLinkModule,
 ┊109┊110┊        HttpClientTestingModule,
-┊110┊   ┊        RouterTestingModule
+┊   ┊111┊        RouterTestingModule,
+┊   ┊112┊        NgxSelectableListModule,
 ┊111┊113┊      ],
 ┊112┊114┊      providers: [
 ┊113┊115┊        ChatsService,
```

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -12,7 +12,10 @@
 ┊12┊12┊      <div class="title">{{ name }}</div>
 ┊13┊13┊    </app-toolbar>
 ┊14┊14┊    <div class="container">
-┊15┊  ┊      <app-messages-list [items]="messages" [isGroup]="isGroup"></app-messages-list>
+┊  ┊15┊      <app-messages-list [items]="messages" [isGroup]="isGroup"
+┊  ┊16┊                         libSelectableList="multiple_press" (multiple)="deleteMessages($event)">
+┊  ┊17┊        <app-confirm-selection #confirmSelection></app-confirm-selection>
+┊  ┊18┊      </app-messages-list>
 ┊16┊19┊      <app-new-message (newMessage)="addMessage($event)"></app-new-message>
 ┊17┊20┊    </div>
 ┊18┊21┊  `,
```
```diff
@@ -47,4 +50,8 @@
 ┊47┊50┊  addMessage(content: string) {
 ┊48┊51┊    this.chatsService.addMessage(this.chatId, content).subscribe();
 ┊49┊52┊  }
+┊  ┊53┊
+┊  ┊54┊  deleteMessages(messageIds: string[]) {
+┊  ┊55┊    this.chatsService.removeMessages(this.chatId, this.messages, messageIds).subscribe();
+┊  ┊56┊  }
 ┊50┊57┊}
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.ts
```diff
@@ -5,7 +5,7 @@
 ┊ 5┊ 5┊  selector: 'app-chat-item',
 ┊ 6┊ 6┊  template: `
 ┊ 7┊ 7┊    <div class="chat-row">
-┊ 8┊  ┊        <div class="chat-recipient" (click)="selectChat()">
+┊  ┊ 8┊        <div class="chat-recipient">
 ┊ 9┊ 9┊          <img *ngIf="chat.picture" [src]="chat.picture" width="48" height="48">
 ┊10┊10┊          <div>{{ chat.name }} [id: {{ chat.id }}]</div>
 ┊11┊11┊        </div>
```
```diff
@@ -18,11 +18,4 @@
 ┊18┊18┊  // tslint:disable-next-line:no-input-rename
 ┊19┊19┊  @Input('item')
 ┊20┊20┊  chat: GetChats.Chats;
-┊21┊  ┊
-┊22┊  ┊  @Output()
-┊23┊  ┊  select = new EventEmitter<string>();
-┊24┊  ┊
-┊25┊  ┊  selectChat() {
-┊26┊  ┊    this.select.emit(this.chat.id);
-┊27┊  ┊  }
 ┊28┊21┊}
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chats-list&#x2F;chats-list.component.ts
```diff
@@ -1,14 +1,17 @@
-┊ 1┊  ┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
 ┊ 2┊ 2┊import {GetChats} from '../../../../types';
+┊  ┊ 3┊import {SelectableListDirective} from 'ngx-selectable-list';
 ┊ 3┊ 4┊
 ┊ 4┊ 5┊@Component({
 ┊ 5┊ 6┊  selector: 'app-chats-list',
 ┊ 6┊ 7┊  template: `
 ┊ 7┊ 8┊    <mat-list>
 ┊ 8┊ 9┊      <mat-list-item *ngFor="let chat of chats">
-┊ 9┊  ┊        <app-chat-item [item]="chat" (select)="selectChat($event)"></app-chat-item>
+┊  ┊10┊        <app-chat-item [item]="chat"
+┊  ┊11┊                       libSelectableItem></app-chat-item>
 ┊10┊12┊      </mat-list-item>
 ┊11┊13┊    </mat-list>
+┊  ┊14┊    <ng-content *ngIf="selectableListDirective.selecting"></ng-content>
 ┊12┊15┊  `,
 ┊13┊16┊  styleUrls: ['chats-list.component.scss'],
 ┊14┊17┊})
```
```diff
@@ -17,12 +20,5 @@
 ┊17┊20┊  @Input('items')
 ┊18┊21┊  chats: GetChats.Chats[];
 ┊19┊22┊
-┊20┊  ┊  @Output()
-┊21┊  ┊  select = new EventEmitter<string>();
-┊22┊  ┊
-┊23┊  ┊  constructor() {}
-┊24┊  ┊
-┊25┊  ┊  selectChat(id: string) {
-┊26┊  ┊    this.select.emit(id);
-┊27┊  ┊  }
+┊  ┊23┊  constructor(public selectableListDirective: SelectableListDirective) {}
 ┊28┊24┊}
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -13,6 +13,7 @@
 ┊13┊13┊import {defaultDataIdFromObject, InMemoryCache} from 'apollo-cache-inmemory';
 ┊14┊14┊import {By} from '@angular/platform-browser';
 ┊15┊15┊import {RouterTestingModule} from '@angular/router/testing';
+┊  ┊16┊import {NgxSelectableListModule} from 'ngx-selectable-list';
 ┊16┊17┊
 ┊17┊18┊describe('ChatsComponent', () => {
 ┊18┊19┊  let component: ChatsComponent;
```
```diff
@@ -329,7 +330,8 @@
 ┊329┊330┊        TruncateModule,
 ┊330┊331┊        HttpLinkModule,
 ┊331┊332┊        HttpClientTestingModule,
-┊332┊   ┊        RouterTestingModule
+┊   ┊333┊        RouterTestingModule,
+┊   ┊334┊        NgxSelectableListModule,
 ┊333┊335┊      ],
 ┊334┊336┊      providers: [
 ┊335┊337┊        ChatsService,
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.ts
```diff
@@ -28,9 +28,13 @@
 ┊28┊28┊      </button>
 ┊29┊29┊    </mat-menu>
 ┊30┊30┊
-┊31┊  ┊    <app-chats-list [items]="chats$ | async" (select)="goToChat($event)"></app-chats-list>
+┊  ┊31┊    <app-chats-list [items]="chats$ | async"
+┊  ┊32┊                    libSelectableList="both"
+┊  ┊33┊                    (single)="goToChat($event)" (multiple)="deleteChats($event)" (isSelecting)="isSelecting = $event">
+┊  ┊34┊      <app-confirm-selection #confirmSelection></app-confirm-selection>
+┊  ┊35┊    </app-chats-list>
 ┊32┊36┊
-┊33┊  ┊    <button class="chat-button" mat-fab color="primary">
+┊  ┊37┊    <button *ngIf="!isSelecting" class="chat-button" mat-fab color="primary">
 ┊34┊38┊      <mat-icon aria-label="Icon-button with a + icon">add</mat-icon>
 ┊35┊39┊    </button>
 ┊36┊40┊  `,
```
```diff
@@ -38,6 +42,7 @@
 ┊38┊42┊})
 ┊39┊43┊export class ChatsComponent implements OnInit {
 ┊40┊44┊  chats$: Observable<GetChats.Chats[]>;
+┊  ┊45┊  isSelecting = false;
 ┊41┊46┊
 ┊42┊47┊  constructor(private chatsService: ChatsService,
 ┊43┊48┊              private router: Router) {
```
```diff
@@ -50,4 +55,10 @@
 ┊50┊55┊  goToChat(chatId: string) {
 ┊51┊56┊    this.router.navigate(['/chat', chatId]);
 ┊52┊57┊  }
+┊  ┊58┊
+┊  ┊59┊  deleteChats(chatIds: string[]) {
+┊  ┊60┊    chatIds.forEach(chatId => {
+┊  ┊61┊      this.chatsService.removeChat(chatId).subscribe();
+┊  ┊62┊    });
+┊  ┊63┊  }
 ┊53┊64┊}
```

##### Added src&#x2F;app&#x2F;shared&#x2F;components&#x2F;confirm-selection&#x2F;confirm-selection.component.scss
```diff
@@ -0,0 +1,6 @@
+┊ ┊1┊:host {
+┊ ┊2┊  display: block;
+┊ ┊3┊  position: absolute;
+┊ ┊4┊  bottom: 5vw;
+┊ ┊5┊  right: 5vw;
+┊ ┊6┊}
```

##### Added src&#x2F;app&#x2F;shared&#x2F;components&#x2F;confirm-selection&#x2F;confirm-selection.component.ts
```diff
@@ -0,0 +1,21 @@
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 2┊
+┊  ┊ 3┊@Component({
+┊  ┊ 4┊  selector: 'app-confirm-selection',
+┊  ┊ 5┊  template: `
+┊  ┊ 6┊    <button mat-fab color="primary" (click)="handleClick()">
+┊  ┊ 7┊      <mat-icon aria-label="Icon-button">{{ icon }}</mat-icon>
+┊  ┊ 8┊    </button>
+┊  ┊ 9┊  `,
+┊  ┊10┊  styleUrls: ['./confirm-selection.component.scss'],
+┊  ┊11┊})
+┊  ┊12┊export class ConfirmSelectionComponent {
+┊  ┊13┊  @Input()
+┊  ┊14┊  icon = 'delete';
+┊  ┊15┊  @Output()
+┊  ┊16┊  emitClick = new EventEmitter<null>();
+┊  ┊17┊
+┊  ┊18┊  handleClick() {
+┊  ┊19┊    this.emitClick.emit();
+┊  ┊20┊  }
+┊  ┊21┊}
```

##### Changed src&#x2F;app&#x2F;shared&#x2F;shared.module.ts
```diff
@@ -1,19 +1,23 @@
 ┊ 1┊ 1┊import {BrowserModule} from '@angular/platform-browser';
 ┊ 2┊ 2┊import {NgModule} from '@angular/core';
 ┊ 3┊ 3┊
-┊ 4┊  ┊import {MatToolbarModule} from '@angular/material';
+┊  ┊ 4┊import {MatButtonModule, MatIconModule, MatToolbarModule} from '@angular/material';
 ┊ 5┊ 5┊import {ToolbarComponent} from './components/toolbar/toolbar.component';
 ┊ 6┊ 6┊import {FormsModule} from '@angular/forms';
 ┊ 7┊ 7┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 8┊import {ConfirmSelectionComponent} from './components/confirm-selection/confirm-selection.component';
 ┊ 8┊ 9┊
 ┊ 9┊10┊@NgModule({
 ┊10┊11┊  declarations: [
 ┊11┊12┊    ToolbarComponent,
+┊  ┊13┊    ConfirmSelectionComponent,
 ┊12┊14┊  ],
 ┊13┊15┊  imports: [
 ┊14┊16┊    BrowserModule,
 ┊15┊17┊    // Material
 ┊16┊18┊    MatToolbarModule,
+┊  ┊19┊    MatIconModule,
+┊  ┊20┊    MatButtonModule,
 ┊17┊21┊    // Animations
 ┊18┊22┊    BrowserAnimationsModule,
 ┊19┊23┊    // Forms
```
```diff
@@ -22,6 +26,7 @@
 ┊22┊26┊  providers: [],
 ┊23┊27┊  exports: [
 ┊24┊28┊    ToolbarComponent,
+┊  ┊29┊    ConfirmSelectionComponent,
 ┊25┊30┊  ],
 ┊26┊31┊})
 ┊27┊32┊export class SharedModule {
```

[}]: #

We also created a `ConfirmSelectionComponent` to use for content projection, since our selectable list directive will be able to listen to its events.
The selectable list directive supports much more different use cases, for info please read the documentation.

As you can see `ngx-selectable-list` takes care of most of the boilerplate, giving us the freedom to concentrate on the actual code.

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step10.md) | [Next Step >](step12.md) |
|:--------------------------------|--------------------------------:|

[}]: #
