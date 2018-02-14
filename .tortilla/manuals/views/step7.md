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

Testing a service is a bit more complicated because we will need to mock our backend in order to get fake results instead of having to fire up the backend each time.
We are going to simply mock the HTTP calls, which is a well known practice in the REST API world. Since we are using HTTP to retrieve the data, it will work as well with Apollo client:

[{]: <helper> (diffStep "3.1" files="src/app/services/chats.service.spec.ts" module="client")

#### Step 3.1: Testing

##### Added src&#x2F;app&#x2F;services&#x2F;chats.service.spec.ts
```diff
@@ -0,0 +1,356 @@
+┊   ┊  1┊import { TestBed, inject } from '@angular/core/testing';
+┊   ┊  2┊
+┊   ┊  3┊import { ChatsService } from './chats.service';
+┊   ┊  4┊import {Apollo} from 'apollo-angular';
+┊   ┊  5┊import {HttpLink, HttpLinkModule, Options} from 'apollo-angular-link-http';
+┊   ┊  6┊import {HttpClientTestingModule, HttpTestingController} from '@angular/common/http/testing';
+┊   ┊  7┊import {defaultDataIdFromObject, InMemoryCache} from 'apollo-cache-inmemory';
+┊   ┊  8┊
+┊   ┊  9┊describe('ChatsService', () => {
+┊   ┊ 10┊  let httpMock: HttpTestingController;
+┊   ┊ 11┊  let httpLink: HttpLink;
+┊   ┊ 12┊  let apollo: Apollo;
+┊   ┊ 13┊
+┊   ┊ 14┊  const chats: any = [
+┊   ┊ 15┊    {
+┊   ┊ 16┊      id: '1',
+┊   ┊ 17┊      __typename: 'Chat',
+┊   ┊ 18┊      name: 'Avery Stewart',
+┊   ┊ 19┊      picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 20┊      allTimeMembers: [
+┊   ┊ 21┊        {
+┊   ┊ 22┊          id: '1',
+┊   ┊ 23┊          __typename: 'User',
+┊   ┊ 24┊        },
+┊   ┊ 25┊        {
+┊   ┊ 26┊          id: '3',
+┊   ┊ 27┊          __typename: 'User',
+┊   ┊ 28┊        }
+┊   ┊ 29┊      ],
+┊   ┊ 30┊      unreadMessages: 1,
+┊   ┊ 31┊      isGroup: false,
+┊   ┊ 32┊      messages: [
+┊   ┊ 33┊        {
+┊   ┊ 34┊          id: '1',
+┊   ┊ 35┊          chat: {
+┊   ┊ 36┊            id: '1',
+┊   ┊ 37┊            __typename: 'Chat',
+┊   ┊ 38┊          },
+┊   ┊ 39┊          __typename: 'Message',
+┊   ┊ 40┊          sender: {
+┊   ┊ 41┊            id: '3',
+┊   ┊ 42┊            __typename: 'User',
+┊   ┊ 43┊            name: 'Avery Stewart'
+┊   ┊ 44┊          },
+┊   ┊ 45┊          content: 'Yep!',
+┊   ┊ 46┊          createdAt: '1514035700',
+┊   ┊ 47┊          type: 0,
+┊   ┊ 48┊          recipients: [
+┊   ┊ 49┊            {
+┊   ┊ 50┊              user: {
+┊   ┊ 51┊                id: '1',
+┊   ┊ 52┊                __typename: 'User',
+┊   ┊ 53┊              },
+┊   ┊ 54┊              message: {
+┊   ┊ 55┊                id: '1',
+┊   ┊ 56┊                __typename: 'Message',
+┊   ┊ 57┊                chat: {
+┊   ┊ 58┊                  id: '1',
+┊   ┊ 59┊                  __typename: 'Chat',
+┊   ┊ 60┊                },
+┊   ┊ 61┊              },
+┊   ┊ 62┊              __typename: 'Recipient',
+┊   ┊ 63┊              chat: {
+┊   ┊ 64┊                id: '1',
+┊   ┊ 65┊                __typename: 'Chat',
+┊   ┊ 66┊              },
+┊   ┊ 67┊              receivedAt: null,
+┊   ┊ 68┊              readAt: null,
+┊   ┊ 69┊            }
+┊   ┊ 70┊          ],
+┊   ┊ 71┊          ownership: false,
+┊   ┊ 72┊        }
+┊   ┊ 73┊      ],
+┊   ┊ 74┊    },
+┊   ┊ 75┊    {
+┊   ┊ 76┊      id: '2',
+┊   ┊ 77┊      __typename: 'Chat',
+┊   ┊ 78┊      name: 'Katie Peterson',
+┊   ┊ 79┊      picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
+┊   ┊ 80┊      allTimeMembers: [
+┊   ┊ 81┊        {
+┊   ┊ 82┊          id: '1',
+┊   ┊ 83┊          __typename: 'User',
+┊   ┊ 84┊        },
+┊   ┊ 85┊        {
+┊   ┊ 86┊          id: '4',
+┊   ┊ 87┊          __typename: 'User',
+┊   ┊ 88┊        }
+┊   ┊ 89┊      ],
+┊   ┊ 90┊      unreadMessages: 0,
+┊   ┊ 91┊      isGroup: false,
+┊   ┊ 92┊      messages: [
+┊   ┊ 93┊        {
+┊   ┊ 94┊          id: '1',
+┊   ┊ 95┊          chat: {
+┊   ┊ 96┊            id: '2',
+┊   ┊ 97┊            __typename: 'Chat',
+┊   ┊ 98┊          },
+┊   ┊ 99┊          __typename: 'Message',
+┊   ┊100┊          sender: {
+┊   ┊101┊            id: '1',
+┊   ┊102┊            __typename: 'User',
+┊   ┊103┊            name: 'Ethan Gonzalez'
+┊   ┊104┊          },
+┊   ┊105┊          content: 'Hey, it\'s me',
+┊   ┊106┊          createdAt: '1514031800',
+┊   ┊107┊          type: 0,
+┊   ┊108┊          recipients: [
+┊   ┊109┊            {
+┊   ┊110┊              user: {
+┊   ┊111┊                id: '4',
+┊   ┊112┊                __typename: 'User',
+┊   ┊113┊              },
+┊   ┊114┊              message: {
+┊   ┊115┊                id: '1',
+┊   ┊116┊                __typename: 'Message',
+┊   ┊117┊                chat: {
+┊   ┊118┊                  id: '2',
+┊   ┊119┊                  __typename: 'Chat',
+┊   ┊120┊                },
+┊   ┊121┊              },
+┊   ┊122┊              __typename: 'Recipient',
+┊   ┊123┊              chat: {
+┊   ┊124┊                id: '2',
+┊   ┊125┊                __typename: 'Chat',
+┊   ┊126┊              },
+┊   ┊127┊              receivedAt: null,
+┊   ┊128┊              readAt: null,
+┊   ┊129┊            }
+┊   ┊130┊          ],
+┊   ┊131┊          ownership: true
+┊   ┊132┊        }
+┊   ┊133┊      ],
+┊   ┊134┊    },
+┊   ┊135┊    {
+┊   ┊136┊      id: '3',
+┊   ┊137┊      __typename: 'Chat',
+┊   ┊138┊      name: 'Ray Edwards',
+┊   ┊139┊      picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
+┊   ┊140┊      allTimeMembers: [
+┊   ┊141┊        {
+┊   ┊142┊          id: '1',
+┊   ┊143┊          __typename: 'User',
+┊   ┊144┊        },
+┊   ┊145┊        {
+┊   ┊146┊          id: '5',
+┊   ┊147┊          __typename: 'User',
+┊   ┊148┊        }
+┊   ┊149┊      ],
+┊   ┊150┊      unreadMessages: 0,
+┊   ┊151┊      isGroup: false,
+┊   ┊152┊      messages: [
+┊   ┊153┊        {
+┊   ┊154┊          id: '1',
+┊   ┊155┊          __typename: 'Message',
+┊   ┊156┊          chat: {
+┊   ┊157┊            id: '3',
+┊   ┊158┊            __typename: 'Chat',
+┊   ┊159┊          },
+┊   ┊160┊          sender: {
+┊   ┊161┊            id: '1',
+┊   ┊162┊            __typename: 'User',
+┊   ┊163┊            name: 'Ethan Gonzalez'
+┊   ┊164┊          },
+┊   ┊165┊          content: 'You still there?',
+┊   ┊166┊          createdAt: '1514010200',
+┊   ┊167┊          type: 0,
+┊   ┊168┊          recipients: [
+┊   ┊169┊            {
+┊   ┊170┊              user: {
+┊   ┊171┊                id: '5',
+┊   ┊172┊                __typename: 'User',
+┊   ┊173┊              },
+┊   ┊174┊              message: {
+┊   ┊175┊                id: '1',
+┊   ┊176┊                __typename: 'Message',
+┊   ┊177┊                chat: {
+┊   ┊178┊                  id: '3',
+┊   ┊179┊                  __typename: 'Chat',
+┊   ┊180┊                },
+┊   ┊181┊              },
+┊   ┊182┊              __typename: 'Recipient',
+┊   ┊183┊              chat: {
+┊   ┊184┊                id: '3',
+┊   ┊185┊                __typename: 'Chat',
+┊   ┊186┊              },
+┊   ┊187┊              receivedAt: null,
+┊   ┊188┊              readAt: null
+┊   ┊189┊            }
+┊   ┊190┊          ],
+┊   ┊191┊          ownership: true
+┊   ┊192┊        }
+┊   ┊193┊      ],
+┊   ┊194┊    },
+┊   ┊195┊    {
+┊   ┊196┊      id: '6',
+┊   ┊197┊      __typename: 'Chat',
+┊   ┊198┊      name: 'Niccolò Belli',
+┊   ┊199┊      picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
+┊   ┊200┊      allTimeMembers: [
+┊   ┊201┊        {
+┊   ┊202┊          id: '1',
+┊   ┊203┊          __typename: 'User',
+┊   ┊204┊        },
+┊   ┊205┊        {
+┊   ┊206┊          id: '6',
+┊   ┊207┊          __typename: 'User',
+┊   ┊208┊        }
+┊   ┊209┊      ],
+┊   ┊210┊      unreadMessages: 0,
+┊   ┊211┊      messages: [],
+┊   ┊212┊      isGroup: false
+┊   ┊213┊    },
+┊   ┊214┊    {
+┊   ┊215┊      id: '8',
+┊   ┊216┊      __typename: 'Chat',
+┊   ┊217┊      name: 'A user 0 group',
+┊   ┊218┊      picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊219┊      allTimeMembers: [
+┊   ┊220┊        {
+┊   ┊221┊          id: '1',
+┊   ┊222┊          __typename: 'User',
+┊   ┊223┊        },
+┊   ┊224┊        {
+┊   ┊225┊          id: '3',
+┊   ┊226┊          __typename: 'User',
+┊   ┊227┊        },
+┊   ┊228┊        {
+┊   ┊229┊          id: '4',
+┊   ┊230┊          __typename: 'User',
+┊   ┊231┊        },
+┊   ┊232┊        {
+┊   ┊233┊          id: '6',
+┊   ┊234┊          __typename: 'User',
+┊   ┊235┊        },
+┊   ┊236┊      ],
+┊   ┊237┊      unreadMessages: 1,
+┊   ┊238┊      isGroup: true,
+┊   ┊239┊      messages: [
+┊   ┊240┊        {
+┊   ┊241┊          id: '1',
+┊   ┊242┊          __typename: 'Message',
+┊   ┊243┊          chat: {
+┊   ┊244┊            id: '8',
+┊   ┊245┊            __typename: 'Chat',
+┊   ┊246┊          },
+┊   ┊247┊          sender: {
+┊   ┊248┊            id: '4',
+┊   ┊249┊            __typename: 'User',
+┊   ┊250┊            name: 'Katie Peterson'
+┊   ┊251┊          },
+┊   ┊252┊          content: 'Awesome!',
+┊   ┊253┊          createdAt: '1512830000',
+┊   ┊254┊          type: 0,
+┊   ┊255┊          recipients: [
+┊   ┊256┊            {
+┊   ┊257┊              user: {
+┊   ┊258┊                id: '1',
+┊   ┊259┊                __typename: 'User',
+┊   ┊260┊              },
+┊   ┊261┊              message: {
+┊   ┊262┊                id: '1',
+┊   ┊263┊                __typename: 'Message',
+┊   ┊264┊                chat: {
+┊   ┊265┊                  id: '8',
+┊   ┊266┊                  __typename: 'Chat',
+┊   ┊267┊                },
+┊   ┊268┊              },
+┊   ┊269┊              __typename: 'Recipient',
+┊   ┊270┊              chat: {
+┊   ┊271┊                id: '8',
+┊   ┊272┊                __typename: 'Chat',
+┊   ┊273┊              },
+┊   ┊274┊              receivedAt: null,
+┊   ┊275┊              readAt: null
+┊   ┊276┊            },
+┊   ┊277┊            {
+┊   ┊278┊              user: {
+┊   ┊279┊                id: '6',
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
+┊   ┊296┊              readAt: null
+┊   ┊297┊            }
+┊   ┊298┊          ],
+┊   ┊299┊          ownership: false
+┊   ┊300┊        }
+┊   ┊301┊      ],
+┊   ┊302┊    },
+┊   ┊303┊  ];
+┊   ┊304┊
+┊   ┊305┊  beforeEach(() => {
+┊   ┊306┊    TestBed.configureTestingModule({
+┊   ┊307┊      imports: [
+┊   ┊308┊        HttpLinkModule,
+┊   ┊309┊        // HttpClientModule,
+┊   ┊310┊        HttpClientTestingModule,
+┊   ┊311┊      ],
+┊   ┊312┊      providers: [
+┊   ┊313┊        ChatsService,
+┊   ┊314┊        Apollo,
+┊   ┊315┊      ]
+┊   ┊316┊    });
+┊   ┊317┊
+┊   ┊318┊    httpMock = TestBed.get(HttpTestingController);
+┊   ┊319┊    httpLink = TestBed.get(HttpLink);
+┊   ┊320┊    apollo = TestBed.get(Apollo);
+┊   ┊321┊
+┊   ┊322┊    apollo.create({
+┊   ┊323┊      link: httpLink.create(<Options>{ uri: 'http://localhost:3000/graphql' }),
+┊   ┊324┊      cache: new InMemoryCache({
+┊   ┊325┊        dataIdFromObject: (object: any) => {
+┊   ┊326┊          switch (object.__typename) {
+┊   ┊327┊            case 'Message': return `${object.chat.id}:${object.id}`; // use `chatId` prefix and `messageId` as the primary key
+┊   ┊328┊            default: return defaultDataIdFromObject(object); // fall back to default handling
+┊   ┊329┊          }
+┊   ┊330┊        }
+┊   ┊331┊      }),
+┊   ┊332┊    });
+┊   ┊333┊  });
+┊   ┊334┊
+┊   ┊335┊  it('should be created', inject([ChatsService], (service: ChatsService) => {
+┊   ┊336┊    expect(service).toBeTruthy();
+┊   ┊337┊  }));
+┊   ┊338┊
+┊   ┊339┊  it('should get chats', inject([ChatsService], (service: ChatsService) => {
+┊   ┊340┊    service.getChats().chats$.subscribe(_chats => {
+┊   ┊341┊      expect(_chats.length).toEqual(chats.length);
+┊   ┊342┊      for (let i = 0; i < _chats.length; i++) {
+┊   ┊343┊        expect(_chats[i]).toEqual(chats[i]);
+┊   ┊344┊      }
+┊   ┊345┊    });
+┊   ┊346┊
+┊   ┊347┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊348┊    expect(req.request.method).toBe('POST');
+┊   ┊349┊    req.flush({
+┊   ┊350┊      data: {
+┊   ┊351┊        chats
+┊   ┊352┊      }
+┊   ┊353┊    });
+┊   ┊354┊    httpMock.verify();
+┊   ┊355┊  }));
+┊   ┊356┊});
```

[}]: #

