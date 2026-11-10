# SLATE — Smart Learning And Teaching Environment

An AI tutoring prototype that answers a learner's question with **three things at once**: a written explanation, a short explainer video, and an interactive 3D model they can rotate and inspect.

**Live demo:** https://interactivetutorbot.netlify.app/

Built as a group hackathon project.

---

## What it does

You type (or speak) a question into the chat box. SLATE then:

1. Sends the prompt to a **Gemini**-backed Express service and returns a written explanation.
2. Reads the answer aloud using the browser's speech synthesis — but only when the question was asked by voice.
3. If the prompt mentions a topic SLATE has curated assets for, it also renders:
   - a short **explainer video**, and
   - an interactive **3D model** in a Babylon.js canvas, with orbit controls and a fullscreen button.

Every exchange is kept in a running, timestamped history, so a session reads like a tutoring transcript rather than a single-shot Q&A.

Currently curated topics: **DNA**, **pyramid**, **planet**.

---

## Architecture

```
Browser (React + Vite)
  |
  |-- Web Speech API ------- voice input + spoken answers
  |-- Babylon.js ----------- glTF model viewer (orbit, fullscreen, inspector)
  |-- bundled .mp4 assets -- topic explainer videos
  |
  \-- POST /api/fetch-text
        |
        v
  Express service (Node)  -->  Google Gemini API
```

- **Frontend** — React 18 + Vite, `@babylonjs/core` for 3D. Deployed on Netlify.
- **Backend** — Express + `@google/generative-ai`, a single `POST /api/fetch-text` endpoint. Deployed on Render at `slatebackend.onrender.com`.
- **Assets** — glTF models (`scene.gltf` + `scene.bin` + textures) served statically from `public/3dmodels/`; videos bundled through Vite from `src/assets/`.

---

## Running it locally

**Backend**

```bash
cd backend
npm install
node geminiserver.js
```

It listens on port 5000. Set your own Gemini API key first — see the security note below.

**Frontend**

```bash
cd SLATE-BABYLON
npm install
npm run dev
```

The frontend posts to `http://localhost:5000/api/fetch-text` in development; the deployed build points at the Render URL instead.

---

## Project history

SLATE went through two quite different designs.

**Phase 1 — generate the visuals on the fly (Python / Manim).**
The original idea was ambitious: take any question, ask Gemini for an explanation, then ask it *again* for [Manim](https://www.manim.community/) code that animates the concept, render that code, and play the resulting video back to the learner. Two variants were prototyped — one that asked the learner whether they wanted a visual, and one that always produced both.

To make the code generation reliable, a **training dataset of ~49 prompt-to-Manim-code pairs** was hand-built and used to attempt a fine-tuned model, alongside a library of **43 rendered reference animations** spanning geometry, calculus, complex numbers, physics and cryptography.

The honest outcome: the fine-tuned model never became dependable enough to ship. Model-written Manim code failed to render often enough that a live demo was a gamble, and the dataset-driven approach was retired after being in use for a while — though the animation library it produced was genuinely useful.

**Phase 2 — curate the visuals, generate the text (React / Babylon.js).**
The project pivoted to the design that is live today: keep Gemini for the *explanations*, where it is reliable, and pre-produce the *visuals* so they always work. Rendered videos and interactive glTF models replaced the on-the-fly Manim rendering, which also moved the experience from a desktop Python script into a browser anyone can open.

---

## Known limitations

This was a hackathon build, and it shows in a few specific places:

- **Topic coverage is keyword-matched, not semantic.** The prompt is scanned for the literal strings `DNA`, `pyramid`, `planet`. Anything else returns text only.
- **That matching is case-sensitive in the deployed build.** `DNA` matches; `dna` does not. An earlier revision lowercased the prompt first, and that behaviour was lost.
- **Text answers are currently failing in production.** The Render backend is up, but the Gemini call returns an error — the hardcoded key is no longer valid and the pinned `gemini-1.5-flash` model has since been retired. Supplying a fresh key and a current model name should restore it.
- **Spoken answers only trigger for voice input.** Typed questions are answered silently by design.
- **AR was started but never shipped.** `index.html` loads A-Frame and AR.js, and an `ARModelScene` component plus a marker pattern and heart model exist locally — but no AR component is present in the deployed bundle.
- **`server.js` (the 3D-model listing service) is dead code.** It references undefined variables and would throw if called; the frontend loads models from static paths instead.

---
