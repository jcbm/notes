# Chapter 5 - Introduction to Wireless Networks

## Ubiquitous Connectivity

Any network not connected by cables.\
Convenience and mobility for the user.

Many use cases and applications => many different wireless technologies with their own performance characteristics, optimized for a specific task and context.\
WiFi, Bluetooth, ZigBee, NFC, WiMAX, LTE, HSPA, EV-DO, earlier 3G standards, satellite services, and more.

Hard to make broad generalizations about performance of wireless networks.\
Most wireless technologies 
- operate on common principles
- have common trade-offs
- subject to common performance criteria and constraints

All applications should perform well regardless of underlying connectivity.\
As a user, you should not care about the underlying technology in use.\
As developers we must think ahead and architect our applications to anticipate the differences between different types of networks. 

## Types of Wireless Networks

A network is a group of devices connected to one another.\
For wireless networks, *radio* communication is usually the medium.\
Within the radio-powered subset, there are dozens of different technologies designed for use at different scales, topologies, and for very different use cases. 
 
Use cases can be partitioned based on their "geographic range"

- Personal area network (PAN)	
-- Within reach of a person	
-- Cable replacement for peripherals
--	Bluetooth, ZigBee, NFC
Local area network (LAN)
--	Within a building or campus
--	Wireless extension of wired network	
-- IEEE 802.11 (WiFi)
Metropolitan area network (MAN)
--	Within a city
--	Wireless inter-network connectivity
--	IEEE 802.15 (WiMAX)
Wide area network (WAN)
--	Worldwide
--	Wireless network access	Cellular (UMTS, LTE, etc.)

 - Some devices have access to a continuous power source
 -- others must optimize their battery life at all costs 
 - Some require Gbit/s+ data rates
 -- others are built to transfer tens or hundreds of bytes of data (e.g., NFC) 
 - Some applications require always-on connectivity
 -- others are delay and latency tolerant 
 
 These and many other criteria determine the original characteristics of each type of network.\
 Once in place, each standard continues to evolve: 
 - better battery capacities
 - faster processors
 - improved modulation algorithms
 - etc
 
Extends the use cases and performance of each wireless standard.
 
 
## Performance Fundamentals of Wireless Networks

Every type of wireless technology has its own set of constraints and limitations. 

Regardless of the specific wireless technology in use, all communication methods have a maximum channel capacity, determined by the same underlying principles.\
Shannon - Exact mathematical model  to determine channel capacity, regardless of the technology in use - releates channel capacity (bits per second) to the available bandwidth (hertz), signal and noise (both watts).

### Bandwidth

Radio communication by its very nature uses a shared medium: radio waves (=electromagnetic radiation). 

The sender and receiver must agree up-front on the specific frequency range over which the communication will occur.\
A well-defined range allows seamless interoperability between devices.\
E.g. the 802.11b and 802.11g standards both use the 2.4–2.5 GHz band across all WiFi devices.

The local government determines the frequency range and its allocation.\
E.g. the Federal Communications Commission (FCC) in the US.\
Due to different government regulations, some wireless technologies may work in one part of the world, but not in others.\
Different countries may, and often do, assign different spectrum ranges to the same wireless technology.

The most important performance factor is the size of the assigned frequency range.\
The overall channel bitrate is directly proportional to the assigned range.\
A doubling in available frequency range will double the data rate. E.g., going from 20 to 40 MHz of bandwidth (how 802.11n improves its performance over earlier WiFi standards).

Not all frequency ranges offer the same performance. 
- Low-frequency signals
-- travel farther - cover large areas (macrocells) 
-- requires larger antennas and having more clients competing for access 
- High-frequency signals 
-- can transfer more data
-- won’t travel as far, resulting in smaller coverage areas (microcells)
-- requires  more infrastructure

Certain frequency ranges are more valuable than others for some applications.
- Broadcast-only applications (e.g., broadcast radio) are well suited for low-frequency ranges
- Two-way communication benefits from use of smaller cells - provide higher bandwidth and less competition

### A Brief History of Worldwide Spectrum Allocation and Regulation

In the early days of radio, anyone could use any frequency range for whatever purpose.\
Changed when the Radio Act of 1912 was signed into law within the United States and mandated licensed use of the radio spectrum. 
 
The Communications Act of 1934 created the Federal Communications Commission (FCC) - responsible for managing the spectrum allocation within the U.S ever since, effectively "zoning" it by subdividing into ever-smaller parcels designed for exclusive use.

E.g. the "industrial, scientific, and medical" (ISM) radio bands, established at the International Telecommunications Conference in 1947 - reserved internationally.\
The 2.4–2.5 GHz (100 MHz) and 5.725–5.875 GHz (150 MHz) bands, which power much of our modern wireless communication (e.g., WiFi) are part of the ISM band.\
Both are considered "unlicensed spectrum," - allows anyone to operate a wireless network —for commercial or private use— in these bands as long as the hardware used respects specified technical requirements (e.g., transmit power).

Due to the rising demand in wireless communication, many governments hold "spectrum auctions," where a license is sold to transmit signals over the specific bands.\
E.g. The 700 MHz FCC auction in 2008: The US 698–806 MHz range was auctioned off for a total of $19.592 billion to over a dozen different bidders (the range was subdivided into blocks).