In the last example we are going to test a container component, which makes use of several services and multiple other components:

[{]: <helper> (diffStep "3.1" files="src/app/chats-lister/containers/chats/chats.component.spec.ts" module="client")

#### Step 3.1: Testing

##### Added src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -0,0 +1,387 @@
+┊   ┊  1┊import { async, ComponentFixture, TestBed } from '@angular/core/testing';
+┊   ┊  2┊
+┊   ┊  3┊import { ChatsComponent } from './chats.component';
+┊   ┊  4┊import {DebugElement, NO_ERRORS_SCHEMA} from '@angular/core';
+┊   ┊  5┊import {ChatsListComponent} from '../../components/chats-list/chats-list.component';
+┊   ┊  6┊import {ChatItemComponent} from '../../components/chat-item/chat-item.component';
+┊   ┊  7┊import {TruncateModule} from 'ng2-truncate';
+┊   ┊  8┊import {MatButtonModule, MatIconModule, MatListModule, MatMenuModule} from '@angular/material';
+┊   ┊  9┊import {ChatsService} from '../../../services/chats.service';
+┊   ┊ 10┊import {Apollo} from 'apollo-angular';
+┊   ┊ 11┊import {HttpClientTestingModule, HttpTestingController} from '@angular/common/http/testing';
+┊   ┊ 12┊import {HttpLink, HttpLinkModule, Options} from 'apollo-angular-link-http';
+┊   ┊ 13┊import {defaultDataIdFromObject, InMemoryCache} from 'apollo-cache-inmemory';
+┊   ┊ 14┊import {By} from '@angular/platform-browser';
+┊   ┊ 15┊import {RouterTestingModule} from '@angular/router/testing';
+┊   ┊ 16┊
+┊   ┊ 17┊describe('ChatsComponent', () => {
+┊   ┊ 18┊  let component: ChatsComponent;
+┊   ┊ 19┊  let fixture: ComponentFixture<ChatsComponent>;
+┊   ┊ 20┊  let el: DebugElement;
+┊   ┊ 21┊
+┊   ┊ 22┊  let httpMock: HttpTestingController;
+┊   ┊ 23┊  let httpLink: HttpLink;
+┊   ┊ 24┊  let apollo: Apollo;
+┊   ┊ 25┊
+┊   ┊ 26┊  const chats: any = [
+┊   ┊ 27┊    {
+┊   ┊ 28┊      id: '1',
+┊   ┊ 29┊      __typename: 'Chat',
+┊   ┊ 30┊      name: 'Avery Stewart',
+┊   ┊ 31┊      picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
+┊   ┊ 32┊      allTimeMembers: [
+┊   ┊ 33┊        {
+┊   ┊ 34┊          id: '1',
+┊   ┊ 35┊          __typename: 'User',
+┊   ┊ 36┊        },
+┊   ┊ 37┊        {
+┊   ┊ 38┊          id: '3',
+┊   ┊ 39┊          __typename: 'User',
+┊   ┊ 40┊        }
+┊   ┊ 41┊      ],
+┊   ┊ 42┊      unreadMessages: 1,
+┊   ┊ 43┊      isGroup: false,
+┊   ┊ 44┊      messages: [
+┊   ┊ 45┊        {
+┊   ┊ 46┊          id: '1',
+┊   ┊ 47┊          chat: {
+┊   ┊ 48┊            id: '1',
+┊   ┊ 49┊            __typename: 'Chat',
+┊   ┊ 50┊          },
+┊   ┊ 51┊          __typename: 'Message',
+┊   ┊ 52┊          sender: {
+┊   ┊ 53┊            id: '3',
+┊   ┊ 54┊            __typename: 'User',
+┊   ┊ 55┊            name: 'Avery Stewart'
+┊   ┊ 56┊          },
+┊   ┊ 57┊          content: 'Yep!',
+┊   ┊ 58┊          createdAt: '1514035700',
+┊   ┊ 59┊          type: 0,
+┊   ┊ 60┊          recipients: [
+┊   ┊ 61┊            {
+┊   ┊ 62┊              user: {
+┊   ┊ 63┊                id: '1',
+┊   ┊ 64┊                __typename: 'User',
+┊   ┊ 65┊              },
+┊   ┊ 66┊              message: {
+┊   ┊ 67┊                id: '1',
+┊   ┊ 68┊                __typename: 'Message',
+┊   ┊ 69┊                chat: {
+┊   ┊ 70┊                  id: '1',
+┊   ┊ 71┊                  __typename: 'Chat',
+┊   ┊ 72┊                },
+┊   ┊ 73┊              },
+┊   ┊ 74┊              __typename: 'Recipient',
+┊   ┊ 75┊              chat: {
+┊   ┊ 76┊                id: '1',
+┊   ┊ 77┊                __typename: 'Chat',
+┊   ┊ 78┊              },
+┊   ┊ 79┊              receivedAt: null,
+┊   ┊ 80┊              readAt: null,
+┊   ┊ 81┊            }
+┊   ┊ 82┊          ],
+┊   ┊ 83┊          ownership: false,
+┊   ┊ 84┊        }
+┊   ┊ 85┊      ],
+┊   ┊ 86┊    },
+┊   ┊ 87┊    {
+┊   ┊ 88┊      id: '2',
+┊   ┊ 89┊      __typename: 'Chat',
+┊   ┊ 90┊      name: 'Katie Peterson',
+┊   ┊ 91┊      picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
+┊   ┊ 92┊      allTimeMembers: [
+┊   ┊ 93┊        {
+┊   ┊ 94┊          id: '1',
+┊   ┊ 95┊          __typename: 'User',
+┊   ┊ 96┊        },
+┊   ┊ 97┊        {
+┊   ┊ 98┊          id: '4',
+┊   ┊ 99┊          __typename: 'User',
+┊   ┊100┊        }
+┊   ┊101┊      ],
+┊   ┊102┊      unreadMessages: 0,
+┊   ┊103┊      isGroup: false,
+┊   ┊104┊      messages: [
+┊   ┊105┊        {
+┊   ┊106┊          id: '1',
+┊   ┊107┊          chat: {
+┊   ┊108┊            id: '2',
+┊   ┊109┊            __typename: 'Chat',
+┊   ┊110┊          },
+┊   ┊111┊          __typename: 'Message',
+┊   ┊112┊          sender: {
+┊   ┊113┊            id: '1',
+┊   ┊114┊            __typename: 'User',
+┊   ┊115┊            name: 'Ethan Gonzalez'
+┊   ┊116┊          },
+┊   ┊117┊          content: 'Hey, it\'s me',
+┊   ┊118┊          createdAt: '1514031800',
+┊   ┊119┊          type: 0,
+┊   ┊120┊          recipients: [
+┊   ┊121┊            {
+┊   ┊122┊              user: {
+┊   ┊123┊                id: '4',
+┊   ┊124┊                __typename: 'User',
+┊   ┊125┊              },
+┊   ┊126┊              message: {
+┊   ┊127┊                id: '1',
+┊   ┊128┊                __typename: 'Message',
+┊   ┊129┊                chat: {
+┊   ┊130┊                  id: '2',
+┊   ┊131┊                  __typename: 'Chat',
+┊   ┊132┊                },
+┊   ┊133┊              },
+┊   ┊134┊              __typename: 'Recipient',
+┊   ┊135┊              chat: {
+┊   ┊136┊                id: '2',
+┊   ┊137┊                __typename: 'Chat',
+┊   ┊138┊              },
+┊   ┊139┊              receivedAt: null,
+┊   ┊140┊              readAt: null,
+┊   ┊141┊            }
+┊   ┊142┊          ],
+┊   ┊143┊          ownership: true
+┊   ┊144┊        }
+┊   ┊145┊      ],
+┊   ┊146┊    },
+┊   ┊147┊    {
+┊   ┊148┊      id: '3',
+┊   ┊149┊      __typename: 'Chat',
+┊   ┊150┊      name: 'Ray Edwards',
+┊   ┊151┊      picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
+┊   ┊152┊      allTimeMembers: [
+┊   ┊153┊        {
+┊   ┊154┊          id: '1',
+┊   ┊155┊          __typename: 'User',
+┊   ┊156┊        },
+┊   ┊157┊        {
+┊   ┊158┊          id: '5',
+┊   ┊159┊          __typename: 'User',
+┊   ┊160┊        }
+┊   ┊161┊      ],
+┊   ┊162┊      unreadMessages: 0,
+┊   ┊163┊      isGroup: false,
+┊   ┊164┊      messages: [
+┊   ┊165┊        {
+┊   ┊166┊          id: '1',
+┊   ┊167┊          __typename: 'Message',
+┊   ┊168┊          chat: {
+┊   ┊169┊            id: '3',
+┊   ┊170┊            __typename: 'Chat',
+┊   ┊171┊          },
+┊   ┊172┊          sender: {
+┊   ┊173┊            id: '1',
+┊   ┊174┊            __typename: 'User',
+┊   ┊175┊            name: 'Ethan Gonzalez'
+┊   ┊176┊          },
+┊   ┊177┊          content: 'You still there?',
+┊   ┊178┊          createdAt: '1514010200',
+┊   ┊179┊          type: 0,
+┊   ┊180┊          recipients: [
+┊   ┊181┊            {
+┊   ┊182┊              user: {
+┊   ┊183┊                id: '5',
+┊   ┊184┊                __typename: 'User',
+┊   ┊185┊              },
+┊   ┊186┊              message: {
+┊   ┊187┊                id: '1',
+┊   ┊188┊                __typename: 'Message',
+┊   ┊189┊                chat: {
+┊   ┊190┊                  id: '3',
+┊   ┊191┊                  __typename: 'Chat',
+┊   ┊192┊                },
+┊   ┊193┊              },
+┊   ┊194┊              __typename: 'Recipient',
+┊   ┊195┊              chat: {
+┊   ┊196┊                id: '3',
+┊   ┊197┊                __typename: 'Chat',
+┊   ┊198┊              },
+┊   ┊199┊              receivedAt: null,
+┊   ┊200┊              readAt: null
+┊   ┊201┊            }
+┊   ┊202┊          ],
+┊   ┊203┊          ownership: true
+┊   ┊204┊        }
+┊   ┊205┊      ],
+┊   ┊206┊    },
+┊   ┊207┊    {
+┊   ┊208┊      id: '6',
+┊   ┊209┊      __typename: 'Chat',
+┊   ┊210┊      name: 'Niccolò Belli',
+┊   ┊211┊      picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
+┊   ┊212┊      allTimeMembers: [
+┊   ┊213┊        {
+┊   ┊214┊          id: '1',
+┊   ┊215┊          __typename: 'User',
+┊   ┊216┊        },
+┊   ┊217┊        {
+┊   ┊218┊          id: '6',
+┊   ┊219┊          __typename: 'User',
+┊   ┊220┊        }
+┊   ┊221┊      ],
+┊   ┊222┊      unreadMessages: 0,
+┊   ┊223┊      messages: [],
+┊   ┊224┊      isGroup: false
+┊   ┊225┊    },
+┊   ┊226┊    {
+┊   ┊227┊      id: '8',
+┊   ┊228┊      __typename: 'Chat',
+┊   ┊229┊      name: 'A user 0 group',
+┊   ┊230┊      picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
+┊   ┊231┊      allTimeMembers: [
+┊   ┊232┊        {
+┊   ┊233┊          id: '1',
+┊   ┊234┊          __typename: 'User',
+┊   ┊235┊        },
+┊   ┊236┊        {
+┊   ┊237┊          id: '3',
+┊   ┊238┊          __typename: 'User',
+┊   ┊239┊        },
+┊   ┊240┊        {
+┊   ┊241┊          id: '4',
+┊   ┊242┊          __typename: 'User',
+┊   ┊243┊        },
+┊   ┊244┊        {
+┊   ┊245┊          id: '6',
+┊   ┊246┊          __typename: 'User',
+┊   ┊247┊        },
+┊   ┊248┊      ],
+┊   ┊249┊      unreadMessages: 1,
+┊   ┊250┊      isGroup: true,
+┊   ┊251┊      messages: [
+┊   ┊252┊        {
+┊   ┊253┊          id: '1',
+┊   ┊254┊          __typename: 'Message',
+┊   ┊255┊          chat: {
+┊   ┊256┊            id: '8',
+┊   ┊257┊            __typename: 'Chat',
+┊   ┊258┊          },
+┊   ┊259┊          sender: {
+┊   ┊260┊            id: '4',
+┊   ┊261┊            __typename: 'User',
+┊   ┊262┊            name: 'Katie Peterson'
+┊   ┊263┊          },
+┊   ┊264┊          content: 'Awesome!',
+┊   ┊265┊          createdAt: '1512830000',
+┊   ┊266┊          type: 0,
+┊   ┊267┊          recipients: [
+┊   ┊268┊            {
+┊   ┊269┊              user: {
+┊   ┊270┊                id: '1',
+┊   ┊271┊                __typename: 'User',
+┊   ┊272┊              },
+┊   ┊273┊              message: {
+┊   ┊274┊                id: '1',
+┊   ┊275┊                __typename: 'Message',
+┊   ┊276┊                chat: {
+┊   ┊277┊                  id: '8',
+┊   ┊278┊                  __typename: 'Chat',
+┊   ┊279┊                },
+┊   ┊280┊              },
+┊   ┊281┊              __typename: 'Recipient',
+┊   ┊282┊              chat: {
+┊   ┊283┊                id: '8',
+┊   ┊284┊                __typename: 'Chat',
+┊   ┊285┊              },
+┊   ┊286┊              receivedAt: null,
+┊   ┊287┊              readAt: null
+┊   ┊288┊            },
+┊   ┊289┊            {
+┊   ┊290┊              user: {
+┊   ┊291┊                id: '6',
+┊   ┊292┊                __typename: 'User',
+┊   ┊293┊              },
+┊   ┊294┊              message: {
+┊   ┊295┊                id: '1',
+┊   ┊296┊                __typename: 'Message',
+┊   ┊297┊                chat: {
+┊   ┊298┊                  id: '8',
+┊   ┊299┊                  __typename: 'Chat',
+┊   ┊300┊                },
+┊   ┊301┊              },
+┊   ┊302┊              __typename: 'Recipient',
+┊   ┊303┊              chat: {
+┊   ┊304┊                id: '8',
+┊   ┊305┊                __typename: 'Chat',
+┊   ┊306┊              },
+┊   ┊307┊              receivedAt: null,
+┊   ┊308┊              readAt: null
+┊   ┊309┊            }
+┊   ┊310┊          ],
+┊   ┊311┊          ownership: false
+┊   ┊312┊        }
+┊   ┊313┊      ],
+┊   ┊314┊    },
+┊   ┊315┊  ];
+┊   ┊316┊
+┊   ┊317┊  beforeEach(async(() => {
+┊   ┊318┊    TestBed.configureTestingModule({
+┊   ┊319┊      declarations: [
+┊   ┊320┊        ChatsComponent,
+┊   ┊321┊        ChatsListComponent,
+┊   ┊322┊        ChatItemComponent
+┊   ┊323┊      ],
+┊   ┊324┊      imports: [
+┊   ┊325┊        MatMenuModule,
+┊   ┊326┊        MatIconModule,
+┊   ┊327┊        MatButtonModule,
+┊   ┊328┊        MatListModule,
+┊   ┊329┊        TruncateModule,
+┊   ┊330┊        HttpLinkModule,
+┊   ┊331┊        HttpClientTestingModule,
+┊   ┊332┊        RouterTestingModule
+┊   ┊333┊      ],
+┊   ┊334┊      providers: [
+┊   ┊335┊        ChatsService,
+┊   ┊336┊        Apollo,
+┊   ┊337┊      ],
+┊   ┊338┊      schemas: [NO_ERRORS_SCHEMA]
+┊   ┊339┊    })
+┊   ┊340┊      .compileComponents();
+┊   ┊341┊
+┊   ┊342┊    httpMock = TestBed.get(HttpTestingController);
+┊   ┊343┊    httpLink = TestBed.get(HttpLink);
+┊   ┊344┊    apollo = TestBed.get(Apollo);
+┊   ┊345┊
+┊   ┊346┊    apollo.create({
+┊   ┊347┊      link: httpLink.create(<Options>{ uri: 'http://localhost:3000/graphql' }),
+┊   ┊348┊      cache: new InMemoryCache({
+┊   ┊349┊        dataIdFromObject: (object: any) => {
+┊   ┊350┊          switch (object.__typename) {
+┊   ┊351┊            case 'Message': return `${object.chat.id}:${object.id}`; // use `chatId` prefix and `messageId` as the primary key
+┊   ┊352┊            default: return defaultDataIdFromObject(object); // fall back to default handling
+┊   ┊353┊          }
+┊   ┊354┊        }
+┊   ┊355┊      }),
+┊   ┊356┊    });
+┊   ┊357┊  }));
+┊   ┊358┊
+┊   ┊359┊  beforeEach(() => {
+┊   ┊360┊    fixture = TestBed.createComponent(ChatsComponent);
+┊   ┊361┊    component = fixture.componentInstance;
+┊   ┊362┊    fixture.detectChanges();
+┊   ┊363┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊364┊    req.flush({
+┊   ┊365┊      data: {
+┊   ┊366┊        chats
+┊   ┊367┊      }
+┊   ┊368┊    });
+┊   ┊369┊  });
+┊   ┊370┊
+┊   ┊371┊  it('should create', () => {
+┊   ┊372┊    expect(component).toBeTruthy();
+┊   ┊373┊  });
+┊   ┊374┊
+┊   ┊375┊  it('should display the chats', () => {
+┊   ┊376┊    fixture.whenStable().then(() => {
+┊   ┊377┊      fixture.detectChanges();
+┊   ┊378┊      el = fixture.debugElement;
+┊   ┊379┊      for (let i = 0; i < chats.length; i++) {
+┊   ┊380┊        expect(el.query(By.css(`app-chats-list > mat-list > mat-list-item:nth-child(${i + 1}) > div > app-chat-item > div > div > div`))
+┊   ┊381┊          .nativeElement.textContent).toContain(chats[i].name);
+┊   ┊382┊      }
+┊   ┊383┊    });
+┊   ┊384┊
+┊   ┊385┊    httpMock.verify();
+┊   ┊386┊  });
+┊   ┊387┊});
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step6.md) | [Next Step >](step8.md) |
|:--------------------------------|--------------------------------:|

[}]: #
