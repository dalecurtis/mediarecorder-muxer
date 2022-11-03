# MediaRecorder for Muxing Explainer

## Authors:

- Dale Curtis @ Google

## Introduction
Today [MediaRecorder](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder) can be used to record a [MediaStream](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream) to a file. I.e., encode audio and video and mux the encoded data into a container (usually webm or mp4).

We propose a small extension to MediaRecorder API to allow it to work with the output of WebCodecs' [AudioEncoder](https://developer.mozilla.org/en-US/docs/Web/API/AudioEncoder) and [VideoEncoder](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder) to handle simple muxing use cases.


## Goals
* Allow developers to avoid including a JavaScript library for basic muxing cases. E.g., one audio stream, one video stream, or one audio and one video.
* Providing an outlet for MediaRecorder use cases requiring more complex editing, encoding settings, or encoding provided via a WASM module.
* Expose MediaRecorder to DedicatedWorker.


## Non-Goals
* Supporting complex muxing cases. For example:
  * Multiple audio and/or multiple video streams.
  * Complex container metadata like custom ISO-BMFF boxes or edit lists.
* Supporting muxing of arbitrary codecs. I.e., codecs must be ones already known by the browser in some form.


## Example Usage

```Javascript
let recorder = null;
let audioStream = {stream: null, config: null};
let videoStream = {stream: null, config: null};
let audioChunks = []
let videoChunks = []

function maybeCreateRecorder() {
  if (!audioStream.stream || !videoStream.stream || recorder)
    return;
  recorder = recorder = new MediaRecorder({
    mimeType: 'video/webm; codecs="..."',
    audio: audioStream,
    video: videoStream
  });
  recorder.start();
}

function createStream() {
  return new ReadableStream({
    start(controller) { this.controller = controller; },
    cancel() { /* ... */ },
    close() { this.controller.close(); },
    addChunk(chunk) { this.controller.enqueue(chunk); },
  });
}

function onAudioChunk(/*EncodedAudioChunk:*/chunk,
                      /*EncodedAudioChunkMetadata:*/metadata) {
  if (!audioStream)
    audioStream = { stream: createStream(); config: metadata.decoderConfig };
  maybeCreateRecorder();
  audioChunks.push(chunk);
  if (recorder)
    audioStream.stream.addChunk(audioChunks.shift());
}

function onVideoChunk(/*EncodedVideoChunk:*/chunk,
                      /*EncodedVideoChunkMetadata:*/metadata) {
  if (!videoStream)
    videoStream = { stream: createStream(); config: metadata.decoderConfig };
  maybeCreateRecorder();
  videoChunks.push(chunk);
  if (recorder)
    videoStream.stream.addChunk(videoChunks.shift());
}

let audioEncoder = new AudioEncoder({/*...*/ output: onAudioChunk /*...*/ });
let videoEncoder = new VideoEncoder({/*...*/ output: onVideoChunk /*...*/ });

// ... configure and use encoders ...

// Stop and flush encoders.
await audioEncoder.flush();
await videoEncoder.flush();

// Flush any remaining chunks.
while (audioChunks.length > 0)
  audioStream.stream.addChunk(audioChunks.shift());
audioStream.stream.close();
while (videoChunks.length > 0)
  videoStream.stream.addChunk(videoChunks.shift());
videoStream.stream.close();

recorder.stop();

// Alternatively: Subscribe to MediaRecorder.ondataavailable
let mediaFile = recorder.requestData();
```

## Open Questions / Notes / Links
* ???

## Considered alternatives

### Design and implement a full containers API.
Containers are complicated and supporting both muxing and demuxing with the full gamut of multi-track streams seems daunting. Whereas this likely requires only minor technical work to implement on top of the existing MediaRecorder implementations.

## Stakeholder Feedback / Opposition

- Chrome : ???
- Developers : ???

## Proposed IDL

```Javascript
typedef (AudioDecoderConfig or VideoDecoderConfig) MediaRecorderConfig;
dictionary EncodedMediaChunkStreamInit {
  required ReadableStream stream;
  required MediaRecorderConfig config;
};

dictionary MediaRecorderChunkStreamInit {
  required DOMString mimeType;
  optional EncodedMediaChunkStreamInit audio;
  optional EncodedMediaChunkStreamInit video;
}

[Exposed=(Window,DedicatedWorker)] interface MediaRecorder : EventTarget {
  // ...

  [CallWith=ExecutionContext, RaisesException, Exposed=Window] constructor(
      MediaStream stream, optional MediaRecorderOptions options = {});

  [CallWith=ExecutionContext, RaisesException] constructor(
      MediaRecorderChunkStreamInit init);

  // ...
};
```
