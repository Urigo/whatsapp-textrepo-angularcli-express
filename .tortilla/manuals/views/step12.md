# Step 12: Chats creation

[//]: # (head-end)


We still cannot create new chats or groups, so let's implement it.

We're going to create a `ChatsCreation` module, with a `NewChat` and a `NewGroup` containers, along with several presentational components.

We're going to make use of the selectable list directive once again, to ease selecting the users when we're creating a new group.
You should also notice that we are looking for existing chats before creating a new one: if it already exists we're are simply redirecting to that chat instead of creating a new one (the server wouldn't allow that anyway and it will simply
return the chat id).

[{]: <helper> (diffStep "8.1" files="src/graphql/" module="client")

#### Step 8.1: New chats and groups

##### Added src&#x2F;graphql&#x2F;addChat.mutation.ts
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const addChatMutation = gql`
+┊  ┊ 6┊  mutation AddChat($recipientId: ID!) {
+┊  ┊ 7┊    addChat(recipientId: $recipientId) {
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

##### Added src&#x2F;graphql&#x2F;addGroup.mutation.ts
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const addGroupMutation = gql`
+┊  ┊ 6┊  mutation AddGroup($recipientIds: [ID!]!, $groupName: String!) {
+┊  ┊ 7┊    addGroup(recipientIds: $recipientIds, groupName: $groupName) {
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

##### Added src&#x2F;graphql&#x2F;getUsers.query.ts
```diff
@@ -0,0 +1,12 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊
+┊  ┊ 3┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 4┊export const getUsersQuery = gql`
+┊  ┊ 5┊  query GetUsers {
+┊  ┊ 6┊    users {
+┊  ┊ 7┊      id,
+┊  ┊ 8┊      name,
+┊  ┊ 9┊      picture,
+┊  ┊10┊    }
+┊  ┊11┊  }
+┊  ┊12┊`;
```

[}]: #

After creating the mutations we should run the generator to create the corresponding types and services:

    yarn generator

Let's jump straight into `ChatsService` and add few new methods:

[{]: <helper> (diffStep "8.1" files="src/app/services/chats.service.ts" module="client")

#### Step 8.1: New chats and groups

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,5 +1,7 @@
 ┊1┊1┊import {map} from 'rxjs/operators';
 ┊2┊2┊import {Injectable} from '@angular/core';
+┊ ┊3┊import {Observable} from 'rxjs';
+┊ ┊4┊import {QueryRef} from 'apollo-angular';
 ┊3┊5┊import {
 ┊4┊6┊  GetChatsGQL,
 ┊5┊7┊  GetChatGQL,
```
```diff
@@ -7,6 +9,9 @@
 ┊ 7┊ 9┊  RemoveChatGQL,
 ┊ 8┊10┊  RemoveMessagesGQL,
 ┊ 9┊11┊  RemoveAllMessagesGQL,
+┊  ┊12┊  GetUsersGQL,
+┊  ┊13┊  AddChatGQL,
+┊  ┊14┊  AddGroupGQL,
 ┊10┊15┊  AddMessage,
 ┊11┊16┊  GetChats,
 ┊12┊17┊  GetChat,
```
```diff
@@ -15,9 +20,14 @@
 ┊15┊20┊} from '../../graphql';
 ┊16┊21┊import { DataProxy } from 'apollo-cache';
 ┊17┊22┊
+┊  ┊23┊const currentUserId = '1';
+┊  ┊24┊
 ┊18┊25┊@Injectable()
 ┊19┊26┊export class ChatsService {
 ┊20┊27┊  messagesAmount = 3;
+┊  ┊28┊  getChatsWq: QueryRef<GetChats.Query, GetChats.Variables>;
+┊  ┊29┊  chats$: Observable<GetChats.Chats[]>;
+┊  ┊30┊  chats: GetChats.Chats[];
 ┊21┊31┊
 ┊22┊32┊  constructor(
 ┊23┊33┊    private getChatsGQL: GetChatsGQL,
```
```diff
@@ -26,17 +36,21 @@
 ┊26┊36┊    private removeChatGQL: RemoveChatGQL,
 ┊27┊37┊    private removeMessagesGQL: RemoveMessagesGQL,
 ┊28┊38┊    private removeAllMessagesGQL: RemoveAllMessagesGQL,
-┊29┊  ┊  ) {}
-┊30┊  ┊
-┊31┊  ┊  getChats() {
-┊32┊  ┊    const query = this.getChatsGQL.watch({
+┊  ┊39┊    private getUsersGQL: GetUsersGQL,
+┊  ┊40┊    private addChatGQL: AddChatGQL,
+┊  ┊41┊    private addGroupGQL: AddGroupGQL
+┊  ┊42┊  ) {
+┊  ┊43┊    this.getChatsWq = this.getChatsGQL.watch({
 ┊33┊44┊      amount: this.messagesAmount,
 ┊34┊45┊    });
-┊35┊  ┊    const chats$ = query.valueChanges.pipe(
+┊  ┊46┊    this.chats$ = this.getChatsWq.valueChanges.pipe(
 ┊36┊47┊      map((result) => result.data.chats)
 ┊37┊48┊    );
+┊  ┊49┊    this.chats$.subscribe(chats => this.chats = chats);
+┊  ┊50┊  }
 ┊38┊51┊
-┊39┊  ┊    return {query, chats$};
+┊  ┊52┊  getChats() {
+┊  ┊53┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊40┊54┊  }
 ┊41┊55┊
 ┊42┊56┊  getChat(chatId: string) {
```
```diff
@@ -209,4 +223,83 @@
 ┊209┊223┊      }, options);
 ┊210┊224┊    }
 ┊211┊225┊  }
+┊   ┊226┊
+┊   ┊227┊  getUsers() {
+┊   ┊228┊    const query = this.getUsersGQL.watch();
+┊   ┊229┊    const users$ = query.valueChanges.pipe(
+┊   ┊230┊      map((result) => result.data.users)
+┊   ┊231┊    );
+┊   ┊232┊
+┊   ┊233┊    return {query, users$};
+┊   ┊234┊  }
+┊   ┊235┊
+┊   ┊236┊  // Checks if the chat is listed for the current user and returns the id
+┊   ┊237┊  getChatId(recipientId: string) {
+┊   ┊238┊    const _chat = this.chats.find(chat => {
+┊   ┊239┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === currentUserId) &&
+┊   ┊240┊        !!chat.allTimeMembers.find(user => user.id === recipientId);
+┊   ┊241┊    });
+┊   ┊242┊    return _chat ? _chat.id : false;
+┊   ┊243┊  }
+┊   ┊244┊
+┊   ┊245┊  addChat(recipientId: string) {
+┊   ┊246┊    return this.addChatGQL.mutate(
+┊   ┊247┊      {
+┊   ┊248┊        recipientId,
+┊   ┊249┊      }, {
+┊   ┊250┊        update: (store, { data: { addChat } }) => {
+┊   ┊251┊          // Read the data from our cache for this query.
+┊   ┊252┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊253┊            query: this.getChatsGQL.document,
+┊   ┊254┊            variables: {
+┊   ┊255┊              amount: this.messagesAmount,
+┊   ┊256┊            },
+┊   ┊257┊          });
+┊   ┊258┊          // Add our comment from the mutation to the end.
+┊   ┊259┊          chats.push(addChat);
+┊   ┊260┊          // Write our data back to the cache.
+┊   ┊261┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊262┊            query: this.getChatsGQL.document,
+┊   ┊263┊            variables: {
+┊   ┊264┊              amount: this.messagesAmount,
+┊   ┊265┊            },
+┊   ┊266┊            data: {
+┊   ┊267┊              chats,
+┊   ┊268┊            },
+┊   ┊269┊          });
+┊   ┊270┊        },
+┊   ┊271┊      }
+┊   ┊272┊    );
+┊   ┊273┊  }
+┊   ┊274┊
+┊   ┊275┊  addGroup(recipientIds: string[], groupName: string) {
+┊   ┊276┊    return this.addGroupGQL.mutate(
+┊   ┊277┊      {
+┊   ┊278┊        recipientIds,
+┊   ┊279┊        groupName,
+┊   ┊280┊      }, {
+┊   ┊281┊        update: (store, { data: { addGroup } }) => {
+┊   ┊282┊          // Read the data from our cache for this query.
+┊   ┊283┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊284┊            query: this.getChatsGQL.document,
+┊   ┊285┊            variables: {
+┊   ┊286┊              amount: this.messagesAmount,
+┊   ┊287┊            },
+┊   ┊288┊          });
+┊   ┊289┊          // Add our comment from the mutation to the end.
+┊   ┊290┊          chats.push(addGroup);
+┊   ┊291┊          // Write our data back to the cache.
+┊   ┊292┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊293┊            query: this.getChatsGQL.document,
+┊   ┊294┊            variables: {
+┊   ┊295┊              amount: this.messagesAmount,
+┊   ┊296┊            },
+┊   ┊297┊            data: {
+┊   ┊298┊              chats,
+┊   ┊299┊            },
+┊   ┊300┊          });
+┊   ┊301┊        },
+┊   ┊302┊      }
+┊   ┊303┊    );
+┊   ┊304┊  }
 ┊212┊305┊}
