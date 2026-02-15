# Design Document: MedSort AI

## Overview

MedSort AI is a full-stack healthcare assistant application that combines machine learning, OCR technology, and generative AI to help users understand their health status. The system architecture follows a clean separation between frontend (React), backend (FastAPI), ML models, and external services (OpenAI API, Tesseract OCR).

The application processes two primary workflows:
1. **Symptom Analysis Flow**: User describes symptoms → GenAI extracts structured symptoms → ML model predicts diseases → GenAI generates explanations
2. **Report Analysis Flow**: User uploads medical report → OCR extracts text → Parser identifies lab parameters → System compares against reference ranges → GenAI generates guidance

Key design principles:
- **Safety First**: All outputs include medical disclaimers and encourage professional consultation
- **Modularity**: Clear separation between symptom analysis, report processing, and AI explanation layers
- **Extensibility**: Easy to add new lab parameters, diseases, or ML models
- **Security**: Input validation, file scanning, and secure storage throughout

## Architecture

### System Architecture

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[React Dashboard]
        UI_Symptom[Symptom Input Component]
        UI_Upload[Report Upload Component]
        UI_Results[Results Display Component]
    end
    
    subgraph "Backend Layer - FastAPI"
        API[API Gateway]
        SymptomRoute[/predict-symptoms]
        ReportRoute[/upload-report]
        HealthRoute[/health]
        GetReportRoute[/report/:id]
    end
    
    subgraph "Service Layer"
        SymptomService[Symptom Analysis Service]
        ReportService[Report Processing Service]
        GenAIService[GenAI Service]
        ValidationService[Validation Service]
    end
    
    subgraph "ML Layer"
        SymptomExtractor[Symptom Extractor - LLM]
        DiseaseClassifier[Disease Classifier - RF/XGBoost]
        FeatureEncoder[Feature Vector Encoder]
    end
    
    subgraph "OCR Layer"
        OCREngine[Tesseract OCR]
        LabParser[Lab Parameter Parser]
        RangeComparator[Reference Range Comparator]
    end
    
    subgraph "Data Layer"
        DB[(SQLite Database)]
        FileStore[Secure File Storage]
        ModelStore[Model Storage - .pkl]
    end
    
    subgraph "External Services"
        OpenAI[OpenAI API]
    end
    
    UI --> API
    API --> SymptomRoute
    API --> ReportRoute
    API --> HealthRoute
    API --> GetReportRoute
    
    SymptomRoute --> SymptomService
    ReportRoute --> ReportService
    
    SymptomService --> SymptomExtractor
    SymptomService --> DiseaseClassifier
    SymptomService --> GenAIService
    
    ReportService --> OCREngine
    ReportService --> LabParser
    ReportService --> RangeComparator
    ReportService --> GenAIService
    
    SymptomExtractor --> OpenAI
    GenAIService --> OpenAI
    
    DiseaseClassifier --> FeatureEncoder
    DiseaseClassifier --> ModelStore
    
    SymptomService --> DB
    ReportService --> DB
    ReportService --> FileStore
    
    ValidationService --> SymptomRoute
    ValidationService --> ReportRoute
```

### Technology Stack

**Frontend:**
- React 18+ with functional components and hooks
- Tailwind CSS for styling
- Axios for API communication
- React Router for navigation

**Backend:**
- Python 3.9+
- FastAPI framework with async support
- Pydantic for data validation
- Uvicorn as ASGI server

**Machine Learning:**
- Scikit-learn (Random Forest or XGBoost classifier)
- Pandas for data manipulation
- NumPy for numerical operations
- Joblib for model serialization

**GenAI:**
- OpenAI API (GPT-4 or GPT-3.5-turbo)
- Structured prompts with safety guidelines
- Temperature tuning for consistent medical responses

**OCR:**
- Tesseract OCR 4.0+
- Pytesseract Python wrapper
- PDF2Image for PDF processing
- Pillow for image preprocessing

**Database:**
- SQLite3 for development and demo
- SQLAlchemy ORM for database operations
- Alembic for migrations (optional)

**Security:**
- Python-multipart for file uploads
- File type validation using python-magic
- Input sanitization using bleach
- Rate limiting using slowapi

## Components and Interfaces

### Frontend Components

#### 1. Dashboard Component
```typescript
interface DashboardProps {
  user?: UserSession;
}

interface DashboardState {
  activeTab: 'symptoms' | 'reports';
  loading: boolean;
}

// Main container component that orchestrates symptom and report sections
```

#### 2. Symptom Input Component
```typescript
interface SymptomInputProps {
  onSubmit: (symptoms: string) => Promise<void>;
}

