# Design Document

## Overview

KiroAgri is an AI-powered agricultural advisory platform that provides intelligent, context-aware recommendations to rural farmers through natural language interaction. The system combines natural language processing (NLP), machine learning (ML), and large language models (LLMs) to interpret farmer queries, analyze regional agricultural data, and generate personalized recommendations for crop planning, irrigation, fertilization, and sustainable farming practices.

The platform is designed with a focus on accessibility (low bandwidth, simple interface), security, scalability, and responsible AI principles. The architecture supports future extensibility for image-based disease detection, voice interfaces, and IoT integration.

## Architecture

### High-Level Architecture

The system follows a microservices architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                          │
│  (Web/Mobile Interface - Progressive Web App)                │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                     API Gateway Layer                        │
│  (Authentication, Rate Limiting, Request Routing)            │
└─────────────────┬───────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐  ┌──────▼──────────────────────────────────┐
│  Query Service │  │     Recommendation Engine               │
│  (NLP/Intent)  │  │  ┌──────────┐  ┌──────────────────┐    │
└───────┬────────┘  │  │  Crop    │  │   Irrigation     │    │
        │           │  │ Advisor  │  │   Advisor        │    │
        │           │  └──────────┘  └──────────────────┘    │
        │           │  ┌──────────┐  ┌──────────────────┐    │
        │           │  │Fertilizer│  │  Sustainability  │    │
        │           │  │ Advisor  │  │   Advisor        │    │
        │           │  └──────────┘  └──────────────────┘    │
        │           └─────────┬───────────────────────────────┘
        │                     │
┌───────▼─────────────────────▼───────────────────────────────┐
│              Data & Context Management Layer                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │   Weather    │ │   Regional   │ │   Crop Knowledge │   │
│  │   Service    │ │   Data Svc   │ │      Base        │   │
│  └──────────────┘ └──────────────┘ └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                    Data Storage Layer                        │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │  User Data   │ │  Agricultural│ │   Cache Layer    │   │
│  │  (Postgres)  │ │  Data (NoSQL)│ │   (Redis)        │   │
│  └──────────────┘ └──────────────┘ └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Key Architectural Principles

1. **Modularity**: Each advisor component (crop, irrigation, fertilizer, sustainability) operates independently
2. **Extensibility**: New advisor modules can be added without modifying existing components
3. **Scalability**: Stateless services enable horizontal scaling
4. **Resilience**: Graceful degradation when external data sources are unavailable
5. **Security**: Defense in depth with encryption, authentication, and access controls at multiple layers

## Components and Interfaces

### 1. Client Layer

**Responsibilities:**
- Render user interface optimized for low bandwidth
- Capture and validate user input
- Display recommendations with explanations
- Manage local caching for offline reference

**Key Features:**
- Progressive Web App (PWA) for cross-platform compatibility
- Responsive design for mobile and desktop
- Minimal JavaScript bundle size (<100KB gzipped)
- Service worker for offline capability

**Interface:**
```typescript
interface ClientAPI {
  submitQuery(query: string, sessionId: string): Promise<Response>;
  getRecommendationHistory(userId: string): Promise<Recommendation[]>;
  updateUserProfile(userId: string, profile: UserProfile): Promise<void>;
}
```

### 2. API Gateway

**Responsibilities:**
- Authenticate and authorize requests
- Rate limiting to prevent abuse
- Route requests to appropriate services
- Compress responses for bandwidth optimization

**Technologies:**
- Kong or AWS API Gateway
- JWT-based authentication
- TLS 1.3 encryption

**Interface:**
```typescript
interface GatewayConfig {
  rateLimits: {
    requestsPerMinute: number;
    burstSize: number;
  };
  compression: {
    enabled: boolean;
    minSize: number;
  };
}
```

### 3. Query Service (NLP/Intent Recognition)

**Responsibilities:**
- Parse natural language queries
- Extract intent and entities (crop names, locations, time references)
- Maintain conversation context across session
- Handle ambiguity and request clarification

**AI Components:**
- Fine-tuned BERT or similar transformer model for intent classification
- Named Entity Recognition (NER) for agricultural terms
- LLM (GPT-4 or open-source alternative) for complex query understanding

