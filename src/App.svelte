<script lang="ts">
  import PouchDB from 'pouchdb-browser'
  import auth from 'pouchdb-authentication'
  import '@picocss/pico'

  PouchDB.plugin(auth);

  type Note = {
    text: string
    type: "note"
  }

  let notes: PouchDB.Core.ExistingDocument<Note>[] = $state([])
  let newNote: string = $state("")

  let username: string = $state("user1")
  let password: string = $state("12341234")
  let hasSession: boolean = $state(false)
  let loginMessage: string = $state("")

  const remoteEndpoint = $derived(`${import.meta.env.VITE_LOCAL_COUCHDB_ENDPOINT}/pouchnotes-${username}`)

  let db: PouchDB.Database<{}>
  let remote
  
  async function go(){
    const changes = db.changes<Note>({
      since: 'now',
      live: true,
      include_docs: true
    }).on('change', function(change) {
      if (change?.doc?.type === 'note') {
        const existingNoteIndex = notes.findIndex(note => {
          return note._id === change.id
        })
        if (existingNoteIndex >= 0) {
          if (change.deleted) {
            notes.splice(existingNoteIndex, 1)
          } else {
            notes[existingNoteIndex] = change.doc
          }
        } else {
          notes.push(change.doc)
        }
      }
    }).on('complete', function(info) {
      console.log('Sync complete/cancelled', info)
    }).on('error', function (err) {
      console.log('Sync error:', err);
    });
    const currentNotes = await db.allDocs<Note>({include_docs: true})
    notes = currentNotes.rows?.map(n => n.doc as PouchDB.Core.ExistingDocument<Note>) 
    db.sync(remoteEndpoint, {
      live: true
    });
  }


  const handleLogin = async (e: SubmitEvent) => {
    e.preventDefault()
    remote = new PouchDB(remoteEndpoint)
    try {
      const loginResponse = await remote.logIn(username, password)
      if (loginResponse.ok) {
        hasSession = true
        db = new PouchDB(`pouchnotes-${username}`)
        go()
      }
    } catch (error: any) {
      loginMessage = `Login failed: ${error?.reason || 'Reason unknown'}`
    }
  }

  const handleNewNote = async (e: SubmitEvent) => {
    e.preventDefault()
    await db.put({
      "_id": `note::${new Date().toISOString()}`,
      "type": "note",
      "text": newNote
    })
    newNote = ""
  }

  const handleDelete = async (noteID: string) => {
    const doc = notes.find(note => note._id === noteID)
    if (doc) {
      db.post({...doc, _deleted: true})
    }
  }

  const formatDate = (ISODateString: string) => {
    return new Intl.DateTimeFormat("en-UK").format(new Date(ISODateString))
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
      {note.text}
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
  .delete {
    background: none;
    border: none;
    padding: 0;
  }
</style>