interface SymptomInputState {
  symptomText: string;
  isSubmitting: boolean;
  error: string | null;
}

// Handles symptom text input with validation (min 10 characters)
// Displays loading state during API call
```

#### 3. Report Upload Component
```typescript
interface ReportUploadProps {
  onUpload: (file: File) => Promise<void>;
  acceptedFormats: string[];
  maxSizeMB: number;
}

interface ReportUploadState {
  selectedFile: File | null;
  isUploading: boolean;
  uploadProgress: number;
  error: string | null;
}

// Handles file selection and upload with client-side validation
// Shows upload progress and error messages
```

#### 4. Results Display Component
```typescript
interface DiseasePrediction {
  disease: string;
  probability: number;
  riskLevel: 'Low' | 'Moderate' | 'High';
  explanation: string;
  causes: string[];
  dietSuggestions: string[];
  lifestyleRecommendations: string[];
  whenToConsult: string;
}

interface LabResult {
  parameter: string;
  value: number;
  unit: string;
  normalRange: string;
  status: 'Low' | 'Normal' | 'High';
  explanation?: string;
}

interface ResultsDisplayProps {
  predictions?: DiseasePrediction[];
  labResults?: LabResult[];
  disclaimer: string;
}

// Displays disease predictions with risk badges
// Highlights abnormal lab values in red
// Shows medical disclaimer prominently
```

### Backend API Interfaces

#### 1. Symptom Prediction Endpoint
```python
# POST /predict-symptoms
class SymptomRequest(BaseModel):
    symptom_text: str = Field(..., min_length=10, max_length=2000)
    user_id: Optional[str] = None
    age: Optional[int] = Field(None, ge=0, le=120)
    gender: Optional[str] = Field(None, pattern="^(male|female|other)$")

class DiseasePrediction(BaseModel):
    disease: str
    probability: float
    risk_level: str
    explanation: str
    causes: List[str]
    diet_suggestions: List[str]
    lifestyle_recommendations: List[str]
    when_to_consult: str

class SymptomResponse(BaseModel):
    predictions: List[DiseasePrediction]
    extracted_symptoms: List[str]
    disclaimer: str
    timestamp: datetime
    analysis_id: str
```

#### 2. Report Upload Endpoint
```python
# POST /upload-report
class ReportUploadRequest:
    file: UploadFile
    user_id: Optional[str] = None
    age: Optional[int] = None
    gender: Optional[str] = None

class LabParameter(BaseModel):
    parameter: str
    value: float
    unit: str
    normal_range: str
    status: str  # "Low", "Normal", "High"
    explanation: Optional[str] = None

class ReportResponse(BaseModel):
    report_id: str
    lab_parameters: List[LabParameter]
    abnormal_count: int
    disclaimer: str
    timestamp: datetime
    file_name: str
```

#### 3. Get Report Endpoint
```python
# GET /report/{report_id}
class ReportDetailResponse(BaseModel):
    report_id: str
    lab_parameters: List[LabParameter]
    abnormal_count: int
    disclaimer: str
    timestamp: datetime
    file_name: str
    user_id: Optional[str]
```

#### 4. Health Check Endpoint
```python
# GET /health
class HealthResponse(BaseModel):
    status: str  # "healthy", "degraded", "unhealthy"
    ml_model_loaded: bool
    database_connected: bool
    genai_available: bool
    timestamp: datetime
```

### Service Layer Interfaces

#### 1. Symptom Analysis Service
```python
class SymptomAnalysisService:
    def __init__(self, ml_model, genai_service, db_session):
        self.ml_model = ml_model
        self.genai_service = genai_service
        self.db = db_session
    
    async def analyze_symptoms(
        self, 
        symptom_text: str, 
        user_id: Optional[str] = None,
        age: Optional[int] = None,
        gender: Optional[str] = None
    ) -> SymptomResponse:
        """
        1. Extract structured symptoms using GenAI
        2. Convert to feature vector
        3. Predict diseases using ML model
        4. Generate explanations using GenAI
        5. Store results in database
        6. Return formatted response
        """
        pass
    
    def _extract_symptoms(self, symptom_text: str) -> List[str]:
        """Use GenAI to extract structured symptom list"""
        pass
    
    def _create_feature_vector(self, symptoms: List[str]) -> np.ndarray:
        """Convert symptom list to ML model input format"""
        pass
    
    def _classify_risk_level(self, probability: float) -> str:
        """Map probability to Low/Moderate/High"""
        pass
