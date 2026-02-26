# Mastra Voice & Streaming

## Voice Providers

| Provider | Package | Features |
|----------|---------|----------|
| OpenAI | `@mastra/voice-openai` | TTS + STT |
| OpenAI Realtime | `@mastra/voice-openai-realtime` | Real-time STS |
| ElevenLabs | `@mastra/voice-elevenlabs` | High-quality TTS |
| PlayAI | `@mastra/voice-playai` | TTS |
| Google | `@mastra/voice-google` | TTS + STT |
| Google Gemini Live | `@mastra/voice-google-gemini-live` | Real-time STS |
| Deepgram | `@mastra/voice-deepgram` | STT |
| Azure | `@mastra/voice-azure` | TTS + STT |
| Cloudflare | `@mastra/voice-cloudflare` | TTS |
| Murf | `@mastra/voice-murf` | TTS |
| Speechify | `@mastra/voice-speechify` | TTS |
| Sarvam | `@mastra/voice-sarvam` | TTS + STT |

## Basic TTS

```typescript
import { Agent } from "@mastra/core/agent";
import { OpenAIVoice } from "@mastra/voice-openai";
import { playAudio } from "@mastra/node-audio";

const agent = new Agent({
  id: "voice-agent",
  model: "openai/gpt-5.1",
  voice: new OpenAIVoice(),
});

const { text } = await agent.generate("What color is the sky?");
const audio = await agent.voice.speak(text, { speaker: "alloy" });
playAudio(audio);
```

## Basic STT

```typescript
import { getMicrophoneStream } from "@mastra/node-audio";

const audio = getMicrophoneStream();
const transcript = await agent.voice.listen(audio, { filetype: "m4a" });
console.log(`User said: ${transcript}`);
```

## Real-Time Speech-to-Speech

### OpenAI Realtime
```typescript
import { OpenAIRealtimeVoice } from "@mastra/voice-openai-realtime";

const agent = new Agent({
  id: "realtime-agent",
  model: "openai/gpt-5.1",
  voice: new OpenAIRealtimeVoice({ speaker: "alloy" }),
});

await agent.voice.connect();

agent.voice.on("speaker", ({ audio }) => playAudio(audio));
agent.voice.on("writing", ({ text, role }) => console.log(`${role}: ${text}`));

await agent.voice.speak("How can I help?");
const mic = getMicrophoneStream();
await agent.voice.send(mic);
```

### Google Gemini Live
```typescript
import { GeminiLiveVoice } from "@mastra/voice-google-gemini-live";

const agent = new Agent({
  id: "gemini-agent",
  model: "openai/gpt-5.1",
  voice: new GeminiLiveVoice({
    apiKey: process.env.GOOGLE_API_KEY,
    model: "gemini-2.0-flash-exp",
    speaker: "Puck",
  }),
});
```

## Composite Voice (Mixed Providers)

```typescript
import { CompositeVoice } from "@mastra/core/voice";
import { OpenAIVoice } from "@mastra/voice-openai";
import { PlayAIVoice } from "@mastra/voice-playai";

const agent = new Agent({
  id: "composite-agent",
  model: "openai/gpt-5.1",
  voice: new CompositeVoice({
    input: new OpenAIVoice(),   // STT
    output: new PlayAIVoice(),  // TTS
  }),
});
```

## AI SDK Voice Integration

```typescript
import { CompositeVoice } from "@mastra/core/voice";
import { openai } from "@ai-sdk/openai";
import { elevenlabs } from "@ai-sdk/elevenlabs";

const voice = new CompositeVoice({
  input: openai.transcription("whisper-1"),
  output: elevenlabs.speech("eleven_turbo_v2"),
});
```

## Streaming Events

### Agent Stream Events
- `start` — Agent begins execution
- `step-start` — Step begins
- `text-delta` — Incremental text
- `tool-call` — Tool invocation
- `tool-result` — Tool result
- `step-finish` — Step completes
- `finish` — Agent completes

### Workflow Stream Events
- `workflow-start` — Workflow begins
- `workflow-step-start` — Step begins
- `workflow-step-progress` — Foreach progress
- `workflow-step-finish` — Step completes
- `workflow-finish` — Workflow completes

### Network Stream Events
- `routing-agent-start/end` — Router selection
- `agent-execution-start/end` — Delegated agent
- `workflow-execution-start/end` — Delegated workflow
- `tool-execution-start/end` — Tool call
- `network-execution-event-step-finish` — Final result

### Inspecting Streams

```typescript
// Agent
const stream = await agent.stream("Hello");
for await (const chunk of stream) {
  if (chunk.type === "text-delta") console.log(chunk.payload.delta);
}

// Workflow
const run = await workflow.createRun();
const stream = await run.stream({ inputData: { value: "data" } });
for await (const chunk of stream) {
  if (chunk.type === "workflow-step-progress") {
    console.log(`${chunk.payload.completedCount}/${chunk.payload.totalCount}`);
  }
}

// Network
const stream = await networkAgent.network("Research dolphins");
for await (const chunk of stream) {
  if (chunk.type === "routing-agent-end") {
    console.log("Selected:", chunk.payload.resourceId);
  }
}
console.log("Result:", await stream.result);
```

## Tool Streaming

```typescript
const tool = createTool({
  execute: async (input, context) => {
    await context?.writer?.write({ type: "progress", status: "pending" });
    const result = await processData(input);
    await context?.writer?.custom({ type: "done", status: "success" });
    return { value: result };
  },
});
```

## Tool Lifecycle Hooks

```typescript
const tool = createTool({
  id: "weather",
  onInputStart: ({ toolCallId }) => console.log("Input streaming started"),
  onInputDelta: ({ delta }) => console.log("Input chunk:", delta),
  onInputAvailable: ({ input }) => console.log("Complete input:", input),
  execute: async (input) => fetchWeather(input.city),
  onOutput: ({ output }) => console.log("Result:", output),
});
```
