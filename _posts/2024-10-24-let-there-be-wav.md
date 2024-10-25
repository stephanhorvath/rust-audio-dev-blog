---
layout: post
title: "Let There Be Wav"
date: 2024-10-24
categories: rust audio dsp
---

# First Milestone: Recording to WAV with Rust

The very first milestone I'm documenting in this blog: capturing sound through a
microphone, and saving it as a .wav file, all through Rust.

Baby steps.

It's relatively simple code, combining two examples from `cpal` and `hound`.

## A Host

In order to get mic input, I first followed the `cpal` example "feedback.rs". In
it, I learned how to initialise each part I need to get input into Rust. First,
we need a host. The `cpal` host struct is what allows the Rust program to access the
my laptop's audio devices. The `cpal` crate guarantees that any platform supported by CPAL
(Cross Platform Audio Library) has a default host.

`let host = cpal::default_host();` initialises our host audio device. From
there, we can ask the host if there are any inputs or outputs, so I query the
host to get access to the default inputs on my laptop, i.e. the single
microphone on it.

## An Input Device

`let input_device = host.default_input_device()`. Given that this returns an
Option (a type in Rust), that can have a value, in which case it holds a
`Some`, or no value, in which case it holds a `None`. In case there are no
default input devices, it would be nice to return a message to the user. This
can be done in a few different ways, and these ways depend on how you and where
you want to handle the error. I decided to do `.expect()`. The input device 
declaration now looks like this:

`let input_device = host.default_input_device().expect("Failed to find input
device")`

The `.expect()` function panics when the function it is called on fails, and
provides an ideally useful, short message with the reason why. It's not
particularly good error handling right now, but it is helpful when trying stuff
out. It would be better to use the `?` operator and propagate the error to where
it can be handled, and avoiding panics altogether.

## An Input Config