**Interface:**
```typescript
interface QueryService {
  parseQuery(query: string, context: SessionContext): Promise<ParsedQuery>;
  extractEntities(query: string): Promise<Entity[]>;
  classifyIntent(query: string): Promise<Intent>;
}

interface ParsedQuery {
  intent: Intent;
  entities: Entity[];
  confidence: number;
  needsClarification: boolean;
  clarificationQuestion?: string;
}

enum Intent {
  CROP_RECOMMENDATION = 'crop_recommendation',
  IRRIGATION_ADVICE = 'irrigation_advice',
  FERTILIZER_ADVICE = 'fertilizer_advice',
  SUSTAINABILITY_GUIDANCE = 'sustainability_guidance',
  GENERAL_QUERY = 'general_query'
}

interface Entity {
  type: 'crop' | 'location' | 'season' | 'soil_type' | 'problem';
  value: string;
  confidence: number;
}
```

### 4. Recommendation Engine

**Responsibilities:**
- Coordinate between specialized advisor modules
- Aggregate recommendations from multiple sources
- Generate explanations for recommendations
- Assess confidence levels

**Architecture Pattern:** Strategy pattern with pluggable advisors

**Interface:**
```typescript
interface RecommendationEngine {
  generateRecommendation(
    intent: Intent,
    context: RecommendationContext
  ): Promise<Recommendation>;
}

interface RecommendationContext {
  userId: string;
  region: Region;
  season: Season;
  weatherData: WeatherData;
  soilInfo?: SoilInfo;
  cropHistory?: Crop[];
  userPreferences?: UserPreferences;
}

interface Recommendation {
  id: string;
  type: Intent;
  suggestions: Suggestion[];
  explanation: Explanation;
  confidence: number;
  disclaimer: string;
  sources: DataSource[];
  timestamp: Date;
}

interface Suggestion {
  title: string;
  description: string;
  priority: number;
  suitabilityScore: number;
  details: Record<string, any>;
}

interface Explanation {
  summary: string;
  keyFactors: Factor[];
  reasoning: string;
  uncertainties?: string[];
}

interface Factor {
  name: string;
  value: string;
  impact: 'high' | 'medium' | 'low';
}
```

### 5. Specialized Advisor Modules

#### Crop Advisor

**Responsibilities:**
- Recommend suitable crops based on region, season, and conditions
- Rank crops by suitability score
- Provide growing duration and resource requirements

**ML Model:**
- Gradient boosting model (XGBoost/LightGBM) trained on historical crop yield data
- Features: climate data, soil properties, historical patterns, market trends

**Interface:**
```typescript
interface CropAdvisor {
  recommendCrops(context: RecommendationContext): Promise<CropRecommendation[]>;
  getCropProfile(cropId: string): Promise<CropProfile>;
}

interface CropRecommendation {
  crop: Crop;
  suitabilityScore: number;
  growingDuration: number; // days
  waterRequirement: 'low' | 'medium' | 'high';
  nutrientNeeds: NutrientProfile;
  expectedYield: YieldEstimate;
  risks: Risk[];
}

interface CropProfile {
  id: string;
  name: string;
  scientificName: string;
  varieties: string[];
  optimalTemperature: TemperatureRange;
  optimalRainfall: RainfallRange;
  soilPreferences: SoilType[];
  growthStages: GrowthStage[];
}
```

#### Irrigation Advisor

**Responsibilities:**
- Calculate irrigation requirements based on crop, weather, and soil
- Adjust recommendations based on rainfall forecasts
- Provide scheduling guidance

**ML Model:**
- Time series forecasting for water needs
- Integration with evapotranspiration (ET) calculations

**Interface:**
```typescript
interface IrrigationAdvisor {
  calculateIrrigationNeeds(
    crop: Crop,
    weather: WeatherData,
    soil: SoilInfo
  ): Promise<IrrigationSchedule>;
}

interface IrrigationSchedule {
  frequency: string; // e.g., "every 3 days"
  amount: number; // liters per square meter
  timing: string; // e.g., "early morning"
  adjustments: Adjustment[];
  nextIrrigationDate: Date;
}

interface Adjustment {
  condition: string;
  action: string;
  reason: string;
}
```

#### Fertilizer Advisor

**Responsibilities:**
- Recommend fertilizer types, quantities, and timing
- Consider soil nutrient levels and crop requirements
- Suggest organic alternatives when appropriate

