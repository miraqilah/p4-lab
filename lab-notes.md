# Week 5 Lab Notes

Student ID (last 4 digits): 1780
My addresses: 10.80.0.1 (h1), 10.80.0.2 (h2), 10.80.0.3 (h3)


1. In Step 3 you changed s2's behaviour without recompiling. Name two more things you could change that way, and two that would need a new program.
- 2 things that I could change without recompiling (at run time): delete/add the the table entry and changing the port of an entry.
- 2 things that would need new program (compile time): teach the parse and deparse new header and also adding the tunnel rules table. then defining new header and actions 

2. Why did the packet in Step 4 reach h3 and not h2 — and why did it still carry the tunnel header?
- packet reach h3 because ingress dont know about the tunnel yet, it looks at the IP address 10.80.0.3 and ignored dst_id = 2. the tunnel header is still carry because parse extracts the tunnel header and deparse emits the tunnel header back. 

3. Why did the tunnelled packet in Step 5 arrive with TTL 64, when the pings' replies had TTL 62?
- TTL 64 means that the tunnel did forwarding. on the othe hand the pings' replies had TTL 62 because it forwards using IPv4 header. ipv4_forward lowers TTL by 1. The ping crosses two switches, so 64 − 2 = 62.

4. Across Steps 4–6, which mistakes did the compiler warn you about, and which did it miss? What does that tell you about testing P4 programs?
- The compiler warns that table myTunnel_exact is unused. 
- Mistake that it miss is that the deparse at s1 left out the tunnel header, but the ethernet type still 1212. then the parser at s2 read the IP header as a tunnel header, in s2.txt key is 4500 and dst_id 0022 instead of 0800 and 0002. After that it found out that there is no exit 34 in the table, so it drop the packet. labelled is not completed and sent then drop the packet.
- compiler check that the code is valid, not the program is correctly writen. Therefor i need to check the code by doing the test and reads the logs.