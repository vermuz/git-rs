# git-rs

Implementing git in rust for fun and education!

This is actually my second stab at it, so big blocks will land in place from my
first attempt. I'm trying again this year after reading more of "Programming
Rust" (Blandy, Orendorff).

## TODO

- [x] Read objects from loose store
- [x] Read objects from pack store
    - [x] Read packfile indexes
    - [x] Read delta'd objects
    - [x] Fix interface so we don't need to run `open` for each `read()`
    - [x] **BUG**: certain OFS deltas are misapplied.
        - [x] Isolate the error case
        - [x] Fix it
- [x] Load refs off of disk
- [x] Parse git signatures ("Identity"'s)
- [x] Create iterator for walking commit graph
- [x] Create iterator for walking trees
    - [ ] Materialize trees to disk (post gitindex?)
- [ ] Create index from packfile
    - [ ] Rename `Storage` trait to `Queryable`
    - [ ] Rework object loading API from `<Type + Boxed reader>` to "we take a writable object"
- [ ] Load packed-refs
- [ ] Create interface for writing new objects
- [ ] `.git/index` support
    - [ ] Read git index cache
    - [ ] Write git index cache
- [ ] Add benchmarks
- [ ] Create packfile from list of objects (API TKTK)
- [ ] Network protocol
    - [ ] receive-pack
    - [ ] send-pack
- [ ] Try publishing to crates
    - [ ] Write documentation
    - [ ] Use crate in another project

* * *

## PLAN

### 2019-01-23 Update

- It's time to start indexing packfiles.
    - This'll let us start talking to external servers and cloning things!
- However, it's kind of a pain.
    - Packfiles (viewed as a store) aren't hugely useful until you have an index, so
      I had designed them as an object that takes an optional index from outside.
        - My thinking was that if an index was not given, we would build one in-memory.
    - That just blew up in my face, a little bit. :boom:
    - In order to build an index from a packfile you have to iterate over all of the objects.
        - For each object, you want to record the offset and the SHA1 id of the object at that offset.
        - **However**, the object might be an offset or a reference delta.
            - That means that in order to index a packfile, you've got to be able
              to read delta'd objects at offsets within the packfile (implying you already
              have the `Packfile` instance created) and outside of the packfile (
              implying you have a `StorageSet`.)
            - In other words: my assumptions about the program design are wrong.
    - So, in the next day or so I'll be reversing course.
        - It should be possible to produce a `Packfile` as a non-store object _and_
          iterate over it.
        - The "store" form of a packfile should be the combination of a `Packfile`
          and an `Index` (a `PackfileStore`.)
            - This means I'll be splitting the logic of `src/stores/mmap_pack` into
              "sequential packfile reads" and "random access packfile reads (with an index.)"
- It's fun to be wrong :tada:

* * *

### 2019-01-21 Update

- Well, that was a fun bug. Let's walk through it, shall we?
    - This _occasionally_ showed up when a delta would decode another delta'd object.
        - I found a hash that would reliably fail to load.
        - We'd fail the read because the incoming base object would not match the 2nd
          delta's "base size". [Here][ref_10].
        - Removing the check to see if I got the deltas wrong would cause the thread to
          panic -- the delta's base size _wasn't a lie_.
    - First, I switched back to my old mmap-less packfile implementation, because I recently
      touched that code. "Revert the thing you touched last" is a winning strategy in these
      cases: doesn't cost expensive thinking, quickly puts bounds around the bug.
        - Alas, the old packfile implementation _also_ had this bug. **No dice.**
    - I compared the output of this git implementation to my [JS implementation][ref_11].
        - I confirmed that the output of the JS implementation worked by comparing its output
          for the hash of concern to vanilla git.
        - After confirming that, I logged out the offsets being read from the file and the expected
          sizes. I compared this to similar debugging output I added to `git-rs`.
            - The offsets are the bound values sent into [the packfile][ref_12]. For the outermost
              read ("Give me the object represented by `eff4c6de1bd15cb0d28581e4da17035dbf9e8596`"),
              the offsets come from the packfile index.
            - For `OFS_DELTA` ("offset delta") types, the start offset is obtained by reading a [varint][ref_13]
              from the packfile and subtracting it from the current start offset.
        - The offsets and expected sizes matched!
            - This meant that:
                1. I was reading the correct data from the packfile
                2. I was reading varints correctly
                3. The bug must be in the application of the delta to a base object
        - From there I added logging above [these state transitions][ref_14], noting the particulars of the
          operation.
            - I added the same logging to the JS implementation, and found that (aside from
              totally bailing out ahead of the 2nd delta application) the commands were the same.
            - So it wasn't even that my delta code was wrong: it was my `Read` state machine.
    - At this point, I was like: "This is a read state machine bug. [I know this][ref_15]."
        - So, one of the things this state machine does is carefully bail out if it
          can't [write all of the bytes for a command][ref_16]. ("If there remains an `extent` to write,
          record the next state and break the loop.")
        - However, at this point we've already consumed the last command. There are no more instructions.
        - So if this function were to be called again, ...
            - We would politely (but firmly) [tell the caller][ref_17] to [buzz off][ref_18] (`written == 0`, here.).
    - [The fix][ref_19] turned out to be simple, as these fixes usually are.
        - (I need to write a test for this, I know. I know. Pile your shame on me.)
