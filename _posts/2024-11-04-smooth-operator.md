---
layout: post
title: "Smooth Operator"
date: 2024-11-04
categories: filters
---

# On Filters

While I write the post on continuous recording, I dove into low-pass filters and
came out the other side with a better understanding of the differences between
finite impulse response (FIR) and infinite impulse response (IIR) filters.

Filters can be classifed as FIR or IIR. In essence, an FIR filter's output will
become zero after a certain number of samples, as it depends on a finite number
of past inputs. By contrast, an IIR filter's output will gradually approach zero
but _theoretically_ never reach it, as its output depends not only on the
current and past input, but also on the previous outputs, creating a feedback
loop. 

In practical digital systems, however, due to the finite precision of numerical
representations (limited by the number of bits), an IIR's output will eventually
be indistinguishable from zero. This is because as the output values get smaller
and smaller, reaching the smallest possible number the system can represent,
rounding errors will result in zeros.

## The Simplest FIR Low-pass Filter

A low-pass filter cancels out high frequencies, letting the low frequencies pass
through the filter unaffected. One example of full cancellation is taking a
signal, inverting its phase by 180˚, and summing it with the original signal.
The two signals will cancel each other out entirely. With the fact that
inverting the phase cancels out all signals, shifting the phase by different
values should result in different amounts of cancellation!

The smallest amount of delay that can be applied to a digital signal is one
sample. It can be represented like this:

`y[n] = x[n] + x[n-1]`

The output is equal to the summation of the current input and the previous
input.

In Rust, I implemented it like this:

```rust
struct SimpleFIRLowpassFilter {
    // a field to store the input, to use with the next sample
    input: f32,
}

impl SimpleFIRLowpassFilter {
    fn new() -> Self {
        // initialise the output as 0.0, so the first sample is not
        // summed with anything
        let input = 0.0;
        
        Self {
            input,
        }
    }
    
    // creating a function that takes in a single sample is more efficient than
    // copying the entire signal and filtering it. This way, the function can be
    // put inside a .map().
    fn process_sample(&mut self, input_sample: f32) -> f32 {
        // calculate the output with the equation above
        let output = input_sample + self.input;
        // store the input for summing with next sample
        self.input = input_sample;
        // return the output
        output
    }
}
```

However, it's important to note that this on its own will also increase the
overall amplitude of the signal, as it sums two of the same signal. In order to
keep the make sure the filter does not amplify the signal, and keeping the gain
consistent, the signal can be averaged by dividing by the number of inputs we
are using.

```rust
    fn process_sample(&mut self, input_sample: f32) -> f32 {
        // produce an average of the two, smoothing out high frequencies
        let output = 0.5 * (input_sample + self.input);
        self.input = input_sample;
        output
    }
```

Putting a signal through this filter will reveal two things:

1. While it sounds fine, it's not a particularly strong filter. The signal is
   only smoothed out a bit.
2. There's not frequency cutoff control! There isn't a way to control where the
   filter is placed.

The first one can be addressed by smoothing over _more_ samples. Increasing the
window averages out the values of more samples, resulting in more cancellation.
I implemented a version of the filter above with a window size parameter that
controls how many samples are averaged out.

## Building on a Foundation

```rust
use std::collections::VecDeque;

struct AdjustableWindowLowpassFilter {
    window: usize, // the number of samples to average
    coefficients: Vec<f32>,// coef for each previous sample
    prev_inputs: VecDeque<f32>, // double-ended queue vector - very useful!
}

impl AdjustableWindowLowpassFilter {
    pub fn new(window_size: usize) -> Self {
        let window = window_size;
        // initialise the coefficient vector so that each one is 1/window
        // e.g. window = 5 -> every element in coefficients is 1/5
        let coefficients = vec![1.0 / window as f32; window];
        // create a double-ended queue vector from a vector of 0.0 values
        let prev_inputs = VecDeque::from(vec![0.0; window]);
        
        Self {
        window,
        coefficients,
        prev_inputs,
        }
    }
    
    pub fn process_sample(&mut self, input_sample: f32) -> f32 {
        self.prev_inputs.pop_back(); // 1
        self.prev_inputs.push_front(input_sample); // 2
        
        let filtered_value = self.coefficients
            .iter() // 3
            .zip(self.prev_inputs.iter()) // 4
            .map(|(coef, &p_i)| coef * p_i) // 5
            .sum::<f32>(); // 6
        
        filtered_value
    }
}
```

Learning to be proficient with Rust's vectors is the best learning
experience. There are 4 chained functions here:

1. Remove the oldest input from the vecdeque
2. Add the current input to the front of the vecdeque
3. `.iter()` → iterate over the _values_ in the coefficients vector
4. `.zip()` → iterates over two iterators simultaneously
5. `.map()` → takes the values inside the closure, which are the two values that
   are being iterated over, and does _something_ with them. In this case it
   multiplies the coefficient with the corresponding sample value. At the start
   of the function all the previous inputs are zero. It returns a new vector.
6. `sum::<f32>()` → sums all the elements in the vecdeque returned by `.map()`

And finally return the filtered value. 

So initialising a new filter as: `AdjustableWindowLowpassFilter::new(20)`
averages out the current sample with the previous 20.

## Reflections

Diving into the world of digital signal processing for my audio app in Rust has
taught me a lot so far. Here are some key takeaways:

**Fundamentals of DSP Filters:** Working on FIR and IIR filters gave me a clearer
understanding of how digital filtering works, especially the way FIRs rely on
previous inputs and IIRs introduce feedback from prior outputs. This distinction
helped me see the unique behavior and use cases for each type of filter.

**Rust’s Iterator Power:** By using chained iterators like .zip() and .map(), I was
able to streamline the filter’s sample processing, making it both concise and
readable. Rust’s iterator tools are powerful for cleanly expressing operations
like coefficient-sample multiplication, and it’s exciting to see how expressive
this approach is.

**Optimizing State with VecDeque:** Managing a sliding window of samples became
straightforward thanks to VecDeque. It’s specifically suited to the FIFO
(first-in-first-out) nature of filters like these, where new values need to push
out the old in an efficient way.

## What’s Next?

With a basic understanding of digital filtering under my belt, here’s what I
plan to tackle next:

**Filter Control and Adjustment:** I want to experiment with adjustable cutoff
controls. This means rethinking the design slightly to incorporate adjustable
coefficients and adding more interactivity to the filters.

**Exploring More Filter Types:** next time I dive into filters, I want to
explore IIR filters.
