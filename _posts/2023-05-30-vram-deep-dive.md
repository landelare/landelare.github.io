---
title:  "VRAM deep dive"
excerpt: "A deep dive into VRAM, how it's used, and when it's not \"enough\"."
---

One of the most popular topics in tech media currently is how 8 GB of VRAM is
"not enough" to play modern games, "even in 1080p".

While this is more or less the case from the perspective of someone who plays
video games (and newly-released video cards having less VRAM than what would be
appropriate for their price class is certainly not helping), there's plenty of
misinformation and widely-repeated half-truths surrounding this topic.

My hope is that by providing a detailed look and explanation of exactly how and
why VRAM is used, I can help you a little to cut through popular misconceptions
and outright marketing bullshit.

# PC architecture

Let's start by going through why there are multiple kinds of memory to begin
with, and when that's not the case.

The relevant parts of a usual desktop "gaming PC" are connected like this:

![Block diagram of an ATX motherboard and PCIe GPU](/images/ram_schematic.svg)

We'll be focusing on the part in black; the chipset can be mostly ignored when
talking about the most common gaming PC build with a single 
<abbr title="Dedicated or Discrete Graphics Processing Unit">dGPU</abbr> and
<abbr title="Non-Volatile Memory Express">NVMe</abbr>
<abbr title="Solid-State Drive">SSD</abbr>, as there will be one of each
connected directly to the CPU's built-in controller.

RAM is also connected directly to the CPU on modern systems, and GPUs always
have VRAM soldered onto their boards and connected directly.

<sup>
Older systems had two major chips on a motherboard, called the northbridge and
the southbridge.
The southbridge, RAM, and PCIe (or its predecessors,
<abbr title="Accelerated Graphics Port">AGP</abbr> or PCI) were accessed through
the northbridge, but this was eliminated over time and integrated into the CPU.
The southbridge still remains on modern motherboards, and is now called just the
chipset.
</sup>

# PCI Express

<abbr title="Peripheral Component Interconnect Express">PCIe</abbr> is a
packet-based network, like Ethernet.
For most consumer-grade computers, the entire network is contained within the PC
chassis and communication on it tends to be the CPU talking to other hardware
and them responding back to the CPU.

This is not a requirement, though: there are PCIe switches, bridges, and even
long-distance external cables running for more than 1 km.
PCIe devices can also communicate with each other.
PCIe packets can even be sent over Ethernet.[^expether]
Although PCIe is mainly known for its high bandwidth (a PCIe 5.0 x16 slot is
capable of ≈58.69 GB/s), it is also very tolerant to transmission errors and
latency, probably more than you are when your
<abbr title="Frames Per Second, you knew this one ;)">FPS</abbr> drops due to
it!

[^expether]: <https://en.wikipedia.org/wiki/ExpEther>

Well-optimized games try and transfer as little as possible (but still a lot)
over PCIe, and arrange transfers in advance to reduce the impact of this
latency.
The most notable effect of this is that the GPU is nearly always at least one
frame behind the CPU's processing, which affects user-input-to-display latency.

NVMe is a protocol most commonly used over PCIe to talk to SSDs.
It replaced
<abbr title="Serial AT Attachment, AT refers to the IBM PC/AT">SATA</abbr>,
which was initially used for compatibility with HDDs in existing systems when
SSDs were first introduced to the consumer segment.
NVMe is often incorrectly referred to as M.2, which is the form factor of most
of these drives.
SATA is also commonly misused to refer to the larger 2.5″ form factor.
M.2 drives can use SATA[^m2sata], and NVMe drives that aren't M.2, 2.5″, or even
SSDs also exist.[^nvmehdd]

[^m2sata]: <https://www.youtube.com/watch?v=8iNf8hRn1N0>

[^nvmehdd]: <https://blog.seagate.com/enterprises/seagate-unveils-worlds-first-native-nvme-hdd-demo-at-ocp/>

<sup>
While we're at this topic, please stop calling SSDs just "memory"...
It's technically true, congratulations, you won, but you're only supporting
bullshit marketing that adds up SSD+RAM capacity because bigger number better.
</sup>

# Unified memory

Although the rest of the article will mainly focus on RAM and VRAM being
separate with a PCIe bus between them, this is not universally true.
<abbr title="Integrated Graphics Processing Unit">iGPU</abbr> systems, such as
many laptops, and most notably current-generation consoles do not have this
separation, and have one universally-accessible pool of system RAM.

This has the benefit of not having to transfer data between RAM and VRAM at all,
full flexibility on how much memory is used for graphics data, but at the price
of the CPU and GPU competing for memory bandwidth and running slower.

Xbox likes to cut costs by having a "fast part" and a "slow part" to its memory,
while PlayStations tend to offer the full bandwidth all around, although the PS5
has a small amount of additional slower memory for background tasks[^ps5ddr4].
The Nintendo Switch is also using unified memory.

[^ps5ddr4]: <https://www.ifixit.com/Teardown/PlayStation+5+Teardown/138280#s277304>

