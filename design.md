# AI Civic Helpdesk - System Design Document

## System Architecture Overview

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                        User Access Layer                        │
├─────────────────┬─────────────────┬─────────────────┬───────────┤
│   WhatsApp      │   IVR System    │   USSD/SMS      │  Web App  │
│   (Primary)     │   (Voice)       │   (Feature Ph.) │   (PWA)   │
└─────────────────┴─────────────────┴─────────────────┴───────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                     API Gateway & Load Balancer                 │
│  • Rate Limiting  • Authentication  • Channel Routing           │
│  • Low-bandwidth optimization  • Request/Response caching       │
└─────────────────────────────────────────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                        Core Services Layer                      │
├─────────────────┬─────────────────┬─────────────────┬───────────┤
│  NLP Engine     │  Service Logic  │  User Manager   │ Analytics │
│  • Intent Class │  • Eligibility  │  • Profiles     │ • Metrics │
│  • Entity Extr. │  • Documents    │  • Sessions     │ • Reports │
│  • Multi-lang   │  • Workflows    │  • Preferences  │ • Insights│
└─────────────────┴─────────────────┴─────────────────┴───────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                      Integration Layer                          │
├─────────────────┬─────────────────┬─────────────────┬───────────┤
│  Govt APIs      │  Communication  │  Language APIs  │  Storage  │
│  • DigiLocker   │  • WhatsApp API │  • Translation  │  • Redis  │
│  • Aadhaar      │  • Twilio IVR   │  • Speech APIs  │  • Files  │
│  • PAN Portal   │  • SMS Gateway  │  • TTS/STT      │  • Backup │
└─────────────────┴─────────────────┴─────────────────┴───────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                         Data Layer                              │
├─────────────────┬─────────────────┬─────────────────┬───────────┤
│  User Database  │  Knowledge Base │  Session Store  │ Analytics │
│  • PostgreSQL   │  • Schemes Info │  • Redis Cache  │ • MongoDB │
│  • Encrypted    │  • Documents    │  • User Context │ • Metrics │
│  • Partitioned  │  • Procedures   │  • Conversations│ • Logs    │
└─────────────────┴─────────────────┴─────────────────┴───────────┘
```

## Technology Stack Recommendations

### Backend Services
```yaml
Core Framework: FastAPI (Python)
  - High performance async framework
  - Automatic API documentation
  - Built-in validation and serialization
  - Excellent for ML model integration

Database Stack:
  Primary: PostgreSQL 14+
    - JSONB support for flexible schemas
    - Full-text search capabilities
    - Excellent performance and reliability
  
  Cache: Redis 7+
    - Session management
    - Response caching
    - Real-time data storage
  
  Analytics: MongoDB
    - Flexible schema for analytics data
    - Time-series data handling
    - Aggregation pipeline for insights

Message Queue: Apache Kafka
  - High-throughput message processing
  - Event streaming for real-time updates
  - Reliable message delivery

AI/ML Stack:
  NLP: spaCy + Transformers (Hugging Face)
    - Multi-language support
    - Custom model training
    - Efficient inference
  
  Speech: Google Cloud Speech-to-Text/Text-to-Speech
    - Indian language support
    - High accuracy for regional accents
    - Cost-effective for scale
```

### Frontend Technologies
```yaml
Web Application: React + TypeScript
  - Progressive Web App (PWA) capabilities
  - Offline functionality
  - Responsive design
  - Accessibility features

Mobile Optimization:
  - Service Workers for offline capability
  - Minimal JavaScript bundle
  - Lazy loading components
  - Touch-friendly interface

Low-bandwidth Optimizations:
  - Text-first design
  - Compressed images (WebP format)
  - Minimal CSS framework (Tailwind CSS)
  - Progressive enhancement
```

### Infrastructure
```yaml
Container Orchestration: Kubernetes
  - Auto-scaling capabilities
  - Service mesh for microservices
  - Rolling deployments
  - Resource management

