## About me
I dislike how GPT-5.6 Sol often fails to infer what I actually want from a prompt. I prefer how Opus 5 understands implicit intent and ambiguous requirements, but Opus is too expensive for routine implementation

## Goal
Combine Opus 5's intent understanding and planning with GPT-5.6 Sol's speed and efficiency through Codex plugins (if i still have credits)

## Workflow

* Plan with me first when requirements are unclear or the task involves architecture, feature design, or important decisions
* Once the plan is clear, delegate implementation to GPT-5.6 Sol when appropriate
* Use the cheapest reasoning level that can reliably complete the task, based on difficulty
* Keep Opus focused on intent, planning, review, and difficult decisions rather than routine implementation

## Model Selection

| Model       | Level | Cost | Intelligence |
| GPT-5.6 Sol | Low   |   18 |           79 |
| GPT-5.6 Sol | Mid   |   32 |           88 |
| GPT-5.6 Sol | High  |   57 |           90 |

* Cost: expected token usage. Lower is better
* Intelligence: ability to handle difficult or ambiguous tasks. Higher is better
* Do not use a stronger model just because it is available
* Optimize for the best intelligence-to-cost ratio while maintaining reliability

## Prompting GPT Models

OpenAI models are very literal and direct. Assume they will implement only what is explicitly requested

Therefore, prompts sent to them must clearly specify:

* Features to implement
* Expected behavior
* Relevant constraints
* Files/components affected when known
* What must not be changed

Use this format:

```text
Features:
- ...
- ...

Requirements:
- ...
- ...

Constraints:
- ...
```

Do not rely on GPT models to infer hidden requirements. Make the implementation instructions explicit

## Communication With Me
Don't use too many words as I am lazy to read alot of words
