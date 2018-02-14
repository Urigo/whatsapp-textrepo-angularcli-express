# Step 9: Mutations

[//]: # (head-end)


In addition to fetching data using queries, Apollo also helps you handle GraphQL mutations. In GraphQL, mutations are identical to queries in syntax, the only difference being that you use the keyword mutation instead of query to indicate that the root fields on this query are going to be performing writes to the backend.
GraphQL mutations represent two things in one query string:

1. The mutation field name with arguments, which represents the actual operation to be done on the server.
2. The fields you want back from the result of the mutation to update the client.

When we use mutations in Apollo, the result is typically integrated into the cache automatically based on the id of the result, which in turn updates the UI automatically, so we often don't need to explicitly handle the results. In order for the client to correctly do this, we need to ensure we select the necessary fields in the result. One good strategy can be to simply ask for any fields that might have been affected by the mutation. Alternatively, you can use fragments to share the fields between a query and a mutation that updates that query.

## Server

Finally we're going to create our mutations in the server:

[{]: <helper> (diffStep "3.1" module="server")

#### Step 3.1: Add mutations

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,6 +1,7 @@
-┊1┊ ┊import { Chat, db, Message, Recipient, User } from "../db";
+┊ ┊1┊import { Chat, db, Message, MessageType, Recipient, User } from "../db";
 ┊2┊2┊import { IResolvers } from "graphql-tools/dist/Interfaces";
 ┊3┊3┊import { ChatQueryArgs } from "../types";
+┊ ┊4┊import * as moment from "moment";
 ┊4┊5┊
 ┊5┊6┊let users = db.users;
 ┊6┊7┊let chats = db.chats;
```
```diff
@@ -13,6 +14,276 @@
 ┊ 13┊ 14┊    chats: (): Chat[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser)),
 ┊ 14┊ 15┊    chat: (obj: any, {chatId}: ChatQueryArgs): Chat | null => chats.find(chat => chat.id === Number(chatId)) || null,
 ┊ 15┊ 16┊  },
