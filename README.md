[README.md](https://github.com/user-attachments/files/32142298/README.md)
# The Drill

An exam-drill tool built on the Anthropic (Claude) API. You paste in your own lecture notes, and it writes exam-style questions from them, puts you through the paper under a live countdown, and marks your answers.

I built it for a specific problem: I understand the material well enough, but I lose marks under time pressure. Existing flashcard apps test recall on generic decks. Nothing let me sit a timed paper generated from *my own* notes and get graded on it. So I made one.

## What it does

- Paste a topic's worth of notes.
- Choose the number of questions, the type (multiple choice, conceptual/short-answer, or mixed), and a time limit.
- Claude generates exam-style questions from the notes — the kind that test application and reasoning, not trivia.
- You sit the paper one question at a time under a countdown that shifts amber, then red, as time runs low.
- Multiple choice is graded instantly on the client. Conceptual answers are sent back to Claude and marked against key points, with a line of feedback.
- You get a percentage, your time, and a per-question review showing the right answer and why.

## How it works

Two calls to the Claude messages endpoint (`POST /v1/messages`), both on `claude-sonnet-4-6`.

### 1. Question generation

The notes go in as the user message; a system prompt casts Claude as an examiner and pins down the output format. Claude returns **only** a JSON array, which drives the interface directly. Each element:

```json
{
  "type": "mcq",
  "q": "question text",
  "options": ["A", "B", "C", "D"],
  "correct": 0,
  "keyPoints": ["marking point 1", "marking point 2"],
  "explain": "one or two sentence explanation"
}
```

`options` and `correct` are present for `mcq` items; `keyPoints` for `short` items. `explain` is always present and shown in the review.

### 2. Grading

Multiple choice is graded locally by comparing the chosen index to `correct` — no API call needed. Short answers are batched into a single grading call: each item carries the question, its key points, and the student's answer, and Claude returns a mark from 0 to 1 (with 0.5 for partial credit) plus a short note per answer. Batching keeps it to one request regardless of how many written answers there are.

### Parsing

Model output is defensive-parsed: markdown fences are stripped and the array is sliced between the first `[` and last `]` before `JSON.parse`, so stray text around the JSON doesn't break the run. A failed parse surfaces a retry rather than a crash.

## Running it

The file is a single self-contained React component (`exam-drill.jsx`). It runs as-is inside the Claude artifact environment, where API access is handled by the session — no key required there.

To run it as a standalone web app, route the two API calls through a small backend that holds your Anthropic API key server-side. **Never ship the key in the client.**

## Files

- `exam-drill.jsx` — the whole app: setup, timed exam, batched grading, and review.

## Possible extensions

- Score history across attempts on the same paper.
- Paste or upload a PDF of notes instead of plain text.
- Per-course question-style presets.
