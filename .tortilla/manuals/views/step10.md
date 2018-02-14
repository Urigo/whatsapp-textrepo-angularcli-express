# Step 10: Updating the store

[//]: # (head-end)


## Client

Did you notice that after creating a new message you'll have to refresh the page in order to see it?
How to fix that? If you thought about re-querying the server you would be wrong! The best solution is to use the response provided by the server to update our Apollo local cache.

Apollo performs two important core tasks: Executing queries and mutations, and caching the results.

Thanks to Apollo’s store design, it’s possible for the results of a query or mutation to update your UI in all the right places. In many cases it’s possible for that to happen automatically, whereas in others you need to help the client out a little in doing so:

[{]: <helper> (diffStep "6.1" module="client")

#### Step 6.1: Update the store

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,4 +1,4 @@
-┊1┊ ┊import {ApolloQueryResult, WatchQueryOptions} from 'apollo-client';
+┊ ┊1┊import {ApolloQueryResult, MutationOptions, WatchQueryOptions} from 'apollo-client';
 ┊2┊2┊import {map} from 'rxjs/operators';
 ┊3┊3┊import {Apollo} from 'apollo-angular';
 ┊4┊4┊import {Injectable} from '@angular/core';
```
```diff
@@ -43,12 +43,49 @@
 ┊43┊43┊  }
 ┊44┊44┊
 ┊45┊45┊  addMessage(chatId: string, content: string) {
-┊46┊  ┊    return this.apollo.mutate({
+┊  ┊46┊    return this.apollo.mutate(<MutationOptions>{
 ┊47┊47┊      mutation: addMessageMutation,
 ┊48┊48┊      variables: <AddMessage.Variables>{
 ┊49┊49┊        chatId,
 ┊50┊50┊        content,
 ┊51┊51┊      },
+┊  ┊52┊      update: (store, { data: { addMessage } }: {data: AddMessage.Mutation}) => {
+┊  ┊53┊        // Update the messages cache
+┊  ┊54┊        {
+┊  ┊55┊          // Read the data from our cache for this query.
+┊  ┊56┊          const {chat}: GetChat.Query = store.readQuery({
+┊  ┊57┊            query: getChatQuery, variables: {
+┊  ┊58┊              chatId,
+┊  ┊59┊            }
+┊  ┊60┊          });
+┊  ┊61┊          // Add our message from the mutation to the end.
+┊  ┊62┊          chat.messages.push(addMessage);
+┊  ┊63┊          // Write our data back to the cache.
+┊  ┊64┊          store.writeQuery({ query: getChatQuery, data: {chat} });
+┊  ┊65┊        }
+┊  ┊66┊        // Update last message cache
+┊  ┊67┊        {
+┊  ┊68┊          // Read the data from our cache for this query.
+┊  ┊69┊          const {chats}: GetChats.Query = store.readQuery({
+┊  ┊70┊            query: getChatsQuery,
+┊  ┊71┊            variables: <GetChats.Variables>{
+┊  ┊72┊              amount: this.messagesAmount,
+┊  ┊73┊            },
+┊  ┊74┊          });
+┊  ┊75┊          // Add our comment from the mutation to the end.
+┊  ┊76┊          chats.find(chat => chat.id === chatId).messages.push(addMessage);
+┊  ┊77┊          // Write our data back to the cache.
+┊  ┊78┊          store.writeQuery({
+┊  ┊79┊            query: getChatsQuery,
+┊  ┊80┊            variables: <GetChats.Variables>{
+┊  ┊81┊              amount: this.messagesAmount,
+┊  ┊82┊            },
+┊  ┊83┊            data: {
+┊  ┊84┊              chats,
+┊  ┊85┊            },
+┊  ┊86┊          });
+┊  ┊87┊        }
+┊  ┊88┊      },
 ┊52┊89┊    });
 ┊53┊90┊  }
 ┊54┊91┊}
```

[}]: #

Now you won't need to reload the page in order to see the new message. What's even more interesting is that the message you wrote would also be shown as the last message in the chats list, just hit the back button in the top-left corner to find out!
This is because we updated our store for both the `GetChat` and the `GetChats` query.

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step9.md) | [Next Step >](step11.md) |
|:--------------------------------|--------------------------------:|

[}]: #
