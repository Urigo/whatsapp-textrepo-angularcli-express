# Step 5: Mutations

[//]: # (head-end)


In addition to fetching data using queries, Apollo also helps you handle GraphQL mutations. In GraphQL, mutations are identical to queries in syntax, the only difference being that you use the keyword mutation instead of query to indicate that the root fields on this query are going to be performing writes to the backend.
GraphQL mutations represent two things in one query string:

1. The mutation field name with arguments, which represents the actual operation to be done on the server.
2. The fields you want back from the result of the mutation to update the client.

When we use mutations in Apollo, the result is typically integrated into the cache automatically based on the id of the result, which in turn updates the UI automatically, so we often don't need to explicitly handle the results. In order for the client to correctly do this, we need to ensure we select the necessary fields in the result. One good strategy can be to simply ask for any fields that might have been affected by the mutation. Alternatively, you can use fragments to share the fields between a query and a mutation that updates that query.

## Server

Finally we're going to create our mutations in the server:

[{]: <helper> (diffStep "3.1" module="server")

#### [Step 3.1: Add mutations](https://github.com/Urigo/WhatsApp-Clone-Server/commit/9953480)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,6 +1,7 @@
-┊1┊ ┊import { db } from "../db";
+┊ ┊1┊import { IResolvers } from "../types";
 ┊2┊2┊import { GraphQLDateTime } from 'graphql-iso-date';
-┊3┊ ┊import { IResolvers } from '../types';
+┊ ┊3┊import { ChatDb, db, MessageDb, MessageType, RecipientDb } from "../db";
+┊ ┊4┊import moment from "moment";
 ┊4┊5┊
 ┊5┊6┊let users = db.users;
 ┊6┊7┊let chats = db.chats;
```
```diff
@@ -14,6 +15,298 @@
 ┊ 14┊ 15┊    chats: () => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
 ┊ 15┊ 16┊    chat: (obj, {chatId}) => chats.find(chat => chat.id === Number(chatId)),
 ┊ 16┊ 17┊  },
+┊   ┊ 18┊  Mutation: {
+┊   ┊ 19┊    updateUser: (obj, {name, picture}) => {
+┊   ┊ 20┊      currentUser.name = name || currentUser.name;
+┊   ┊ 21┊      currentUser.picture = picture || currentUser.picture;
+┊   ┊ 22┊
+┊   ┊ 23┊      return currentUser;
+┊   ┊ 24┊    },
+┊   ┊ 25┊    addChat: (obj, {userId}) => {
+┊   ┊ 26┊      if (!users.find(user => user.id === Number(userId))) {
+┊   ┊ 27┊        throw new Error(`User ${userId} doesn't exist.`);
+┊   ┊ 28┊      }
+┊   ┊ 29┊
+┊   ┊ 30┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser.id) && chat.allTimeMemberIds.includes(Number(userId)));
+┊   ┊ 31┊      if (chat) {
+┊   ┊ 32┊        // Chat already exists. Both users are already in the allTimeMemberIds array
+┊   ┊ 33┊        const chatId = chat.id;
+┊   ┊ 34┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
+┊   ┊ 35┊          // The chat isn't listed for the current user. Add him to the memberIds
+┊   ┊ 36┊          chat.listingMemberIds.push(currentUser.id);
+┊   ┊ 37┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser.id);
+┊   ┊ 38┊          return chat;
+┊   ┊ 39┊        } else {
+┊   ┊ 40┊          throw new Error(`Chat already exists.`);
+┊   ┊ 41┊        }
+┊   ┊ 42┊      } else {
+┊   ┊ 43┊        // Create the chat
+┊   ┊ 44┊        const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
+┊   ┊ 45┊        const chat: ChatDb = {
+┊   ┊ 46┊          id,
+┊   ┊ 47┊          createdAt: moment().toDate(),
+┊   ┊ 48┊          name: null,
+┊   ┊ 49┊          picture: null,
+┊   ┊ 50┊          adminIds: null,
+┊   ┊ 51┊          ownerId: null,
+┊   ┊ 52┊          allTimeMemberIds: [currentUser.id, Number(userId)],
+┊   ┊ 53┊          // Chat will not be listed to the other user until the first message gets written
+┊   ┊ 54┊          listingMemberIds: [currentUser.id],
+┊   ┊ 55┊          actualGroupMemberIds: null,
+┊   ┊ 56┊          messages: [],
+┊   ┊ 57┊        };
+┊   ┊ 58┊        chats.push(chat);
+┊   ┊ 59┊        return chat;
+┊   ┊ 60┊      }
+┊   ┊ 61┊    },
+┊   ┊ 62┊    addGroup: (obj, {userIds, groupName, groupPicture}) => {
+┊   ┊ 63┊      userIds.forEach(userId => {
+┊   ┊ 64┊        if (!users.find(user => user.id === Number(userId))) {
+┊   ┊ 65┊          throw new Error(`User ${userId} doesn't exist.`);
+┊   ┊ 66┊        }
+┊   ┊ 67┊      });
+┊   ┊ 68┊
+┊   ┊ 69┊      const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
+┊   ┊ 70┊      const chat: ChatDb = {
+┊   ┊ 71┊        id,
+┊   ┊ 72┊        createdAt: moment().toDate(),
+┊   ┊ 73┊        name: groupName,
+┊   ┊ 74┊        picture: groupPicture || null,
+┊   ┊ 75┊        adminIds: [currentUser.id],
+┊   ┊ 76┊        ownerId: currentUser.id,
+┊   ┊ 77┊        allTimeMemberIds: [currentUser.id, ...userIds.map(id => Number(id))],
+┊   ┊ 78┊        listingMemberIds: [currentUser.id, ...userIds.map(id => Number(id))],
+┊   ┊ 79┊        actualGroupMemberIds: [currentUser.id, ...userIds.map(id => Number(id))],
+┊   ┊ 80┊        messages: [],
+┊   ┊ 81┊      };
+┊   ┊ 82┊      chats.push(chat);
+┊   ┊ 83┊      return chat;
+┊   ┊ 84┊    },
+┊   ┊ 85┊    updateGroup: (obj, {chatId, groupName, groupPicture}) => {
+┊   ┊ 86┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊ 87┊
+┊   ┊ 88┊      if (!chat) {
+┊   ┊ 89┊        throw new Error(`The chat ${chatId} doesn't exist.`);
+┊   ┊ 90┊      }
+┊   ┊ 91┊
+┊   ┊ 92┊      if (!chat.name) {
+┊   ┊ 93┊        throw new Error(`The chat ${chatId} is not a group.`);
+┊   ┊ 94┊      }
+┊   ┊ 95┊
+┊   ┊ 96┊      chat.name = groupName || chat.name;
+┊   ┊ 97┊      chat.picture = groupPicture || chat.picture;
+┊   ┊ 98┊
+┊   ┊ 99┊      return chat;
+┊   ┊100┊    },
+┊   ┊101┊    removeChat: (obj, {chatId}) => {
+┊   ┊102┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊103┊
+┊   ┊104┊      if (!chat) {
+┊   ┊105┊        throw new Error(`The chat ${chatId} doesn't exist.`);
+┊   ┊106┊      }
+┊   ┊107┊
+┊   ┊108┊      if (!chat.name) {
+┊   ┊109┊        // Chat
+┊   ┊110┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
+┊   ┊111┊          throw new Error(`The user is not a member of the chat ${chatId}.`);
+┊   ┊112┊        }
+┊   ┊113┊
+┊   ┊114┊        // Instead of chaining map and filter we can loop once using reduce
+┊   ┊115┊        const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
+┊   ┊116┊          // Remove the current user from the message holders
+┊   ┊117┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊118┊
+┊   ┊119┊          if (message.holderIds.length !== 0) {
+┊   ┊120┊            filtered.push(message);
+┊   ┊121┊          } // else discard the message
+┊   ┊122┊
+┊   ┊123┊          return filtered;
+┊   ┊124┊        }, []);
+┊   ┊125┊
+┊   ┊126┊        // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
+┊   ┊127┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊128┊
+┊   ┊129┊        // Check how many members are left
+┊   ┊130┊        if (listingMemberIds.length === 0) {
+┊   ┊131┊          // Delete the chat
+┊   ┊132┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊133┊        } else {
+┊   ┊134┊          // Update the chat
+┊   ┊135┊          chats = chats.map(chat => {
+┊   ┊136┊            if (chat.id === Number(chatId)) {
+┊   ┊137┊              chat = {...chat, listingMemberIds, messages};
+┊   ┊138┊            }
+┊   ┊139┊            return chat;
+┊   ┊140┊          });
+┊   ┊141┊        }
+┊   ┊142┊        return chatId;
+┊   ┊143┊      } else {
+┊   ┊144┊        // Group
+┊   ┊145┊        if (chat.ownerId !== currentUser.id) {
+┊   ┊146┊          throw new Error(`Group ${chatId} is not owned by the user.`);
+┊   ┊147┊        }
+┊   ┊148┊
+┊   ┊149┊        // Instead of chaining map and filter we can loop once using reduce
+┊   ┊150┊        const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
+┊   ┊151┊          // Remove the current user from the message holders
+┊   ┊152┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊153┊
+┊   ┊154┊          if (message.holderIds.length !== 0) {
+┊   ┊155┊            filtered.push(message);
+┊   ┊156┊          } // else discard the message
+┊   ┊157┊
+┊   ┊158┊          return filtered;
+┊   ┊159┊        }, []);
+┊   ┊160┊
+┊   ┊161┊        // Remove the current user from who gets the group listed. The group will no longer appear in his list
+┊   ┊162┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊163┊
+┊   ┊164┊        // Check how many members (including previous ones who can still access old messages) are left
+┊   ┊165┊        if (listingMemberIds.length === 0) {
+┊   ┊166┊          // Remove the group
+┊   ┊167┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊168┊        } else {
+┊   ┊169┊          // Update the group
+┊   ┊170┊
+┊   ┊171┊          // Remove the current user from the chat members. He is no longer a member of the group
+┊   ┊172┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser.id);
+┊   ┊173┊          // Remove the current user from the chat admins
+┊   ┊174┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser.id);
+┊   ┊175┊          // Set the owner id to be null. A null owner means the group is read-only
+┊   ┊176┊          let ownerId: number | null = null;
+┊   ┊177┊
+┊   ┊178┊          // Check if there is any admin left
+┊   ┊179┊          if (adminIds!.length) {
+┊   ┊180┊            // Pick an admin as the new owner. The group is no longer read-only
+┊   ┊181┊            ownerId = chat.adminIds![0];
+┊   ┊182┊          }
+┊   ┊183┊
+┊   ┊184┊          chats = chats.map(chat => {
+┊   ┊185┊            if (chat.id === Number(chatId)) {
+┊   ┊186┊              chat = {...chat, messages, listingMemberIds, actualGroupMemberIds, adminIds, ownerId};
+┊   ┊187┊            }
+┊   ┊188┊            return chat;
+┊   ┊189┊          });
+┊   ┊190┊        }
+┊   ┊191┊        return chatId;
+┊   ┊192┊      }
+┊   ┊193┊    },
+┊   ┊194┊    addMessage: (obj, {chatId, content}) => {
+┊   ┊195┊      if (content === null || content === '') {
+┊   ┊196┊        throw new Error(`Cannot add empty or null messages.`);
+┊   ┊197┊      }
+┊   ┊198┊
+┊   ┊199┊      let chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊200┊
+┊   ┊201┊      if (!chat) {
+┊   ┊202┊        throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊203┊      }
+┊   ┊204┊
+┊   ┊205┊      let holderIds = chat.listingMemberIds;
+┊   ┊206┊
+┊   ┊207┊      if (!chat.name) {
+┊   ┊208┊        // Chat
+┊   ┊209┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊210┊          throw new Error(`The chat ${chatId} must be listed for the current user before adding a message.`);
+┊   ┊211┊        }
+┊   ┊212┊
+┊   ┊213┊        // Receiver's user
+┊   ┊214┊        const receiverId = chat.allTimeMemberIds.find(userId => userId !== currentUser.id);
+┊   ┊215┊
+┊   ┊216┊        if (!receiverId) {
+┊   ┊217┊          throw new Error(`Cannot find receiver's user.`);
+┊   ┊218┊        }
+┊   ┊219┊
+┊   ┊220┊        if (!chat.listingMemberIds.find(listingId => listingId === receiverId)) {
+┊   ┊221┊          // Chat is not listed for the receiver user. Add him to the listingMemberIds
+┊   ┊222┊          chat.listingMemberIds = chat.listingMemberIds.concat(receiverId);
+┊   ┊223┊
+┊   ┊224┊          holderIds = chat.listingMemberIds;
+┊   ┊225┊        }
+┊   ┊226┊      } else {
+┊   ┊227┊        // Group
+┊   ┊228┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser.id)) {
+┊   ┊229┊          throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
+┊   ┊230┊        }
+┊   ┊231┊
+┊   ┊232┊        holderIds = chat.actualGroupMemberIds!;
+┊   ┊233┊      }
+┊   ┊234┊
+┊   ┊235┊      const id = (chat.messages.length && chat.messages[chat.messages.length - 1].id + 1) || 1;
+┊   ┊236┊
+┊   ┊237┊      let recipients: RecipientDb[] = [];
+┊   ┊238┊
+┊   ┊239┊      holderIds.forEach(holderId => {
+┊   ┊240┊        if (holderId !== currentUser.id) {
+┊   ┊241┊          recipients.push({
+┊   ┊242┊            userId: holderId,
+┊   ┊243┊            messageId: id,
+┊   ┊244┊            chatId: Number(chatId),
+┊   ┊245┊            receivedAt: null,
+┊   ┊246┊            readAt: null,
+┊   ┊247┊          });
+┊   ┊248┊        }
+┊   ┊249┊      });
+┊   ┊250┊
+┊   ┊251┊      const message: MessageDb = {
+┊   ┊252┊        id,
+┊   ┊253┊        chatId: Number(chatId),
+┊   ┊254┊        senderId: currentUser.id,
+┊   ┊255┊        content,
+┊   ┊256┊        createdAt: moment().toDate(),
+┊   ┊257┊        type: MessageType.TEXT,
+┊   ┊258┊        recipients,
+┊   ┊259┊        holderIds,
+┊   ┊260┊      };
+┊   ┊261┊
+┊   ┊262┊      chats = chats.map(chat => {
+┊   ┊263┊        if (chat.id === Number(chatId)) {
+┊   ┊264┊          chat = {...chat, messages: chat.messages.concat(message)}
+┊   ┊265┊        }
+┊   ┊266┊        return chat;
+┊   ┊267┊      });
+┊   ┊268┊
+┊   ┊269┊      return message;
+┊   ┊270┊    },
+┊   ┊271┊    removeMessages: (obj, {chatId, messageIds, all}) => {
+┊   ┊272┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊273┊
+┊   ┊274┊      if (!chat) {
+┊   ┊275┊        throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊276┊      }
+┊   ┊277┊
+┊   ┊278┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊279┊        throw new Error(`The chat/group ${chatId} is not listed for the current user, so there is nothing to delete.`);
+┊   ┊280┊      }
+┊   ┊281┊
+┊   ┊282┊      if (all && messageIds) {
+┊   ┊283┊        throw new Error(`Cannot specify both 'all' and 'messageIds'.`);
+┊   ┊284┊      }
+┊   ┊285┊
+┊   ┊286┊      let deletedIds: string[] = [];
+┊   ┊287┊      chats = chats.map(chat => {
+┊   ┊288┊        if (chat.id === Number(chatId)) {
+┊   ┊289┊          // Instead of chaining map and filter we can loop once using reduce
+┊   ┊290┊          const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
+┊   ┊291┊            if (all || messageIds!.map(Number).includes(message.id)) {
+┊   ┊292┊              deletedIds.push(String(message.id));
+┊   ┊293┊              // Remove the current user from the message holders
+┊   ┊294┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊295┊            }
+┊   ┊296┊
+┊   ┊297┊            if (message.holderIds.length !== 0) {
+┊   ┊298┊              filtered.push(message);
+┊   ┊299┊            } // else discard the message
+┊   ┊300┊
+┊   ┊301┊            return filtered;
+┊   ┊302┊          }, []);
+┊   ┊303┊          chat = {...chat, messages};
+┊   ┊304┊        }
+┊   ┊305┊        return chat;
+┊   ┊306┊      });
+┊   ┊307┊      return deletedIds;
+┊   ┊308┊    },
+┊   ┊309┊  },
 ┊ 17┊310┊  Chat: {
 ┊ 18┊311┊    name: (chat) => chat.name ? chat.name : users
 ┊ 19┊312┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
```

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -73,4 +73,22 @@
 ┊73┊73┊    picture: String
 ┊74┊74┊    phone: String
 ┊75┊75┊  }
+┊  ┊76┊
+┊  ┊77┊  type Mutation {
+┊  ┊78┊    updateUser(name: String, picture: String): User!
+┊  ┊79┊    addChat(userId: ID!): Chat
+┊  ┊80┊    addGroup(userIds: [ID!]!, groupName: String!, groupPicture: String): Chat
+┊  ┊81┊    updateGroup(chatId: ID!, groupName: String, groupPicture: String): Chat
+┊  ┊82┊    removeChat(chatId: ID!): ID
+┊  ┊83┊    addMessage(chatId: ID!, content: String!): Message
+┊  ┊84┊    removeMessages(chatId: ID!, messageIds: [ID], all: Boolean): [ID]
+┊  ┊85┊    addMembers(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊86┊    removeMembers(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊87┊    addAdmins(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊88┊    removeAdmins(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊89┊    setGroupName(groupId: ID!): String
+┊  ┊90┊    setGroupPicture(groupId: ID!): String
+┊  ┊91┊    markAsReceived(chatId: ID!): Boolean
+┊  ┊92┊    markAsRead(chatId: ID!): Boolean
+┊  ┊93┊  }
 ┊76┊94┊`;
```

##### Changed types.d.ts
```diff
@@ -98,6 +98,38 @@
 ┊ 98┊ 98┊  readAt?: Maybe<Date>;
 ┊ 99┊ 99┊}
 ┊100┊100┊
