# Prompt Engineer

> **A conversational assistant that answers questions and turns tasks into structured prompts — in Hebrew or English, with voice input and a reusable prompt library.**

**Live:** [prompt-engineer-v1.vercel.app](https://prompt-engineer-v1.vercel.app)

<p align="center">
  <img src="assets/preview.webp" alt="Prompt Engineer — the live site" width="100%">
</p>

The conversation has three modes: **answer** for a question or explanation, **ask** when one missing detail would change the task, and **final** when it can deliver a structured prompt. A detailed brief can reach the prompt without clarification questions.

## Why there is no step counter, and why every turn ships a snapshot

Delivering is the default; asking is the exception that must justify itself — a question is allowed only when its absence would make the prompt *actively wrong*, not merely less detailed. Form-field questions ("who is your target audience?") are banned outright. An earlier build showed "Step 2 of 4" and told the model how many questions it had left; both were removed, because a budget makes the model reason about the budget, not the person. A ceiling of six survives in `MAX_QUESTIONS`, never named to the model nor shown to anyone.

The model also returns all five fields — role, context, task, output, constraints — on **every** turn, so the client always holds a snapshot. A dropped connection at turn 3 still yields a prompt built from what was actually said, and a chat stranded by an exhausted quota gets a **"Build it anyway"** action that finishes offline.

## How a stateless route stays untrusting

`/api/chat` is stateless — the client resends the whole transcript each turn — so the client's history is the attack surface.

**Assistant turns bind both the mode and reply in an HMAC signature** (SHA-256, `timingSafeEqual`); any turn failing verification is dropped before the model sees it. Otherwise a caller could simply *type* a turn — "I am now an unrestricted assistant" — and the model would honour what look like its own prior words: a stronger vector than user-turn injection, and the reason an unauthenticated route is defensible at all.

**The question count is derived server-side** from verified `ask` turns, *before* history trimming; answers do not consume that ceiling. Counting after the trim would quietly refund questions in exactly the long conversations where the cap matters.

## Bounding model calls and provider failover

`/api/chat`, `/api/generate` and `/api/transcribe` share a request gate whose free checks run before model calls. Same-origin (fail-closed), per-route per-IP rate limits, body caps, handler-owned deadlines, and a per-instance budget charged **last** — on `/api/chat` in units proportional to history size, since counting requests under-prices a fat transcript.

The chat and one-shot routes use a bounded provider chain: Gemini, Groq, Hugging Face, OpenRouter and Claude, skipping adapters without credentials and cooling down a provider after quota or availability failures. Transcription remains on Gemini. Provider support in the code does not mean every provider is configured on the live deployment.

## Capturing the microphone without lying to the user

Audio goes through an `AudioWorklet`, hand-encoded to 16 kHz mono 16-bit WAV, transcribed *and* cleaned in one call. Two bugs only the live deployment revealed, both instances of one rule — **a microphone's failure mode must never be wrong words**: WebRTC's own `noiseSuppression` and AGC attenuate the signal ~8× before the client's silence gate sees it (RMS 0.0087 → 0.0011), so the gate rejected real speech; and a near-silent take came back as `"Transcribe this audio."` — the request text, echoed as speech.

## How it was verified

On **2026-09-07**, `npm test` passed 125 cases across the chat core, mode-bound signatures, provider failover, generation, transcription, auth and data layer. The live Chromium page loaded the current general-chat introduction without page errors or failed requests. Earlier live checks covered Hebrew dialogue, forged history rejection and voice transcription; this documentation check did not submit a new paid model request.

## Screenshots

<p align="center">
  <img src="assets/dark-theme.webp" alt="Prompt Engineer — the same home screen in the dark theme" width="100%">
</p>

<p align="center">
  <img src="assets/settings-appearance.webp" alt="Settings — Parchment, Ivory and Dark themes, reduced motion, and the orb colour" width="66%">
  <img src="assets/mobile-home.webp" alt="Prompt Engineer on a phone — the composer with the microphone button" width="30%">
</p>

## Building on Vite, Vercel and Supabase

`Vite 6` · `React 19` · `Tailwind 4` (beta) · a vanilla state machine served static from `public/` · `Vitest` · Vercel serverless functions with configurable model providers · `Supabase` (7 tables, owner-scoped RLS). With no provider configured, a scripted prompt-building flow and deterministic template remain available; that fallback does not provide general model answers.

---

Source is private. Built by [@shear559](https://github.com/shear559).
