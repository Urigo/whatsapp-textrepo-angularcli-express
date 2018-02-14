# Step 8: Chats creation

[//]: # (head-end)


We still cannot create new chats or groups, so let's implement it.

We're going to create a `ChatsCreation` module, with a `NewChat` and a `NewGroup` containers, along with several presentational components.

We're going to make use of the selectable list directive once again, to ease selecting the users when we're creating a new group.
You should also notice that we are looking for existing chats before creating a new one: if it already exists we're are simply redirecting to that chat instead of creating a new one (the server wouldn't allow that anyway and it will simply
return the chat id).

[{]: <helper> (diffStep "8.1" files="src/graphql/" module="client")

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

##### Added src&#x2F;graphql&#x2F;addChat.mutation.ts
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const addChatMutation = gql`
+┊  ┊ 6┊  mutation AddChat($userId: ID!) {
+┊  ┊ 7┊    addChat(userId: $userId) {
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
+┊  ┊ 6┊  mutation AddGroup($userIds: [ID!]!, $groupName: String!) {
+┊  ┊ 7┊    addGroup(userIds: $userIds, groupName: $groupName) {
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

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

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
@@ -208,4 +222,83 @@
 ┊208┊222┊      }, options);
 ┊209┊223┊    }
 ┊210┊224┊  }
+┊   ┊225┊
+┊   ┊226┊  getUsers() {
+┊   ┊227┊    const query = this.getUsersGQL.watch();
+┊   ┊228┊    const users$ = query.valueChanges.pipe(
+┊   ┊229┊      map((result) => result.data.users)
+┊   ┊230┊    );
+┊   ┊231┊
+┊   ┊232┊    return {query, users$};
+┊   ┊233┊  }
+┊   ┊234┊
+┊   ┊235┊  // Checks if the chat is listed for the current user and returns the id
+┊   ┊236┊  getChatId(userId: string) {
+┊   ┊237┊    const _chat = this.chats.find(chat => {
+┊   ┊238┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === currentUserId) &&
+┊   ┊239┊        !!chat.allTimeMembers.find(user => user.id === userId);
+┊   ┊240┊    });
+┊   ┊241┊    return _chat ? _chat.id : false;
+┊   ┊242┊  }
+┊   ┊243┊
+┊   ┊244┊  addChat(userId: string) {
+┊   ┊245┊    return this.addChatGQL.mutate(
+┊   ┊246┊      {
+┊   ┊247┊        userId,
+┊   ┊248┊      }, {
+┊   ┊249┊        update: (store, { data: { addChat } }) => {
+┊   ┊250┊          // Read the data from our cache for this query.
+┊   ┊251┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊252┊            query: this.getChatsGQL.document,
+┊   ┊253┊            variables: {
+┊   ┊254┊              amount: this.messagesAmount,
+┊   ┊255┊            },
+┊   ┊256┊          });
+┊   ┊257┊          // Add our comment from the mutation to the end.
+┊   ┊258┊          chats.push(addChat);
+┊   ┊259┊          // Write our data back to the cache.
+┊   ┊260┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊261┊            query: this.getChatsGQL.document,
+┊   ┊262┊            variables: {
+┊   ┊263┊              amount: this.messagesAmount,
+┊   ┊264┊            },
+┊   ┊265┊            data: {
+┊   ┊266┊              chats,
+┊   ┊267┊            },
+┊   ┊268┊          });
+┊   ┊269┊        },
+┊   ┊270┊      }
+┊   ┊271┊    );
+┊   ┊272┊  }
+┊   ┊273┊
+┊   ┊274┊  addGroup(userIds: string[], groupName: string) {
+┊   ┊275┊    return this.addGroupGQL.mutate(
+┊   ┊276┊      {
+┊   ┊277┊        userIds,
+┊   ┊278┊        groupName,
+┊   ┊279┊      }, {
+┊   ┊280┊        update: (store, { data: { addGroup } }) => {
+┊   ┊281┊          // Read the data from our cache for this query.
+┊   ┊282┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊283┊            query: this.getChatsGQL.document,
+┊   ┊284┊            variables: {
+┊   ┊285┊              amount: this.messagesAmount,
+┊   ┊286┊            },
+┊   ┊287┊          });
+┊   ┊288┊          // Add our comment from the mutation to the end.
+┊   ┊289┊          chats.push(addGroup);
+┊   ┊290┊          // Write our data back to the cache.
+┊   ┊291┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊292┊            query: this.getChatsGQL.document,
+┊   ┊293┊            variables: {
+┊   ┊294┊              amount: this.messagesAmount,
+┊   ┊295┊            },
+┊   ┊296┊            data: {
+┊   ┊297┊              chats,
+┊   ┊298┊            },
+┊   ┊299┊          });
+┊   ┊300┊        },
+┊   ┊301┊      }
+┊   ┊302┊    );
+┊   ┊303┊  }
 ┊211┊304┊}
```

[}]: #