```

[}]: #

Okay, now since we got the GraphQL part ready, we're going to focus on the component.

First, space to add a new group:

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/containers/new-group/new-group.component.ts, src/app/chats-creation/containers/new-group/new-group.component.scss" module="client")

#### Step 8.1: New chats and groups

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-group&#x2F;new-group.component.scss


##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-group&#x2F;new-group.component.ts
```diff
@@ -0,0 +1,60 @@
+┊  ┊ 1┊import {Component, OnInit} from '@angular/core';
+┊  ┊ 2┊import {Location} from '@angular/common';
+┊  ┊ 3┊import {Router} from '@angular/router';
+┊  ┊ 4┊import {AddGroup, GetUsers} from '../../../../graphql';
+┊  ┊ 5┊import {ChatsService} from '../../../services/chats.service';
+┊  ┊ 6┊
+┊  ┊ 7┊@Component({
+┊  ┊ 8┊  template: `
+┊  ┊ 9┊    <app-toolbar>
+┊  ┊10┊      <button class="navigation" mat-button (click)="goBack()">
+┊  ┊11┊        <mat-icon aria-label="Icon-button with an arrow back icon">arrow_back</mat-icon>
+┊  ┊12┊      </button>
+┊  ┊13┊      <div class="title">New group</div>
+┊  ┊14┊    </app-toolbar>
+┊  ┊15┊
+┊  ┊16┊    <app-users-list *ngIf="!recipientIds.length" [items]="users"
+┊  ┊17┊                    libSelectableList="multiple_tap" (multiple)="selectUsers($event)">
+┊  ┊18┊      <app-confirm-selection #confirmSelection icon="arrow_forward"></app-confirm-selection>
+┊  ┊19┊    </app-users-list>
+┊  ┊20┊    <app-new-group-details *ngIf="recipientIds.length" [users]="getSelectedUsers()"
+┊  ┊21┊                           (groupDetails)="addGroup($event)"></app-new-group-details>
+┊  ┊22┊  `,
+┊  ┊23┊  styleUrls: ['new-group.component.scss'],
+┊  ┊24┊})
+┊  ┊25┊export class NewGroupComponent implements OnInit {
+┊  ┊26┊  users: GetUsers.Users[];
+┊  ┊27┊  recipientIds: string[] = [];
+┊  ┊28┊
+┊  ┊29┊  constructor(private router: Router,
+┊  ┊30┊              private location: Location,
+┊  ┊31┊              private chatsService: ChatsService) {}
+┊  ┊32┊
+┊  ┊33┊  ngOnInit () {
+┊  ┊34┊    this.chatsService.getUsers().users$.subscribe(users => this.users = users);
+┊  ┊35┊  }
+┊  ┊36┊
+┊  ┊37┊  goBack() {
+┊  ┊38┊    if (this.recipientIds.length) {
+┊  ┊39┊      this.recipientIds = [];
+┊  ┊40┊    } else {
+┊  ┊41┊      this.location.back();
+┊  ┊42┊    }
+┊  ┊43┊  }
+┊  ┊44┊
+┊  ┊45┊  selectUsers(recipientIds: string[]) {
+┊  ┊46┊    this.recipientIds = recipientIds;
+┊  ┊47┊  }
+┊  ┊48┊
+┊  ┊49┊  getSelectedUsers() {
+┊  ┊50┊    return this.users.filter(user => this.recipientIds.includes(user.id));
+┊  ┊51┊  }
+┊  ┊52┊
+┊  ┊53┊  addGroup(groupName: string) {
+┊  ┊54┊    if (groupName && this.recipientIds.length) {
+┊  ┊55┊      this.chatsService.addGroup(this.recipientIds, groupName).subscribe(({data: {addGroup: {id}}}: { data: AddGroup.Mutation }) => {
+┊  ┊56┊        this.router.navigate(['/chat', id]);
+┊  ┊57┊      });
+┊  ┊58┊    }
+┊  ┊59┊  }
+┊  ┊60┊}
```

[}]: #

Next, a new chat view:

src/app/chats-creation/containers/new-chat/new-chat.component.scss, src/app/chats-creation/containers/new-chat/new-chat.component.ts

[{]: <helper> (diffStep "8.1" files="src/app/app.module.ts, src/app/chats-creation, src/app/services" module="client")

#### Step 8.1: New chats and groups

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -7,6 +7,7 @@
 ┊ 7┊ 7┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
 ┊ 8┊ 8┊import {RouterModule, Routes} from '@angular/router';
 ┊ 9┊ 9┊import {ChatViewerModule} from './chat-viewer/chat-viewer.module';
+┊  ┊10┊import {ChatsCreationModule} from './chats-creation/chats-creation.module';
 ┊10┊11┊const routes: Routes = [];
 ┊11┊12┊
 ┊12┊13┊@NgModule({
```
```diff
@@ -22,6 +23,7 @@
 ┊22┊23┊    // Feature modules
 ┊23┊24┊    ChatsListerModule,
 ┊24┊25┊    ChatViewerModule,
+┊  ┊26┊    ChatsCreationModule,
 ┊25┊27┊  ],
 ┊26┊28┊  providers: [],
 ┊27┊29┊  bootstrap: [AppComponent]
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;chats-creation.module.ts
```diff
@@ -0,0 +1,60 @@
+┊  ┊ 1┊import { BrowserModule } from '@angular/platform-browser';
+┊  ┊ 2┊import { NgModule } from '@angular/core';
+┊  ┊ 3┊
+┊  ┊ 4┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 5┊import {
+┊  ┊ 6┊  MatButtonModule, MatFormFieldModule, MatGridListModule, MatIconModule, MatInputModule, MatListModule, MatMenuModule,
+┊  ┊ 7┊  MatToolbarModule
+┊  ┊ 8┊} from '@angular/material';
+┊  ┊ 9┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊10┊import {FormsModule} from '@angular/forms';
+┊  ┊11┊import {ChatsService} from '../services/chats.service';
+┊  ┊12┊import {UserItemComponent} from './components/user-item/user-item.component';
+┊  ┊13┊import {UsersListComponent} from './components/users-list/users-list.component';
+┊  ┊14┊import {NewGroupComponent} from './containers/new-group/new-group.component';
+┊  ┊15┊import {NewChatComponent} from './containers/new-chat/new-chat.component';
+┊  ┊16┊import {NewGroupDetailsComponent} from './components/new-group-details/new-group-details.component';
+┊  ┊17┊import {SharedModule} from '../shared/shared.module';
+┊  ┊18┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊19┊
+┊  ┊20┊const routes: Routes = [
+┊  ┊21┊  {path: 'new-chat', component: NewChatComponent},
+┊  ┊22┊  {path: 'new-group', component: NewGroupComponent},
+┊  ┊23┊];
+┊  ┊24┊
+┊  ┊25┊@NgModule({
+┊  ┊26┊  declarations: [
+┊  ┊27┊    NewChatComponent,
+┊  ┊28┊    UsersListComponent,
+┊  ┊29┊    NewGroupComponent,
+┊  ┊30┊    UserItemComponent,
+┊  ┊31┊    NewGroupDetailsComponent,
+┊  ┊32┊  ],
+┊  ┊33┊  imports: [
+┊  ┊34┊    BrowserModule,
+┊  ┊35┊    // Animations (for Material)
+┊  ┊36┊    BrowserAnimationsModule,
+┊  ┊37┊    // Material
+┊  ┊38┊    MatToolbarModule,
+┊  ┊39┊    MatMenuModule,
+┊  ┊40┊    MatIconModule,
+┊  ┊41┊    MatButtonModule,
+┊  ┊42┊    MatListModule,
+┊  ┊43┊    MatGridListModule,
+┊  ┊44┊    MatInputModule,
+┊  ┊45┊    MatFormFieldModule,
+┊  ┊46┊    MatGridListModule,
+┊  ┊47┊    // Routing
+┊  ┊48┊    RouterModule.forChild(routes),
+┊  ┊49┊    // Forms
+┊  ┊50┊    FormsModule,
+┊  ┊51┊    // Feature modules
+┊  ┊52┊    NgxSelectableListModule,
+┊  ┊53┊    SharedModule,
+┊  ┊54┊  ],
+┊  ┊55┊  providers: [
+┊  ┊56┊    ChatsService,
+┊  ┊57┊  ],
+┊  ┊58┊})
+┊  ┊59┊export class ChatsCreationModule {
+┊  ┊60┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;new-group-details&#x2F;new-group-details.component.scss
```diff
@@ -0,0 +1,25 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊}
+┊  ┊ 4┊
+┊  ┊ 5┊div {
+┊  ┊ 6┊  padding: 16px;
+┊  ┊ 7┊  mat-form-field {
+┊  ┊ 8┊    width: 100%;
+┊  ┊ 9┊  }
+┊  ┊10┊}
+┊  ┊11┊
+┊  ┊12┊.new-group {
+┊  ┊13┊  position: absolute;
+┊  ┊14┊  bottom: 5vw;
+┊  ┊15┊  right: 5vw;
+┊  ┊16┊}
+┊  ┊17┊
+┊  ┊18┊.users {
+┊  ┊19┊  display: flex;
+┊  ┊20┊  flex-flow: row wrap;
+┊  ┊21┊  img {
+┊  ┊22┊    flex: 0 1 8vh;
+┊  ┊23┊    height: 8vh;
+┊  ┊24┊  }
+┊  ┊25┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;new-group-details&#x2F;new-group-details.component.ts
```diff
@@ -0,0 +1,34 @@
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Component({
+┊  ┊ 5┊  selector: 'app-new-group-details',
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <div>
+┊  ┊ 8┊      <mat-form-field>
+┊  ┊ 9┊        <input matInput placeholder="Group name" [(ngModel)]="groupName">
+┊  ┊10┊      </mat-form-field>
+┊  ┊11┊    </div>
+┊  ┊12┊    <button [disabled]="!groupName" class="new-group" mat-fab color="primary" (click)="emitGroupDetails()">
+┊  ┊13┊      <mat-icon aria-label="Icon-button with a + icon">arrow_forward</mat-icon>
+┊  ┊14┊    </button>
+┊  ┊15┊    <div>Members</div>
+┊  ┊16┊    <div class="users">
+┊  ┊17┊      <img *ngFor="let user of users;" [src]="user.picture"/>
+┊  ┊18┊    </div>
+┊  ┊19┊  `,
+┊  ┊20┊  styleUrls: ['new-group-details.component.scss'],
+┊  ┊21┊})
+┊  ┊22┊export class NewGroupDetailsComponent {
+┊  ┊23┊  groupName: string;
+┊  ┊24┊  @Input()
+┊  ┊25┊  users: GetUsers.Users[];
+┊  ┊26┊  @Output()
+┊  ┊27┊  groupDetails = new EventEmitter<string>();
+┊  ┊28┊
+┊  ┊29┊  emitGroupDetails() {
+┊  ┊30┊    if (this.groupDetails) {
+┊  ┊31┊      this.groupDetails.emit(this.groupName);
+┊  ┊32┊    }
+┊  ┊33┊  }
+┊  ┊34┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;user-item&#x2F;user-item.component.scss
```diff
@@ -0,0 +1,28 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  width: 100%;
+┊  ┊ 4┊  height: 100%;
+┊  ┊ 5┊}
+┊  ┊ 6┊
+┊  ┊ 7┊button {
+┊  ┊ 8┊  padding: 0;
+┊  ┊ 9┊  display: flex;
+┊  ┊10┊  align-items: center;
+┊  ┊11┊  height: 100%;
+┊  ┊12┊  width: 100%;
+┊  ┊13┊  border: none;
+┊  ┊14┊
+┊  ┊15┊  div:first-of-type {
+┊  ┊16┊    display: flex;
+┊  ┊17┊    justify-content: center;
+┊  ┊18┊    align-items: center;
+┊  ┊19┊
+┊  ┊20┊    img {
+┊  ┊21┊      max-width: 100%;
+┊  ┊22┊    }
+┊  ┊23┊  }
+┊  ┊24┊
+┊  ┊25┊  div:nth-of-type(2) {
+┊  ┊26┊    padding-left: 16px;
+┊  ┊27┊  }
+┊  ┊28┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;user-item&#x2F;user-item.component.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Component({
+┊  ┊ 5┊  selector: 'app-user-item',
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <button mat-menu-item>
+┊  ┊ 8┊      <div>
+┊  ┊ 9┊        <img [src]="user.picture" *ngIf="user.picture">
+┊  ┊10┊      </div>
+┊  ┊11┊      <div>{{ user.name }}</div>
+┊  ┊12┊    </button>
+┊  ┊13┊  `,
+┊  ┊14┊  styleUrls: ['user-item.component.scss']
+┊  ┊15┊})
+┊  ┊16┊export class UserItemComponent {
+┊  ┊17┊  // tslint:disable-next-line:no-input-rename
+┊  ┊18┊  @Input('item')
+┊  ┊19┊  user: GetUsers.Users;
+┊  ┊20┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.scss
```diff
@@ -0,0 +1,3 @@
+┊ ┊1┊:host {
+┊ ┊2┊  display: block;
+┊ ┊3┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.ts
```diff
@@ -0,0 +1,24 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊import {SelectableListDirective} from 'ngx-selectable-list';
+┊  ┊ 4┊
+┊  ┊ 5┊@Component({
+┊  ┊ 6┊  selector: 'app-users-list',
+┊  ┊ 7┊  template: `
+┊  ┊ 8┊    <mat-list>
+┊  ┊ 9┊      <mat-list-item *ngFor="let user of users">
+┊  ┊10┊        <app-user-item [item]="user"
+┊  ┊11┊                       libSelectableItem></app-user-item>
+┊  ┊12┊      </mat-list-item>
+┊  ┊13┊    </mat-list>
+┊  ┊14┊    <ng-content *ngIf="selectableListDirective.selecting"></ng-content>
+┊  ┊15┊  `,
+┊  ┊16┊  styleUrls: ['users-list.component.scss'],
+┊  ┊17┊})
+┊  ┊18┊export class UsersListComponent {
+┊  ┊19┊  // tslint:disable-next-line:no-input-rename
+┊  ┊20┊  @Input('items')
+┊  ┊21┊  users: GetUsers.Users[];
+┊  ┊22┊
+┊  ┊23┊  constructor(public selectableListDirective: SelectableListDirective) {}
+┊  ┊24┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.scss
```diff
@@ -0,0 +1,23 @@
+┊  ┊ 1┊.new-group {
+┊  ┊ 2┊  display: flex;
+┊  ┊ 3┊  height: 8vh;
+┊  ┊ 4┊  align-items: center;
+┊  ┊ 5┊
+┊  ┊ 6┊  div:first-of-type {
+┊  ┊ 7┊    height: 8vh;
+┊  ┊ 8┊    width: 8vh;
+┊  ┊ 9┊    display: flex;
+┊  ┊10┊    justify-content: center;
+┊  ┊11┊    align-items: center;
+┊  ┊12┊
+┊  ┊13┊    mat-icon {
+┊  ┊14┊      height: 5vh;
+┊  ┊15┊      width: 5vh;
+┊  ┊16┊      font-size: 5vh;
+┊  ┊17┊    }
+┊  ┊18┊  }
+┊  ┊19┊
+┊  ┊20┊  div:nth-of-type(2) {
+┊  ┊21┊    padding: 16px;
+┊  ┊22┊  }
+┊  ┊23┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.ts
```diff
@@ -0,0 +1,59 @@
+┊  ┊ 1┊import {Component, OnInit} from '@angular/core';
+┊  ┊ 2┊import {Location} from '@angular/common';
+┊  ┊ 3┊import {Router} from '@angular/router';
+┊  ┊ 4┊import {AddChat, GetUsers} from '../../../../graphql';
+┊  ┊ 5┊import {ChatsService} from '../../../services/chats.service';
+┊  ┊ 6┊
+┊  ┊ 7┊@Component({
+┊  ┊ 8┊  template: `
+┊  ┊ 9┊    <app-toolbar>
+┊  ┊10┊      <button class="navigation" mat-button (click)="goBack()">
+┊  ┊11┊        <mat-icon aria-label="Icon-button with an arrow back icon">arrow_back</mat-icon>
+┊  ┊12┊      </button>
+┊  ┊13┊      <div class="title">New chat</div>
+┊  ┊14┊    </app-toolbar>
+┊  ┊15┊
+┊  ┊16┊    <div class="new-group" (click)="goToNewGroup()">
+┊  ┊17┊      <div>
+┊  ┊18┊        <mat-icon aria-label="Icon-button with a group add icon">group_add</mat-icon>
+┊  ┊19┊      </div>
+┊  ┊20┊      <div>New group</div>
+┊  ┊21┊    </div>
+┊  ┊22┊
+┊  ┊23┊    <app-users-list [items]="users"
+┊  ┊24┊                    libSelectableList="single" (single)="addChat($event)">
+┊  ┊25┊    </app-users-list>
+┊  ┊26┊  `,
+┊  ┊27┊  styleUrls: ['new-chat.component.scss'],
+┊  ┊28┊})
+┊  ┊29┊export class NewChatComponent implements OnInit {
+┊  ┊30┊  users: GetUsers.Users[];
+┊  ┊31┊
+┊  ┊32┊  constructor(private router: Router,
+┊  ┊33┊              private location: Location,
+┊  ┊34┊              private chatsService: ChatsService) {}
+┊  ┊35┊
+┊  ┊36┊  ngOnInit () {
+┊  ┊37┊    this.chatsService.getUsers().users$.subscribe(users => this.users = users);
+┊  ┊38┊  }
+┊  ┊39┊
+┊  ┊40┊  goBack() {
+┊  ┊41┊    this.location.back();
+┊  ┊42┊  }
+┊  ┊43┊
+┊  ┊44┊  goToNewGroup() {
+┊  ┊45┊    this.router.navigate(['/new-group']);
+┊  ┊46┊  }
+┊  ┊47┊
+┊  ┊48┊  addChat(recipientId: string) {
+┊  ┊49┊    const chatId = this.chatsService.getChatId(recipientId);
+┊  ┊50┊    if (chatId) {
+┊  ┊51┊      // Chat is already listed for the current user
+┊  ┊52┊      this.router.navigate(['/chat', chatId]);
+┊  ┊53┊    } else {
+┊  ┊54┊      this.chatsService.addChat(recipientId).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
+┊  ┊55┊        this.router.navigate(['/chat', id]);
+┊  ┊56┊      });
+┊  ┊57┊    }
+┊  ┊58┊  }
+┊  ┊59┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-group&#x2F;new-group.component.scss


##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-group&#x2F;new-group.component.ts
```diff
@@ -0,0 +1,60 @@
+┊  ┊ 1┊import {Component, OnInit} from '@angular/core';
+┊  ┊ 2┊import {Location} from '@angular/common';
+┊  ┊ 3┊import {Router} from '@angular/router';
+┊  ┊ 4┊import {AddGroup, GetUsers} from '../../../../graphql';
+┊  ┊ 5┊import {ChatsService} from '../../../services/chats.service';
+┊  ┊ 6┊
+┊  ┊ 7┊@Component({
+┊  ┊ 8┊  template: `
+┊  ┊ 9┊    <app-toolbar>
+┊  ┊10┊      <button class="navigation" mat-button (click)="goBack()">
+┊  ┊11┊        <mat-icon aria-label="Icon-button with an arrow back icon">arrow_back</mat-icon>
+┊  ┊12┊      </button>
+┊  ┊13┊      <div class="title">New group</div>
+┊  ┊14┊    </app-toolbar>
+┊  ┊15┊
+┊  ┊16┊    <app-users-list *ngIf="!recipientIds.length" [items]="users"
+┊  ┊17┊                    libSelectableList="multiple_tap" (multiple)="selectUsers($event)">
+┊  ┊18┊      <app-confirm-selection #confirmSelection icon="arrow_forward"></app-confirm-selection>
+┊  ┊19┊    </app-users-list>
+┊  ┊20┊    <app-new-group-details *ngIf="recipientIds.length" [users]="getSelectedUsers()"
+┊  ┊21┊                           (groupDetails)="addGroup($event)"></app-new-group-details>
+┊  ┊22┊  `,
+┊  ┊23┊  styleUrls: ['new-group.component.scss'],
+┊  ┊24┊})
+┊  ┊25┊export class NewGroupComponent implements OnInit {
+┊  ┊26┊  users: GetUsers.Users[];
+┊  ┊27┊  recipientIds: string[] = [];
+┊  ┊28┊
+┊  ┊29┊  constructor(private router: Router,
+┊  ┊30┊              private location: Location,
+┊  ┊31┊              private chatsService: ChatsService) {}
+┊  ┊32┊
+┊  ┊33┊  ngOnInit () {
+┊  ┊34┊    this.chatsService.getUsers().users$.subscribe(users => this.users = users);
+┊  ┊35┊  }
+┊  ┊36┊
+┊  ┊37┊  goBack() {
+┊  ┊38┊    if (this.recipientIds.length) {
+┊  ┊39┊      this.recipientIds = [];
+┊  ┊40┊    } else {
+┊  ┊41┊      this.location.back();
+┊  ┊42┊    }
+┊  ┊43┊  }
+┊  ┊44┊
+┊  ┊45┊  selectUsers(recipientIds: string[]) {
+┊  ┊46┊    this.recipientIds = recipientIds;
+┊  ┊47┊  }
+┊  ┊48┊
+┊  ┊49┊  getSelectedUsers() {
+┊  ┊50┊    return this.users.filter(user => this.recipientIds.includes(user.id));
+┊  ┊51┊  }
+┊  ┊52┊
+┊  ┊53┊  addGroup(groupName: string) {
+┊  ┊54┊    if (groupName && this.recipientIds.length) {
+┊  ┊55┊      this.chatsService.addGroup(this.recipientIds, groupName).subscribe(({data: {addGroup: {id}}}: { data: AddGroup.Mutation }) => {
+┊  ┊56┊        this.router.navigate(['/chat', id]);
+┊  ┊57┊      });
+┊  ┊58┊    }
+┊  ┊59┊  }
+┊  ┊60┊}
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,5 +1,7 @@
 ┊1┊1┊import {map} from 'rxjs/operators';
 ┊2┊2┊import {Injectable} from '@angular/core';
+┊ ┊3┊import {Observable} from 'rxjs';
+┊ ┊4┊import {QueryRef} from 'apollo-angular';
 ┊3┊5┊import {
 ┊4┊6┊  GetChatsGQL,
 ┊5┊7┊  GetChatGQL,
```
```diff
@@ -7,6 +9,9 @@
 ┊ 7┊ 9┊  RemoveChatGQL,
 ┊ 8┊10┊  RemoveMessagesGQL,
 ┊ 9┊11┊  RemoveAllMessagesGQL,
+┊  ┊12┊  GetUsersGQL,
+┊  ┊13┊  AddChatGQL,
+┊  ┊14┊  AddGroupGQL,
 ┊10┊15┊  AddMessage,
 ┊11┊16┊  GetChats,
 ┊12┊17┊  GetChat,
```
```diff
@@ -15,9 +20,14 @@
 ┊15┊20┊} from '../../graphql';
 ┊16┊21┊import { DataProxy } from 'apollo-cache';
 ┊17┊22┊
+┊  ┊23┊const currentUserId = '1';
+┊  ┊24┊
 ┊18┊25┊@Injectable()
 ┊19┊26┊export class ChatsService {
 ┊20┊27┊  messagesAmount = 3;
+┊  ┊28┊  getChatsWq: QueryRef<GetChats.Query, GetChats.Variables>;
+┊  ┊29┊  chats$: Observable<GetChats.Chats[]>;
+┊  ┊30┊  chats: GetChats.Chats[];
 ┊21┊31┊
 ┊22┊32┊  constructor(
 ┊23┊33┊    private getChatsGQL: GetChatsGQL,
```
```diff
@@ -26,17 +36,21 @@
 ┊26┊36┊    private removeChatGQL: RemoveChatGQL,
 ┊27┊37┊    private removeMessagesGQL: RemoveMessagesGQL,
 ┊28┊38┊    private removeAllMessagesGQL: RemoveAllMessagesGQL,
-┊29┊  ┊  ) {}
-┊30┊  ┊
-┊31┊  ┊  getChats() {
-┊32┊  ┊    const query = this.getChatsGQL.watch({
+┊  ┊39┊    private getUsersGQL: GetUsersGQL,
+┊  ┊40┊    private addChatGQL: AddChatGQL,
+┊  ┊41┊    private addGroupGQL: AddGroupGQL
+┊  ┊42┊  ) {
+┊  ┊43┊    this.getChatsWq = this.getChatsGQL.watch({
 ┊33┊44┊      amount: this.messagesAmount,
 ┊34┊45┊    });
-┊35┊  ┊    const chats$ = query.valueChanges.pipe(
+┊  ┊46┊    this.chats$ = this.getChatsWq.valueChanges.pipe(
 ┊36┊47┊      map((result) => result.data.chats)
 ┊37┊48┊    );
+┊  ┊49┊    this.chats$.subscribe(chats => this.chats = chats);
+┊  ┊50┊  }
 ┊38┊51┊
-┊39┊  ┊    return {query, chats$};
+┊  ┊52┊  getChats() {
+┊  ┊53┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊40┊54┊  }
 ┊41┊55┊
 ┊42┊56┊  getChat(chatId: string) {
```
```diff
@@ -209,4 +223,83 @@
 ┊209┊223┊      }, options);
 ┊210┊224┊    }
 ┊211┊225┊  }
+┊   ┊226┊
+┊   ┊227┊  getUsers() {
+┊   ┊228┊    const query = this.getUsersGQL.watch();
+┊   ┊229┊    const users$ = query.valueChanges.pipe(
+┊   ┊230┊      map((result) => result.data.users)
+┊   ┊231┊    );
+┊   ┊232┊
+┊   ┊233┊    return {query, users$};
+┊   ┊234┊  }
+┊   ┊235┊
+┊   ┊236┊  // Checks if the chat is listed for the current user and returns the id
+┊   ┊237┊  getChatId(recipientId: string) {
+┊   ┊238┊    const _chat = this.chats.find(chat => {
+┊   ┊239┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === currentUserId) &&
+┊   ┊240┊        !!chat.allTimeMembers.find(user => user.id === recipientId);
+┊   ┊241┊    });
+┊   ┊242┊    return _chat ? _chat.id : false;
+┊   ┊243┊  }
+┊   ┊244┊
+┊   ┊245┊  addChat(recipientId: string) {
+┊   ┊246┊    return this.addChatGQL.mutate(
+┊   ┊247┊      {
+┊   ┊248┊        recipientId,
+┊   ┊249┊      }, {
+┊   ┊250┊        update: (store, { data: { addChat } }) => {
+┊   ┊251┊          // Read the data from our cache for this query.
+┊   ┊252┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊253┊            query: this.getChatsGQL.document,
+┊   ┊254┊            variables: {
+┊   ┊255┊              amount: this.messagesAmount,
+┊   ┊256┊            },
+┊   ┊257┊          });
+┊   ┊258┊          // Add our comment from the mutation to the end.
+┊   ┊259┊          chats.push(addChat);
+┊   ┊260┊          // Write our data back to the cache.
+┊   ┊261┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊262┊            query: this.getChatsGQL.document,
+┊   ┊263┊            variables: {
+┊   ┊264┊              amount: this.messagesAmount,
+┊   ┊265┊            },
+┊   ┊266┊            data: {
+┊   ┊267┊              chats,
+┊   ┊268┊            },
+┊   ┊269┊          });
+┊   ┊270┊        },
+┊   ┊271┊      }
+┊   ┊272┊    );
+┊   ┊273┊  }
+┊   ┊274┊
+┊   ┊275┊  addGroup(recipientIds: string[], groupName: string) {
+┊   ┊276┊    return this.addGroupGQL.mutate(
+┊   ┊277┊      {
+┊   ┊278┊        recipientIds,
+┊   ┊279┊        groupName,
+┊   ┊280┊      }, {
+┊   ┊281┊        update: (store, { data: { addGroup } }) => {
+┊   ┊282┊          // Read the data from our cache for this query.
+┊   ┊283┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊284┊            query: this.getChatsGQL.document,
+┊   ┊285┊            variables: {
+┊   ┊286┊              amount: this.messagesAmount,
+┊   ┊287┊            },
+┊   ┊288┊          });
+┊   ┊289┊          // Add our comment from the mutation to the end.
+┊   ┊290┊          chats.push(addGroup);
+┊   ┊291┊          // Write our data back to the cache.
+┊   ┊292┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊293┊            query: this.getChatsGQL.document,
+┊   ┊294┊            variables: {
+┊   ┊295┊              amount: this.messagesAmount,
+┊   ┊296┊            },
+┊   ┊297┊            data: {
+┊   ┊298┊              chats,
+┊   ┊299┊            },
+┊   ┊300┊          });
+┊   ┊301┊        },
+┊   ┊302┊      }
+┊   ┊303┊    );
+┊   ┊304┊  }
 ┊212┊305┊}