+┊   ┊ 17┊  Mutation: {
+┊   ┊ 18┊    addChat: (obj: any, {recipientId}: any): Chat => {
+┊   ┊ 19┊      if (!users.find(user => user.id === recipientId)) {
+┊   ┊ 20┊        throw new Error(`Recipient ${recipientId} doesn't exist.`);
+┊   ┊ 21┊      }
+┊   ┊ 22┊
+┊   ┊ 23┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser) && chat.allTimeMemberIds.includes(recipientId));
+┊   ┊ 24┊      if (chat) {
+┊   ┊ 25┊        // Chat already exists. Both users are already in the allTimeMemberIds array
+┊   ┊ 26┊        const chatId = chat.id;
+┊   ┊ 27┊        if (!chat.listingMemberIds.includes(currentUser)) {
+┊   ┊ 28┊          // The chat isn't listed for the current user. Add him to the memberIds
+┊   ┊ 29┊          chat.listingMemberIds.push(currentUser);
+┊   ┊ 30┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser);
+┊   ┊ 31┊          return chat;
+┊   ┊ 32┊        } else {
+┊   ┊ 33┊          throw new Error(`Chat already exists.`);
+┊   ┊ 34┊        }
+┊   ┊ 35┊      } else {
+┊   ┊ 36┊        // Create the chat
+┊   ┊ 37┊        const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
+┊   ┊ 38┊        const chat: Chat = {
+┊   ┊ 39┊          id,
+┊   ┊ 40┊          name: null,
+┊   ┊ 41┊          picture: null,
+┊   ┊ 42┊          adminIds: null,
+┊   ┊ 43┊          ownerId: null,
+┊   ┊ 44┊          allTimeMemberIds: [currentUser, recipientId],
+┊   ┊ 45┊          // Chat will not be listed to the other user until the first message gets written
+┊   ┊ 46┊          listingMemberIds: [currentUser],
+┊   ┊ 47┊          actualGroupMemberIds: null,
+┊   ┊ 48┊          messages: [],
+┊   ┊ 49┊        };
+┊   ┊ 50┊        chats.push(chat);
+┊   ┊ 51┊        return chat;
+┊   ┊ 52┊      }
+┊   ┊ 53┊    },
+┊   ┊ 54┊    addGroup: (obj: any, {recipientIds, groupName}: any): Chat => {
+┊   ┊ 55┊      recipientIds.forEach((recipientId: any) => {
+┊   ┊ 56┊        if (!users.find(user => user.id === recipientId)) {
+┊   ┊ 57┊          throw new Error(`Recipient ${recipientId} doesn't exist.`);
+┊   ┊ 58┊        }
+┊   ┊ 59┊      });
+┊   ┊ 60┊
+┊   ┊ 61┊      const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
+┊   ┊ 62┊      const chat: Chat = {
+┊   ┊ 63┊        id,
+┊   ┊ 64┊        name: groupName,
+┊   ┊ 65┊        picture: null,
+┊   ┊ 66┊        adminIds: [currentUser],
+┊   ┊ 67┊        ownerId: currentUser,
+┊   ┊ 68┊        allTimeMemberIds: [currentUser, ...recipientIds],
+┊   ┊ 69┊        listingMemberIds: [currentUser, ...recipientIds],
+┊   ┊ 70┊        actualGroupMemberIds: [currentUser, ...recipientIds],
+┊   ┊ 71┊        messages: [],
+┊   ┊ 72┊      };
+┊   ┊ 73┊      chats.push(chat);
+┊   ┊ 74┊      return chat;
+┊   ┊ 75┊    },
+┊   ┊ 76┊    removeChat: (obj: any, {chatId}: any): number => {
+┊   ┊ 77┊      const chat = chats.find(chat => chat.id === chatId);
+┊   ┊ 78┊
+┊   ┊ 79┊      if (!chat) {
+┊   ┊ 80┊        throw new Error(`The chat ${chatId} doesn't exist.`);
+┊   ┊ 81┊      }
+┊   ┊ 82┊
+┊   ┊ 83┊      if (!chat.name) {
+┊   ┊ 84┊        // Chat
+┊   ┊ 85┊        if (!chat.listingMemberIds.includes(currentUser)) {
+┊   ┊ 86┊          throw new Error(`The user is not a member of the chat ${chatId}.`);
+┊   ┊ 87┊        }
+┊   ┊ 88┊
+┊   ┊ 89┊        // Instead of chaining map and filter we can loop once using reduce
+┊   ┊ 90┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
+┊   ┊ 91┊          // Remove the current user from the message holders
+┊   ┊ 92┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊ 93┊
+┊   ┊ 94┊          if (message.holderIds.length !== 0) {
+┊   ┊ 95┊            filtered.push(message);
+┊   ┊ 96┊          } // else discard the message
+┊   ┊ 97┊
+┊   ┊ 98┊          return filtered;
+┊   ┊ 99┊        }, []);
+┊   ┊100┊
+┊   ┊101┊        // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
+┊   ┊102┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser);
+┊   ┊103┊
+┊   ┊104┊        // Check how many members are left
+┊   ┊105┊        if (listingMemberIds.length === 0) {
+┊   ┊106┊          // Delete the chat
+┊   ┊107┊          chats = chats.filter(chat => chat.id !== chatId);
+┊   ┊108┊        } else {
+┊   ┊109┊          // Update the chat
+┊   ┊110┊          chats = chats.map(chat => {
+┊   ┊111┊            if (chat.id === chatId) {
+┊   ┊112┊              chat = {...chat, listingMemberIds, messages};
+┊   ┊113┊            }
+┊   ┊114┊            return chat;
+┊   ┊115┊          });
+┊   ┊116┊        }
+┊   ┊117┊        return chatId;
+┊   ┊118┊      } else {
+┊   ┊119┊        // Group
+┊   ┊120┊        if (chat.ownerId !== currentUser) {
+┊   ┊121┊          throw new Error(`Group ${chatId} is not owned by the user.`);
+┊   ┊122┊        }
+┊   ┊123┊
+┊   ┊124┊        // Instead of chaining map and filter we can loop once using reduce
+┊   ┊125┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
+┊   ┊126┊          // Remove the current user from the message holders
+┊   ┊127┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊128┊
+┊   ┊129┊          if (message.holderIds.length !== 0) {
+┊   ┊130┊            filtered.push(message);
+┊   ┊131┊          } // else discard the message
+┊   ┊132┊
+┊   ┊133┊          return filtered;
+┊   ┊134┊        }, []);
+┊   ┊135┊
+┊   ┊136┊        // Remove the current user from who gets the group listed. The group will no longer appear in his list
+┊   ┊137┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser);
+┊   ┊138┊
+┊   ┊139┊        // Check how many members (including previous ones who can still access old messages) are left
+┊   ┊140┊        if (listingMemberIds.length === 0) {
+┊   ┊141┊          // Remove the group
+┊   ┊142┊          chats = chats.filter(chat => chat.id !== chatId);
+┊   ┊143┊        } else {
+┊   ┊144┊          // Update the group
+┊   ┊145┊
+┊   ┊146┊          // Remove the current user from the chat members. He is no longer a member of the group
+┊   ┊147┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser);
+┊   ┊148┊          // Remove the current user from the chat admins
+┊   ┊149┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser);
+┊   ┊150┊          // Set the owner id to be null. A null owner means the group is read-only
+┊   ┊151┊          let ownerId: number | null = null;
+┊   ┊152┊
+┊   ┊153┊          // Check if there is any admin left
+┊   ┊154┊          if (adminIds!.length) {
+┊   ┊155┊            // Pick an admin as the new owner. The group is no longer read-only
+┊   ┊156┊            ownerId = chat.adminIds![0];
+┊   ┊157┊          }
+┊   ┊158┊
+┊   ┊159┊          chats = chats.map(chat => {
+┊   ┊160┊            if (chat.id === chatId) {
+┊   ┊161┊              chat = {...chat, messages, listingMemberIds, actualGroupMemberIds, adminIds, ownerId};
+┊   ┊162┊            }
+┊   ┊163┊            return chat;
+┊   ┊164┊          });
+┊   ┊165┊        }
+┊   ┊166┊        return chatId;
+┊   ┊167┊      }
+┊   ┊168┊    },
+┊   ┊169┊    addMessage: (obj: any, {chatId, content}: any): Message => {
+┊   ┊170┊      if (content === null || content === '') {
+┊   ┊171┊        throw new Error(`Cannot add empty or null messages.`);
+┊   ┊172┊      }
+┊   ┊173┊
+┊   ┊174┊      let chat = chats.find(chat => chat.id === chatId);
+┊   ┊175┊
+┊   ┊176┊      if (!chat) {
+┊   ┊177┊        throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊178┊      }
+┊   ┊179┊
+┊   ┊180┊      let holderIds = chat.listingMemberIds;
+┊   ┊181┊
+┊   ┊182┊      if (!chat.name) {
+┊   ┊183┊        // Chat
+┊   ┊184┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser)) {
+┊   ┊185┊          throw new Error(`The chat ${chatId} must be listed for the current user before adding a message.`);
+┊   ┊186┊        }
+┊   ┊187┊
+┊   ┊188┊        const recipientId = chat.allTimeMemberIds.filter(userId => userId !== currentUser)[0];
+┊   ┊189┊
+┊   ┊190┊        if (!chat.listingMemberIds.find(listingId => listingId === recipientId)) {
+┊   ┊191┊          // Chat is not listed for the recipient. Add him to the listingMemberIds
+┊   ┊192┊          const listingMemberIds = chat.listingMemberIds.concat(recipientId);
+┊   ┊193┊
+┊   ┊194┊          chats = chats.map(chat => {
+┊   ┊195┊            if (chat.id === chatId) {
+┊   ┊196┊              chat = {...chat, listingMemberIds};
+┊   ┊197┊            }
+┊   ┊198┊            return chat;
+┊   ┊199┊          });
+┊   ┊200┊
+┊   ┊201┊          holderIds = listingMemberIds;
+┊   ┊202┊        }
+┊   ┊203┊      } else {
+┊   ┊204┊        // Group
+┊   ┊205┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser)) {
+┊   ┊206┊          throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
+┊   ┊207┊        }
+┊   ┊208┊
+┊   ┊209┊        holderIds = chat.actualGroupMemberIds!;
+┊   ┊210┊      }
+┊   ┊211┊
+┊   ┊212┊      const id = (chat.messages.length && chat.messages[chat.messages.length - 1].id + 1) || 1;
+┊   ┊213┊
+┊   ┊214┊      let recipients: Recipient[] = [];
+┊   ┊215┊
+┊   ┊216┊      holderIds.forEach(holderId => {
+┊   ┊217┊        if (holderId !== currentUser) {
+┊   ┊218┊          recipients.push({
+┊   ┊219┊            userId: holderId,
+┊   ┊220┊            messageId: id,
+┊   ┊221┊            chatId: chatId,
+┊   ┊222┊            receivedAt: null,
+┊   ┊223┊            readAt: null,
+┊   ┊224┊          });
+┊   ┊225┊        }
+┊   ┊226┊      });
+┊   ┊227┊
+┊   ┊228┊      const message: Message = {
+┊   ┊229┊        id,
+┊   ┊230┊        chatId,
+┊   ┊231┊        senderId: currentUser,
+┊   ┊232┊        content,
+┊   ┊233┊        createdAt: moment().unix(),
+┊   ┊234┊        type: MessageType.TEXT,
+┊   ┊235┊        recipients,
+┊   ┊236┊        holderIds,
+┊   ┊237┊      };
+┊   ┊238┊
+┊   ┊239┊      chats = chats.map(chat => {
+┊   ┊240┊        if (chat.id === chatId) {
+┊   ┊241┊          chat = {...chat, messages: chat.messages.concat(message)}
+┊   ┊242┊        }
+┊   ┊243┊        return chat;
+┊   ┊244┊      });
+┊   ┊245┊
+┊   ┊246┊      return message;
+┊   ┊247┊    },
+┊   ┊248┊    removeMessages: (obj: any, {chatId, messageIds, all}: any): number[] => {
+┊   ┊249┊      const chat = chats.find(chat => chat.id === chatId);
+┊   ┊250┊
+┊   ┊251┊      if (!chat) {
+┊   ┊252┊        throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊253┊      }
+┊   ┊254┊
+┊   ┊255┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser)) {
+┊   ┊256┊        throw new Error(`The chat/group ${chatId} is not listed for the current user, so there is nothing to delete.`);
+┊   ┊257┊      }
+┊   ┊258┊
+┊   ┊259┊      if (all && messageIds) {
+┊   ┊260┊        throw new Error(`Cannot specify both 'all' and 'messageIds'.`);
+┊   ┊261┊      }
+┊   ┊262┊
+┊   ┊263┊      let deletedIds: number[] = [];
+┊   ┊264┊      chats = chats.map(chat => {
+┊   ┊265┊        if (chat.id === chatId) {
+┊   ┊266┊          // Instead of chaining map and filter we can loop once using reduce
+┊   ┊267┊          const messages = chat.messages.reduce<Message[]>((filtered, message) => {
+┊   ┊268┊            if (all || messageIds!.includes(message.id)) {
+┊   ┊269┊              deletedIds.push(message.id);
+┊   ┊270┊              // Remove the current user from the message holders
+┊   ┊271┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊272┊            }
+┊   ┊273┊
+┊   ┊274┊            if (message.holderIds.length !== 0) {
+┊   ┊275┊              filtered.push(message);
+┊   ┊276┊            } // else discard the message
+┊   ┊277┊
+┊   ┊278┊            return filtered;
+┊   ┊279┊          }, []);
+┊   ┊280┊          chat = {...chat, messages};
+┊   ┊281┊        }
+┊   ┊282┊        return chat;
+┊   ┊283┊      });
+┊   ┊284┊      return deletedIds;
+┊   ┊285┊    },
+┊   ┊286┊  },
 ┊ 16┊287┊  Chat: {
 ┊ 17┊288┊    name: (chat: Chat): string => chat.name ? chat.name : users
 ┊ 18┊289┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser))!.name,
