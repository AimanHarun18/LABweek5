# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2206**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

<!-- How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. -->

     i started the network with the finished lab program. On s2, I change tunnel exit 3 so it goes back to s1 instead of h3. The dump showed key 3 is handle2:
this is the command:
sh ./p4cli s2 table_dump MyIngress.myTunnel_exact
sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
h1 python3 send.py 10.6.0.3 "loop" --tunnel 3

so now s1 sends dst_id 3 to s2 and s2 sends it back to s1, again and again.

what does the logs showed 

right after the sending s1=64 s2=61
5 seconds later s1=1079 s2=1078 

h1 sent only one packet, but the counts kept going up by about 1000 in 5 seconds, the loop never stop by itself.

why nothing stop it : myTunnel_forward only sets the output port, nothing in the tunnel header changes from hop to hop so a switch cannot tell if a packet has been going around in circles.

tunnelled packets never touch the TTL. Only ipv4_forward decrement it and tunnelled packets use myTunnel_forward. In my tests, tunnelled packets always arrived with TTl 64.

even for plain IPv4, ipv4_forward does ttl = ttl -1 but never checks the result. When the TTL reaches 0, the packet is still forwarded and on the next hop 0-1 wraps to 255. A real router drops the packet when the TTL runs out, this program doesnt so an IPv4 loop would also go on forever.

## 2. Design: the hop limit

<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->

- proto_id(16 bit) same as before
-dst_id(16 bit) same as before
- hop_limit (8 bit) new: its means how many more switches the packet may cross
- reserved(8 bit) always 0

size 8 bits means it holds 0-255 the same range as the ipv4 TTL, that is far more than this network needs, 16 bits would just wate the byte in every packet.

position at the end of the header, proto_id and dst_id stay at the same place so an old switch can still read dst_id correctly. The reserved byte makes the header 6 bytes an even number so the ipv4 header behind it stays 2 byte aligned. It can also be used later.

starting value is 8, the longest real path here is 2 switches so, 8 leaves room for a bigger network. It is still small so a looping packet dies quickly.

It is decremented in Ingress before the table:
if hop_limit less or the same as 1, it will drop, while apply myTunnel_exact whose action myTunnel_forward takes 1 off hop_limit

everything must aggre on the header, the parser and deparser use the header type so they read and write 6 bytes automatically. The host must match too. i added the same two fields to the scrapy MyTunnel class in both send.py and receive.py. If one side is different, every byte after the header is read fromthe wrong place.

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->

thus is what i changed for this lab

tunnel.py
- Added bit<8> hop_limit and bit<8> reserved to myTunnel_t
- myTunnel_forward now also does hdr.myTunnel.hop_limit = hdr.myTunnel.hop_limit - 1;
-In ingress apply, a tunnelled packet with hop_limit <= 1 is dropped before myTunnel_exact is applied.
- i did not change the ipv4 path to keep the change on the tunnel

send py
- added ByteField("hop_limit", 8) and ByteField("reserved", 0) to MyTunnel.
added --hops H (0–255, default 8) to set the starting value.

receive.py
- same two fields in MyTunnel
- prints hop_limit= for tunnelled packets

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

       the evidence that it works is in the delivery.txt fike that i push into the github.

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.

before running, old s2 only know a 4 byte tunnel header. proto_id and dst_id are still in the same place, so in a loop only s1 countdown. the loop should still end, but take about twice as many passes.

what happened after: 
tcpdump on h2 captured the packet, so delivery to h2 worked
but log/h2.txt showed nothing: receive.py did not see it as our UDP packet

the bytes that reached h2 
0x0000:  ffff ffff ffff 0000 0000 0001 1212 0800
0x0010:  0002 0700 4500 0021 0001 0000 3914 66bc

The bytes that reached h2 show what happened. The EtherType is still 1212, so the frame is still marked as tunnelled. proto_id is still 0800 (IPv4), and dst_id is still 0002. Old s2 reads these first 4 bytes correctly, which is why it still forwarded the packet to h2. The next byte, hop_limit, is 07. s1 took it from 8 to 7, but old s2 does not know this field, so it did not decrement it. The reserved byte is still 00, and the first 8 bytes of the IPv4 header (4500 0021 0001 0000) are unchanged.

