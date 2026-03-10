# 2 DoS vulnerabilities on Floresta

I just discovered and reported 2 DoS vulnerabilities that affect the Floresta software. 
The first one is regarding a large memory allocation that is done when allocating the header from a V1 message. I noticed that when the V2 (BIP324) handshake fails, it falls back to the V1. 
However, when deserializing the header, there was no size check, so a "huge" header would cause the data to be resized in a way that could cause it to run out of memory. See:

```rs
let header: V1MessageHeader = deserialize_partial(&data)?.0;
data.resize(24 + header.length as usize, 0);
reader.read_exact(&mut data[24..]).await?;
```

The other vulnerability is regarding the rate-limiting mechanism implemented in Floresta. It checks if a peer is sending more than 10 messages per second; if so, the peer is disconnected. However, the counter (`messages`) was never incremented, so it was always zero, and the mechanism had no effect.

```rs
// divide the number of messages by the number of seconds we've been connected,
// if it's more than 10 msg/sec, this peer is sending us too many messages, and we should
// disconnect.
let msg_sec = self
    .messages
    .checked_div(Instant::now().duration_since(self.start_time).as_secs())
    .unwrap_or(0);

if msg_sec > 10 {
    error!(
        "Peer {} is sending us too many messages, disconnecting",
        self.id
    );
    return Err(PeerError::TooManyMessages);    
```

-------

Both vulnerabilities are fixed in https://github.com/getfloresta/Floresta/pull/880.