```

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -67,4 +67,20 @@
 ┊67┊67┊    picture: String
 ┊68┊68┊    phone: String
 ┊69┊69┊  }
+┊  ┊70┊
+┊  ┊71┊  type Mutation {
+┊  ┊72┊    addChat(recipientId: ID!): Chat
+┊  ┊73┊    addGroup(recipientIds: [ID!]!, groupName: String!): Chat
+┊  ┊74┊    removeChat(chatId: ID!): ID
+┊  ┊75┊    addMessage(chatId: ID!, content: String!): Message
+┊  ┊76┊    removeMessages(chatId: ID!, messageIds: [ID], all: Boolean): [ID]
+┊  ┊77┊    addMembers(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊78┊    removeMembers(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊79┊    addAdmins(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊80┊    removeAdmins(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊81┊    setGroupName(groupId: ID!): String
+┊  ┊82┊    setGroupPicture(groupId: ID!): String
+┊  ┊83┊    markAsReceived(chatId: ID!): Boolean
+┊  ┊84┊    markAsRead(chatId: ID!): Boolean
+┊  ┊85┊  }
 ┊70┊86┊`;
```

[}]: #

Let me briefly explain what's going on here.
For each chat/group we store the `allTimeMemberIds`, `listingMemberIds` and `actualGroupMemberIds` properties in our NoSQL-like fake db.
What's the difference between `allTimeMemberIds` and `listingMemberIds`? When a chat gets created only the user who created it will be able too see it, the chat will be displayed to the other user only once the first messaged gets sent. `allTimeMemberIds` is an array which always contain both the users, while `listingMemberIds` contains only the users which get the chat listed (initially the creator, later both users). `actualGroupMemberIds` is only used for groups.
Groups, instead, get listed by all members immediately since the creation. So initially both `allTimeMemberIds`, `listingMemberIds` and `actualGroupMemberIds` are similar. Later users can leave the group or get deleted (so they will be removed from `actualGroupMemberIds`) but they will still be able to list the group in read-only mode, thus remaining in the `listingMemberIds`. Once they remove the group they will also be remove from the `listingMemberIds` array.
That's why we have to check for several different conditions before adding/deleting messages: it could be necessary to add the other peer to the `listingMemberIds` (for example if we are writing the first message of a chat) or it could be necessary to physically remove the messages instead of simply removing the current user from the `holderIds`. `holderIds` is a field in each message which states which user will currently display that specific message. In fact each user can delete a specific message without affecting what the others will see. Once there will be no more users in the `holderIds` array it will be safe to delete the message.
Each message has also a `recipients` array containing the receiving date and the viewing date of that particular message for all the other users. That's necessary to implement the single, double and blue ticks used by the real Whatsapp.

It may seem a bit overwhelming at first, but you should keep in mind that the real Whatsapp has tons of features and also takes advantage of a local database to store messages, so it's easier for them to implement features like per-user messages: their source of truth is not the server because once downloaded the messages are kept in the client itself, so deleting messages doesn't affect anyone else. On the contrary our source of truth is the server, so our approach is more similar to Telegram instead. This is a better approach in my opinion because it allows us to show the messages for the same user on multiple clients, instead of having to rely on questionable approaches like Whatsapp Web.
Also we already implemented our mutations to take care of future use cases (like reading notifications) which we still didn't implement.

I said we were going to take greater advantage of `graphql-code-generator` once we started writing our first mutation and I'm going to show you why. Let's run the generator first:

    $ npm run generator

Then let's use the generated types:

[{]: <helper> (diffStep "3.3" module="server")

#### Step 3.3: Use generated types

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,6 +1,9 @@
 ┊1┊1┊import { Chat, db, Message, MessageType, Recipient, User } from "../db";
 ┊2┊2┊import { IResolvers } from "graphql-tools/dist/Interfaces";
-┊3┊ ┊import { ChatQueryArgs } from "../types";
+┊ ┊3┊import {
+┊ ┊4┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs,
+┊ ┊5┊  RemoveChatMutationArgs, RemoveMessagesMutationArgs
+┊ ┊6┊} from "../types";
 ┊4┊7┊import * as moment from "moment";
 ┊5┊8┊
 ┊6┊9┊let users = db.users;
```
```diff
@@ -15,12 +18,12 @@
 ┊15┊18┊    chat: (obj: any, {chatId}: ChatQueryArgs): Chat | null => chats.find(chat => chat.id === Number(chatId)) || null,
 ┊16┊19┊  },
 ┊17┊20┊  Mutation: {
-┊18┊  ┊    addChat: (obj: any, {recipientId}: any): Chat => {
-┊19┊  ┊      if (!users.find(user => user.id === recipientId)) {
+┊  ┊21┊    addChat: (obj: any, {recipientId}: AddChatMutationArgs): Chat => {
+┊  ┊22┊      if (!users.find(user => user.id === Number(recipientId))) {
 ┊20┊23┊        throw new Error(`Recipient ${recipientId} doesn't exist.`);
 ┊21┊24┊      }
 ┊22┊25┊
