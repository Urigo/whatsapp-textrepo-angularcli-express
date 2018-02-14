# Step 7: Testing

[//]: # (head-end)


## Client

Testing is a very important part of each application and for the sake of showing different testing techniques we are going to show how to test a presentational component, a container component and a service.

First let's start by importing `hammerjs` for the Material Gestures inside the test script.

[{]: <helper> (diffStep "3.1" files="src/test.ts" module="client")

#### Step 3.1: Testing

##### Changed src&#x2F;test.ts
```diff
@@ -7,6 +7,9 @@
 ┊ 7┊ 7┊  platformBrowserDynamicTesting
 ┊ 8┊ 8┊} from '@angular/platform-browser-dynamic/testing';
 ┊ 9┊ 9┊
+┊  ┊10┊// Material gestures
+┊  ┊11┊import 'hammerjs';
+┊  ┊12┊
 ┊10┊13┊declare const require: any;
 ┊11┊14┊
 ┊12┊15┊// First, initialize the Angular testing environment.
```

[}]: #

### Testing a Presentional Component

Let's start with the simplest one: the presentational component.
We are not going to inject any service and we don't need to access our backend, so things are quite simple: we just need to pass our Chat object as an Input, detect the changes and use the query selector to match the UI content to the one we passed as input:

[{]: <helper> (diffStep "3.1" files="src/app/chats-lister/components/chat-item/chat-item.component.spec.ts" module="client")

#### Step 3.1: Testing

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.spec.ts
```diff
@@ -0,0 +1,107 @@
+┊   ┊  1┊import { async, ComponentFixture, TestBed } from '@angular/core/testing';
+┊   ┊  2┊
+┊   ┊  3┊import { ChatItemComponent } from './chat-item.component';
+┊   ┊  4┊import {DebugElement} from '@angular/core';
+┊   ┊  5┊import {By} from '@angular/platform-browser';
+┊   ┊  6┊import {TruncateModule} from 'ng2-truncate';
+┊   ┊  7┊
+┊   ┊  8┊describe('ChatItemComponent', () => {
+┊   ┊  9┊  let component: ChatItemComponent;
+┊   ┊ 10┊  let fixture: ComponentFixture<ChatItemComponent>;
+┊   ┊ 11┊  let el: DebugElement;
+┊   ┊ 12┊
+┊   ┊ 13┊  const chat: any = {
+┊   ┊ 14┊    id: '1',
+┊   ┊ 15┊    __typename: 'Chat',
+┊   ┊ 16┊    name: 'Niccolo\' Belli',
+┊   ┊ 17┊    picture: null,
+┊   ┊ 18┊    allTimeMembers: [
+┊   ┊ 19┊      {
+┊   ┊ 20┊        id: '1',
+┊   ┊ 21┊        __typename: 'User',
+┊   ┊ 22┊      },
+┊   ┊ 23┊      {
+┊   ┊ 24┊        id: '2',
+┊   ┊ 25┊        __typename: 'User',
+┊   ┊ 26┊      }
+┊   ┊ 27┊    ],
+┊   ┊ 28┊    unreadMessages: 0,
+┊   ┊ 29┊    isGroup: false,
+┊   ┊ 30┊    messages: [
+┊   ┊ 31┊      {
+┊   ┊ 32┊        id: '1',
+┊   ┊ 33┊        chat: {
+┊   ┊ 34┊          id: '1',
+┊   ┊ 35┊          __typename: 'Chat',
+┊   ┊ 36┊        },
+┊   ┊ 37┊        __typename: 'Message',
+┊   ┊ 38┊        sender: {
+┊   ┊ 39┊          id: '1',
+┊   ┊ 40┊          __typename: 'User',
+┊   ┊ 41┊          name: 'Niccolo\' Belli',
+┊   ┊ 42┊        },
+┊   ┊ 43┊        content: 'Hello! How are you? A lot happened since last time',
+┊   ┊ 44┊        createdAt: '1513435525',
+┊   ┊ 45┊        type: 1,
+┊   ┊ 46┊        recipients: [
+┊   ┊ 47┊          {
+┊   ┊ 48┊            user: {
+┊   ┊ 49┊              id: '2',
+┊   ┊ 50┊              __typename: 'User',
+┊   ┊ 51┊            },
+┊   ┊ 52┊            message: {
+┊   ┊ 53┊              id: '1',
+┊   ┊ 54┊              __typename: 'Message',
+┊   ┊ 55┊              chat: {
+┊   ┊ 56┊                id: '1',
+┊   ┊ 57┊                __typename: 'Chat',
+┊   ┊ 58┊              },
+┊   ┊ 59┊            },
+┊   ┊ 60┊            __typename: 'Recipient',
+┊   ┊ 61┊            chat: {
+┊   ┊ 62┊              id: '1',
+┊   ┊ 63┊              __typename: 'Chat',
+┊   ┊ 64┊            },
+┊   ┊ 65┊            receivedAt: null,
+┊   ┊ 66┊            readAt: null,
+┊   ┊ 67┊          }
+┊   ┊ 68┊        ],
+┊   ┊ 69┊        ownership: true,
+┊   ┊ 70┊      }
+┊   ┊ 71┊    ],
+┊   ┊ 72┊  };
+┊   ┊ 73┊
+┊   ┊ 74┊  beforeEach(async(() => {
+┊   ┊ 75┊    TestBed.configureTestingModule({
+┊   ┊ 76┊      declarations: [ ChatItemComponent ],
+┊   ┊ 77┊      imports: [TruncateModule]
+┊   ┊ 78┊    })
+┊   ┊ 79┊    .compileComponents();
+┊   ┊ 80┊  }));
+┊   ┊ 81┊
+┊   ┊ 82┊  beforeEach(() => {
+┊   ┊ 83┊    fixture = TestBed.createComponent(ChatItemComponent);
+┊   ┊ 84┊    component = fixture.componentInstance;
+┊   ┊ 85┊    component.chat = chat;
+┊   ┊ 86┊    fixture.detectChanges();
+┊   ┊ 87┊    el = fixture.debugElement;
+┊   ┊ 88┊  });
+┊   ┊ 89┊
+┊   ┊ 90┊  it('should create', () => {
+┊   ┊ 91┊    expect(component).toBeTruthy();
+┊   ┊ 92┊  });
+┊   ┊ 93┊
+┊   ┊ 94┊  it('should contain the chat name', () => {
+┊   ┊ 95┊    expect(el.query(By.css('.chat-recipient > div:first-child')).nativeElement.textContent).toContain(chat.name);
+┊   ┊ 96┊  });
+┊   ┊ 97┊
+┊   ┊ 98┊  it('should contain the first couple of characters of the message content', () => {
+┊   ┊ 99┊    expect(el.query(By.css('.chat-content')).nativeElement.textContent)
+┊   ┊100┊      .toContain(chat.messages[chat.messages.length - 1].content.slice(0, 20));
+┊   ┊101┊  });
+┊   ┊102┊
+┊   ┊103┊  it('should not contain the latest characters of the message content', () => {
+┊   ┊104┊    expect(el.query(By.css('.chat-content')).nativeElement.textContent)
+┊   ┊105┊      .not.toContain(chat.messages[chat.messages.length - 1].content.slice(20));
+┊   ┊106┊  });
+┊   ┊107┊});
```

[}]: #

### Testing a Service

Testing a service is a bit more complicated because we will need to mock our backend in order to get fake results instead of having to fire up the backend each time.

We could simply simply mock the HTTP calls, which is a well known practice in the REST API world. Since we are using HTTP to retrieve the data, it would work as well with our Apollo Client setup.

#### Testing with Apollo Angular

Because Apollo Angular provides its own testing utilities, let's bring them in!

Although `apollo-angular` has a lot going on under the hood, the library provides multiple tools for testing that simplify those abstractions, and allows complete focus on the component logic.

The `apollo-angular/testing` module exports a `ApolloTestingModule` module and `ApolloTestingController` service which simplifies the testing of Angular components by mocking calls to the GraphQL endpoint. This allows the tests to be run in isolation and provides consistent results on every run by removing the dependence on remote data.

By using this `ApolloTestingController` service, it’s possible to specify the exact results that should be returned for a certain query.

Worth mentioning, `ApolloTestingModule` comes with a default cache but you can use your own with `APOLLO_TESTING_CACHE` token.

Let's show everything on an example:

[{]: <helper> (diffStep "3.1" files="src/app/services/chats.service.spec.ts" module="client")

#### Step 3.1: Testing

##### Added src&#x2F;app&#x2F;services&#x2F;chats.service.spec.ts
```diff
@@ -0,0 +1,352 @@
+┊   ┊  1┊import { TestBed, inject } from '@angular/core/testing';
+┊   ┊  2┊
+┊   ┊  3┊import { Apollo } from 'apollo-angular';
+┊   ┊  4┊import {
+┊   ┊  5┊  ApolloTestingModule,
+┊   ┊  6┊  ApolloTestingController,
+┊   ┊  7┊  APOLLO_TESTING_CACHE,
+┊   ┊  8┊} from 'apollo-angular/testing';
+┊   ┊  9┊import { InMemoryCache } from 'apollo-cache-inmemory';
+┊   ┊ 10┊
+┊   ┊ 11┊import { GetChats } from '../../graphql';
+┊   ┊ 12┊import { dataIdFromObject } from '../graphql.module';
+┊   ┊ 13┊import { ChatsService } from './chats.service';
+┊   ┊ 14┊
+┊   ┊ 15┊describe('ChatsService', () => {
+┊   ┊ 16┊  let controller: ApolloTestingController;
+┊   ┊ 17┊  let apollo: Apollo;
+┊   ┊ 18┊
+┊   ┊ 19┊  const chats: GetChats.Chats[] = [
+┊   ┊ 20┊    {
+┊   ┊ 21┊      id: '1',
+┊   ┊ 22┊      __typename: 'Chat',
+┊   ┊ 23┊      name: 'Avery Stewart',
+┊   ┊ 24┊      picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 25┊      allTimeMembers: [
+┊   ┊ 26┊        {
+┊   ┊ 27┊          id: '1',
+┊   ┊ 28┊          __typename: 'User',
+┊   ┊ 29┊        },
+┊   ┊ 30┊        {
+┊   ┊ 31┊          id: '3',
+┊   ┊ 32┊          __typename: 'User',
+┊   ┊ 33┊        },
+┊   ┊ 34┊      ],
+┊   ┊ 35┊      unreadMessages: 1,
+┊   ┊ 36┊      isGroup: false,
+┊   ┊ 37┊      messages: [
+┊   ┊ 38┊        {
+┊   ┊ 39┊          id: '1',
+┊   ┊ 40┊          chat: {
+┊   ┊ 41┊            id: '1',
+┊   ┊ 42┊            __typename: 'Chat',
+┊   ┊ 43┊          },
+┊   ┊ 44┊          __typename: 'Message',
+┊   ┊ 45┊          sender: {
+┊   ┊ 46┊            id: '3',
+┊   ┊ 47┊            __typename: 'User',
+┊   ┊ 48┊            name: 'Avery Stewart',
+┊   ┊ 49┊          },
+┊   ┊ 50┊          content: 'Yep!',
+┊   ┊ 51┊          createdAt: '1514035700',
+┊   ┊ 52┊          type: 0,
+┊   ┊ 53┊          recipients: [
+┊   ┊ 54┊            {
+┊   ┊ 55┊              user: {
+┊   ┊ 56┊                id: '1',
+┊   ┊ 57┊                __typename: 'User',
+┊   ┊ 58┊              },
+┊   ┊ 59┊              message: {
+┊   ┊ 60┊                id: '1',
+┊   ┊ 61┊                __typename: 'Message',
+┊   ┊ 62┊                chat: {
+┊   ┊ 63┊                  id: '1',
+┊   ┊ 64┊                  __typename: 'Chat',
+┊   ┊ 65┊                },
+┊   ┊ 66┊              },
+┊   ┊ 67┊              __typename: 'Recipient',
+┊   ┊ 68┊              chat: {
+┊   ┊ 69┊                id: '1',
+┊   ┊ 70┊                __typename: 'Chat',
+┊   ┊ 71┊              },
+┊   ┊ 72┊              receivedAt: null,
+┊   ┊ 73┊              readAt: null,
+┊   ┊ 74┊            },
+┊   ┊ 75┊          ],
+┊   ┊ 76┊          ownership: false,
+┊   ┊ 77┊        },
+┊   ┊ 78┊      ],
+┊   ┊ 79┊    },
+┊   ┊ 80┊    {
+┊   ┊ 81┊      id: '2',
+┊   ┊ 82┊      __typename: 'Chat',
+┊   ┊ 83┊      name: 'Katie Peterson',
+┊   ┊ 84┊      picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
+┊   ┊ 85┊      allTimeMembers: [
+┊   ┊ 86┊        {
+┊   ┊ 87┊          id: '1',
+┊   ┊ 88┊          __typename: 'User',
+┊   ┊ 89┊        },
+┊   ┊ 90┊        {
+┊   ┊ 91┊          id: '4',
+┊   ┊ 92┊          __typename: 'User',
+┊   ┊ 93┊        },
+┊   ┊ 94┊      ],
+┊   ┊ 95┊      unreadMessages: 0,
+┊   ┊ 96┊      isGroup: false,
+┊   ┊ 97┊      messages: [
+┊   ┊ 98┊        {
+┊   ┊ 99┊          id: '1',
+┊   ┊100┊          chat: {
+┊   ┊101┊            id: '2',
+┊   ┊102┊            __typename: 'Chat',
+┊   ┊103┊          },
+┊   ┊104┊          __typename: 'Message',
+┊   ┊105┊          sender: {
+┊   ┊106┊            id: '1',
+┊   ┊107┊            __typename: 'User',
+┊   ┊108┊            name: 'Ethan Gonzalez',
+┊   ┊109┊          },
+┊   ┊110┊          content: `Hey, it's me`,
+┊   ┊111┊          createdAt: '1514031800',
+┊   ┊112┊          type: 0,
+┊   ┊113┊          recipients: [
+┊   ┊114┊            {
+┊   ┊115┊              user: {
+┊   ┊116┊                id: '4',
+┊   ┊117┊                __typename: 'User',
+┊   ┊118┊              },
+┊   ┊119┊              message: {
+┊   ┊120┊                id: '1',
+┊   ┊121┊                __typename: 'Message',
+┊   ┊122┊                chat: {
+┊   ┊123┊                  id: '2',
+┊   ┊124┊                  __typename: 'Chat',
+┊   ┊125┊                },
+┊   ┊126┊              },
+┊   ┊127┊              __typename: 'Recipient',
+┊   ┊128┊              chat: {
+┊   ┊129┊                id: '2',
+┊   ┊130┊                __typename: 'Chat',
+┊   ┊131┊              },
+┊   ┊132┊              receivedAt: null,
+┊   ┊133┊              readAt: null,
+┊   ┊134┊            },
+┊   ┊135┊          ],
+┊   ┊136┊          ownership: true,
+┊   ┊137┊        },
+┊   ┊138┊      ],
+┊   ┊139┊    },
+┊   ┊140┊    {
+┊   ┊141┊      id: '3',
+┊   ┊142┊      __typename: 'Chat',
+┊   ┊143┊      name: 'Ray Edwards',
+┊   ┊144┊      picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
+┊   ┊145┊      allTimeMembers: [
+┊   ┊146┊        {
+┊   ┊147┊          id: '1',
+┊   ┊148┊          __typename: 'User',
+┊   ┊149┊        },
+┊   ┊150┊        {
+┊   ┊151┊          id: '5',
+┊   ┊152┊          __typename: 'User',
+┊   ┊153┊        },
+┊   ┊154┊      ],
+┊   ┊155┊      unreadMessages: 0,
+┊   ┊156┊      isGroup: false,
+┊   ┊157┊      messages: [
+┊   ┊158┊        {
+┊   ┊159┊          id: '1',
+┊   ┊160┊          __typename: 'Message',
+┊   ┊161┊          chat: {
+┊   ┊162┊            id: '3',
+┊   ┊163┊            __typename: 'Chat',
+┊   ┊164┊          },
+┊   ┊165┊          sender: {
+┊   ┊166┊            id: '1',
+┊   ┊167┊            __typename: 'User',
+┊   ┊168┊            name: 'Ethan Gonzalez',
+┊   ┊169┊          },
+┊   ┊170┊          content: 'You still there?',
+┊   ┊171┊          createdAt: '1514010200',
+┊   ┊172┊          type: 0,
+┊   ┊173┊          recipients: [
+┊   ┊174┊            {
+┊   ┊175┊              user: {
+┊   ┊176┊                id: '5',
+┊   ┊177┊                __typename: 'User',
+┊   ┊178┊              },
+┊   ┊179┊              message: {
+┊   ┊180┊                id: '1',
+┊   ┊181┊                __typename: 'Message',
+┊   ┊182┊                chat: {
+┊   ┊183┊                  id: '3',
+┊   ┊184┊                  __typename: 'Chat',
+┊   ┊185┊                },
+┊   ┊186┊              },
+┊   ┊187┊              __typename: 'Recipient',
+┊   ┊188┊              chat: {
+┊   ┊189┊                id: '3',
+┊   ┊190┊                __typename: 'Chat',
+┊   ┊191┊              },
+┊   ┊192┊              receivedAt: null,
+┊   ┊193┊              readAt: null,
+┊   ┊194┊            },
+┊   ┊195┊          ],
+┊   ┊196┊          ownership: true,
+┊   ┊197┊        },
+┊   ┊198┊      ],
+┊   ┊199┊    },
+┊   ┊200┊    {
+┊   ┊201┊      id: '6',
+┊   ┊202┊      __typename: 'Chat',
+┊   ┊203┊      name: 'Niccolò Belli',
+┊   ┊204┊      picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
+┊   ┊205┊      allTimeMembers: [
+┊   ┊206┊        {
+┊   ┊207┊          id: '1',
+┊   ┊208┊          __typename: 'User',
+┊   ┊209┊        },
+┊   ┊210┊        {
+┊   ┊211┊          id: '6',
+┊   ┊212┊          __typename: 'User',
+┊   ┊213┊        },
+┊   ┊214┊      ],
+┊   ┊215┊      unreadMessages: 0,
+┊   ┊216┊      messages: [],
+┊   ┊217┊      isGroup: false,
+┊   ┊218┊    },
+┊   ┊219┊    {
+┊   ┊220┊      id: '8',
+┊   ┊221┊      __typename: 'Chat',
+┊   ┊222┊      name: 'A user 0 group',
+┊   ┊223┊      picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊224┊      allTimeMembers: [
+┊   ┊225┊        {
+┊   ┊226┊          id: '1',
+┊   ┊227┊          __typename: 'User',
+┊   ┊228┊        },
+┊   ┊229┊        {
+┊   ┊230┊          id: '3',
+┊   ┊231┊          __typename: 'User',
+┊   ┊232┊        },
+┊   ┊233┊        {
+┊   ┊234┊          id: '4',
+┊   ┊235┊          __typename: 'User',
+┊   ┊236┊        },
+┊   ┊237┊        {
+┊   ┊238┊          id: '6',
+┊   ┊239┊          __typename: 'User',
+┊   ┊240┊        },
+┊   ┊241┊      ],
+┊   ┊242┊      unreadMessages: 1,
+┊   ┊243┊      isGroup: true,
+┊   ┊244┊      messages: [
+┊   ┊245┊        {
+┊   ┊246┊          id: '1',
+┊   ┊247┊          __typename: 'Message',
+┊   ┊248┊          chat: {
+┊   ┊249┊            id: '8',
+┊   ┊250┊            __typename: 'Chat',
+┊   ┊251┊          },
+┊   ┊252┊          sender: {
+┊   ┊253┊            id: '4',
+┊   ┊254┊            __typename: 'User',
+┊   ┊255┊            name: 'Katie Peterson',
+┊   ┊256┊          },
+┊   ┊257┊          content: 'Awesome!',
+┊   ┊258┊          createdAt: '1512830000',
+┊   ┊259┊          type: 0,
+┊   ┊260┊          recipients: [
+┊   ┊261┊            {
+┊   ┊262┊              user: {
+┊   ┊263┊                id: '1',
+┊   ┊264┊                __typename: 'User',
+┊   ┊265┊              },
+┊   ┊266┊              message: {
+┊   ┊267┊                id: '1',
+┊   ┊268┊                __typename: 'Message',
+┊   ┊269┊                chat: {
+┊   ┊270┊                  id: '8',
+┊   ┊271┊                  __typename: 'Chat',
+┊   ┊272┊                },
+┊   ┊273┊              },
+┊   ┊274┊              __typename: 'Recipient',
+┊   ┊275┊              chat: {
+┊   ┊276┊                id: '8',
+┊   ┊277┊                __typename: 'Chat',
+┊   ┊278┊              },
+┊   ┊279┊              receivedAt: null,
+┊   ┊280┊              readAt: null,
+┊   ┊281┊            },
+┊   ┊282┊            {
+┊   ┊283┊              user: {
+┊   ┊284┊                id: '6',
+┊   ┊285┊                __typename: 'User',
+┊   ┊286┊              },
+┊   ┊287┊              message: {
+┊   ┊288┊                id: '1',
+┊   ┊289┊                __typename: 'Message',
+┊   ┊290┊                chat: {
+┊   ┊291┊                  id: '8',
+┊   ┊292┊                  __typename: 'Chat',
+┊   ┊293┊                },
+┊   ┊294┊              },
+┊   ┊295┊              __typename: 'Recipient',
+┊   ┊296┊              chat: {
+┊   ┊297┊                id: '8',
+┊   ┊298┊                __typename: 'Chat',
+┊   ┊299┊              },
+┊   ┊300┊              receivedAt: null,
+┊   ┊301┊              readAt: null,
+┊   ┊302┊            },
+┊   ┊303┊          ],
+┊   ┊304┊          ownership: false,
+┊   ┊305┊        },
+┊   ┊306┊      ],
+┊   ┊307┊    },
+┊   ┊308┊  ];
+┊   ┊309┊
+┊   ┊310┊  beforeEach(() => {
+┊   ┊311┊    TestBed.configureTestingModule({
+┊   ┊312┊      imports: [ApolloTestingModule],
+┊   ┊313┊      providers: [
+┊   ┊314┊        ChatsService,
+┊   ┊315┊        {
+┊   ┊316┊          provide: APOLLO_TESTING_CACHE,
+┊   ┊317┊          useFactory() {
+┊   ┊318┊            return new InMemoryCache({
+┊   ┊319┊              dataIdFromObject,
+┊   ┊320┊            });
+┊   ┊321┊          },
+┊   ┊322┊        },
+┊   ┊323┊      ],
+┊   ┊324┊    });
+┊   ┊325┊
+┊   ┊326┊    controller = TestBed.get(ApolloTestingController);
+┊   ┊327┊    apollo = TestBed.get(Apollo);
+┊   ┊328┊  });
+┊   ┊329┊
+┊   ┊330┊  it('should be created', inject([ChatsService], (service: ChatsService) => {
+┊   ┊331┊    expect(service).toBeTruthy();
+┊   ┊332┊  }));
+┊   ┊333┊
+┊   ┊334┊  it('should get chats', inject([ChatsService], (service: ChatsService) => {
+┊   ┊335┊    service.getChats().chats$.subscribe(_chats => {
+┊   ┊336┊      expect(_chats.length).toEqual(chats.length);
+┊   ┊337┊      for (let i = 0; i < _chats.length; i++) {
+┊   ┊338┊        expect(_chats[i]).toEqual(chats[i]);
+┊   ┊339┊      }
+┊   ┊340┊    });
+┊   ┊341┊
+┊   ┊342┊    const req = controller.expectOne('GetChats', 'GetChats operation');
+┊   ┊343┊
+┊   ┊344┊    req.flush({
+┊   ┊345┊      data: {
+┊   ┊346┊        chats,
+┊   ┊347┊      },
+┊   ┊348┊    });
+┊   ┊349┊
+┊   ┊350┊    controller.verify();
+┊   ┊351┊  }));
+┊   ┊352┊});
```

[}]: #

As you can see, the API of `ApolloTestingController` is pretty similar to the `HttpTestingController`.

There are four methods you should know.

**`expectOne()`**
Important thing, it accepts two arguments. First is different for different use cases, the second one stays always the same, it’s a string with a description of your assertion. In case of failing assertion, the error is thrown with an error message including the given description.

Let's explore all those possible cases `expectOne` accepts:

- you can match an operation by its name, simply by passing a string as a first argument.
- by passing the whole Operation object the expectOne method compares: operation’s name, variables, document and extensions.
- the first argument can also be a function that provides an Operation object and expect a boolean in return
- or passing a GraphQL Document

**`exceptNone()`**
It accepts the same arguments as expectOne but it's a negation of it.

**`match()`**
Search for operations that match the given parameters, without any expectations.

**`verify()`**
Verify that no unmatched operations are outstanding. If any operations are outstanding, fail with an error message indicating which operations were not handled.

If you want to learn more about testing, we got you covered, please visit ["Testing Apollo in Angular" chapter](https://www.apollographql.com/docs/angular/guides/testing.html) in Apollo documentation.

### Testing a Container Component

In the last example we are going to test a container component, which makes use of several services and multiple other components:

[{]: <helper> (diffStep "3.1" files="src/app/chats-lister/containers/chats/chats.component.spec.ts" module="client")

#### Step 3.1: Testing

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -0,0 +1,392 @@
+┊   ┊  1┊import { async, ComponentFixture, TestBed } from '@angular/core/testing';
+┊   ┊  2┊import { DebugElement, NO_ERRORS_SCHEMA } from '@angular/core';
+┊   ┊  3┊import { TruncateModule } from 'ng2-truncate';
+┊   ┊  4┊import {
+┊   ┊  5┊  MatButtonModule,
+┊   ┊  6┊  MatIconModule,
+┊   ┊  7┊  MatListModule,
+┊   ┊  8┊  MatMenuModule,
+┊   ┊  9┊} from '@angular/material';
+┊   ┊ 10┊import { Apollo } from 'apollo-angular';
+┊   ┊ 11┊import {
+┊   ┊ 12┊  ApolloTestingModule,
+┊   ┊ 13┊  ApolloTestingController,
+┊   ┊ 14┊  APOLLO_TESTING_CACHE,
+┊   ┊ 15┊} from 'apollo-angular/testing';
+┊   ┊ 16┊import { InMemoryCache } from 'apollo-cache-inmemory';
+┊   ┊ 17┊import { By } from '@angular/platform-browser';
+┊   ┊ 18┊import { RouterTestingModule } from '@angular/router/testing';
+┊   ┊ 19┊
+┊   ┊ 20┊import { GetChats } from '../../../../graphql';
+┊   ┊ 21┊import { dataIdFromObject } from '../../../graphql.module';
+┊   ┊ 22┊import { ChatsComponent } from './chats.component';
+┊   ┊ 23┊import { ChatsListComponent } from '../../components/chats-list/chats-list.component';
+┊   ┊ 24┊import { ChatItemComponent } from '../../components/chat-item/chat-item.component';
+┊   ┊ 25┊import { ChatsService } from '../../../services/chats.service';
+┊   ┊ 26┊
+┊   ┊ 27┊describe('ChatsComponent', () => {
+┊   ┊ 28┊  let component: ChatsComponent;
+┊   ┊ 29┊  let fixture: ComponentFixture<ChatsComponent>;
+┊   ┊ 30┊  let el: DebugElement;
+┊   ┊ 31┊
+┊   ┊ 32┊  let controller: ApolloTestingController;
+┊   ┊ 33┊  let apollo: Apollo;
+┊   ┊ 34┊
+┊   ┊ 35┊  const chats: GetChats.Chats[] = [
+┊   ┊ 36┊    {
+┊   ┊ 37┊      id: '1',
+┊   ┊ 38┊      __typename: 'Chat',
+┊   ┊ 39┊      name: 'Avery Stewart',
+┊   ┊ 40┊      picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 41┊      allTimeMembers: [
+┊   ┊ 42┊        {
+┊   ┊ 43┊          id: '1',
+┊   ┊ 44┊          __typename: 'User',
+┊   ┊ 45┊        },
+┊   ┊ 46┊        {
+┊   ┊ 47┊          id: '3',
+┊   ┊ 48┊          __typename: 'User',
+┊   ┊ 49┊        },
+┊   ┊ 50┊      ],
+┊   ┊ 51┊      unreadMessages: 1,
+┊   ┊ 52┊      isGroup: false,
+┊   ┊ 53┊      messages: [
+┊   ┊ 54┊        {
+┊   ┊ 55┊          id: '1',
+┊   ┊ 56┊          chat: {
+┊   ┊ 57┊            id: '1',
+┊   ┊ 58┊            __typename: 'Chat',
+┊   ┊ 59┊          },
+┊   ┊ 60┊          __typename: 'Message',
+┊   ┊ 61┊          sender: {
+┊   ┊ 62┊            id: '3',
+┊   ┊ 63┊            __typename: 'User',
+┊   ┊ 64┊            name: 'Avery Stewart',
+┊   ┊ 65┊          },
+┊   ┊ 66┊          content: 'Yep!',
+┊   ┊ 67┊          createdAt: '1514035700',
+┊   ┊ 68┊          type: 0,
+┊   ┊ 69┊          recipients: [
+┊   ┊ 70┊            {
+┊   ┊ 71┊              user: {
+┊   ┊ 72┊                id: '1',
+┊   ┊ 73┊                __typename: 'User',
+┊   ┊ 74┊              },
+┊   ┊ 75┊              message: {
+┊   ┊ 76┊                id: '1',
+┊   ┊ 77┊                __typename: 'Message',
+┊   ┊ 78┊                chat: {
+┊   ┊ 79┊                  id: '1',
+┊   ┊ 80┊                  __typename: 'Chat',
+┊   ┊ 81┊                },
+┊   ┊ 82┊              },
+┊   ┊ 83┊              __typename: 'Recipient',
+┊   ┊ 84┊              chat: {
+┊   ┊ 85┊                id: '1',
+┊   ┊ 86┊                __typename: 'Chat',
+┊   ┊ 87┊              },
+┊   ┊ 88┊              receivedAt: null,
+┊   ┊ 89┊              readAt: null,
+┊   ┊ 90┊            },
+┊   ┊ 91┊          ],
+┊   ┊ 92┊          ownership: false,
+┊   ┊ 93┊        },
+┊   ┊ 94┊      ],
+┊   ┊ 95┊    },
+┊   ┊ 96┊    {
+┊   ┊ 97┊      id: '2',
+┊   ┊ 98┊      __typename: 'Chat',
+┊   ┊ 99┊      name: 'Katie Peterson',
+┊   ┊100┊      picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
+┊   ┊101┊      allTimeMembers: [
+┊   ┊102┊        {
+┊   ┊103┊          id: '1',
+┊   ┊104┊          __typename: 'User',
+┊   ┊105┊        },
+┊   ┊106┊        {
+┊   ┊107┊          id: '4',
+┊   ┊108┊          __typename: 'User',
+┊   ┊109┊        },
+┊   ┊110┊      ],
+┊   ┊111┊      unreadMessages: 0,
+┊   ┊112┊      isGroup: false,
+┊   ┊113┊      messages: [
+┊   ┊114┊        {
+┊   ┊115┊          id: '1',
+┊   ┊116┊          chat: {
+┊   ┊117┊            id: '2',
+┊   ┊118┊            __typename: 'Chat',
+┊   ┊119┊          },
+┊   ┊120┊          __typename: 'Message',
+┊   ┊121┊          sender: {
+┊   ┊122┊            id: '1',
+┊   ┊123┊            __typename: 'User',
+┊   ┊124┊            name: 'Ethan Gonzalez',
+┊   ┊125┊          },
+┊   ┊126┊          content: `Hey, it's me`,
+┊   ┊127┊          createdAt: '1514031800',
+┊   ┊128┊          type: 0,
+┊   ┊129┊          recipients: [
+┊   ┊130┊            {
+┊   ┊131┊              user: {
+┊   ┊132┊                id: '4',
+┊   ┊133┊                __typename: 'User',
+┊   ┊134┊              },
+┊   ┊135┊              message: {
+┊   ┊136┊                id: '1',
+┊   ┊137┊                __typename: 'Message',
+┊   ┊138┊                chat: {
+┊   ┊139┊                  id: '2',
+┊   ┊140┊                  __typename: 'Chat',
+┊   ┊141┊                },
+┊   ┊142┊              },
+┊   ┊143┊              __typename: 'Recipient',
+┊   ┊144┊              chat: {
+┊   ┊145┊                id: '2',
+┊   ┊146┊                __typename: 'Chat',
+┊   ┊147┊              },
+┊   ┊148┊              receivedAt: null,
+┊   ┊149┊              readAt: null,
+┊   ┊150┊            },
+┊   ┊151┊          ],
+┊   ┊152┊          ownership: true,
+┊   ┊153┊        },
+┊   ┊154┊      ],
+┊   ┊155┊    },
+┊   ┊156┊    {
+┊   ┊157┊      id: '3',
+┊   ┊158┊      __typename: 'Chat',
+┊   ┊159┊      name: 'Ray Edwards',
+┊   ┊160┊      picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
+┊   ┊161┊      allTimeMembers: [
+┊   ┊162┊        {
+┊   ┊163┊          id: '1',
+┊   ┊164┊          __typename: 'User',
+┊   ┊165┊        },
+┊   ┊166┊        {
+┊   ┊167┊          id: '5',
+┊   ┊168┊          __typename: 'User',
+┊   ┊169┊        },
+┊   ┊170┊      ],
+┊   ┊171┊      unreadMessages: 0,
+┊   ┊172┊      isGroup: false,
+┊   ┊173┊      messages: [
+┊   ┊174┊        {
+┊   ┊175┊          id: '1',
+┊   ┊176┊          __typename: 'Message',
+┊   ┊177┊          chat: {
+┊   ┊178┊            id: '3',
+┊   ┊179┊            __typename: 'Chat',
+┊   ┊180┊          },
+┊   ┊181┊          sender: {
+┊   ┊182┊            id: '1',
+┊   ┊183┊            __typename: 'User',
+┊   ┊184┊            name: 'Ethan Gonzalez',
+┊   ┊185┊          },
+┊   ┊186┊          content: 'You still there?',
+┊   ┊187┊          createdAt: '1514010200',
+┊   ┊188┊          type: 0,
+┊   ┊189┊          recipients: [
+┊   ┊190┊            {
+┊   ┊191┊              user: {
+┊   ┊192┊                id: '5',
+┊   ┊193┊                __typename: 'User',
+┊   ┊194┊              },
+┊   ┊195┊              message: {
+┊   ┊196┊                id: '1',
+┊   ┊197┊                __typename: 'Message',
+┊   ┊198┊                chat: {
+┊   ┊199┊                  id: '3',
+┊   ┊200┊                  __typename: 'Chat',
+┊   ┊201┊                },
+┊   ┊202┊              },
+┊   ┊203┊              __typename: 'Recipient',
+┊   ┊204┊              chat: {
+┊   ┊205┊                id: '3',
+┊   ┊206┊                __typename: 'Chat',
+┊   ┊207┊              },
+┊   ┊208┊              receivedAt: null,
+┊   ┊209┊              readAt: null,
+┊   ┊210┊            },
+┊   ┊211┊          ],
+┊   ┊212┊          ownership: true,
+┊   ┊213┊        },
+┊   ┊214┊      ],
+┊   ┊215┊    },
+┊   ┊216┊    {
+┊   ┊217┊      id: '6',
+┊   ┊218┊      __typename: 'Chat',
+┊   ┊219┊      name: 'Niccolò Belli',
+┊   ┊220┊      picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
+┊   ┊221┊      allTimeMembers: [
+┊   ┊222┊        {
+┊   ┊223┊          id: '1',
+┊   ┊224┊          __typename: 'User',
+┊   ┊225┊        },
+┊   ┊226┊        {
+┊   ┊227┊          id: '6',
+┊   ┊228┊          __typename: 'User',
+┊   ┊229┊        },
+┊   ┊230┊      ],
+┊   ┊231┊      unreadMessages: 0,
+┊   ┊232┊      messages: [],
+┊   ┊233┊      isGroup: false,
+┊   ┊234┊    },
+┊   ┊235┊    {
+┊   ┊236┊      id: '8',
+┊   ┊237┊      __typename: 'Chat',
+┊   ┊238┊      name: 'A user 0 group',
+┊   ┊239┊      picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊240┊      allTimeMembers: [
+┊   ┊241┊        {
+┊   ┊242┊          id: '1',
+┊   ┊243┊          __typename: 'User',
+┊   ┊244┊        },
+┊   ┊245┊        {
+┊   ┊246┊          id: '3',
+┊   ┊247┊          __typename: 'User',
+┊   ┊248┊        },
+┊   ┊249┊        {
+┊   ┊250┊          id: '4',
+┊   ┊251┊          __typename: 'User',
+┊   ┊252┊        },
+┊   ┊253┊        {
+┊   ┊254┊          id: '6',
+┊   ┊255┊          __typename: 'User',
+┊   ┊256┊        },
+┊   ┊257┊      ],
+┊   ┊258┊      unreadMessages: 1,
+┊   ┊259┊      isGroup: true,
+┊   ┊260┊      messages: [
+┊   ┊261┊        {
+┊   ┊262┊          id: '1',
+┊   ┊263┊          __typename: 'Message',
+┊   ┊264┊          chat: {
+┊   ┊265┊            id: '8',
+┊   ┊266┊            __typename: 'Chat',
+┊   ┊267┊          },
+┊   ┊268┊          sender: {
+┊   ┊269┊            id: '4',
+┊   ┊270┊            __typename: 'User',
+┊   ┊271┊            name: 'Katie Peterson',
+┊   ┊272┊          },
+┊   ┊273┊          content: 'Awesome!',
+┊   ┊274┊          createdAt: '1512830000',
+┊   ┊275┊          type: 0,
+┊   ┊276┊          recipients: [
+┊   ┊277┊            {
+┊   ┊278┊              user: {
+┊   ┊279┊                id: '1',
+┊   ┊280┊                __typename: 'User',
+┊   ┊281┊              },
+┊   ┊282┊              message: {
+┊   ┊283┊                id: '1',
+┊   ┊284┊                __typename: 'Message',
+┊   ┊285┊                chat: {
+┊   ┊286┊                  id: '8',
+┊   ┊287┊                  __typename: 'Chat',
+┊   ┊288┊                },
+┊   ┊289┊              },
+┊   ┊290┊              __typename: 'Recipient',
+┊   ┊291┊              chat: {
+┊   ┊292┊                id: '8',
+┊   ┊293┊                __typename: 'Chat',
+┊   ┊294┊              },
+┊   ┊295┊              receivedAt: null,
+┊   ┊296┊              readAt: null,
+┊   ┊297┊            },
+┊   ┊298┊            {
+┊   ┊299┊              user: {
+┊   ┊300┊                id: '6',
+┊   ┊301┊                __typename: 'User',
+┊   ┊302┊              },
+┊   ┊303┊              message: {
+┊   ┊304┊                id: '1',
+┊   ┊305┊                __typename: 'Message',
+┊   ┊306┊                chat: {
+┊   ┊307┊                  id: '8',
+┊   ┊308┊                  __typename: 'Chat',
+┊   ┊309┊                },
+┊   ┊310┊              },
+┊   ┊311┊              __typename: 'Recipient',
+┊   ┊312┊              chat: {
+┊   ┊313┊                id: '8',
+┊   ┊314┊                __typename: 'Chat',
+┊   ┊315┊              },
+┊   ┊316┊              receivedAt: null,
+┊   ┊317┊              readAt: null,
+┊   ┊318┊            },
+┊   ┊319┊          ],
+┊   ┊320┊          ownership: false,
+┊   ┊321┊        },
+┊   ┊322┊      ],
+┊   ┊323┊    },
+┊   ┊324┊  ];
+┊   ┊325┊
+┊   ┊326┊  beforeEach(async(() => {
+┊   ┊327┊    TestBed.configureTestingModule({
+┊   ┊328┊      declarations: [ChatsComponent, ChatsListComponent, ChatItemComponent],
+┊   ┊329┊      imports: [
+┊   ┊330┊        MatMenuModule,
+┊   ┊331┊        MatIconModule,
+┊   ┊332┊        MatButtonModule,
+┊   ┊333┊        MatListModule,
+┊   ┊334┊        TruncateModule,
+┊   ┊335┊        ApolloTestingModule,
+┊   ┊336┊        RouterTestingModule,
+┊   ┊337┊      ],
+┊   ┊338┊      providers: [
+┊   ┊339┊        ChatsService,
+┊   ┊340┊        {
+┊   ┊341┊          provide: APOLLO_TESTING_CACHE,
+┊   ┊342┊          useFactory() {
+┊   ┊343┊            return new InMemoryCache({ dataIdFromObject });
+┊   ┊344┊          },
+┊   ┊345┊        },
+┊   ┊346┊      ],
+┊   ┊347┊      schemas: [NO_ERRORS_SCHEMA],
+┊   ┊348┊    }).compileComponents();
+┊   ┊349┊
+┊   ┊350┊    controller = TestBed.get(ApolloTestingController);
+┊   ┊351┊    apollo = TestBed.get(Apollo);
+┊   ┊352┊  }));
+┊   ┊353┊
+┊   ┊354┊  beforeEach(() => {
+┊   ┊355┊    fixture = TestBed.createComponent(ChatsComponent);
+┊   ┊356┊    component = fixture.componentInstance;
+┊   ┊357┊    fixture.detectChanges();
+┊   ┊358┊
+┊   ┊359┊    const req = controller.expectOne('GetChats', 'GetChats operation');
+┊   ┊360┊
+┊   ┊361┊    req.flush({
+┊   ┊362┊      data: {
+┊   ┊363┊        chats,
+┊   ┊364┊      },
+┊   ┊365┊    });
+┊   ┊366┊  });
+┊   ┊367┊
+┊   ┊368┊  it('should create', () => {
+┊   ┊369┊    expect(component).toBeTruthy();
+┊   ┊370┊  });
+┊   ┊371┊
+┊   ┊372┊  it('should display the chats', () => {
+┊   ┊373┊    fixture.whenStable().then(() => {
+┊   ┊374┊      fixture.detectChanges();
+┊   ┊375┊      el = fixture.debugElement;
+┊   ┊376┊      for (let i = 0; i < chats.length; i++) {
+┊   ┊377┊        expect(
+┊   ┊378┊          el.query(
+┊   ┊379┊            By.css(
+┊   ┊380┊              `app-chats-list > mat-list > mat-list-item:nth-child(${i +
+┊   ┊381┊                1}) > div > app-chat-item > div > div > div`,
+┊   ┊382┊            ),
+┊   ┊383┊          ).nativeElement.textContent,
+┊   ┊384┊        ).toContain(chats[i].name);
+┊   ┊385┊      }
+┊   ┊386┊    });
+┊   ┊387┊  });
+┊   ┊388┊
+┊   ┊389┊  afterAll(() => {
+┊   ┊390┊    controller.verify();
+┊   ┊391┊  });
+┊   ┊392┊});
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step6.md) | [Next Step >](step8.md) |
|:--------------------------------|--------------------------------:|

[}]: #
