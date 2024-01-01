### 🖥️ 100Gbps and beyond
Many of the organizations adopting Cilium – cloud providers, financial institutions and telecommunication providers – all have something in common: they all want to extract as much performance from the network as possible.

These organizations are building networks capable of 100Gbps and beyond but with the adoption of 100Gbps network adapters comes the inevitable challenge: how can a CPU deal with 8,000,000 packets per second (assuming a MTU of 1,538 bytes)?

That leaves only 120 nanoseconds per packet for the system to handle, which is unrealistic.

How could we reduce the number of packets a CPU has to deal with?

### By grouping packets together, of course!
GRO and GSO
Within the Linux stack, grouping packets has been done for a while through the GRO (Generic Receive Offload) and TSO (Transmit Segmentation Offload) protocols.

On the receiving end, GRO would group packets into a super-sized 64KB packet within the stack and pass it up the networking stack. Likewise, on the transmitting end, TSO would segment TCP super-sized packets for the NIC to handle.  


BIG TCP lets you group more packets together as they cross the networking stack and can significantly improve performance.