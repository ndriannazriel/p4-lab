# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2299**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

### The tunnel loop
`sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`
The entry for tunnel exit 3 (key 0003) had handle 2 and sent packets out of port 3, to h3. I changed it so it sends them out of port 1.

### The logs
From the `assignment\evidence\loop-before.txt` file, by counting how many packets each switch processed after sending this command:
`h1 python3 send.py 10.99.0.3 "loop" --tunnel 3`

s1 had handled 1422 and s2 1421. Five seconds later, it was 1548 and 1547, about 126 more each eventhough i sent only 1 packet. Because of the loop, it kept rising until I stopped mininet.

### Does Ipv4 TTL stop an IPv4 loop in this program?
No. The ipv4_forward action does hdr.ipv4.ttl = hdr.ipv4.ttl - 1, but nothing ever checks whether the TTL has reached 0. Therefore, an IPv4 loop will also continue forver until I stop the mininet.

## 2. Design: the hop limit

### The header
An 8 bit field hop_limit positioned to the end of the tunnel header. With an 8 bit, it allows upto 255 hops which is way more than enough for just 2 switches following the same design as the TTL for the IPv4 header. The reason as to why it is positioned at the end after the proto_id and dst_id is because mainly for backward compatibility. An old switch that has an old program can still read it correctly. This is talked about more in the Evaluation section.

### The starting value
The value uses a default of 8 but can be changed via the command --hops. The value has to be at least 2 because anything less than 2 which is less than the number of switches/longest path will drop normal traffic. With a limit of 8 hops, this also enables stopping loops from going on forever.

### Where decrementation happens
The check happens at the Ingress section of the program so that a packet that should die is dropped before any switch resources is used. If hop_limit is 0, the switch drops the packet. Otherwise it subtracts 1 and applies myTunnel_exact as usual. A check needs to go before decrementing because bit<8> if decrements 1 from 0 will just give it another 255 hops.

### What else needed change
In tunnel.p4, the parser and deparser didn’t need new code. packet.extract(hdr.myTunnel) and packet.emit(hdr.myTunnel) work on the whole header, so they automatically read and write the extra byte. The hosts, however, had to match the switches exactly. In both send.py and receive.py, I added ByteField("hop_limit", 8) as the last field of the tunnel header, in the same order as in P4.


## 3. Implementation

### tunnel.p4
Added bit<8> hop_limit as the last field of myTunnel_t. In the ingress apply block, the tunnel branch now first checks whether hop_limit is 0: if so, it calls drop(). Otherwise it subtracts 1 and then applies myTunnel_exact as before. The IPv4 branch is unchanged.

Checking a packet before decrementing is important so that a dead packet does not come back alive.

### send.py & receive.py
Define the tunnel header with the same 3 fields so it matches.

## 4. Evidence: it works
From `assignment\evidence\delivery.txt`, shows both tunnelled packets arrived: h2 received 'c4 to h2' with dst_id=2, hop_limit=6, and h3 received 'c4 to h3' with dst_id=3, hop_limit=6. Plain IPv4 still works (delivery.txt). h1 ping -c 2 h3 succeeded with [0% packet loss] and replies at TTL [62]. The IPv4 path is unaffected by the change.

The loops now ends as the number of packet counts stop rising after 5 seconds as seen in `assigment\evidence\loop-after.txt` so it does match my design because each switch subtracts 1: s1 8→7, s2 7→6, s1 6→5, s2 5→4, s1 4→3, s2 3→2, s1 2→1, s2 1→0. Then s1 receives 0 and drops the packet.

## 5. Evaluation

### Mixed rollout (mixed.txt)
        s1 emits a 5-byte tunnel header (proto_id, dst_id, hop_limit). s2 only knows the 4-byte version. So if I had to picture it, it would look something like this:
        s1 with the additional header
        [08 00] [00 02] [07] [45 00 ... IPv4 ...]
                        proto   dst_id  hop
        s2 without the additional header
        [08 00] [00 02] | [07 45 00 ... ] ← treated as "IPv4"
                        proto   dst_id    starts one byte too early