Okay, now since we got the GraphQL part ready, we're going to focus on the component.

First, space to add a new group:

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/containers/new-group/new-group.component.ts, src/app/chats-creation/containers/new-group/new-group.component.scss" module="client")

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

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
+┊  ┊16┊    <app-users-list *ngIf="!userIds.length" [items]="users"
+┊  ┊17┊                    libSelectableList="multiple_tap" (multiple)="selectUsers($event)">
+┊  ┊18┊      <app-confirm-selection #confirmSelection icon="arrow_forward"></app-confirm-selection>
+┊  ┊19┊    </app-users-list>
+┊  ┊20┊    <app-new-group-details *ngIf="userIds.length" [users]="getSelectedUsers()"
+┊  ┊21┊                           (groupDetails)="addGroup($event)"></app-new-group-details>
+┊  ┊22┊  `,
+┊  ┊23┊  styleUrls: ['new-group.component.scss'],
+┊  ┊24┊})
+┊  ┊25┊export class NewGroupComponent implements OnInit {
+┊  ┊26┊  users: GetUsers.Users[];
+┊  ┊27┊  userIds: string[] = [];
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
+┊  ┊38┊    if (this.userIds.length) {
+┊  ┊39┊      this.userIds = [];
+┊  ┊40┊    } else {
+┊  ┊41┊      this.location.back();
+┊  ┊42┊    }
+┊  ┊43┊  }
+┊  ┊44┊
+┊  ┊45┊  selectUsers(userIds: string[]) {
+┊  ┊46┊    this.userIds = userIds;
+┊  ┊47┊  }
+┊  ┊48┊
+┊  ┊49┊  getSelectedUsers() {
+┊  ┊50┊    return this.users.filter(user => this.userIds.includes(user.id));
+┊  ┊51┊  }
+┊  ┊52┊
+┊  ┊53┊  addGroup(groupName: string) {
+┊  ┊54┊    if (groupName && this.userIds.length) {
+┊  ┊55┊      this.chatsService.addGroup(this.userIds, groupName).subscribe(({data: {addGroup: {id}}}: { data: AddGroup.Mutation }) => {
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

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

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
@@ -0,0 +1,42 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  font-family: Roboto, "Helvetica Neue", sans-serif;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊div {
+┊  ┊ 7┊  padding: 16px;
+┊  ┊ 8┊  mat-form-field {
+┊  ┊ 9┊    width: 100%;
+┊  ┊10┊  }
+┊  ┊11┊}
+┊  ┊12┊
+┊  ┊13┊.new-group {
+┊  ┊14┊  position: absolute;
+┊  ┊15┊  bottom: 5vw;
+┊  ┊16┊  right: 5vw;
+┊  ┊17┊}
+┊  ┊18┊
+┊  ┊19┊.users {
+┊  ┊20┊  display: flex;
+┊  ┊21┊  overflow: overlay;
+┊  ┊22┊
+┊  ┊23┊  .user {
+┊  ┊24┊    padding-top: 0;
+┊  ┊25┊    flex-flow: row wrap;
+┊  ┊26┊    text-align: center;
+┊  ┊27┊  }
+┊  ┊28┊
+┊  ┊29┊  .user-profile-pic {
+┊  ┊30┊    flex: 0 1 50px;
+┊  ┊31┊    height: 50px;
+┊  ┊32┊    border-radius: 50%;
+┊  ┊33┊    display: block;
+┊  ┊34┊    margin-left: auto;
+┊  ┊35┊    margin-right: auto;
+┊  ┊36┊  }
+┊  ┊37┊
+┊  ┊38┊  .user-name {
+┊  ┊39┊    line-height: 10px;
+┊  ┊40┊    font-size: 14px;
+┊  ┊41┊  }
+┊  ┊42┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;new-group-details&#x2F;new-group-details.component.ts
```diff
@@ -0,0 +1,37 @@
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Component({
+┊  ┊ 5┊  selector: 'app-new-group-details',
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <div>
+┊  ┊ 8┊      <mat-form-field color="default">
+┊  ┊ 9┊        <input matInput placeholder="Group name" [(ngModel)]="groupName">
+┊  ┊10┊      </mat-form-field>
+┊  ┊11┊    </div>
+┊  ┊12┊    <button *ngIf="groupName" class="new-group" mat-fab color="secondary" (click)="emitGroupDetails()">
+┊  ┊13┊      <mat-icon aria-label="Icon-button with a + icon">arrow_forward</mat-icon>
+┊  ┊14┊    </button>
+┊  ┊15┊    <div>Participants: {{ users.length }}</div>
+┊  ┊16┊    <span class="users">
+┊  ┊17┊      <div class="user" *ngFor="let user of users">
+┊  ┊18┊        <img class="user-profile-pic" [src]="user.picture || 'assets/default-profile-pic.jpg'"/>
+┊  ┊19┊        <span class="user-name">{{ user.name }}</span>
+┊  ┊20┊      </div>
+┊  ┊21┊    </span>
+┊  ┊22┊  `,
+┊  ┊23┊  styleUrls: ['new-group-details.component.scss'],
+┊  ┊24┊})
+┊  ┊25┊export class NewGroupDetailsComponent {
+┊  ┊26┊  groupName: string;
+┊  ┊27┊  @Input()
+┊  ┊28┊  users: GetUsers.Users[];
+┊  ┊29┊  @Output()
+┊  ┊30┊  groupDetails = new EventEmitter<string>();
+┊  ┊31┊
+┊  ┊32┊  emitGroupDetails() {
+┊  ┊33┊    if (this.groupDetails) {
+┊  ┊34┊      this.groupDetails.emit(this.groupName);
+┊  ┊35┊    }
+┊  ┊36┊  }
+┊  ┊37┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;user-item&#x2F;user-item.component.scss
```diff
@@ -0,0 +1,43 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  width: 100%;
+┊  ┊ 4┊  height: 100%;
+┊  ┊ 5┊}
+┊  ┊ 6┊
+┊  ┊ 7┊button {
+┊  ┊ 8┊  background-color: inherit !important;
+┊  ┊ 9┊  outline: none !important;
+┊  ┊10┊  margin: 10px;
+┊  ┊11┊  height: 50px;
+┊  ┊12┊  padding: 1px 6px;
+┊  ┊13┊  line-height: 50px;
+┊  ┊14┊  border: none;
+┊  ┊15┊  margin: 0;
+┊  ┊16┊  position: relative;
+┊  ┊17┊
+┊  ┊18┊  img {
+┊  ┊19┊    float: left;
+┊  ┊20┊    margin-right: 10px;
+┊  ┊21┊    height: 50px;
+┊  ┊22┊    width: 50px;
+┊  ┊23┊    text-align: center;
+┊  ┊24┊    line-height: inherit;
+┊  ┊25┊    border-radius: 50%;
+┊  ┊26┊  }
+┊  ┊27┊
+┊  ┊28┊  div {
+┊  ┊29┊    float: left;
+┊  ┊30┊    line-height: inherit;
+┊  ┊31┊    font-family: Roboto, "Helvetica Neue", sans-serif;
+┊  ┊32┊    font-weight: bold;
+┊  ┊33┊    font-size: 16px;
+┊  ┊34┊  }
+┊  ┊35┊
+┊  ┊36┊  mat-icon {
+┊  ┊37┊    color: var(--primary) !important;
+┊  ┊38┊    opacity: .7;
+┊  ┊39┊    position: absolute;
+┊  ┊40┊    left: 40px;
+┊  ┊41┊    top: 30px;
+┊  ┊42┊  }
+┊  ┊43┊}🚫↵
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
+┊  ┊ 8┊      <img [src]="user.picture || 'assets/default-profile-pic.jpg'">
+┊  ┊ 9┊      <div>{{ user.name }}</div>
+┊  ┊10┊      <mat-icon *ngIf="selected" aria-label="Icon-button with a group add icon">check_circle</mat-icon>
+┊  ┊11┊    </button>
+┊  ┊12┊  `,
+┊  ┊13┊  styleUrls: ['user-item.component.scss']
+┊  ┊14┊})
+┊  ┊15┊export class UserItemComponent {
+┊  ┊16┊  // tslint:disable-next-line:no-input-rename
+┊  ┊17┊  @Input('item') user: GetUsers.Users;
+┊  ┊18┊  // tslint:disable-next-line:no-input-rename
+┊  ┊19┊  @Input() selected: string;
+┊  ┊20┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.scss
```diff
@@ -0,0 +1,13 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊}
+┊  ┊ 4┊
+┊  ┊ 5┊:host ::ng-deep mat-list:first-child > mat-list-item div:first-child {
+┊  ┊ 6┊  padding: 0 !important;
+┊  ┊ 7┊  margin: 5px;
+┊  ┊ 8┊  margin-top: 20px;
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊mat-list {
+┊  ┊12┊  padding: 0;
+┊  ┊13┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.ts
```diff
@@ -0,0 +1,25 @@
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
+┊  ┊11┊                       [selected]="selectableListDirective.selectedItemIds.includes(user.id)"
+┊  ┊12┊                       libSelectableItem></app-user-item>
+┊  ┊13┊      </mat-list-item>
+┊  ┊14┊    </mat-list>
+┊  ┊15┊    <ng-content *ngIf="selectableListDirective.selecting"></ng-content>
+┊  ┊16┊  `,
+┊  ┊17┊  styleUrls: ['users-list.component.scss'],
+┊  ┊18┊})
+┊  ┊19┊export class UsersListComponent {
+┊  ┊20┊  // tslint:disable-next-line:no-input-rename
+┊  ┊21┊  @Input('items')
+┊  ┊22┊  users: GetUsers.Users[];
+┊  ┊23┊
+┊  ┊24┊  constructor(public selectableListDirective: SelectableListDirective) {}
+┊  ┊25┊}
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.scss
```diff
@@ -0,0 +1,23 @@
+┊  ┊ 1┊.new-group {
+┊  ┊ 2┊  margin: 10px;
+┊  ┊ 3┊  height: 50px;
+┊  ┊ 4┊  line-height: 50px;
+┊  ┊ 5┊  margin-top: 20px;
+┊  ┊ 6┊
+┊  ┊ 7┊  mat-icon {
+┊  ┊ 8┊    float: left;
+┊  ┊ 9┊    margin-right: 10px;
+┊  ┊10┊    height: 50px;
+┊  ┊11┊    width: 50px;
+┊  ┊12┊    text-align: center;
+┊  ┊13┊    line-height: inherit;
+┊  ┊14┊    border-radius: 50%;
+┊  ┊15┊  }
+┊  ┊16┊
+┊  ┊17┊  div {
+┊  ┊18┊    float: left;
+┊  ┊19┊    line-height: inherit;
+┊  ┊20┊    font-family: Roboto, "Helvetica Neue", sans-serif;
+┊  ┊21┊    font-weight: bold;
+┊  ┊22┊  }
+┊  ┊23┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;containers&#x2F;new-chat&#x2F;new-chat.component.ts
```diff
@@ -0,0 +1,55 @@
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
+┊  ┊15┊    <div class="new-group" (click)="goToNewGroup()">
+┊  ┊16┊      <mat-icon color="secondary" aria-label="Icon-button with a group add icon">group_add</mat-icon>
+┊  ┊17┊      <div>New group</div>
+┊  ┊18┊    </div>
+┊  ┊19┊    <app-users-list [items]="users"
+┊  ┊20┊                    libSelectableList="single" (single)="addChat($event)">
+┊  ┊21┊    </app-users-list>
+┊  ┊22┊  `,
+┊  ┊23┊  styleUrls: ['new-chat.component.scss'],
+┊  ┊24┊})
+┊  ┊25┊export class NewChatComponent implements OnInit {
+┊  ┊26┊  users: GetUsers.Users[];
+┊  ┊27┊
+┊  ┊28┊  constructor(private router: Router,
+┊  ┊29┊              private location: Location,
+┊  ┊30┊              private chatsService: ChatsService) {}
+┊  ┊31┊
+┊  ┊32┊  ngOnInit () {
+┊  ┊33┊    this.chatsService.getUsers().users$.subscribe(users => this.users = users);
+┊  ┊34┊  }
+┊  ┊35┊
+┊  ┊36┊  goBack() {
+┊  ┊37┊    this.location.back();
+┊  ┊38┊  }
+┊  ┊39┊
+┊  ┊40┊  goToNewGroup() {
+┊  ┊41┊    this.router.navigate(['/new-group']);
+┊  ┊42┊  }
+┊  ┊43┊
+┊  ┊44┊  addChat(userId: string) {
+┊  ┊45┊    const chatId = this.chatsService.getChatId(userId);
+┊  ┊46┊    if (chatId) {
+┊  ┊47┊      // Chat is already listed for the current user
+┊  ┊48┊      this.router.navigate(['/chat', chatId]);
+┊  ┊49┊    } else {
+┊  ┊50┊      this.chatsService.addChat(userId).subscribe(({data: {addChat: {id}}}: { data: AddChat.Mutation }) => {
+┊  ┊51┊        this.router.navigate(['/chat', id]);
+┊  ┊52┊      });
+┊  ┊53┊    }
+┊  ┊54┊  }
+┊  ┊55┊}
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
+┊  ┊16┊    <app-users-list *ngIf="!userIds.length" [items]="users"
+┊  ┊17┊                    libSelectableList="multiple_tap" (multiple)="selectUsers($event)">
+┊  ┊18┊      <app-confirm-selection #confirmSelection icon="arrow_forward"></app-confirm-selection>
+┊  ┊19┊    </app-users-list>
+┊  ┊20┊    <app-new-group-details *ngIf="userIds.length" [users]="getSelectedUsers()"
+┊  ┊21┊                           (groupDetails)="addGroup($event)"></app-new-group-details>
+┊  ┊22┊  `,
+┊  ┊23┊  styleUrls: ['new-group.component.scss'],
+┊  ┊24┊})
+┊  ┊25┊export class NewGroupComponent implements OnInit {
+┊  ┊26┊  users: GetUsers.Users[];
+┊  ┊27┊  userIds: string[] = [];
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
+┊  ┊38┊    if (this.userIds.length) {
+┊  ┊39┊      this.userIds = [];
+┊  ┊40┊    } else {
+┊  ┊41┊      this.location.back();
+┊  ┊42┊    }
+┊  ┊43┊  }
+┊  ┊44┊
+┊  ┊45┊  selectUsers(userIds: string[]) {
+┊  ┊46┊    this.userIds = userIds;
+┊  ┊47┊  }
+┊  ┊48┊
+┊  ┊49┊  getSelectedUsers() {
+┊  ┊50┊    return this.users.filter(user => this.userIds.includes(user.id));
+┊  ┊51┊  }
+┊  ┊52┊
+┊  ┊53┊  addGroup(groupName: string) {
+┊  ┊54┊    if (groupName && this.userIds.length) {
+┊  ┊55┊      this.chatsService.addGroup(this.userIds, groupName).subscribe(({data: {addGroup: {id}}}: { data: AddGroup.Mutation }) => {
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
@@ -208,4 +222,83 @@
 ┊208┊222┊      }, options);
 ┊209┊223┊    }
 ┊210┊224┊  }
