# VoiceX Framework Architecture

## System Status: ✅ FULLY OPERATIONAL

**Last Verified**: All architectural components tested and working correctly  
**Current Deployment**: Local development environment with live agents  

## Overview

VoiceX Framework is a verified, state-of-the-art conversational AI system built on LiveKit with OpenAI and Eleven Labs V3 integration, specifically optimized for IT sales voice agents.

**Operational Agents**: 1 Swyt Solutions BDR (Business Development Representative)  
**System Health**: All components verified and functional

## Verified System Components

### ✅ Operational Status
- **LiveKit Server**: Running on localhost:7880
- **Agent Registry**: 1 agent registered (Swyt Solutions BDR)
- **Configuration System**: All API keys validated
- **STT Integration**: OpenAI Whisper ready
- **TTS Integration**: Eleven Labs Flash V2.5 (~75ms latency)
- **LLM Integration**: OpenAI GPT-4o-mini operational
- **Voice Processing**: Adam voice (pNInz6obpgDQGcFmaJgB) configured

### 🔧 Technical Stack (Verified)
- **Framework**: LiveKit Agents Framework
- **Voice Processing**: Eleven Labs V3 with Flash V2.5 model
- **Conversation AI**: OpenAI GPT-4o-mini
- **Audio Enhancement**: Audio tags for expressive speech
- **Session Management**: Full conversation state tracking
- **Sales Intelligence**: Lead qualification and scoring

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        UI[LiveKit UI]
        WebClient[Web Client]
        Mobile[Mobile App]
    end
    
    subgraph "VoiceX Framework Core"
        Registry[Agent Registry]
        Session[Agent Session Manager]
        Agent[Voice Agent Core]
    end
    
    subgraph "Audio Processing Pipeline"
        VAD[Voice Activity Detection]
        Noise[Noise Reduction]
        STT[Speech-to-Text]
        TTS[Text-to-Speech]
    end
    
    subgraph "AI Services"
        OpenAI[OpenAI GPT-4]
        ElevenLabs[Eleven Labs V3/Flash]
    end
    
    subgraph "LiveKit Infrastructure"
        LKServer[LiveKit Server]
        LKRoom[Room Management]
        LKMedia[Media Routing]
    end
    
    subgraph "Configuration & Context"
        Config[Environment Config]
        Context[Agent Context Files]
        Prompts[Prompt Templates]
    end
    
    UI --> LKServer
    WebClient --> LKServer
    Mobile --> LKServer
    
    LKServer --> Registry
    Registry --> Session
    Session --> Agent
    
    Agent --> VAD
    VAD --> Noise
    Noise --> STT
    STT --> OpenAI
    OpenAI --> TTS
    TTS --> LKMedia
    
    STT --> OpenAI
    TTS --> ElevenLabs
    
    Agent --> Context
    Agent --> Config
    Context --> Prompts
    
    LKMedia --> UI
    LKMedia --> WebClient
    LKMedia --> Mobile
```

## Component Details

### 1. Agent Core Layer

```mermaid
classDiagram
    class VoiceAgent {
        +AgentConfig config
        +AgentPersonality personality
        +AgentCapabilities capabilities
        +ChatContext chat_context
        +initialize_context(path)
        +handle_user_query(query)
        +start_session(ctx)
        +end_session()
    }
    
    class ITSupportAgent {
        +prospect_company: str
        +employee_count: int
        +qualification_score: int
        +current_sales_stage: str
        +identified_pain_points: List
        +qualify_prospect(query)
        +handle_objections(objection)
        +schedule_demo()
    }
    
    class AgentSession {
        +agent_name: str
        +session_id: str
        +metrics: SessionMetrics
        +start(ctx)
        +_setup_agent_events()
        +end_session()
    }
    
    class AgentRegistry {
        +agents: Dict
        +active_sessions: Dict
        +register_agent()
        +create_agent_instance()
        +track_session()
        +health_check_all()
    }
    
    VoiceAgent <|-- ITSupportAgent
    AgentSession --> VoiceAgent
    AgentRegistry --> AgentSession
