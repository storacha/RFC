**Draft**, full writeup to come. Copying from the [`#core-dev` thread that spawned the thought](https://discord.com/channels/1247475892435816553/1402343123048464414):

I'm testing `guppy`'s actual upload process, which means testing the `space/blob/add`. Of course, `space/blob/add` is a [rather complicated conversation](https://github.com/storacha/specs/blob/main/w3-blob.md#add-blob-diagram). It involves several effects and promises. In my test, I have to provide a stub implementation of the server, so that's a whole lot of implementation in the test, which is not what we want. But in theory, it shouldn't be so complex. Something's leaking.

And what's leaking is the implementation details of the `space/blob/add` ability. We have this whole lovely effects system for describing the flow at runtime in data in the receipt, but then we *also* couple the client implementation directly to a specific expected configuration of those effects. In other words, the *spec* describes something complex: that the client should expect certain effects and expect them to mean certain things.

If, instead, the effect handling was entirely dynamic, all of that goes away. That would seem to be the promise (as it were) of the effects system, but we haven't implemented it that way so far. That is, I'd love my test implementation of `space/blob/add`, rather than to return ([as per the spec](https://github.com/storacha/specs/blob/main/w3-blob.md#add-blob-receipt-example)):

```json
{
  "ran": "bafy..add",
  "out": {
    "ok": {
      // Result of the add is the content (location) commitment that is produced
      // as result of "bafy..accept"
      "site": {
        "ucan/await": [
          ".out.ok.site",
          { "/": "bafy...accept" }
        ]
      }
    }
  },
  // Async tasks to be scheduled.
  "fx": {
    "fork": [
      // 1. System attempts to allocate memory in user space for the blob.
      { "/": "bafy...alloc" },
      // 2. System requests user agent (or anyone really) to upload the content
      // corresponding to the blob via HTTP PUT to given location.
      { "/": "bafy...put" },
      // 3. System will attempt to accept uploaded content that matches blob
      // multihash and size.
      { "/": "bafy...accept" }
    ]
  }
}
```

to return:

```json
{
  "ran": "bafy..add",
  "out": {
    "ok": {
      "site": { "/": "bafy...location" }
    }
  }
}
```

and to have the client use that transparently, not performing any effects (because none were requested), not resolving a promise, and immediately having the location commitment link. The long effect flow becomes purely an implementation detail in the server, even though that "detail" involves, in part, asking the client to do things for it.

Having poked at it for a couple of minutes, I can see that this is frustratingly non-trivial:

1. The spec expects that this specific flow of forks happens—it's probably okay if it doesn't and the spec just describes the typical flow, but it might need some thought.
2. There's quite a bit of logic in the `space/blob/add` implementations which picks apart the effects manually to achieve this, which would be lovely to get rid of, but will certainly take effort to change.
3. This logic is generic, and really belongs in `ucanto`/`go-ucanto`, which makes it a whole separate can of worms.
4. At that point, it's pretty tied up in the move from UCAN 0.9 to 1.0.
5. Also, dealing with this kind of polymorphic-ish, union-typed data in Go is slightly infuriating. It's not obvious how to correctly model "value or promise of value", especially when the places where a promise can be used are currently not constrained by any specs I've seen. Dealing with it in `go-ipld-prime` doesn't sound super fun either.

All of which is to say, unless someone has a brilliant idea, we probably can't and shouldn't do anything along these lines right now. But as a note for future prioritization, it would make what I'm doing easier if we'd already done it.