Cloud Provider: AWS/Azure (Government Cloud)
  - Data residency compliance
  - Government-grade security
  - Disaster recovery capabilities
  - Cost optimization tools

CDN: CloudFlare
  - Global content delivery
  - DDoS protection
  - SSL/TLS termination
  - Edge caching
```

## Database Schema Design

### User Management Schema
```sql
-- Users table with minimal data collection
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number VARCHAR(15) UNIQUE NOT NULL,
    phone_verified BOOLEAN DEFAULT FALSE,
    preferred_language VARCHAR(10) DEFAULT 'hi',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Optional profile data (encrypted)
    profile_data JSONB DEFAULT '{}',
    
    -- Privacy and consent
    data_consent BOOLEAN DEFAULT FALSE,
    consent_timestamp TIMESTAMP,
    
    -- Indexes for performance
    INDEX idx_phone_number (phone_number),
    INDEX idx_last_active (last_active),
    INDEX idx_language (preferred_language)
);

-- User sessions for context management
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    channel VARCHAR(20) NOT NULL, -- whatsapp, ivr, web, sms
    session_data JSONB DEFAULT '{}',
    context JSONB DEFAULT '{}',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    
    INDEX idx_user_active (user_id, is_active),
    INDEX idx_expires_at (expires_at)
);

-- Conversation history (with retention policy)
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES user_sessions(id) ON DELETE CASCADE,
    message_type VARCHAR(20) NOT NULL, -- user, bot, system
    content TEXT NOT NULL,
    intent VARCHAR(50),
    entities JSONB DEFAULT '{}',
    confidence_score DECIMAL(3,2),
    language VARCHAR(10),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Partitioning by month for performance
    PARTITION BY RANGE (timestamp),
    INDEX idx_session_timestamp (session_id, timestamp),
    INDEX idx_intent (intent)
);
```

### Knowledge Base Schema
```sql
-- Government schemes and services
CREATE TABLE government_schemes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scheme_code VARCHAR(50) UNIQUE NOT NULL,
    scheme_name JSONB NOT NULL, -- Multi-language names
    description JSONB NOT NULL, -- Multi-language descriptions
    category VARCHAR(50) NOT NULL,
    
    -- Eligibility criteria
    eligibility_rules JSONB NOT NULL,
    income_criteria JSONB,
    age_criteria JSONB,
    geographic_scope JSONB,
    
    -- Application details
    required_documents JSONB NOT NULL,
    application_process JSONB NOT NULL,
    processing_time VARCHAR(50),
    application_fee DECIMAL(10,2) DEFAULT 0,
    
    -- Status and metadata
    is_active BOOLEAN DEFAULT TRUE,
    effective_from DATE,
    effective_to DATE,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_category (category),
    INDEX idx_active (is_active),
    INDEX idx_geographic (geographic_scope) USING GIN
);

-- Document requirements matrix
CREATE TABLE document_requirements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_type VARCHAR(50) NOT NULL,
    document_category VARCHAR(50) NOT NULL,
    document_name JSONB NOT NULL, -- Multi-language
    is_mandatory BOOLEAN DEFAULT TRUE,
    alternatives JSONB DEFAULT '[]',
    specifications JSONB DEFAULT '{}',
    sample_url VARCHAR(255),
    
    INDEX idx_service_type (service_type),
    INDEX idx_mandatory (is_mandatory)
);

-- FAQ and common queries
CREATE TABLE knowledge_articles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title JSONB NOT NULL, -- Multi-language
    content JSONB NOT NULL, -- Multi-language
    category VARCHAR(50) NOT NULL,
    tags TEXT[] DEFAULT '{}',
    search_keywords TEXT[] DEFAULT '{}',
    view_count INTEGER DEFAULT 0,
    is_published BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_category (category),
    INDEX idx_published (is_published),
    INDEX idx_keywords (search_keywords) USING GIN
);
```

### Analytics Schema
```sql
-- User interaction analytics
CREATE TABLE user_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    session_id UUID,
    channel VARCHAR(20) NOT NULL,
    intent VARCHAR(50),
    query_text TEXT,
    response_provided BOOLEAN DEFAULT FALSE,
    satisfaction_score INTEGER, -- 1-5 scale
    resolution_time INTEGER, -- seconds
    language VARCHAR(10),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Partitioning for performance
    PARTITION BY RANGE (timestamp),
    INDEX idx_timestamp (timestamp),
    INDEX idx_channel_intent (channel, intent)
);

