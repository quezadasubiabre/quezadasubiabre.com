---
title: "English Teacher Agent: real-time feedback while you speak"
date: 2026-08-04 12:00:00 +0000
---

I've been building [english-teacher-agent](https://github.com/quezadasubiabre/english-teacher-agent-public), a small project born from a simple observation about my own English lessons.

## The motivation

When I turned on the AI analytics feature in my English learning app, I found out that in a 25-minute class I was speaking for 21 minutes, while the teacher spoke for only 4. That's 84% of the time being me.

If your main goal is practicing speaking, the teacher's role in that interaction is structurally replaceable by AI — as long as it's natural enough to keep the conversation flowing. I already do two daily 15-minute sessions with ChatGPT Voice Mode, and that works well. So the goal of this project isn't to build yet another conversational agent. The real value to add is **real-time feedback while I speak**.

## Demo

<div style="position: relative; width: 100%; height: 0; padding-bottom: 62.5%;">
  <iframe src="https://streamable.com/e/sjfgaq" frameborder="0" width="100%" height="100%"
    allow="fullscreen" allowfullscreen
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"></iframe>
</div>

## What it does

It's a web app (and potentially a Google Meet plugin) with four core features:

- **Live transcription** — your words appear on screen as you say them, in real time
- **Inline corrections** — grammar mistakes are underlined in red with a correction tooltip, a few seconds after they happen
- **Ghost text** — after 3 seconds of silence, a grayed-out suggestion appears showing how to continue the sentence
- **Filler word tracker** — words like "uh", "like", or "you know" are highlighted in amber, with a per-session counter for each one

## Stack

The backend bot is built with [pipecat](https://github.com/pipecat-ai/pipecat) in Python, and the frontend is a Node.js app. Locally, it's just:

```bash
# backend
cd server
uv run bot.py

# frontend
cd client
npm run dev
```

Code is up at [github.com/quezadasubiabre/english-teacher-agent-public](https://github.com/quezadasubiabre/english-teacher-agent-public).
