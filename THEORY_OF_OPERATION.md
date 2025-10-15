# Theory of Operation

### Background

Part of the ethos of DIY is understanding what you're working with, and so I will attempt to describe how this reverb algorithm works so that it is not just a monolithic wall of magic numbers.

The factory FV-1 reverbs are allpass loop topologies comparable to those found in later Alesis devices. Many public domain programs are derived from the Lexicon-ish plate topology described in [Jon Dattorro's 1997 paper "Effect Design, Part 1: Reverberator and Other Filters"](https://ccrma.stanford.edu/~dattorro/EffectDesignPart1.pdf). These topologies work by recirculating a single delay line which continuously builds up echoes by interrupting it with many allpass diffusion stages. The FV-1 chip was designed to make it very straightforward to build these types of topologies, including instructions specifically designed for computing Direct Form II allpass filters.

The Kiloverb algorithm is a [Feedback Delay Network](https://ccrma.stanford.edu/~jos/pasp/History_FDNs_Artificial_Reverberation.html), or FDN. An FDN topology builds up echo density by recirculating several delay lines in parallel and continuously mixing them together with a mixing matrix. The basic form of an FDN and the logic behind it is described in many places so I won't dwell on the fundamentals here. However this algorithm uses a few extensions to the basic concept which are not particularly well known.

Every reverb designer has a particular goal for each algorithm: it should sound like a spring, or be dark and cavernous, or work well with piano, or whatever. The overarching theme with this algorithm is to attain levels of echo density that are seemingly impossible within the constraints of the FV-1's limited memory (32K samples at 32kHz, or roughly 1 second) and program length (128 instructions per sample). The sparseness in many FV-1 algorithms can be charming on a lot of sources, but can also be obnoxiously grainy on material with distinct transients—instead of a smooth wash, you hear individual echoes like popcorn popping. Another reason extra density is necessary is that this algorithm is designed to run on a pedal with variable clock rate. At lower clock rates, a sparse reverb becomes really sparse. 

### FDNs

The basic unit that this topology uses is a 4 channel FDN with a [Hadamard mixing matrix](https://ccrma.stanford.edu/~jos/pasp/Hadamard_Matrix.html). This is a textbook FDN structure. The Hadamard matrix is special because it's orthogonal (preserves energy, preventing buildup or decay) and computationally efficient. By itself, this 4 channel FDN produces a stable but chattery and metallic reverb. It is barely usable for anything.

The next stop for most designs is to make a bigger FDN. The first FV-1 reverbs that I wrote were 8 channels wide. They sounded better than 4 but still only OK. The core constraint is that the matrix computation increases in complexity quadratically with the number of channels. A four channel matrix requires 4² = 16 multiply and accumulate instructions (MACs). An eight channel matrix requires 8² = 64 instructions—that's already half our instruction budget! There are some algebraic tricks with the Hadamard matrix to bend that curve from quadratic to logarithmic, but we still have the constraint that the FV-1 has only 128 instructions per sample and we just can't fit a very large, fully connected matrix.

Here's another way to think about this computation: every time an impulse travels through N delay lines and a matrix of size N×N, that single impulse becomes N impulses. So every loop through the system multiplies the number of reflections in the reverb by N. With an 8 channel FDN, you are using 64 MACs to produce 8 reflections. On the next trip through, that 8 becomes 64 and then 512 and so on—geometric growth.

Luckily, there are a few ways to cheat this math!

### FDNs in series

The textbook FDN routes delay lines into its mixing matrix and then the outputs of that matrix always immediately return to the same delay lines as feedback—a simple closed loop. However, this isn't the only way. We are going to use four of these FDN structures and route them all in series. So the outputs of matrix #1 go into the delays of #2 and so on. At the end, it will feed back into the first set of delays. You can visualize it as four stages of delays, interrupted four times by mixing matrices, all connected in a chain that eventually loops back to the beginning. The overall topology is just as numerically stable as its simpler prototype because the stability criteria (the eigenvalues of the mixing matrices) don't change.

This structure is incredibly effective and it straightaway fixes most of the problems that people have tuning FDNs to sound good. It's a long topic to explain why, but going through multiple series structures of different delay times just sounds much better than continuously looping through a single one. The modal structure of the reverb becomes much more complex—think of it like adding more rooms to a building, each with different dimensions, versus making one room bigger.

And what if we revisit the math from above? Each 4 channel FDN requires 16 MACs. Four of these matrices require 64 instructions, which is the same amount as the 8 channel version. But in terms of reflection density, we are getting 4 × 4 × 4 × 4 = 4⁴ = 256 reflections per feedback loop—versus just 8² = 64 for the single 8-channel approach. This demonstrates why series structures are so much more efficient than higher order structures in the raw terms of building up density. 

For a more comprehensive discussion of this and related concepts I recommend S.J. Schlecht, et al. [Scattering in Feedback Delay Networks](https://acris.aalto.fi/ws/portalfiles/portal/44331990/ScatteringFDN.pdf).


### Scattering

I was first exposed to this idea in the paper by S.J. Schlecht, et al. [Dense Reverberation with Delay Feedback Matrices](https://ieeexplore.ieee.org/document/8937284), which is unfortunately behind a paywall. The core concept is also discussed in their open-access paper linked above.

In a traditional FDN, the end of the delay line is read and that same sample value serves as the input to all four channels of the mixing matrix. But intuitively, imagine we could read four different locations in the delay line instead of using the same one four times. Wouldn't the echo density be much higher? The answer is yes but randomly choosing these times would cause the structure to become unstable—the energy preservation property would break down. However, the paper proposes a formulation in which we do get to do that and the structure maintains stability. The trick is that all of the output taps must be spaced uniformly. If delay line 1 reads at times [100, 120, 140, 160], and delay line 2 reads at times [200, 220, 240, 260], notice both use a spacing of 20 samples. Looking a little deeper, you can see that this structure is essentially equivalent to going through another set of delays and a whole additional matrix—except we get it for free!

The tradeoff is that we have to read each delay line in four different places. In many digital systems, memory reads are slow compared to multiplications and so it's not actually a very good tradeoff. This probably explains why the technique isn't more widely considered. However, in the FV-1 a memory read is the same cost as a MAC in the matrix calculation. Also reading four different samples costs the same as reading the same sample four times. And so this optimization is completely free in the FV-1 instruction set. So it appears that we've completely cheated space and time itself. Our topology of four series FDNs is actually capable of computing the equivalent of eight series FDNs using this structure.

To return to the (admittedly over-simplified) discussion of reflection density above, this is now 4⁸ = 65,536 (or 64k) reflections per trip through this network.

I call this "scattering" (as do the paper authors) since this additional free set of reflections is best used with the scattering delays set to tiny sizes compared to the main delay sizes. The main delays determine the sense of space ("this sounds like a 20-foot room"), while the scattering delays add diffusion and smoothness without changing the perceived room size. Think of it as the difference between smooth plaster walls versus the same sized room full of bookshelves and statuary.

The colab notebook implements some functions which calculate all of these scattering times. I found that it worked best to have them evenly spread over a range of values (akin to "velvet noise" or jittered sampling) rather than completely random. This creates a more uniform distribution of early reflections, which shapes the scattering to be wider and smoother.

This approach is effective enough that it produces structures that are too dense for some applications where a sparser reverb might be more appropriate. So I wrote versions of this algorithm that use scattering this way (the "dense" ones) and those that do not (the "sparse" ones).

### Choosing delay times

With the raw density problem solved, we'll turn to the more aesthetic considerations of how the ambiance should be shaped.

I had a goal for this algorithm to be able to do both "plate style" and "room style" reverbs. Here's how I think about the differences between these:

*  **Plates** are as dense as possible as quickly as possible. No individual reflections are discernable and there is no sense of a natural space. The envelope is explosive at the beginning and trails off exponentially. This matches the behavior of actual plate reverbs, which are thin metal sheets that vibrate in complex modes.
*  **Rooms** (or halls) begin with discernable initial reflections and then have an audible build up in the reflection density. The reflections, density and decay time give the impression of some natural space. The envelope has some fade in, proportional to the size of the space—a larger room takes longer for sound to bounce around and build up.

In this algorithm, the choice of delay times is the main factor that differentiates these shapes. It causes them to sound quite different.

*  In the plates, there is a very even mix of delay times spanning a wide range. This causes there to be no clearly discernable pattern of reflections—just immediate dense texture.
*  In the rooms, there is a very distinct pattern. Two of the four FDN stages use delays sized so that they roughly correspond to the overall dimensions of the simulated space (e.g., a 30-foot room might use delays around 30ms, since sound travels roughly 1 foot per millisecond). The other two stages use much smaller delays, and they primarily contribute diffusion rather than a sense of space.

The colab notebook implements the algebraic shenanigans that calculate the desired relationships and then fit it into the available memory.

### Input and output taps

In an FDN structure, the input signal can be injected almost anywhere into the system, and the outputs can also be taken from almost anywhere. There is a lot of space to explore here. One of the main differences between the different algorithms is where the input and output taps are placed.

*  The more input taps, the more diffuse the initial output. Injecting into all 16 delay lines (across 4 FDN stages) creates instant density, while injecting into just one creates a sparser initial response.
*  The more output taps, the more overall density in the output. Summing all 16 delay lines gives maximum thickness.
*  The more delay time between the input tap point and the output tap point, the more pre-delay before any output is heard. This type of pre-delay tends to sound more natural than just delaying the input signal before the reverb, because the reverb tail starts building up during the pre-delay period.

The output taps are summed and then they go through a waveshaping function which has a soft-knee saturation characteristic. This makes the output sound fuller and have more balanced dynamics—loud transients are gently compressed while quieter tail components remain clear.

### Clock manipulation

This algorithm is designed to run on hardware with a variable clock rate. The FV-1 is normally set by a crystal to run at 32kHz, but it can also be given a digital clock which can slow its sample rate considerably. These low clock rate reverbs sound really cool. They get very dark and the space sounds enormous due to stretching out all the delay lengths. For example, running at 16kHz instead of 32kHz doubles all the delay times, making a "20-foot room" sound like a "40-foot room."

One consequence of this is that filtering becomes complicated. Most reverbs need some tonal shaping to sound realistic. However in a variable clock rate system, the filter frequencies also change with the clock rate, and the real-time computations which would be required to recalculate their coefficients are somewhat beyond the capability of the processor. To sound their best, these programs should have some analog filtering around them to roll off the extreme high and low frequencies.