In such a system, there's usually a minimum amount of memory dedicated to the
CPU and GPU to ensure that they can always function.
This is only a few megabytes, most memory used for graphics is allocated
dynamically from shared RAM as needed.

# Memory interface width

One of the many values that a video card is marketed with is the width of its
memory interface.

If a card has some amount of VRAM at a width of, e.g., 256 bits, that means one
memory transfer contains 256 bits = 32 bytes.

The width of an individual memory chip is standardized, but they add up when
more of them are used.
For instance, 8 GB of VRAM can be built from eight 8 Gb chips (1 byte=8 bits)
giving a total width of 256 bits, or four 16 Gb chips for only 128 bits and half
the bandwidth if it's running at the same clock speed.

Most of the cost associated with the interface's width is on GPU core itself.
Higher bus widths are more expensive to design and manufacture.
The same total capacity is also usually more expensive in more, smaller chips
than fewer, larger ones.

# RAM standards and specs

In a standard PC with a dGPU, "regular" RAM (connected to the CPU) is DDR while
VRAM is mostly GDDR (sometimes HBM), with only some of the cheapest, borderline
scam video cards using DDR as VRAM.

## DDR
{:.no_toc}

DDR, full name
<abbr title="Double Data Rate Synchronous Dynamic Random-Access Memory">DDR
SDRAM</abbr>
is a series of standards (DDR, DDR2, DDR3, etc.) for memory modules by
<abbr title="Joint Electron Device Engineering Council">JEDEC</abbr>.
Let's break this down:
* DDR means that two transfers occur per clock cycle
(one on each edge: ↑‾↓\_).<br>
DDR memory running at 2500 MHz is capable of 5000M transfers per second.
* S refers to the fact that there's a clock signal at all that _synchronizes_
with the memory controller.
* DRAM is a much cheaper construction per bit than its premium counterpart,
<abbr title="Static">S</abbr>RAM, but it's slower and needs to be constantly
refreshed to not lose data.[^sram]

[^sram]: SRAM is used for memory where speed is paramount, such as chip caches.
    Due to its high cost, these are still only kilobytes or megabytes large.

Bus width for DDR is usually not given as a number of bits but rather the number
of channels that the CPU's memory controller uses.

A single DDR memory stick's interface is always 64 bits wide, making the most
common dual-channel setup 128 bits wide.
This is why it's recommended to always buy memory in pairs for most
consumer-grade CPUs.[^ddrkits]
Buying, e.g., one 64 GB stick instead of 2x32 GB effectively halves your memory
bandwidth, leaving performance on the table.
Some expensive CPUs have quad-channel, or even 8 or 12-channel memory
controllers, and for a short time, tri-channel was also a thing.

[^ddrkits]: And compatibility with each other, but that's an arcane topic.

<sup>
Speaking of leaving performance on the table, make sure you read up on what XMP,
DOCP, or EXPO is and check if you can stably enable it on your system!
</sup>

## GDDR
{:.no_toc}

<abbr title="Graphics">G</abbr>DDR is another set of memory standards from
JEDEC, having higher bandwidth than "regular" DDR.

The numbering is independent, e.g., DDR3 is not related to GDDR3.
GDDR chips tend to be more expensive per byte than DDR, leading to most consumer
PCs having more RAM than VRAM to balance costs.

Despite using bandwidth-optimized chips, VRAM bandwidth is one of the most
precious resources for a game when it comes to GPU performance.
The way GPUs execute instructions is tailored to hide some of the memory
latency, so while it's still present and detrimental, it has a smaller impact
than it does on a CPU.
Optimizing a game that's bottlenecked by VRAM normally requires using
vendor-specific tools (NVIDIA Nsight, Radeon GPU Profiler, Intel GPA) for
measurements.

The most recent GDDR standards, such as GDDR5X or GDDR6(X) go beyond their
names and can operate in <abbr title="Quad Data Rate">QDR</abbr> or even
<abbr title="Octal Data Rate">ODR</abbr>.
They've also recently started using advanced signaling techniques to deliver
more bits on a single wire at the same time (e.g., 4 levels of voltage
correspond to 00, 01, 10, or 11).

This is how very high numbers such as a 24 GHz "effective" clock are put onto
marketing sheets.
The true clock frequency is ⅛ of that in this example: 3 GHz.
Recently, the "marketing unit" of this changed to Gbps, which has a weird
meaning when used as a direct replacement:
it's the bandwidth of one single wire running between your GPU core and a GDDR
chip, completely irrelevant for you both as an end user and as a game developer.

Total memory bandwidth (hundreds of GB/s or even TB/s on high-end GPUs) is more
interesting.
To get from single-wire bandwidth to total bandwidth, multiply by the memory
interface's bus width, then divide by 8 to get from bits/s to bytes/s.
Keep in mind that GHz is using the 1000-based definition of giga.

## LPDDR
{:.no_toc}

<abbr title="Low Power">LP</abbr>DDR is yet another set of standards focusing
on low power use, sacrificing performance.
It's mainly used in laptops and other mobile devices, such as the Nintendo
Switch.

