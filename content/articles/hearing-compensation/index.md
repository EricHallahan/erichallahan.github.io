---
title: Compensating for Hearing with Basic Signal Processing
date: 2023-04-15T19:00:00-04:00
publishDate: 2023-04-15T19:00:00-04:00
lastMod: 2023-04-15T22:30:00-04:00
description: A look at (and an explanation of) how I came to compensate for my hearing when I am not wearing a hearing aid.
tags: ["Signal Processing", "Audio", "Equalization", "PipeWire", "Linux", "Virtual Surround"]
type: blogpost
---

<section>

## Background & Motivation

As a patient of unilateral hearing impairment, also known as single-sided deafness (<abbr title="single-sided deafness">SSD</abbr>), I have long desired a way to augment myself to compensate for its existence. In my daily life, I am fortunate to have a hearing aid to do that for me. In my digital life however, it has not been as easy, and despite many attempts of trying to work around it (leaving the left earbud in a tangle and only using the right, adjusting the audio balance, etc.), I never quite found a satisfactory solution—I couldn't help but feel like I was missing something regardless of the bodge I tried.

In the <time datetime="2022-07">summer of 2022</time> I decided to sit down, get my hands dirty, and do something about it. Having quite a bit of experience with signal processing and control systems, I was intimately familiar with the tools that might be necessary to do this. And so I set off to produce a solution that I would find satisfying.

</section><section>

## Filter Construction

