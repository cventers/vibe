# Audio Processing Pipeline

This document describes how Vibe processes audio files during transcription. It focuses on normalisation, diarisation, progress updates and reliability checks.

## Normalisation and Pre-processing

1. **File Validation** – the pipeline first verifies that the source file exists. If it is missing, the process stops with an error.
2. **Normalisation** – audio is converted to a standard format (16 kHz, mono, 16-bit PCM) using `ffmpeg` through the `normalize` function. Extra `ffmpeg` arguments can be supplied by the caller.
3. **Caching** – the normalised file is written to a temporary location determined by a cache key built from the source path and optional `ffmpeg` arguments. Although caching is currently disabled, the infrastructure is present for future use.
4. **Parsing** – the `parse_wav_file` function reads the normalised `.wav` file and ensures it meets the expected sample rate, sample format and channel count. Inconsistent files result in an error.

## Transcription

1. **Whisper Context** – a Whisper context is created using the selected model and GPU configuration. The creation is wrapped in `catch_unwind` to guard against panics during initialisation.
2. **Parameter Setup** – transcription parameters such as language, sampling strategy, number of threads and token timestamps are configured by `setup_params` based on user options.
3. **Processing** – the audio samples are converted to `f32` and passed to `state.full`, producing Whisper segments.

## Multi Speaker Diarization

When diarization is enabled via `DiarizeOptions`:

1. **Segmentation** – `pyannote_rs::segment` divides the audio into regions containing a single speaker.
2. **Embeddings** – for each segment an embedding is computed with `pyannote_rs::EmbeddingExtractor` and compared with an `EmbeddingManager`. The manager keeps track of known speakers up to the configured `max_speakers`.
3. **Speaker Assignment** – the result is either an existing speaker label or a new one if the similarity falls below the `threshold` provided in `DiarizeOptions`; otherwise `?` is returned.
4. **Transcription** – each diarized segment is transcribed individually. Segment start and stop times are converted to Whisper timestamps and stored in the resulting `Segment` struct along with the speaker name.
5. **Callbacks** – after every segment, the optional `new_segment_callback` is invoked with the text and speaker information. Progress is updated as a percentage of processed diarization segments.

### Segmentation Details

The segmentation step relies on the `segmentation-3.0.onnx` model (an ONNX export
of the Pyannote v3 network) shipped with Vibe. The path can be overridden through
`DiarizeOptions.segment_model_path` when using a custom model. `pyannote_rs`
analyses the 16&nbsp;kHz waveform in small frames (around 25&nbsp;ms), predicting
speech activity and potential speaker changes. Adjacent speech frames are merged
and pauses shorter than roughly 300&nbsp;ms are bridged so that natural sentence
pauses do not create new segments. Each region retains its start and end
timestamps along with the raw samples, which are later fed into the embedding
extractor for speaker recognition.

## Progress Tracking

The pipeline supports realtime progress notifications using a `progress_callback` function. When diarization is active, progress reflects how many diarized segments were transcribed. In single-pass mode, Whisper's internal progress updates are forwarded through a global callback. This enables UI components to display completion percentages.

An `abort_callback` can be supplied to stop processing early. It is checked before handling each diarization segment and periodically during transcription.

## Reliability Features

- **Error Handling** – important operations such as file reads and command execution use `eyre`'s `Context` and `bail!` macros to return descriptive errors.
- **Sanity Checks** – `parse_wav_file` validates channel count, sample rate and bit depth, ensuring unexpected files are rejected early.
- **Crash Protection** – creating the Whisper context is wrapped with `catch_unwind` to catch panics from the underlying C++ library. If a panic occurs, a user facing error is returned instead of crashing the application.
- **Resource Availability** – the `find_ffmpeg_path` helper attempts multiple locations to locate `ffmpeg` on the user's system and provides a clear error when it cannot be found.

## Output

Once transcription completes the collected segments are returned as a `Transcript`. The structure can be converted into plain text, VTT or SRT formats, depending on the user’s needs. The processing time in seconds is also included.