```

### 2. Audio Processing Pipeline

```mermaid
sequenceDiagram
    participant User
    participant LiveKit
    participant VAD as Voice Activity Detection
    participant Noise as Noise Reduction
    participant STT as Speech-to-Text (OpenAI)
    participant Agent as Voice Agent
    participant LLM as Language Model (GPT-4)
    participant TTS as Text-to-Speech (ElevenLabs)
    
    User->>LiveKit: Audio Stream
    LiveKit->>VAD: Raw Audio Chunks
    VAD->>VAD: Detect Voice Activity
    
    alt Voice Detected
        VAD->>Noise: Audio Chunk
        Noise->>Noise: Apply Noise Reduction
        Noise->>STT: Clean Audio
        STT->>STT: Transcribe to Text
        STT->>Agent: User Query Text
        
        Agent->>Agent: Process with Context
        Agent->>LLM: Enhanced Prompt + Query
        LLM->>Agent: Response Text
        
        Agent->>TTS: Response + Voice Config
        TTS->>TTS: Apply Audio Tags ([confident], [empathetic])
        TTS->>TTS: Generate with Flash V2.5/V3
        TTS->>LiveKit: Audio Stream
        LiveKit->>User: Agent Response
    else Silence
        VAD->>VAD: Continue Monitoring
    end
```

### 3. STT Layer Architecture

```mermaid
graph LR
    subgraph "STT Processing"
        AudioIn[Raw Audio] --> Format[Audio Format Detection]
        Format --> OpenAISTT[OpenAI Whisper API]
        
        subgraph "OpenAI STT Config"
            Model[whisper-1]
            Lang[Language: en]
            ResponseFormat[text/json]
        end
        
        OpenAISTT --> Model
        OpenAISTT --> Lang
        OpenAISTT --> ResponseFormat
        
        OpenAISTT --> Transcription[Text Output]
        
        subgraph "Error Handling"
            RateLimit[Rate Limit Retry]
            Timeout[Timeout Handling]
            Fallback[Fallback Strategies]
        end
        
        OpenAISTT --> RateLimit
        OpenAISTT --> Timeout
        OpenAISTT --> Fallback
    end
```

### 4. TTS Layer Architecture

```mermaid
graph TD
    subgraph "TTS Processing Pipeline"
        TextIn[Input Text] --> Enhancement[Audio Tag Enhancement]
        Enhancement --> ModelSelect{Select Model}
        
        ModelSelect -->|Real-time| FlashV25[Eleven Flash V2.5<br/>~75ms latency]
        ModelSelect -->|High Quality| V3Model[Eleven V3<br/>Maximum expressiveness]
        
        subgraph "Voice Configuration"
            VoiceID[Voice ID: Adam/Bella/Antoni]
            Stability[Stability: 0.5]
            Similarity[Similarity Boost: 0.8]
            Style[Style: 0.0]
            SpeakerBoost[Speaker Boost: true]
        end
        
        FlashV25 --> VoiceConfig[Apply Voice Settings]
        V3Model --> VoiceConfig
        
        VoiceConfig --> VoiceID
        VoiceConfig --> Stability
        VoiceConfig --> Similarity
        VoiceConfig --> Style
        VoiceConfig --> SpeakerBoost
        
        VoiceConfig --> AudioGen[Generate Audio]
        AudioGen --> AudioOut[PCM/MP3 Stream]
        
        subgraph "Audio Tags Enhancement"
            Excited["[excited] Great!"]
            Empathetic["[empathetic] I understand"]
            Confident["[confident] Here's what we'll do"]
            Thoughtful["[thoughtful] Let me think"]
        end
        
        Enhancement --> Excited
        Enhancement --> Empathetic
        Enhancement --> Confident
        Enhancement --> Thoughtful
    end
```

### 5. LLM Layer Architecture

```mermaid
graph TB
    subgraph "LLM Processing"
        UserQuery[User Query] --> ContextBuilder[Build Enhanced Context]
        
        subgraph "Context Components"
            SystemPrompt[System Prompt<br/>Agent Identity & Instructions]
            ConversationHistory[Chat History]
            SalesStage[Current Sales Stage]
            QualificationNotes[Qualification Data]
        end
        
        ContextBuilder --> SystemPrompt
        ContextBuilder --> ConversationHistory
        ContextBuilder --> SalesStage
        ContextBuilder --> QualificationNotes
        
        ContextBuilder --> OpenAIGPT[OpenAI GPT-4o-mini]
        
        subgraph "GPT Configuration"
            Temperature[Temperature: 0.7]
            MaxTokens[Max Tokens: 1000]
            Model[gpt-4o-mini]
        end
        
        OpenAIGPT --> Temperature
        OpenAIGPT --> MaxTokens
        OpenAIGPT --> Model
        
        OpenAIGPT --> ResponseAnalysis[Analyze Response]
        ResponseAnalysis --> UpdateState[Update Sales State]
        ResponseAnalysis --> FinalResponse[Agent Response]
        
        subgraph "Sales State Updates"
            QualificationScore[Update Qualification Score]
            PainPoints[Identify Pain Points]
            NextStage[Determine Next Stage]
        end
        
        UpdateState --> QualificationScore
        UpdateState --> PainPoints
        UpdateState --> NextStage
    end