-┊23┊  ┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser) && chat.allTimeMemberIds.includes(recipientId));
+┊  ┊26┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser) && chat.allTimeMemberIds.includes(Number(recipientId)));
 ┊24┊27┊      if (chat) {
 ┊25┊28┊        // Chat already exists. Both users are already in the allTimeMemberIds array
 ┊26┊29┊        const chatId = chat.id;
```
```diff
@@ -41,7 +44,7 @@
 ┊41┊44┊          picture: null,
 ┊42┊45┊          adminIds: null,
 ┊43┊46┊          ownerId: null,
-┊44┊  ┊          allTimeMemberIds: [currentUser, recipientId],
+┊  ┊47┊          allTimeMemberIds: [currentUser, Number(recipientId)],
 ┊45┊48┊          // Chat will not be listed to the other user until the first message gets written
 ┊46┊49┊          listingMemberIds: [currentUser],
 ┊47┊50┊          actualGroupMemberIds: null,
```
```diff
@@ -51,9 +54,9 @@
 ┊51┊54┊        return chat;
 ┊52┊55┊      }
 ┊53┊56┊    },
-┊54┊  ┊    addGroup: (obj: any, {recipientIds, groupName}: any): Chat => {
-┊55┊  ┊      recipientIds.forEach((recipientId: any) => {
-┊56┊  ┊        if (!users.find(user => user.id === recipientId)) {
+┊  ┊57┊    addGroup: (obj: any, {recipientIds, groupName}: AddGroupMutationArgs): Chat => {
+┊  ┊58┊      recipientIds.forEach(recipientId => {
+┊  ┊59┊        if (!users.find(user => user.id === Number(recipientId))) {
 ┊57┊60┊          throw new Error(`Recipient ${recipientId} doesn't exist.`);
 ┊58┊61┊        }
 ┊59┊62┊      });
```
```diff
@@ -65,16 +68,16 @@
 ┊65┊68┊        picture: null,
 ┊66┊69┊        adminIds: [currentUser],
 ┊67┊70┊        ownerId: currentUser,
-┊68┊  ┊        allTimeMemberIds: [currentUser, ...recipientIds],
-┊69┊  ┊        listingMemberIds: [currentUser, ...recipientIds],
-┊70┊  ┊        actualGroupMemberIds: [currentUser, ...recipientIds],
+┊  ┊71┊        allTimeMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
+┊  ┊72┊        listingMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
+┊  ┊73┊        actualGroupMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
 ┊71┊74┊        messages: [],
 ┊72┊75┊      };
 ┊73┊76┊      chats.push(chat);
 ┊74┊77┊      return chat;
 ┊75┊78┊    },
-┊76┊  ┊    removeChat: (obj: any, {chatId}: any): number => {
-┊77┊  ┊      const chat = chats.find(chat => chat.id === chatId);
+┊  ┊79┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs): number => {
+┊  ┊80┊      const chat = chats.find(chat => chat.id === Number(chatId));
 ┊78┊81┊
 ┊79┊82┊      if (!chat) {
 ┊80┊83┊        throw new Error(`The chat ${chatId} doesn't exist.`);
```
```diff
@@ -104,17 +107,17 @@
 ┊104┊107┊        // Check how many members are left
 ┊105┊108┊        if (listingMemberIds.length === 0) {
 ┊106┊109┊          // Delete the chat
-┊107┊   ┊          chats = chats.filter(chat => chat.id !== chatId);
+┊   ┊110┊          chats = chats.filter(chat => chat.id !== Number(chatId));
 ┊108┊111┊        } else {
 ┊109┊112┊          // Update the chat
 ┊110┊113┊          chats = chats.map(chat => {
-┊111┊   ┊            if (chat.id === chatId) {
+┊   ┊114┊            if (chat.id === Number(chatId)) {
 ┊112┊115┊              chat = {...chat, listingMemberIds, messages};
 ┊113┊116┊            }
 ┊114┊117┊            return chat;
 ┊115┊118┊          });
 ┊116┊119┊        }
-┊117┊   ┊        return chatId;
+┊   ┊120┊        return Number(chatId);
 ┊118┊121┊      } else {
 ┊119┊122┊        // Group
 ┊120┊123┊        if (chat.ownerId !== currentUser) {
```
```diff
@@ -139,7 +142,7 @@
 ┊139┊142┊        // Check how many members (including previous ones who can still access old messages) are left
 ┊140┊143┊        if (listingMemberIds.length === 0) {
 ┊141┊144┊          // Remove the group
-┊142┊   ┊          chats = chats.filter(chat => chat.id !== chatId);
+┊   ┊145┊          chats = chats.filter(chat => chat.id !== Number(chatId));
 ┊143┊146┊        } else {
 ┊144┊147┊          // Update the group
 ┊145┊148┊
```
```diff
@@ -157,21 +160,21 @@
 ┊157┊160┊          }
 ┊158┊161┊
 ┊159┊162┊          chats = chats.map(chat => {
-┊160┊   ┊            if (chat.id === chatId) {
+┊   ┊163┊            if (chat.id === Number(chatId)) {
 ┊161┊164┊              chat = {...chat, messages, listingMemberIds, actualGroupMemberIds, adminIds, ownerId};
 ┊162┊165┊            }
 ┊163┊166┊            return chat;
 ┊164┊167┊          });
 ┊165┊168┊        }
-┊166┊   ┊        return chatId;
+┊   ┊169┊        return Number(chatId);
 ┊167┊170┊      }
 ┊168┊171┊    },
-┊169┊   ┊    addMessage: (obj: any, {chatId, content}: any): Message => {
+┊   ┊172┊    addMessage: (obj: any, {chatId, content}: AddMessageMutationArgs): Message => {
 ┊170┊173┊      if (content === null || content === '') {
 ┊171┊174┊        throw new Error(`Cannot add empty or null messages.`);
 ┊172┊175┊      }
 ┊173┊176┊
-┊174┊   ┊      let chat = chats.find(chat => chat.id === chatId);
+┊   ┊177┊      let chat = chats.find(chat => chat.id === Number(chatId));
 ┊175┊178┊
 ┊176┊179┊      if (!chat) {
 ┊177┊180┊        throw new Error(`Cannot find chat ${chatId}.`);
```
```diff
@@ -192,7 +195,7 @@
 ┊192┊195┊          const listingMemberIds = chat.listingMemberIds.concat(recipientId);
 ┊193┊196┊
 ┊194┊197┊          chats = chats.map(chat => {
-┊195┊   ┊            if (chat.id === chatId) {
+┊   ┊198┊            if (chat.id === Number(chatId)) {
 ┊196┊199┊              chat = {...chat, listingMemberIds};
 ┊197┊200┊            }
 ┊198┊201┊            return chat;
```
```diff
@@ -218,7 +221,7 @@
 ┊218┊221┊          recipients.push({
 ┊219┊222┊            userId: holderId,
 ┊220┊223┊            messageId: id,
-┊221┊   ┊            chatId: chatId,
+┊   ┊224┊            chatId: Number(chatId),
 ┊222┊225┊            receivedAt: null,
 ┊223┊226┊            readAt: null,
 ┊224┊227┊          });
```
```diff
@@ -227,7 +230,7 @@
 ┊227┊230┊
 ┊228┊231┊      const message: Message = {
 ┊229┊232┊        id,
-┊230┊   ┊        chatId,
+┊   ┊233┊        chatId: Number(chatId),
 ┊231┊234┊        senderId: currentUser,
 ┊232┊235┊        content,
 ┊233┊236┊        createdAt: moment().unix(),
```
```diff
@@ -237,7 +240,7 @@
 ┊237┊240┊      };
 ┊238┊241┊
 ┊239┊242┊      chats = chats.map(chat => {
-┊240┊   ┊        if (chat.id === chatId) {
+┊   ┊243┊        if (chat.id === Number(chatId)) {
 ┊241┊244┊          chat = {...chat, messages: chat.messages.concat(message)}
 ┊242┊245┊        }
 ┊243┊246┊        return chat;
```
```diff
@@ -245,8 +248,8 @@
 ┊245┊248┊
 ┊246┊249┊      return message;
 ┊247┊250┊    },
-┊248┊   ┊    removeMessages: (obj: any, {chatId, messageIds, all}: any): number[] => {
-┊249┊   ┊      const chat = chats.find(chat => chat.id === chatId);
+┊   ┊251┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs): number[] => {
+┊   ┊252┊      const chat = chats.find(chat => chat.id === Number(chatId));
 ┊250┊253┊
 ┊251┊254┊      if (!chat) {
 ┊252┊255┊        throw new Error(`Cannot find chat ${chatId}.`);
```
```diff
@@ -262,10 +265,10 @@
 ┊262┊265┊
 ┊263┊266┊      let deletedIds: number[] = [];
 ┊264┊267┊      chats = chats.map(chat => {
-┊265┊   ┊        if (chat.id === chatId) {
+┊   ┊268┊        if (chat.id === Number(chatId)) {
 ┊266┊269┊          // Instead of chaining map and filter we can loop once using reduce
 ┊267┊270┊          const messages = chat.messages.reduce<Message[]>((filtered, message) => {
-┊268┊   ┊            if (all || messageIds!.includes(message.id)) {
+┊   ┊271┊            if (all || messageIds!.includes(String(message.id))) {
 ┊269┊272┊              deletedIds.push(message.id);
 ┊270┊273┊              // Remove the current user from the message holders
 ┊271┊274┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
```

[}]: #

## Client

For the client I'll only show you how to make use of the addMessage mutation in this chapters. The other mutations will require much more boilerplate so I left them for their own chapter.

Let's start by wiring the addMessage mutation. We're going to write the GraphQL query and then use the generator to generate the types:

[{]: <helper> (diffStep "5.1" files="^\(?!src/types.d.ts$\).*" module="client")

#### Step 5.1: Create addMessage mutation and generate types

##### Added src&#x2F;graphql&#x2F;addMessage.mutation.ts
```diff
@@ -0,0 +1,13 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const addMessageMutation = gql`
+┊  ┊ 6┊  mutation AddMessage($chatId: ID!, $content: String!) {
+┊  ┊ 7┊    addMessage(chatId: $chatId, content: $content) {
+┊  ┊ 8┊      ...Message
+┊  ┊ 9┊    }
+┊  ┊10┊  }
+┊  ┊11┊
+┊  ┊12┊  ${fragments['message']}
+┊  ┊13┊`;
```

[}]: #

Run the generator:

    $ npm run generator

Now let's use the just-created query:

[{]: <helper> (diffStep "5.2" module="client")

#### Step 5.2: Modify chat component and service

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -13,7 +13,7 @@
 ┊13┊13┊    </app-toolbar>
 ┊14┊14┊    <div class="container">
 ┊15┊15┊      <app-messages-list [items]="messages" [isGroup]="isGroup"></app-messages-list>
-┊16┊  ┊      <app-new-message></app-new-message>
+┊  ┊16┊      <app-new-message (newMessage)="addMessage($event)"></app-new-message>
 ┊17┊17┊    </div>
 ┊18┊18┊  `,
 ┊19┊19┊  styleUrls: ['./chat.component.scss']
```
```diff
@@ -43,4 +43,8 @@
 ┊43┊43┊  goToChats() {
 ┊44┊44┊    this.router.navigate(['/chats']);
 ┊45┊45┊  }
+┊  ┊46┊
+┊  ┊47┊  addMessage(content: string) {
+┊  ┊48┊    this.chatsService.addMessage(this.chatId, content).subscribe();
+┊  ┊49┊  }
 ┊46┊50┊}
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -3,8 +3,9 @@
 ┊ 3┊ 3┊import {Apollo} from 'apollo-angular';
 ┊ 4┊ 4┊import {Injectable} from '@angular/core';
 ┊ 5┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊ 6┊  ┊import {GetChat, GetChats} from '../../types';
+┊  ┊ 6┊import {AddMessage, GetChat, GetChats} from '../../types';
 ┊ 7┊ 7┊import {getChatQuery} from '../../graphql/getChat.query';
+┊  ┊ 8┊import {addMessageMutation} from '../../graphql/addMessage.mutation';
 ┊ 8┊ 9┊
 ┊ 9┊10┊@Injectable()
 ┊10┊11┊export class ChatsService {
```
```diff
@@ -40,4 +41,14 @@
 ┊40┊41┊
 ┊41┊42┊    return {query, chat$};
 ┊42┊43┊  }
+┊  ┊44┊
+┊  ┊45┊  addMessage(chatId: string, content: string) {
+┊  ┊46┊    return this.apollo.mutate({
+┊  ┊47┊      mutation: addMessageMutation,
+┊  ┊48┊      variables: <AddMessage.Variables>{
+┊  ┊49┊        chatId,
+┊  ┊50┊        content,
+┊  ┊51┊      },
+┊  ┊52┊    });
+┊  ┊53┊  }
 ┊43┊54┊}
```

[}]: #

It's that simple! You would be tempted to say that it doesn't work, but you should try to refresh the page first ;)

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step8.md) | [Next Step >](step10.md) |
|:--------------------------------|--------------------------------:|

[}]: #
