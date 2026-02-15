# Requirements Document: MedSort AI

## Introduction

MedSort AI is a GenAI-powered intelligent healthcare assistant that helps users understand their health status through symptom analysis and medical report interpretation. The system accepts natural language symptom descriptions, predicts possible diseases using machine learning, processes uploaded medical lab reports using OCR, and provides personalized health guidance through a generative AI layer. The application emphasizes user safety by providing clear medical disclaimers and encouraging professional medical consultation.

## Glossary

- **System**: The MedSort AI application (frontend + backend + ML models)
- **User**: An individual using the application to analyze symptoms or medical reports
- **Symptom_Analyzer**: The ML-based component that predicts diseases from symptoms
- **Report_Processor**: The OCR and parsing component that extracts lab parameters from medical reports
- **GenAI_Layer**: The generative AI component that provides explanations and guidance
- **Lab_Parameter**: A measurable value from a medical lab report (e.g., Hemoglobin, WBC, RBC, Platelets)
- **Reference_Range**: Age and gender-specific normal value ranges for lab parameters
- **Risk_Level**: Classification of disease probability (Low, Moderate, High)
- **Medical_Report**: A PDF or image file containing lab test results
- **Disease_Prediction**: A structured output containing predicted diseases with probability scores

## Requirements

### Requirement 1: Symptom Input and Processing

**User Story:** As a user, I want to describe my symptoms in natural language, so that I can get disease predictions without needing medical terminology.

#### Acceptance Criteria

1. WHEN a user submits symptom text, THE Symptom_Analyzer SHALL accept free-text input of at least 10 characters
2. WHEN symptom text is received, THE GenAI_Layer SHALL extract a structured list of medical symptoms from the natural language input
3. WHEN symptoms are extracted, THE Symptom_Analyzer SHALL convert the symptom list into a feature vector compatible with the ML model
4. IF the symptom text is empty or contains only whitespace, THEN THE System SHALL reject the input and return a validation error
5. IF the symptom extraction fails, THEN THE System SHALL return an error message and log the failure

### Requirement 2: Disease Prediction

**User Story:** As a user, I want to receive disease predictions based on my symptoms, so that I can understand potential health concerns.

#### Acceptance Criteria

1. WHEN a feature vector is generated, THE Symptom_Analyzer SHALL use a trained classification model to predict diseases
2. WHEN predictions are made, THE Symptom_Analyzer SHALL return the top 3 most probable diseases with probability scores
3. WHEN predictions are returned, THE System SHALL include a risk level classification (Low, Moderate, or High) for each disease
4. THE Symptom_Analyzer SHALL use a Random Forest or XGBoost classifier trained on validated medical data
5. WHEN probability scores are below 0.3, THE System SHALL classify the risk level as Low
6. WHEN probability scores are between 0.3 and 0.7, THE System SHALL classify the risk level as Moderate
7. WHEN probability scores are above 0.7, THE System SHALL classify the risk level as High

### Requirement 3: Medical Report Upload

**User Story:** As a user, I want to upload my medical lab reports as PDF or images, so that I can get automated analysis of my test results.

#### Acceptance Criteria

1. WHEN a user uploads a file, THE System SHALL accept PDF and image formats (PNG, JPG, JPEG)
2. WHEN a file is uploaded, THE System SHALL validate that the file size does not exceed 10MB
3. WHEN a file is uploaded, THE System SHALL validate the file type matches allowed formats
4. IF an invalid file type is uploaded, THEN THE System SHALL reject the upload and return a descriptive error message
5. IF the file size exceeds the limit, THEN THE System SHALL reject the upload and return a size limit error
6. WHEN a valid file is uploaded, THE System SHALL store the file securely with a unique identifier
7. WHEN a file is stored, THE System SHALL associate it with the user session or user ID

### Requirement 4: Lab Parameter Extraction

**User Story:** As a user, I want my lab report values to be automatically extracted, so that I don't have to manually enter test results.

#### Acceptance Criteria