```

#### 2. Report Processing Service
```python
class ReportProcessingService:
    def __init__(self, ocr_engine, genai_service, db_session):
        self.ocr = ocr_engine
        self.genai_service = genai_service
        self.db = db_session
        self.reference_ranges = self._load_reference_ranges()
    
    async def process_report(
        self,
        file: UploadFile,
        user_id: Optional[str] = None,
        age: Optional[int] = None,
        gender: Optional[str] = None
    ) -> ReportResponse:
        """
        1. Validate and save file
        2. Extract text using OCR
        3. Parse lab parameters
        4. Compare against reference ranges
        5. Generate explanations for abnormal values
        6. Store results in database
        7. Return formatted response
        """
        pass
    
    def _extract_text_from_file(self, file_path: str) -> str:
        """Use OCR to extract text from PDF or image"""
        pass
    
    def _parse_lab_parameters(self, text: str) -> List[Dict]:
        """Extract parameter names, values, and units"""
        pass
    
    def _get_reference_range(
        self, 
        parameter: str, 
        age: Optional[int], 
        gender: Optional[str]
    ) -> Tuple[float, float]:
        """Get age and gender-specific reference range"""
        pass
    
    def _compare_to_range(
        self, 
        value: float, 
        range_min: float, 
        range_max: float
    ) -> str:
        """Return "Low", "Normal", or "High" """
        pass
```

#### 3. GenAI Service
```python
class GenAIService:
    def __init__(self, api_key: str, model: str = "gpt-3.5-turbo"):
        self.client = OpenAI(api_key=api_key)
        self.model = model
    
    async def extract_symptoms(self, symptom_text: str) -> List[str]:
        """
        Extract structured symptom list from natural language.
        Uses prompt: "Extract medical symptoms from the following text..."
        """
        pass
    
    async def generate_disease_explanation(
        self, 
        disease: str, 
        probability: float,
        symptoms: List[str]
    ) -> Dict[str, Any]:
        """
        Generate explanation, causes, diet suggestions, lifestyle recommendations.
        Uses safety-focused prompt with medical disclaimer requirements.
        Returns: {
            "explanation": str,
            "causes": List[str],
            "diet_suggestions": List[str],
            "lifestyle_recommendations": List[str],
            "when_to_consult": str
        }
        """
        pass
    
    async def generate_lab_explanation(
        self,
        parameter: str,
        value: float,
        status: str,
        normal_range: str
    ) -> str:
        """
        Generate explanation for abnormal lab value.
        Uses prompt emphasizing safety and professional consultation.
        """
        pass
    
    def _create_safe_prompt(self, base_prompt: str) -> str:
        """
        Wrap prompts with safety instructions:
        - Avoid definitive diagnoses
        - Encourage professional consultation
        - Use simple language
        - Include appropriate caveats
        """
        pass
```

#### 4. Validation Service
```python
class ValidationService:
    ALLOWED_EXTENSIONS = {'.pdf', '.png', '.jpg', '.jpeg'}
    MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB
    
    def validate_symptom_text(self, text: str) -> Tuple[bool, Optional[str]]:
        """
        Validate symptom input:
        - Not empty or whitespace only
        - Minimum 10 characters
        - Maximum 2000 characters
        - No malicious content
        Returns: (is_valid, error_message)
        """
        pass
    
    def validate_file_upload(self, file: UploadFile) -> Tuple[bool, Optional[str]]:
        """
        Validate uploaded file:
        - File extension in allowed list
        - File size under limit
        - File type matches extension (using python-magic)
        - Not executable or script
        Returns: (is_valid, error_message)
        """
        pass
    
    def sanitize_input(self, text: str) -> str:
        """Remove potentially harmful content from text input"""
        pass
```

## Data Models

### Database Schema

```python
from sqlalchemy import Column, Integer, String, Float, DateTime, Text, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class SymptomAnalysis(Base):
    __tablename__ = 'symptom_analyses'
    
    id = Column(String, primary_key=True)  # UUID
    user_id = Column(String, nullable=True, index=True)
    symptom_text = Column(Text, nullable=False)
    extracted_symptoms = Column(Text, nullable=False)  # JSON string
    predictions = Column(Text, nullable=False)  # JSON string
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)
    age = Column(Integer, nullable=True)
    gender = Column(String, nullable=True)

class ReportAnalysis(Base):
    __tablename__ = 'report_analyses'
    
    id = Column(String, primary_key=True)  # UUID
    user_id = Column(String, nullable=True, index=True)
    file_name = Column(String, nullable=False)
    file_path = Column(String, nullable=False)
    lab_parameters = Column(Text, nullable=False)  # JSON string
    abnormal_count = Column(Integer, nullable=False)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)
    age = Column(Integer, nullable=True)
    gender = Column(String, nullable=True)
    ocr_text = Column(Text, nullable=True)  # Store extracted text for debugging

