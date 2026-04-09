---
title: Daixd5520 的 blog
tags:
  - intro
---

---

title: Home
---

# Hi, I'm Dai Xindi

Working on **large language models, decision systems, and scalable learning**.
Interested in how imperfect models behave inside real-world systems.

---

## 🧠 Research Interests

My work is loosely organized around a central question:

> How to make large models **reliable under constraints** —
> limited compute, noisy inputs, and structured decision requirements.

This unfolds into three directions:

- **Inference-time control**
  (routing, uncertainty, selective reasoning)

- **Data efficiency in alignment**
  (SFT data selection, augmentation, difficulty modeling)

- **System integration**
  (how LLMs interact with retrieval, ranking, and structured environments)

---

## 🚧 Selected Work

Recent work spans both **research prototypes** and **production systems**,
with a consistent focus on **controllability rather than scale alone**.

In industry settings, I worked on integrating LLMs into a structured decision pipeline:

- transforming natural language into executable constraints
- constraining generation under large schema spaces
- evaluating whether content *actually answers* a query
- orchestrating multi-stage inference under latency constraints

This line of work treats LLMs not as standalone models, but as components in a **decision system**.

---

In parallel, I explored **training and inference efficiency**:

- dynamic reasoning routing (**RICO**)
  separating fast-path vs. deliberative reasoning based on uncertainty signals

- distributed training and system optimization
  (DeepSpeed, pipeline parallelism, heterogeneous hardware adaptation)

- retrieval-augmented systems with improved ranking consistency

The common thread is:

> allocating computation where it matters, instead of uniformly increasing it

---

On the research side, current projects focus on:

- **hard sample mining for SFT** (ASPIRE)
- **bidirectional reasoning augmentation** (SimSFT)
- **geometry-aware parameter-efficient transfer** (HOLA)

These attempts approach the same constraint from different angles:

> improving capability without proportional increases in data or parameters

---

## 📂 Map

### Research

- [[RLHF]]
- [[LLM Training]]
- [[Inference & Routing]]
- [[Recommendation System]]

### Systems

- [[Distributed Training]]
- [[Inference Optimization]]
- [[System Design]]

### Notes

- [[Papers]]
- [[Fragments]]
- [[Open Questions]]

---

## ✍️ Writing

This is not a polished blog.

Most entries are:

- intermediate thoughts
- partially verified ideas
- or failed attempts worth keeping

---

## 🧬 Personal

Tends to oscillate between:

- strict structure and complete drift
- deep focus and total disengagement

The system is not always stable,
but remains functional.

---

## 🧭 Current Focus

- improving stability in LLM-based systems
- reducing unnecessary reasoning overhead
- understanding failure modes under distribution shift

---

## 📡 Contact

- GitHub: https://github.com/daixd5520
- Email: daixd5520@gmail.com

---

## Note

This site is an evolving workspace.
Incomplete pages are expected.
