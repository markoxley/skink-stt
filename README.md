# skink-stt

Speech-to-text for Skink with a pluggable provider interface: an
OpenAI-compatible REST provider (`/v1/audio/transcriptions`, multipart
upload) and subprocess providers (whisper.cpp CLI, plus a generic
command-template provider for vosk wrappers or any custom CLI).

## Install

```
scales add markoxley/skink-stt
```

## Quick start

```skink
import "stt"
import "stt_iface"

p := stt.Detect()    // whisper.cpp if STT_WHISPER_MODEL set,
                     // else OpenAI if STT_API_KEY/OPENAI_API_KEY set
text, err := p.Transcribe(wavBytes, wavLen, stt_iface.DefaultOpts())
println(text)
```

## Providers

| Constructor | Backend | Notes |
|---|---|---|
| `stt_rest.OpenAI(key)` | `api.openai.com` (`whisper-1`) | needs `STT_API_KEY`/`OPENAI_API_KEY` |
| `stt_rest.WithBase(base, key, model)` | any compatible endpoint | e.g. LocalAI, whisper.cpp's `server` |
| `stt_proc.Whisper(model)` | `whisper-cli` on PATH | model = ggml `*.bin` path; `-otxt` output |
| `stt_proc.WhisperBin(bin, model)` | custom whisper binary | e.g. `"main"` in older builds |
| `stt_proc.Template(tmpl)` | any CLI | `{in}`/`{out}` slots; e.g. a vosk wrapper script |
| `stt.Detect()` | first configured | whisper.cpp → OpenAI → `"none"` |

## Options

```skink
o := stt_iface.DefaultOpts()
o.Lang = "en"          // language hint (auto-detect when empty)
o.Prompt = "jargon"    // style/context prompt (REST only)
```

## Template provider

For engines without a first-class provider, `Template` runs a command
with two placeholders — `{in}` is the input WAV path (written to a
temp file), `{out}` is where the command must write plain text:

```skink
p := stt_proc.Template("python3 /opt/vosk_cli.py --wav {in} --out {out}")
```

## Custom providers

`stt_iface.Provider` is a vtable:

```skink
fn myTr(p: *stt_iface.Provider, audio: string, audioLen: int, o: stt_iface.Opts) -> (string, errors.Error) {
    return "...", errors.New("")
}
p := stt_iface.New("mine")
p.transcribeFn = myTr
```

## Tests

```
bash run_tests.sh
```

- `stt_test` — vtable dispatch, multipart construction (binary-safe),
  REST provider end-to-end over real TCP against a scripted fake
  server, template provider via a stub command.
- `live_test` — self-skipping; runs whisper.cpp (`STT_WHISPER_MODEL`),
  a template (`STT_TEMPLATE`), and/or OpenAI (`STT_API_KEY`,
  `OPENAI_API_KEY`, optional `STT_BASE_URL`) against a generated WAV.