class ReferenceRange(Base):
    __tablename__ = 'reference_ranges'
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    parameter = Column(String, nullable=False, index=True)
    age_min = Column(Integer, nullable=True)
    age_max = Column(Integer, nullable=True)
    gender = Column(String, nullable=True)  # "male", "female", "all"
    range_min = Column(Float, nullable=False)
    range_max = Column(Float, nullable=False)
    unit = Column(String, nullable=False)
```

### ML Model Data Structures

```python
# Feature vector structure for disease prediction
class SymptomFeatureVector:
    """
    Binary vector representing presence/absence of symptoms.
    Length: Number of unique symptoms in training data (e.g., 132 symptoms)
    Example: [0, 1, 0, 1, 1, 0, ...] where 1 = symptom present
    """
    def __init__(self, symptom_list: List[str], all_symptoms: List[str]):
        self.vector = self._encode(symptom_list, all_symptoms)
    
    def _encode(self, symptoms: List[str], all_symptoms: List[str]) -> np.ndarray:
        """Create binary vector from symptom list"""
        vector = np.zeros(len(all_symptoms))
        for i, symptom in enumerate(all_symptoms):
            if symptom in symptoms:
                vector[i] = 1
        return vector

# Disease prediction output
class ModelPrediction:
    disease: str
    probability: float
    
# Training data format
class TrainingDataset:
    """
    CSV format:
    symptom_1, symptom_2, ..., symptom_n, disease
    1, 0, ..., 1, "Diabetes"
    0, 1, ..., 0, "Hypertension"
    """
    pass
```

### Reference Range Data Structure

```python
# Reference ranges stored as JSON or database table
REFERENCE_RANGES = {
    "Hemoglobin": {
        "male": {"adult": (13.5, 17.5), "unit": "g/dL"},
        "female": {"adult": (12.0, 15.5), "unit": "g/dL"}
    },
    "WBC": {
        "all": {"adult": (4.0, 11.0), "unit": "10^3/μL"}
    },
    "RBC": {
        "male": {"adult": (4.5, 5.9), "unit": "10^6/μL"},
        "female": {"adult": (4.0, 5.2), "unit": "10^6/μL"}
    },
    "Platelets": {
        "all": {"adult": (150, 400), "unit": "10^3/μL"}
    },
    "Glucose_Fasting": {
        "all": {"adult": (70, 100), "unit": "mg/dL"}
    },
    "Cholesterol_Total": {
        "all": {"adult": (0, 200), "unit": "mg/dL"}
    },
    "HDL": {
        "male": {"adult": (40, 999), "unit": "mg/dL"},
        "female": {"adult": (50, 999), "unit": "mg/dL"}
    },
    "LDL": {
        "all": {"adult": (0, 100), "unit": "mg/dL"}
    },
    "Triglycerides": {
        "all": {"adult": (0, 150), "unit": "mg/dL"}
    },
    "Creatinine": {
        "male": {"adult": (0.7, 1.3), "unit": "mg/dL"},
        "female": {"adult": (0.6, 1.1), "unit": "mg/dL"}
    }
}
```

### Configuration Data Model

```python
# .env configuration structure
class Config:
    # API Configuration
    API_HOST: str = "0.0.0.0"
    API_PORT: int = 8000
    API_WORKERS: int = 4
    
    # Database
    DATABASE_URL: str = "sqlite:///./medsort_ai.db"
    
    # OpenAI
    OPENAI_API_KEY: str  # Required
    OPENAI_MODEL: str = "gpt-3.5-turbo"
    OPENAI_TEMPERATURE: float = 0.3  # Lower for consistent medical responses
    
    # File Upload
    UPLOAD_DIR: str = "./uploads"
    MAX_FILE_SIZE: int = 10 * 1024 * 1024  # 10MB
    ALLOWED_EXTENSIONS: List[str] = [".pdf", ".png", ".jpg", ".jpeg"]
    
    # ML Model
    MODEL_PATH: str = "./models/disease_classifier.pkl"
    SYMPTOM_LIST_PATH: str = "./models/symptom_list.json"
    
    # Security
    RATE_LIMIT_PER_MINUTE: int = 10
    ENABLE_CORS: bool = True
    CORS_ORIGINS: List[str] = ["http://localhost:3000"]
    
    # OCR
    TESSERACT_PATH: str = "/usr/bin/tesseract"  # System-dependent
    OCR_LANGUAGE: str = "eng"
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property 1: Symptom Text Length Validation
*For any* input string, the system should accept it as valid symptom text if and only if it contains at least 10 non-whitespace characters.
**Validates: Requirements 1.1, 1.4**

