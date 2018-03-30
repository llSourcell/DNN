# IPFS Module Guide

This document is just a quick guide for if you want to implement IPFS -- its modelled after go-ipfs, and this serves as a template for both js-ipfs and py-ipfs.

Sections:
- IPFS Types
- API Commands
- API Transports
- Implementing bindings for the HTTP API

## The Libraries

There are lots of non-IPFS related modules that have been buiult for IPFS that acts as dependencies. 

### libp2p

There are some nontrivial peer to peer protocols necessary for IPFS. These have been abstracted and placed into an entirely seperate module called libp2p. This is a very thin wrapper around a lot of modules that interact with each other. 

Implementations:
- [go-libp2p](https://github.com/libp2p/go-libp2p)
- [js-libp2p](https://github.com/libp2p/js-libp2p)

`libp2p` could very well be _the bulk_ of an ipfs implementation. The rest is very simple.

### The Multi libraries. Things to remember

There are quite a number of self-describing protocols/formats in use all over ipfs.

- [multiaddr](https://github.com/multiformats/multiaddr)
- [multicodec](https://github.com/multiformats/multicodec)
- [multihash](https://github.com/multiformats/multihash)
- [multistream](https://github.com/multiformats/multistream)

## Core Pieces

### IPLD

IPLD is the format for IPFS objects, but it can be used outside of ipfs (hence a module). It's layered on top of `multihash` and `multicodec`, and provides the heart of ipfs: the merkledag.

Implementations:
- [go-ipld](https://github.com/ipfs/go-ipld)
- [js-ipld](https://github.com/ipld/js-ipld-dag-cbor)

### IPRS

IPRS is the record system for IPFS, but it can be used outside of ipfs (hence a module). This deals with p2p system records -- it is also used by `libp2p`.

Implementations:
- [go-iprs](https://github.com/ipfs/go-iprs)
- js-iprs _Forthcoming_

### IPNS

IPNS provides name resolution on top of IPRS -- and a choice of record routing system.

### IPFS-Repo

The IPFS-Repo is an IPFS Node's "local storage" or "database", though the storage may not be in a database nor local at all (e.g. `s3-repo`). There are common formats so that multiple implementations can read and write to the same repos. Though today we only have one repo format, more are easy to add so that we can create IPFS nodes on top of other storage solutions.

Implementations:
- [go-ipfs-repo](https://github.com/ipfs/go-ipfs/tree/master/repo)
- [js-ipfs-repo](https://github.com/ipfs/js-ipfs-repo)

## IPFS Core

The Core of IPFS is an interface of functions layered over all the other pieces.

### IPFS Node

The IPFS Node is an entity that bundles all the other pieces together, and implements the interface (described below). In its most basic sense, an IPFS node is really just:

```go
type ipfs.Node struct {

    Config      // has a configuration
    repo.Repo   // has a Repo for storing all the local data
    libp2p.Node // has an embedded libp2p.Node, and thus a peer.ID, and keys
    dag.Store   // has a DAG Store (over the repo + network)

}
```

IPFS itself is very, very simple. The complexity lies within `libp2p.Node` and how the different IPFS commands should run depending on the `libp2p.Node` configuration.

### IPFS Node Config

IPFS Nodes can be configured. The basic configuration format is a JSON file, and so naturally converters to other formats can be made. Eventually, the configuration will be an ipfs object itself.

The config is stored in the IPFS Repo, but is separate because some implementations may give it knowledge of other packages (like routing, http, etc).

### IPFS Interface or API

The IPFS Interface or API (not to be confused with the IPFS HTTP API) is the set of functions that IPFS Nodes must support. These are classified into sections, like _node, network, data, util_ etc.

The IPFS Interface can be implemented:
- as a library - first and foremost
- as a commandline toolchain, so users can use it directly
- as RPC API, so that other programs could use it
 - over HTTP (the IPFS HTTP API)
 - over unix domain sockets
 - over IPC

One goal for the core interface libraries is to produce an interface that could operate on a local or a remote node. This means that, for example:

```go
func Cat(n ipfs.Node, p ipfs.Path) io.Reader { ... }
```
should be able to work whether `n` represents a local node (in-process, local storage), or a remote node (over an RPC API, say HTTP).

_**For now, i list these from the commandline, but the goal is to produce a proper typed function interface/API that we can all agree on.**_

#### Node Commands

These are the for the node itself.

- ipfs stats
- ipfs diag
- ipfs init
- ipfs config
- ipfs repo
- ipfs repo gc

#### Data Commands

- ipfs pin
- ipfs files
- ipfs tar
- ipfs resolve
- ipfs block
- ipfs object
- ipfs {cat, ls, refs}

#### Network Commands

These are carried over from libp2p, so ideally the libp2p implementations do the heavy lifting here.

- ipfs exchange
- ipfs routing
- ipfs bitswap
- ipfs bootstrap
- ipfs id
- ipfs ping
- ipfs swarm

#### Naming commands

These are carried over from IPNS (can make that its own tool/lib).

- ipfs name
- ipfs dns

#### Tool Commands

- ipfs log
- ipfs update
- ipfs daemon
- ipfs version
- ipfs tour

## IPFS Datastructures and Data Handling

There are many useful datastructures on top of IPFS. Things like `unixfs`, `tar`, `keychain`, etc. And there are a number of ways of importing data -- whether posix files or not.

### IPLD Data Importing

Importing data into IPFS can be done in a variety of ways. These are use-case specific, produce different datastructures, produce different graph topologies, and so on. These are not _strictly_ needed in an IPFS implementation, but definitely make it more useful. They are really tools on top of IPLD though, so these can be generic and separate from IPFS itself.

- graph topologies - shape of the graphs
  - balanced - dumb, dead simple
  - trickledag - optimized for seeking
  - live stream
  - database indices
- file chunking - how to split a continuous stream/file
  - fixed size
  - rabin fingerprinting
  - format chunking (use knowledge of formats, e.g. audio, video, etc)
- special format datastructures
  - tar
  - document formats - pdf, doc, etc
  - audio and video formats - ogg, mpeg, etc
  - container and vm images
  - and many more


