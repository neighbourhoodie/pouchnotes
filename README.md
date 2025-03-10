# Pouchnotes

A simple, offline-first notes app. Intended as a demo application for [CouchDB Minihosting](https://github.com/neighbourhoodie/couchdb-minihosting).


Pouchnotes allows you to take notes on multiple devices, online or offline, and they will all be synced with each other in real time through a central CouchDB.

The app uses Vite + Svelte 5 + Pico.css + PouchDB.

For simplicity, it does _not_ store a session, so you’ll have to log in every time you reload.

![screenshot of the pouchnotes app](<pouchnotes_screenshot.png>)

## Setup

Pouchnotes is intended to be deployed on a CouchDB Minihosting instance via the deployment method provided by that repo. After you’ve set up your CouchDB Minihosting, follow the [deployment instructions](https://github.com/neighbourhoodie/couchdb-minihosting/tree/main/deployment).

To build Pouchnotes:

1. Install dependencies:
    ```sh
    $ npm i
    ```
2. Update the `.env.production` file with your CouchDB Minihosting endpoint URL, eg. 
    ```
    VITE_LOCAL_COUCHDB_ENDPOINT=https://backend.lol/_api
    ```
3. Build and deploy:
    ```sh
    $ npm run build && ./deploy.sh
    ```

Before you can use the app, you need to set up a user and a target DB in your CouchDB. We’re going do this with cURL. In all of these commands, replace `COUCHDB_USERNAME` and `COUCHDB_PASSWORD` with the admin credentials to your CouchDB, and the `COUCHDB_ENDPOINT` with your CouchDB Minihosting endpoint, same as in the `.env.production` file above.

Replace _all_ instances of `YOUR_USERNAME` in all commands with the username you’d like to log into Pouchnotes with (the first command has two!). 

1. **Add a user**
   
   Because we’re going to be using the username as part of a database name in a second, make sure it conforms to these rules: only lowercase characters `a-z`, digits `0-9`, and any of the characters `_ $ ( ) + -` and `/` are allowed. Must begin with a letter.

   You might also want to change the password 😅

    ```sh 
    curl  -X PUT \
    'COUCHDB_ENDPOINT/_users/org.couchdb.user:YOUR_USERNAME' \
    --user COUCHDB_USERNAME:COUCHDB_PASSWORD \
    --data-raw '{
      "password": "12341234",
      "roles": [
        "users"
      ],
      "name": "YOUR_USERNAME",
      "type": "user"
    }'
    ```
2. **Add a database**
   
   Call it `pouchnotes-${your-username-from-step-1}`, eg. `pouchnotes-fenella`:

    ```sh
    curl  -X PUT \
    'COUCHDB_ENDPOINT/pouchnotes-YOUR_USERNAME' \
    --user COUCHDB_USERNAME:COUCHDB_PASSWORD
    ```

3. **Give your user permission to access that database**:

    ```sh
    curl  -X PUT \
    'COUCHDB_ENDPOINT/pouchnotes-YOUR_USERNAME/_security' \
    --user COUCHDB_USERNAME:COUCHDB_PASSWORD \
    --data-raw '{
      "members": {
        "names": [
          "YOUR_USERNAME"
        ]
      }
    }'
    ```

You should now have a user, a DB with that user’s name as a suffix, and that user should be a member of that DB. Now when you deploy Pouchnotes to your CouchDB Minihosting instance, you should be able to log in with that user’s credentials and add notes! For more users, just repeat the process with a different username.

## Miscellaneous

- Using PouchDB with Vite required two small extra steps: 
  1. `$ npm i events`
  2. Add `define: { global: "window" }` to the `defineConfig` object in `vite.config.ts`