### Property 2: Feature Vector Shape Consistency
*For any* list of extracted symptoms, the generated feature vector should have a fixed length equal to the total number of known symptoms in the model's vocabulary.
**Validates: Requirements 1.3**

### Property 3: Disease Prediction Output Format
*For any* valid feature vector input, the prediction output should contain exactly 3 disease predictions, each with a disease name, probability score between 0 and 1, and a risk level from the set {Low, Moderate, High}.
**Validates: Requirements 2.2, 2.3**

### Property 4: Risk Level Classification Correctness
*For any* probability score p, the risk level should be "Low" if p < 0.3, "Moderate" if 0.3 ≤ p ≤ 0.7, and "High" if p > 0.7.
**Validates: Requirements 2.5, 2.6, 2.7**

### Property 5: File Upload Validation
*For any* uploaded file, it should be accepted if and only if: (1) its extension is in {.pdf, .png, .jpg, .jpeg}, (2) its size is ≤ 10MB, and (3) its MIME type matches its extension.
**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**

### Property 6: File Storage Uniqueness
*For any* valid uploaded file, the system should store it with a unique identifier such that no two files share the same identifier.
**Validates: Requirements 3.6**

### Property 7: User Association Persistence
*For any* stored file or analysis result, if it was created with a user_id, then retrieving it should return the same user_id.
**Validates: Requirements 3.7, 9.4**

### Property 8: Lab Parameter Extraction Completeness
*For any* parsed lab parameter, the result should include both a parameter name (non-empty string) and a numeric value.
**Validates: Requirements 4.3**

### Property 9: Lab Value Classification Correctness
*For any* lab value v with reference range [min, max], the status should be "Low" if v < min, "Normal" if min ≤ v ≤ max, and "High" if v > max.
**Validates: Requirements 5.3, 5.4, 5.5**

### Property 10: Reference Range Selection Logic
*For any* lab parameter analysis, the system should use gender-specific ranges when gender is provided, age-specific ranges when age is provided, and general adult ranges when neither is provided.
**Validates: Requirements 5.1, 5.2, 5.7**

### Property 11: Lab Analysis Output Completeness
*For any* completed lab analysis, each lab parameter in the result should include: parameter name, user value, normal range, status, and unit.
**Validates: Requirements 5.6**

### Property 12: Disease Explanation Completeness
*For any* disease prediction, the generated explanation should include all of: simple explanation text, list of possible causes, diet suggestions, lifestyle recommendations, and when-to-consult guidance.
**Validates: Requirements 6.1, 6.3, 6.4, 6.5, 6.6**

### Property 13: Abnormal Lab Value Explanation Presence
*For any* lab parameter with status "Low" or "High", the result should include an explanation field with non-empty content.
**Validates: Requirements 6.2**

### Property 14: Medical Safety Language
*For any* generated health guidance text, it should not contain definitive diagnostic phrases (e.g., "you have", "you are diagnosed with") and should include language encouraging professional consultation.
**Validates: Requirements 6.7, 6.8, 7.5**

### Property 15: Medical Disclaimer Presence
*For any* prediction or analysis response, the output should include a disclaimer field containing text that states results are informational only and not medical advice.
**Validates: Requirements 7.1, 7.2, 7.3**

### Property 16: High-Risk Warning Emphasis
*For any* disease prediction with risk level "High", the response should include emphasized language about seeking immediate medical consultation.
**Validates: Requirements 7.4**

### Property 17: API Input Validation
*For any* API endpoint, when receiving a request with invalid input data, the system should reject it with a 4xx status code before processing, rather than attempting to process invalid data.
**Validates: Requirements 8.5**

### Property 18: API Error Response Format
*For any* API endpoint that encounters an error, the response should include an appropriate HTTP status code (4xx for client errors, 5xx for server errors) and a JSON body with an error message.
**Validates: Requirements 8.6**

### Property 19: API Response Format Consistency
*For any* successful API response, the Content-Type header should be "application/json" and the body should be valid JSON.
**Validates: Requirements 8.7**

### Property 20: Analysis Persistence Round-Trip
*For any* completed symptom analysis or report analysis, storing it in the database and then retrieving it by ID should return data equivalent to the original analysis (same predictions, values, and metadata).
**Validates: Requirements 9.1, 9.2, 9.6**