-- System performance metrics
CREATE TABLE system_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    metric_name VARCHAR(50) NOT NULL,
    metric_value DECIMAL(10,4) NOT NULL,
    dimensions JSONB DEFAULT '{}',
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PARTITION BY RANGE (timestamp),
    INDEX idx_metric_timestamp (metric_name, timestamp)
);
```

## API Design

### RESTful API Endpoints
```python
# FastAPI application structure
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel
from typing import Optional, List

app = FastAPI(title="AI Civic Helpdesk API", version="1.0.0")

# Request/Response Models
class UserQuery(BaseModel):
    text: str
    language: Optional[str] = "hi"
    channel: str = "web"
    user_id: Optional[str] = None
    session_id: Optional[str] = None

class BotResponse(BaseModel):
    text: str
    language: str
    intent: str
    confidence: float
    suggestions: List[str] = []
    requires_followup: bool = False
    session_id: str

class EligibilityRequest(BaseModel):
    scheme_code: str
    user_profile: dict
    
class EligibilityResponse(BaseModel):
    eligible: bool
    confidence: float
    missing_criteria: List[str] = []
    next_steps: List[str] = []

# Core API Endpoints
@app.post("/api/v1/chat", response_model=BotResponse)
async def process_chat_message(
    query: UserQuery,
    current_user: dict = Depends(get_current_user)
):
    """Process user query and return bot response"""
    # NLP processing
    intent, entities, confidence = await nlp_service.process(
        query.text, query.language
    )
    
    # Generate response
    response = await response_generator.generate(
        intent, entities, current_user, query.language
    )
    
    return BotResponse(
        text=response.text,
        language=query.language,
        intent=intent,
        confidence=confidence,
        suggestions=response.suggestions,
        session_id=response.session_id
    )

@app.post("/api/v1/eligibility", response_model=EligibilityResponse)
async def check_eligibility(request: EligibilityRequest):
    """Check user eligibility for government schemes"""
    eligibility_result = await eligibility_service.check(
        request.scheme_code, request.user_profile
    )
    return eligibility_result

@app.get("/api/v1/schemes")
async def get_schemes(
    category: Optional[str] = None,
    language: str = "hi",
    limit: int = 20
):
    """Get list of government schemes"""
    schemes = await scheme_service.get_schemes(category, language, limit)
    return {"schemes": schemes}

@app.get("/api/v1/documents/{service_type}")
async def get_document_requirements(
    service_type: str,
    language: str = "hi"
):
    """Get required documents for a service"""
    documents = await document_service.get_requirements(
        service_type, language
    )
    return {"documents": documents}

# WhatsApp webhook endpoint
@app.post("/api/v1/webhooks/whatsapp")
async def whatsapp_webhook(request: dict):
    """Handle incoming WhatsApp messages"""
    await whatsapp_handler.process_message(request)
    return {"status": "ok"}

# IVR callback endpoint
@app.post("/api/v1/webhooks/ivr")
async def ivr_webhook(request: dict):
    """Handle IVR system callbacks"""
    await ivr_handler.process_call(request)
    return {"status": "ok"}
```

### WebSocket API for Real-time Communication
```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await websocket.accept()
    
    try:
        while True:
            # Receive message from client
            data = await websocket.receive_text()
            
            # Process through NLP pipeline
            response = await process_user_message(data, user_id)
            
            # Send response back
            await websocket.send_text(response.json())
            
    except WebSocketDisconnect:
        await connection_manager.disconnect(websocket, user_id)