+┊   ┊225┊
+┊   ┊226┊  getUsers() {
+┊   ┊227┊    const query = this.getUsersGQL.watch();
+┊   ┊228┊    const users$ = query.valueChanges.pipe(
+┊   ┊229┊      map((result) => result.data.users)
+┊   ┊230┊    );
+┊   ┊231┊
+┊   ┊232┊    return {query, users$};
+┊   ┊233┊  }
+┊   ┊234┊
+┊   ┊235┊  // Checks if the chat is listed for the current user and returns the id
+┊   ┊236┊  getChatId(userId: string) {
+┊   ┊237┊    const _chat = this.chats.find(chat => {
+┊   ┊238┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === currentUserId) &&
+┊   ┊239┊        !!chat.allTimeMembers.find(user => user.id === userId);
+┊   ┊240┊    });
+┊   ┊241┊    return _chat ? _chat.id : false;
+┊   ┊242┊  }
+┊   ┊243┊
+┊   ┊244┊  addChat(userId: string) {
+┊   ┊245┊    return this.addChatGQL.mutate(
+┊   ┊246┊      {
+┊   ┊247┊        userId,
+┊   ┊248┊      }, {
+┊   ┊249┊        update: (store, { data: { addChat } }) => {
+┊   ┊250┊          // Read the data from our cache for this query.
+┊   ┊251┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊252┊            query: this.getChatsGQL.document,
+┊   ┊253┊            variables: {
+┊   ┊254┊              amount: this.messagesAmount,
+┊   ┊255┊            },
+┊   ┊256┊          });
+┊   ┊257┊          // Add our comment from the mutation to the end.
+┊   ┊258┊          chats.push(addChat);
+┊   ┊259┊          // Write our data back to the cache.
+┊   ┊260┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊261┊            query: this.getChatsGQL.document,
+┊   ┊262┊            variables: {
+┊   ┊263┊              amount: this.messagesAmount,
+┊   ┊264┊            },
+┊   ┊265┊            data: {
+┊   ┊266┊              chats,
+┊   ┊267┊            },
+┊   ┊268┊          });
+┊   ┊269┊        },
+┊   ┊270┊      }
+┊   ┊271┊    );
+┊   ┊272┊  }
+┊   ┊273┊
+┊   ┊274┊  addGroup(userIds: string[], groupName: string) {
+┊   ┊275┊    return this.addGroupGQL.mutate(
+┊   ┊276┊      {
+┊   ┊277┊        userIds,
+┊   ┊278┊        groupName,
+┊   ┊279┊      }, {
+┊   ┊280┊        update: (store, { data: { addGroup } }) => {
+┊   ┊281┊          // Read the data from our cache for this query.
+┊   ┊282┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊283┊            query: this.getChatsGQL.document,
+┊   ┊284┊            variables: {
+┊   ┊285┊              amount: this.messagesAmount,
+┊   ┊286┊            },
+┊   ┊287┊          });
+┊   ┊288┊          // Add our comment from the mutation to the end.
+┊   ┊289┊          chats.push(addGroup);
+┊   ┊290┊          // Write our data back to the cache.
+┊   ┊291┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊   ┊292┊            query: this.getChatsGQL.document,
+┊   ┊293┊            variables: {
+┊   ┊294┊              amount: this.messagesAmount,
+┊   ┊295┊            },
+┊   ┊296┊            data: {
+┊   ┊297┊              chats,
+┊   ┊298┊            },
+┊   ┊299┊          });
+┊   ┊300┊        },
+┊   ┊301┊      }
+┊   ┊302┊    );
+┊   ┊303┊  }
 ┊211┊304┊}
