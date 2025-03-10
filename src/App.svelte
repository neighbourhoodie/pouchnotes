<script lang="ts">
  import PouchDB from 'pouchdb-browser'
  import auth from 'pouchdb-authentication'
  import { onMount } from 'svelte';

  import '@picocss/pico'

  // We'll use the `pouchdb-authentication` plugin to auth with the remote CouchDB
  PouchDB.plugin(auth);

  // Auth vars
  let username: string = $state("")
  let password: string = $state("")
  let hasSession: boolean = $state(false)
  let loginMessage: string = $state("")

  // A derived store that updates as soon as we have a username. After login, this gives us a database name that includes the username
  const databaseName = $derived(`pouchnotes-${username}`)
  const remoteEndpoint = $derived(`${import.meta.env.VITE_LOCAL_COUCHDB_ENDPOINT}/${databaseName}`)

  // We currently only have a single document type in our DB
  type Note = {
    text: string
    type: "note"
  }

  // Future-proofing: this would be a union of all possible document types you store in this DB, eg. `type AnyDocumentType = Note | ToDo | Message`
  type AnyDocumentType = Note

  // We store the replication and changes feed so we can cancel them later
  let replication: PouchDB.Replication.Sync<AnyDocumentType>
  let changes: PouchDB.Core.Changes<AnyDocumentType>

  // This is our reactive notes store we use to populate the UI. This will only end up containing documents of type `note`
  let notes: PouchDB.Core.ExistingDocument<Note>[] = $state([])
  // The contents of the textarea for adding a new note
  let newNote: string = $state("")

  // We don't know our DB names yet, since they depend on the username, which we only get after login. We prep the empty vars here.
  let localDB: PouchDB.Database<AnyDocumentType>
  let remoteDB: PouchDB.Database<AnyDocumentType>
  
  async function getInitialDataAndStartChangesFeed(){
    // Attach the changes listener to the local database
    // We’re typing these all with `Note` because that’s the only type we have
    changes = localDB.changes<AnyDocumentType>({
      since: 'now',
      live: true,
      include_docs: true
    }).on('change', function(change) {
      // Check whether the document in the change is really of the correct type. We only have a single type, but this is good practise anyway, and keeps TypeScript happy
      if (change?.doc?.type === 'note') {
        // First check whether this change affects a note we already have, or is new
        const existingNoteIndex = notes.findIndex(note => {
          return note._id === change.id
        })
        if (existingNoteIndex >= 0) {
          // If the change is a deletion, remove the note from our notes store
          if (change.deleted) {
            notes.splice(existingNoteIndex, 1)
          } else {
            // If the change is just a… well, change, we replace our note with the changed one
            notes[existingNoteIndex] = change.doc
          }
        } else {
          // If we don't have the note yet, we add it to our notes store
          notes.push(change.doc)
        }
      }
    }).on('complete', function(info) {
      console.log('Sync complete/cancelled', info)
    }).on('error', function (err) {
      console.log('Sync error:', err);
    });
    // Fetch all docs from the local DB
    const allDocsResult = await localDB.allDocs<AnyDocumentType>({include_docs: true})
    // Get all the documents of tyoe `note` and populate our notes store with it. We use `reduce()` to find all documents of the `note` type, this ends up being the cleanest method. Using `map()` and/or `filter()` requires additional hoop-jumping.
    notes = allDocsResult.rows?.reduce(
      (result: PouchDB.Core.ExistingDocument<Note>[], row) => {
        if(row.doc?.type === 'note') {
          result.push(row.doc)
        }
        return result
      }, []
    )
    // Turn on continuous replication between our local PouchDB and the remote CouchDB
    replication = localDB.sync(remoteEndpoint, {
      live: true
    });
  }

  onMount(() => {
    return () => {
      // Clean up on unmount
      replication.cancel();
      changes.cancel()
    }
  });

  const handleLogin = async (e: SubmitEvent) => {
    e.preventDefault()
    remoteDB = new PouchDB(remoteEndpoint)
    try {
      // Use the `login()` method provided by the `pouchdb-authentication` plugin to auth with the remote CouchDB
      const loginResponse = await remoteDB.logIn(username, password)
      if (loginResponse.ok) {
        hasSession = true
        // Now that we have the username, we can instantiate the local DB
        localDB = new PouchDB(databaseName)
        getInitialDataAndStartChangesFeed()
      }
    } catch (error: any) {
      loginMessage = `Login failed: ${error?.reason || 'Reason unknown'}`
    }
  }

  const handleNewNote = async (e: SubmitEvent) => {
    e.preventDefault()
    // We put the note into our local PouchDB. The continuous replication will make sure it eventually ends up in the remote CouchDB as well.
    await localDB.put({
      // We’re using the `_id` (CouchDB’s primary key) to encode the type of the document and a sortable, human readable date, with an arbitrary separator `::` that doesn’t occur elsewhere in the id.
      "_id": `note::${new Date().toISOString()}`,
      // It’s always useful to explicitly encode the `type` of a document, even if you’ve only got a single one to start out with. Later, you might have many, and may want to filter or group documents, and this makes it easier.
      "type": "note",
      "text": newNote
    })
    // Reset the textarea
    newNote = ""
  }

  const handleDelete = async (noteID: string) => {
    const doc = notes.find(note => note._id === noteID)
    if (doc) {
      // Delete documents by adding `_deleted: true` to them and POST-ing them back to the DB. The deletion will also be replicated.
      localDB.post({...doc, _deleted: true})
    }
  }

  const formatDate = (ISODateString: string) => {
    return new Intl.DateTimeFormat("en-UK", {
      dateStyle: "medium", 
      timeStyle: 'short'
    }).format(new Date(ISODateString))
  }

</script>

<main>
  <h1>Pouchnotes</h1>
  <p>A simple, offline-first notes app. Intended as a demo application for <a href="https://github.com/neighbourhoodie/couchdb-minihosting">CouchDB Minihosting</a>. Find out more about Pouchnotes at the <a href="https://github.com/neighbourhoodie/pouchnotes">GitHub repo</a>.</p>
  {#if hasSession}
    <form onsubmit={handleNewNote}>
      <textarea bind:value={newNote} placeholder="Enter you new note here"></textarea>
      <input type="submit" value="Save Note">
    </form>
    <small>You have {notes.length} note{notes.length !== 1 ? 's' : ''}</small>
    {#each notes as note, i}
    <article>
      <header>{i+1}. {formatDate(note._id.split('::')[1])}</header>
      <div class="note-text">{note.text}</div>
      <footer><button class="delete" onclick={() => handleDelete(note._id)}>❌</button></footer>
    </article>
    {/each}
  {:else}
    <form onsubmit={handleLogin}>
      <input type="text" bind:value={username} placeholder="Username" />
      <input type="password" bind:value={password} placeholder="Password" />
      <input type="submit" value="Login">
    </form>
    {#if loginMessage}
      <mark>{loginMessage}</mark>
    {/if}
  {/if}
</main>

<style>
  .note-text {
    /* Preserve newlines */
    white-space: pre-line;
  }
  .delete {
    background: none;
    border: none;
    padding: 0;
  }
</style>