1. WHEN a medical report is uploaded, THE Report_Processor SHALL extract text using OCR technology
2. WHEN text is extracted, THE Report_Processor SHALL parse lab parameters including Hemoglobin, WBC, RBC, Platelets, and other common blood test values
3. WHEN parameters are parsed, THE Report_Processor SHALL extract both the parameter name and its numeric value
4. WHEN parameters are parsed, THE Report_Processor SHALL extract the unit of measurement when present
5. IF OCR extraction fails, THEN THE System SHALL return an error indicating the report could not be processed
6. IF no lab parameters are found in the extracted text, THEN THE System SHALL return an error indicating no valid lab data was detected

### Requirement 5: Lab Value Analysis

**User Story:** As a user, I want my lab values compared against normal ranges, so that I can identify abnormal results.

#### Acceptance Criteria

1. WHEN lab parameters are extracted, THE System SHALL compare each value against age-specific reference ranges
2. WHEN lab parameters are extracted, THE System SHALL compare each value against gender-specific reference ranges
3. WHEN a lab value is below the reference range, THE System SHALL mark it as "Low"
4. WHEN a lab value is within the reference range, THE System SHALL mark it as "Normal"
5. WHEN a lab value is above the reference range, THE System SHALL mark it as "High"
6. WHEN analysis is complete, THE System SHALL return structured data containing parameter name, user value, normal range, and status for each lab parameter
7. WHERE age or gender information is not provided, THE System SHALL use adult general reference ranges

### Requirement 6: GenAI Health Guidance

**User Story:** As a user, I want personalized explanations of my health results, so that I can understand what my symptoms or lab values mean.

#### Acceptance Criteria

1. WHEN disease predictions are generated, THE GenAI_Layer SHALL provide a simple explanation for each predicted disease
2. WHEN abnormal lab values are detected, THE GenAI_Layer SHALL provide explanations for each abnormal parameter
3. WHEN generating explanations, THE GenAI_Layer SHALL include possible causes for the condition or abnormal value
4. WHEN generating explanations, THE GenAI_Layer SHALL include diet suggestions relevant to the condition
5. WHEN generating explanations, THE GenAI_Layer SHALL include lifestyle recommendations
6. WHEN generating explanations, THE GenAI_Layer SHALL include guidance on when to consult a doctor
7. WHEN generating any health guidance, THE GenAI_Layer SHALL avoid making definitive diagnoses
8. WHEN generating any health guidance, THE GenAI_Layer SHALL encourage professional medical consultation

### Requirement 7: Medical Safety and Disclaimers

**User Story:** As a user, I want clear disclaimers about the limitations of the system, so that I understand this is not a replacement for professional medical advice.

#### Acceptance Criteria

1. WHEN any prediction or analysis is displayed, THE System SHALL include a prominent medical disclaimer
2. THE System SHALL state that results are for informational purposes only and not medical advice
3. THE System SHALL advise users to consult healthcare professionals for medical decisions
4. WHEN high-risk predictions are made, THE System SHALL emphasize the importance of immediate medical consultation
5. THE GenAI_Layer SHALL not provide treatment recommendations or prescribe medications

### Requirement 8: API Endpoints

**User Story:** As a frontend developer, I want well-defined REST API endpoints, so that I can integrate the backend services.

#### Acceptance Criteria

1. THE System SHALL provide a POST endpoint at /predict-symptoms that accepts symptom text and returns disease predictions
2. THE System SHALL provide a POST endpoint at /upload-report that accepts file uploads and returns analysis results
3. THE System SHALL provide a GET endpoint at /report/:id that retrieves stored report analysis by ID
4. THE System SHALL provide a GET endpoint at /health that returns system health status
5. WHEN any endpoint receives a request, THE System SHALL validate input data before processing
6. WHEN any endpoint encounters an error, THE System SHALL return appropriate HTTP status codes and error messages
7. WHEN endpoints return data, THE System SHALL use JSON format

### Requirement 9: Data Persistence

**User Story:** As a user, I want my analysis history saved, so that I can review past results.

#### Acceptance Criteria