## HBM
{:.no_toc}

<abbr title="High Bandwidth Memory">HBM</abbr> seems to come and go on graphics
cards, being quickly replaced by GDDR on the next model or generation whenever
it's introduced for a product.

It's even more expensive than GDDR per stored byte, and its main benefit is
having a very high bus width per single chip (1024 bits vs. 32 bits for GDDR).
This makes connecting it to the GPU very difficult, which is why HBM chips are
always right next to the GPU die instead of only being somewhere nearby on the
board as GDDR chips are.

# RAM vs VRAM

Since PCIe has higher latency and lower bandwidth than even DDR, it makes sense
to place data used by the CPU into RAM, and data used by the GPU into VRAM, so
that everything is "closer" to the processing unit that needs it.

## RAM

The CPU usually deals with data related to game logic and peripherals:
where things are, their stats such as health, user input, etc.
Physics, networking, and audio are also handled mostly or entirely by the CPU.
A lot of this data tends to be in granular, complex data structures.

CPU memory is also "virtual", without exception:
this means that the layout of RAM that a program sees is specific to it.
Other programs could be storing something entirely different at the same
apparent location without even being aware of this, they're mostly protected
from each other's memory errors, and your OS is free to decide what is backed by
actual RAM, or perhaps most infamously a pagefile or swapfile (two terms for the
same technique).

While swapping is most commonly associated with a corresponding nosedive in
performance in end users' minds, it is overall beneficial:
you can still save your game, or Alt+Tab out, close something else and continue
instead of losing progress because your game crashed.
Most of the time you won't notice swapping even happening, and of course your
OS is also avoiding swapping as much as it can.

CPU caches help game performance by holding some data even closer to the CPU
than RAM, so that it has to talk to the cheaper, worse-performing DDR less
often.
Games tend to especially benefit from larger caches than other workloads, which
is why processors with extra eDRAM or "3D V-Cache" tend to punch above their
weight when compared to processors without the extra cache, often outperforming
even their successors!

GPUs also use caches in the same way, with the same performance-enhancing goals.

## VRAM

