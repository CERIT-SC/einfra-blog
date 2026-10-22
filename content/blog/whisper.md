---
date: '2026-10-22T08:00:00Z'
title: 'Whisper on e-INFRA CZ'
thumbnail: '/img/whisper/whisper-tit.png'
author: "Jan Nehyba"
description: "Speech-to-text on Czech academic infrastructure"
tags: ["Jan Nehyba", "Whisper", "Speech-to-Text", "AI", "Social Sciences", "MUNI"]
colormode: true
draft: false
---

## Transcribing Speech with Whisper on e-INFRA CZ

The [CERIT-SC AI tools](https://blog.e-infra.cz/blog/chat-ai/) have offered large language models through a chat interface and an OpenAI-compatible API since 2025. Since May 2026, the same API has also offered speech recognition: the CERIT-SC team deployed OpenAI's `whisper-large-v3`, and in our first test it transcribed 103 seconds of Czech speech in about five seconds.

In this article we show how to call it, what the endpoint can and cannot do, and what we learned while building transcription into [QualReAI](https://qualreai.dyn.cloud.e-infra.cz/), a tool for AI-assisted qualitative analysis developed at the Faculty of Education, Masaryk University.

### The challenge

Qualitative researchers work with recorded conversations: interviews, focus groups, therapy sessions. Transcribing one hour of talk by hand takes an afternoon, and a study has dozens of such hours.

Commercial speech-to-text services solved the typing years ago, but not the second problem: recordings like these are sensitive data. The e-INFRA CZ models run on CERIT-SC's own hardware, and the [AI Data Privacy Policy](https://docs.cerit-sc.cz/en/docs/ai-as-a-service/ai-data-privacy) recommends direct API access for sensitive or protected datasets; for highly sensitive data, such as health data under GDPR, CERIT-SC offers to discuss a dedicated, isolated deployment. Whether a particular kind of recording may be processed this way still has to be assessed case by case, for example with your ethics committee or data protection officer.

### How to call Whisper on e-INFRA

The endpoint follows the OpenAI audio API, so any OpenAI-compatible client works. How to get an API key is described in the [AI API documentation](https://docs.cerit-sc.cz/en/docs/ai-as-a-service/ai-api).

#### Your first transcript

With `curl`:

```bash
curl -sS https://llm.ai.e-infra.cz/v1/audio/transcriptions \
  -H "Authorization: Bearer ${EINFRA_API_KEY}" \
  -F file=@interview.mp3 \
  -F model=whisper-large-v3 \
  -F language=cs \
  -F response_format=verbose_json
```

With Python and the `openai` package:

```python
import os
from openai import OpenAI

client = OpenAI(base_url="https://llm.ai.e-infra.cz/v1/",
                api_key=os.environ["EINFRA_API_KEY"])

with open("interview.mp3", "rb") as audio:
    result = client.audio.transcriptions.create(
        model="whisper-large-v3", file=audio,
        language="cs", response_format="verbose_json")

for seg in result.segments:
    print(f"[{seg.start:7.1f} – {seg.end:7.1f}] {seg.text.strip()}")
```

On a 76-second public-domain recording from the Czech Spoken Wikipedia ([Vladivoj.ogg](https://commons.wikimedia.org/wiki/File:Vladivoj.ogg)), the call took 2.5 seconds and printed:

```text
[    0.0 –     4.2] Vladivoj. Z Wikipedie otevřené encyklopedie.
[    4.9 –    10.7] Vladivoj zemřel v lednu 1003, byl český kníže v letech 1002 až 1003.
[   11.3 –    17.4] Jeho původ není zcela jasný, pravděpodobně byl syn Doubravky Přemyslovny a polského knížete Měška,
[   17.9 –    23.0] brater Boleslava Chrabrého, byl tedy příbuzní Přemyslovců a zároveň pěstovec.
```

Even on clear, read-aloud speech the last line contains two errors: *brater* instead of *bratr* (brother), and *pěstovec* ("grower") instead of *Piastovec*, a member of the Piast dynasty. Rare proper nouns are usually the first thing a speech recognition model gets wrong.

#### What the endpoint can and cannot do

We tested every parameter against the live endpoint in October 2026; the limits were measured on two different days with identical results:

| | Behaviour on e-INFRA |
|---|---|
| Model | `whisper-large-v3`, the only speech model in the [model list](https://docs.cerit-sc.cz/en/docs/ai-as-a-service/chat-ai). Models can be replaced at any time; the [status page](https://llm.ai.e-infra.cz/status/) shows what is currently served. |
| Endpoint | `/v1/audio/transcriptions` only. `/v1/audio/translations` returns 404. |
| File size | About 25 MB per request: a 23.8 MiB file passed, 25.9 MiB was rejected with `Maximum file size exceeded`. That is 13 minutes of 16 kHz WAV, but about 50 minutes of 64 kbps mono MP3. |
| Formats in | ogg, mp3 and wav worked. |
| Parallel requests | 4 per API key, shared with your chat calls. Of six simultaneous requests, two got HTTP 429 `max_parallel_requests`. |
| `language` | Tells Whisper which language is spoken, so set it to the language of the recording. A wrong value raises no error: with `language=en`, our Czech clip came back as an English translation, with mistakes such as "Doubravka the industrialist" and the year 2003 instead of 1003. |
| `response_format` | `json`, `text` (also returned as JSON) and `verbose_json`. `srt` and `vtt` are rejected with HTTP 400, so subtitles have to be built from the segments. |
| `verbose_json` | `text`, `language`, `duration` and `segments` with `start`, `end`, `text`, `avg_logprob` and `compression_ratio`. `no_speech_prob` is always `null`. |
| `timestamp_granularities[]=word` | Accepted but ignored: `words` comes back empty. Segment times are all you get. |
| `temperature` | Controls how much randomness the decoder uses. At `0` repeated calls gave identical text; at `0.8` punctuation and capitalisation changed. The original Whisper starts at 0 and retries a window at higher temperatures when the output loops or has low confidence. vLLM, which serves the model here, does not implement this fallback ([a TODO in its source](https://github.com/vllm-project/vllm/blob/main/vllm/entrypoints/speech_to_text/transcription/protocol.py)), so checking the output is up to you. |
| `prompt` | Accepted, but see the lessons learned below before using it. |
| Not available yet | Speaker diarization, word timestamps, subtitles, a translation endpoint, and the decoding options of the original Whisper (`no_speech_threshold`, `condition_on_previous_text`). |

Long calls are covered by the general 30-minute limit for non-streaming requests in the [AI API documentation](https://docs.cerit-sc.cz/en/docs/ai-as-a-service/ai-api).

### Long recordings

Our 47–78 MB focus-group recordings were rejected outright. Re-encoding to 64 kbps mono MP3 brings most interviews under 25 MB, and a 33-minute public-domain recording ([Kosatka dravá](https://commons.wikimedia.org/wiki/File:Kosatka_drava.ogg)) went through in one request in 78 seconds. Sending the recording in four pieces of up to 10 minutes, three at a time, took about 40 seconds, but splitting has its own traps.

**Re-encode, do not stream-copy.** Our first chunker used `ffmpeg -c copy`, which is fast and lossless. On one 42-minute interview, the last of five pieces came back with `400 unsupported audio file`. Both `ffprobe` and a test decode reported the file as valid, but it carried a malformed MP3 frame header. Re-encoding each piece from the source solved it. Whisper resamples everything to 16 kHz mono anyway, so there is little point in uploading more than `-ac 1 -ar 16000 -b:a 64k`, which makes a 10-minute piece about 5 MB.

**Cut at a pause.** We first cut the 33-minute public recording blindly every 10 minutes. Sentences were split at the seams, and Whisper filled one of the cuts with invented words: the speaker said *zůstávají ve stejných sociálních skupinách* ("stay in the same social groups"), and the transcript says *zůstávají významnější, než než než* ("stay more significant than than than"). Moving each cut to the nearest pause (`ffmpeg -af silencedetect`, looking back up to one minute) fixed it.

**Clamp timestamps, keep the words.** Near the end of a piece, Whisper's timestamps drift past the piece's real length, so segments from neighbouring pieces overlap and the transcript goes out of order. The text in those segments is real speech. Our first fix dropped segments that started after the end of their piece, and it silently lost three sentences. Clamping their times to the piece boundary keeps the words.

**Check how many words came back.** On the same recording with pause-aligned cuts, three 30-second windows came back as a single short phrase or as nothing at all: 91 seconds of speech, about 5 % of the words, gone without an error or a warning. Normal speech runs at two to three words per second, so a stretch of more than 10 seconds with fewer than 0.5 words per second, or a gap between segments, is worth sending again on its own. All three windows came back on the second try.

```python
def skipped(seg):
    """A long stretch with almost no words = Whisper jumped over a 30 s window."""
    dur = seg["end"] - seg["start"]
    return dur > 10 and len(seg["text"].split()) / dur < 0.5
```

### From raw transcript to research data

A raw Whisper transcript is not yet research data. It does not know who is speaking, it splits speech into short fragments, and it confidently writes words that do not exist. In QualReAI we built a chain of small, auditable steps around it. For sensitive recordings it matters where each step runs:

{{< image src="/img/whisper/pipeline.png" class="rounded" wrapper="text-center w-80" >}}

The top row turns audio into raw text. The recording is cut at pauses and re-encoded, `whisper-large-v3` on e-INFRA transcribes the pieces, and our code stitches them back together and checks for skipped windows. The bottom row turns raw text into research data. pyannote in our own Kubernetes pod identifies who speaks when, chat models on e-INFRA clean up the text sentence by sentence, MorphoDiTa at LINDAT flags words that are probably misheard, and the researcher has the final word on every change. Only the mishear check sends anything outside e-INFRA and our own infrastructure, and it sends the transcript's words, never the audio.

The steps that made the biggest difference:

- **Artifact cleaner.** In silent stretches, Whisper writes subtitle credits ("Titulky vytvořil JohnyX", *subtitles by JohnyX*), music notes and chains of ellipses. A dozen regular expressions remove them before any LLM sees the text.
- **Diarization without word timestamps.** [pyannote](https://huggingface.co/pyannote/speaker-diarization-community-1) (`speaker-diarization-community-1`, open weights, run in our own pod) determines *when* each person speaks. Because the endpoint returns no word times, we split Whisper segments at pyannote's turn boundaries before assigning speakers, and compute chunk offsets from each piece's measured length so the timeline does not drift.
- **Speaker smoothing with a question guard.** Smoothing removes flickering speaker labels, but a naive version also merges the interviewer's short follow-up questions ("A proč?", *And why?*) into the respondent's answer. The guard keeps genuine short questions as separate turns.
- **Embedded-question repair.** When an interviewer's question is still glued inside a respondent's paragraph, an LLM (via the same e-INFRA API) returns the question as a **verbatim quote**. Our code checks that the quote really occurs in the text and splits it out. This avoids relying on character offsets generated by the LLM.
- **Merging same-speaker fragments.** Consecutive fragments of one speaker are joined into whole turns. On one 42-minute interview, 937 fragments with a median of five words became 128 turns that can be coded.
- **Sentence-level correction with context.** Asked to clean up a 150-word paragraph at once, models returned fragments, and our length guard rejected 69 of 93 segments. Splitting paragraphs into sentences, correcting each with up to two sentences on either side as context, and reassembling them solved it. The raw transcript is never overwritten, and a re-run never overwrites the researcher's own edits.
- **Mishear detection with a dictionary.** Whisper's confidence (`avg_logprob`) cannot tell a garbled word from a short answer like "Aha.", but a dictionary can. Words that the Czech morphological analyser [MorphoDiTa](https://lindat.mff.cuni.cz/services/morphodita/) cannot analyse are flagged as possible mishears. An LLM proposes a fix only when it is confident, and the researcher accepts, rejects or edits it. The same analysis also tags people and places (`Novák_;Y`, `Brno_;G`), a useful starting point for pseudonymising transcripts. Two caveats: MorphoDiTa runs as a public LINDAT service at Charles University, so the transcript's words (not the audio) leave e-INFRA for this step, and its Czech models are licensed CC BY-NC-SA, i.e. for non-commercial use only.
- **Uncertainty marks.** What stays unclear is marked in the transcript using conventions from conversation analysis (Jefferson notation) instead of being smoothed over.

### Results

{{< image src="/img/whisper/speed.png" class="rounded w-60" wrapper="text-center" >}}

| What we measured | Result |
|---|---|
| 45-minute Czech research interview | 42 s (64× real time) |
| 33-minute public recording: one 16 MB request vs 4 pieces, 3 in parallel | 78 s vs ≈ 40 s; Whisper silently skipped 30 s windows in both (1 and 3) |
| Mishear detection on a 42-minute interview | 82 of 143 flagged words were real mishears (57 %) |
| Automatic correction of suspected mishears | 115 of 137 fixed (84 %); the researcher's review queue fell from 45 to 22 items |
| Uncertainty marks after calibration | 30 → 7 marks, all 7 at genuinely unclear passages |

The interview figures come from our own research recordings, which cannot be published. The other figures come from public-domain recordings, so anyone can repeat the measurement.

### Lessons learned

**Do not send a prompt.** The endpoint accepts a `prompt` (Whisper's `initial_prompt`) for names and jargon. On long recordings with silent stretches, even a 38-character Czech header made the model echo the prompt into the silence and loop. In the first run it produced 7 distinct repeating sentences with 65 hallucinated copies and lost about a third of the content; a second run on the same 45-minute interview lost 70 % of the characters. Lowering the temperature did not help. Our pipeline now sends no prompt by default and handles terminology later, in the correction steps above.

**Silence also produces text.** On the same 45-minute interview, the subtitle-credit hallucination appeared exactly four times both on e-INFRA and in our CPU fallback (faster-whisper large-v3-turbo, [edumuni/speech2text-api](https://github.com/edumuni/speech2text-api)). It is a property of the model family, not of the server.

**Test on a short sample first.** Every failure described here took minutes to find on a short, difficult excerpt and an hour on a full file, so we now start with the hardest three minutes.

**Check the output limit of LLM steps.** Our correction step once reported success while 92 % of segments were untouched: the JSON response hit the default 4096-token output limit, was truncated and parsed as "no edits". Smaller batches, an explicit `max_tokens` and treating an unparseable response as a failure fixed it.

**Public benchmarks may not fit your use case.** Other open models look better on paper: NVIDIA's [Canary-1B-v2](https://huggingface.co/nvidia/canary-1b-v2) reports 7.86 % word error rate on Czech FLEURS, against 11.33 % for Whisper large-v3 ([tech report](https://arxiv.org/abs/2509.14128)). But FLEURS consists of sentences read aloud, while research interviews are spontaneous, colloquial and full of jargon, and recordings like these rarely end up in public test sets. Before switching models, compare them on 30–60 minutes of your own hand-corrected audio, and coordinate such tests on the shared e-INFRA models with CERIT-SC in advance, as their documentation asks.

### Conclusion

To sum up, `whisper-large-v3` on e-INFRA CZ transcribes a 45-minute Czech interview in well under a minute, inside academic infrastructure, with the same API key as the chat models. Calling it is a single request; most of the work lies around it: splitting long recordings at pauses, checking that no speech went missing, and turning raw segments into speaker turns a researcher can trust. Only a test on your own recordings shows how a model handles your kind of speech.

### What next?

Speech-to-text on national academic infrastructure is useful well beyond our project: oral history, linguistics, journalism, lecture capture or dictated field notes. CERIT-SC [is open](https://blog.e-infra.cz/blog/chat-ai/) to adding models based on research needs, so if your community needs speaker diarization as a service or a better Czech model, let them know.

**Thanks** to the CERIT-SC team for hosting open-weight models, from large language models to Whisper, on national academic infrastructure, and for adding new ones when researchers need them.

*AI use statement: During the preparation of this article, the author used Claude Opus 5.5 (Anthropic) to draft and edit the text, run the API tests and produce the figures, and GPT-6 Astra (OpenAI) for an independent editorial review. All numbers come from live tests and the cited sources. The author reviewed and edited the content and takes full responsibility for it.*
