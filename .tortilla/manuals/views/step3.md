# Step 3: Testing

[//]: # (head-end)


## Client

Testing is a very important part of each application and for the sake of showing different testing techniques we are going to show how to test a presentational component, a container component and a service.

First let's start by importing `hammerjs` for the Material Gestures inside the test script.

[{]: <helper> (diffStep "3.1" files="src/test.ts" module="client")

#### [Step 3.1: Testing](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/d67e5a8)

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

#### [Step 3.1: Testing](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/d67e5a8)

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;components&#x2F;chat-item&#x2F;chat-item.component.spec.ts
```diff
@@ -0,0 +1,100 @@
+┊   ┊  1┊import { async, ComponentFixture, TestBed } from '@angular/core/testing';
+┊   ┊  2┊
+┊   ┊  3┊import { ChatItemComponent } from './chat-item.component';
+┊   ┊  4┊import {DebugElement} from '@angular/core';
+┊   ┊  5┊import {By} from '@angular/platform-browser';
+┊   ┊  6┊
+┊   ┊  7┊describe('ChatItemComponent', () => {
+┊   ┊  8┊  let component: ChatItemComponent;
+┊   ┊  9┊  let fixture: ComponentFixture<ChatItemComponent>;
+┊   ┊ 10┊  let el: DebugElement;
+┊   ┊ 11┊
+┊   ┊ 12┊  const chat: any = {
+┊   ┊ 13┊    id: '1',
+┊   ┊ 14┊    __typename: 'Chat',
+┊   ┊ 15┊    name: 'Niccolo\' Belli',
+┊   ┊ 16┊    picture: null,
+┊   ┊ 17┊    allTimeMembers: [
+┊   ┊ 18┊      {
+┊   ┊ 19┊        id: '1',
+┊   ┊ 20┊        __typename: 'User',
+┊   ┊ 21┊      },
+┊   ┊ 22┊      {
+┊   ┊ 23┊        id: '2',
+┊   ┊ 24┊        __typename: 'User',
+┊   ┊ 25┊      }
+┊   ┊ 26┊    ],
+┊   ┊ 27┊    unreadMessages: 0,
+┊   ┊ 28┊    isGroup: false,
+┊   ┊ 29┊    messages: [
+┊   ┊ 30┊      {
+┊   ┊ 31┊        id: '1',
+┊   ┊ 32┊        chat: {
+┊   ┊ 33┊          id: '1',
+┊   ┊ 34┊          __typename: 'Chat',
+┊   ┊ 35┊        },
+┊   ┊ 36┊        __typename: 'Message',
+┊   ┊ 37┊        sender: {
+┊   ┊ 38┊          id: '1',
+┊   ┊ 39┊          __typename: 'User',
+┊   ┊ 40┊          name: 'Niccolo\' Belli',
+┊   ┊ 41┊        },
+┊   ┊ 42┊        content: 'Hello! How are you? A lot happened since last time',
+┊   ┊ 43┊        createdAt: '1513435525',
+┊   ┊ 44┊        type: 1,
+┊   ┊ 45┊        recipients: [
+┊   ┊ 46┊          {
+┊   ┊ 47┊            user: {
+┊   ┊ 48┊              id: '2',
+┊   ┊ 49┊              __typename: 'User',
+┊   ┊ 50┊            },
+┊   ┊ 51┊            message: {
+┊   ┊ 52┊              id: '1',
+┊   ┊ 53┊              __typename: 'Message',
+┊   ┊ 54┊              chat: {
+┊   ┊ 55┊                id: '1',
+┊   ┊ 56┊                __typename: 'Chat',
+┊   ┊ 57┊              },
+┊   ┊ 58┊            },
+┊   ┊ 59┊            __typename: 'Recipient',
+┊   ┊ 60┊            chat: {
+┊   ┊ 61┊              id: '1',
+┊   ┊ 62┊              __typename: 'Chat',
+┊   ┊ 63┊            },
+┊   ┊ 64┊            receivedAt: null,
+┊   ┊ 65┊            readAt: null,
+┊   ┊ 66┊          }
+┊   ┊ 67┊        ],
+┊   ┊ 68┊        ownership: true,
+┊   ┊ 69┊      }
+┊   ┊ 70┊    ],
+┊   ┊ 71┊  };
+┊   ┊ 72┊
+┊   ┊ 73┊  beforeEach(async(() => {
+┊   ┊ 74┊    TestBed.configureTestingModule({
+┊   ┊ 75┊      declarations: [ ChatItemComponent ],
+┊   ┊ 76┊    })
+┊   ┊ 77┊    .compileComponents();
+┊   ┊ 78┊  }));
+┊   ┊ 79┊
+┊   ┊ 80┊  beforeEach(() => {
+┊   ┊ 81┊    fixture = TestBed.createComponent(ChatItemComponent);
+┊   ┊ 82┊    component = fixture.componentInstance;
+┊   ┊ 83┊    component.chat = chat;
+┊   ┊ 84┊    fixture.detectChanges();
+┊   ┊ 85┊    el = fixture.debugElement;
+┊   ┊ 86┊  });
+┊   ┊ 87┊
+┊   ┊ 88┊  it('should create', () => {
+┊   ┊ 89┊    expect(component).toBeTruthy();
+┊   ┊ 90┊  });
+┊   ┊ 91┊
+┊   ┊ 92┊  it('should contain the chat name', () => {
+┊   ┊ 93┊    expect(el.query(By.css('div.chat-info > div.chat-name')).nativeElement.textContent).toContain(chat.name);
+┊   ┊ 94┊  });
+┊   ┊ 95┊
+┊   ┊ 96┊  it('should contain the first couple of characters of the message content', () => {
+┊   ┊ 97┊    expect(el.query(By.css('div.chat-info > div.chat-last-message')).nativeElement.textContent)
+┊   ┊ 98┊      .toContain(chat.messages[chat.messages.length - 1].content.slice(0, 20));
+┊   ┊ 99┊  });
+┊   ┊100┊});
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

#### [Step 3.1: Testing](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/d67e5a8)

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

#### [Step 3.1: Testing](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/d67e5a8)

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -0,0 +1,390 @@
+┊   ┊  1┊import { async, ComponentFixture, TestBed } from '@angular/core/testing';
+┊   ┊  2┊import { DebugElement, NO_ERRORS_SCHEMA } from '@angular/core';
+┊   ┊  3┊import {
+┊   ┊  4┊  MatButtonModule,
+┊   ┊  5┊  MatIconModule,
+┊   ┊  6┊  MatListModule,
+┊   ┊  7┊  MatMenuModule,
+┊   ┊  8┊} from '@angular/material';
+┊   ┊  9┊import { Apollo } from 'apollo-angular';
+┊   ┊ 10┊import {
+┊   ┊ 11┊  ApolloTestingModule,
+┊   ┊ 12┊  ApolloTestingController,
+┊   ┊ 13┊  APOLLO_TESTING_CACHE,
+┊   ┊ 14┊} from 'apollo-angular/testing';
+┊   ┊ 15┊import { InMemoryCache } from 'apollo-cache-inmemory';
+┊   ┊ 16┊import { By } from '@angular/platform-browser';
+┊   ┊ 17┊import { RouterTestingModule } from '@angular/router/testing';
+┊   ┊ 18┊
+┊   ┊ 19┊import { GetChats } from '../../../../graphql';
+┊   ┊ 20┊import { dataIdFromObject } from '../../../graphql.module';
+┊   ┊ 21┊import { ChatsComponent } from './chats.component';
+┊   ┊ 22┊import { ChatsListComponent } from '../../components/chats-list/chats-list.component';
+┊   ┊ 23┊import { ChatItemComponent } from '../../components/chat-item/chat-item.component';
+┊   ┊ 24┊import { ChatsService } from '../../../services/chats.service';
+┊   ┊ 25┊
+┊   ┊ 26┊describe('ChatsComponent', () => {
+┊   ┊ 27┊  let component: ChatsComponent;
+┊   ┊ 28┊  let fixture: ComponentFixture<ChatsComponent>;
+┊   ┊ 29┊  let el: DebugElement;
+┊   ┊ 30┊
+┊   ┊ 31┊  let controller: ApolloTestingController;
+┊   ┊ 32┊  let apollo: Apollo;
+┊   ┊ 33┊
+┊   ┊ 34┊  const chats: GetChats.Chats[] = [
+┊   ┊ 35┊    {
+┊   ┊ 36┊      id: '1',
+┊   ┊ 37┊      __typename: 'Chat',
+┊   ┊ 38┊      name: 'Avery Stewart',
+┊   ┊ 39┊      picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 40┊      allTimeMembers: [
+┊   ┊ 41┊        {
+┊   ┊ 42┊          id: '1',
+┊   ┊ 43┊          __typename: 'User',
+┊   ┊ 44┊        },
+┊   ┊ 45┊        {
+┊   ┊ 46┊          id: '3',
+┊   ┊ 47┊          __typename: 'User',
+┊   ┊ 48┊        },
+┊   ┊ 49┊      ],
+┊   ┊ 50┊      unreadMessages: 1,
+┊   ┊ 51┊      isGroup: false,
+┊   ┊ 52┊      messages: [
+┊   ┊ 53┊        {
+┊   ┊ 54┊          id: '1',
+┊   ┊ 55┊          chat: {
+┊   ┊ 56┊            id: '1',
+┊   ┊ 57┊            __typename: 'Chat',
+┊   ┊ 58┊          },
+┊   ┊ 59┊          __typename: 'Message',
+┊   ┊ 60┊          sender: {
+┊   ┊ 61┊            id: '3',
+┊   ┊ 62┊            __typename: 'User',
+┊   ┊ 63┊            name: 'Avery Stewart',
+┊   ┊ 64┊          },
+┊   ┊ 65┊          content: 'Yep!',
+┊   ┊ 66┊          createdAt: '1514035700',
+┊   ┊ 67┊          type: 0,
+┊   ┊ 68┊          recipients: [
+┊   ┊ 69┊            {
+┊   ┊ 70┊              user: {
+┊   ┊ 71┊                id: '1',
+┊   ┊ 72┊                __typename: 'User',
+┊   ┊ 73┊              },
+┊   ┊ 74┊              message: {
+┊   ┊ 75┊                id: '1',
+┊   ┊ 76┊                __typename: 'Message',
+┊   ┊ 77┊                chat: {
+┊   ┊ 78┊                  id: '1',
+┊   ┊ 79┊                  __typename: 'Chat',
+┊   ┊ 80┊                },
+┊   ┊ 81┊              },
+┊   ┊ 82┊              __typename: 'Recipient',
+┊   ┊ 83┊              chat: {
+┊   ┊ 84┊                id: '1',
+┊   ┊ 85┊                __typename: 'Chat',
+┊   ┊ 86┊              },
+┊   ┊ 87┊              receivedAt: null,
+┊   ┊ 88┊              readAt: null,
+┊   ┊ 89┊            },
+┊   ┊ 90┊          ],
+┊   ┊ 91┊          ownership: false,
+┊   ┊ 92┊        },
+┊   ┊ 93┊      ],
+┊   ┊ 94┊    },
+┊   ┊ 95┊    {
+┊   ┊ 96┊      id: '2',
+┊   ┊ 97┊      __typename: 'Chat',
+┊   ┊ 98┊      name: 'Katie Peterson',
+┊   ┊ 99┊      picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
+┊   ┊100┊      allTimeMembers: [
+┊   ┊101┊        {
+┊   ┊102┊          id: '1',
+┊   ┊103┊          __typename: 'User',
+┊   ┊104┊        },
+┊   ┊105┊        {
+┊   ┊106┊          id: '4',
+┊   ┊107┊          __typename: 'User',
+┊   ┊108┊        },
+┊   ┊109┊      ],
+┊   ┊110┊      unreadMessages: 0,
+┊   ┊111┊      isGroup: false,
+┊   ┊112┊      messages: [
+┊   ┊113┊        {
+┊   ┊114┊          id: '1',
+┊   ┊115┊          chat: {
+┊   ┊116┊            id: '2',
+┊   ┊117┊            __typename: 'Chat',
+┊   ┊118┊          },
+┊   ┊119┊          __typename: 'Message',
+┊   ┊120┊          sender: {
+┊   ┊121┊            id: '1',
+┊   ┊122┊            __typename: 'User',
+┊   ┊123┊            name: 'Ethan Gonzalez',
+┊   ┊124┊          },
+┊   ┊125┊          content: `Hey, it's me`,
+┊   ┊126┊          createdAt: '1514031800',
+┊   ┊127┊          type: 0,
+┊   ┊128┊          recipients: [
+┊   ┊129┊            {
+┊   ┊130┊              user: {
+┊   ┊131┊                id: '4',
+┊   ┊132┊                __typename: 'User',
+┊   ┊133┊              },
+┊   ┊134┊              message: {
+┊   ┊135┊                id: '1',
+┊   ┊136┊                __typename: 'Message',
+┊   ┊137┊                chat: {
+┊   ┊138┊                  id: '2',
+┊   ┊139┊                  __typename: 'Chat',
+┊   ┊140┊                },
+┊   ┊141┊              },
+┊   ┊142┊              __typename: 'Recipient',
+┊   ┊143┊              chat: {
+┊   ┊144┊                id: '2',
+┊   ┊145┊                __typename: 'Chat',
+┊   ┊146┊              },
+┊   ┊147┊              receivedAt: null,
+┊   ┊148┊              readAt: null,
+┊   ┊149┊            },
+┊   ┊150┊          ],
+┊   ┊151┊          ownership: true,
+┊   ┊152┊        },
+┊   ┊153┊      ],
+┊   ┊154┊    },
+┊   ┊155┊    {
+┊   ┊156┊      id: '3',
+┊   ┊157┊      __typename: 'Chat',
+┊   ┊158┊      name: 'Ray Edwards',
+┊   ┊159┊      picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
+┊   ┊160┊      allTimeMembers: [
+┊   ┊161┊        {
+┊   ┊162┊          id: '1',
+┊   ┊163┊          __typename: 'User',
+┊   ┊164┊        },
+┊   ┊165┊        {
+┊   ┊166┊          id: '5',
+┊   ┊167┊          __typename: 'User',
+┊   ┊168┊        },
+┊   ┊169┊      ],
+┊   ┊170┊      unreadMessages: 0,
+┊   ┊171┊      isGroup: false,
+┊   ┊172┊      messages: [
+┊   ┊173┊        {
+┊   ┊174┊          id: '1',
+┊   ┊175┊          __typename: 'Message',
+┊   ┊176┊          chat: {
+┊   ┊177┊            id: '3',
+┊   ┊178┊            __typename: 'Chat',
+┊   ┊179┊          },
+┊   ┊180┊          sender: {
+┊   ┊181┊            id: '1',
+┊   ┊182┊            __typename: 'User',
+┊   ┊183┊            name: 'Ethan Gonzalez',
+┊   ┊184┊          },
+┊   ┊185┊          content: 'You still there?',
+┊   ┊186┊          createdAt: '1514010200',
+┊   ┊187┊          type: 0,
+┊   ┊188┊          recipients: [
+┊   ┊189┊            {
+┊   ┊190┊              user: {
+┊   ┊191┊                id: '5',
+┊   ┊192┊                __typename: 'User',
+┊   ┊193┊              },
+┊   ┊194┊              message: {
+┊   ┊195┊                id: '1',
+┊   ┊196┊                __typename: 'Message',
+┊   ┊197┊                chat: {
+┊   ┊198┊                  id: '3',
+┊   ┊199┊                  __typename: 'Chat',
+┊   ┊200┊                },
+┊   ┊201┊              },
+┊   ┊202┊              __typename: 'Recipient',
+┊   ┊203┊              chat: {
+┊   ┊204┊                id: '3',
+┊   ┊205┊                __typename: 'Chat',
+┊   ┊206┊              },
+┊   ┊207┊              receivedAt: null,
+┊   ┊208┊              readAt: null,
+┊   ┊209┊            },
+┊   ┊210┊          ],
+┊   ┊211┊          ownership: true,
+┊   ┊212┊        },
+┊   ┊213┊      ],
+┊   ┊214┊    },
+┊   ┊215┊    {
+┊   ┊216┊      id: '6',
+┊   ┊217┊      __typename: 'Chat',
+┊   ┊218┊      name: 'Niccolò Belli',
+┊   ┊219┊      picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
+┊   ┊220┊      allTimeMembers: [
+┊   ┊221┊        {
+┊   ┊222┊          id: '1',
+┊   ┊223┊          __typename: 'User',
+┊   ┊224┊        },
+┊   ┊225┊        {
+┊   ┊226┊          id: '6',
+┊   ┊227┊          __typename: 'User',
+┊   ┊228┊        },
+┊   ┊229┊      ],
+┊   ┊230┊      unreadMessages: 0,
+┊   ┊231┊      messages: [],
+┊   ┊232┊      isGroup: false,
+┊   ┊233┊    },
+┊   ┊234┊    {
+┊   ┊235┊      id: '8',
+┊   ┊236┊      __typename: 'Chat',
+┊   ┊237┊      name: 'A user 0 group',
+┊   ┊238┊      picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊239┊      allTimeMembers: [
+┊   ┊240┊        {
+┊   ┊241┊          id: '1',
+┊   ┊242┊          __typename: 'User',
+┊   ┊243┊        },
+┊   ┊244┊        {
+┊   ┊245┊          id: '3',
+┊   ┊246┊          __typename: 'User',
+┊   ┊247┊        },
+┊   ┊248┊        {
+┊   ┊249┊          id: '4',
+┊   ┊250┊          __typename: 'User',
+┊   ┊251┊        },
+┊   ┊252┊        {
+┊   ┊253┊          id: '6',
+┊   ┊254┊          __typename: 'User',
+┊   ┊255┊        },
+┊   ┊256┊      ],
+┊   ┊257┊      unreadMessages: 1,
+┊   ┊258┊      isGroup: true,
+┊   ┊259┊      messages: [
+┊   ┊260┊        {
+┊   ┊261┊          id: '1',
+┊   ┊262┊          __typename: 'Message',
+┊   ┊263┊          chat: {
+┊   ┊264┊            id: '8',
+┊   ┊265┊            __typename: 'Chat',
+┊   ┊266┊          },
+┊   ┊267┊          sender: {
+┊   ┊268┊            id: '4',
+┊   ┊269┊            __typename: 'User',
+┊   ┊270┊            name: 'Katie Peterson',
+┊   ┊271┊          },
+┊   ┊272┊          content: 'Awesome!',
+┊   ┊273┊          createdAt: '1512830000',
+┊   ┊274┊          type: 0,
+┊   ┊275┊          recipients: [
+┊   ┊276┊            {
+┊   ┊277┊              user: {
+┊   ┊278┊                id: '1',
+┊   ┊279┊                __typename: 'User',
+┊   ┊280┊              },
+┊   ┊281┊              message: {
+┊   ┊282┊                id: '1',
+┊   ┊283┊                __typename: 'Message',
+┊   ┊284┊                chat: {
+┊   ┊285┊                  id: '8',
+┊   ┊286┊                  __typename: 'Chat',
+┊   ┊287┊                },
+┊   ┊288┊              },
+┊   ┊289┊              __typename: 'Recipient',
+┊   ┊290┊              chat: {
+┊   ┊291┊                id: '8',
+┊   ┊292┊                __typename: 'Chat',
+┊   ┊293┊              },
+┊   ┊294┊              receivedAt: null,
+┊   ┊295┊              readAt: null,
+┊   ┊296┊            },
+┊   ┊297┊            {
+┊   ┊298┊              user: {
+┊   ┊299┊                id: '6',
+┊   ┊300┊                __typename: 'User',
+┊   ┊301┊              },
+┊   ┊302┊              message: {
+┊   ┊303┊                id: '1',
+┊   ┊304┊                __typename: 'Message',
+┊   ┊305┊                chat: {
+┊   ┊306┊                  id: '8',
+┊   ┊307┊                  __typename: 'Chat',
+┊   ┊308┊                },
+┊   ┊309┊              },
+┊   ┊310┊              __typename: 'Recipient',
+┊   ┊311┊              chat: {
+┊   ┊312┊                id: '8',
+┊   ┊313┊                __typename: 'Chat',
+┊   ┊314┊              },
+┊   ┊315┊              receivedAt: null,
+┊   ┊316┊              readAt: null,
+┊   ┊317┊            },
+┊   ┊318┊          ],
+┊   ┊319┊          ownership: false,
+┊   ┊320┊        },
+┊   ┊321┊      ],
+┊   ┊322┊    },
+┊   ┊323┊  ];
+┊   ┊324┊
+┊   ┊325┊  beforeEach(async(() => {
+┊   ┊326┊    TestBed.configureTestingModule({
+┊   ┊327┊      declarations: [ChatsComponent, ChatsListComponent, ChatItemComponent],
+┊   ┊328┊      imports: [
+┊   ┊329┊        MatMenuModule,
+┊   ┊330┊        MatIconModule,
+┊   ┊331┊        MatButtonModule,
+┊   ┊332┊        MatListModule,
+┊   ┊333┊        ApolloTestingModule,
+┊   ┊334┊        RouterTestingModule,
+┊   ┊335┊      ],
+┊   ┊336┊      providers: [
+┊   ┊337┊        ChatsService,
+┊   ┊338┊        {
+┊   ┊339┊          provide: APOLLO_TESTING_CACHE,
+┊   ┊340┊          useFactory() {
+┊   ┊341┊            return new InMemoryCache({ dataIdFromObject });
+┊   ┊342┊          },
+┊   ┊343┊        },
+┊   ┊344┊      ],
+┊   ┊345┊      schemas: [NO_ERRORS_SCHEMA],
+┊   ┊346┊    }).compileComponents();
+┊   ┊347┊
+┊   ┊348┊    controller = TestBed.get(ApolloTestingController);
+┊   ┊349┊    apollo = TestBed.get(Apollo);
+┊   ┊350┊  }));
+┊   ┊351┊
+┊   ┊352┊  beforeEach(() => {
+┊   ┊353┊    fixture = TestBed.createComponent(ChatsComponent);
+┊   ┊354┊    component = fixture.componentInstance;
+┊   ┊355┊    fixture.detectChanges();
+┊   ┊356┊
+┊   ┊357┊    const req = controller.expectOne('GetChats', 'GetChats operation');
+┊   ┊358┊
+┊   ┊359┊    req.flush({
+┊   ┊360┊      data: {
+┊   ┊361┊        chats,
+┊   ┊362┊      },
+┊   ┊363┊    });
+┊   ┊364┊  });
+┊   ┊365┊
+┊   ┊366┊  it('should create', () => {
+┊   ┊367┊    expect(component).toBeTruthy();
+┊   ┊368┊  });
+┊   ┊369┊
+┊   ┊370┊  it('should display the chats', () => {
+┊   ┊371┊    fixture.whenStable().then(() => {
+┊   ┊372┊      fixture.detectChanges();
+┊   ┊373┊      el = fixture.debugElement;
+┊   ┊374┊      for (let i = 0; i < chats.length; i++) {
+┊   ┊375┊        expect(
+┊   ┊376┊          el.query(
+┊   ┊377┊            By.css(
+┊   ┊378┊              `app-chats-list > mat-list > mat-list-item:nth-child(${i +
+┊   ┊379┊                1}) > div > app-chat-item > div > div > div`,
+┊   ┊380┊            ),
+┊   ┊381┊          ).nativeElement.textContent,
+┊   ┊382┊        ).toContain(chats[i].name);
+┊   ┊383┊      }
+┊   ┊384┊    });
+┊   ┊385┊  });
+┊   ┊386┊
+┊   ┊387┊  afterAll(() => {
+┊   ┊388┊    controller.verify();
+┊   ┊389┊  });
+┊   ┊390┊});
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step2.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step4.md) |
|:--------------------------------|--------------------------------:|

[}]: #