```

[}]: #

They're both missing few components:

**UsersList**

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/components/users-list/users-list.component.ts, src/app/chats-creation/components/users-list/users-list.component.scss" module="client")

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.scss
```diff
@@ -0,0 +1,13 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊}
+┊  ┊ 4┊
+┊  ┊ 5┊:host ::ng-deep mat-list:first-child > mat-list-item div:first-child {
+┊  ┊ 6┊  padding: 0 !important;
+┊  ┊ 7┊  margin: 5px;
+┊  ┊ 8┊  margin-top: 20px;
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊mat-list {
+┊  ┊12┊  padding: 0;
+┊  ┊13┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;users-list&#x2F;users-list.component.ts
```diff
@@ -0,0 +1,25 @@
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
+┊  ┊11┊                       [selected]="selectableListDirective.selectedItemIds.includes(user.id)"
+┊  ┊12┊                       libSelectableItem></app-user-item>
+┊  ┊13┊      </mat-list-item>
+┊  ┊14┊    </mat-list>
+┊  ┊15┊    <ng-content *ngIf="selectableListDirective.selecting"></ng-content>
+┊  ┊16┊  `,
+┊  ┊17┊  styleUrls: ['users-list.component.scss'],
+┊  ┊18┊})
+┊  ┊19┊export class UsersListComponent {
+┊  ┊20┊  // tslint:disable-next-line:no-input-rename
+┊  ┊21┊  @Input('items')
+┊  ┊22┊  users: GetUsers.Users[];
+┊  ┊23┊
+┊  ┊24┊  constructor(public selectableListDirective: SelectableListDirective) {}
+┊  ┊25┊}
```

[}]: #

**UserItem**

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/components/user-item/user-item.component.ts, src/app/chats-creation/components/user-item/user-item.component.scss" module="client")

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;user-item&#x2F;user-item.component.scss
```diff
@@ -0,0 +1,43 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  width: 100%;
+┊  ┊ 4┊  height: 100%;
+┊  ┊ 5┊}
+┊  ┊ 6┊
+┊  ┊ 7┊button {
+┊  ┊ 8┊  background-color: inherit !important;
+┊  ┊ 9┊  outline: none !important;
+┊  ┊10┊  margin: 10px;
+┊  ┊11┊  height: 50px;
+┊  ┊12┊  padding: 1px 6px;
+┊  ┊13┊  line-height: 50px;
+┊  ┊14┊  border: none;
+┊  ┊15┊  margin: 0;
+┊  ┊16┊  position: relative;
+┊  ┊17┊
+┊  ┊18┊  img {
+┊  ┊19┊    float: left;
+┊  ┊20┊    margin-right: 10px;
+┊  ┊21┊    height: 50px;
+┊  ┊22┊    width: 50px;
+┊  ┊23┊    text-align: center;
+┊  ┊24┊    line-height: inherit;
+┊  ┊25┊    border-radius: 50%;
+┊  ┊26┊  }
+┊  ┊27┊
+┊  ┊28┊  div {
+┊  ┊29┊    float: left;
+┊  ┊30┊    line-height: inherit;
+┊  ┊31┊    font-family: Roboto, "Helvetica Neue", sans-serif;
+┊  ┊32┊    font-weight: bold;
+┊  ┊33┊    font-size: 16px;
+┊  ┊34┊  }
+┊  ┊35┊
+┊  ┊36┊  mat-icon {
+┊  ┊37┊    color: var(--primary) !important;
+┊  ┊38┊    opacity: .7;
+┊  ┊39┊    position: absolute;
+┊  ┊40┊    left: 40px;
+┊  ┊41┊    top: 30px;
+┊  ┊42┊  }
+┊  ┊43┊}🚫↵
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
+┊  ┊ 8┊      <img [src]="user.picture || 'assets/default-profile-pic.jpg'">
+┊  ┊ 9┊      <div>{{ user.name }}</div>
+┊  ┊10┊      <mat-icon *ngIf="selected" aria-label="Icon-button with a group add icon">check_circle</mat-icon>
+┊  ┊11┊    </button>
+┊  ┊12┊  `,
+┊  ┊13┊  styleUrls: ['user-item.component.scss']
+┊  ┊14┊})
+┊  ┊15┊export class UserItemComponent {
+┊  ┊16┊  // tslint:disable-next-line:no-input-rename
+┊  ┊17┊  @Input('item') user: GetUsers.Users;
+┊  ┊18┊  // tslint:disable-next-line:no-input-rename
+┊  ┊19┊  @Input() selected: string;
+┊  ┊20┊}🚫↵
```

