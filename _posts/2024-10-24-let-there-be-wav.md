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

Before a stream can be opened to start taking in input data, `cpal` has to know
the type of input that's incoming. The configuration tells the code the number
of channels, the sample rate, the buffer size, and the sample format. 

`let input_config = input_device.default_input_config().expect("Failed to get
input config")`

The default configuration on my device is:

* Channels: 1
* Sample Rate: 96kHz
* Buffer Size: { min: 29, max: 4096 }
* Sample Format: F32

Now that the program knows what type of data to expect, I can open a stream with
the right configuration.

## An Input Stream

The stream is what allows the program to receive input or audio data, depending
on the type of stream. Right now I'm only interested in input data.

Different devices can have different configurations, so it's easy enough to
match the stream to the correct sample format. I did that with a match over the
sample format (though I am only showing to examples here):

```
let stream = match input_config.sample_format() {
    cpal::SampleFormat::I16 => input_device.build_input_stream(
        &input_config.into(),
        move |data, _: &_| write_input_data::<i16, i16>(data, &writer_2),
        err_fn,
        None,
    )?,
    cpal::SampleFormat::F32 => input_device.build_input_stream(
        &input_config.into(),
        move |data, _: &_| write_input_data::<f32, f32>(data, &writer_2),
        err_fn,
        None,
    )?,
}
```

The stream is created through `build_input_stream()`. This function takes in
four parameters:

* The input config, but not as a SupportedStreamConfig, but as a SteamConfig,
  which can be converted through `.into()`,
* a data callback function,
* an error callback,
* and a timeout option

I had some trouble understanding the data callback function. I understand that
it's a function that is repeatedly called by `cpal` to handle the incoming (or
outgoing) data. I believe the 'data' value is provided by `cpal` when a callback
function is created.

Additionally:

* The `move` keyword transfers ownership of variables to the closure. In this
  case, writer_2. Ownership needs to be transferred in this closure
  because of how Rust manages concurrency. Because `cpal` operates on a separate
  thread and the data callback function may be called asynchronously, Rust
  requires the thread to own the data, or wrap it in an `Arc` and a `Mutex`. By
  transferring ownership of writer_2 into the closure, Rust can guarantee that
  it will be valid and accessible within the closure as long as the stream
  exists. Without `move`, writer_2 would be borrowed, and issues could arise if
  the closure outlives the scope where writer_2 was defined.
* `data` is a block of audio input data.
* The argument I'm ignoring in the closure, `_: &_` is actually the
  InputCallbackInfo. This `cpal` struct contains some metadata about the stream.

There is also an error function, and a timeout Option.

## Writing to WAV

As you can see in the stream code, there is a function called `write_input_data`
that takes in the data from the input, and a writer. This writer is actually
part of the `hound` crate.

In order for `hound` to create and write a WAV file, it needs to know the format
of the data it will record. This can be obtained directly from the `input_config`. The `WavSpec` struct exists within `hound`, with the same fields provided by the config, so telling `hound` the format of the data is as easy as:

```
hound::WavSpec {
    channels: config.channels() as _,
    sample_rate: config.sample_rate().0 as _,
    bits_per_sample: (config.sample_format().sample_size() * 8) as _,
    sample_format: sample_format(config.sample_format()) as _,
}
```

I learned that the expression `as _` can be used to ask Rust to infer the type
to convert to when the type is obvious from context.

For the bits_per_sample field, the byte size is just being converted to bits, as
required by the WavSpec field. As for the sample_format, `hound` just needs to
know whether it's a Float or an Int, so `sample_format()` just returns either
`hound::SampleFormat::Float` or `hound::SampleFormat::Int`.

Once the spec is set up, the writer can be initialised using `hound`'s WavWriter
struct.

`let writer = hound::WavWriter::create(PATH, spec)?;`

All you need to pass the create function is the path it will write to, and the
spec.

But first, because the writer will be used in a multi-thread environment, it has
to be wrapped inside an `Arc` and a `Mutex`. So writer is shadowed by:

`let writer = Arc::new(Mutex::new(Some(writer)));`

And in order to use the writer inside the closures, it is cloned, so that it can
be safely moved into the closures, and have an independent, thread-safe
reference to the WavWriter.

Lastly, the stream is allowed to run through `stream.play()?;`. I first let the
stream run for a specified amount of time by sleeping the main thread. Then the
stream is dropped. The writer is locked, unwrapped, taken, unwrapped again, and
finalised, and an `Ok(())` is returned at last, to indicate no errors occurred.

How is the writer handled through
`writer.lock().unwrap().take().unwrap().finalize()?;`?

First, acquiring a lock on the Mutex gives exclusive access to the data within.
`.lock()` returns a Result, which needs to be unwrapped. Then, `.take()`
accesses the Option that writer was initially wrapped in earlier (Some(writer))
and replaces the Some(writer) with a None, leaving no way for more data to be
written and returning the Some(writer). 

With the Some(writer), the second `.unwrap()` ensures that there is a valid 
WavWriter before calling `.finalize()`, which writes the remaining buffered 
data to the file, and closes it.