- So [what did we learn][ref_20]?
    - Always test your state machines, folks.
    - (I've said it once, and I'm saying it again.) Malleable reference implementations will save your bacon.
        - Make sure you can trust your reference implementation.
- Anyway. The tree reader works now! :evergreen_tree::deciduous_tree::evergreen_tree:

* * *

### 2019-01-19 Update

- It's slightly faster! :tada:
    - mmap sped things along nicely, shaving 20ms off of our runtime.
    - We're still reliably _slower_ than git, though. It might be because we load the refset
      immediately.
    - I kept the immutable "file per read" packfile store around; I think it may come in handy
      in the future.
    - It would be excellent to capture this as a benchmark instead of running it ad-hoc.
- I integrated the tree walking iterator and got a nice surprise:
    - There's a bug in my OFS delta code!
    - This is interesting, because it only appears for certain blobs in certain repositories.
        - Otherwise other OFS deltas seem to resolve cleanly.
        - Case in point: many of the commits I load as a test of the commit walk-er are OFS-delta'd.
    - Also of note: I've split from `src/bin.rs` into dedicated binaries for tree walking and commit walking.
- Today's theme: isolate the bug in a test case.
    - **EOD Update**: It's really helpful to have a reference implementation.
    - I've confirmed that the reference implementation _can read_ the object that breaks this project.
        - We are reading the same offsets, as well (phew)
    - I've further confirmed that swapping out the packfile implementation for the older, slower packfile
      doesn't affect anything.
    - *I suspect* this means there's either a problem in my delta code (highly possible!), my varint decoding
      code (*very* possible), or the Read implementation for Deltas. Yay, narrowed down results!

* * *

### 2019-01-15 Update

- I added an (experimental) `git_rs::walk::tree` iterator to take a Tree and yield
  a path + a blob for each item.
    - It's probably slower than it should be: for each item it has to clone a `PathBuf`, because I couldn't work out the lifetimes.
    - **If you know how to fix that**, please [open an issue][ref_8] and let me know :revolving_hearts:
- I took some time to clean up the warnings during builds.
    - Oh! I also installed [Clippy][ref_9] which warns about higher level antipatterns in Rust!
- I'm still noodling over the **2-3x** slowdown between vanilla git and Our Git.
    - I think I might create two packfile interfaces -- one "generic" and one "mmap"'d, to see if
      one or the other makes up the difference in performance.
        - This also has the virtue of being `unsafe` code, which is something I have not yet used
          in Rust!

* * *

### 2019-01-06 Update

- I wrote an iterator for commits! The [first cut][ref_6] kept a `Vec` of `(Id, Commit)` around,
  so we could always pick the most recent "next" commit out of the graph (since commits may have
  many parents.)
    - But in finishing up the collections section of "Programming Rust" I noticed that `BinaryHeap`
      was available, which keeps entries in sorted order. You don't often get to choose the underlying
      storage mechanism of your collections in JS, so this hadn't occurred to me!
    - Anyway. I swapped out the `Vec` for a `BinaryHeap` [in this commit][ref_7]. Because this pushes
      the ordering into an `Ord` impl for a type, this opens up the possibility of using the one iterator
      definition for multiple different orderings. Neat!
- Testing against a couple long-lived repo, the results coming out of `git_rs` are exactly the same as
  `git`!
    - However, it takes about **twice** the time: **60ms** for `git_rs` where `git` takes **30ms**.
    - I think I have a lead on this, and it has to do with packfile stores: each read from a packfile
      opens a new `File` instance.
- I've added a **TODO** section to keep track of what comes next!

* * *

### 2019-01-02 Update

- I implemented [ref loading][ref_2]. It was a bit of a pain! Translating to and
  from `Path` types took a bit of doing.
- I've been trying to read up on Rust idioms -- I found a couple of resources:
    - [The Rust API Guidelines][ref_3] doc has been _very_ helpful.
    - **@mre**'s [idiomatic rust repo][ref_4] collects many interesting links.
    - I've also been idly checking out [videos from RustConf 2018][ref_5]
