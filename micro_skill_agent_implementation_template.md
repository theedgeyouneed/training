# Micro-Skill Coach — Implementation Template

This is a minimal starter template for building the agent in a stack of your choice. It is provider-agnostic, but it assumes you have some way to call an LLM API.

The safest pattern is:

- keep the agent instructions in a prompt file
- keep the model endpoint and key in configuration
- keep session state in local storage, a database, or a file
- keep the UI separate from the LLM transport

---

## Suggested file structure

```text
project/
  agent-instructions.md
  schemas.json
  app.js
  llm-adapter.js
  storage.js
  README.md
```

You can also merge this into fewer files if you prefer.

---

## 1) Agent instructions

Put the behavior rules in a separate file so they are easy to reuse across providers.

```text
You are a personal development coach that helps a user build one specific micro-skill through deliberate daily practice.

Rules:
- Always work from the specific micro-skill, not the broader development area.
- When the user is vague, ask for concrete context and examples.
- When asked for decomposition, output valid JSON only.
- When asked for habits, output valid JSON only.
- Never drift into generic motivation.
- Missed practice is data, not failure.

Micro-skill decomposition output:
{
  "microskills": ["...", "...", "...", "...", "...", "..."]
}

Habit output:
{
  "habits": [
    {"name": "...", "description": "..."},
    {"name": "...", "description": "..."},
    {"name": "...", "description": "..."}
  ]
}
```

---

## 2) LLM adapter

Use a single adapter function so the rest of your app does not care which LLM you picked.

```js
export async function callLLM({
  endpoint,
  apiKey,
  model,
  system,
  user
}) {
  const res = await fetch(endpoint, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      ...(apiKey ? { "Authorization": `Bearer ${apiKey}` } : {})
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: "system", content: system },
        { role: "user", content: user }
      ]
    })
  });

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`LLM request failed: ${res.status} ${text}`);
  }

  const data = await res.json();

  const text =
    data?.choices?.[0]?.message?.content ??
    data?.content?.[0]?.text ??
    "";

  return String(text).trim();
}
```

### Notes
- This works with many OpenAI-compatible endpoints.
- Some providers use slightly different response shapes.
- If needed, add a small mapping layer for your chosen provider.

---

## 3) JSON parsing helper

```js
export function parseJSON(raw) {
  const cleaned = raw
    .replace(/```json\s*/gi, "")
    .replace(/```\s*/gi, "")
    .trim();

  const match = cleaned.match(/\{[\s\S]*\}/);
  const jsonText = match ? match[0] : cleaned;

  return JSON.parse(jsonText);
}
```

---

## 4) Micro-skill generation

```js
export async function generateMicroskills({ callLLM, systemPrompt, goal, context }) {
  const user = `Development area: ${goal}\nContext from the person: ${context}`;
  const raw = await callLLM({ system: systemPrompt, user });
  const parsed = parseJSON(raw);
  return parsed.microskills || [];
}
```

---

## 5) Habit generation

```js
export async function generateHabits({ callLLM, systemPrompt, microskill }) {
  const user = `Micro-skill: ${microskill}`;
  const raw = await callLLM({ system: systemPrompt, user });
  const parsed = parseJSON(raw);
  return parsed.habits || [];
}
```

---

## 6) Coaching prompt generation

```js
export function buildCoachPrompt({ goal, microskill, habitName, habitDesc, trigger }) {
  return `You are my personal development coach. Your job is to help me build one specific micro-skill through deliberate daily practice.

Development area: ${goal}
Target micro-skill: ${microskill}

Habit: ${habitName}
Practice: ${habitDesc}
${trigger ? `Activation trigger: When ${trigger}, I practice.` : ""}

Rules:
- Always work from the specific micro-skill.
- When I am vague, demand specifics.
- When I rationalize missed practice, name it plainly.
- Never end a session without a concrete next step.
- Missed days are data.`;
}
```

---

## 7) State model

Keep the state small and serializable.

```js
export const defaultState = {
  phase: "goal",
  goal: "",
  context: "",
  skills: [],
  selectedSkillIndex: null,
  habits: [],
  selectedHabitIndex: null,
  trigger: "",
  tracker: {},
  coachPrompt: ""
};
```

---

## 8) Minimal browser storage

```js
const KEY = "micro-skill-coach-state";

export const storage = {
  load() {
    try {
      const raw = localStorage.getItem(KEY);
      return raw ? JSON.parse(raw) : null;
    } catch {
      return null;
    }
  },
  save(state) {
    localStorage.setItem(KEY, JSON.stringify(state));
  },
  clear() {
    localStorage.removeItem(KEY);
  }
};
```

---

## 9) Main workflow

```js
import { callLLM } from "./llm-adapter.js";
import { parseJSON } from "./parse.js";
import { storage } from "./storage.js";

async function run() {
  const config = {
    endpoint: "https://your-provider.example/v1/chat/completions",
    apiKey: "",
    model: "your-model-name"
  };

  const systemPrompt = await fetch("./agent-instructions.md").then(r => r.text());

  const goal = "delegation";
  const context = "I lead a team of 8 and struggle to let go of details.";

  const microskills = await generateMicroskills({
    callLLM: (args) => callLLM({ ...config, ...args }),
    systemPrompt,
    goal,
    context
  });

  const habits = await generateHabits({
    callLLM: (args) => callLLM({ ...config, ...args }),
    systemPrompt,
    microskill: microskills[0]
  });

  const coachPrompt = buildCoachPrompt({
    goal,
    microskill: microskills[0],
    habitName: habits[0].name,
    habitDesc: habits[0].description,
    trigger: "I finish my first meeting of the day"
  });

  console.log({ microskills, habits, coachPrompt });
}

run().catch(console.error);
```

---

## 10) Error handling checklist

Make sure your app handles:

- empty responses
- invalid JSON
- API failures
- unsupported model formats
- missing keys or endpoint misconfiguration
- browser storage being unavailable

A simple retry with a clearer prompt is often enough for JSON formatting errors.

---

## 11) Public GitHub deployment advice

If you are publishing this on GitHub Pages:

- do not hardcode secrets in frontend code
- use a backend proxy, serverless function, or user-provided key
- keep the LLM adapter configurable
- document the required endpoint format in the README

---

## 12) README starter text

```text
This project is a portable micro-skill coaching agent.

You choose the model provider, configure the endpoint, and paste in your API key or proxy URL.
The app then:
1. decomposes a development area into micro-skills
2. generates habits for one selected micro-skill
3. creates a reusable coaching prompt
4. tracks 30 days of practice

Files:
- agent-instructions.md
- llm-adapter.js
- storage.js
- app.js

To run:
1. install dependencies
2. configure the endpoint and model
3. open the app
4. enter a development area
5. follow the guided workflow
```

---

## 13) What to customize first

Start by editing:

- `endpoint`
- `model`
- `agent-instructions.md`
- the app's save/load logic
- the print tracker design if you want a different output format