```

## Component Breakdown

### 1. Natural Language Processing Engine
```python
class NLPEngine:
    def __init__(self):
        # Load pre-trained models for Indian languages
        self.models = {
            'hi': spacy.load('hi_core_news_sm'),
            'en': spacy.load('en_core_web_sm'),
            'ta': spacy.load('ta_core_news_sm'),
            'te': spacy.load('te_core_news_sm'),
            'bn': spacy.load('bn_core_news_sm')
        }
        
        # Intent classifier
        self.intent_classifier = pipeline(
            "text-classification",
            model="ai4bharat/indic-bert",
            tokenizer="ai4bharat/indic-bert"
        )
        
        # Entity extractor
        self.entity_extractor = pipeline(
            "ner",
            model="ai4bharat/indic-bert-ner",
            aggregation_strategy="simple"
        )
    
    async def process_query(self, text: str, language: str) -> ProcessingResult:
        # Language detection if not specified
        if not language:
            language = self.detect_language(text)
        
        # Preprocess text
        cleaned_text = self.preprocess_text(text, language)
        
        # Intent classification
        intent_result = self.intent_classifier(cleaned_text)
        intent = intent_result[0]['label']
        confidence = intent_result[0]['score']
        
        # Entity extraction
        entities = self.entity_extractor(cleaned_text)
        
        # Context understanding
        context = await self.understand_context(
            text, intent, entities, language
        )
        
        return ProcessingResult(
            intent=intent,
            entities=entities,
            confidence=confidence,
            language=language,
            context=context
        )
    
    def detect_language(self, text: str) -> str:
        """Detect language using character patterns and common words"""
        # Implementation for Indian language detection
        pass
    
    def preprocess_text(self, text: str, language: str) -> str:
        """Clean and normalize text for processing"""
        # Remove noise, normalize Unicode, handle code-mixing
        pass
```

### 2. Service Logic Components
```python
class ServiceRouter:
    def __init__(self):
        self.services = {
            'aadhaar': AadhaarService(),
            'pan': PANService(),
            'ration_card': RationCardService(),
            'schemes': SchemeService(),
            'documents': DocumentService(),
            'eligibility': EligibilityService()
        }
    
    async def route_request(self, intent: str, entities: dict, 
                          user_context: dict) -> ServiceResponse:
        # Determine which service to use
        service_type = self.determine_service(intent, entities)
        service = self.services.get(service_type)
        
        if not service:
            return await self.handle_general_query(intent, entities)
        
        return await service.handle_request(intent, entities, user_context)

class EligibilityService:
    async def check_scheme_eligibility(self, scheme_code: str, 
                                     user_profile: dict) -> EligibilityResult:
        # Get scheme details
        scheme = await self.get_scheme_details(scheme_code)
        
        # Check each eligibility criterion
        results = {}
        
        # Income check
        if 'income_limit' in scheme.eligibility_rules:
            results['income'] = self.check_income_eligibility(
                user_profile.get('income'), 
                scheme.eligibility_rules['income_limit']
            )
        
        # Age check
        if 'age_range' in scheme.eligibility_rules:
            results['age'] = self.check_age_eligibility(
                user_profile.get('age'),
                scheme.eligibility_rules['age_range']
            )
        
        # Geographic check
        if 'geographic_scope' in scheme.eligibility_rules:
            results['location'] = self.check_location_eligibility(
                user_profile.get('location'),
                scheme.eligibility_rules['geographic_scope']
            )
        
        # Calculate overall eligibility
        eligible = all(results.values())
        missing_criteria = [k for k, v in results.items() if not v]
        
        return EligibilityResult(
            eligible=eligible,
            criteria_results=results,
            missing_criteria=missing_criteria,
            confidence=self.calculate_confidence(results)
        )
