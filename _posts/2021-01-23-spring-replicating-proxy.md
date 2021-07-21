---
layout: post
title: "Building an RTP receiver with Spring"
categories: [Spring, RTP]
---

I have been really into understanding more about how audio/video streamings work. One piece of that world is RTP. It is defined in [RFC 3550](https://tools.ietf.org/html/rfc3550){:target="_blank"}, which I read but I don't actually understand things until I see them working for myself, so let's do that. This is what we are building:

<image src="/public/img/rtp-receiver-flow.png"></image>

1. ffmpeg generates an RTP stream from an audio file
2. The RTP receiver consumes that stream, creating a websocket stream from it and sending it to the websocket server 
3. The websocket server consumes the stream and writes it to a file

The code for this experiment can be found at [https://github.com/lucaspin/spring-replication-proxy](https://github.com/lucaspin/spring-replication-proxy){:target="_blank"}.

## RTP header structure

Here's the RTP header structure, from [the RFC](https://tools.ietf.org/html/rfc3550#page-13){:target="_blank"}:

<pre>
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|            contributing source (CSRC) identifiers             |
|                             ....                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
</pre>

A few things to note there:
- `version (V)`: the first two bytes in the packet. This is always 2, at least until someone comes up with a newer version of RTP. Food for thought: as just 2 bits are used for this, would it only be possible for a version 3 to exist?
- `contributing sources count (CC)`: works together with the `CSRC identifiers` part of the header, defining how many of those we'll have in the packet. As only 4 bits are used for this, the maximum number of CSRC identifiers we can have is 15.
- `payload type (PT)`: the format that the media in this packet uses. This only makes sense in the context of a profile. [RFC 3551](https://tools.ietf.org/html/rfc3551){:target="_blank"} defines a good example of one. The profile defines which values map to which formats. The payload type can be static or dynamic. For a static payload type, you just need to know which profile the sender is using and you're good. For a dynamic payload type, you need the profile and something else to know the format. Usually, an [SDP description](https://tools.ietf.org/html/rfc4566){:target="_blank"} is that "something else".
- `sequence number`: the sender puts sequence numbers on all packets before sending them, so that the receiver can recompose the media packets in the correct order. Later on, you'll be able to hear why that is important. 
- `timestamp`: Why would you need a sequence number and a timestamp?
- `SSRC identifier`: synchronization source; a unique id used to distinguish media streams coming from different sources.
- `CSRC identifiers`: contributing sources; a way of RTP translators to keep the original sources that were mixed into the same media stream.

The `P` (padding), `X` (extension) and `M` (marker) fields are flags to indicate changes in how the packet is structured, and are not really important for us right now. So, the RTP header can be defined as this:

```java
public class RTPPacket {
    private final int version;
    private final boolean padding;
    private final boolean extension;
    private final int contributingSourcesCount;
    private final boolean marker;
    private final int payloadType;
    private final int sequenceNumber;
    private final int timestamp;
    private final int synchronizationSourceId;
    private final byte[] payload;
}
```

## Parsing the RTP header

If you're like me and doesn't speak binary fluently, you'll need some little reminders of how to manipulate bits to grab the information you want.

For instance, we want to grab the version of the packet. We know that field takes the first two bits of the first byte of the packet.
But the first byte also has three other fields. How do we get just the version then? Using the AND and SHIFT bitwise operations.

Let's say that the first byte is `10010010`. In order to grab just the first two bits, we need to "erase" the other six bits. We can use an AND operation where the bits in the same position as the ones we want to erase are 0. Remember: ANDing anything with 0, you get 0.

<pre>
         10010010
     AND 11000000
     ------------
         10000000
</pre>

Cool, we zeroed them out, but the bits we want are still in the leftmost position. Now, we shift them right 6 times:

<pre>
         10000000
         --------
    (1x) 01000000
    (2x) 00100000
    (3x) 00010000
    (4x) 00001000
    (5x) 00000100
    (6x) 00000010
</pre>

There you go. Now, we have `00000010`, which in the fantastic decimal system means 2. With that idea in mind, let's create a `parsePacket()` method to transform an array of bytes into an `RTPPacket` object:

```java
public static RTPPacket parsePacket(byte[] packet) {
    return RTPPacket.builder()
        .version((packet[0] & 0b11000000) >>> 6)
        .padding(((packet[0] & 0b00100000) >> 5) == 1)
        .extension(((packet[0] & 0b00010000) >> 4) == 1)
        .contributingSourcesCount(packet[0] & 0b00001111)
        .marker(((packet[1] & 0b10000000) >> 7) == 1)
        .payloadType(packet[1] & 0b01111111)
        .sequenceNumber(ByteBuffer.wrap(packet, 2, 2).getShort())
        .timestamp(ByteBuffer.wrap(packet, 4, 4).getInt())
        .synchronizationSourceId(ByteBuffer.wrap(packet, 8, 4).getInt())
        .payload(Arrays.copyOfRange(packet, 12, packet.length))
        .build();
}
```

## The UDP inbound receiver

As we're only interested in understanding RTP, let's let Spring take care of the UDP complexity for us. Spring [has support for TCP and UDP](https://docs.spring.io/spring-integration/docs/current/reference/html/ip.html){:target="_blank"} and here is how you create a UDP inbound receiver on port `11111` with it:

```java
IntegrationFlow flow = IntegrationFlows.from(new UnicastReceivingChannelAdapter(11111))
    .handle(new RTPMessageHandler(rtpManager))
    .get();
```

The `RTPMessageHandler` class extends Spring's [AbstractMessageHandler](https://docs.spring.io/spring-integration/api/org/springframework/integration/handler/AbstractMessageHandler.html){:target="_blank"} and does just two things:
1. Parses the RTP packet
2. Forwards it to our `RTPManager`

```java
public class RTPMessageHandler extends AbstractMessageHandler {
    private final RTPManager rtpManager;

    @Override
    protected void handleMessageInternal(Message<?> message) {
        RTPPacket packet = parsePacket((byte[]) message.getPayload());
        rtpManager.onPacketReceived(packet);
    }
}
```

The `RTPManager` is the one responsible for a very fundamental part of an RTP receiver using UDP: packet reordering.

## Packet reordering

RTP may be used over TCP (less common) or UDP (more common). As it doesn't rely on the delivery order correctness that might be provided by the transport protocol, it needs a way to sequence packets. And that's why the RTP header has a sequence number on it.
The RTP receiver that will play the media stream needs to reorder the packets received, otherwise the media will not sound/look very nice. In our example, we won't be playing it, but the consumer of the WebSocket will, so we need to reorder the packets for them.

The `RTPManager` will be responsible for two things:
1. Reordering packets before sending them through the websocket
2. Grouping the packets that come in by SSRC id

```java
public class RTPManager {
    private final Map<Integer, SyncSourceStatus> syncSources = new HashMap<>();

    public synchronized void onPacketReceived(RTPPacket packet) {
        if (syncSources.containsKey(packet.getSynchronizationSourceId())) {
            SyncSourceStatus status = syncSources.get(packet.getSynchronizationSourceId());
            synchronized (status.getLock()) {
                status.addPacket(packet);
                if (status.getPackets().size() > MAX_PACKETS_BEFORE_FLUSHING) {
                    status.flush();
                }
            }
        } else {
            syncSources.put(packet.getSynchronizationSourceId(), SyncSourceStatus.builder()
                    .syncSourceId(packet.getSynchronizationSourceId())
                    .packets(new ArrayList<>(List.of(packet)))
                    .webSocket(initializeSocket())
                    .lock(new Object())
                    .build());
        }
    }
}
```

Here, I use another class called `SyncSourceStatus`, to help me with those things:

```java
static class SyncSourceStatus {
    private int syncSourceId;
    private List<RTPPacket> packets;
    private Socket webSocket;
    private final Object lock;

    public void addPacket(RTPPacket packet) {
        packets.add(packet);
    }

    public void flush() {
        packets = packets.stream().sorted().collect(Collectors.toList());
        packets.forEach(packet -> webSocket.send(packet.getPayload()));
        packets = new ArrayList<>();
    }
}
```

Note that the `SyncSourceStatus.flush()` sorts the packets before sending them through the websocket, so we need to make our `RTPPacket` implement `Comparable`:

```java
public class RTPPacket implements Comparable<RTPPacket> {
    // Fields and getters

    @Override
    public int compareTo(RTPPacket o) {
        return Integer.compare(getSequenceNumber(), o.getSequenceNumber());
    }
}
```

## Bringing everything up

For our WebSocket server and client, we'll use [socket.io](https://socket.io/){:target="_blank"}. Our server just grabs everything that comes from the websocket and saves it into a file:

```js
const http = require('http').Server();
const fs = require('fs');
const io = require('socket.io')(http);

io.on('connection', (socket) => {
  const wstream = fs.createWriteStream('/tmp/audio-from-ffmpeg');

  socket.on('disconnect', () => {
    wstream.end();
  });

  socket.on('message', msg => {
    wstream.write(Buffer.from(msg));
  });
});

http.listen(4010, () => {
  console.log('listening on *:4010');
});
```

We'll use [ffmpeg](https://ffmpeg.org/){:target="_blank"} to generate the RTP stream that'll be consumed by our RTP receiver:

```sh
ffmpeg \
    -re \
    -i media/pcm_s16le-44100hz-s16-10s.wav \
    -c:a copy \
    -f rtp \
    "rtp://127.0.0.1:11111"
```

Once the ffmpeg command exits, we can check the audio written to `/tmp/audio-from-ffmpeg` by the server and make sure it plays well.

## To reorder or not to reorder

Here's a media stream that was replicated with packet reordering:

<audio controls="controls">
  <source src="/public/media/reordering-yes.ogg" type="audio/ogg" />
  Your browser does not support the audio element.
</audio>

And here's the same media stream replicated without reordering the packets:

<audio controls="controls">
  <source src="/public/media/reordering-no.ogg" type="audio/ogg" />
  Your browser does not support the audio element.
</audio>

Big difference, huh?

And that's why you need the sequence number field, because the RTP receiver needs a way to put the packets that arrive in the same order in which they were sent.

And that's all.