```

## Prompting Strategy Architecture

### 1. System Prompt Structure

```mermaid
graph TD
    subgraph "System Prompt Components"
        Identity[Agent Identity<br/>Sarah Mitchell, BDR at Swyt Solutions]
        Company[Company Mission<br/>IT Outsourcing, 40% Cost Reduction]
        Personality[Personality Traits<br/>Professional, Consultative, Results-oriented]
        
        subgraph "Sales Framework"
            Goals[Core Objectives<br/>Qualify prospects, Schedule demos]
            Flow[Conversation Flow<br/>Opening → Qualifying → Closing]
            Objections[Objection Handling<br/>Structured responses]
        end
        
        subgraph "Behavioral Guidelines"
            Questions[Question Strategy<br/>One at a time, consultative]
            Language[Professional Language<br/>Business value focused]
            Disqualification[Disqualification Criteria<br/>Clear exit conditions]
        end
        
        Identity --> Company
        Company --> Personality
        Personality --> Goals
        Goals --> Flow
        Flow --> Objections
        Objections --> Questions
        Questions --> Language
        Language --> Disqualification
    end
```

### 2. Context Enhancement Flow

```mermaid
sequenceDiagram
    participant User
    participant Agent
    participant ContextBuilder
    participant LLM
    
    User->>Agent: "We have 50 employees"
    Agent->>ContextBuilder: Analyze Query
    
    ContextBuilder->>ContextBuilder: Extract: employee_count=50
    ContextBuilder->>ContextBuilder: Update: qualification_score +20
    ContextBuilder->>ContextBuilder: Set: current_stage=qualifying
    
    ContextBuilder->>LLM: Enhanced Context:<br/>PROSPECT SIZE: 50 employees (QUALIFIED)<br/>CURRENT STAGE: qualifying<br/>NEXT QUESTIONS: IT structure, pain points
    
    LLM->>Agent: "That's a great size for our solution. With 50 employees, you're exactly in our sweet spot. Tell me, how do you currently handle IT onboarding when you hire new people?"
    
    Agent->>User: Response with context-aware follow-up
```

## Data Flow Architecture

```mermaid
graph LR
    subgraph "Input Processing"
        Microphone[User Microphone] --> LiveKitRoom[LiveKit Room]
        LiveKitRoom --> AudioChunks[Audio Chunks]
        AudioChunks --> VAD[Voice Activity Detection]
        VAD --> NoiseReduction[Noise Reduction]
        NoiseReduction --> STTProcess[STT Processing]
    end
    
    subgraph "AI Processing"
        STTProcess --> TextQuery[Text Query]
        TextQuery --> ContextEnhancement[Context Enhancement]
        ContextEnhancement --> LLMProcessing[LLM Processing]
        LLMProcessing --> ResponseGeneration[Response Generation]
        ResponseGeneration --> AudioTagsEnhancement[Audio Tags Enhancement]
        AudioTagsEnhancement --> TTSProcessing[TTS Processing]
    end
    
    subgraph "Output Processing"
        TTSProcessing --> AudioStream[Audio Stream]
        AudioStream --> LiveKitMedia[LiveKit Media Server]
        LiveKitMedia --> UserSpeakers[User Speakers]
    end
    
    subgraph "State Management"
        SessionState[Session State]
        QualificationData[Qualification Data]
        ConversationHistory[Conversation History]
    end
    
    ContextEnhancement --> SessionState
    LLMProcessing --> QualificationData
    ResponseGeneration --> ConversationHistory