+┊   ┊101┊export interface Mutation {
+┊   ┊102┊  updateUser: User;
+┊   ┊103┊
+┊   ┊104┊  addChat?: Maybe<Chat>;
+┊   ┊105┊
+┊   ┊106┊  addGroup?: Maybe<Chat>;
+┊   ┊107┊
+┊   ┊108┊  updateGroup?: Maybe<Chat>;
+┊   ┊109┊
+┊   ┊110┊  removeChat?: Maybe<string>;
+┊   ┊111┊
+┊   ┊112┊  addMessage?: Maybe<Message>;
+┊   ┊113┊
+┊   ┊114┊  removeMessages?: Maybe<(Maybe<string>)[]>;
+┊   ┊115┊
+┊   ┊116┊  addMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊117┊
+┊   ┊118┊  removeMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊119┊
+┊   ┊120┊  addAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊121┊
+┊   ┊122┊  removeAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊123┊
+┊   ┊124┊  setGroupName?: Maybe<string>;
+┊   ┊125┊
+┊   ┊126┊  setGroupPicture?: Maybe<string>;
+┊   ┊127┊
+┊   ┊128┊  markAsReceived?: Maybe<boolean>;
+┊   ┊129┊
+┊   ┊130┊  markAsRead?: Maybe<boolean>;
+┊   ┊131┊}
+┊   ┊132┊
 ┊101┊133┊// ====================================================
 ┊102┊134┊// Arguments
 ┊103┊135┊// ====================================================
