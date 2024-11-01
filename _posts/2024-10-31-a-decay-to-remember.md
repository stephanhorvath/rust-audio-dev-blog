---
layout: post
title: "A Decay To Remember: Low Pass Filtering"
date: 2024-10-31
categories: rust audio dsp filters
---

While I write the post on continuous recording, I dove into single-pole low-pass
filters and came out the other side with a new appreciation for simplicity DSP.
My SinglePoleLowpassFilter struct is about as straightforward as it gets, with
just three parameters: a0, b1, and previous_output.

```rust
struct SinglePoleLowpassFilter {
    a1: f32,
    b0: f32,
    previous_output: f32,
}

impl SinglePoleLowpassFilter {
    fn new(decay: f32) -> Self {
        let x = decay;
        
        let a1 = 1.0 - x;
        let b0 = x;
        
        Self {
            a0,
            b1,
            previous_output: 0.0,
        }
    }
    
    fn process_sample(&mut self, input_sample: f32) -> f32 {
    
        self.previous_output = self.b0 * input_sample + self.a1 * previous_output;
        self.previous_output
    }
}
```

Since a sample is essentially is just a number, and audio data can be
represented as a vector of samples, it's trivial to manipulate these samples by
accessing different parts of the audio data.

In new, we set a1 and b0 using (1.0 - decay) and (decay), respectively, which
define how much we favor the previous output versus the new input sample.  This
balance gives the filter its low-pass response, where low frequencies linger
while higher ones get gradually stripped away.

With `process_sample()`, I’m taking in input samples and running them through the
core filter formula: `filtered_value = a0 * previous_output + b1 *
sample_as_float`. This lets me keep a smooth, lagging average that emphasizes the
lower frequencies.

I was initially tempted to overcomplicate it, but thought that writing a
mini-post on the simplest possible low-pass filter would have its own benefits
for my understanding. Less is more with single-pole filters, and the result is
an effective low-pass filter that’s lean, efficient, and gets the job done.