```

## Configuration Architecture

```mermaid
graph TB
    subgraph "Configuration Sources"
        EnvFile[.env File<br/>API Keys & Settings]
        ContextYAML[context.yaml<br/>Agent Configuration]
        InstructionsTXT[instructions.txt<br/>Detailed Instructions]
    end
    
    subgraph "Configuration Classes"
        AgentConfig[AgentConfig<br/>Complete configuration]
        OpenAIConfig[OpenAI Settings]
        ElevenLabsConfig[ElevenLabs Settings]
        AudioConfig[Audio Settings]
    end
    
    subgraph "Runtime Configuration"
        ConfigManager[Config Manager]
        AgentRegistry[Agent Registry]
        SessionConfig[Session Configuration]
    end
    
    EnvFile --> ConfigManager
    ContextYAML --> ConfigManager
    InstructionsTXT --> ConfigManager
    
    ConfigManager --> AgentConfig
    AgentConfig --> OpenAIConfig
    AgentConfig --> ElevenLabsConfig
    AgentConfig --> AudioConfig
    
    ConfigManager --> AgentRegistry
    AgentRegistry --> SessionConfig
```

## Performance & Scalability

### Latency Optimization

```mermaid
graph LR
    subgraph "Latency Targets"
        UserSpeech[User Speech] -->|<100ms| VADDetection[VAD Detection]
        VADDetection -->|<500ms| STTProcessing[STT Processing]
        STTProcessing -->|<1000ms| LLMResponse[LLM Response]
        LLMResponse -->|<75ms| TTSGeneration[TTS Generation]
        TTSGeneration -->|<50ms| AudioPlayback[Audio Playback]
    end
    
    subgraph "Optimization Strategies"
        StreamingSTT[Streaming STT]
        FlashTTS[Flash V2.5 TTS]
        ContextCaching[Context Caching]
        ParallelProcessing[Parallel Processing]
    end
    
    VADDetection -.-> StreamingSTT
    STTProcessing -.-> StreamingSTT
    LLMResponse -.-> ContextCaching
    TTSGeneration -.-> FlashTTS
    AudioPlayback -.-> ParallelProcessing
```

### Scalability Architecture

```mermaid
graph TB
    subgraph "Load Balancing"
        LoadBalancer[Load Balancer]
        Agent1[Agent Instance 1]
        Agent2[Agent Instance 2]
        Agent3[Agent Instance N]
    end
    
    subgraph "Session Management"
        SessionStore[Session Store]
        AgentRegistry[Agent Registry]
        MetricsCollector[Metrics Collector]
    end
    
    subgraph "External Services"
        LiveKitCluster[LiveKit Cluster]
        OpenAIAPI[OpenAI API]
        ElevenLabsAPI[ElevenLabs API]
    end
    
    LoadBalancer --> Agent1
    LoadBalancer --> Agent2
    LoadBalancer --> Agent3
    
    Agent1 --> SessionStore
    Agent2 --> SessionStore
    Agent3 --> SessionStore
    
    Agent1 --> LiveKitCluster
    Agent2 --> LiveKitCluster
    Agent3 --> LiveKitCluster
    
    SessionStore --> AgentRegistry
    SessionStore --> MetricsCollector
    
    Agent1 --> OpenAIAPI
    Agent1 --> ElevenLabsAPI
```

## Security Architecture

```mermaid
graph TB
    subgraph "API Security"
        APIKeys[Encrypted API Keys]
        RateLimit[Rate Limiting]
        TokenValidation[Token Validation]
    end
    
    subgraph "Data Security"
        AudioEncryption[Audio Stream Encryption]
        ConversationEncryption[Conversation Encryption]
        PIIScrubbing[PII Scrubbing]
    end
    
    subgraph "Access Control"
        Authentication[Authentication]
        Authorization[Authorization]
        SessionValidation[Session Validation]
    end
    
    subgraph "Monitoring"
        SecurityLogs[Security Logging]
        AuditTrails[Audit Trails]
        ThreatDetection[Threat Detection]
    end
    
    APIKeys --> RateLimit
    RateLimit --> TokenValidation
    
    AudioEncryption --> ConversationEncryption
    ConversationEncryption --> PIIScrubbing
    
    Authentication --> Authorization
    Authorization --> SessionValidation
    
    SecurityLogs --> AuditTrails
    AuditTrails --> ThreatDetection
```

This architecture ensures high performance, scalability, and security while maintaining the flexibility to add new agents and capabilities to the VoiceX Framework.