```

### 3. Multi-Channel Communication Handler
```python
class WhatsAppHandler:
    def __init__(self, api_key: str, webhook_url: str):
        self.client = WhatsAppClient(api_key)
        self.webhook_url = webhook_url
    
    async def handle_incoming_message(self, message_data: dict):
        user_phone = message_data['from']
        message_text = message_data['text']['body']
        message_type = message_data.get('type', 'text')
        
        # Handle different message types
        if message_type == 'text':
            response = await self.process_text_message(
                message_text, user_phone
            )
        elif message_type == 'audio':
            # Convert audio to text
            text = await self.speech_to_text(message_data['audio'])
            response = await self.process_text_message(text, user_phone)
        elif message_type == 'image':
            # Handle document images (OCR)
            response = await self.process_document_image(
                message_data['image'], user_phone
            )
        
        # Send response
        await self.send_message(user_phone, response)
    
    async def send_message(self, phone: str, response: BotResponse):
        if response.requires_followup:
            # Send interactive buttons
            await self.send_interactive_message(phone, response)
        else:
            # Send simple text message
            await self.send_text_message(phone, response.text)

class IVRHandler:
    def __init__(self, twilio_client):
        self.client = twilio_client
        self.call_flows = self.load_call_flows()
    
    async def handle_incoming_call(self, call_data: dict):
        caller_phone = call_data['From']
        
        # Start with language selection
        twiml = self.generate_language_menu()
        return twiml
    
    async def handle_dtmf_input(self, call_sid: str, digits: str):
        call_context = await self.get_call_context(call_sid)
        current_step = call_context.get('current_step', 'language_selection')
        
        if current_step == 'language_selection':
            language = self.map_digit_to_language(digits)
            await self.update_call_context(call_sid, {
                'language': language,
                'current_step': 'main_menu'
            })
            return self.generate_main_menu(language)
        
        elif current_step == 'main_menu':
            service = self.map_digit_to_service(digits)
            return await self.handle_service_selection(call_sid, service)
    
    async def handle_speech_input(self, call_sid: str, speech_text: str):
        call_context = await self.get_call_context(call_sid)
        language = call_context.get('language', 'hi')
        
        # Process speech through NLP
        response = await nlp_engine.process_query(speech_text, language)
        
        # Generate TwiML response
        twiml_response = await self.generate_speech_response(
            response, language
        )
        
        return twiml_response
```

## User Flow Descriptions

### Flow 1: WhatsApp Scheme Inquiry
```
1. User sends message: "मुझे किसान योजना के बारे में बताएं" (Tell me about farmer schemes)

2. System Processing:
   - Detects language: Hindi
   - Classifies intent: scheme_inquiry
   - Extracts entities: {category: "farmer", type: "schemes"}

3. Bot Response:
   - Lists relevant farmer schemes (PM-KISAN, Kisan Credit Card, etc.)
   - Provides brief description in Hindi
   - Offers interactive buttons: "Eligibility Check", "Application Process", "Documents Required"

4. User selects "Eligibility Check"

5. System asks qualifying questions:
   - "आपके पास कितनी जमीन है?" (How much land do you have?)
   - "आपकी सालाना आय कितनी है?" (What is your annual income?)
   - "आप किस राज्य से हैं?" (Which state are you from?)

6. Based on answers, system provides:
   - Eligible schemes list
   - Expected benefit amounts
   - Next steps for application

7. System offers to guide through application process or save information for later
```

### Flow 2: IVR Voice Interaction
```
1. User calls IVR number

2. System greets: "नमस्कार, सरकारी सेवा हेल्पलाइन में आपका स्वागत है"
   (Welcome to Government Service Helpline)

3. Language selection menu:
   - "हिंदी के लिए 1 दबाएं" (Press 1 for Hindi)
   - "For English, press 2"
   - "தமிழுக்கு 3 ஐ அழுத்தவும்" (Press 3 for Tamil)

4. Main menu (in selected language):
   - "आधार कार्ड के लिए 1" (Press 1 for Aadhaar)
   - "पैन कार्ड के लिए 2" (Press 2 for PAN)
   - "राशन कार्ड के लिए 3" (Press 3 for Ration Card)
   - "सरकारी योजनाओं के लिए 4" (Press 4 for Government Schemes)
   - "अपनी समस्या बताने के लिए 0" (Press 0 to speak your query)