**ML Model:**
- Nutrient optimization model based on soil tests and crop needs
- Rule-based system for organic alternatives

**Interface:**
```typescript
interface FertilizerAdvisor {
  recommendFertilizer(
    crop: Crop,
    soil: SoilInfo,
    growthStage: GrowthStage
  ): Promise<FertilizerRecommendation>;
}

interface FertilizerRecommendation {
  applications: FertilizerApplication[];
  organicAlternatives?: OrganicOption[];
  warnings: string[];
}

interface FertilizerApplication {
  type: string; // e.g., "NPK 10-26-26"
  quantity: number; // kg per hectare
  timing: string;
  method: string; // e.g., "broadcast", "side-dress"
  expectedBenefit: string;
}
```

#### Sustainability Advisor

**Responsibilities:**
- Provide best practices for soil conservation, water management, biodiversity
- Suggest transition paths to sustainable practices
- Explain long-term benefits

**Knowledge Base:**
- Curated database of sustainable farming practices
- Regional adaptation guidelines
- Case studies and success stories

**Interface:**
```typescript
interface SustainabilityAdvisor {
  recommendPractices(
    currentPractices: FarmingPractice[],
    context: RecommendationContext
  ): Promise<SustainabilityRecommendation>;
}

interface SustainabilityRecommendation {
  practices: SustainablePractice[];
  transitionPlan?: TransitionPlan;
  longTermBenefits: Benefit[];
  environmentalImpact: ImpactAssessment;
}

interface SustainablePractice {
  name: string;
  description: string;
  category: 'soil' | 'water' | 'biodiversity' | 'energy';
  difficulty: 'easy' | 'moderate' | 'challenging';
  implementationSteps: string[];
  resources: Resource[];
}
```

### 6. Data Services

#### Weather Service

**Responsibilities:**
- Fetch current and historical weather data
- Provide rainfall forecasts
- Cache data to minimize API calls

**Data Sources:**
- OpenWeatherMap API, NOAA, or regional meteorological services
- Fallback to historical averages when real-time data unavailable

**Interface:**
```typescript
interface WeatherService {
  getCurrentWeather(location: Location): Promise<WeatherData>;
  getForecast(location: Location, days: number): Promise<WeatherForecast>;
  getHistoricalData(location: Location, period: DateRange): Promise<WeatherData[]>;
}

interface WeatherData {
  temperature: number;
  rainfall: number;
  humidity: number;
  windSpeed: number;
  timestamp: Date;
  source: string;
}
```

#### Regional Data Service

**Responsibilities:**
- Manage region-specific agricultural data
- Provide crop calendars and local varieties
- Handle regional climate classifications

**Interface:**
```typescript
interface RegionalDataService {
  getRegionInfo(location: Location): Promise<Region>;
  getCropCalendar(region: Region): Promise<CropCalendar>;
  getSimilarRegions(region: Region): Promise<Region[]>;
}

interface Region {
  id: string;
  name: string;
  climateZone: string;
  soilTypes: SoilType[];
  commonCrops: Crop[];
  rainfallPattern: string;
  dataQuality: 'high' | 'medium' | 'low';
}
```

#### Crop Knowledge Base

**Responsibilities:**
- Store comprehensive crop information
- Provide crop profiles and growing requirements
- Support search and filtering

**Storage:**
- Document database (MongoDB) for flexible schema
- Full-text search capabilities

**Interface:**
```typescript
interface CropKnowledgeBase {
  searchCrops(query: string, filters?: CropFilter): Promise<Crop[]>;
  getCropById(id: string): Promise<CropProfile>;
  getCropsByRegion(region: Region): Promise<Crop[]>;
}
```

## Data Models

### Core Entities