```
```diff
@@ -108,6 +140,75 @@
 ┊108┊140┊export interface MessagesChatArgs {
 ┊109┊141┊  amount?: Maybe<number>;
 ┊110┊142┊}
+┊   ┊143┊export interface UpdateUserMutationArgs {
+┊   ┊144┊  name?: Maybe<string>;
+┊   ┊145┊
+┊   ┊146┊  picture?: Maybe<string>;
+┊   ┊147┊}
+┊   ┊148┊export interface AddChatMutationArgs {
+┊   ┊149┊  userId: string;
+┊   ┊150┊}
+┊   ┊151┊export interface AddGroupMutationArgs {
+┊   ┊152┊  userIds: string[];
+┊   ┊153┊
+┊   ┊154┊  groupName: string;
+┊   ┊155┊
+┊   ┊156┊  groupPicture?: Maybe<string>;
+┊   ┊157┊}
+┊   ┊158┊export interface UpdateGroupMutationArgs {
+┊   ┊159┊  chatId: string;
+┊   ┊160┊
+┊   ┊161┊  groupName?: Maybe<string>;
+┊   ┊162┊
+┊   ┊163┊  groupPicture?: Maybe<string>;
+┊   ┊164┊}
+┊   ┊165┊export interface RemoveChatMutationArgs {
+┊   ┊166┊  chatId: string;
+┊   ┊167┊}
+┊   ┊168┊export interface AddMessageMutationArgs {
+┊   ┊169┊  chatId: string;
+┊   ┊170┊
+┊   ┊171┊  content: string;
+┊   ┊172┊}
+┊   ┊173┊export interface RemoveMessagesMutationArgs {
+┊   ┊174┊  chatId: string;
+┊   ┊175┊
+┊   ┊176┊  messageIds?: Maybe<(Maybe<string>)[]>;
+┊   ┊177┊
+┊   ┊178┊  all?: Maybe<boolean>;
+┊   ┊179┊}
+┊   ┊180┊export interface AddMembersMutationArgs {
+┊   ┊181┊  groupId: string;
+┊   ┊182┊
+┊   ┊183┊  userIds: string[];
+┊   ┊184┊}
+┊   ┊185┊export interface RemoveMembersMutationArgs {
+┊   ┊186┊  groupId: string;
+┊   ┊187┊
+┊   ┊188┊  userIds: string[];
+┊   ┊189┊}
+┊   ┊190┊export interface AddAdminsMutationArgs {
+┊   ┊191┊  groupId: string;
+┊   ┊192┊
+┊   ┊193┊  userIds: string[];
+┊   ┊194┊}
+┊   ┊195┊export interface RemoveAdminsMutationArgs {
+┊   ┊196┊  groupId: string;
+┊   ┊197┊
+┊   ┊198┊  userIds: string[];
+┊   ┊199┊}
+┊   ┊200┊export interface SetGroupNameMutationArgs {
+┊   ┊201┊  groupId: string;
+┊   ┊202┊}
+┊   ┊203┊export interface SetGroupPictureMutationArgs {
+┊   ┊204┊  groupId: string;
+┊   ┊205┊}
+┊   ┊206┊export interface MarkAsReceivedMutationArgs {
+┊   ┊207┊  chatId: string;
+┊   ┊208┊}
+┊   ┊209┊export interface MarkAsReadMutationArgs {
+┊   ┊210┊  chatId: string;
+┊   ┊211┊}
 ┊111┊212┊
 ┊112┊213┊import {
 ┊113┊214┊  GraphQLResolveInfo,
```
```diff
@@ -454,6 +555,227 @@
 ┊454┊555┊  > = Resolver<R, Parent, TContext>;
 ┊455┊556┊}
 ┊456┊557┊