```

[}]: #

They're both missing few components:

**UsersList**

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/components/users-list/users-list.component.ts, src/app/chats-creation/components/users-list/users-list.component.scss" module="client")

#### Step 8.1: New chats and groups

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.scss
```diff
@@ -0,0 +1,3 @@
+┊ ┊1┊:host {
+┊ ┊2┊  display: block;
+┊ ┊3┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.ts
```diff
@@ -0,0 +1,24 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊import {SelectableListDirective} from 'ngx-selectable-list';
+┊  ┊ 4┊
+┊  ┊ 5┊@Component({
+┊  ┊ 6┊  selector: 'app-users-list',
+┊  ┊ 7┊  template: `
+┊  ┊ 8┊    <mat-list>
+┊  ┊ 9┊      <mat-list-item *ngFor="let user of users">
+┊  ┊10┊        <app-user-item [item]="user"
+┊  ┊11┊                       libSelectableItem></app-user-item>
+┊  ┊12┊      </mat-list-item>
+┊  ┊13┊    </mat-list>
+┊  ┊14┊    <ng-content *ngIf="selectableListDirective.selecting"></ng-content>
+┊  ┊15┊  `,
+┊  ┊16┊  styleUrls: ['users-list.component.scss'],
+┊  ┊17┊})
+┊  ┊18┊export class UsersListComponent {
+┊  ┊19┊  // tslint:disable-next-line:no-input-rename
+┊  ┊20┊  @Input('items')
+┊  ┊21┊  users: GetUsers.Users[];
+┊  ┊22┊
+┊  ┊23┊  constructor(public selectableListDirective: SelectableListDirective) {}
+┊  ┊24┊}
```

[}]: #

