# A Whatsapp clone written with Angular, Material, Express, Postgresql and Apollo GraphQL.

[//]: # (head-end)


## Startup instructions

This project is made out of 2 sub-projects - the one is client and the other is server. We will go through the initialization for each of these individually. We will start with the server since the client is dependent on it.

### Server

First we need to clone the server project:

    $ git clone git@github.com:Urigo/whatsapp-server-express.git

Then we need to install the NPM dependencies:

    $ npm install

Before starting the server make sure that postgresql is installed:

    $ sudo apt-get update
    $ sudo apt-get install postgresql postgresql-contrib

Setup a user named "test" with no password by first switching into "postgres" account:

    $ sudo -u postgres psql

And running the following command:

    postgres=# ALTER USER test WITH PASSWORD '';

Try to run the server:

    $ npm start

If logs show that connection refused, kill the process and set a random password for the "test" user (e.g. "test"):

    postgres=# ALTER USER test WITH PASSWORD 'test';

Be sure to set the password in the `ormconfig.json` as well:

```js
{
  // ...
  "password": "test",
  // ...
}
```

Run the start command again:

    $ npm start

### Client

**Be sure to go through server initialization first**

First we need to clone the server project:

    $ git clone git@github.com:Urigo/whatsapp-client-angularcli-material.git

Then we need to install the NPM dependencies:

    $ npm install

Start the app:

    $ npm start


[//]: # (foot-start)

[{]: <helper> (navStep)

| [Begin Tutorial >](.tortilla/manuals/views/step1.md) |
|----------------------:|

[}]: #
