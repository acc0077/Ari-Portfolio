# Voice Travel Agent — Real-Time Speech-to-Speech Conversational Assistant

**Primary Focus:** Real-time voice agent pipeline (STT → LLM → TTS)  
**Secondary Capabilities:** Streaming UX, interruption handling, threaded audio systems  
**Domains:** Voice AI, conversational agents, low-latency systems  

**Role:** Sole builder (end-to-end)  
**Stack:** Python, OpenAI API (Chat + TTS), Google Cloud Speech-to-Text, PyAudio, threading, queues

---

## Executive Summary

Built a real-time, speech-to-speech **voice travel assistant** that behaves like a phone-based conversational agent. The system listens continuously via microphone input, transcribes user speech in real time, streams LLM-generated responses, and speaks responses back to the user with **near-immediate audio playback**—while supporting **barge-in** (user interruption) and multi-turn conversational memory.

This project demonstrates practical voice-agent engineering challenges rarely covered in simple demos:
- low-latency streaming pipelines
- partial response playback
- concurrent audio input/output
- interruption-safe playback
- real-time conversational UX constraints

---

## Motivation / Why I Built This

Most “voice assistants” are stitched together demos that:
- wait for full transcription before responding
- wait for full LLM completion before speaking
- cannot handle user interruptions naturally

I wanted to build a **true speech-to-speech loop** that feels conversational:
- the agent starts speaking quickly
- the user can interrupt at any time
- the system responds fluidly, like a phone call

This project was a hands-on exploration of **voice-first agent architecture**, not just prompt engineering.

---

## Core Capabilities

### 1) Real-Time Speech-to-Speech Agent Loop

The system runs a continuous loop:
Microphone → Streaming STT → LLM (streaming) → TTS → Speaker

Each stage is designed to operate incrementally rather than waiting for full outputs, minimizing perceived latency.

---

### 2) Streaming Speech-to-Text (STT)

- Uses **Google Cloud Speech-to-Text** in streaming mode.
- Processes microphone audio in small frames via `pyaudio`.
- Supports:
  - **interim transcripts** (partial speech)
  - **final transcripts** (turn completion)

Interim transcripts are used not just for display, but as **signals for interruption detection**.

---

### 3) Streaming LLM Response Generation

- Uses **OpenAI Chat Completions** in streaming mode.
- Tokens are received incrementally rather than as a single response.
- Enables the system to begin downstream processing (TTS) before the full response is complete.

This significantly improves conversational feel and responsiveness.

---

### 4) Sentence-Aware Streaming Text Chunking

Streaming tokens are accumulated into **“speakable chunks”** using:
- punctuation boundaries (`.`, `?`, `!`)
- minimal timing thresholds

Each completed chunk is immediately sent to TTS and played back, allowing the agent to **start speaking early** instead of waiting for full completion.

---

### 5) Low-Latency Text-to-Speech (TTS) Playback

- Uses **OpenAI Text-to-Speech** to synthesize audio.
- Audio is streamed, buffered, and played in near real time.
- Playback runs in **separate threads** so the system can:
  - keep listening to the user
  - handle interruptions cleanly

Audio format handling and buffer management were implemented explicitly rather than relying on blocking helpers.

---

### 6) Interruption Handling (Barge-In)

A core feature of the system is **barge-in support**:

- If the user starts speaking while the agent is talking:
  - interim STT transcripts trigger an interruption signal
  - a shared stop flag is set
  - all active TTS playback threads stop immediately
- The system then resumes listening and processes the new user input

This prevents overlapping speech and creates a natural conversational rhythm.

---

### 7) Conversation Memory

The agent maintains a session-level message history using a simple `memory_list` pattern:
- alternating user / assistant messages
- appended on each completed turn

This enables:
- follow-up questions
- contextual responses
- continuity across multiple conversational turns

The pattern is intentionally lightweight and easily extendable to persistent storage.

---

## Agent Behavior & Prompting

The assistant operates under a structured system prompt as a **travel agent persona (“Emma”)**:

- asks clarifying questions (destination, dates, budget, preferences)
- proposes options and iterates quickly
- keeps responses concise to optimize voice UX

Prompt constraints were tuned specifically for **spoken interaction**, not chat verbosity.

---

## System Architecture Overview

### Runtime Flow

1. **Microphone Capture**
   - Audio captured in real time via `pyaudio`
   - Frames pushed into a thread-safe queue

2. **Streaming STT**
   - Audio frames streamed to Google Cloud Speech
   - Interim results monitored for interruption signals
   - Final transcript triggers LLM generation

3. **LLM Streaming**
   - User transcript + conversation memory sent to the LLM
   - Tokens streamed incrementally

4. **Chunking + TTS**
   - Tokens buffered into sentence-level chunks
   - Each chunk synthesized and played immediately

5. **Interruption Control**
   - Shared flags + locks coordinate playback stop/resume
   - System remains responsive throughout

---

## UI / Interaction Layer

- A lightweight **PyQt5 desktop UI** was prototyped to:
  - start/stop the voice loop
  - display transcripts
- The core system also runs **headless**, driven entirely by microphone input.

---

## Libraries & Tooling

- Audio capture/playback: `pyaudio`, `soundfile`, `pydub`
- Streaming STT: `google-cloud-speech`
- LLM + TTS: OpenAI API
- Concurrency: `threading`, `queue`
- Utilities: `io`, `time`, `numpy`

---

## Practical Engineering Notes

- **Echo avoidance:** headphones recommended to prevent TTS audio from re-entering the microphone.
- **STT session limits:** implemented restart logic for long-running streaming sessions.
- **Latency optimization:**
  - speak on partial completions
  - keep responses short
  - minimize blocking calls
- **Thread safety:** playback locks ensure only one audio stream speaks at a time.

---

## Limitations & Future Extensions

- Automatic voice activity detection (VAD) instead of transcript-based interruption
- Integration with real travel data (flights, hotels, POIs)
- Multi-user session handling
- Cost / token tracking per session
- Deployment as a phone or web-based voice agent

---

## Key Takeaway

This project demonstrates **real-world voice agent engineering**, not just prompt design:  
a fully streaming, interruption-aware, speech-to-speech conversational system that behaves like a live phone agent—showcasing practical experience with low-latency audio pipelines, concurrent systems, and voice-first UX constraints.
