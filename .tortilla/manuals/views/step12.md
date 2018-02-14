# Step 12: Chats creation

[//]: # (head-end)


We still cannot create new chats or groups, so let's implement it.
We're going to create a `ChatsCreation` module, with a `NewChat` and a `NewGroup` containers, along with several presentational components.
We're going to make use of the selectable list directive once again, to ease selecting the users when we're creating a new group.
You should also notice that we are looking for existing chats before creating a new one: if it already exists we're are simply redirecting to that chat instead of creating a new one (the server wouldn't allow that anyway and it will simply
return the chat id).

[{]: <helper> (diffStep "8.1" files="src/graphql" module="client")

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

After creating the mutations we should run the generator to create the corresponding types:

    npm run generator

[{]: <helper> (diffStep "8.1" files="src/app/app.module.ts, src/app/chats-creation, src/app/services" module="client")

#### Step 8.1: New chats and groups

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -9,6 +9,7 @@
 ┊ 9┊ 9┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
 ┊10┊10┊import {RouterModule, Routes} from '@angular/router';
 ┊11┊11┊import {ChatViewerModule} from './chat-viewer/chat-viewer.module';
+┊  ┊12┊import {ChatsCreationModule} from './chats-creation/chats-creation.module';
 ┊12┊13┊const routes: Routes = [];
 ┊13┊14┊
 ┊14┊15┊@NgModule({
```
```diff
@@ -26,6 +27,7 @@
 ┊26┊27┊    // Feature modules
 ┊27┊28┊    ChatsListerModule,
 ┊28┊29┊    ChatViewerModule,
+┊  ┊30┊    ChatsCreationModule,
 ┊29┊31┊  ],
 ┊30┊32┊  providers: [],
 ┊31┊33┊  bootstrap: [AppComponent]
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
+┊  ┊ 2┊import {GetUsers} from '../../../../types';
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
+┊  ┊ 2┊import {GetUsers} from '../../../../types';
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
+┊  ┊ 2┊import {GetUsers} from '../../../../types';
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
+┊  ┊ 4┊import {AddChat, GetUsers} from '../../../../types';
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
+┊  ┊ 4┊import {AddGroup, GetUsers} from '../../../../types';
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
@@ -1,34 +1,44 @@
 ┊ 1┊ 1┊import {ApolloQueryResult, MutationOptions, WatchQueryOptions} from 'apollo-client';
 ┊ 2┊ 2┊import {map} from 'rxjs/operators';
-┊ 3┊  ┊import {Apollo} from 'apollo-angular';
+┊  ┊ 3┊import {Apollo, QueryRef} from 'apollo-angular';
 ┊ 4┊ 4┊import {Injectable} from '@angular/core';
 ┊ 5┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊ 6┊  ┊import {AddMessage, GetChat, GetChats, RemoveAllMessages, RemoveChat, RemoveMessages} from '../../types';
+┊  ┊ 6┊import {AddChat, AddGroup, AddMessage, GetChat, GetChats, GetUsers, RemoveAllMessages, RemoveChat, RemoveMessages} from '../../types';
 ┊ 7┊ 7┊import {getChatQuery} from '../../graphql/getChat.query';
 ┊ 8┊ 8┊import {addMessageMutation} from '../../graphql/addMessage.mutation';
 ┊ 9┊ 9┊import {removeChatMutation} from '../../graphql/removeChat.mutation';
 ┊10┊10┊import {DocumentNode} from 'graphql';
 ┊11┊11┊import {removeAllMessagesMutation} from '../../graphql/removeAllMessages.mutation';
 ┊12┊12┊import {removeMessagesMutation} from '../../graphql/removeMessages.mutation';
+┊  ┊13┊import {getUsersQuery} from '../../graphql/getUsers.query';
+┊  ┊14┊import {Observable} from 'rxjs';
+┊  ┊15┊import {addChatMutation} from '../../graphql/addChat.mutation';
+┊  ┊16┊import {addGroupMutation} from '../../graphql/addGroup.mutation';
+┊  ┊17┊
+┊  ┊18┊const currentUserId = '1';
 ┊13┊19┊
 ┊14┊20┊@Injectable()
 ┊15┊21┊export class ChatsService {
 ┊16┊22┊  messagesAmount = 3;
+┊  ┊23┊  getChatsWq: QueryRef<GetChats.Query>;
+┊  ┊24┊  chats$: Observable<GetChats.Chats[]>;
+┊  ┊25┊  chats: GetChats.Chats[];
 ┊17┊26┊
-┊18┊  ┊  constructor(private apollo: Apollo) {}
-┊19┊  ┊
-┊20┊  ┊  getChats() {
-┊21┊  ┊    const query = this.apollo.watchQuery<GetChats.Query>(<WatchQueryOptions>{
+┊  ┊27┊  constructor(private apollo: Apollo) {
+┊  ┊28┊    this.getChatsWq = this.apollo.watchQuery<GetChats.Query>(<WatchQueryOptions>{
 ┊22┊29┊      query: getChatsQuery,
 ┊23┊30┊      variables: {
 ┊24┊31┊        amount: this.messagesAmount,
 ┊25┊32┊      },
 ┊26┊33┊    });
-┊27┊  ┊    const chats$ = query.valueChanges.pipe(
+┊  ┊34┊    this.chats$ = this.getChatsWq.valueChanges.pipe(
 ┊28┊35┊      map((result: ApolloQueryResult<GetChats.Query>) => result.data.chats)
 ┊29┊36┊    );
+┊  ┊37┊    this.chats$.subscribe(chats => this.chats = chats);
+┊  ┊38┊  }
 ┊30┊39┊
-┊31┊  ┊    return {query, chats$};
+┊  ┊40┊  getChats() {
+┊  ┊41┊    return {query: this.getChatsWq, chats$: this.chats$};
 ┊32┊42┊  }
 ┊33┊43┊
 ┊34┊44┊  getChat(chatId: string) {
```
```diff
@@ -192,4 +202,85 @@
 ┊192┊202┊      },
 ┊193┊203┊    });
 ┊194┊204┊  }
+┊   ┊205┊
+┊   ┊206┊  getUsers() {
+┊   ┊207┊    const query = this.apollo.watchQuery<GetUsers.Query>(<WatchQueryOptions>{
+┊   ┊208┊      query: getUsersQuery,
+┊   ┊209┊    });
+┊   ┊210┊    const users$ = query.valueChanges.pipe(
+┊   ┊211┊      map((result: ApolloQueryResult<GetUsers.Query>) => result.data.users)
+┊   ┊212┊    );
+┊   ┊213┊
+┊   ┊214┊    return {query, users$};
+┊   ┊215┊  }
+┊   ┊216┊
+┊   ┊217┊  // Checks if the chat is listed for the current user and returns the id
+┊   ┊218┊  getChatId(recipientId: string) {
+┊   ┊219┊    const _chat = this.chats.find(chat => {
+┊   ┊220┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === currentUserId) &&
+┊   ┊221┊        !!chat.allTimeMembers.find(user => user.id === recipientId);
+┊   ┊222┊    });
+┊   ┊223┊    return _chat ? _chat.id : false;
+┊   ┊224┊  }
+┊   ┊225┊
+┊   ┊226┊  addChat(recipientId: string) {
+┊   ┊227┊    return this.apollo.mutate({
+┊   ┊228┊      mutation: addChatMutation,
+┊   ┊229┊      variables: <AddChat.Variables>{
+┊   ┊230┊        recipientId,
+┊   ┊231┊      },
+┊   ┊232┊      update: (store, { data: { addChat } }) => {
+┊   ┊233┊        // Read the data from our cache for this query.
+┊   ┊234┊        const {chats}: GetChats.Query = store.readQuery({
+┊   ┊235┊          query: getChatsQuery,
+┊   ┊236┊          variables: <GetChats.Variables>{
+┊   ┊237┊            amount: this.messagesAmount,
+┊   ┊238┊          },
+┊   ┊239┊        });
+┊   ┊240┊        // Add our comment from the mutation to the end.
+┊   ┊241┊        chats.push(addChat);
+┊   ┊242┊        // Write our data back to the cache.
+┊   ┊243┊        store.writeQuery({
+┊   ┊244┊          query: getChatsQuery,
+┊   ┊245┊          variables: <GetChats.Variables>{
+┊   ┊246┊            amount: this.messagesAmount,
+┊   ┊247┊          },
+┊   ┊248┊          data: {
+┊   ┊249┊            chats,
+┊   ┊250┊          },
+┊   ┊251┊        });
+┊   ┊252┊      },
+┊   ┊253┊    });
+┊   ┊254┊  }
+┊   ┊255┊
+┊   ┊256┊  addGroup(recipientIds: string[], groupName: string) {
+┊   ┊257┊    return this.apollo.mutate({
+┊   ┊258┊      mutation: addGroupMutation,
+┊   ┊259┊      variables: <AddGroup.Variables>{
+┊   ┊260┊        recipientIds,
+┊   ┊261┊        groupName,
+┊   ┊262┊      },
+┊   ┊263┊      update: (store, { data: { addGroup } }) => {
+┊   ┊264┊        // Read the data from our cache for this query.
+┊   ┊265┊        const {chats}: GetChats.Query = store.readQuery({
+┊   ┊266┊          query: getChatsQuery,
+┊   ┊267┊          variables: <GetChats.Variables>{
+┊   ┊268┊            amount: this.messagesAmount,
+┊   ┊269┊          },
+┊   ┊270┊        });
+┊   ┊271┊        // Add our comment from the mutation to the end.
+┊   ┊272┊        chats.push(addGroup);
+┊   ┊273┊        // Write our data back to the cache.
+┊   ┊274┊        store.writeQuery({
+┊   ┊275┊          query: getChatsQuery,
+┊   ┊276┊          variables: <GetChats.Variables>{
+┊   ┊277┊            amount: this.messagesAmount,
+┊   ┊278┊          },
+┊   ┊279┊          data: {
+┊   ┊280┊            chats,
+┊   ┊281┊          },
+┊   ┊282┊        });
+┊   ┊283┊      },
+┊   ┊284┊    });
+┊   ┊285┊  }
 ┊195┊286┊}
```

[}]: #

Finally we should update our tests:

[{]: <helper> (diffStep "8.1" files="src/app/chat-viewer/containers/chat/chat.component.spec.ts" module="client")

#### Step 8.1: New chats and groups

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -144,7 +144,8 @@
 ┊144┊144┊    fixture = TestBed.createComponent(ChatComponent);
 ┊145┊145┊    component = fixture.componentInstance;
 ┊146┊146┊    fixture.detectChanges();
-┊147┊   ┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊147┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChats', 'call to getChats api');
+┊   ┊148┊    const req = httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChat', 'call to getChat api');
 ┊148┊149┊    req.flush({
 ┊149┊150┊      data: {
 ┊150┊151┊        chat
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step11.md) | [Next Step >](step13.md) |
|:--------------------------------|--------------------------------:|

[}]: #