The GPU (to no one's surprise) deals with graphics:
the shape of things, where and what color they are, how shiny they are, etc.

Most data in VRAM is held in coarse blocks called resources:[^oglobj] textures
(1D, 2D, or 3D collections of pixels called texels) and buffers (1D
collections of repeated data elements that aren't necessarily color) tend to
occupy the most space.
The colors in a texture are sometimes interpreted as something else.
For instance, normal maps are textures whose texels store directions:
the red, green, and blue values represent the X, Y, and Z coordinates of
vectors.

[^oglobj]: Or more confusingly, objects. Thanks, OpenGL!

All modern GPUs use virtual memory, but it was introduced decades after CPUs had
it.
Swapping VRAM into regular RAM (and then possibly to disk) was available
_before_ GPU virtual memory.
The benefits are different: the most relevant one for this article is that GPU
virtual memory eliminates nearly all VRAM fragmentation.
Memory fragmentation can lead to situations where even though there's
some amount of free VRAM available, it cannot be used without taking a
significant performance hit while memory is defragmented.[^fragmentation]

[^fragmentation]: A HDD being slowed down by fragmentation is a different
    symptom of the same underlying problem.
    Using 100% of its capacity without defragmentation is possible because file
    systems use different data structures than GPUs.

Engineers are not the best at naming things consistently (_\*cough\*_ USB 3.2
Gen 2x2), so sometimes a texture is called a buffer:
stencil buffers, Z-buffers, G-buffers, framebuffers are common terms for certain
usages of textures.
These are usually recalculated every single frame, and the name highlights their
usage instead of their dimensionality or format.

<sup>
Additional terms for (mostly 2D) textures include surfaces and images.
Textures that represent a precomputed function (often 1D) are sometimes called
<abbr title="Look-Up Table">LUT</abbr>s.
</sup>

# Resolution's impact on VRAM usage

Calling the entirety of VRAM a "buffer", or even worse, a "framebuffer" is
therefore wrong:
there are multiple, smaller buffers within VRAM, and they only account for a
small fraction of total VRAM usage.

A framebuffer in particular is only a few dozen megabytes in size, and games
only use a handful of them, sometimes as low as 2 (but usually more in modern
games).
A 4K framebuffer using a relatively common
<abbr title="High Dynamic Range">HDR</abbr> pixel format (R10G10B10A2) is only
about[^stride] 3840\*2160\*4 bytes large, less than 32 MB!

[^stride]: Textures (including framebuffers) sometimes use more memory than
    what's strictly needed to store their pixels to simplify and speed up some
    calculations that are done directly by the GPU hardware.

Therefore, increasing your rendering resolution in itself is extremely unlikely
to make you run out of VRAM: every texture tied to your resolution combined
won't even account for 1 GB of total VRAM usage.
Running multiple monitors will only add a similar amount of extra memory
required, not even close to account for 4+ gigabytes of VRAM usage (or
allocation—more on that later).

The main reason why VRAM usage correlates with your resolution is the amount of
texture detail needed, with mesh detail
(<abbr title="Level of Detail">LOD</abbr>s) being a distant second.[^nanitelod]
Most textures in a game are stored in
"<abbr title="multum in parvo - much in little">mip</abbr> chains": the texture
at full resolution, then horizontal/vertical size halved giving ¼ the texels,
then ¼ of that, etc.
Mip chains can be partially loaded: if a texture only appears on a distant or
small object, data from the most detailed levels can be discarded in order to
make space for other textures that have a bigger visual impact.

[^nanitelod]: Geometry virtualization techniques such as Unreal Engine 5's
    Nanite reduce the gap between texture and mesh data by using less of the
    former and more of the latter, but the overall correlation with resolution
    remains the same.

A higher resolution needs more detailed textures: if something covers 20000
pixels in 1920x1080, it will cover about 80000 pixels on the same image at
3840x2160 (twice as many pixels horizontally, then twice again vertically for
4x).
Using the same amount of texels would lead to geometry having sharp edges and
blurry surfaces, both contributing to an outdated look.

Upscaling techniques, such as DLSS, FSR, and XeSS have a complex interaction
with VRAM usage.
On one hand, they reduce rendering resolution below display resolution, which
would imply less texture detail required and therefore less VRAM used, but on
the other hand, to compensate for this, they also tend to use textures that are
"too detailed" (having a negative mip bias) for this reduced resolution in order
to have a sharper source image for upscaling, which increases VRAM usage.
These effects add up to an overall reduction in VRAM use.
Claims that these technologies somehow "compress the framebuffer" to explain the
reduction in total VRAM use make no sense.[^fbc]

[^fbc]: There are compression algorithms for framebuffers.
    They are mainly used to reduce power consumption when transferring data
    between different parts of a chip.
    Games are not even aware of them being used.

# Texture compression

The vast majority of textures used by games are compressed in VRAM.
They use special compression
[algorithms](https://docs.unity3d.com/Manual/class-TextureImporterOverride.html)
that are designed for slow compression, but nearly-instant decompression and
random access while compressed.
Compression _increases_ performance by reducing the VRAM bandwidth required to
read the same amount of texels from a particular texture:
the smaller a texture is after compression, the faster it is to use.
This is the opposite of what happens with traditional compression algorithms.

Texture compression is done before the game even ships; your computer only sees
the compressed data.
As a result, they also load faster from disc than uncompressed textures would.
Framebuffers require fast random access for both reading and writing, so they
are not compressed.[^fbc]

# Ray tracing

Ray tracing also contributes to higher VRAM usage.
In order to quickly figure out where and how rays bounce, the GPU needs
additional acceleration data structures stored in VRAM on top of everything else
already needed for rasterization.[^dxrplug]
The size of this extra memory loosely corresponds to the geometric detail found
in the game scene (more objects, more triangles ⇒ more VRAM) and is unaffected
by texture resolution or quantity.

[^dxrplug]: I wrote
    [a tutorial](/2023/02/18/dxr-tutorial.html#raytracing-acceleration-structures)
    that covers the basics of creating and maintaining raytracing acceleration
    structures in memory.
    It also features a small permanently-mapped buffer used for animation.

Ray tracing requires significant VRAM bandwidth in addition to the higher usage
to perform look-ups in these structures.

# Memory transfers, PCIe bandwidth and latency

In practice, it's not possible to perfectly assign everything to RAM or VRAM and
keep it there.

The location of objects within a game is a major crossover: the CPU needs to
know if, e.g., an enemy is within line of sight to the player to determine if it
should alert other enemies, it needs to apply your input to move the camera with
you, but the GPU also needs to know where these are in order to create the
correct visual representation of the scene on your screen.
Since these change constantly, data needs to be copied from RAM to VRAM very
frequently.
Most of the transfers are in this direction, but there are a few techniques that
require CPU access to data originating from the VRAM.

This kind of transfer is low volume and highly sensitive to latency.
It's more or less impossible to perform these updates and use the updated VRAM
within the time allotted to a single frame, so in most games the GPU works to
render the _previous_ frame that the CPU has already finished calculating.
Some game engines delay rendering by two frames even.

While there's always _something_ transferred in each frame to do any kind of
animation, not all data needs to be updated that often.
The wooden texture of a barrel will stay exactly like that for many frames and
can be reused without transferring it to VRAM over and over again.
Using the texture will also involve loading it from disk (hopefully an SSD, but
it could be very slow optical media), so game engines are prepared for very high
latency spanning multiple frames before the transfer is finalized.

Most modern games have far more visual assets that can fit into VRAM at the same
time.
If you move somewhere else where there are no barrels but there are flowers,
the VRAM area might be repurposed to contain the newly-loaded textures of the
flowers, and the barrels' textures might need to be copied back into VRAM
somewhere else if you look at flowers next to barrels.
This is mainly sensitive to RAM and PCIe bandwidth (the textures were already
loaded from disk for the barrels' first appearance), but it consumes VRAM
bandwidth to perform the copy, which is otherwise also heavily used to read the
enormous amount of texture data already present in VRAM.

One of the many methods to lessen the impact of texture loading being slow is to
load them mip by mip instead of all at once, so that the game can continue to
run smoothly with less texture detail.
This is what causes some games to display blurry textures for a few moments if
you rapidly turn the camera 180°, especially if the game is already struggling
with low VRAM.

Some game engines never unload the smallest few mip levels, to always have a
fallback.
These will only account for overall megabytes, but it makes developers' lives
much easier if it's guaranteed that there will be some kind of texture
available, even if very blurry.

The other extreme is a game that prioritizes visual fidelity over FPS, in which
case the effect will be significantly reduced FPS for a few frames but things
still looking great.
This is not a binary choice, and many games will be somewhere between these two
extremes.
Game developers have to make a call in the end on what to do if full quality at
full FPS is not possible.
The correct answer is often determined by the game's genre.

# MMIO

Although PCIe is packet-based, data is sent across very differently than, e.g.,
downloading a .jpg file from the Internet.
Instead of asking for some data ("give me barrel_01_d.dds"), PCIe packets can
contain instructions to read from or write to memory addresses, and load data
exactly where it will be used.[^msi]
For maximum speed, these packets are understood by dedicated hardware
controllers, instead of software parsing them and manipulating memory
accordingly (which is how HTTPS works for those .jpg files).

[^msi]: This is not all they can do.
    Notifying each other of certain events using a mechanism called interrupts
    is also very important, but not relevant to this article.

Memory controllers (your CPU and GPU both have one) can be programmed to
understand special address regions as being mapped to a device instead of RAM
using a technique called
<abbr title="Memory-Mapped Input/Output">MMIO</abbr>.
Accesses to these special regions will never reach RAM, but they will generate
PCIe packets with the corresponding instructions, such as "read 4 bytes from
0xffffa1ee73f108b0", or "starting at 0xc000, write the following bytes:
162, 0, 138, 157, 0, 4, 232, 208, 249, 96".
<!-- You found the Easter egg, enjoy your well-deserved SYS49152 :) -->
This greatly simplifies hardware access from a programming perspective, since
it's handled almost exactly like other data.

GPUs (mostly) expose these special address regions through PCIe
<abbr title="Base Address Register">BAR</abbr>s.
A PCIe device can have up to 6 BARs (BAR0-BAR5), but for simplicity, let's
assume there's only one.
Having more doesn't change the fundamentals, it's just a different way to
organize some numbers.
While a BAR is in fact a register, for this discussion we'll treat a BAR as
simply a range of numbers, such as 0xf3000000 to 0xf3ffffff.
The real range is assigned to the device very early when it boots.

For many different reasons, the size of a GPU's BAR was limited to 256 MB for a
long time, and still is on many systems in active use.

The GPU receives PCIe packets describing reads or writes to addresses as
described above, and it has complete freedom in how it interprets these
packets.
Usually, a small region within the BAR (or the entirety of the BAR) is used for
special addresses, that when read from or written to control the hardware
directly.[^volatile]
In a sense, these addresses represent commands instead of a location, and
pretending that something is being read or written is only done for MMIO.

[^volatile]: This is the true use of `volatile` in C/C++.

For something a little more tangible, let's say that Example® GPU, Mk I
interprets 2 bytes at 0x0180-0x0181 in its BAR0 as the first connected display
output's desired horizontal resolution, 0x0182-0x0183 as the vertical
resolution, and writing the number 1 into 0x0184 causes the hardware to switch.

This is how setting the first screen to 1920x1080 would look if BAR0 was mapped
to 0xf3810000 on the CPU:
```c++
*(volatile uint16_t*)0xf381'0180 = 1920;
*(volatile uint16_t*)0xf381'0182 = 1080;
*(volatile uint8_t*) 0xf381'0184 = 1;
```

Some of these addresses could be even more abstract: in our example GPU, reading
from 0xf38100f6 could start running vertex shaders based on parameters stored
elsewhere, without producing a meaningful value.

Games are strongly isolated from these low-level specifics.
Even if they knew the addresses that were mapped to the GPU and the exact model
to know how to interpret these, they still couldn't use them due to virtual
memory; attempting to do so would cause a crash.
These address ranges are so special that even looking at them requires special
privileges only granted to drivers and the OS itself.

Instead, games use a standardized API such as Direct3D, OpenGL, Vulkan, or Metal
to issue higher-level commands ("copy this texture to that texture", instead of
"send this data to that PCIe address") to the driver.
This has the additional benefit of games working on GPUs that are released after
they were written (including games that were written before PCIe existed!),
something that we now take for granted, but it wasn't always the case.

Even on consoles, where GPUs will not be upgraded without releasing an
entirely-new generation, there are similar higher-level facilities: libGNM, AGC,
or NVN perform similar tasks.
Details on these are sadly behind non-disclosure agreements.

# Resizable BAR

BARs being limited to 256 MB sounds like a serious problem, since VRAMs are
multiple gigabytes in size.
The impact of this limitation is relatively mild in practice: individual copies
seldom exceed this size, and even if they do, drivers can break them up into
multiple smaller transfers behind the scenes.
It's also possible for PCIe devices to access certain regions of RAM directly,
outside a BAR region.

Not all VRAM operations are simple copies, though.
Sometimes, a region in VRAM is made accessible for the CPU for richer (and less
efficient) random access than a simple copy.
This is called locking or mapping that area of VRAM.
Traditional APIs (OpenGL, Direct3D up to 11) lock a GPU resource out of being
used by the GPU when it's mapped, but using modern APIs such as DX12, a resource
can be mapped and used by the GPU at the same time.[^dxrplug]
The main overhead comes from either having to shuffle things in and out of this
256 MB window[^barsched], or being unable to map additional GPU resources
without unmapping others first.
Unmapping can have surprising hidden costs: drivers can optimize to make mapping
quicker by deferring some operations until they're _really_ needed, when the
resource is unmapped.

[^barsched]: With multiple programs running, it's possible that BAR space is
    overall exhausted, but no individual program has reached the maximum.
    In this case, resources can be shuffled in and out of BAR space to keep all
    of them running, albeit slower.

Resizable BAR (sometimes abbreviated to ReBAR) is a supplementary part of the
PCIe specification that's now heavily marketed for consumer GPUs, mostly by AMD
using the brand name Smart Access Memory.
It's ancient tech: version 2 of its specification was released in 2008![^rebar]

[^rebar]: <https://pcisig.com/specifications/pciexpress/technical_library/pciexpress_whitepaper.pdf?speclib=resizable>

Combined with "above 4G decoding", which essentially lets BARs have addresses
above 0xffffffff, it makes it possible to extend a BAR to cover the entirety of
VRAM, eliminating all practical limits on what can be mapped and the associated
overhead.

It doesn't "unlock the full PCIe bandwidth", "expand the data channel", or "let
you utilize the full VRAM".
It does not affect bandwidth at all, the channel is as wide as the wires that
physically exist on the video card make it, and the full VRAM was already
utilizable, as evidenced by multiple generations of video cards having GBs of
VRAM without ReBAR support.
These claims are disingenuous, misleading, or simply false.

Mitigations for the 256 MB limit work _really_ well.
Drivers optimize when and what to map to BARs for the best performance, and
years' worth of games were written to not go over this limit.
In a lot of cases, the games didn't even have to do anything about this:
AMD's and NVIDIA's mature and established architectures and drivers get minor
benefits at best from ReBAR, sometimes even reduced performance!
NVIDIA notably disabled ReBAR behavior for _Horizon Zero Dawn_ in a driver
update recently,[^nvhzd] counterintuitively making the game run faster.

[^nvhzd]: <https://www.nvidia.com/en-us/drivers/results/200383/>

This doesn't mean that the game was somehow bad and deserving of such
punishment:
any overhead caused by a limited BAR was already eliminated without extra help.
Although, ideally this should not be done as an "is this game one that I know?"
check, but the true cause of the slowdowns fixed and applied generally to all
games.
Smaller studios don't benefit from this kind of special attention paid to their
games, and they can't really do anything about this other than attempting to
change their pattern of transferring data between RAM and VRAM, hopefully
managing to move away from the bottleneck.
For most indies, this is practically out of reach, as they're using ready-made
engines without the skillset to deal with issues this low level.

Standing in stark contrast to the big 2 GPU vendors, Intel's Arc
architecture—being relatively new and immature—all but requires ReBAR to even
work properly.
Optimizations are either missing from its driver and/or less effective than its
competitors', in particular its memory controller scales badly with the number
of operations but not the size of them.[^arcrebar]

[^arcrebar]: <https://www.youtube.com/watch?v=-hncVQKXjR8>

# VRAM usage vs allocation

Few topics have as much confusion surrounding them as this one.
If you work your way backwards, observing games and trying to draw conclusions
on how they manage VRAM, it's not hard to come up with an incomplete and wrong
model that mixes together multiple different strategies and attempts to explain
them in a common logical framework that was never there to begin with.

Usage is perhaps the simpler of these two.
The VRAM that's absolutely **required** for a game to run is the sum of the
resources used by its most expensive draw call, compute dispatch, or command
list[^vkcmd] execution.
Simply put, these are the smallest units of work that a GPU can be asked to
perform, and all data that they use must be present in VRAM.
They cannot wait for additional data to arrive in VRAM once they
start.[^preemption]

[^vkcmd]: Vulkan calls them command buffers.
    They are buffers, but that term is overloaded enough already, so I'm going
    with command lists.

[^preemption]: It's Not That Simple.™
    There's GPU preemption, and command lists can be broken down into multiple
    smaller command lists.
    Drivers won't bother with the latter; this responsibility shifted to the
    game in DX12 and Vulkan.
    OpenGL display lists have different semantics than command lists in
    low-level APIs and are not affected.

Even the most complex AAA scene full of highly-detailed textures and models will
not strictly require more VRAM than the one single FPS destroyer boss standing
in front of you that uses more VRAM than any other object in the scene.
This will be far less than a single gigabyte of VRAM in practice, and games have
control over this requirement, e.g., by reducing the number of mip levels used.

Even if this somehow hits 100% VRAM usage, data can be shuffled in and out of
VRAM to prepare for the next draw call, then the next one, and all of them will
be satisfied.
This shuffling has a massive cost, though, and is very likely to get your FPS
below enjoyable levels.
Therefore, the required amount of VRAM _to run at an acceptable FPS_ will be
much higher: the sum of every resource used frame after frame to minimize VRAM
transfers.

If a game refuses to run because you don't have, let's say, 4 GB of VRAM, that's
a manual check put in by the developers to reduce complaints to customer
service.

If it _crashes_ because of that, that's the game engine's fault.

## VRAM in legacy APIs

Games didn't have direct control over what data was in VRAM in legacy APIs
(OpenGL, DX11 and below).
If they asked for a texture, they did so with the expectation and intent that it
would end up in VRAM, but this was (and still is) controlled by the GPU driver.
Games could only communicate their intented use of the resource that influence
the driver's decision on how to manage memory for it.

What is mapped to a BAR or not is completely up to the driver, and this often
changes dynamically as resources are used, especially on a pre-ReBAR system.
Whenever a draw call or similar is issued, the driver makes sure everything that
it uses is _resident_ in VRAM first, before telling the GPU to actually start.

Trying to not go over maximum VRAM and avoid swapping to RAM is also mostly
done by heuristics with these APIs:
games can query the OS how much total VRAM is in the system, how much is already
used, etc., but they can only change what they do with their resources in
response, hoping that it keeps VRAM use in check (although in practice this
determines VRAM usage well enough).

This lack of direct control is _great_ for overall system stability.
It's also not optimal, since drivers do not have foreknowledge of what resources
will be used and when by the game engine, so they have to determine how to
reduce VRAM transfers by their own generic heuristics.
Ideally, a resource should be resident in VRAM before the draw call that uses
it is issued, not made resident in response to it.

## VRAM in modern APIs

Modern APIs (especially DX12) provide explicit control over what's in
VRAM.[^vkmemory]
Although games still can't directly request something to be mapped to a BAR,
DX12-style Map() (being available for CPU _and_ GPU access) essentially requires
this, so in practice they can.

[^vkmemory]: Vulkan is not as explicit as DX12, so I'll leave you with
    `VK_EXT_memory_priority`, `VK_EXT_memory_budget`,
    `VK_EXT_pageable_device_local_memory`, and focus on DX12.

Memory is also managed differently: VRAM is allocated in larger, coarse blocks
("heaps") that are then subdivided for the individual resources.
A heap could contain 27 textures and 31 meshes, and from the game's perspective,
they're either all in VRAM or none of them are.
This is bad when you only need one of them, but if the game manages its heaps
well enough to avoid such situations, this reduced granularity can increase
performance:
instead of tracking the VRAM residency of 58 separate things, there's only one
large block of memory.[^arcrebar2]

[^arcrebar2]: This is one of many reasons why Intel Arc performs better using
    DX12/Vulkan than DX9/DX11/OpenGL.

This is what's commonly referred to as VRAM allocation when it's used in
contrast to VRAM usage.
A game could be programmed to always grab 512 MB heaps of VRAM, then put as many
resources in it as possible, then allocate the next 512 MB, etc.
Heaps are also divided based on their usage, so even if there's space in one
heap intended to store textures, a buffer used by compute shaders might need to
be placed in another one.
In a relatively bad scenario, this could mean that an entirely new heap might
need to be allocated even though there was otherwise plenty of free space.
This further increases the amount of VRAM allocated by games using modern APIs
compared to older ones.

Separating these two values from an end user's perspective is not very
productive.
If a game uses 32 MB of VRAM in a 256 MB heap, you will need 256 MB of VRAM for
that heap to be usable at all.
The 32 MB value is completely internal to the game, and the rest being
technically unused isn't going to help with your FPS.

Modern APIs also submit work to the GPU in batches called command lists[^vkcmd]
to increase efficiency.
This is one reason why a DX12 draw call is much cheaper than a DX11 one.[^pso]

[^pso]: Monolithic pipeline state objects are another large contributor.

A command list having multiple commands in it increases the amount of data that
must be present in VRAM at once compared to individual calls in legacy APIs:
for example, if a command list contains 4 draw calls, with the first three
requiring heaps A, B, and C to be present in VRAM, and the fourth using A, B,
and D, all four heaps will need to be in VRAM before the first draw call can
start.

Once again, games need to make sure they manage this well so that they benefit
from the increased efficiency and not end up worse than when this was the
driver's job.
Especially in the early days of DX12, it was not rare to see games that
performed better in DX11.
Even today, some games still do.

# Memory residency

With modern APIs, it's now the game's responsibility to be an upstanding citizen
and manage its VRAM in light of the entire system's VRAM needs, as opposed to
just its own.
You might have browser tabs open while the game runs, you could be streaming
your gameplay online, it's entirely possible that you left a 3D modeling program
running before you started your game, or it's rendering and you want to play
something to kill time...

Sadly, dealing with overall system stability in low-VRAM situations is seldom
high on game developers' list of priorities.

DX12 resources start as _conceptually_ being locked into VRAM[^heapflag], and
there are two straightforward methods to mark a resource (in practice, a heap)
eligible to be moved out of VRAM if something else needs the space
([ID3D12Device::Evict](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-evict)),
or to bring it back if it was moved out
([ID3D12Device::MakeResident](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-makeresident)
or [ID3D12Device3::EnqueueMakeResident](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device3-enqueuemakeresident)).

[^heapflag]: There's an opt-out: `D3D12_HEAP_FLAG_CREATE_NOT_RESIDENT`

Most calls to Evict and MakeResident will do nothing, despite what the name and
MSDN documentation suggest.
[This page](https://learn.microsoft.com/en-us/samples/microsoft/directx-graphics-samples/d3d12-residency-starter-library-win32/)
is more accurate; MakeEvictable would've been a more precise name.
Evict only marks a heap as a candidate for eviction, and Windows will do its
best to leave the heap in VRAM.
In this case, MakeResident will notice that the heap never left VRAM and do
nothing substantial.

This lets a program play nice with other programs running in the system, and
following the theme, even run faster if this is done well.

It is also not what _really_ happens.
Windows has some tricks up its sleeve to make sure the overall system is stable
even if a program misbehaves and doesn't manage its residency correctly.

# Residency example

There's no requirement anywhere that a DX12 resident heap actually has to be
resident in VRAM.
Some DX12 drivers don't even deal with VRAM: iGPUs with shared memory are an
obvious example, but the "Microsoft Basic Display Adapter" (née
<abbr title="Windows Advanced Rasterization Platform">WARP</abbr>) is an even
stronger one:
it's a completely software implementation of feature level 12_1 used as a
fallback if no GPU is present in the system.

Here's a sample program that misbehaves (compile as C++20):

```c++
#include <d3d12.h>
#include <format>
#include <iostream>
#pragma comment(lib, "d3d12.lib")
int main()
{
    ID3D12Device* device;
    D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_12_0, IID_PPV_ARGS(&device));

    constexpr int TOTAL_GB = /*Enter a larger value than your VRAM in GB here!*/;
    for (int i = 0; i < TOTAL_GB; ++i)
    {
        D3D12_HEAP_DESC desc = {
            .SizeInBytes = 1073741824ULL, // 1 GB
            .Properties = {
                .Type = D3D12_HEAP_TYPE_DEFAULT,
                .CreationNodeMask = 1,
                .VisibleNodeMask = 1,
        }};
        ID3D12Heap* heap;
        // Report errors for each allocation: 0x00000000 is success
        std::cout << std::format("{:3} GB: {:#010x}\n", i + 1,
                                 device->CreateHeap(&desc, IID_PPV_ARGS(&heap)));
        // device->Evict(1, (ID3D12Pageable**)&heap);
    }
    // Keep holding every resource
    while (true)
        Sleep(1000);
}
```

Open Task Manager (Ctrl+Shift+Esc), switch to the "Performance" tab, and observe
your GPU's dedicated/shared memory usage when you run this program.

On my system, dedicated usage never goes above true VRAM capacity minus 1 GB,
with the rest being put into shared GPU memory (regular RAM assigned to the GPU
through PCIe, very slow).
Even with all of my VRAM theoretically being allocated and locked in place, I
have no trouble opening a very complex scene with Unreal Engine at full
performance while this program is running.

Its heaps were moved out from VRAM to make space for Unreal's new allocations,
and if this program was doing any rendering, Windows would make sure they're
back in place (and move Unreal's heaps out) before it would continue to execute,
pretending that they were always resident.
Doing so is **very** inefficient, and depending on the amount of VRAM that was
overcommitted, it can easily bring even the most powerful systems below 10 FPS.

You can observe sampleprogram.exe's private working set increasing in Task
Manager (Details panel) when Unreal launches, despite it doing nothing but
Sleep() by that point.
If you uncomment the line containing Evict, shared GPU memory will not be used,
however general RAM use will still increase by as much as you're overallocating.

# Summary

Modern games use more VRAM than older ones did.
The vast majority of this additional usage is caused by textures and other
graphical resources being more detailed, but modern APIs are also a factor.
Framebuffer size is not the cause of high VRAM usage, but it correlates with it.
Upscaling techniques also push for more VRAM usage compared to rendering
natively at the lower, internal resolution.

Games are in control of their own VRAM usage, and can greatly reduce how much
they need using various techniques.
Some slowness in return for high-resolution assets can be acceptable depending
on the genre, but it should be a playable framerate.
If a game goes over total VRAM and runs very slowly or crash as a result, that's
almost invariably an issue with its engine.

Games using modern APIs have more responsibilities to manage VRAM than they did
with legacy APIs.
This provides opportunities for good games to run even better, but also
opportunities for bad games to run even worse.
