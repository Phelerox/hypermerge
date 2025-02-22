# Hypermerge

Hypermerge is a Node.js library for building p2p collaborative applications
without any server infrastructure. It combines [Automerge][automerge], a CRDT,
with [hypercore][hypercore], a distributed append-only log.


If successful this project would provide a way to have apps data sets that are
conflict free and offline first (thanks to CRDT's) and serverless (thanks to
hypercore/DAT).

While the DAT community has done a lot of work to secure their tool set, zero
effort has been made with hypermerge itself to deal with security and privacy
concerns.  Due to the secure nature of the tools its built upon a properly
audited and secure version of this library would be possible in the future.

## How it works

## Concepts

The base object you make with hypermerge is a Repo.  A repo is responsible for
managing documents and replicating to peers.

### Basic Setup (Serverless or with a Server)

```ts
import { Repo } from "hypermerge"

const storage = require("random-access-file")
const path = ".data"

const repo = new Repo({ path, storage })

// DAT's discovery swarm or truly serverless discovery
const DiscoverySwarm = require("discovery-swarm");
const defaults = require('dat-swarm-defaults')
const discovery = new DiscoverySwarm(defaults({stream: repo.stream, id: repo.id }));

repo.replicate(discovery)
```

### Create / Edit / View / Delete a document

```ts

  const url = repo.create({ hello: "world" })

  repo.doc<any>(url, (doc) => {
    console.log(doc) // { hello: "world" }
  })

  // this is an automerge change function - see automerge for more info
  // basically you get to treat the state as a plain old javacript object
  // operations you make will be added to an internal append only log and
  // replicated to peers

  repo.change(url, (state:any) => {
    state.foo = "bar"
  })

  repo.doc<any>(url, (doc) => {
    console.log(doc) // { hello: "world", foo: "bar" }
  })

  // to watch a document that changes over time ...
  const handle = repo.watch(url, (doc:any) => {
    console.log(doc)
    if (doc.foo === "bar") {
      handle.close()
    }
  })

```

*NOTE*: If you're familiar with Automerge: the `change` function in Hypermerge is asynchronous, while the `Automerge.change` function is synchronous. What this means is that although `Automerge.change` returns an object representing the new state of your document, `repo.change` (or `handle.change`) does NOT. So:

```ts
// ok in Automerge!
doc1 = Automerge.change(doc1, "msg", (doc) => {
  doc.foo = "bar"
})

// NOT ok in Hypermerge!
doc1 = repo.change(url1, (doc) => {
  doc.foo = "bar"
})
```

Instead, you should expect to get document state updates via `repo.watch` (or `handle.subscribe`) as shown in the example above.

### Two repos on different machines

```ts

const docUrl = repoA.create({ numbers: [ 2,3,4 ]})
// this will block until the state has replicated to machine B

repoA.watch<MyDoc>(docUrl, state => {
  console.log("RepoA", state)
  // { numbers: [2,3,4] } 
  // { numbers: [2,3,4,5], foo: "bar" }
  // { numbers: [2,3,4,5], foo: "bar" } // (local changes repeat)
  // { numbers: [1,2,3,4,5], foo: "bar", bar: "foo" }
})

repoB.watch<MyDoc>(docUrl, state => {
  console.log("RepoB", state)
  // { numbers: [1,2,3,4,5], foo: "bar", bar: "foo" }
})


repoA.change<MyDoc>(docUrl, (state) => {
  state.numbers.push(5)
  state.foo = "bar"
})

repoB.change<MyDoc>(docUrl, (state) => {
  state.numbers.unshift(1)
  state.bar = "foo"
})

```

### Accessing Files
Hypermerge supports a special kind of core called a hyperfile. Hyperfiles are unchanging, written-once hypercores that store binary data.

Here's a simple example of reading and writing hyperfile buffers.

```ts

export function writeBuffer(buffer, callback) {
  const hyperfileUrl = repo.writeFile(buffer, 'application/octet-stream') // TODO: mime type
  callback(null, hyperfileUrl)
}

export function fetch(hyperfileId, callback) {
  repo.readFile(hyperfileId, (error, data) => {
    if (error) {
      callback(error)
      return
    }

    callback(null, data)
  })
}

```

Note that hyperfiles are conveniently well-suited to treatment as a native protocol for Electron applications. This allows you to refer to them throughout your application directly as though they were regular files for images and other assets without any special treatment. Here's how to register that:

```js

  protocol.registerBufferProtocol('hyperfile', (request, callback) => {
    try {
      Hyperfile.fetch(request.url, (data) => {
        callback({ data })
      })
    } catch (e) {
      log(e)
    }
  }, (error) => {
    if (error) {
      log('Failed to register protocol')
    }
  })
 ```

### Splitting the Front-end and Back-end

Both Hypermerge and Automerge supports separating the front-end (where materialized documents live and changes are made) from the backend (where CRDT computations are handled as well as networking and compression.) This is useful for maintaining application performance by moving expensive computation operations off of the render thread to another location where they don't block user input.

The communication between front-end and back-end is all done via simple Javascript objects and can be serialized/deserialized through JSON if required. 

```js
  // establish a back-end
  const back = new RepoBackend({ storage: raf, path: HYPERMERGE_PATH, port: 0 })
  const discovery = new DiscoverySwarm({ /* your config here */ })
  back.replicate(discovery)

  // elsewhere, create a front-end (you'll probably want to do this in different threads)
  const front = new RepoFrontend()

  // the `subscribe` method sends a message stream, the `receive` receives it
  // for demonstration here we simply output JSON and parse it back in the same location
  // note that front-end and back-end must each subscribe to each other's streams
  back.subscribe((msg) => front.receive(JSON.parse(JSON.stringify(msg))))
  front.subscribe((msg) => back.receive(JSON.parse(JSON.stringify(msg))))
  
}
```

*Note: each back-end only supports a single front-end today.*


### Related libraries

[automerge]: https://github.com/automerge/automerge
[hypercore]: https://github.com/mafintosh/hypercore