```typescript
// User
interface User {
  id: string;
  phoneNumber?: string;
  email?: string;
  name: string;
  location: Location;
  farmSize?: number; // hectares
  preferredLanguage: string;
  createdAt: Date;
  lastActive: Date;
}

// Location
interface Location {
  latitude: number;
  longitude: number;
  regionId?: string;
  manualDescription?: string; // for users who can't provide GPS
}

// Session
interface Session {
  id: string;
  userId: string;
  startTime: Date;
  endTime?: Date;
  queries: Query[];
  context: SessionContext;
}

interface SessionContext {
  conversationHistory: Message[];
  extractedEntities: Entity[];
  currentTopic?: Intent;
}

// Query
interface Query {
  id: string;
  sessionId: string;
  text: string;
  timestamp: Date;
  parsedIntent: Intent;
  response: Recommendation;
  responseTime: number; // milliseconds
}

// Crop
interface Crop {
  id: string;
  name: string;
  localNames: Record<string, string>; // language -> name
  category: 'cereal' | 'pulse' | 'vegetable' | 'fruit' | 'cash_crop';
  profile: CropProfile;
}

// Soil
interface SoilInfo {
  type: SoilType;
  pH?: number;
  organicMatter?: number; // percentage
  nitrogen?: number; // ppm
  phosphorus?: number; // ppm
  potassium?: number; // ppm
  source: 'user_input' | 'soil_test' | 'regional_default';
  lastUpdated?: Date;
}

enum SoilType {
  CLAY = 'clay',
  SANDY = 'sandy',
  LOAMY = 'loamy',
  SILTY = 'silty',
  PEATY = 'peaty',
  CHALKY = 'chalky'
}

// Season
interface Season {
  name: string;
  startMonth: number;
  endMonth: number;
  characteristics: string[];
}
```

### Data Flow

1. **Query Processing Flow:**
   ```
   User Query → API Gateway → Query Service (NLP) → Intent + Entities
   ```

2. **Recommendation Generation Flow:**
   ```
   Intent + Context → Recommendation Engine → Appropriate Advisor(s)
   → Data Services (Weather, Regional, Crop KB) → ML Models
   → Recommendation + Explanation → Response Compression → User
   ```

3. **Data Update Flow:**
   ```
   External APIs → Weather/Regional Services → Cache Layer → Database
   ```

## AI Pipeline

### 1. Natural Language Understanding (NLU)

**Model:** Fine-tuned BERT or DistilBERT for agricultural domain

**Training Data:**
- Synthetic queries generated from templates
- Real farmer queries (when available)
- Agricultural extension service FAQs
- Translated queries from multiple languages

**Training Process:**
1. Pre-train on general agricultural corpus
2. Fine-tune on intent classification task
3. Fine-tune NER model for agricultural entities
4. Validate on held-out test set with diverse regional queries

**Inference:**
- Batch processing for efficiency
- Caching of common query patterns
- Fallback to rule-based parsing for simple queries

### 2. Crop Recommendation Model

**Model:** Gradient Boosting (XGBoost)

**Features:**
- Climate: temperature, rainfall, humidity (current and historical)
- Soil: type, pH, nutrients
- Temporal: season, month, historical planting patterns
- Regional: crop success rates, market prices
- User: farm size, previous crops, preferences

**Target:** Crop suitability score (0-1)

**Training:**
- Historical yield data from agricultural databases
- Regional crop performance records
- Cross-validation by region to ensure generalization

**Evaluation Metrics:**
- Precision@K for top-K recommendations
- NDCG (Normalized Discounted Cumulative Gain)
- Regional performance analysis

### 3. Irrigation Optimization Model

**Model:** Time series forecasting + rule-based system

**Approach:**
- Calculate crop evapotranspiration (ET) using Penman-Monteith equation
- Forecast water needs using LSTM or Prophet model
- Adjust for rainfall predictions
- Apply crop-specific coefficients

**Inputs:**
- Weather forecast (7-14 days)
- Crop growth stage
- Soil moisture retention characteristics
- Historical irrigation patterns

### 4. Explanation Generation

**Model:** LLM (GPT-4, Claude, or open-source alternative like Llama)

**Prompt Engineering:**
```
You are an agricultural advisor explaining recommendations to a farmer.

Context:
- Region: {region}
- Crop: {crop}
- Season: {season}
- Key factors: {factors}

Recommendation: {recommendation}

Explain why this recommendation was made in simple language, highlighting the 2-3 most important factors. Keep the explanation under 100 words.
```

**Constraints:**
- Maximum token limit for cost control
- Temperature setting for consistent outputs
- Content filtering for inappropriate responses

### 5. Model Monitoring and Updates

**Monitoring:**
- Track model accuracy and user feedback
- Monitor for data drift in weather patterns
- A/B testing for model improvements

**Update Cycle:**
- Quarterly retraining with new data
- Immediate updates for critical issues
- Versioning for rollback capability