The next two bytes are wrong. The TTL should be 40 (64) but is 39 (57), and the protocol should be 11 (17, UDP) but is 14 (20). Old s2 thinks the tunnel header is 4 bytes, so it believes IPv4 starts 2 bytes earlier than it really does. When it recalculates what it thinks is the IPv4 checksum, it writes the result (0x3914) over the real TTL and protocol bytes. Because the protocol is no longer UDP, receive.py ignored the packet. That is why tcpdump captured it but logs/h2.txt stayed empty.

Why the TTL and protocol changed. Old s2 reads only 4 bytes of tunnel header, so its "IPv4 header" starts 2 bytes early. In s2's view, the checksum field is 10 bytes into IPv4. In the real packet, those 2 bytes are the TTL and protocol. The old program recalculates the IPv4 checksum and writes it there (0x3914), overwriting TTL and protocol. Protocol 20 is not UDP, so receive.py ignored the packet.

So the packet was delivered but silently corrupted. I didn't fully predict this part. I expected s2 to misread IPv4, but not to write into the packet.

for the loop only s1 decremented, so s1 saw the packet with 8, 7, 6 … 1 and dropped it on its 8th visit. Old s2 just passed it back 7 times. The loop still ended, but after 15 switch visits instead of 8. With half the switches upgraded, the protection was about half as strong, as I predicted.

     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).

There are three ways to roll out a header change when not all switches can be upgraded at once.

The first option, which I used, is to put the new fields at the end of the header. Old switches can still find dst_id, so delivery keeps working during the upgrade. The problem is that old switches misread everything after the header and can corrupt the packet, as section 5a showed. Old switches also do not enforce the hop limit, so loop protection is only partial.

The second option is to give the new header its own EtherType, for example 0x1213. Old switches do not recognise it, so they drop the packet instead of misreading it. This fails safe, and the old and new headers can be used side by side. The downside is that a new-format packet cannot pass through any old switch.

The third option is a version field, for example using the reserved byte, so that switches check the version before reading the rest. This makes future changes easy, but it does not help now, because the old header has no version field.

My proposal is a two-step rollout. First, upgrade every switch so it understands both the old header (0x1212) and the new one (0x1213), while hosts keep sending the old header, so nothing breaks. Then, once all switches on a path are upgraded, hosts start sending the new header. Old switches never see the new header, so nothing gets corrupted, and rolling back only means hosts go back to the old header.

     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?

The control plane check (check_paths() from Week 3) runs before routes are installed. It can find wrong routes, loops and dead ends before any packet is lost, and it can say exactly which entry is wrong. It cannot see changes made outside it, though. For example, my p4cli table_modify went straight to the switch and bypassed it. It also misses short loops that appear while switches are updated one by one, and bugs inside a switch.

The data plane hop limit works on every packet while the network runs, so it stops any loop whatever caused it. However, it only acts after the loop already exists, so up to 8 copies of a packet are wasted first. It also does not fix or report the loop, and it cannot find dead ends.

Real networks use both. The control plane prevents most mistakes in advance, and the hop limit is the safety net for whatever gets through.

     d) What would this change cost on a hardware switch running at line rate? -->

- Bandwidth is 2 extra bytes per tunnelled packet. That is about 3% of a minimum 64-byte frame, and the usable MTU is 2 bytes smaller.
- Processing for one compare (<= 1) and one subtract on an 8-bit field. A hardware switch does this in the same pipeline stage as the tunnel lookup. No new table, no extra memory, no extra pass, so the switch still runs at full line rate.
- The parser reads 2 more bytes and the header uses a little more space in the switch's header storage. The biggest real cost is the rollout (section 5b), not the speed.


## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

I used Claude (Anthropic) to:

- explain the lab and assignment steps and the commands;
- suggest the hop-limit design and the code changes (I typed and tested them myself);
- help fix git and command mistakes during the work;
- draft this report from my own results.

All experiments were run by me, and all evidence files are from my own runs.     

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