[}]: #

**NewGroupDetails**

[{]: <helper> (diffStep "8.1" files="src/app/chats-creation/components/new-group-details/new-group-details.component.ts, src/app/chats-creation/components/new-group-details/new-group-details.component.scss" module="client")

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;new-group-details&#x2F;new-group-details.component.scss
```diff
@@ -0,0 +1,42 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊  font-family: Roboto, "Helvetica Neue", sans-serif;
+┊  ┊ 4┊}
+┊  ┊ 5┊
+┊  ┊ 6┊div {
+┊  ┊ 7┊  padding: 16px;
+┊  ┊ 8┊  mat-form-field {
+┊  ┊ 9┊    width: 100%;
+┊  ┊10┊  }
+┊  ┊11┊}
+┊  ┊12┊
+┊  ┊13┊.new-group {
+┊  ┊14┊  position: absolute;
+┊  ┊15┊  bottom: 5vw;
+┊  ┊16┊  right: 5vw;
+┊  ┊17┊}
+┊  ┊18┊
+┊  ┊19┊.users {
+┊  ┊20┊  display: flex;
+┊  ┊21┊  overflow: overlay;
+┊  ┊22┊
+┊  ┊23┊  .user {
+┊  ┊24┊    padding-top: 0;
+┊  ┊25┊    flex-flow: row wrap;
+┊  ┊26┊    text-align: center;
+┊  ┊27┊  }
+┊  ┊28┊
+┊  ┊29┊  .user-profile-pic {
+┊  ┊30┊    flex: 0 1 50px;
+┊  ┊31┊    height: 50px;
+┊  ┊32┊    border-radius: 50%;
+┊  ┊33┊    display: block;
+┊  ┊34┊    margin-left: auto;
+┊  ┊35┊    margin-right: auto;
+┊  ┊36┊  }
+┊  ┊37┊
+┊  ┊38┊  .user-name {
+┊  ┊39┊    line-height: 10px;
+┊  ┊40┊    font-size: 14px;
+┊  ┊41┊  }
+┊  ┊42┊}🚫↵
```

##### Added src&#x2F;app&#x2F;chats-creation&#x2F;components&#x2F;new-group-details&#x2F;new-group-details.component.ts
```diff
@@ -0,0 +1,37 @@
+┊  ┊ 1┊import {Component, EventEmitter, Input, Output} from '@angular/core';
+┊  ┊ 2┊import {GetUsers} from '../../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Component({
+┊  ┊ 5┊  selector: 'app-new-group-details',
+┊  ┊ 6┊  template: `
+┊  ┊ 7┊    <div>
+┊  ┊ 8┊      <mat-form-field color="default">
+┊  ┊ 9┊        <input matInput placeholder="Group name" [(ngModel)]="groupName">
+┊  ┊10┊      </mat-form-field>
+┊  ┊11┊    </div>
+┊  ┊12┊    <button *ngIf="groupName" class="new-group" mat-fab color="secondary" (click)="emitGroupDetails()">
+┊  ┊13┊      <mat-icon aria-label="Icon-button with a + icon">arrow_forward</mat-icon>
+┊  ┊14┊    </button>
+┊  ┊15┊    <div>Participants: {{ users.length }}</div>
+┊  ┊16┊    <span class="users">
+┊  ┊17┊      <div class="user" *ngFor="let user of users">
+┊  ┊18┊        <img class="user-profile-pic" [src]="user.picture || 'assets/default-profile-pic.jpg'"/>
+┊  ┊19┊        <span class="user-name">{{ user.name }}</span>
+┊  ┊20┊      </div>
+┊  ┊21┊    </span>
+┊  ┊22┊  `,
+┊  ┊23┊  styleUrls: ['new-group-details.component.scss'],
+┊  ┊24┊})
+┊  ┊25┊export class NewGroupDetailsComponent {
+┊  ┊26┊  groupName: string;
+┊  ┊27┊  @Input()
+┊  ┊28┊  users: GetUsers.Users[];
+┊  ┊29┊  @Output()
+┊  ┊30┊  groupDetails = new EventEmitter<string>();
+┊  ┊31┊
+┊  ┊32┊  emitGroupDetails() {
+┊  ┊33┊    if (this.groupDetails) {
+┊  ┊34┊      this.groupDetails.emit(this.groupName);
+┊  ┊35┊    }
+┊  ┊36┊  }
+┊  ┊37┊}🚫↵
```

[}]: #

With all that, we can now link NewChat component with Chat container:

[{]: <helper> (diffStep "8.1" files="src/app/chats-lister/containers/chats/chats.component.ts, src/app/chat-viewer/containers/chat/chat.component.spec.ts" module="client")

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

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
-┊37┊  ┊    <button *ngIf="!isSelecting" class="chat-button" mat-fab color="secondary">
+┊  ┊37┊    <button *ngIf="!isSelecting" class="chat-button" mat-fab color="secondary" (click)="goToNewChat()">
 ┊38┊38┊      <mat-icon aria-label="Icon-button with a + icon">chat</mat-icon>
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

#### [Step 8.1: New chats and groups](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/fc3a8a3)

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

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step7.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step9.md) |
|:--------------------------------|--------------------------------:|

[}]: #