5. If user presses 0 (voice input):
   - System prompts: "अपनी समस्या बताएं" (Tell us your problem)
   - Converts speech to text
   - Processes through NLP engine
   - Provides spoken response with options to repeat or get SMS with details

6. For specific service selection:
   - Provides sub-menu with common queries
   - Offers callback option for complex issues
   - Sends SMS with relevant information and links
```

### Flow 3: Document Checklist Generation
```
1. User query: "PAN card बनाने के लिए क्या documents चाहिए?" 
   (What documents are needed to make PAN card?)

2. System Processing:
   - Intent: document_inquiry
   - Service: pan_card
   - Request_type: new_application

3. System asks clarifying questions:
   - "क्या आप भारतीय नागरिक हैं?" (Are you an Indian citizen?)
   - "आपकी उम्र 18 साल से ज्यादा है?" (Are you above 18 years?)

4. Based on responses, generates personalized checklist:
   - Mandatory documents with alternatives
   - Format specifications (size, type)
   - Sample document images
   - Common mistakes to avoid

5. Offers additional help:
   - Application form download link
   - Nearest PAN center locations
   - Estimated processing time
   - Fee information

6. Saves checklist to user profile for future reference
```

## Integration Architecture

### WhatsApp Business API Integration
```python
class WhatsAppIntegration:
    def __init__(self):
        self.api_base = "https://graph.facebook.com/v17.0"
        self.phone_number_id = os.getenv("WHATSAPP_PHONE_NUMBER_ID")
        self.access_token = os.getenv("WHATSAPP_ACCESS_TOKEN")
    
    async def send_text_message(self, to: str, text: str):
        url = f"{self.api_base}/{self.phone_number_id}/messages"
        
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "text",
            "text": {"body": text}
        }
        
        headers = {
            "Authorization": f"Bearer {self.access_token}",
            "Content-Type": "application/json"
        }
        
        async with httpx.AsyncClient() as client:
            response = await client.post(url, json=payload, headers=headers)
            return response.json()
    
    async def send_interactive_message(self, to: str, message: InteractiveMessage):
        """Send message with buttons or list options"""
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "interactive",
            "interactive": {
                "type": "button",
                "body": {"text": message.body},
                "action": {
                    "buttons": [
                        {
                            "type": "reply",
                            "reply": {
                                "id": btn.id,
                                "title": btn.title
                            }
                        } for btn in message.buttons
                    ]
                }
            }
        }
        
        # Send request
        return await self.send_message(payload)
    
    async def handle_webhook(self, request_data: dict):
        """Process incoming WhatsApp messages"""
        for entry in request_data.get("entry", []):
            for change in entry.get("changes", []):
                if change.get("field") == "messages":
                    await self.process_message_change(change["value"])
```

### IVR System Integration (Twilio)
```python
class IVRIntegration:
    def __init__(self):
        self.client = Client(
            os.getenv("TWILIO_ACCOUNT_SID"),
            os.getenv("TWILIO_AUTH_TOKEN")
        )
        self.tts_voice = "Polly.Aditi"  # Hindi voice
    
    def generate_twiml_response(self, text: str, language: str = "hi"):
        """Generate TwiML for text-to-speech"""
        response = VoiceResponse()
        
        # Set appropriate voice based on language
        voice_map = {
            "hi": "Polly.Aditi",
            "en": "Polly.Joanna",
            "ta": "Polly.Aditi",  # Fallback to Hindi voice
            "te": "Polly.Aditi",
            "bn": "Polly.Aditi"
        }
        
        voice = voice_map.get(language, "Polly.Aditi")
        
        response.say(
            text,
            voice=voice,
            language=self.get_twiml_language_code(language)
        )
        
        return str(response)
    
    def create_menu_twiml(self, menu_items: List[MenuItem], language: str):
        """Create interactive menu with DTMF options"""
        response = VoiceResponse()
        
        # Create gather for DTMF input
        gather = Gather(
            num_digits=1,
            action="/api/v1/webhooks/ivr/menu",
            method="POST",
            timeout=10
        )
        
        # Add menu text
        menu_text = self.build_menu_text(menu_items, language)
        gather.say(menu_text, voice=self.get_voice_for_language(language))
        
        response.append(gather)
        
        # Fallback if no input
        response.say(
            self.get_text("no_input_received", language),
            voice=self.get_voice_for_language(language)
        )
        response.redirect("/api/v1/webhooks/ivr/menu")
        
        return str(response)
    
    async def handle_speech_input(self, call_sid: str, speech_result: str):
        """Process speech input through NLP"""
        call_context = await self.get_call_context(call_sid)
        language = call_context.get("language", "hi")
        
        # Process through NLP engine
        nlp_result = await nlp_engine.process_query(speech_result, language)
        
        # Generate appropriate response
        bot_response = await response_generator.generate_response(
            nlp_result, call_context
        )
        
        # Convert to TwiML
        twiml = self.generate_twiml_response(bot_response.text, language)
        
        return twiml