+┊   ┊558┊export namespace MutationResolvers {
+┊   ┊559┊  export interface Resolvers<TContext = {}, TypeParent = {}> {
+┊   ┊560┊    updateUser?: UpdateUserResolver<UserDb, TypeParent, TContext>;
+┊   ┊561┊
+┊   ┊562┊    addChat?: AddChatResolver<Maybe<ChatDb>, TypeParent, TContext>;
+┊   ┊563┊
+┊   ┊564┊    addGroup?: AddGroupResolver<Maybe<ChatDb>, TypeParent, TContext>;
+┊   ┊565┊
+┊   ┊566┊    updateGroup?: UpdateGroupResolver<Maybe<ChatDb>, TypeParent, TContext>;
+┊   ┊567┊
+┊   ┊568┊    removeChat?: RemoveChatResolver<Maybe<string>, TypeParent, TContext>;
+┊   ┊569┊
+┊   ┊570┊    addMessage?: AddMessageResolver<Maybe<MessageDb>, TypeParent, TContext>;
+┊   ┊571┊
+┊   ┊572┊    removeMessages?: RemoveMessagesResolver<
+┊   ┊573┊      Maybe<(Maybe<string>)[]>,
+┊   ┊574┊      TypeParent,
+┊   ┊575┊      TContext
+┊   ┊576┊    >;
+┊   ┊577┊
+┊   ┊578┊    addMembers?: AddMembersResolver<
+┊   ┊579┊      Maybe<(Maybe<string>)[]>,
+┊   ┊580┊      TypeParent,
+┊   ┊581┊      TContext
+┊   ┊582┊    >;
+┊   ┊583┊
+┊   ┊584┊    removeMembers?: RemoveMembersResolver<
+┊   ┊585┊      Maybe<(Maybe<string>)[]>,
+┊   ┊586┊      TypeParent,
+┊   ┊587┊      TContext
+┊   ┊588┊    >;
+┊   ┊589┊
+┊   ┊590┊    addAdmins?: AddAdminsResolver<
+┊   ┊591┊      Maybe<(Maybe<string>)[]>,
+┊   ┊592┊      TypeParent,
+┊   ┊593┊      TContext
+┊   ┊594┊    >;
+┊   ┊595┊
+┊   ┊596┊    removeAdmins?: RemoveAdminsResolver<
+┊   ┊597┊      Maybe<(Maybe<string>)[]>,
+┊   ┊598┊      TypeParent,
+┊   ┊599┊      TContext
+┊   ┊600┊    >;
+┊   ┊601┊
+┊   ┊602┊    setGroupName?: SetGroupNameResolver<Maybe<string>, TypeParent, TContext>;
+┊   ┊603┊
+┊   ┊604┊    setGroupPicture?: SetGroupPictureResolver<
+┊   ┊605┊      Maybe<string>,
+┊   ┊606┊      TypeParent,
+┊   ┊607┊      TContext
+┊   ┊608┊    >;
+┊   ┊609┊
+┊   ┊610┊    markAsReceived?: MarkAsReceivedResolver<
+┊   ┊611┊      Maybe<boolean>,
+┊   ┊612┊      TypeParent,
+┊   ┊613┊      TContext
+┊   ┊614┊    >;
+┊   ┊615┊
+┊   ┊616┊    markAsRead?: MarkAsReadResolver<Maybe<boolean>, TypeParent, TContext>;
+┊   ┊617┊  }
+┊   ┊618┊
+┊   ┊619┊  export type UpdateUserResolver<
+┊   ┊620┊    R = UserDb,
+┊   ┊621┊    Parent = {},
+┊   ┊622┊    TContext = {}
+┊   ┊623┊  > = Resolver<R, Parent, TContext, UpdateUserArgs>;
+┊   ┊624┊  export interface UpdateUserArgs {
+┊   ┊625┊    name?: Maybe<string>;
+┊   ┊626┊
+┊   ┊627┊    picture?: Maybe<string>;
+┊   ┊628┊  }
+┊   ┊629┊
+┊   ┊630┊  export type AddChatResolver<
+┊   ┊631┊    R = Maybe<ChatDb>,
+┊   ┊632┊    Parent = {},
+┊   ┊633┊    TContext = {}
+┊   ┊634┊  > = Resolver<R, Parent, TContext, AddChatArgs>;
+┊   ┊635┊  export interface AddChatArgs {
+┊   ┊636┊    userId: string;
+┊   ┊637┊  }
+┊   ┊638┊
+┊   ┊639┊  export type AddGroupResolver<
+┊   ┊640┊    R = Maybe<ChatDb>,
+┊   ┊641┊    Parent = {},
+┊   ┊642┊    TContext = {}
+┊   ┊643┊  > = Resolver<R, Parent, TContext, AddGroupArgs>;
+┊   ┊644┊  export interface AddGroupArgs {
+┊   ┊645┊    userIds: string[];
+┊   ┊646┊
+┊   ┊647┊    groupName: string;
+┊   ┊648┊
+┊   ┊649┊    groupPicture?: Maybe<string>;
+┊   ┊650┊  }
+┊   ┊651┊
+┊   ┊652┊  export type UpdateGroupResolver<
+┊   ┊653┊    R = Maybe<ChatDb>,
+┊   ┊654┊    Parent = {},
+┊   ┊655┊    TContext = {}
+┊   ┊656┊  > = Resolver<R, Parent, TContext, UpdateGroupArgs>;
+┊   ┊657┊  export interface UpdateGroupArgs {
+┊   ┊658┊    chatId: string;
+┊   ┊659┊
+┊   ┊660┊    groupName?: Maybe<string>;
+┊   ┊661┊
+┊   ┊662┊    groupPicture?: Maybe<string>;
+┊   ┊663┊  }
+┊   ┊664┊
+┊   ┊665┊  export type RemoveChatResolver<
+┊   ┊666┊    R = Maybe<string>,
+┊   ┊667┊    Parent = {},
+┊   ┊668┊    TContext = {}
+┊   ┊669┊  > = Resolver<R, Parent, TContext, RemoveChatArgs>;
+┊   ┊670┊  export interface RemoveChatArgs {
+┊   ┊671┊    chatId: string;
+┊   ┊672┊  }
+┊   ┊673┊
+┊   ┊674┊  export type AddMessageResolver<
+┊   ┊675┊    R = Maybe<MessageDb>,
+┊   ┊676┊    Parent = {},
+┊   ┊677┊    TContext = {}
+┊   ┊678┊  > = Resolver<R, Parent, TContext, AddMessageArgs>;
+┊   ┊679┊  export interface AddMessageArgs {
+┊   ┊680┊    chatId: string;
+┊   ┊681┊
+┊   ┊682┊    content: string;
+┊   ┊683┊  }
+┊   ┊684┊
+┊   ┊685┊  export type RemoveMessagesResolver<
+┊   ┊686┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊687┊    Parent = {},
+┊   ┊688┊    TContext = {}
+┊   ┊689┊  > = Resolver<R, Parent, TContext, RemoveMessagesArgs>;
+┊   ┊690┊  export interface RemoveMessagesArgs {
+┊   ┊691┊    chatId: string;
+┊   ┊692┊
+┊   ┊693┊    messageIds?: Maybe<(Maybe<string>)[]>;
+┊   ┊694┊
+┊   ┊695┊    all?: Maybe<boolean>;
+┊   ┊696┊  }
+┊   ┊697┊
+┊   ┊698┊  export type AddMembersResolver<
+┊   ┊699┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊700┊    Parent = {},
+┊   ┊701┊    TContext = {}
+┊   ┊702┊  > = Resolver<R, Parent, TContext, AddMembersArgs>;
+┊   ┊703┊  export interface AddMembersArgs {
+┊   ┊704┊    groupId: string;
+┊   ┊705┊
+┊   ┊706┊    userIds: string[];
+┊   ┊707┊  }
+┊   ┊708┊
+┊   ┊709┊  export type RemoveMembersResolver<
+┊   ┊710┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊711┊    Parent = {},
+┊   ┊712┊    TContext = {}
+┊   ┊713┊  > = Resolver<R, Parent, TContext, RemoveMembersArgs>;
+┊   ┊714┊  export interface RemoveMembersArgs {
+┊   ┊715┊    groupId: string;
+┊   ┊716┊
+┊   ┊717┊    userIds: string[];
+┊   ┊718┊  }
+┊   ┊719┊
+┊   ┊720┊  export type AddAdminsResolver<
+┊   ┊721┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊722┊    Parent = {},
+┊   ┊723┊    TContext = {}
+┊   ┊724┊  > = Resolver<R, Parent, TContext, AddAdminsArgs>;
+┊   ┊725┊  export interface AddAdminsArgs {
+┊   ┊726┊    groupId: string;
+┊   ┊727┊
+┊   ┊728┊    userIds: string[];
+┊   ┊729┊  }
+┊   ┊730┊
+┊   ┊731┊  export type RemoveAdminsResolver<
+┊   ┊732┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊733┊    Parent = {},
+┊   ┊734┊    TContext = {}
+┊   ┊735┊  > = Resolver<R, Parent, TContext, RemoveAdminsArgs>;
+┊   ┊736┊  export interface RemoveAdminsArgs {
+┊   ┊737┊    groupId: string;
+┊   ┊738┊
+┊   ┊739┊    userIds: string[];
+┊   ┊740┊  }
+┊   ┊741┊
+┊   ┊742┊  export type SetGroupNameResolver<
+┊   ┊743┊    R = Maybe<string>,
+┊   ┊744┊    Parent = {},
+┊   ┊745┊    TContext = {}
+┊   ┊746┊  > = Resolver<R, Parent, TContext, SetGroupNameArgs>;
+┊   ┊747┊  export interface SetGroupNameArgs {
+┊   ┊748┊    groupId: string;
+┊   ┊749┊  }
+┊   ┊750┊
+┊   ┊751┊  export type SetGroupPictureResolver<
+┊   ┊752┊    R = Maybe<string>,
+┊   ┊753┊    Parent = {},
+┊   ┊754┊    TContext = {}
+┊   ┊755┊  > = Resolver<R, Parent, TContext, SetGroupPictureArgs>;
+┊   ┊756┊  export interface SetGroupPictureArgs {
+┊   ┊757┊    groupId: string;
+┊   ┊758┊  }
+┊   ┊759┊
+┊   ┊760┊  export type MarkAsReceivedResolver<
+┊   ┊761┊    R = Maybe<boolean>,
+┊   ┊762┊    Parent = {},
+┊   ┊763┊    TContext = {}
+┊   ┊764┊  > = Resolver<R, Parent, TContext, MarkAsReceivedArgs>;
+┊   ┊765┊  export interface MarkAsReceivedArgs {
+┊   ┊766┊    chatId: string;
+┊   ┊767┊  }
+┊   ┊768┊
+┊   ┊769┊  export type MarkAsReadResolver<
+┊   ┊770┊    R = Maybe<boolean>,
+┊   ┊771┊    Parent = {},
+┊   ┊772┊    TContext = {}
+┊   ┊773┊  > = Resolver<R, Parent, TContext, MarkAsReadArgs>;
+┊   ┊774┊  export interface MarkAsReadArgs {
+┊   ┊775┊    chatId: string;
+┊   ┊776┊  }
+┊   ┊777┊}
+┊   ┊778┊
 ┊457┊779┊/** Directs the executor to skip this field or fragment when the `if` argument is true. */
 ┊458┊780┊export type SkipDirectiveResolver<Result> = DirectiveResolverFn<
 ┊459┊781┊  Result,