1. WHEN a symptom analysis is completed, THE System SHALL store the results in the database
2. WHEN a report analysis is completed, THE System SHALL store the results in the database
3. WHEN storing analysis results, THE System SHALL include a timestamp
4. WHEN storing analysis results, THE System SHALL associate them with a user session or user ID
5. THE System SHALL use SQLite as the database for storing user history and analysis results
6. WHEN retrieving stored results, THE System SHALL return all associated data including predictions, lab values, and timestamps

### Requirement 10: User Interface

**User Story:** As a user, I want an intuitive dashboard interface, so that I can easily access symptom analysis and report upload features.

#### Acceptance Criteria

1. THE System SHALL provide a symptom input section in the dashboard
2. THE System SHALL provide a report upload section in the dashboard
3. WHEN abnormal lab values are displayed, THE System SHALL highlight them in red
4. WHEN risk levels are displayed, THE System SHALL show them as visual badges
5. THE System SHALL use a clean professional medical theme
6. THE System SHALL be responsive and functional on desktop and mobile devices
7. WHEN displaying results, THE System SHALL organize information in a clear, scannable layout

### Requirement 11: Model Training and Evaluation

**User Story:** As a data scientist, I want a model training pipeline, so that I can train and evaluate the disease prediction model.

#### Acceptance Criteria

1. THE System SHALL provide a model training script that loads training data
2. WHEN training the model, THE System SHALL split data into training and test sets
3. WHEN training is complete, THE System SHALL output accuracy and F1 score metrics
4. WHEN training is complete, THE System SHALL save the trained model as a .pkl file
5. THE System SHALL support retraining the model with updated datasets
6. WHEN evaluating the model, THE System SHALL use standard classification metrics

### Requirement 12: Security and Validation

**User Story:** As a system administrator, I want robust security measures, so that user data is protected and the system is resilient to attacks.

#### Acceptance Criteria

1. WHEN files are uploaded, THE System SHALL validate file types against an allowlist
2. WHEN files are uploaded, THE System SHALL scan for malicious content
3. WHEN API requests are received, THE System SHALL validate and sanitize all input data
4. WHEN errors occur, THE System SHALL log them without exposing sensitive information to users
5. THE System SHALL implement rate limiting on API endpoints to prevent abuse
6. WHEN storing files, THE System SHALL use secure file storage with unique identifiers
7. THE System SHALL not execute uploaded files or allow arbitrary code execution

### Requirement 13: Error Handling

**User Story:** As a user, I want clear error messages when something goes wrong, so that I understand what happened and what to do next.

#### Acceptance Criteria

1. WHEN an error occurs, THE System SHALL return a user-friendly error message
2. WHEN an error occurs, THE System SHALL log detailed error information for debugging
3. IF the ML model fails to load, THEN THE System SHALL return a service unavailable error
4. IF the GenAI API is unavailable, THEN THE System SHALL return an error indicating the explanation service is temporarily unavailable
5. IF OCR processing fails, THEN THE System SHALL provide guidance on uploading a clearer image
6. WHEN validation fails, THE System SHALL specify which input field caused the error

### Requirement 14: Configuration Management

**User Story:** As a developer, I want environment-based configuration, so that I can easily deploy the system across different environments.

#### Acceptance Criteria

1. THE System SHALL use environment variables for configuration
2. THE System SHALL store API keys and secrets in environment variables, not in code
3. THE System SHALL provide a .env.example file documenting required configuration
4. THE System SHALL validate that required environment variables are present at startup
5. IF required configuration is missing, THEN THE System SHALL fail to start and log the missing configuration

### Requirement 15: Performance and Scalability

**User Story:** As a system administrator, I want the system to handle multiple concurrent users, so that it can scale for production use.

#### Acceptance Criteria

1. WHEN processing symptom predictions, THE System SHALL return results within 5 seconds under normal load
2. WHEN processing report uploads, THE System SHALL handle files up to 10MB within 30 seconds
3. THE System SHALL support at least 10 concurrent API requests without degradation
4. WHEN the database grows, THE System SHALL maintain query performance through appropriate indexing
5. THE System SHALL implement connection pooling for database connections