```

### Government API Integration
```python
class GovernmentAPIIntegration:
    def __init__(self):
        self.apis = {
            "digilocker": DigiLockerAPI(),
            "aadhaar": AadhaarAPI(),
            "pan": PANPortalAPI(),
            "ration": RationCardAPI(),
            "schemes": SchemePortalAPI()
        }
    
    async def verify_aadhaar(self, aadhaar_number: str, otp: str):
        """Verify Aadhaar through UIDAI API"""
        try:
            result = await self.apis["aadhaar"].verify_otp(
                aadhaar_number, otp
            )
            return {
                "verified": result.status == "success",
                "name": result.name,
                "address": result.address
            }
        except Exception as e:
            logger.error(f"Aadhaar verification failed: {e}")
            return {"verified": False, "error": str(e)}
    
    async def check_pan_status(self, pan_number: str):
        """Check PAN card status"""
        try:
            result = await self.apis["pan"].check_status(pan_number)
            return {
                "valid": result.is_valid,
                "name": result.name,
                "status": result.status
            }
        except Exception as e:
            logger.error(f"PAN status check failed: {e}")
            return {"valid": False, "error": str(e)}
    
    async def get_scheme_details(self, scheme_code: str):
        """Fetch latest scheme information"""
        try:
            scheme_data = await self.apis["schemes"].get_scheme(scheme_code)
            return {
                "name": scheme_data.name,
                "description": scheme_data.description,
                "eligibility": scheme_data.eligibility_criteria,
                "benefits": scheme_data.benefits,
                "application_process": scheme_data.application_process
            }
        except Exception as e:
            logger.error(f"Scheme data fetch failed: {e}")
            return None
```

## Deployment Strategy

### Containerized Deployment
```yaml
# docker-compose.yml for development
version: '3.8'

services:
  # API Gateway
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - api-server

  # Main API Server
  api-server:
    build: .
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/civic_helpdesk
      - REDIS_URL=redis://redis:6379
      - WHATSAPP_ACCESS_TOKEN=${WHATSAPP_ACCESS_TOKEN}
      - TWILIO_ACCOUNT_SID=${TWILIO_ACCOUNT_SID}
    depends_on:
      - postgres
      - redis
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 1G
          cpus: '0.5'

  # NLP Service (separate for scaling)
  nlp-service:
    build: ./nlp-service
    environment:
      - MODEL_PATH=/models
    volumes:
      - ./models:/models
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 2G
          cpus: '1.0'

  # Database
  postgres:
    image: postgres:14
    environment:
      - POSTGRES_DB=civic_helpdesk
      - POSTGRES_USER=civic_user
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  # Cache and Session Store
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data

  # Message Queue
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

volumes:
  postgres_data:
  redis_data:
```

### Kubernetes Production Deployment
```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: civic-helpdesk-api
spec:
  replicas: 5
  selector:
    matchLabels:
      app: civic-helpdesk-api
  template:
    metadata:
      labels:
        app: civic-helpdesk-api
    spec:
      containers:
      - name: api-server
        image: civic-helpdesk/api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: civic-helpdesk-service
