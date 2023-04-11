---
title: Compensating for Hearing with Basic Signal Processing
date: 2023-04-13
description: A look at how I came to compensate for my hearing when I am not wearing a hearing aid.
tags: ["Signal Processing", "Audio", "Equalization", "PipeWire", "Linux"]
type: blogpost
draft: true
---


## Motivation

As a 

My headphones are are pair of [<samp>WH-CH710N</samp>](https://electronics.sony.com/audio/headphones/all-headphones/p/whch710n-b) produced by Sony Electronics, an over-ear design featuring [Active Noise Cancellation](https://en.wikipedia.org/wiki/Active_noise_control) (<abbr title="Active Noise Cancellation">ANC</abbr>). I use these nearly every day on average, and my only gripes with them is a lack of multi-device connectivity[^1] and the inability to use them while charging.

[^1]: It seems like this choice is intentionally made for the purpose of segmentation: Both of the flagship Bluetooth headphone models sold by Sony Electronics, the older [<samp>WH-1000XM4</samp>](https://electronics.sony.com/audio/headphones/headband/p/wh1000xm4-b) and newer [<samp>WH-1000XM5</samp>](https://electronics.sony.com/audio/headphones/headband/p/wh1000xm5-b), do support multi-device connectivity—but the [<samp>WH-CH720N</samp>](https://electronics.sony.com/audio/headphones/headband/p/whch720n-b) (the replacement for the <samp>WH-CH710N</samp>) does not. Considering that other (and cheaper) Bluetooth headphones from other vendors do have this feature, I find it hard to believe it is not artificial segmentation.

## Filter Construction

To construct my filter, I chose to use a parametric equalizer composed of 32 peaking-biquads equally spaced across the human audible band between 20 Hz to 20 KHz. 

### Profiling hearing 

## Productionalization

Creating a filter is good and all, but my motivation is to actually use it in my daily life. 

### Impulse response

One of the most accurate ways to productionize a filter is to use its <dfn>impluse response</dfn>.

The unit impulse (in continuous-time the Dirac Delta function) represents a signal with a unique property: It contains all frequencies at equal intensity. By simulating the filter's response, we recieve a new signal that, by the convolution thereom, can be convolved other signals to result in the same signal that would have existed if we had used the orginal filter on those other signals.





### Filtering audio ahead-of-time

One of the things we can do with our productionized filter is apply it to audio ahead-of-time. This lets us apply our filter to audio and playback on any suitable system.



### Linux Audio and PipeWire real-time filtering

#### The PipeWire Filter-Chain Module

One of PipeWire's numerous modules is the [Filter-Chain module](https://docs.pipewire.org/page_module_filter_chain.html), which provides a whole host of useful capabilities for real-time filtering.



<section>

## Mixing across channels for improved intelligibility

By this point, the designed filter is productionized and adequately fits the needs of my daily life. However, I found something irritating enough to not be completely satisfied.

<section>

### Polyphonic sound, Mixing, and Headphones

<dfn>Polyphonic sound</dfn> is one of many core innovations to the advancement of audio reproduction technology. All polyphonic sound does over monophonic sound is what its eytmology describes: Expand the audio signal from one spacial dimension to multiple spacial dimensions.

It doesn't seem too complicated on the surface beyond having to deal with the extra overhead of more than one spacial signal dimension. However, ask a mastering engineer about their job, and they will tell you all sorts of intricacies they need to keep in mind when dealing with polyphonic sound. Ask them about headphones, and they might tell you that they are (for lack of better description) a curse upon this world—accounting for headphones adds a layer of complexity to their jobs that they rather not need to account for, and when headphones are expected to be used with a statically-mixed audio track, there is a requirement for compromise.

This is because what seems simple in the ideal does not match reality. A mastering engineer's job is to make a track reproduce as well as they can, regardless of the equipment used. The problem is that equipment for reproducing sound from a recording is in no way consistent, with wide variance in quality and configuration—which are often-times intentionally tuned to be objectively inferior for the purpose of subjective superiority! This problem exists for monophonic sound, but adding additional channels greatly expands this configuration space.

The issue with headphones is that they represent a completely different type of configuration than standard stereophonic speaker setups; headphones have the property that the sounds of each channel are *isolated* from each other, with each ear receiving its own signal without interference. In contrast, when a single stereophonic channel produces a sound in free space, *both* ears receive the same sound with a delay slightly out of phase from each other (dependent on positioning of that channel), and when both channels are producing sound, they interfere. **This means that listening to a stereophonic track with headphones is not equivilant to listening to the same track through speakers.**

It happens to be that the better compromise when statically mixing audio is to mix for speakers, not headphones. This means that certain information may only exist on a subset of channels, and in the case of stereophonic sound, exclusively exist on one channel. This brings an issue to be solved—how to make this information available to both ears when the listener wants or needs it.

Patients with unilateral hearing loss often need to lean on their good ear to intelligible complex sounds like speech, and I am no exception—I will often turn my head to face a speaker on my left and ask the speaker to repeat themself if the intelligibility of their speech is low.[^2]

[^2]: This is a widely recognized phenomenon by designers of assistive systems for the unilaterally hearing impaired, and so the hearing aid manufacturers catering to this subset of patients already have a solution for this: Sell them two hearing aids, and configure the hearing aid on the weaker side send audio to the stronger one. This of course makes little sense for users who have one side with reasonable hearing (like me), but for users who have a weaker strong ear and disparity between ears, this solution is excellent and is a no-brainer solution.


</section><section>

### Mono Audio (and its problems)

The poor-man's solution to this is to simply mix stereophonic channels together equally into a monophonic channel shared on all outputs. On most computerized systems, this can be achieved by toggling the <dfn>mono audio</dfn> feature in the accessibility settings.

While cheap, this sort of mixing results in a major loss of information. By collapsing the stereophonic channels into a monophonic channel and duplicating them across both stereophonic channels, the listener looses all ability to locate sounds with respect to the stereo axis—much to the frustration of those properly mastering such audio everywhere who likely meticulously crafted that information.

</section><section>

### Virtual Surround

We can compensate for this with <dfn>virtual surround</dfn>—simulate how the audio would sound if the listener was in an open environment with polyphonic speakers. This sounds like a tall order—simulating the audio from speakers, how they interact with the environment and the listener… But we have already learned the secret to this by now: Yep, that's right, basic filtering!

A <dfn>Head-Realated Transfer Function</dfn> describes how 


Virtual surround also provides other benifits. Mono Audio for 

<section>

#### Virtual Surround in the PipeWire Filter-Chain

It is possible to implement Virtual Surround in the PipeWire Filter-Chain—in fact, it is one of the examples provided on how to leverage the module. However, documentation on how to acheive this is quite sparse. Luckily, it isn't that complicated.

</section>

</section>
</section>

## Conclusion

For me, this results in the track being mostly (if not entirely) unintelligible. 