### Property 21: Timestamp Presence
*For any* stored analysis result, the record should include a timestamp field with a valid datetime value.
**Validates: Requirements 9.3**

### Property 22: Model Serialization Round-Trip
*For any* trained ML model, saving it as a .pkl file and then loading it should produce a model that makes equivalent predictions on the same input.
**Validates: Requirements 11.4**

### Property 23: File Type Allowlist Enforcement
*For any* file upload attempt, only files with extensions in the allowlist {.pdf, .png, .jpg, .jpeg} should be accepted for processing.
**Validates: Requirements 12.1**

### Property 24: Input Sanitization
*For any* API request with text input, the system should sanitize the input to remove potentially harmful content before processing.
**Validates: Requirements 12.3**

### Property 25: Error Message Safety
*For any* error response, the message should not contain sensitive information such as file paths, database connection strings, or API keys.
**Validates: Requirements 12.4**

### Property 26: Rate Limiting Enforcement
*For any* API endpoint, when requests from the same source exceed the configured rate limit, subsequent requests should be rejected with a 429 status code.
**Validates: Requirements 12.5**

### Property 27: Secure File Identifier Generation
*For any* two files uploaded at different times, their identifiers should be different and not predictable from each other.
**Validates: Requirements 12.6**

### Property 28: Error Logging Completeness
*For any* error that occurs during request processing, the system should create a log entry containing the error type, message, timestamp, and relevant context.
**Validates: Requirements 13.2**

### Property 29: Validation Error Field Identification
*For any* validation error response, the error message should specify which input field or parameter caused the validation failure.
**Validates: Requirements 13.6**

### Property 30: Symptom Prediction Performance
*For any* symptom prediction request under normal load, the response time should be less than 5 seconds from request receipt to response sent.
**Validates: Requirements 15.1**

### Property 31: Report Processing Performance
*For any* valid medical report file up to 10MB, the processing time from upload to analysis completion should be less than 30 seconds.
**Validates: Requirements 15.2**

## Error Handling

### Error Categories

**1. Validation Errors (4xx)**
- Invalid symptom text (too short, empty)
- Invalid file type or size
- Missing required fields
- Malformed JSON

**2. Processing Errors (5xx)**
- ML model loading failure
- OCR extraction failure
- GenAI API unavailable
- Database connection failure

**3. Business Logic Errors (4xx)**
- No lab parameters found in report
- Unable to extract symptoms from text
- Report analysis not found by ID

### Error Response Format

All errors follow a consistent JSON structure:

```python
class ErrorResponse(BaseModel):
    error: str  # Error type/category
    message: str  # User-friendly message
    detail: Optional[str]  # Additional context (optional)
    field: Optional[str]  # Field that caused error (for validation)
    timestamp: datetime
    request_id: str  # For tracking/debugging
```

### Error Handling Strategy

**Symptom Analysis Errors:**
- Symptom extraction failure → Return error suggesting clearer symptom description
- Model prediction failure → Return 503 with retry suggestion
- GenAI explanation failure → Return predictions without explanations, log error

**Report Processing Errors:**
- File validation failure → Return 400 with specific validation issue
- OCR failure → Return 422 with guidance on image quality
- No parameters found → Return 422 suggesting different report format
- GenAI explanation failure → Return lab results without explanations, log error

**Database Errors:**
- Connection failure → Return 503 with retry suggestion
- Query failure → Log detailed error, return generic 500 to user
- Record not found → Return 404 with clear message

**External Service Errors:**
- OpenAI API timeout → Retry once, then return error
- OpenAI API rate limit → Return 429 with retry-after header
- OpenAI API error → Log error, return 503

### Logging Strategy

**Log Levels:**
- ERROR: System failures, external service errors, unexpected exceptions
- WARNING: Validation failures, missing optional data, degraded performance
- INFO: Successful requests, analysis completions, file uploads
- DEBUG: Detailed processing steps, ML model inputs/outputs

**Log Content:**
- Request ID for tracing
- User ID (if available)
- Endpoint and method
- Input summary (sanitized)
- Processing time
- Error details (for failures)
- No sensitive data (passwords, API keys, PII)

## Testing Strategy

### Dual Testing Approach

MedSort AI requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests** focus on:
- Specific examples of symptom analysis and disease prediction
- Edge cases (empty files, malformed reports, boundary values)
- Error conditions (missing config, API failures, invalid inputs)
- Integration points (database operations, file storage, API endpoints)
- Mock external services (OpenAI API, OCR engine)

