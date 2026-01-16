# Live2D Avatar Integration Architecture

## Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                              App.tsx                                │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │             DeepTreeEchoAvatarProvider                        │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │                  ScreenController                       │ │ │
│  │  │  ┌───────────────────────────────────────────────────┐ │ │ │
│  │  │  │          MessageListView                         │ │ │ │
│  │  │  │  ┌─────────────────────────────────────────────┐ │ │ │ │
│  │  │  │  │     MessageListAndComposer                  │ │ │ │ │
│  │  │  │  │  ┌───────────────────────────────────────┐ │ │ │ │ │
│  │  │  │  │  │       MessageList                     │ │ │ │ │ │
│  │  │  │  │  └───────────────────────────────────────┘ │ │ │ │ │
│  │  │  │  │  ┌───────────────────────────────────────┐ │ │ │ │ │
│  │  │  │  │  │       Composer                        │ │ │ │ │ │
│  │  │  │  │  └───────────────────────────────────────┘ │ │ │ │ │
│  │  │  │  │  ┌───────────────────────────────────────┐ │ │ │ │ │
│  │  │  │  │  │  DeepTreeEchoAvatarDisplay (FLOATING) │ │ │ │ │ │
│  │  │  │  │  │  ┌─────────────────────────────────┐ │ │ │ │ │ │
│  │  │  │  │  │  │     Live2DAvatar              │ │ │ │ │ │ │
│  │  │  │  │  │  │  (@deltecho/avatar)           │ │ │ │ │ │ │
│  │  │  │  │  │  │  - Model: Miara               │ │ │ │ │ │ │
│  │  │  │  │  │  │  - Expressions: 9 types       │ │ │ │ │ │ │
│  │  │  │  │  │  │  - Motions: 7 types           │ │ │ │ │ │ │
│  │  │  │  │  │  │  - Lip Sync: Active           │ │ │ │ │ │ │
│  │  │  │  │  │  └─────────────────────────────────┘ │ │ │ │ │ │
│  │  │  │  │  │  ┌─────────────────────────────────┐ │ │ │ │ │ │
│  │  │  │  │  │  │   Status Indicator            │ │ │ │ │ │ │
│  │  │  │  │  │  │   - Idle / Listening          │ │ │ │ │ │ │
│  │  │  │  │  │  │   - Thinking / Responding     │ │ │ │ │ │ │
│  │  │  │  │  │  └─────────────────────────────────┘ │ │ │ │ │ │
│  │  │  │  │  └───────────────────────────────────────┘ │ │ │ │ │
│  │  │  │  └─────────────────────────────────────────────┘ │ │ │ │
│  │  │  └───────────────────────────────────────────────────┘ │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Flow

```
┌─────────────────────┐
│   User Sends        │
│   Message           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│              DeepTreeEchoBot.processMessage()           │
│                                                         │
│  1. setAvatarListening() ◄─── AvatarStateManager      │
│  2. Store in RAGMemoryStore                            │
│  3. setAvatarThinking()  ◄─── AvatarStateManager      │
│  4. LLMService.generateResponse()                      │
│  5. setAvatarResponding() + startLipSync()             │
│  6. sendMessage()                                       │
│  7. setAvatarIdle() + stopLipSync()                    │
└────────────┬────────────────────────────────────────────┘
             │
             ▼
   ┌─────────────────────┐
   │ AvatarStateManager   │
   │                     │
   │ setProcessingState  ├──┐
   │ setIsSpeaking       │  │
   │ setAudioLevel       │  │
   └─────────────────────┘  │
             │              │
             ▼              │
┌─────────────────────────────────────────────────┐
│     DeepTreeEchoAvatarContext                   │
│                                                 │
│  - processingState: LISTENING | THINKING |     │
│                     RESPONDING | IDLE           │
│  - isSpeaking: boolean                          │
│  - audioLevel: 0-1                              │
│  - config: { visible, position, size, model }   │
└────────────┬────────────────────────────────────┘
             │
             ▼
   ┌─────────────────────────────────┐
   │  DeepTreeEchoAvatarDisplay      │
   │                                 │
   │  1. Read state from context     │
   │  2. Poll CognitiveBridge        │
   │  3. Map emotional state         │
   │  4. Update expressions          │
   └────────────┬────────────────────┘
                │
                ▼
      ┌─────────────────────┐
      │   Live2DAvatar      │
      │                     │
      │  - Load model       │
      │  - Set expression   │
      │  - Play motion      │
      │  - Update lip sync  │
      └─────────────────────┘
```