**UserItem**

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/components/user-item/user-item.component.ts, src/app/chats-creation/components/user-item/user-item.component.scss" module="client")

#### Step 8.1: New chats and groups

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;user-item&#x2F;user-item.component.scss
```diff
@@ -0,0 +1,28 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  width: 100%;
+┊  ┊ 4┊  height: 100%;
+┊  ┊ 5┊}
+┊  ┊ 6┊
+┊  ┊ 7┊button {
+┊  ┊ 8┊  padding: 0;
+┊  ┊ 9┊  display: flex;
+┊  ┊10┊  align-items: center;
+┊  ┊11┊  height: 100%;
+┊  ┊12┊  width: 100%;
+┊  ┊13┊  border: none;
+┊  ┊14┊
+┊  ┊15┊  div:first-of-type {
+┊  ┊16┊    display: flex;
+┊  ┊17┊    justify-content: center;
+┊  ┊18┊    align-items: center;
+┊  ┊19┊
+┊  ┊20┊    img {
+┊  ┊21┊      max-width: 100%;
+┊  ┊22┊    }
+┊  ┊23┊  }
+┊  ┊24┊
+┊  ┊25┊  div:nth-of-type(2) {
+┊  ┊26┊    padding-left: 16px;
+┊  ┊27┊  }
+┊  ┊28┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;user-item&#x2F;user-item.component.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import {Component, Input} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Component({
+┊  ┊ 5┊  selector: 'app-user-item',
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <button mat-menu-item>
+┊  ┊ 8┊      <div>
+┊  ┊ 9┊        <img [src]="user.picture" *ngIf="user.picture">
+┊  ┊10┊      </div>
+┊  ┊11┊      <div>{{ user.name }}</div>
+┊  ┊12┊    </button>
+┊  ┊13┊  `,
+┊  ┊14┊  styleUrls: ['user-item.component.scss']
+┊  ┊15┊})
+┊  ┊16┊export class UserItemComponent {
+┊  ┊17┊  // tslint:disable-next-line:no-input-rename
+┊  ┊18┊  @Input('item')
+┊  ┊19┊  user: GetUsers.Users;
+┊  ┊20┊}
```

[}]: #

**NewGroupDetails**

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/components/new-group-details/new-group-details.component.ts, src/app/chats-creation/components/new-group-details/new-group-details.component.scss" module="client")

#### Step 8.1: New chats and groups

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;new-group-details&#x2F;new-group-details.component.scss
```diff
@@ -0,0 +1,25 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊}
+┊  ┊ 4┊
+┊  ┊ 5┊div {
+┊  ┊ 6┊  padding: 16px;
+┊  ┊ 7┊  mat-form-field {
+┊  ┊ 8┊    width: 100%;
+┊  ┊ 9┊  }
+┊  ┊10┊}
+┊  ┊11┊
+┊  ┊12┊.new-group {
+┊  ┊13┊  position: absolute;
+┊  ┊14┊  bottom: 5vw;
+┊  ┊15┊  right: 5vw;
+┊  ┊16┊}
+┊  ┊17┊
+┊  ┊18┊.users {
+┊  ┊19┊  display: flex;
+┊  ┊20┊  flex-flow: row wrap;
+┊  ┊21┊  img {
+┊  ┊22┊    flex: 0 1 8vh;
+┊  ┊23┊    height: 8vh;
+┊  ┊24┊  }
+┊  ┊25┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;new-group-details&#x2F;new-group-details.component.ts
```diff
@@ -0,0 +1,34 @@
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Component({
+┊  ┊ 5┊  selector: 'app-new-group-details',
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <div>
+┊  ┊ 8┊      <mat-form-field>
+┊  ┊ 9┊        <input matInput placeholder="Group name" [(ngModel)]="groupName">
+┊  ┊10┊      </mat-form-field>
+┊  ┊11┊    </div>
+┊  ┊12┊    <button [disabled]="!groupName" class="new-group" mat-fab color="primary" (click)="emitGroupDetails()">
+┊  ┊13┊      <mat-icon aria-label="Icon-button with a + icon">arrow_forward</mat-icon>
+┊  ┊14┊    </button>
+┊  ┊15┊    <div>Members</div>
+┊  ┊16┊    <div class="users">
+┊  ┊17┊      <img *ngFor="let user of users;" [src]="user.picture"/>
+┊  ┊18┊    </div>
+┊  ┊19┊  `,
+┊  ┊20┊  styleUrls: ['new-group-details.component.scss'],
+┊  ┊21┊})
+┊  ┊22┊export class NewGroupDetailsComponent {
+┊  ┊23┊  groupName: string;
+┊  ┊24┊  @Input()
+┊  ┊25┊  users: GetUsers.Users[];
+┊  ┊26┊  @Output()
+┊  ┊27┊  groupDetails = new EventEmitter<string>();
+┊  ┊28┊
+┊  ┊29┊  emitGroupDetails() {
+┊  ┊30┊    if (this.groupDetails) {
+┊  ┊31┊      this.groupDetails.emit(this.groupName);
+┊  ┊32┊    }
+┊  ┊33┊  }
+┊  ┊34┊}
```

[}]: #

With all that, we can now link NewChat component with Chat container:

[{]: <helper> (diffStep "8.1" files="src/app/chats-lister/containers/chats/chats.component.ts, src/app/chat-viewer/containers/chat/chat.component.spec.ts" module="client")

#### Step 8.1: New chats and groups

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -143,7 +143,9 @@
 ┊143┊143┊    component = fixture.componentInstance;
 ┊144┊144┊    fixture.detectChanges();
 ┊145┊145┊
-┊146┊   ┊    const req = controller.expectOne('GetChat', 'call to api');
+┊   ┊146┊    controller.expectOne('GetChats', 'call to getChats api');
+┊   ┊147┊
+┊   ┊148┊    const req = controller.expectOne('GetChat', 'call to getChat api');
 ┊147┊149┊
 ┊148┊150┊    req.flush({
 ┊149┊151┊      data: {
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.ts
```diff
@@ -34,7 +34,7 @@
 ┊34┊34┊      <app-confirm-selection #confirmSelection></app-confirm-selection>
 ┊35┊35┊    </app-chats-list>
 ┊36┊36┊