#### Prediction 
Before running s1 with new program and s2 with new (tunnel_v1.p4)

Forwarding of tunneled packets will still work. This is because as long as the position of the new header[hop_limit(8 bits)] is located at the end, proto_id and dst_id stay at the same byte positions as in v1. So s2 still reads dst_id=2 correctly and applies myTunnel_exact and sends the packet out of port 2 to h2.

As for the loop, since s2 doesnt know that hop_limit exists, it will not decrement it and after s1 (With hop_limit 8-1=7), s2 does not decrement it therefore h2 gets hop_limit=7. With both switches having the new program, h2 will receive hop_limit=6 instead.

#### What actually happened
s2 forwarded the packet correctly. logs/s2.txt shows it read dst_id = 2, found a match in myTunnel_exact, and sent the packet out of port 2 to h2. This happened because I put hop_limit at the end of the header, so the fields the old program knows (proto_id, dst_id) stayed in the same place.
     
The packet arrived at h2. h2’s interface counter went up by 1 packet and 53 bytes, exactly the size s2 logged.

But receive.py printed nothing. The packet arrived damaged:
The old s2 thinks the tunnel header is 4 bytes, but mine is 5. So s2 read the IPv4 header starting one byte too early (Bytes parsed: 38 = 14 + 4 + 20).
Before sending, the old program recalculates the IPv4 checksum (Updating checksum 'cksum' in the log). Because its view of IPv4 was shifted by one byte, it wrote the checksum over the wrong bytes, including the real IPv4 protocol field, which says the packet is UDP.
With the protocol field changed, h2 no longer saw the packet as UDP, so receive.py ignored it.
The old s2 still forwarded the packet correctly, but because it misread where the IPv4 header starts, its checksum update overwrote the real IPv4 protocol byte, so h2 received a corrupted packet that receive.py didn’t recognise as UDP.

For the loop, with exit 3 on s2 pointed back to s1, the counts went from 818/816 to 827/824: s1 +9, s2 +8. They didn’t change 5 seconds later, so the loop still ended.

**I couldnt confirm a hop limit of 7 at h2, because the capture recorded nothing.

### A safe way to roll out a header change across a network of switches that can't all be upgraded at once.

#### Option 1: Using the same design of putting the new header at the end and by keeping the same Ethertype.
Pros
Old switches still find dst_id, so they still forward correctly. Switches can be upgraded one at a time.
Cons
Old switches don’t decrement the hop limit, so loops last twice as long.
Old switches misread everything after their header. In my test, s2’s checksum update corrupted the IPv4 protocol byte, and its log looked completely normal. The failure is silent.

#### Option 2: New Ethertype for new header rolled

Phase 1: upgrade every switch to understand both the old (0x1212) and new (0x1213) headers. Senders keep using the old one.
Phase 2: once every switch is upgraded, switch the senders to the new header.
Pros
An old switch that sees 0x1213 doesn’t recognise it and drops it cleanly. It’s never misread, so it can’t be corrupted.
Easy to undo: switch the senders back.
Cons
Two rounds of upgrades, and switches carry both parsers for a while.

Recommendation is Option 2 because a packet that is dropped is easy to notice, but a packet that is silently corrupted is not like in my case is not.

### Loop Protection data plane Vs control plane
What check_paths() catches that the hop limit can’t:
-It checks the routes and catches anything before it is run ensuring loops are caught befrore they happen.
-It knows exactly where the loops is with a short print that lets programmers know the details.
-Missing routes.

What the hop limit catches that check_paths() can’t:
-Changes made after the check. If someone were to add a route in mininet>, check_paths() wont catch if a loop has occured. Having a hop limit ensures that every packet is checked every time.

### Hardware cost at line rate
Memory: 8 extra bits per packet while it’s processed.

## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->
### Claude
1)Guided me through understand and designing the optimal hop limit starting value
2)Helped me with figuring out where to decrement
3)Helped me with mostly from Evaluation part with understanding as things did not go according to my prediction

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