spec:
  selector:
    app: civic-helpdesk-api
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: civic-helpdesk-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: civic-helpdesk-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Scalability Considerations

### Horizontal Scaling Strategy
```python
# Auto-scaling configuration
class AutoScalingManager:
    def __init__(self):
        self.metrics_client = CloudWatchClient()
        self.k8s_client = KubernetesClient()
    
    async def monitor_and_scale(self):
        """Monitor system metrics and trigger scaling"""
        while True:
            # Get current metrics
            cpu_usage = await self.get_average_cpu_usage()
            memory_usage = await self.get_average_memory_usage()
            request_rate = await self.get_request_rate()
            response_time = await self.get_average_response_time()
            
            # Scaling decisions
            if cpu_usage > 70 or memory_usage > 80:
                await self.scale_up()
            elif cpu_usage < 30 and memory_usage < 40:
                await self.scale_down()
            
            # Special handling for traffic spikes
            if request_rate > self.get_threshold("high_traffic"):
                await self.emergency_scale_up()
            
            await asyncio.sleep(60)  # Check every minute
    
    async def scale_up(self):
        """Increase number of replicas"""
        current_replicas = await self.k8s_client.get_replica_count(
            "civic-helpdesk-api"
        )
        new_replicas = min(current_replicas + 2, 20)  # Max 20 replicas
        
        await self.k8s_client.scale_deployment(
            "civic-helpdesk-api", new_replicas
        )
        
        logger.info(f"Scaled up from {current_replicas} to {new_replicas}")

# Database scaling strategy
class DatabaseScalingManager:
    def __init__(self):
        self.read_replicas = []
        self.write_master = None
    
    async def setup_read_replicas(self):
        """Set up read replicas for better performance"""
        # Create read replicas in different regions
        regions = ["us-east-1", "us-west-2", "ap-south-1"]
        
        for region in regions:
            replica = await self.create_read_replica(region)
            self.read_replicas.append(replica)
    
    def get_database_connection(self, operation_type: str):
        """Route database operations to appropriate instance"""
        if operation_type == "read":
            # Use read replica with lowest latency
            return self.get_best_read_replica()
        else:
            # Use master for writes
            return self.write_master
```

### Caching Strategy for Performance
```python
class CachingStrategy:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.local_cache = {}
        self.cache_ttl = {
            "scheme_data": 86400,  # 24 hours
            "user_sessions": 3600,  # 1 hour
            "nlp_responses": 1800,  # 30 minutes
            "document_templates": 604800,  # 1 week
        }
    
    async def get_cached_response(self, query_hash: str, user_id: str):
        """Get cached response for similar queries"""
        # Try local cache first (fastest)
        local_key = f"local:{query_hash}"
        if local_key in self.local_cache:
            return self.local_cache[local_key]
        
        # Try Redis cache
        redis_key = f"response:{query_hash}"
        cached_response = await self.redis_client.get(redis_key)
        
        if cached_response:
            # Store in local cache for faster access
            self.local_cache[local_key] = cached_response
            return cached_response
        
        return None
    
    async def cache_response(self, query_hash: str, response: str, 
                           cache_type: str = "nlp_responses"):
        """Cache response with appropriate TTL"""
        ttl = self.cache_ttl.get(cache_type, 1800)
        
        # Store in Redis
        await self.redis_client.setex(
            f"response:{query_hash}", ttl, response
        )
        
        # Store in local cache
        self.local_cache[f"local:{query_hash}"] = response
    
    async def invalidate_cache(self, pattern: str):
        """Invalidate cache entries matching pattern"""
        keys = await self.redis_client.keys(pattern)
        if keys:
            await self.redis_client.delete(*keys)
```

This comprehensive design document provides a robust foundation for building the AI Civic Helpdesk system with focus on accessibility, scalability, and the specific needs of Indian citizens seeking government services.