-┊37┊  ┊    <button *ngIf="!isSelecting" class="chat-button" mat-fab color="primary">
+┊  ┊37┊    <button *ngIf="!isSelecting" class="chat-button" mat-fab color="primary" (click)="goToNewChat()">
 ┊38┊38┊      <mat-icon aria-label="Icon-button with a + icon">add</mat-icon>
 ┊39┊39┊    </button>
 ┊40┊40┊  `,
```
```diff
@@ -56,6 +56,10 @@
 ┊56┊56┊    this.router.navigate(['/chat', chatId]);
 ┊57┊57┊  }
 ┊58┊58┊
+┊  ┊59┊  goToNewChat() {
+┊  ┊60┊    this.router.navigate(['/new-chat']);
+┊  ┊61┊  }
+┊  ┊62┊
 ┊59┊63┊  deleteChats(chatIds: string[]) {
 ┊60┊64┊    chatIds.forEach(chatId => {
 ┊61┊65┊      this.chatsService.removeChat(chatId).subscribe();
```

[}]: #

Now let's wrap it all together within the `ChatsCreation` module:

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/chats-creation.module.ts, src/app/app.module.ts" module="client")

#### Step 8.1: New chats and groups

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -7,6 +7,7 @@
 ┊ 7┊ 7┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
 ┊ 8┊ 8┊import {RouterModule, Routes} from '@angular/router';
 ┊ 9┊ 9┊import {ChatViewerModule} from './chat-viewer/chat-viewer.module';