## State Transitions

```
                                ┌──────┐
                           ┌────┤ IDLE ├────┐
                           │    └──────┘    │
                           │                │
            User Message   │                │  Response Complete
                           │                │  (after 1s delay)
                           ▼                │
                      ┌───────────┐         │
                      │ LISTENING │         │
                      └─────┬─────┘         │
                            │               │
                 Message    │               │
                 Received   │               │
                            ▼               │
                      ┌──────────┐          │
                      │ THINKING │          │
                      └────┬─────┘          │
                           │                │
                Response   │                │
                Ready      │                │
                           ▼                │
                    ┌─────────────┐         │
                    │ RESPONDING  │─────────┘
                    │ (+ LipSync) │
                    └─────────────┘
                           │
                           │ Error
                           ▼
                      ┌───────┐
                      │ ERROR │
                      └───┬───┘
                          │
               After 3s   │
                          ▼
                      ┌──────┐
                      │ IDLE │
                      └──────┘
```

## Expression Mapping

```
Cognitive State                    Avatar Expression
──────────────────────────────────────────────────────
emotionalValence > 0.5            →  happy
emotionalValence > 0.5            →  playful
  + arousal > 0.4
emotionalValence > 0.5            →  surprised
  + arousal > 0.7

emotionalValence < -0.5           →  concerned
  + arousal > 0.5
emotionalValence < -0.5           →  contemplative
  + arousal low

arousal > 0.6                     →  curious
  (neutral valence)

processingState = THINKING        →  thinking
processingState = RESPONDING      →  focused
processingState = ERROR           →  concerned

default                           →  neutral
```

## Module Dependencies

```
┌──────────────────────────────────────────────────────┐
│              DeepTreeEchoAvatarDisplay               │
└──────┬───────────────────────┬───────────────────────┘
       │                       │
       │                       │
       ▼                       ▼
┌─────────────────┐    ┌──────────────────────┐
│ Live2DAvatar    │    │ CognitiveBridge      │
│ (from AIHub)    │    │                      │
│                 │    │ - getOrchestrator()  │
│ - Model loading │    │ - getState()         │
│ - Expression    │    │ - UnifiedState       │
│ - Motion        │    └──────────────────────┘
│ - Lip sync      │
└─────────────────┘
       │
       ▼
┌──────────────────────┐
│  @deltecho/avatar    │
│                      │
│  - Live2DManager     │
│  - PixiRenderer      │
│  - CubismAdapter     │
│  - Expression Maps   │
└──────────────────────┘
       │
       ▼
┌──────────────────────┐
│  Cubism SDK + PixiJS │
│                      │
│  - Model rendering   │
│  - Parameter control │
│  - Animation system  │
└──────────────────────┘
```

## Configuration Flow

```
┌────────────────────────┐
│  Desktop Settings      │
│                        │
│  deepTreeEchoBotEnabled│
│  deepTreeEchoBotAvatar │
│         Enabled        │
└──────────┬─────────────┘
           │
           ▼
┌────────────────────────┐
│  MessageListAndComposer│
│                        │
│  Check settings        │
│  Render avatar if true │
└──────────┬─────────────┘
           │
           ▼
┌────────────────────────────────┐
│  DeepTreeEchoAvatarContext     │
│                                │
│  Load from localStorage:       │
│  - visible                     │
│  - position (floating/inline)  │
│  - width, height               │
│  - model (miara)               │
└────────────┬───────────────────┘
             │
             ▼
   ┌─────────────────────┐
   │  AvatarDisplay      │
   │                     │
   │  Apply config       │
   │  Render avatar      │
   └─────────────────────┘
```

## Integration Summary

**Total Integration Points:** 5
1. App.tsx - Provider wrapper
2. MessageListAndComposer.tsx - Display mounting
3. DeepTreeEchoBot.ts - State updates
4. CognitiveBridge.ts - State source (read-only)
5. Settings - Configuration source

**State Management Layers:** 3
1. AvatarStateManager (singleton, imperative)
2. DeepTreeEchoAvatarContext (React context)
3. Component state (DeepTreeEchoAvatarDisplay)

**Data Sources:** 2
1. CognitiveBridge (cognitive/emotional state)
2. Processing lifecycle (bot state)

**Rendering Pipeline:** 4 layers
1. React component (DeepTreeEchoAvatarDisplay)
2. Avatar wrapper (Live2DAvatar from AIHub)
3. Avatar manager (@deltecho/avatar)
4. Renderer (PixiJS + Cubism SDK)
