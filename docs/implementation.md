# Implementation Guides

This guide is for people who build MineChat clients and MineChat servers. It collects practical advice that does not belong in the specification.

The specification defines what the protocol does. This guide explains how to write code that does it well. Nothing on these pages is required. A single-threaded server that ignores all of it is still a valid MineChat server.

Read [the specification](specification.md) first if you have not. This guide assumes you already know the packet names and the field names.

## Framing in practice

The specification asks you to check the declared sizes before you allocate memory. This is the most important advice on this page, because it is what stops a small packet from exhausting your memory.

A few habits make this easier:

- Check both the compressed size and the decompressed size before you allocate a buffer. The decompressed size is the dangerous one, because that is how much memory you need once the payload expands.
- The default limit is 1 MiB. Lower it on small devices such as phones. Most chat messages use a fraction of that.
- Reject a size of zero or less immediately. It is never valid.
- Compare the real decompressed length with the length the sender declared. A difference means the sender is broken, or is attacking you.
- Flush after every frame. Without a flush, a peer that has gone away can sit unnoticed in a buffer for a long time.
- Read the frame header one byte at a time. This lets you tell a clean disconnect from a cut-off frame. A clean disconnect is normal and needs no action. A cut-off frame is an error.

## Compression in practice

The protocol always uses zstd. There is no negotiation, so both sides can simply assume it.

Keep compression behind a small interface of your own. There are two good reasons:

- You can swap in a pass-through handler for tests, where compression only makes things slower.
- Some languages need a native library for zstd. An interface lets you keep that dependency in one place.

## Reading packets

The specification says CBOR map keys carry meaning, not order. Never write code that depends on the order keys arrive in. A quick way to check: encode a packet with its keys in a different order and confirm your decoder still reads it.

Ignore unknown keys and unknown packet types. This is what lets a newer client talk to an older server. When you meet an unknown key, skip it and carry on reading the rest of the map.

Reject missing or wrong-typed required fields. Do not fill them in with empty strings or zeroes. Defaults turn a broken packet into a plausible-looking packet, and you will spend a long time hunting for the real bug. Include the packet name and the field name in the error message, and debugging becomes much faster.

Keep protocol errors separate from errors in your own code. A packet that violates the specification is a protocol error. A handler that throws because of your own bug is not, and the two deserve different log messages.

## Session stages

The specification defines four stages in §7.4, and lists which packets each stage accepts in §7.5.

In code, most implementations keep the current stage on the connection object. Move to the next stage when the server accepts a packet, not when the packet merely arrives. A client asking to link is not the same thing as the server agreeing to link.

When a packet arrives at the wrong stage, log it and close the connection. Put the stage and the packet name in the log line. This is one of the most common reasons a client fails to connect, and one of the hardest to diagnose from the client side.

## Keep-alive in practice

The specification suggests a 15-second interval and a 30-second timeout.

- Use a monotonic clock to measure elapsed time. Monotonic clocks do not jump when the system clock is corrected. If you use the wall clock, a clock adjustment can disconnect every client at once.
- Treat every received packet as proof of life, not only `PONG`. When a client sends a chat message, it has already proved the connection works.
- Measure round-trip time with your own clock. Do not subtract the peer's timestamp from yours. The two clocks do not agree, so the result is wrong, sometimes wildly wrong.
- Let your application own the scheduler that sends `PING`. If a library creates a timer thread of its own, that thread becomes your responsibility to shut down, and it is easy to forget.

## TLS in practice

The first connection during linking is the awkward one, because the client has nothing to compare against yet.

A workable approach:

1. On the first connection, accept whatever certificate the server presents.
2. Compute a SHA-256 digest of it. Either the whole certificate, or the public key inside it.
3. Store that digest alongside the account details.
4. On every later connection, compute the digest again and compare it with the stored value. Close the connection if they differ.

Prefer pinning the public key over the whole certificate. It costs the client no more work, but it lets a server replace or renew its certificate without breaking every linked device.

The default port is 7632. Make it configurable anyway. Players often run behind a proxy, or on shared hosting where the port is already taken.

Hostname verification and pinning are separate checks. The specification leaves hostname handling to you. If you decide not to verify hostnames, then the pin is the only thing protecting the connection, so do not skip it.

## Enforcing moderation

A protocol library reports moderation events. It does not enforce them. Enforcement belongs to the server, and it reaches further than one connection.

- **Warn.** Record it. There is nothing to enforce.
- **Mute.** Stop distributing messages from that client. Check before you broadcast, not after.
- **Kick.** Send `MODERATION`, then close the connection. The client must not be able to reconnect straight away.
- **Ban.** Record the ban, and check it at the start of every session, including the `LINK` step. A ban that is not checked at `LINK` time does not survive a restart.

Scope decides how widely an action applies. A client-scoped action affects one device. An account-scoped action affects every device linked to that Minecraft account, including devices that are not connected at the moment. This matters when you apply an account-scoped ban: closing the one open connection is not enough.

## Threading

This section only matters if your implementation sends from more than one thread. A single-threaded implementation can skip it.

- Serialize whole frames when several threads can send. If two threads write at the same time, the bytes can interleave and corrupt the stream. This is a hard bug to find, because it only shows up under load.
- Reads may block on their own thread. There is no need to synchronize reads against writes.
- Closing should be safe to call more than once, and safe to call from a different thread than the one reading.

## Handling untrusted content

The `content` field is a string that arrived over the network. Treat it as untrusted input.

- Parse it with limits. Cap the input size, and cap how deeply structures may nest. A deeply nested component tree is a common way to crash a parser.
- You do not have to understand the content. If you only relay messages, pass it through as an opaque string.
- The 1 MiB frame limit applies to the whole packet, not only to this field.

## Where to go next

- §3 for TLS and certificate pinning
- §4 for the framing layer
- §6 for the CBOR rules
- §7.4 for session stages
- §8 for each packet in turn
- §9 for keep-alive