```
```diff
@@ -497,6 +819,7 @@
 ┊497┊819┊  Chat?: ChatResolvers.Resolvers<TContext>;
 ┊498┊820┊  Message?: MessageResolvers.Resolvers<TContext>;
 ┊499┊821┊  Recipient?: RecipientResolvers.Resolvers<TContext>;
+┊   ┊822┊  Mutation?: MutationResolvers.Resolvers<TContext>;
 ┊500┊823┊  Date?: GraphQLScalarType;
 ┊501┊824┊}
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

Finally, remember to run the generator:

    yarn generator

## Client

For the client I'll only show you how to make use of the addMessage mutation in this chapters. The other mutations will require much more boilerplate so I left them for their own chapter.

Let's start by wiring the addMessage mutation. We're going to write the GraphQL query and then use the generator to generate the types:

[{]: <helper> (diffStep "5.1" files="^\(?!src/types.d.ts$\).*" module="client")

#### [Step 5.1: Create addMessage mutation and generate types](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/f68127c)

##### Changed src&#x2F;graphql.ts
```diff
@@ -13,6 +13,21 @@
 ┊13┊13┊// Documents
 ┊14┊14┊// ====================================================
 ┊15┊15┊
+┊  ┊16┊export namespace AddMessage {
+┊  ┊17┊  export type Variables = {
+┊  ┊18┊    chatId: string;
+┊  ┊19┊    content: string;
+┊  ┊20┊  };
+┊  ┊21┊
+┊  ┊22┊  export type Mutation = {
+┊  ┊23┊    __typename?: "Mutation";
+┊  ┊24┊
+┊  ┊25┊    addMessage: Maybe<AddMessage>;
+┊  ┊26┊  };
+┊  ┊27┊
+┊  ┊28┊  export type AddMessage = Message.Fragment;
+┊  ┊29┊}
+┊  ┊30┊
 ┊16┊31┊export namespace GetChat {
 ┊17┊32┊  export type Variables = {
 ┊18┊33┊    chatId: string;
```
```diff
@@ -244,6 +259,38 @@
 ┊244┊259┊  readAt?: Maybe<DateTime>;
 ┊245┊260┊}
 ┊246┊261┊
+┊   ┊262┊export interface Mutation {
+┊   ┊263┊  updateUser: User;
+┊   ┊264┊
+┊   ┊265┊  addChat?: Maybe<Chat>;
+┊   ┊266┊
+┊   ┊267┊  addGroup?: Maybe<Chat>;
+┊   ┊268┊
+┊   ┊269┊  updateGroup?: Maybe<Chat>;
+┊   ┊270┊
+┊   ┊271┊  removeChat?: Maybe<string>;
+┊   ┊272┊
+┊   ┊273┊  addMessage?: Maybe<Message>;
+┊   ┊274┊
+┊   ┊275┊  removeMessages?: Maybe<(Maybe<string>)[]>;
+┊   ┊276┊
+┊   ┊277┊  addMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊278┊
+┊   ┊279┊  removeMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊280┊
+┊   ┊281┊  addAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊282┊
+┊   ┊283┊  removeAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊284┊
+┊   ┊285┊  setGroupName?: Maybe<string>;
+┊   ┊286┊
+┊   ┊287┊  setGroupPicture?: Maybe<string>;
+┊   ┊288┊
+┊   ┊289┊  markAsReceived?: Maybe<boolean>;
+┊   ┊290┊
+┊   ┊291┊  markAsRead?: Maybe<boolean>;
+┊   ┊292┊}
+┊   ┊293┊
 ┊247┊294┊// ====================================================
 ┊248┊295┊// Arguments
 ┊249┊296┊// ====================================================
```
```diff
@@ -254,6 +301,75 @@
 ┊254┊301┊export interface MessagesChatArgs {
 ┊255┊302┊  amount?: Maybe<number>;
 ┊256┊303┊}
+┊   ┊304┊export interface UpdateUserMutationArgs {
+┊   ┊305┊  name?: Maybe<string>;
+┊   ┊306┊
+┊   ┊307┊  picture?: Maybe<string>;
+┊   ┊308┊}
+┊   ┊309┊export interface AddChatMutationArgs {
+┊   ┊310┊  userId: string;
+┊   ┊311┊}
+┊   ┊312┊export interface AddGroupMutationArgs {
+┊   ┊313┊  userIds: string[];
+┊   ┊314┊
+┊   ┊315┊  groupName: string;
+┊   ┊316┊
+┊   ┊317┊  groupPicture?: Maybe<string>;
+┊   ┊318┊}
+┊   ┊319┊export interface UpdateGroupMutationArgs {
+┊   ┊320┊  chatId: string;
+┊   ┊321┊
+┊   ┊322┊  groupName?: Maybe<string>;
+┊   ┊323┊
+┊   ┊324┊  groupPicture?: Maybe<string>;
+┊   ┊325┊}
+┊   ┊326┊export interface RemoveChatMutationArgs {
+┊   ┊327┊  chatId: string;
+┊   ┊328┊}
+┊   ┊329┊export interface AddMessageMutationArgs {
+┊   ┊330┊  chatId: string;
+┊   ┊331┊
+┊   ┊332┊  content: string;
+┊   ┊333┊}
+┊   ┊334┊export interface RemoveMessagesMutationArgs {
+┊   ┊335┊  chatId: string;
+┊   ┊336┊
+┊   ┊337┊  messageIds?: Maybe<(Maybe<string>)[]>;
+┊   ┊338┊
+┊   ┊339┊  all?: Maybe<boolean>;
+┊   ┊340┊}
+┊   ┊341┊export interface AddMembersMutationArgs {
+┊   ┊342┊  groupId: string;
+┊   ┊343┊
+┊   ┊344┊  userIds: string[];
+┊   ┊345┊}
+┊   ┊346┊export interface RemoveMembersMutationArgs {
+┊   ┊347┊  groupId: string;
+┊   ┊348┊
+┊   ┊349┊  userIds: string[];
+┊   ┊350┊}
+┊   ┊351┊export interface AddAdminsMutationArgs {
+┊   ┊352┊  groupId: string;
+┊   ┊353┊
+┊   ┊354┊  userIds: string[];
+┊   ┊355┊}
+┊   ┊356┊export interface RemoveAdminsMutationArgs {
+┊   ┊357┊  groupId: string;
+┊   ┊358┊
+┊   ┊359┊  userIds: string[];
+┊   ┊360┊}
+┊   ┊361┊export interface SetGroupNameMutationArgs {
+┊   ┊362┊  groupId: string;
+┊   ┊363┊}
+┊   ┊364┊export interface SetGroupPictureMutationArgs {
+┊   ┊365┊  groupId: string;
+┊   ┊366┊}
+┊   ┊367┊export interface MarkAsReceivedMutationArgs {
+┊   ┊368┊  chatId: string;
+┊   ┊369┊}
+┊   ┊370┊export interface MarkAsReadMutationArgs {
+┊   ┊371┊  chatId: string;
+┊   ┊372┊}
 ┊257┊373┊
 ┊258┊374┊// ====================================================
 ┊259┊375┊// START: Apollo Angular template
```
```diff
@@ -318,6 +434,23 @@
 ┊318┊434┊// Apollo Services
 ┊319┊435┊// ====================================================
 ┊320┊436┊
+┊   ┊437┊@Injectable({
+┊   ┊438┊  providedIn: "root"
+┊   ┊439┊})
+┊   ┊440┊export class AddMessageGQL extends Apollo.Mutation<
+┊   ┊441┊  AddMessage.Mutation,
+┊   ┊442┊  AddMessage.Variables
+┊   ┊443┊> {
+┊   ┊444┊  document: any = gql`
+┊   ┊445┊    mutation AddMessage($chatId: ID!, $content: String!) {
+┊   ┊446┊      addMessage(chatId: $chatId, content: $content) {
+┊   ┊447┊        ...Message
+┊   ┊448┊      }
+┊   ┊449┊    }
+┊   ┊450┊
+┊   ┊451┊    ${MessageFragment}
+┊   ┊452┊  `;
+┊   ┊453┊}
 ┊321┊454┊@Injectable({
 ┊322┊455┊  providedIn: "root"
 ┊323┊456┊})
```

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

    yarn generator

Now let's use the just-created query:

[{]: <helper> (diffStep "5.2" module="client")

#### [Step 5.2: Modify chat component and service](https://github.com/Urigo/WhatsApp-Clone-Client-Angular/commit/e71b468)

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -14,7 +14,7 @@
 ┊14┊14┊    </app-toolbar>
 ┊15┊15┊    <div class="container">
 ┊16┊16┊      <app-messages-list [items]="messages" [isGroup]="isGroup"></app-messages-list>
-┊17┊  ┊      <app-new-message></app-new-message>
+┊  ┊17┊      <app-new-message (newMessage)="addMessage($event)"></app-new-message>
 ┊18┊18┊    </div>
 ┊19┊19┊  `,
 ┊20┊20┊  styleUrls: ['./chat.component.scss']
```
```diff
@@ -44,4 +44,8 @@
 ┊44┊44┊  goToChats() {
 ┊45┊45┊    this.router.navigate(['/chats']);
 ┊46┊46┊  }
+┊  ┊47┊
+┊  ┊48┊  addMessage(content: string) {
+┊  ┊49┊    this.chatsService.addMessage(this.chatId, content).subscribe();
+┊  ┊50┊  }
 ┊47┊51┊}
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,6 +1,6 @@
 ┊1┊1┊import {map} from 'rxjs/operators';
 ┊2┊2┊import {Injectable} from '@angular/core';
-┊3┊ ┊import {GetChatsGQL, GetChatGQL} from '../../graphql';
+┊ ┊3┊import {GetChatsGQL, GetChatGQL, AddMessageGQL} from '../../graphql';
 ┊4┊4┊
 ┊5┊5┊@Injectable()
 ┊6┊6┊export class ChatsService {
```
```diff
@@ -8,7 +8,8 @@
 ┊ 8┊ 8┊
 ┊ 9┊ 9┊  constructor(
 ┊10┊10┊    private getChatsGQL: GetChatsGQL,
-┊11┊  ┊    private getChatGQL: GetChatGQL
+┊  ┊11┊    private getChatGQL: GetChatGQL,
+┊  ┊12┊    private addMessageGQL: AddMessageGQL
 ┊12┊13┊  ) {}
 ┊13┊14┊
 ┊14┊15┊  getChats() {
```
```diff
@@ -33,4 +34,11 @@
 ┊33┊34┊
 ┊34┊35┊    return {query, chat$};
 ┊35┊36┊  }
+┊  ┊37┊
+┊  ┊38┊  addMessage(chatId: string, content: string) {
+┊  ┊39┊    return this.addMessageGQL.mutate({
+┊  ┊40┊      chatId,
+┊  ┊41┊      content,
+┊  ┊42┊    });
+┊  ┊43┊  }
 ┊36┊44┊}
```

[}]: #

It's that simple! You would be tempted to say that it doesn't work, but you should try to refresh the page first ;)


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step4.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@2.0.0/.tortilla/manuals/views/step6.md) |
|:--------------------------------|--------------------------------:|

[}]: #
