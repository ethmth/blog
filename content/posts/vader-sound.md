+++
title = "Recreating Darth Vader's Breathing in WebAudio"
author = "Ethan Thomas"
date = 2024-02-25T14:16:59-05:00
draft = false
+++

Inspired by the sounds in Andy Farnell's _Designing Sound_ Part IV: Practicals, particularly the Science Fiction ones (R2D2, Red Alert), as well as the [Lightsaber Example](https://youtu.be/9DqBNDHKFC4?feature=shared) shown in my [Computational Sound](https://www.marksantolucito.com/COMS3430/spring2024/) lecture, I sought out to recreate one of the most iconic, recognizable sounds from Star Wars: Darth Vader's breathing.

**Experience the final result [here](https://sound.ethanmt.com/hw3/).**

## The Inhale {#the-inhale}

To simplify the problem I thought of Darth Vader's breathing pattern as two distinct parts: [the inhale]({{< relref "#the-inhale" >}}) and [the exhale]({{< relref "#the-exhale" >}}), which I later put together using a [gain envelope]({{< relref "#the-envelope" >}}). I started with the inhale. I began by listening to various Darth Vader sound effects, from the film versions to recreations on YouTube. Still, by simply listening I wasn't sure how to get started, so I popped the sound into Audacity and extracted a Frequency Analysis Spectrogram. The Frequency Analysis Spectrogram for Vader's inhale looks like this:

![Frequency Diagram for Inhale](/images/vader_inhale1.png)

I noticed six major peaks at the frequencies 395 Hz, 510 Hz, 785 Hz, 2477 Hz, 4463 Hz, and 5355 Hz. So, using the methology from the [Babbling Brook example](https://www.marksantolucito.com/COMS3430/spring2024/Lab3), I started with Brown Noise and attempted to filter out the frequencies that I wanted. Here is what that code looks like:

```js
// Noises
let brownNoise1 = nodeBrownNoise();
let brownNoise2 = nodeBrownNoise();
let brownNoise3 = nodeBrownNoise();
let brownNoise4 = nodeBrownNoise();
let brownNoise5 = nodeBrownNoise();
let brownNoise6 = nodeBrownNoise();
let brownNoise7 = nodeBrownNoise();

// Filters
let filter1 = nodeBandpass(30, 0, 785);
let filter2 = nodeBandpass(30, 0, 510);
let filter3 = nodeBandpass(30, 0, 395);
let filter4 = nodeRHPF(20, 0, 5500);
let filter5 = nodeBandpass(200, 0, 2477);
let filter6 = nodeBandpass(300, -3, 5355);
let filter7 = nodeBandpass(200, 0, 4463);

// Gains
let gain1 = nodeGain(0.6);
let gain2 = nodeGain(0.005);
let gain3 = nodeGain(0.4);

brownNoise1.connect(filter1);
brownNoise2.connect(filter2);
brownNoise3.connect(filter3);
brownNoise4.connect(filter4);
brownNoise5.connect(filter5);
brownNoise6.connect(filter6);
brownNoise7.connect(filter7);

filter1.connect(gain1);
filter2.connect(gain1);
filter3.connect(gain1);
filter4.connect(gain2);
filter5.connect(gain3);
filter6.connect(gain3);
filter7.connect(gain3);
```

You'll notice that I played around the Q-factors of the filters, giving the higher frequencies higher Q-factors due to their sharper peaks. Additionally, I connected each filter to different gain nodes to control their prevalence in the sound. Finally, I added a high pass filter to capture some of the higher frequencies and give Vader's inhale a whistling-like sound. For the most part, this simple approach worked! I was able to achieve this similar looking frequency distribution, and most importantly, it sounded like Vader:

![My Frequency Diagram for Inhale](/images/vader_inhale2.png)

## The Exhale {#the-exhale}

Vader's exhale wasn't so simple to recreate. I started with the same process, analyzing the frequencies in Audacity, which gave me a jumping off point:

![Frequency Diagram for Exhale](/images/vader_exhale1.png)

From that, I was able to come up with the following noise-filter sequence which resulted in the sound below:

```js
let brownNoise1 = nodeBrownNoise();
let brownNoise2 = nodeBrownNoise();
let brownNoise3 = nodeBrownNoise();

let filter1 = nodeBandpass(100, 0, 654);
let filter2 = nodeBandpass(100, 0, 1260);
let filter3 = nodeLPF(1240, 30);

let gain1 = nodeGain(1);
let gain2 = nodeGain(0.1);

let filterN1 = nodeLPF(1600, 1);
let filterN2 = nodeRHPF(1, 0, 350);

let exhaleGain = nodeGain();

brownNoise1.connect(filter1);
brownNoise2.connect(filter2);
brownNoise3.connect(filter3);

filter1.connect(gain1);
filter2.connect(gain1);
filter3.connect(gain2);

gain1.connect(filterN1);
gain2.connect(filterN1);

filterN1.connect(filterN2);
filterN2.connect(exhaleGain);
```

![My Frequency Diagram for Exhale](/images/vader_exhale2.png)

However, this really didn't sound like Vader's exhale. Compared to Vader's inhale, which is mostly the same sound held for a duration of time, Vader's exhale is much more dynamic, with the start of the sound almost popping as he lets out his breath.

This led me to explore another synthesis technique covered in lecture: [Convolution](https://www.marksantolucito.com/COMS3430/spring2024/convolution/). As I listened to Vader's breathing over and over again, I noticed that Vader's exhale has a similar popping impulse to a tennis racket hitting a tennis ball. And so, with a simple [tennis sound effect](https://www.youtube.com/watch?v=KOMtMxqrKqU), I convolved Vader's exhale like so:

```js
async function nodeConvolver() {
  let convolver = audioCtx.createConvolver();

  let response = await fetch("tennis-hit.wav");
  let arraybuffer = await response.arrayBuffer();
  convolver.buffer = await audioCtx.decodeAudioData(arraybuffer);

  return convolver;
}

let convolver = await nodeConvolver();
exhaleGain.connect(convolver);
```

While the resulting sound isn't perfect, and perhaps there is a better technique I could have used, ultimately the resulting sound is closer to Vader's exhale than it was before, so I deem it a success.

## The Envelope {#the-envelope}

Finally, since I couldn't have Vader inhaling and exhaling at the same time, I constructed a gain envelope around the inhale and exhale sounds such that they would alternate on and off at the correct times. I did this using a for loop to schedule Vader's breathing for the forseeable future, adding some randomness in timing:

```js
for (let i = 0; i < BIG_NUMBER; i++) {
  const randomness = Math.random() * 0.1 - 0.05;
  let duration = CYCLE_DURATION + randomness;

  inhaleGain.gain.setTargetAtTime(1, audioCtx.currentTime + duration * i, 0.1);
  inhaleGain.gain.setTargetAtTime(
    0,
    audioCtx.currentTime + duration * i + 1.43,
    0.2
  );

  exhaleGain.gain.setTargetAtTime(
    1,
    audioCtx.currentTime + duration * i + 2.1 + 0.001,
    0.001
  );
  exhaleGain.gain.setTargetAtTime(
    0,
    audioCtx.currentTime + duration * i + 2.1 + 1.2,
    0.3
  );
}
```

As you can see, the gain envelope for each "breath" is relatively simple, exponentially going up to 1 and back down to 0 at the correct times.

Ultimately, the WebAudio signal flow graph looks like this:

![Vader Diagram](/images/vader_diagram.png)

With the top half corresponding to the inhale part of the sound and the bottom half corresponding to the exhale part of the sound.

## The March {#the-march}

For fun, I added a sequenced _Imperial March_ that uses Additive Synthesis and the same note-playing techniques as [the second homework](https://sound.ethanmt.com/hw2/), just with the frequencies a couple octaves lower.
Learning how to first play the imperial march, then sequence the notes, was a fun challenge.

Finally, I added a rotating Darth Vader model using ThreeJS to complete the experience. You can play the sounds yourself [here](https://sound.ethanmt.com/hw3/), and here is a demo video of the final product:

{{< youtube ibtOw66MoI4 >}}

Overall, I put a lot of work into this, and I'm happy with how it turned out!

### Resources {#resources}

Partner for in-class "Babbling Brook" activity: Yara Studer

Computational Sound Course Website: [https://www.marksantolucito.com/COMS3430/spring2024/](https://www.marksantolucito.com/COMS3430/spring2024/)

Darth Vader 3D Model: [https://cults3d.com/en/3d-model/various/star-wars-darth-vader-3d-print](https://cults3d.com/en/3d-model/various/star-wars-darth-vader-3d-print)

Tennis Sound Effect for Impulse: [https://www.youtube.com/watch?v=KOMtMxqrKqU](https://www.youtube.com/watch?v=KOMtMxqrKqU)

Darth Vader Breathing Sound Effect: [https://www.youtube.com/watch?v=Nn6QWRmCPuY](https://www.youtube.com/watch?v=Nn6QWRmCPuY)

Ryan's WebAudio Node Editor: [https://ruan-xian.github.io/webaudio-node-editor/#/](https://ruan-xian.github.io/webaudio-node-editor/#/)
