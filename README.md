# Simple Device Drawing Protocol

#### Current version: `3`

## Terminology & Conventions

* *vendor*: any implementation (device) which has a display it exposes via the SDDP
* *consumer*: any interface for connecting with and driving an SDDP vendor's display
* *prefix*: a prefix appended to all channel names

Hereafter, anything in `monospaced` font has special meaning (matching [`docopt`](http://docopt.org/) where appropriate/possible):
* `literal string`
* `<SymbolicName>`
* `(optional|option1|...|optionN)`
* `[required|option1|...|optionN]`

Any vendor & consumer that have been configured with the same prefix will be able to communicate with each other. In this way, different SDDP deployments can be logically separated on the same physical bus.

Hereafter, this prefix will be referred to symbolically as `<Prefix>`, and concretely we'll use `sddpExample`.

## Theory of Operation

A request/response model, wherein once the connection has been established and the bulk of communication is one-way (consumer-to-vendor), the responses consist of simple ACKnowledgment messages (currently used to facilitate connection monitoring & reconnection logic).

## Details

### Connection initialization handshake

#### Consumer request:
| | |
|-|-|
| Channel |  `<Prefix>:ctrl-init` |
| Message Form | `<ConsumerId> request <DisplayID> <ProtocolVersion>` |

#### Vendor's response:
| | |
|-|-|
| Channel | `<Prefix>:ctrl-init:request:resp` |
| Message Form | `<DisplayID> [ok\|reject] <ConsumerId> (RejectionReason)` |

A vendor must only publish its response message once it has initialized the established channel & is ready to recieve message on it.

Upon successful handshake, both sides must be listening on a newly-established, unique channel for further communication. This establishment channel will be named:

```
<Prefx>:estab:<DisplayId>|<ConsumerId>
```

Finally, the consumer must listen on and the vendor must publish message acknowledgement to channel:

```
<Prefx>:estab:<DisplayId>|<ConsumerId>:ack
```

(the establishment channel name postfixed with "`:ack`")

#### Example

A consumer - here using the ID `sddp.ConsumerA` - who wishes to establish a connection with a display vending with the ID `sddp.Display1` would publish the following on channel `sddpExample:ctrl-init`:

```
sddp.ConsumerA request sddp.Display1 3
```

If the display identifying as `sddp.Display1` is accepting of the request, it will respond with the following on channel `sddpExample:ctrl-init:request:resp`:

```
sddp.Display1 ok sddp.ConsumerA
```

If however the display rejects the connection request for any reason, it will respond accordingly on `sddpExample:ctrl-init:request:resp`, in this case rejecting for a protocol version mistmatch:

```
sddp.Display1 reject sddp.ConsumerA bad_protocol_version
```

Assuming an `ok` response from `sddp.Display1`, both sides would begin listening on and `sddp.ConsumerA` publishing draw commands to:

```
sddpExample:estab:sddp.Display1|sddp.ConsumerA
```

while `sddp.Display1` would be required to publish its acknowledgments to `sddpExample:estab:sddp.Display1|sddp.ConsumerA:ack`.

### Established connection

Once a connection is established between vendor & consumer, the vendor needn't send any further message save for the required acknowledgments.

Acknowledgements must be received in monotonic order & a timely manner for the connection to be maintained, otherwise the consumer is expected to end the current connection and reattempt.

#### General message form

From consumer to vendor:

```
<SequenceNumber> <DrawCommand> (SpaceDelimitedArgumentList)
```

and the acknowledgment is expected to simply consist of the sequence number being acknowledged, nothing else.

### Draw commands

#### writeat

Draws `[string]` (may contain spaces) on the connected display at `[column]` and `[row]`.

```
writeat [column] [row] [string]
```

#### clear

Clears the display.

```
clear
```

#### toggleDisplay

Toggles the display.

```
toggleDisplay [on|off]
```

#### toggleCursor

Toggles the cursor.

```
toggleCursor [on|off]
```

#### toggleCursorBlink

Toggles the cursor blink.

```
toggleCursorBlink [on|off]
```

### Full example transcript

Demonstrating the entirety  of traffic sent when running the [hello-world](x) example.

```
$ redis-cli --csv -h <redacted> 'psubscribe' 'sddpExample*'
"pmessage","sddpExample*","sddpExample:ctrl-init","sddp.ConsumerA request sddp.Display1 3"
"pmessage","sddpExample*","sddpExample:ctrl-init:request:resp","sddp.Display1 ok sddp.ConsumerA"
"pmessage","sddpExample*","sddpExample:estab:sddp.Display1|sddp.ConsumerA","1 clear"
"pmessage","sddpExample*","sddpExample:estab:sddp.Display1|sddp.ConsumerA","2 writeat 10 1 Hello"
"pmessage","sddpExample*","sddpExample:estab:sddp.Display1|sddp.ConsumerA","3 writeat 11 2 World!"
"pmessage","sddpExample*","sddpExample:estab:sddp.Display1|sddp.ConsumerA:ack","1"
"pmessage","sddpExample*","sddpExample:estab:sddp.Display1|sddp.ConsumerA:ack","2"
"pmessage","sddpExample*","sddpExample:estab:sddp.Display1|sddp.ConsumerA:ack","3"
```

## Change History

| Version | Release Date | Change Highlights
|-|-|-|
| 3 | May 3 2021 | First public release |
| 2 | Not released ||
| 1 | Not released ||

## `TODO`

[ ] Optional capability negotiation

## Notes, caveats & errata

As of this version, the transport layer is *assumed* to be Redis pub/sub and accordingly the discussion below may use concepts and/or language specific to this transport layer.

### Authors

* Ryan Joseph, [Electric Sheep Co.][8]

### License

[MIT](LICENSE)