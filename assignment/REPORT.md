# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): 1780

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

- How tunnel loop created: on s2 the entry table is modify from handle 2 to port 1 by using sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1" command
- Switch logs shown: the count is increase from 1422 to 1441 on s1 and 1421 to 1441 on s2
- Check tunnel.p4 code: no it does not stop the IPv4 loop because there is no a single line of code that check if TTL=0... so basically if TTL reach 0 the loop keep looping because to TTL is set to 8-bit number which if 0-1 it will wraps around to 255 and it keep going to 0 again and again.

<!-- How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. -->

## 2. Design: the hop limit
- field's size: 16 bits because it holds 0 to 65535, far more tha a 2-switch network needs. Then, since proto_id and dst_id has 16 bits i also wanted all the field in header to look uniform.
- position: at the end of the tunnel header which is after bit<16> dst_id; so that the old field which is [ proto_id ][ dst_id ] stay in the same place. 
- starting value: since the min value should be 2, i choose 10 so that it have some round of loop (expecting only 10 passes between the switches) before the packet drop. Then it also is not that long for 2 switches. 
- where decremented: at each switch the packet reach, the decrement will happend in the ingress in the tunnel branch, before the myTunnel_exact lookup. This will check the current hop number. if number > 0 it will decrease by 1. if number = 0 the switch will drop the packet immediately. By doing this, the number wont be wrapping from 0 to 255 like IPv4 TTL before.
- Other things that need to change: tunnel.p4 (header definition)=adding new field size in the header; after dst_id. tunnel.p4 (ingress)=the check for > 0, the drop, and the decrement. send.py=write the starting number on the packet, default 10 and receive.py=print the hop limit that arrives. Then the host need to be updated too so that they wonldn't disagree with the switches about the tunnel header length. Parser and deparser=no edit needed, because extract and emit use the whole header definition.
<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->

## 3. Implementation
- tunnel.p4: added a 16-bit hop_limit field after dst_id in myTunnel_t header. 
     In ingress, the tunneled packet checks hop_limit > 0, if true the hop_limit subtract 1, then look up myTunnel_exact; otherwise it drop.
     The parse and deparse is not edited because extract and emit automatically pick up the new field since it use hdr.myTunnel in the code.
- send.py: adding MyTunnel field with a ShortField hop_limit that has default 10 and a --hops option as a hops starter.
- receive.py: adding the fields same as send.py, and it also prints the hop limit that arrives at the host.
- Things the diff doesn't show: The tunnel header grow from 4 bytes to 6. Therefore, the host's field (send.py and receive.py) needs to be changed too in order to prevent from disagrement regarding where exaclty IPv4 header should start.

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->

## 4. Evidence: it works
- delivery.txt: thunnel packet that send via --tunnel 2 and --tunnel 3 reach h2 and h3 respectively. Then, hop limit = 8 (10-2[s1,s2]). Then ping IPv4 still works because of else if (hdr.ipv4.isValid()) branch in ingress.
- loop-after.txt: the loop is ended after 10 passes and 1 arrival that dropped , which is 236-242=6 passes from s1 and 235-240=5 passes from s2. Therefore this result match the design as it has 10 passes and the last hop=0 is dropped

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

## 5. Evaluation

a)THE PREDICTION
- delivery: if using h1 python3 send.py... --tunnel 2... the delivery still happen meaning that packet form h1 will be send to h2 because both of the tunnel.p4 and tunnel_v1.p4 has applied tunnel header. Also, since s2 run the old program s2 will mistakenly parse byte 5 and 6 as the IPv4. Then IPv4 ping still works normally because it does not involded the tunnel header.

- loop: s1 will do 10 passes and 1 arrival to drop since the s2 do not have instruction/code to reduce the hop limit and only check the tunnel id and forward it only.



b)PROPOSE SAFE WAY
- the ordering for new field should be placed at the back of the header, so that when applying mixing rollout both of the switches s1 and s2 can read the old field since it stay in the same place. 

- The switch across the network can be program with the both old and new header first... Only when all switches are upgraded, change the senders (send.py) to use the new header. Then remove the old format.


c) LOOP PROTECTION
- check_paths(): It detects looping or missing routes in the routing tables before packets are sent, and works for all traffic types.

- Limitations: It cannot detect problems that happen during runtime, such as changed routing entries or incorrect packet parsing.

- Hop limit: It can detect loops that are happening during packet transmission, but only for tunnelled packets and switches that support it. For example, mixed.txt takes 21 passes instead of 11.



d)cost on a hardware switch running at line rate
- Extra bytes: Each packet that goes through the tunnel will become 2 bytes bigger because the header increases from 4 bytes to 6 bytes. If we use 8 bits instead, it would only increase by 1 byte.

- Extra work per packet: The system needs to perform one comparison (to check whether the value is greater than 0) and one subtraction for each packet. However, these operations are simple and can usually be handled within one pipeline stage. So, the processing speed should remain the same as long as there is an available pipeline stage.

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

## AI use
- Im using Claude (Anthropic) to explained the lab and P4 concepts, explained the logs and code, suggested the design options and the P4/Python edits, and gave feedback on my report drafts.
- I ran every command, chose the 16-bit size and starting value of 10 (claude suggested other number but i ask it whether i can use my own number so that claude consult me), tested everything, and wrote the report.

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
