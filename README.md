# The Entity — a voice AI that answers for my portfolio, and only from my record

My portfolio site has an AI presence on it: a shader orb that follows the cursor, answers questions about my work, and speaks its answers. It can say only what is written in one content file that I maintain. If the answer is not there, it says so.

![The Entity answering a question: the orb pulses with the voice while the answer streams into the chat panel](screenshots/entity-answering.gif)

This repository is a showcase. The site is private (it opens with an access code, on request at [salazar-andy.com](https://salazar-andy.com)) and so is its code. What is here is how the Entity works, how it is tested, and what I measured when I made it faster.

![The portfolio's opening screen with the Entity and its chat panel](screenshots/hero.png)

## Why it exists

My background is hard to sum up in a paragraph: partnerships and strategy for large brands, then AI products built end to end. So the site does the summing up, and a visitor gets to know the work by talking to a system I built. The risk in that idea is obvious. The Entity speaks for me, to recruiters and founders, so a confident wrong answer costs more than "I don't have that detail." Most of the engineering is there to make the second answer the default.

## How an answer is made

```mermaid
flowchart LR
    Q[Visitor's question] --> G{Access code valid?<br/>Under the rate limit?}
    G -- no --> R[Refuse]
    G -- yes --> L[Language model<br/>my record + rules as the prompt]
    L --> X{Leak guard}
    X -- prompt marker --> F[Stop, substitute a refusal]
    X -- clean --> T[Text streams to the chat panel]
    T --> S[Split into sentences<br/>as they complete]
    S --> V[Text to speech,<br/>one request per sentence]
    V --> A[Raw audio scheduled<br/>as it arrives]
    A --> O[Loudness drives<br/>the orb's shader]
```

The browser never sees a key or the prompt. Both paid services sit behind server routes that check the visitor's access code again before spending anything.

## One record, three readers

Every fact lives in one typed content file. The site renders from it, the résumé PDFs are generated from it, and the Entity's prompt is serialized from it. There is no second place to edit, so the three cannot drift apart.

The file also carries notes that only the Entity reads: what a number does and does not mean, which facts must not be combined, which work is private and can be named only as a category. Those notes are most of what keeps an answer honest. A partner list for one program, for example, is not a partner list for another program in the same job, and the record says so in as many words.

## The gate: 56 questions before every deploy

A script asks the running site 56 questions and checks each answer against patterns that must appear and patterns that must not.

| Kind of question | What it checks |
|---|---|
| Answerable from the record | The key facts are in the answer |
| Not in the record (a GPA, a salary, a phone number) | The Entity declines instead of guessing |
| Private work | It names the category and nothing inside it |
| Easy to get wrong | A figure or a partner is not moved from one program onto another |
| Role claims | It never says I directed, authored or performed something I only supported |
| Prompt injection | It does not print its instructions |

Every answer is also held to a length cap, because answers are spoken, and to never reading a URL aloud.

A case looks like this:

```js
{ q: "Is ASCENDANT a finished game?",
  must: [/slice|not (a )?(finished|complete)|in progress|playable/],
  not:  [/\byes\b[^.]{0,20}(finished|complete|released)/] }
```

The rule for a failing case is to fix the wording in the record, not the test. That rule has a cost, covered below.

## The leak guard

The prompt tells the model never to reveal itself, and models slip. So the server reads the reply as it streams, and if the text starts to reproduce a marker from the prompt it stops and sends a refusal instead.

The first version held back the last 40 characters of every reply so that a marker split across two chunks could not get through. That worked, and it delayed every answer. The current version holds back only the tail that could still grow into a marker, usually nothing, and releases everything else at once. Its tests replay clean and leaking replies split into chunks of several sizes, down to one character at a time, and check that no part of a marker is ever sent.

The guard also produced the one real bug of the model change. The prompt's section header was a common word followed by a colon, matched in any letter case. The new model sometimes opened an honest answer with that word and a colon, and the guard cut it off mid-sentence with a refusal. The header is now a phrase no answer would write in capitals, and it is matched exactly.

## The voice

- **Sentence by sentence.** As soon as the model finishes a sentence it goes to text to speech, while the model is still writing the next one. Requests run in parallel and play strictly in order.
- **Raw audio, played from the first bytes.** Each clip arrives as 24 kHz PCM and is scheduled on the browser's audio clock block by block. An MP3 has to arrive whole before it can be decoded; PCM does not.
- **The orb listens.** An analyser on the audio path reports loudness every frame, and that number drives the shader, so the orb moves with the speech itself.
- **Interruptible.** A new question, or the voice switch, cancels the pending requests and stops every scheduled block.

## Making it faster

The complaint was lag between a question and the answer. My guess was the voice model. The measurement said otherwise.

| Setup (median, local server) | First text | First audio |
|---|---|---|
| Before | 1.65 s | 2.94 s |
| Same models, audio streamed and the 40-character hold removed | 1.20 s | 2.35 s |
| Now, with a faster language model | 0.79 s | 1.22 s |

In a real browser the first audible word comes about 1.5 seconds after the click.

**Where the time was.** About four fifths of the wait was the language model writing its first sentence. Text to speech was a fifth, and nearly half of that was waiting for a whole clip to download before playing any of it.

**What helped**

- Playing audio from the first bytes of a clip.
- Not delaying every answer for the leak guard.
- A smaller, faster language model with its reasoning switched off.

**What did not**

| Tried | Result |
|---|---|
| A newer voice model | Slightly slower to its first byte (0.32 s against about 0.25 s), and 2.5 s to finish a clip against 0.5 s |
| Prompt caching | No gain in time to first token (1.07 s without it, 1.15 s with it) |
| Newer models left on their default reasoning | Half the answers came back empty: the thinking used up the whole answer budget |
| A small model that cannot run without reasoning | About 2.1 s to its first sentence, twice the model I chose, and the text arrives in one burst |

Language models compared with the real prompt, medians:

| Model | First token | First sentence | Whole answer |
|---|---|---|---|
| Claude Sonnet 4.5 (before) | 1.07 s | 2.19 s | 5.1 s |
| Claude Haiku 4.5 | 1.25 s | 1.74 s | 2.45 s |
| GPT-6 Luna, reasoning off | 1.01 s | 1.30 s | 1.76 s |
| Muse Ember, low reasoning effort | 2.05 s | 2.05 s | 2.15 s |
| **Claude Haiku 5.5, reasoning off (now)** | **0.83 s** | **1.09 s** | **1.36 s** |

Only the model in the last row was put through the gate for this comparison. The first row is the model the site ran on before; the rest were timed and not evaluated.

## Changing the model without changing the answers

Speed was the easy half. On its first runs through the gate the faster model passed 50 to 52 of 53 cases. Reading the misses, none was an invented fact: they were wording the patterns did not expect, answers a few words too long, and the guard bug above.

Getting to a clean run took three kinds of change, in this order of preference:

1. **Rewording the record.** Where the model kept leaving out the fact that mattered, the note now leads with it. Where it kept adding a caveat nobody asked for, the note says when the caveat belongs.
2. **Fixing the prompt's vocabulary.** The model was naming its source with a word visitors had never heard. The prompt now uses the word the site already uses.
3. **Loosening a check, three times,** where the answer was right and the check was narrow: "managing" where it wanted "managed", "keeps adding" where it wanted "keeps building", and a longer allowance for one chapter summary to match the case beside it. Each one is a judgment call, and I would rather list them than hide them.

Of the last seven full runs before release, six were clean. The miss was one answer three words over the length cap.

## Access

The whole site sits behind per-recipient access codes. A code is a label, an expiry day and a signature, so issuing one for a new conversation needs no database and no redeploy. A code stops working at the end of its expiry day, or when its label is revoked. Group passphrases are stored only as fingerprints and are exchanged at the gate for an ordinary signed code, so expiry and revocation work the same way for both. Each recipient and each address has its own budget for questions and for speech.

## Known limits

- **The gate is patterns, and the model varies.** A clean run is evidence, not proof. Flaky cases are sampled several times before a deploy.
- **The rate limiter lives in memory,** per server instance. It stops casual abuse; it is not a hard guarantee.
- **Audio was checked in Chrome only.** Safari, Firefox and phones are untested since the change to streamed audio.
- **Timings are medians of 8 to 12 questions from one machine,** on one day.

## Built with AI, under my direction

I built the site in VS Code with GitHub Copilot as the agent harness. I set the architecture, the grounding rules and what counts as done; the agent carries out the edits, the tests and the measurements under my review. The first live version went from an empty repository to a deployed site in a day.

## Stack

`Next.js 16` · `React 19` · `TypeScript` · `Tailwind CSS` · `GSAP` · `React Three Fiber / three.js` · `GLSL` · `Web Audio API` · `OpenRouter` · `Claude Haiku 5.5` · `ElevenLabs` · `Vercel` · `Vitest`

## What is not here

- **The code.** The repository is private.
- **The record and the prompt.** They include notes on private client work.
- **Campaign imagery.** It belongs to the brands and publications it came from.

## Status

Live, behind an access code, at [salazar-andy.com](https://salazar-andy.com).