For my compensation filter, I chose to simply construct a fixed-band equalizer composed of 32 peaking biquads equispaced and tuned for perceived equal-loudness between ears across the generally-accepted human [audible band](https://en.wikipedia.org/wiki/Hearing_range) of 20 Hz to 20 KHz.

I assume the audience had no ability to grasp most of the words in the preceding statement, and as such I feel the need to provide context as to what any of it means.

<section>

### Introduction to Linear Time-Invariant Systems

Describing a system as being <dfn>Linear Time-Invariant</dfn> (<abbr title="Linear Time-Invariant">LTI</abbr>) indicates two seperate realities about it simultaniously:

1. A system that is <dfn>linear</dfn> is one that is described by a [linear differential equation](https://en.wikipedia.org/wiki/Linear_differential_equation) (in continuous-time) or a [linear difference equation](https://en.wikipedia.org/wiki/Linear_recurrence_with_constant_coefficients) (in discrete-time). Linear differential equations are very basic and, in comparison to other classes of differential equations, have straightforward solutions. 
2. <dfn>Time-invariance</dfn> represents the property that system behavior is static over time (regardless of what input has come before, the dynamics of the system remain unchanged).

[<abbr title="Linear Time-Invariant">LTI</abbr> systems](https://en.wikipedia.org/wiki/Linear_time-invariant_system) are much more convenient to work with than other classes of systems (such as those that are non-linear or time-variant), and it is usually strongly preferable to work with them. Luckily, all systems and filters described in this article will be <abbr title="Linear Time-Invariant">LTI</abbr>.

In the general context of systems, a <dfn>transfer function</dfn> is a function that transforms an input into an output. In the context of <abbr title="Linear Time-Invariant">LTI</abbr> systems, a <dfn>transfer function</dfn> is a representation of such a system as a complex rational polynomial. This rational polynomial describes the behavior of a system in the frequency domain.

<section>

#### Continuous and Discrete-time Systems

Astute readers will have noticed an earlier qualification of a certain distinction: [<dfn>continuous</dfn> and <dfn>discrete</dfn> time](https://en.wikipedia.org/wiki/Discrete_time_and_continuous_time).

<dl>
<dt>Continuous time</dt>
<dd>

A transfer function is related to the time domain by the [Laplace transform](https://en.wikipedia.org/wiki/Laplace_transform), and in terminology, we call the complex domain upon which it operates the [<math xmlns="http://www.w3.org/1998/Math/MathML" display="inline"><mi>s</mi></math>-plane](https://en.wiktionary.org/wiki/S_plane).

</dd>
<dt>Discrete time</dt>
<dd>

A transfer function is related to the time domain by the [<math xmlns="http://www.w3.org/1998/Math/MathML" display="inline"><mi>Z</mi></math>-transform](https://en.wikipedia.org/wiki/Z-transform), and in terminology, we call the complex domain upon which it operates the <math xmlns="http://www.w3.org/1998/Math/MathML" display="inline"><mi>z</mi></math>-plane.

</dd>
</dl>

<aside>

The near-equivalence of differential equations (continuous time) and difference equations (discrete time) has been long-recognized in mathematics and is the basis of [time-scale calculus](https://en.wikipedia.org/wiki/Time-scale_calculus).

</aside>

These two scenarios are deeply interconnected, and with a conformal mapping the <math xmlns="http://www.w3.org/1998/Math/MathML" display="inline"><mi>s</mi></math>- and <math xmlns="http://www.w3.org/1998/Math/MathML" display="inline"><mi>z</mi></math>-planes are (on paper) exactly transformable to one another. This nature is going to be valuable [later](#creating-ir) when applying a filter designed in continuous time to a discrete-time signal (like [digital audio](https://en.wikipedia.org/wiki/Digital_audio)).

</section>
</section><section>

### Biquads & Equalizers

An <abbr title="Linear Time-Invariant">LTI</abbr> transfer function, like all rational polynomials, has a number of complex <dfn>poles</dfn> and <dfn>zeros</dfn> equal to the degree of the denominator and numerator polynomials, respectively. The values (or when plotted on the complex plane, *positions*) of these poles and zeros are critical parameters to describing its behavior.

A biquad filter (as is relevant to this post) is a transfer function that has polynomials of degree 2 in both the numerator and the denominator—hence its name, *bi-quadratic*.

Biquads are quite simple but are also quite flexible—they have a number of useful basic configurations in which they can be applied. One of those configurations is the <dfn>peaking biquad</dfn>, which happens to have a log-magnitude response [resembling a bell curve when plotted](https://en.wikipedia.org/wiki/Bell-shaped_function). Such a peeking biquad can be parametrized by three variables:

* The *frequency* at which it is centered.
* The *gain* that it provides at that center frequency.
* The [*quality factor*](https://en.wikipedia.org/wiki/Q_factor) (also known as the <em><math xmlns="http://www.w3.org/1998/Math/MathML" display="inline"><mi>Q</mi></math>-factor</em>), which controls the bandwidth that it spans.

A low-order transfer function like a biquad is useful in simple situations, but when a number of them are equispaced across the human audible band we would like to control the magnitude of the frequency response over (and manipulating the aforementioned variables for each of them), their usefulness as a basic unit of filter design becomes a bit more clear.

<img src="https://cdn.discordapp.com/attachments/445631164871344139/1096249191375835306/plot_sys_incomplete.svg?ex=65ea6d1e&amp;is=65d7f81e&amp;hm=eaf18e6fbf8132dcade86d3aa5d7b4b871ffd0240012bc1b1623652426e981df&amp;"/>

Now that we have a collection of simple filters, we would like to combine these into something more complex. What is desired is a way to mathematically put them in series, or *cascade* them together into a big filter.

It happens to be that the mathematical operation we desire is simply to multiply the complex rational polynomials of these transfer functions together to yield a high-order transfer function.

<img src="https://cdn.discordapp.com/attachments/445631164871344139/1096249705291321376/plot_sys.svg?ex=65ea6d99&amp;is=65d7f899&amp;hm=3ab9335bb3b7a85704fce07673a8d99a0a7d5651c26bca2e10115e942bed1d79&amp;"/>

What we have done is constructed a <dfn>fixed-band equalizer</dfn>, the same type of one that might be seen on the front of a piece of Hi-Fi equipment. While it is not as flexible as a parametric equalizer, it is straightforward to use given that the center frequencies and quality factors are fixed after their initial selection, with the focus then shifting exclusively to tuning the gain of each band.

</section><section>

### Profiling Hearing and Tuning

As someone with a hearing impairment, I semi-frequently have my hearing tested by an audiologist. These professionally-conducted tests are calibrated in a way that they can not only report the differences in frequency response between ears but also the difference to the calibration baseline and reported in an [audiogram](https://en.wikipedia.org/wiki/Audiogram).

However, in this case, I simply wanted to correct the sound by creating [equal-loudness](https://en.wikipedia.org/wiki/Equal-loudness_contour) across the spectrum. While I might be able to derive relevant information from one of those many audiograms, the reality is that audiograms do not strictly measure equal-loudness.

Even though equal-loudness is not a linear system (with absolute intensity mapping non-linearly to loudness), in the tradition of mathematical modeling of physical phenomena, we are going to assume that equal-loudness contours are linear with respect to absolute intensity and that the [cow is a sphere](https://en.wikipedia.org/wiki/Spherical_cow).

Given the reality that the tuning was going to be highly subjective, my approach was simply to minimize the perceived loudness difference between ears by repeatedly tuning the equalizer gains during sweeps from 20 Hz to 20 KHz. This tuning process took a substantial amount of menial effort to complete, but as I continued to tune it in, the less I felt the need to make changes, until at long last I felt comfortable to spend the effort to solidify it into production.

<figure>
<img src="https://cdn.discordapp.com/attachments/445631164871344139/1096921367720501348/plot_impl.svg?ex=65ecdf21&amp;is=65da6a21&amp;hm=0e607d19a6520d500913fbded25fb9a7f39d686f646cac0ed4fb7447dfe0a98b&amp;"/>
<figcaption><p>Filter as put into production, with 22.4 dB of attenuation applied to both channels to prevent clipping.</p></figcaption>
</figure>

</section>
</section><section>

## Productionalization

Creating a filter is good and all, but my motivation is to use it in my daily life. 

<section>

### Unit impulse response and Convolution

One of the most accurate and easily transferable ways to productionalize a filter is to convolve its <dfn>unit <a href="https://en.wikipedia.org/wiki/Impulse_response">impulse response</a></dfn> on a signal.

In short, the unit impulse (the [Dirac delta function](https://en.wikipedia.org/wiki/Dirac_delta_function) in continuous-time) represents a signal with a unique property: It simultaneously contains *all* frequencies at equal intensity. By simulating or recording a <abbr title="Linear Time-Invariant">LTI</abbr> system's response to a unit impulse, we receive a signal that, by the [convolution theorem](https://en.wikipedia.org/wiki/Convolution_theorem), can be convolved on other signals to result in the same signal that would have existed if we had exposed the original <abbr title="Linear Time-Invariant">LTI</abbr> system on those other signals.

This may sound like magic, but it is not—it's math! Mathematically, the unit impulse response is simply another representation of the <abbr title="Linear Time-Invariant">LTI</abbr> filter, and one which happens to be convenient to utilize in practice. This is because the convolution theorem establishes that multiplication in the frequency domain is convolution in the time domain and vice-versa.

Given this fact, considering the unit impulse in the frequency domain provides some clarity as to *why* the unit impulse response in-factly represents the filter: The filter has some spectral envelope, and by multiplying that envelope uniformly by [unity](https://en.wikipedia.org/wiki/1), the result is the same spectral envelope (by nothing other than the property of multiplicative identity).

Astute readers may realize that this also implies that to uniformly amplify or attenuate a signal (as will be relevant later), all that is necessary is to convolve that signal with an impulse of corresponding magnitude—a uniform multiplication in the frequency domain is simultaneously a pointwise multiplication of all points in the time domain. This might make it clear as to why uniformly multiplying the frequency representation of a signal also results in a uniform multiplication in the time domain, and should strengthen intuitions on how the transformation between the time domain and frequency domain is, in fact, lossless and magnitude-preserving

This application of the convolution theorem also demonstrates how it is possible to combine the spectral envelopes of multiple signals together—if the unit impulse response represents the combination of some arbitrary filter signal and the [identity element](https://en.wikipedia.org/wiki/Identity_element) for this operation (the unit impulse), the same operation of multiplying the spectral envelopes via convolution is applicable to two arbitrary signals in general.

</section><section id="creating-ir">

### Creating the Impulse Response

To create my discrete-time impulse response from my continuous time filter, I had a few options. For this application, however, I decided to simply simulate the impulse response directly with a high-quality differential equation solver. This is a rather lazy way of calculating the impulse response, but it works, and I would like to reserve discussion of transforming continuous-time <abbr title="Linear Time-Invariant">LTI</abbr> systems to discrete-time impulse responses for another time in the near future where it is more relevant and I can go more in-depth on the topic (this article is already long enough as it is).[^9]

[^9]: If you guessed that this topic is orthogonal state-space models, pat yourself on the back. I *will* get to it eventually—I have a lot I would like to discuss on that topic and the mathematics involved.

<figure>
<img src="https://cdn.discordapp.com/attachments/445631164871344139/1096579284384096267/impulse.svg?ex=65eba08a&is=65d92b8a&amp;hm=bbc484ef769bc2d0efab338eee190084781b7aa5c4cf233738291821a102fab4&amp;"/>
<figcaption><p>Simulated discrete-time impulse response of the desired filter, sampled at 48 kHz over 0.1 s. Note the right channel's existence as simply an attenuated impulse, which is a result of the attenuation that was applied to prevent clipping earlier.</p></figcaption>
</figure>

<section>

#### Integrating AutoEq for the <samp>WH-CH710N</samp>

[AutoEq](https://github.com/jaakkopasanen/AutoEq) is a software package designed to produce filters to correct/compensate for the acoustical characteristics of headphones (usually by making their frequency response flatter), and a collection of filter parameters and impulse responses for achieving corrections on a variety of commercially available headphone models.

My headphones are pair of [<samp>WH-CH710N</samp>](https://electronics.sony.com/audio/headphones/all-headphones/p/whch710n-b) produced by Sony Electronics, an over-ear design featuring [Active Noise Cancellation](https://en.wikipedia.org/wiki/Active_noise_control) (<abbr title="Active Noise Cancellation">ANC</abbr>). I use these nearly every day on average, and my only gripes with them are a lack of multi-device connectivity[^1] and the inability to use them while charging.

[^1]: It seems like this choice is intentionally made for the purpose of segmentation: Both of the flagship Bluetooth headphone models sold by Sony Electronics, the older [<samp>WH-1000XM4</samp>](https://electronics.sony.com/audio/headphones/headband/p/wh1000xm4-b) and newer [<samp>WH-1000XM5</samp>](https://electronics.sony.com/audio/headphones/headband/p/wh1000xm5-b), do support multi-device connectivity—but the [<samp>WH-CH720N</samp>](https://electronics.sony.com/audio/headphones/headband/p/whch720n-b) (the replacement for the <samp>WH-CH710N</samp>) does not. Considering that other (and cheaper) Bluetooth headphones from other vendors do have this feature, I find it hard to believe it is not artificial segmentation.

The AutoEq collection includes an [entry for the <samp>WH-CH710N</samp>](https://github.com/jaakkopasanen/AutoEq/tree/master/results/rtings/rtings_harman_over-ear_2018/Sony%20WH-CH710N) and provides parameters for parametric and non-parametric equalizers and impulse responses to correct towards a more “ideal” frequency response. Since the filter compensating for my hearing is only making a relative correction to my stronger ear, it remains logical to simply combine my filter with the correction provided by AutoEq.

This combination could be achieved by placing these two filters in series, but we can be more efficient here; having covered the convolution theorem earlier in this article, we can put it into practice: By convolving both impulse responses, we receive the impulse response of a new filter with a multiplication of their frequency responses/spectral envelopes.

<aside><p>This filter is longer than the previous one, which implies a greater latency penalty when applied to an incoming signal <abbr title="Finite Impulse Response">FIR</abbr>. I am willing to make this tradeoff personally. Regardless of my integration of AutoEq, all demos in this article will forego the <samp>WH-CH710N</samp>-specific filters and instead use the filter that directly results from my fixed-band equalizer.</p></aside>

</section>
</section><section>

### Filtering audio ahead-of-time

One of the things we can do with our productionalized filter is apply it to audio ahead-of-time. This enables us to apply our filter to audio and play it back on any suitable system. It also lets me share samples of what audio sounds like when using the filter—it isn't particularly meaningful to anyone but me, but that is the point!

To do this, I can leverage the Swiss Army knife of the multimedia world: FFmpeg. With a single command, I can straightforwardly apply the impulse response I have produced—a before/after comparison of applying this filter is found in the adjacent figure. Note the heavy attenuation on the right channel required to balance the audio so that it does not sound panned-right.

<figure>
<div>
<figure>

<div>

```sh
ffmpeg -i space_debris.mod space_debris.mp3
```
<audio controls="" src="https://cdn.discordapp.com/attachments/1095536142167842950/1096617170307387452/space_debris.mp3?ex=65ebc3d3&amp;is=65d94ed3&amp;hm=f8ebae9150af4556a89eadc1b5dbdc76a4d317222dc17d1edb4777c450106106&amp;"></audio>
</div>
<figcaption><p>Original audio</p></figcaption>
</figure>
<figure>
<div> 

```sh
ffmpeg -i space_debris.mod -filter_complex 'amovie=impulse.wav[ir]; [0][ir]afir=gtype=none' space_debris-filter.mp3
```
<audio controls="" src="https://cdn.discordapp.com/attachments/1095536142167842950/1096617170684887150/space_debris-filter.mp3?ex=65ebc3d3&amp;is=65d94ed3&amp;hm=137aa962a6582c544dbfe9aea0d159272c45d1deaaadce896a4d18082bc47cbd&amp;"></audio>
</div>
<figcaption><p>Filtered audio</p></figcaption>
</figure>
</div>
<figcaption><p>A demonstration of the productionalized fixed-band equalizer as applied as a <abbr title="Finite Impulse Response">FIR</abbr> filter to <a href="https://markuskaarlonen.com/space-debris"><i>Space Debris</i> by Markus <q>Captain</q> Kaarlonen</a>. <strong>Wear headphones!</strong></p></figcaption>
</figure>

</section><section>

### Real-time filtering Linux audio with PipeWire 

It is nice to apply the filter to audio ahead of time, but the full power of compensating for my hearing is when the correction is applied in real-time—such as is required when I am on a call and need to better inteligiblize the speech of others.

This is among the prospects that enticed me to switch away from Windows 10 to Fedora 36 in <time datetime="2022-07">July 2022</time>, as Linux has a particular weapon to make this straightforward to implement and productionalize in a way that is unavailable on other platforms.

[PipeWire](https://pipewire.org/) is a graph-based multimedia server that looks to drastically improve the handling of audio (and video) on Linux through a new robust and low-latency subsystem aimed to supersede (while also providing interfaces to replace) many previous Linux audio APIs such as ALSA, PulseAudio, and JACK.

PipeWire is extremely powerful and configurable when combined with a session manager (such as the *de facto* standard session manager WirePlumber), allowing for scripted behaviors and modifications to the graph. In the present context however, the role of the session manager can be ignored, as everything we need to do to apply our filter is available in PipeWire itself.

<section>

#### The PipeWire Filter-Chain Module

One of PipeWire's numerous modules is the [Filter-Chain module](https://docs.pipewire.org/page_module_filter_chain.html), which provides a whole host of useful capabilities for real-time audio filtering—biquad filters, convolutions, gains, and mixing are all available to filter audio directly in PipeWire without additional software. 

To use it, we can create a directory <samp>~/.config/pipewire/filter-chain.conf.d/</samp>. After that, writing and placing config files in that directory and activating the filter chain (either through the PipeWire Filter-Chain systemd daemon via <kbd>systemctl --user restart filter-chain</kbd> or running <kbd>pipewire -c filter-chain.conf</kbd> from <samp>~/.config/pipewire</samp>) immediately applies the configurations stored there.

In my case, I have a config that creates two convolutions to apply—one for each channel of a referenced impulse response—so that they are applied to the corresponding incoming audio. This is also configured to automatically link its output to an appropriate sink device.

I leave the deeper details of the configuration experience for [later in this article](#walkthrough), where a very similar application (that is more relevant to those other than myself) makes for a better stage for discussing some of the minor foot guns involved with this setup.


</section>
</section>
</section>
<section>

## Mixing across channels for improved intelligibility

By this point, the designed filter is productionalized and adequately fits the needs of my daily life. However, I found something irritating enough to not be completely satisfied.

<section>

### Multichannel audio, Mixing, and Headphones

<dfn>Multichannel audio</dfn>[^8] (or what I would prefer to call it in this context, *polyphonic sound*)[^7] was one of many core innovations to the advancement of audio reproduction technology. Regardless of what it is called, all it does over monophonic configurations is what its etymology usually describes: Expand the audio signal from one spacial dimension to multiple spacial dimensions (or put another way, multiple one-dimensional signals in space sharing simultaneous locations in the time dimension).

[^8]: I believe there should probably be a stronger common distinction between *surround* and *multichannel audio*: *Multichannel audio* does not necessarily need to be mixed for surround, and the fact that two-channel sound is “multichannel” (yet is universally recognized as *not* surround) supports this lack of equivalency.

[^7]: As far as I can tell, [*polyphonic*](https://en.wikipedia.org/wiki/Polyphony) has been exclusively reserved for usage in music, despite its seemingly logical application to the context of audio configurations with more than one spatial signal dimension. I, therefore, am inclined to use the word *polyphonic* outside of this context for the purpose of clarity to the audience.

It doesn't seem too complicated on the surface beyond having to deal with the extra overhead of more than one spatial signal dimension. However, ask a mastering engineer about their job, and they will tell you all sorts of intricacies they need to keep in mind when dealing with such audio. Ask them about headphones, and they might tell you that they are (for lack of a better description) a curse upon this world—accounting for headphones adds a layer of complexity to their jobs that they rather not need to account for, and when headphones are expected to be used with a statically-mixed audio track, there is a requirement for compromise.

This is because what seems simple in the ideal does not match reality. A mastering engineer's job is to make a track reproduce as well as it can, regardless of the equipment used. The problem is that equipment for reproducing sound from a recording is in no way consistent, with wide variance in quality and configuration—which are often-times intentionally tuned to be objectively inferior for the purpose of subjective superiority! This problem exists for monophonic sound, but adding additional channels greatly expands this configuration space.

The issue with headphones is that they represent a completely different type of configuration than standard stereophonic speaker setups; headphones have the property that the sounds of each channel are *isolated* from each other, with each ear receiving its own signal without interference. In contrast, when a single stereophonic channel produces a sound in free space, *both* ears receive the same sound with a delay slightly out of phase from each other (dependent upon the positioning of that channel), and when both channels are producing sound, they [interfere](https://en.wikipedia.org/wiki/Phase_cancellation). **This means that listening to a stereophonic track with headphones is not equivalent to listening to the same track through speakers.**

It happens to be that the better compromise when statically mixing audio is to mix for speakers, not headphones. This means that certain information may only exist on a subset of channels, and in the case of stereophonic sound, exclusively exist on one channel. This brings an issue to be solved—how to make this information available to both ears when the listener wants or needs it.

Patients with unilateral hearing loss often need to lean on their good ear to intelligible complex sounds like speech, and I am no exception—I will often turn my head to face a speaker on my left and ask the speaker to repeat themself if the intelligibility of their speech is low. This is a widely recognized phenomenon by designers of assistive systems for the unilaterally hearing impaired, and so the hearing aid manufacturers catering to this subset of patients already have a solution for this: Sell them two hearing aids, and configure the hearing aid on the weaker side send audio to the stronger one. This unidirectional crossfeed arrangement with two hearing aids makes less sense for patients who have one side with reasonable hearing (like me), but for patients who have a weaker strong ear and weaker disparity between ears, this solution is a no-brainer.

<aside><p>This topic is specifically one of the reasons why I have chosen to feature <i>Space Debris</i> in demos in this article; due to the influence of the Amiga's hardware and the MOD format, <i>Space Debris</i> has very strong stereo separation meant for speakers. This represents a particularly strong example of a mix that I find mostly inaccessible to listen to with headphones without compensation for my hearing: Without any sort of audio crossfeed, I end up receiving an underrepresentation of the modulefile's first and fourth channels (which are aggressively panned left) and overrepresentation of the second and third channels (which are aggressively panned right).</p></aside>

</section><section>

### Mono Audio (and its problems)

The poor man's solution to this is to simply mix stereophonic channels equally into a monophonic signal shared on all outputs. On most computerized systems, this can be achieved by toggling the <dfn>mono audio</dfn> feature in the accessibility settings.

While computationally cheap, this sort of mixing results in a major loss of information; by collapsing the stereophonic mix into a monophonic one, the listener loses all ability to locate sounds with respect to the stereo axis—much to the frustration of those properly mastering such audio everywhere (who likely meticulously-crafted that information). This can be attempted to be compensated for with a more limited degree of crossfeed, but regardless of where the strength of the mixing resides on the axis between fully separated and monophonic, there is going to be a tradeoff for what information can exist on both ears and the separation between them.

</section><section>

### Binaural sound & Virtual Surround

The topic that is slowly being stumbled into here is what is known as [<dfn>binaural sound</dfn>](https://en.wikipedia.org/wiki/Binaural_recording)—producing sound for each ear in a way that factors in the physical characteristics of auditory sensing. In other words, the production of stereophonic sound so that when both ears receive independent signals (like when wearing headphones), the listener perceives a sound stage that is more reflective of how the audio would sound if that listener were instead in an environment where there is interference between channels.

This seems like a tall order—an effect that would require massive amounts of experience, knowledge, and equipment to produce. Historically, it was, with deep connections to psychoacoustics and requiring specialized equipment specifically designed for the purpose of replicating what a listener would hear if they were present in the recording environment (not to mention that the recordings are headphone-specific). However, with the advancement of technology, the production of binaural sound has been widely accessible for quite some time. In fact, it is not even a niche technology and has been sold to consumers for decades.

The aforementioned technology is [<dfn>virtual surround</dfn>](https://en.wikipedia.org/wiki/Virtual_surround)—simulating the sound of audio reproduced in an environment of an ideal multi-channel speaker configuration. By leveraging virtual surround, it is possible to mitigate at least some of the flaws found in naively downmixing to monophonic sound. 

</section><section>

### Signal Processing of Virtual Surround

This seems complex, but this article has already covered all the knowledge required to understand and implement virtual surround—yes, it is that simple!

A <dfn>Head-Realated Transfer Function</dfn> (<abbr title="Head-Realated Transfer Function">HRTF</abbr>) is a transfer function that describes how incoming sounds (the input) are affected by the geometry of the human head and the auditory system on its way to the inner ear (the output). Virtual surround is most often implemented as a collection of impulse response signals representing a particular <abbr title="Head-Realated Transfer Function">HRTF</abbr> to convolve over the incoming surround sound mix before the convolved signals are mixed together for each ear. This representation as a collection of impulse responses is known as a <dfn>Head-Realated Impulse Response</dfn> (<abbr title="Head-Realated Impulse Response">HRIR</abbr>).[^4]

[^4]: Non-symmetric <abbr title="Head-Realated Impulse Responses">HRIRs</abbr> are normally distributed with double the number of source channels, as constructing a bipartite graph between an input number of vertexes and output number of vertexes (in this case fixed at two) requires a number of edges equal to their product. It is however often possible to exploit the bilateral symmetries involved in an ideal surround sound setup when distributing <abbr title="Head-Realated Impulse Responses">HRIRs</abbr>—for example, since the front-left channel to the left ear is symmetric to the front-right channel to the right ear, they can share the same impulse response. This halves the number of required impulse response channels to transport to one-per-input channel.  

Something to consider is that audio (especially recordings) is, much more often than not, mixed for stereophonic configurations and not for surround sound. This leads to a desire to map this audio onto surround sound systems. While the simplest mapping is to output the left and right channels to the front-left and front-right, this leaves the system underutilized. Instead of this simple mapping, a better utilization can be achieved by more (very basic) signal processing: In the case of virtual surround, this will often lead (unintuitively) to mixing the opposite direction first—most often *upmixing* to many channels before downmixing to stereophonic (and now binaural) audio to be reproduced by headphones. However, when sound is mixed for surround sound, it is possible to leverage a virtual surround system to its full potential.[^5]

[^5]: This is why game settings often have dedicated “headphones” sound options as well as options for “mono”, “stereo”, and potentially a few “surround sound” configurations. When spatial audio cues are important in particular 3D games, this “headphone” mode often is some implementation of virtual surround or even more complex signal processing within the game engine to achieve more precise and accurate positional audio.

</section><section>

### Virtual surround ahead-of-time

Like before, we can demonstrate virtual surround by filtering audio ahead-of-time with FFmpeg. To straightforwardly apply a given <abbr title="Head-Realated Impulse Response">HRIR</abbr> in multi-channel format, we can use <samp>libavfilter</samp>'s `headphone` filter. A before/after comparison can be found in the adjacent figure.

<figure>
<div>
<figure>
<div>

```sh
ffmpeg -i space_debris.mod space_debris.mp3
```
<audio controls="" src="https://cdn.discordapp.com/attachments/1095536142167842950/1096617170307387452/space_debris.mp3?ex=65ebc3d3&amp;is=65d94ed3&amp;hm=f8ebae9150af4556a89eadc1b5dbdc76a4d317222dc17d1edb4777c450106106&amp;"></audio>
</div>
<figcaption><p>Original audio</p></figcaption>
</figure>
<figure>
<div>

```sh
ffmpeg -i space_debris.mod -filter_complex "amovie=hrir.wav[hrir]; [0][hrir]headphone=map=FL|FR|FC|LFE|BL|BR|SL|SR:hrir=multich" space_debris-hrir.mp3
```
<audio controls="" src="https://cdn.discordapp.com/attachments/1095536142167842950/1096634080315064350/space_debris-hrir.mp3?ex=65ebd393&amp;is=65d95e93&amp;hm=64f42ea3a2aaf7ec76a66a5ce17967a2f0e7b1b0638216e037a3874144b4a8e9&amp;"></audio>
</div>
<figcaption><p>Audio processed with virtual surround</p></figcaption>
</figure>
</div>
<figcaption><p>A demonstration of virtual surround as applied to <a href="https://markuskaarlonen.com/space-debris"><i>Space Debris</i> by Markus <q>Captain</q> Kaarlonen</a>. <strong>Wear headphones!</strong></p></figcaption>
</figure>

</section><section>

### Virtual Surround in the PipeWire Filter-Chain

It is possible to implement virtual surround in the PipeWire Filter-Chain—in fact, it represents two of the examples provided by PipeWire upstream on how to leverage the Filter-Chain module: [<samp>sink-virtual-surround-5.1-kemar.conf</samp>](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/src/daemon/filter-chain/sink-virtual-surround-5.1-kemar.conf) and [<samp>sink-virtual-surround-7.1-hesuvi.conf</samp>](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/src/daemon/filter-chain/sink-virtual-surround-7.1-hesuvi.conf) for 5.1 and 7.1 channel virtual surround respectively. Because of their inclusion upstream, most downstream distributions of PipeWire will have these example configurations included. However, documentation on how to achieve this is quite sparse. Luckily, once the time is spent figuring out how to do it, it isn't too complicated.

For this walkthrough, I am going to demonstrate the setup of the 7.1 channel virtual surround configuration, as it is what I found to be easier to set up and more customizable to fit my needs.

<section id="walkthrough">

#### Setting up HeSuVi 7.1 virtual surround in PipeWire

As is indicated in the file header, place a copy of <samp>sink-virtual-surround-7.1-hesuvi.conf</samp> in <samp>~/.config/pipewire/filter-chain.conf.d/</samp>. As far as I know, this has to be a copy and not a soft link to the file if it is included in the local distribution of PipeWire—we will need to edit this file later.[^6]

As the name of the sample config file implies, we will need to acquire a 7.1-channel <abbr title="Head-Realated Impulse Response">HRIR</abbr>, and is specifically intended to be sourced from the [HeSuVi](https://sourceforge.net/projects/hesuvi/) project on SourceForge. To do this, download <samp>HeSuVi_2.0.0.1.exe</samp> and extract it with a tool like 7-Zip that can extract Windows Executables. After extracting, enter the extracted directory and, if completed correctly, there should be a directory <samp>./hrir/</samp> inside with the files we are after.

At this point, we have a decision to make—HeSuVi is bundled with a wide variety of impulse responses. Each of the files in the  <samp>./hrir/</samp> directory are documented in the directly adjacent <samp>info.csv</samp>. Due to my requirements and specifically not wanting reverb, I chose to proceed with <samp>atmos-.wav</samp> (<q>Dolby Atmos 7.1 virtual surround sound for headphones without reverb</q>). I recommend moving or copying the selected <abbr title="Head-Realated Impulse Response">HRIR</abbr> file to a directory in <samp>~/.config/pipewire/filter-chain.conf.d/</samp> to keep things neat—I placed it in <samp>~/.config/pipewire/filter-chain.conf.d/hrir_hesuvi/</samp> as is implied to be a good spot for it by the example configuration.

Now that everything is in place, we can edit <samp>sink-virtual-surround-7.1-hesuvi.conf</samp> by replacing all the files referenced in the configuration (originally configured with <code>hrir_hesuvi/hrir.wav</code>) with the **absolute** path to the <abbr title="Head-Realated Impulse Response">HRIR</abbr> file. In my case, I replaced all instances of <code>hrir_hesuvi/hrir.wav</code> with <code>/home/erichallahan/.config/pipewire/filter-chain.conf.d/hrir_hesuvi/atmos-.wav</code>.[^6]

[^6]: Note that, in my experience, a relative path (as indicated in the example) does *not* work—the path *must* be absolute. The requirement to copy the config and modify would not be always necessary if this were not the case, as it would be possible to instead move/copy the desired <abbr title="Head-Realated Impulse Response">HRIR</abbr> to reside at <samp>~/.config/pipewire/filter-chain.conf.d/hrir_hesuvi/hrir.wav</samp> (assuming relative paths would work the way I would expect). I am not sure if this is intended behavior or not, and have not personally investigated it any further than noticing this behavior and finding a nearly-skilless workaround.

At this point, the setup process should be complete. Starting/restarting the PipeWire Filter-Chain daemon (with <kbd>systemctl --user restart filter-chain</kbd> if using systemd) or running <kbd>pipewire -c filter-chain.conf</kbd> from <samp>~/.config/pipewire</samp> should expose <samp>Virtual Surround Sink</samp> as a new virtual sink device to play audio to.

<section>

##### PipeWire Surround Upmixing Behavior

It is notable that in [PipeWire 0.3.68](https://gitlab.freedesktop.org/pipewire/pipewire/-/tags/0.3.68), default upstream behavior with respect to upmixing audio was changed: Previous behavior upmixed audio to surround format by default, while such upmixing is disabled by default in PipeWire 0.3.68 and later—the channels are directly mapped without any signal processing involved.

The expectation by PipeWire upstream is that downstream packagers of PipeWire will include a new configuration file, [<samp>20-upmix.conf</samp>](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/src/daemon/pipewire.conf.avail/20-upmix.conf.in), to make it easy to enable upmixing. The choice to enable upmixing by default in future distribution of the software is intended for those downstream maintaining those distributions; package maintainers are supposed to weigh the tradeoff for their own users, rather than PipeWire making that decision for them upstream.

As of writing, Fedora has followed upstream PipeWire and upmixing is indeed disabled by default in <samp>pipewire-0.3.68-1</samp>. To reenable for the user, the new configuration file can be soft-linked from its packaged location into <samp>~/.config/pipewire/pipewire.conf.d/</samp>:

```sh
ln -s /usr/share/pipewire/pipewire.conf.avail/20-upmix.conf ~/.config/pipewire/pipewire.conf.d/20-upmix.conf
```

While at it, we can optionally also do the same for [<samp>10-rates.conf</samp>](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/src/daemon/pipewire.conf.avail/10-rates.conf.in), which allows PipeWire to unfix the graph rate from its default behavior of fixing the graph rate to 48 KHz:

```sh 
ln -s /usr/share/pipewire/pipewire.conf.avail/10-rates.conf ~/.config/pipewire/pipewire.conf.d/10-rates.conf
```

Restart the PipeWire daemon (with <kbd>systemctl --user restart pipewire</kbd> if using systemd) to put these configurations into effect. 

</section>
</section>
</section>
</section><section>

## Conclusion

The fixed-band equalizer I constructed for my personal use and presented here is not particularly impressive from a technical standpoint. It is *far* from optimal, and is likely more complicated than required to receive a similar magnitude response while failing to recognize the role of phase. However, it has stood unchanged for months of heavy usage as something “good enough”—probably since it is a “massive improvement” from no compensation at all or other bodges I have tried.

If I were to continue to improve on this project, I would first start by ditching the fixed-band equalizer for a more expressive parametric one—I should be able to construct a substantially lower-order filter to achieve nearly the same response in magnitude and tune it in a way that achieves a more minimal phase distortion.

</section>