**Property-Based Tests** focus on:
- Universal properties that hold for all inputs
- Input validation across random data
- Data transformation correctness (symptom → feature vector)
- Classification logic (risk levels, lab value status)
- Round-trip properties (serialization, database storage)
- API contract compliance

### Property-Based Testing Configuration

**Framework:** Hypothesis (Python)
- Minimum 100 iterations per property test
- Custom generators for medical data (symptoms, lab values, reports)
- Shrinking enabled for minimal failing examples

**Test Tagging:**
Each property test must reference its design document property:
```python
@given(symptom_text=text(min_size=1, max_size=2000))
def test_symptom_length_validation(symptom_text):
    """
    Feature: medsort-ai, Property 1: Symptom Text Length Validation
    For any input string, the system should accept it as valid symptom text
    if and only if it contains at least 10 non-whitespace characters.
    """
    # Test implementation
```

### Unit Testing Strategy

**Backend Tests (pytest):**
- API endpoint tests with various inputs
- Service layer tests with mocked dependencies
- Database model tests
- Validation logic tests
- Error handling tests
- Configuration loading tests

**Frontend Tests (Jest + React Testing Library):**
- Component rendering tests
- User interaction tests (form submission, file upload)
- API integration tests (mocked)
- Error display tests
- Responsive layout tests

**Integration Tests:**
- End-to-end symptom analysis flow
- End-to-end report processing flow
- Database persistence and retrieval
- File upload and storage
- Error propagation through layers

### ML Model Testing

**Model Evaluation:**
- Train/test split (80/20)
- Cross-validation (5-fold)
- Metrics: Accuracy, Precision, Recall, F1-score
- Confusion matrix analysis
- Per-disease performance metrics

**Model Validation:**
- Test on held-out validation set
- Compare against baseline (random, most-frequent)
- Check for overfitting (train vs. test performance)
- Verify prediction probabilities are calibrated

### Test Data

**Symptom Analysis Test Data:**
- Synthetic symptom descriptions (various lengths, formats)
- Known disease-symptom mappings
- Edge cases (single symptom, many symptoms, ambiguous symptoms)

**Report Processing Test Data:**
- Sample lab reports (PDF and images)
- Reports with various formats and layouts
- Reports with missing or incomplete data
- Non-medical documents (for negative testing)

**Reference Range Test Data:**
- Complete reference range database
- Age and gender variations
- Edge cases (boundary values, missing demographics)

### Test Coverage Goals

- Backend code coverage: >80%
- Frontend code coverage: >70%
- All API endpoints tested
- All correctness properties tested
- All error paths tested
- All validation rules tested

### Continuous Testing

**Pre-commit:**
- Linting (pylint, eslint)
- Type checking (mypy, TypeScript)
- Unit tests (fast subset)

**CI Pipeline:**
- Full unit test suite
- Property-based tests
- Integration tests
- Code coverage reporting
- Security scanning (bandit, npm audit)

**Performance Testing:**
- Load testing (10+ concurrent users)
- Response time monitoring
- Database query performance
- File processing benchmarks
- Memory usage profiling

### Safety Testing

**Medical Content Review:**
- Manual review of GenAI prompt templates
- Sample output review for safety
- Disclaimer presence verification
- Negative testing (ensure no definitive diagnoses)

**Security Testing:**
- File upload security (malicious files)
- Input injection testing (SQL, XSS)
- API authentication/authorization
- Rate limiting verification
- Sensitive data exposure checks

## Implementation Notes

### GenAI Prompt Engineering

**Symptom Extraction Prompt Template:**
```
You are a medical assistant helping to extract structured symptom information.

Task: Extract a list of medical symptoms from the following patient description.

Rules:
- Return only symptom names, one per line
- Use standard medical terminology
- Do not include patient demographics or history
- Do not make diagnoses or interpretations

Patient description:
{symptom_text}

Extracted symptoms:
```

**Disease Explanation Prompt Template:**
```
You are a medical information assistant providing educational content.

Task: Provide a simple explanation for the following predicted disease.

Disease: {disease}
Probability: {probability}
Patient symptoms: {symptoms}

IMPORTANT SAFETY RULES:
- Do NOT make definitive diagnoses
- Do NOT prescribe treatments or medications
- ALWAYS encourage consulting healthcare professionals
- Use simple, non-technical language
- Be supportive and non-alarming

Provide:
1. Simple explanation (2-3 sentences)
2. Possible causes (3-5 bullet points)
3. Diet suggestions (3-5 bullet points)
4. Lifestyle recommendations (3-5 bullet points)
5. When to consult a doctor (specific guidance)

Format as JSON:
{
  "explanation": "...",
  "causes": ["...", "..."],
  "diet_suggestions": ["...", "..."],
  "lifestyle_recommendations": ["...", "..."],
  "when_to_consult": "..."
}
```