Bandwidth is a scarce and expensive commodity. The current allocation process is a highly contested area of discussion.

### Signal Power
Second fundamental limiting factor in all wireless communication - Signal power between the sender and receiver (*the signal-power-to-noise-power*, S/N ratio, or SNR).\
Compares the level of desired signal to the level of background noise and interference.\
The larger the amount of background noise, the stronger the signal has to be to carry the information.

All radio communication is done over a shared medium, which means that other devices may generate unwanted interference.\
E.g., a microwave oven operating at 2.5 GHz may overlap with the frequency range used by WiFi, creating cross-standard interference.\
Other WiFi devices, such as your neighbors’ WiFi access point or your coworker’s laptop accessing the same WiFi network, also create interference for your transmissions.

Ideally, you would be the one and only user within a certain frequency range, with no other background noise or interference.\
Unfortunately, that’s unlikely. 
- bandwidth is scarce
- simply too many wireless devices to make that work

To achieve the desired data rate where interference is present, either
- increase the transmit power, thereby increasing the strength of the signal
- decrease the distance between the transmitter and the receiver
- or both

*Path loss*, or path *attenuation*, is the reduction in signal power with respect to distance traveled — the exact reduction rate depends on the environment. 

Imagine you are in a small room and talking to someone 20 feet away.\
With nobody else present, you can hold a conversation at normal volume.\
With a few dozen people into the same room, each carrying their own conversations - it becomes impossible for you to hear your peer. 

You could speak louder, but doing so would raise the amount of "noise" for everyone around you.\
In turn, they would start speaking louder also and further escalate the amount of noise and interference.\
Before you know it, everyone in the room is only able to communicate from a few feet away from each other.

Illustrates two important effects:
- Near-far problem 
-- A receiver captures a strong signal and thereby makes it impossible for the receiver to detect a weaker signal, effectively "crowding out" the weaker signal
-- One, or more loud speakers beside you can block out weaker signals from farther away
- Cell-breathing 
-- the coverage area, or the distance of the signal, expands and shrinks based on the cumulative noise and interference levels
-- the larger the number of other conversations around you, the higher the interference and the smaller the range from which you can discern a useful signal

These limitations are present in all forms of radio communication, regardless of protocol or underlying technology.

### Modulation
While available bandwidth and SNR are the two primary, physical factors that dictate the capacity of every wireless channel, the algorithm by which the signal is encoded can also have a significant effect.

Our digital alphabet (1's and 0's), needs to be translated into an analog signal (i.e. a radio wave).\
Modulation is the process of digital-to-analog conversion, and different "modulation alphabets" can be used to encode the digital signal with different efficiency.\
The combination of the alphabet and the symbol rate is what then determines the final throughput of the channel. 

Example:\
Receiver and sender can process 1,000 pulses or symbols per second (1,000 baud).\
Each transmitted symbol represents a different bit-sequence, determined by the chosen alphabet (e.g., 2-bit alphabet: 00, 01, 10, 11).\
The bit rate of the channel is 1,000 baud × 2 bits per symbol, or 2,000 bits per second.

Choice of the modulation algorithm depends on 
- the available technology
- computing power of both the receiver and sender
- the SNR ratio

A higher-order modulation alphabet comes at a cost of reduced robustness to noise and interference.

## Measuring Real-World Wireless Performance
The performance of any wireless network is fundamentally limited by a small number of well-known parameters.\
Specifically, the amount of allocated bandwidth and the signal-to-noise ratio between receiver and sender. 

All radio-powered communication is:
- Over a shared communication medium (radio waves)
- Regulated to use 
-- specific bandwidth frequency ranges 
-- specific transmit power rates
- Subject to 
-- continuously changing background noise and interference
-- technical constraints of the chosen wireless technology
-- constraints of the device: form factor, power, etc.

All wireless technologies advertise a *peak*, or a *maximum data rate*. 
E.g. 
- 802.11g standard is capable of 54 Mbit/s 
- 802.11n standard raises the bar up to 600 Mbit/s. 
- Some mobile carriers are advertising 100+ MBit/s throughput with LTE. 

Be aware that these numbers have been meausred under ideal conditions.
E.g. 
- maximum amount of allotted bandwidth
- exclusive use of the frequency spectrum
- minimum or no background noise
- highest-throughput modulation alphabet
- increasingly, multiple radio streams (multiple-input and multiple-output, or MIMO) transmitting in parallel 

Factors that may affect the performance of your wireless network:

- distance between receiver and sender
- background noise in current location
- interference from users in the same network (*intra*-cell)
- interference from users in other, nearby networks (*inter*-cell)
- available transmit power, both at receiver and sender
- processing power and the chosen modulation scheme

I.e. for maximum throughput, eliminate any noise and interference you can control, place your receiver and sender as close as possible, give them all the power they desire, and make sure both select the best modulation method.\
If you want performance, use a physical wire. The convenience of wireless communication comes at a cost.

Measuring wireless performance is tricky.\
A small change, on the order of a few inches, in the location of the receiver can easily double throughput, and a few instants later the throughput could be halved again because another receiver has just woken up and is now competing for access to the radio channel.\
By its very nature, wireless performance is highly variable.

Section has been focused exclusively on throughput.\
Latency have been omitted on purpose - latency performance in wireless networks is directly tied to the specific technology used. See next chapter.