- As a result, I've implemented `FromStr` for `Id`, (hopefully) giving it a
  more idiomatic API -- `let id: Id = str.parse()?`

* * *

### 2018-12-27 Update

- Rust is feeling more natural. [This chain][ref_0] felt natural to write. I
  was even able to [cross-index a list][ref_1] with only a minimum of fighting
  the borrow checker.
- I split the objects interface into Type + boxed read with a method for reifying
  the data into an Object. This feels good! It lets folks check to see, for example,
  if they're referring to a Blob without having to load the entire Blob into memory.
- The closure interface for the loose interface works pretty well, but pack interfaces
  need to be able to ask the overarching set of stores for a reference due to REF_DELTA
  objects. This is a bummer, because it really quickly turns into "fighting the borrow
  checker." Right now I think the way forward is to build a StorageSet that holds a Vec
  of heterogenous `Box<Storage>` objects, where `Storage` is a new trait that specifies
  `get(&Id, &StorageSet)`.
    - A sidenote re: the loose store: it feels kind of odd to have to produce a
      `git_rs::Error` instead of a `std::io::Error`. Room for improvement!
- Oh! It was pretty easy to add a binary to this lib crate. And now we can `git log`
  other repos!

* * *

### 2018-12-21 Update

- Decided to focus on moving a bit slower and making sure I have tests for
  primitives this time around.
- Moved away from my original `Box<Write>` trait object design for object
  instance reading & storage format in favor of generics.

* * *

[ref_0]: https://github.com/chrisdickinson/git-rs/blob/fdbe4ac7c781a5c085777baafbd15655be2eca0b/src/objects/commit.rs#L20-L30
[ref_1]: https://github.com/chrisdickinson/git-rs/blob/fdbe4ac7c781a5c085777baafbd15655be2eca0b/src/packindex.rs#L116-L126
[ref_2]: https://github.com/chrisdickinson/git-rs/commit/6157317fb18acac0633c624e9831282a950b4db0
[ref_3]: https://rust-lang-nursery.github.io/api-guidelines/
[ref_4]: https://github.com/mre/idiomatic-rust
[ref_5]: https://www.youtube.com/playlist?list=PL85XCvVPmGQi3tivxDDF1hrT9qr5hdMBZ
[ref_6]: https://github.com/chrisdickinson/git-rs/blob/254d97e3d840eded4e5ff5a06b9414ff9396e976/src/walk/commits.rs#L56-L71
[ref_7]: https://github.com/chrisdickinson/git-rs/commit/f8f4cf5f1430b14d3ef0b298ffa9f2cd880d5c28/src/walk/commits.rs#L40
[ref_8]: https://github.com/chrisdickinson/git-rs/issues/new?title=Here%27s%20how%20to%20remove%20the%20clone()%20from%20walk::tree
[ref_9]: https://github.com/rust-lang/rust-clippy
[ref_10]: https://github.com/chrisdickinson/git-rs/blob/12afab5462f67c8670671177b4053aa566b45338/src/delta.rs#L45-L47
[ref_11]: https://github.com/chrisdickinson/git-odb-pack
[ref_12]: https://github.com/chrisdickinson/git-rs/blob/eff4c6de1bd15cb0d28581e4da17035dbf9e8596/src/stores/mmap_pack.rs#L41
[ref_13]: https://developers.google.com/protocol-buffers/docs/encoding#varints
[ref_14]: https://github.com/chrisdickinson/git-rs/blob/eff4c6de1bd15cb0d28581e4da17035dbf9e8596/src/delta.rs#L133-L140
[ref_15]: https://66.media.tumblr.com/cd7765c4bfbe7d124ad2b7bf344b9588/tumblr_p902faD2n61wzvt9qo1_500.gif
[ref_16]: https://github.com/chrisdickinson/git-rs/blob/eff4c6de1bd15cb0d28581e4da17035dbf9e8596/src/delta.rs#L154-L161
[ref_17]: https://github.com/chrisdickinson/git-rs/blob/12afab5462f67c8670671177b4053aa566b45338/src/delta.rs#L93
[ref_18]: https://github.com/chrisdickinson/git-rs/blob/12afab5462f67c8670671177b4053aa566b45338/src/delta.rs#L187
[ref_19]: https://github.com/chrisdickinson/git-rs/commit/1442539cd01b7140ff2f58bc5df39e4686bd6843
[ref_20]: https://thumbs.gfycat.com/PiercingSnarlingArabianhorse-size_restricted.gif