+┊  ┊10┊import {ChatsCreationModule} from './chats-creation/chats-creation.module';
 ┊10┊11┊const routes: Routes = [];
 ┊11┊12┊
 ┊12┊13┊@NgModule({
```
```diff
@@ -22,6 +23,7 @@
 ┊22┊23┊    // Feature modules
 ┊23┊24┊    ChatsListerModule,
 ┊24┊25┊    ChatViewerModule,
+┊  ┊26┊    ChatsCreationModule,
 ┊25┊27┊  ],
 ┊26┊28┊  providers: [],
 ┊27┊29┊  bootstrap: [AppComponent]
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;chats-creation.module.ts
```diff
@@ -0,0 +1,60 @@
+┊  ┊ 1┊import { BrowserModule } from '@angular/platform-browser';
+┊  ┊ 2┊import { NgModule } from '@angular/core';
+┊  ┊ 3┊
+┊  ┊ 4┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 5┊import {
+┊  ┊ 6┊  MatButtonModule, MatFormFieldModule, MatGridListModule, MatIconModule, MatInputModule, MatListModule, MatMenuModule,
+┊  ┊ 7┊  MatToolbarModule
+┊  ┊ 8┊} from '@angular/material';
+┊  ┊ 9┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊10┊import {FormsModule} from '@angular/forms';
+┊  ┊11┊import {ChatsService} from '../services/chats.service';
+┊  ┊12┊import {UserItemComponent} from './components/user-item/user-item.component';
+┊  ┊13┊import {UsersListComponent} from './components/users-list/users-list.component';
+┊  ┊14┊import {NewGroupComponent} from './containers/new-group/new-group.component';
+┊  ┊15┊import {NewChatComponent} from './containers/new-chat/new-chat.component';
+┊  ┊16┊import {NewGroupDetailsComponent} from './components/new-group-details/new-group-details.component';
+┊  ┊17┊import {SharedModule} from '../shared/shared.module';
+┊  ┊18┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊19┊
+┊  ┊20┊const routes: Routes = [
+┊  ┊21┊  {path: 'new-chat', component: NewChatComponent},
+┊  ┊22┊  {path: 'new-group', component: NewGroupComponent},
+┊  ┊23┊];
+┊  ┊24┊
+┊  ┊25┊@NgModule({
+┊  ┊26┊  declarations: [
+┊  ┊27┊    NewChatComponent,
+┊  ┊28┊    UsersListComponent,
+┊  ┊29┊    NewGroupComponent,
+┊  ┊30┊    UserItemComponent,
+┊  ┊31┊    NewGroupDetailsComponent,
+┊  ┊32┊  ],
+┊  ┊33┊  imports: [
+┊  ┊34┊    BrowserModule,
+┊  ┊35┊    // Animations (for Material)
+┊  ┊36┊    BrowserAnimationsModule,
+┊  ┊37┊    // Material
+┊  ┊38┊    MatToolbarModule,
+┊  ┊39┊    MatMenuModule,
+┊  ┊40┊    MatIconModule,
+┊  ┊41┊    MatButtonModule,
+┊  ┊42┊    MatListModule,
+┊  ┊43┊    MatGridListModule,
+┊  ┊44┊    MatInputModule,
+┊  ┊45┊    MatFormFieldModule,
+┊  ┊46┊    MatGridListModule,
+┊  ┊47┊    // Routing
+┊  ┊48┊    RouterModule.forChild(routes),
+┊  ┊49┊    // Forms
+┊  ┊50┊    FormsModule,
+┊  ┊51┊    // Feature modules
+┊  ┊52┊    NgxSelectableListModule,
+┊  ┊53┊    SharedModule,
+┊  ┊54┊  ],
+┊  ┊55┊  providers: [
+┊  ┊56┊    ChatsService,
+┊  ┊57┊  ],
+┊  ┊58┊})
+┊  ┊59┊export class ChatsCreationModule {
+┊  ┊60┊}
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step11.md) | [Next Step >](step13.md) |
|:--------------------------------|--------------------------------:|

[}]: #
