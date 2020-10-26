
Swarm is the start point to create your own `libp2p-rs` node.

# Basic concepts

### Swarm Main Loop
Swarm is desinged and implemented to run a main loop for handling commands received from the controler and events generated by inner transports. This is to avoid introducing the synchronous primitives like Mutex<> into Swarm. The ownership of Swarm itself is taken by the task loop and a Swarm Controller is exposed as the Swarm API interface.


### Swarm Controller
Swarm Controller is the API interface of Swarm. To manipulate Swarm, we have to send a control command to Swarm main loop via a futures::mpsc::channel. The controller is actualy the sender part of the channel.

### Swarm Transport
Swarm listens on multiple transports for incomming connections and chooses the best transport according to the dialing multiaddr to setup outgoing connections.    

### Connection and Substream

Swarm contains all established connections to remotes and manages the state of all the substreams that have been opened. The Swarm connection is multiplexed connection, which can be used to open or accept new substreams. Furthermore, a raw substream opened by connection has to be upgraded to the Swarm Substream via multistream select procedure.

Swarm manages all connection in two hash maps:

- id map: by unique connection Id
- PeerId map: by PeerId, a Peer might have multiple connections established

And each connection manages all its substreams in a Vec<>. 

### Swarm PeerStore
PeerStore is the database in Swarm, which holds：
- Address Book: mapping from PeerId to Multiaddr. Note one single PeerId might point to multiple Multiaddr.
- Key Book: mapping from PeerId to its public key.
- ...

 

### Protocol handler trait

Protocol Handler trait defines how each active connection to a remote peer should behave: how to handle incoming substreams. The substream opened in Swarm has to be specified with some protocol. In order to add protocol support, Swarm must be initialized with the protocol handlers which have to implement the trait.



### Ping & Identify

These two are built-in protocols in Swarm. 

`Ping` is used to do an application-layer health check for connections setup by transport. The successful ping result is used to measure and record the roud trip time of the corresponding peer, or the failed result leads to closing the connection to peer when failure reaching maximum retries.

The prtocol Id of `Identify` is "/ipfs/ping/1.0.0".

`Identify` is used to identify a connection by Swarm. In general, Swarm would start the `Identify` service as soon as a new connection is established. The remote peer should report its identification response as responding to the identification request. identification message is encoded in protobuf and contains some basic information including the public key, supported protocols and so on.

The prtocol Id of `Identify` is "/ipfs/id/1.0.0".

### Limitations

#### Swarm Dialer, to be done

Swarm should make outgoing connections by dialing the multiaddr of the peer specified by PeerId. There might be multiple multi-address for a PeerId, so that Swarm should dial multi addresses at the same time. Besides, the dialing procedure should be done asynchronously to avoid blocking Swarm main task loop. All these require a Dialer component should be implemented considerately for Swarm. Swarm Dialer should be able to manage asynchronous dialing tasks, check the connection limit and properly handle the dialing results.  

#### Event Bus

Swarm should have an event bus for publish/subscribe connection or sub-stream events generated internally. Thus applications could receive the notifications that they are interested in and possiblely they could react to.

#### Statistics

There are some statistics implemented, but not sufficient as we know.




## Getting started

In general, to use libp2p-rs, you would always create a Swarm object to access the low-level network:

Creating and initializing a swarm:

```no_run
    let mut swarm = Swarm::new(keys.public())
        .with_transport(Box::new(t1))
        .with_transport(Box::new(t2))
        .with_protocol(Box::new(DummyProtocolHandler::new()))
        .with_protocol(Box::new(MyProtocolHandler))
        .with_ping(PingConfig::new().with_unsolicited(true).with_interval(Duration::from_secs(1)))
        .with_identify(IdentifyConfig::new(false));
```

The code above creates a swarm and initialize it with two transports and a few protocol handlers. Note Ping and Identify are built-in protocols that Swarm supports internally.

Starting the swarm:

```no_run
    let mut control = swarm.control();
    swarm
        .peers
        .addrs
        .add_addr(&remote_peer_id, "/ip4/127.0.0.1/tcp/8086".parse().unwrap(), Duration::default());

    swarm.start();
```

The code grabs a swarm controller and then starts the swarm main task loop.

```no_run
    task::block_on(async move {
        control.new_connection(remote_peer_id.clone()).await.unwrap();
        let mut stream = control.new_stream(remote_peer_id, vec![b"/my/1.0.0"]).await.unwrap();

        log::info!("stream {:?} opened, writing something...", stream);
        let _ = stream.write_all2(b"hello").await;
        stream.close2().await;

        log::info!("shutdown is completed");
    });
```

The code is using async-std to spawn a task to handle connection and substream. Note the substream is specified with protocol id "/my/1.0.0", which means the remote peer has to support this protocol so that a substream can be opened by both peers.