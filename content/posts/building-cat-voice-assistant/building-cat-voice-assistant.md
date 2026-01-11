---
title: "I made a cat, an orange cat"
date: 2026-01-11
draft: false
tags: ["ai", "llm", "voice-assistant", "langgraph", "whisper", "ollama", "privacy", "local-first"]
description: "Building Cat, a home assistant. Lazy and asks for churu."
---


This post walks through building Cat, a home assistant that runs entirely on your machine. No cloud APIs, no subscriptions. We'll cover the voice loop, agent architecture, and adding vision capabilities.


## What We're Building

A home assistant that can:
- Listen continuously for speech
- Process queries through a local LLM
- Speak responses aloud
- See through a webcam (only when asked)
- Blur faces before any image processing

All of this runs locally using Whisper, Ollama, and Piper TTS.

## Part 1: The Voice Loop

A home assistant is fundamentally a loop: listen, think, speak, repeat.

This sounds simple. It isn't.

### 1.1 The Basic Problem

The microphone picks up the assistant's own voice. If you don't pause recognition during speech output, you get feedback loops. The assistant hears itself, responds to itself, and you've created a very confused robot.

The solution is a choreography of pause and resume:

```python
# Listen for speech
text = recognizer.listen()

# Pause recognizer before speaking
recognizer.pause()

# Speak the response
synthesizer.speak(response)

# Resume listening
recognizer.resume()
```

This seems obvious in hindsight. It took longer than I'd like to admit to figure out.

### 1.2 The Components

We need three things:

1. **Speech Recognition** - Whisper runs locally and handles this well
2. **LLM** - Ollama serves local models like llama3.1
3. **Text-to-Speech** - pyttsx3 works but sounds robotic; Piper sounds human

The first version used pyttsx3. Functional, but the voice sounded like a GPS from 2008. Piper is a neural TTS that runs locally and actually sounds like a person. It also provides amplitude callbacks, which become useful later.

### 1.3 The VoiceAssistant Class

Here's the core loop:

```python
class VoiceAssistant:
    def run(self):
        print("Listening...")
        
        while self._running:
            text = self.recognizer.listen()
            
            if text is None:
                continue  # No speech detected
            
            # Check for exit commands
            if self._is_exit_command(text):
                self.synthesizer.speak("Goodbye!")
                break
            
            # Process through LLM
            response = self.processor.process(text)
            
            # Speak response (with pause/resume)
            self.recognizer.pause()
            self.synthesizer.speak(response)
            self.recognizer.resume()
```

This works. But it's limited. The assistant can only have conversations. It can't do anything else.

## Part 2: From Simple Loop to Agent

I wanted the assistant to do more than chat. Check the weather. Control lights. See what's in front of it. This requires tools.

### 2.1 The ReAct Mistake

LangGraph provides a ReAct agent pattern. Give it tools, let it reason about which to use. Sounds perfect.

It wasn't.

The agent would listen, then decide to listen again. Or speak, then speak again. Or reason forever about whether it should listen or speak. ReAct is designed for exploration and problem-solving. A home assistant needs reliability. When you say "turn off the lights," you don't need the agent to contemplate the nature of darkness.

### 2.2 The StateGraph Solution

I replaced ReAct with a deterministic StateGraph:

```
listen → analyze → [route] → respond/vision/exit → react → speak → END
```

The `analyze` node classifies intent into categories: conversational, vision, or exit. No ambiguity. The graph knows exactly where to go.

```python
from langgraph.graph import StateGraph, END

class AgentState(TypedDict):
    user_input: str
    intent: str
    response: str
    tool_response: str

def route_by_intent(state: AgentState) -> str:
    intent = state.get("intent", "conversational")
    if intent == "exit":
        return "goodbye"
    elif intent == "vision":
        return "vision"
    return "respond"

# Build the graph
graph = StateGraph(AgentState)
graph.add_node("listen", listen_node)
graph.add_node("analyze", analyze_node)
graph.add_node("respond", respond_node)
graph.add_node("vision", vision_node)
graph.add_node("react", react_node)
graph.add_node("speak", speak_node)
graph.add_node("goodbye", goodbye_node)

graph.set_entry_point("listen")
graph.add_edge("listen", "analyze")
graph.add_conditional_edges("analyze", route_by_intent)
graph.add_edge("respond", "speak")
graph.add_edge("vision", "react")
graph.add_edge("react", "speak")
graph.add_edge("speak", END)
graph.add_edge("goodbye", END)
```

Boring? Yes. Predictable? Exactly the point.

### 2.3 Intent Classification

The `analyze` node uses the LLM to classify intent:

```python
def analyze_node(state: AgentState) -> dict:
    user_input = state["user_input"]
    
    prompt = """Classify this input into one category:
    - "exit" if the user wants to stop (goodbye, quit, exit, stop)
    - "vision" if the user asks about what you can see
    - "conversational" for everything else
    
    Input: {input}
    Category:"""
    
    result = llm.invoke(prompt.format(input=user_input))
    return {"intent": result.strip().lower()}
```

This is simple classification, not reasoning. The LLM picks a category, the graph handles the rest.

## Part 3: Adding Vision

Cat can see. But only when asked.

### 3.1 Blind-First Design

This was deliberate. The camera is a tool, not a surveillance system. "What do you see?" activates vision. Everything else? Cat is blind.

The intent classifier routes vision-related queries to the vision node. All other queries never touch the camera.

### 3.2 From YOLO to Multimodal LLM

I started with YOLO for object detection:

```
"I see: person (0.87), chair (0.72), laptop (0.91)"
```

Accurate, but mechanical. Nobody wants their assistant to sound like an inventory scanner.

I switched to a multimodal LLM (ministral-3:3b via Ollama). Same camera capture, but natural descriptions:

```
"Looks like you're at your desk. Coffee cup nearby, laptop open."
```

Much better.

### 3.3 The Vision Pipeline

```python
class VisionDescriber:
    def get_scene_description(self) -> str:
        # 1. Capture frame from webcam
        image_path = self._capture_frame()
        
        # 2. Send to multimodal LLM
        description = self._describe_with_ollama(image_path)
        
        return description
    
    def _capture_frame(self) -> str:
        cap = cv2.VideoCapture(self._camera_index)
        
        # Warm up camera (first frames are often dark)
        for _ in range(10):
            cap.read()
        
        ret, frame = cap.read()
        
        # Save to captures directory
        filepath = self._captures_dir / f"capture_{timestamp}.jpg"
        cv2.imwrite(str(filepath), frame)
        
        cap.release()
        return str(filepath)
    
    def _describe_with_ollama(self, image_path: str) -> str:
        image_b64 = self._encode_image_base64(image_path)
        
        payload = {
            "model": "minicpm-v",
            "prompt": "Describe what you see in 1-2 sentences.",
            "images": [image_b64],
        }
        
        response = requests.post(f"{self._ollama_url}/api/generate", json=payload)
        return response.json()["response"]
```

### 3.4 Privacy: Face Blurring

Before any image reaches the LLM, faces get blurred:

```python
def _blur_faces_in_image(self, image):
    detector = MTCNN()
    faces = detector.detect_faces(cv2.cvtColor(image, cv2.COLOR_BGR2RGB))
    
    for face in faces:
        x, y, w, h = face['box']
        roi = image[y:y+h, x:x+w]
        blurred = cv2.GaussianBlur(roi, (99, 99), 30)
        image[y:y+h, x:x+w] = blurred
```

The LLM never sees identifiable faces. The blurred image is what gets saved, what gets analyzed, what exists. Privacy isn't a feature here. It's the architecture.

## Part 4: The React Node

Raw tool output isn't a response.

If you ask "is someone breaking in?" and vision returns "I see a person in the living room," that's not helpful. Is it a burglar? Your roommate? You, reflected in a mirror?

The react node combines the user's query with the tool output:

```python
def react_node(state: AgentState) -> dict:
    query = state["user_input"]
    tool_response = state.get("tool_response", "")
    
    prompt = f"""The user asked: {query}
    You observed: {tool_response}
    
    Respond naturally to their question based on what you observed."""
    
    response = llm.invoke(prompt)
    return {"response": response}
```

Now "is there theft?" + "person in living room" becomes "Just your roommate on the couch. All clear."

## Part 5: The Animated Cat

Every assistant needs personality. Mine is an orange cat.

### 5.1 Why Pygame

Pygame can draw simple graphics and runs on macOS. The catch: macOS requires pygame in the main thread. So the cat animation runs in main, agent logic runs in a background thread.

### 5.2 Amplitude-Based Animation

The cat has three mouth states based on audio amplitude:

```python
class CatAnimator:
    THRESHOLD_HALF = 0.1
    THRESHOLD_OPEN = 0.4
    
    def _get_mouth_state(self) -> str:
        amp = self._amplitude
        
        if amp < self.THRESHOLD_HALF:
            return "closed"
        elif amp < self.THRESHOLD_OPEN:
            return "half"
        return "open"
```

Piper TTS provides amplitude callbacks during speech. The callback updates the animator's amplitude value. The pygame loop reads this value and draws the appropriate mouth state.

```python
def amplitude_callback(amp: float):
    animator.set_amplitude(amp)

synthesizer = PiperSynthesizer(
    model_path="voice_models/en_US-lessac-medium.onnx",
    amplitude_callback=amplitude_callback
)
```

The cat's mouth moves with the speech. Simple, but it makes the assistant feel alive.

## The Complete Stack

All local, no cloud dependencies:

| Component | Tool |
|-----------|------|
| Speech Recognition | Whisper |
| Conversation LLM | Ollama (llama3.1) |
| Vision LLM | Ollama (minicpm-v) |
| Text-to-Speech | Piper |
| Face Detection | MTCNN |
| Animation | Pygame |
| Orchestration | LangGraph StateGraph |

## Running the Assistant

```bash
python -m cat.src.main \
    --agent-mode \
    --tts-backend piper \
    --piper-model cat/voice_models/en_US-lessac-medium.onnx \
    --enable-vision \
    --show-cat
```

## What's Next

Cat is a foundation. The vision system opens possibilities:

- **Motion detection** for security alerts
- **IoT integration** with lights, thermostat, locks
- **Multi-room awareness** with multiple cameras
- **Voice-activated routines**

The goal isn't to build Alexa. It's to build something that runs on my machine, that I understand, that I control.

## Lessons

1. **Deterministic beats clever** for daily-use tools. Save the reasoning for problems that need it.

2. **Privacy is architecture**, not a feature. If the data never exists unblurred, it can't leak.

3. **Start with the interaction loop**. Everything else is details.

4. **Dead ends are data**. YOLO taught me I wanted natural descriptions. ReAct taught me I wanted reliability.

---

That's it. Cat listens, thinks, sees (when asked), and speaks. All without leaving my machine.

The code is available on GitHub: https://github.com/Gauravpadam/Cat