**Lab Value Explanation Prompt Template:**
```
You are a medical information assistant explaining lab test results.

Task: Explain the following abnormal lab value in simple terms.

Parameter: {parameter}
Patient value: {value} {unit}
Normal range: {normal_range}
Status: {status}

IMPORTANT SAFETY RULES:
- Do NOT make definitive diagnoses
- Do NOT prescribe treatments
- ALWAYS encourage consulting healthcare professionals
- Use simple language
- Be factual and supportive

Provide a brief explanation (2-3 sentences) of what this abnormal value might indicate and what the patient should do.

Explanation:
```

### OCR Preprocessing

To improve OCR accuracy:
1. Convert PDF to images (300 DPI)
2. Grayscale conversion
3. Noise reduction (Gaussian blur)
4. Contrast enhancement
5. Binarization (Otsu's method)
6. Deskewing (if needed)

### Lab Parameter Parsing Strategy

**Pattern Matching:**
- Use regex patterns for common lab report formats
- Match parameter names (case-insensitive)
- Extract numeric values with units
- Handle various separators (colon, dash, equals)

**Common Patterns:**
```python
PATTERNS = {
    "hemoglobin": r"(?:hemoglobin|hb|hgb)[\s:=-]*(\d+\.?\d*)\s*(g/dl|g/l)?",
    "wbc": r"(?:wbc|white blood cell)[\s:=-]*(\d+\.?\d*)\s*(10\^3/μl|k/μl)?",
    "rbc": r"(?:rbc|red blood cell)[\s:=-]*(\d+\.?\d*)\s*(10\^6/μl|m/μl)?",
    # ... more patterns
}
```

### Database Indexing

**Indexes for Performance:**
```sql
CREATE INDEX idx_symptom_analyses_user_id ON symptom_analyses(user_id);
CREATE INDEX idx_symptom_analyses_timestamp ON symptom_analyses(timestamp);
CREATE INDEX idx_report_analyses_user_id ON report_analyses(user_id);
CREATE INDEX idx_report_analyses_timestamp ON report_analyses(timestamp);
CREATE INDEX idx_reference_ranges_parameter ON reference_ranges(parameter);
```

### Security Considerations

**File Upload Security:**
- Validate MIME type using python-magic (not just extension)
- Store files outside web root
- Use UUIDs for filenames (prevent path traversal)
- Scan for malware (optional: ClamAV integration)
- Set file permissions (read-only for web server)

**API Security:**
- Input validation using Pydantic
- SQL injection prevention (use ORM)
- XSS prevention (sanitize outputs)
- CSRF protection (if using sessions)
- Rate limiting per IP address
- CORS configuration (restrict origins)

**Data Privacy:**
- No PII in logs
- Secure file storage
- Optional: Encrypt sensitive data at rest
- Optional: User authentication and authorization
- Clear data retention policy

### Deployment Considerations

**Environment Variables (.env):**
```
# API Configuration
API_HOST=0.0.0.0
API_PORT=8000
API_WORKERS=4

# Database
DATABASE_URL=sqlite:///./medsort_ai.db

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-3.5-turbo
OPENAI_TEMPERATURE=0.3

# File Upload
UPLOAD_DIR=./uploads
MAX_FILE_SIZE=10485760

# ML Model
MODEL_PATH=./models/disease_classifier.pkl
SYMPTOM_LIST_PATH=./models/symptom_list.json

# Security
RATE_LIMIT_PER_MINUTE=10
CORS_ORIGINS=http://localhost:3000

# OCR
TESSERACT_PATH=/usr/bin/tesseract
```

**Production Recommendations:**
- Use PostgreSQL instead of SQLite
- Deploy behind reverse proxy (nginx)
- Enable HTTPS (Let's Encrypt)
- Use environment-specific configs
- Implement proper logging (structured logs)
- Set up monitoring (health checks, metrics)
- Use container orchestration (Docker, Kubernetes)
- Implement backup strategy
- Set up CI/CD pipeline

### Scalability Enhancements

**For Production Scale:**
- Replace SQLite with PostgreSQL
- Implement caching (Redis) for reference ranges
- Use message queue (Celery) for async processing
- Separate OCR processing to worker nodes
- Implement CDN for frontend assets
- Use load balancer for multiple API instances
- Implement database connection pooling
- Add database read replicas
- Implement file storage on S3/cloud storage
- Add monitoring and alerting (Prometheus, Grafana)
