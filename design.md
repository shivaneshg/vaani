# Vaani – Living Knowledge Infrastructure
## Technical Design Document v1.0
### AWS AI for Bharat – Architecture Submission

---

## 1. System Overview

### 1.1 Purpose

Vaani is a voice-first AI infrastructure platform designed to preserve indigenous oral knowledge across India's linguistic landscape. The system transforms oral traditions into structured, consent-governed digital datasets by capturing audio recordings, performing speech-to-text conversion, translating to multiple languages, and enabling searchable storage.

### 1.2 Architectural Philosophy

Vaani follows a serverless-first, event-driven architecture built entirely on AWS managed services. This approach eliminates operational overhead, enables automatic scaling, and optimizes costs through pay-per-use pricing. The architecture prioritizes:

- **Serverless-first:** No server management, automatic scaling, reduced operational complexity
- **Event-driven orchestration:** Asynchronous processing decouples components and improves fault isolation
- **Data sovereignty:** All data stored in AWS Mumbai region (ap-south-1) to address indigenous knowledge protection
- **Security by design:** Encryption, least-privilege access, audit logging built into every layer
- **Cost optimization:** Pay-per-use services, intelligent storage tiering, efficient resource utilization

### 1.3 Why Serverless and Event-Driven

Traditional server-based architectures require capacity planning, server management, and scaling configuration. For a platform with unpredictable workloads (field researchers uploading batches of recordings intermittently), serverless architecture provides:

1. **Elastic scaling:** Lambda functions scale from 0 to 1000+ concurrent executions automatically
2. **Cost efficiency:** Pay only for actual compute time, not idle server capacity
3. **Fault isolation:** Failed processing jobs don't impact other recordings
4. **Simplified operations:** AWS manages patching, availability, and infrastructure
5. **Event-driven workflows:** S3 events trigger processing pipelines without polling or orchestration servers

---

## 2. Architectural Design Principles

### 2.1 Serverless-First

All compute workloads run on AWS Lambda. No EC2 instances, no container orchestration, no server management. Lambda functions handle API requests, process audio uploads, orchestrate transcription jobs, and manage translations.

### 2.2 Event-Driven Orchestration

S3 event notifications trigger Lambda functions when audio files are uploaded. Amazon Transcribe completion events trigger result processing. This asynchronous, event-driven model ensures non-blocking workflows and horizontal scalability.

### 2.3 Data Sovereignty

All infrastructure deployed in AWS Mumbai region (ap-south-1). Audio files, transcripts, translations, and metadata never leave India. This addresses indigenous community concerns about knowledge extraction and complies with data residency requirements.

### 2.4 Cost Optimization

- S3 Intelligent-Tiering automatically moves infrequently accessed audio to lower-cost storage tiers
- Lambda pay-per-use eliminates idle compute costs
- Amazon Transcribe and Translate charge per second of audio/character processed
- RDS free tier supports MVP; scales to paid tier only when needed
- Estimated MVP cost: <$50/month for 100 recordings

### 2.5 Security by Design

- All data encrypted at rest (S3 SSE, RDS encryption)
- All data encrypted in transit (TLS 1.2+)
- IAM roles enforce least-privilege access
- No public S3 bucket access; pre-signed URLs for controlled audio playback
- API Gateway throttling prevents abuse
- CloudTrail audit logging for compliance

### 2.6 Fault Isolation

Asynchronous processing ensures one failed transcription job doesn't impact other recordings. Lambda automatic retries handle transient failures. CloudWatch alarms notify administrators of persistent errors.

---

## 3. High-Level Architecture

### 3.1 Component Overview

**Frontend Layer:**
- CloudFront: CDN for low-latency content delivery
- S3 (Frontend): Static hosting for React.js web application

**API Layer:**
- API Gateway: RESTful API with authentication and throttling
- Lambda (API Handlers): Process API requests

**Storage Layer:**
- S3 (Audio): Durable audio file storage with encryption
- RDS (PostgreSQL): Metadata, transcripts, translations, consent records

**Processing Layer:**
- Lambda (Orchestrators): Trigger and manage async jobs
- Amazon Transcribe: Speech-to-text conversion
- Amazon Translate: Multilingual translation

**Orchestration Layer:**
- EventBridge: Event routing for Transcribe job completion

**Observability Layer:**
- CloudWatch: Logs, metrics, alarms
- CloudTrail: Audit logging

**Security Layer:**
- IAM: Role-based access control
- Cognito: User authentication (Phase 2)

### 3.2 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Browser                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CloudFront (CDN)                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                S3 Bucket (Frontend Hosting)                     │
│                    React.js Application                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway                                │
│              (REST API + Throttling + Auth)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Lambda (Upload Handler)                        │
│         - Generate pre-signed S3 URL                            │
│         - Create metadata record                                │
│         - Store consent                                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              S3 Bucket (Audio Storage)                          │
│                  - Encrypted at rest                            │
│                  - Intelligent-Tiering                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ (S3 Event Notification)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│          Lambda (Transcription Orchestrator)                    │
│         - Submit Amazon Transcribe async job                    │
│         - Update status: "transcribing"                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Amazon Transcribe (Async Job)                      │
│         - Hindi/English ASR                                     │
│         - Confidence scoring                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ (Job Completion Event)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│           Lambda (Result Processor)                             │
│         - Retrieve transcript from Transcribe                   │
│         - Store transcript in RDS                               │
│         - Trigger translation                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Amazon Translate                                   │
│         - Translate to Hindi/English                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              RDS (PostgreSQL)                                   │
│         - Metadata                                              │
│         - Transcripts                                           │
│         - Translations                                          │
│         - Consent records                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              CloudWatch                                         │
│         - Logs                                                  │
│         - Metrics                                               │
│         - Alarms                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Detailed Data Flow

### 4.1 Recording and Upload Flow

**Step 1: User Initiates Recording**
- User opens web application (served via CloudFront from S3)
- Browser MediaRecorder API captures audio
- User provides metadata: speaker name, language, location, topic
- User completes consent workflow

**Step 2: Pre-Signed URL Generation**
- Frontend calls API Gateway: `POST /api/recordings/initiate`
- Lambda (Upload Handler) executes:
  - Validates request
  - Generates unique recording ID (UUID)
  - Creates pre-signed S3 PUT URL (15-minute expiration)
  - Inserts metadata record in RDS with status "uploading"
  - Stores consent record (immutable)
  - Returns pre-signed URL to frontend

**Step 3: Client-Side Upload**
- Frontend uploads audio directly to S3 using pre-signed URL
- No audio data passes through API Gateway or Lambda (bandwidth optimization)
- S3 stores audio with server-side encryption (SSE-S3)

**Step 4: Upload Confirmation**
- Frontend calls API Gateway: `POST /api/recordings/{id}/confirm`
- Lambda updates RDS status to "uploaded"

### 4.2 Transcription Flow

**Step 5: S3 Event Trigger**
- S3 emits event notification when audio file upload completes
- Event triggers Lambda (Transcription Orchestrator)

**Step 6: Transcribe Job Submission**
- Lambda (Transcription Orchestrator) executes:
  - Retrieves metadata from RDS
  - Determines language (Hindi/English/Gondi)
  - If language supported by Transcribe:
    - Submits Amazon Transcribe async job with:
      - S3 audio URI
      - Language code
      - Output S3 location
    - Updates RDS status to "transcribing"
    - Stores Transcribe job ID
  - If language unsupported (e.g., Gondi):
    - Updates RDS status to "needs_manual_review"
    - Admin notified via CloudWatch alarm

**Step 7: Transcribe Processing**
- Amazon Transcribe processes audio asynchronously
- Generates transcript with timestamps and confidence scores
- Writes output to S3 (JSON format)

**Step 8: Transcribe Completion Event**
- Amazon EventBridge rule listens for Transcribe job state changes (event pattern: `TranscriptionJobStatus = COMPLETED`)
- EventBridge triggers Lambda (Result Processor) when job completes
- Alternative: SNS notification channel configured in Transcribe job submission

### 4.3 Translation and Storage Flow

**Step 9: Transcript Retrieval**
- Lambda (Result Processor) executes:
  - Retrieves transcript JSON from S3
  - Parses transcript text and confidence scores
  - Stores transcript in RDS linked to recording ID

**Step 10: Translation**
- Lambda calls Amazon Translate API:
  - If source language is Gondi: translate to Hindi and English
  - If source language is Hindi: translate to English
  - If source language is English: no translation needed
- Stores translations in RDS

**Step 11: Status Update**
- Lambda updates RDS status to "complete"
- Records processing completion timestamp

### 4.4 Search and Retrieval Flow

**Step 12: Search Request**
- User submits search query via frontend
- Frontend calls API Gateway: `GET /api/recordings/search?q=...&language=...&topic=...`
- Lambda (Search Handler) executes:
  - Queries RDS with filters (full-text search on transcripts/translations)
  - Returns paginated results with metadata

**Step 13: Audio Playback**
- User clicks recording to play audio
- Frontend calls API Gateway: `GET /api/recordings/{id}/audio-url`
- Lambda (Audio URL Handler) executes:
  - Verifies consent permissions
  - Generates pre-signed S3 GET URL (1-hour expiration)
  - Returns URL to frontend
- Frontend plays audio using HTML5 audio player

---

## 5. Component Responsibilities

### 5.1 CloudFront
**Purpose:** Content delivery network for low-latency frontend access
**Responsibilities:**
- Cache static assets (HTML, CSS, JS, images)
- Serve React.js application globally with low latency
- HTTPS termination

**Why Chosen:** Reduces latency for users across India, offloads traffic from S3

### 5.2 S3 (Frontend Hosting)
**Purpose:** Static website hosting for React.js application
**Responsibilities:**
- Store compiled frontend assets
- Serve index.html and static resources

**Why Chosen:** Cost-effective static hosting, integrates seamlessly with CloudFront

### 5.3 API Gateway
**Purpose:** RESTful API endpoint for frontend-backend communication
**Responsibilities:**
- Route HTTP requests to Lambda functions
- Request validation
- Throttling (1000 requests/second default)
- CORS configuration
- API key authentication (Phase 1) / Cognito integration (Phase 2)

**Why Chosen:** Managed service, automatic scaling, built-in security features

### 5.4 Lambda Functions
**Purpose:** Serverless compute for business logic
**Responsibilities:**
- **Upload Handler:** Generate pre-signed URLs, create metadata records, store consent
- **Transcription Orchestrator:** Submit Transcribe jobs, update status
- **Result Processor:** Retrieve transcripts, trigger translations, store results
- **Search Handler:** Query RDS, return filtered results
- **Audio URL Handler:** Generate pre-signed S3 URLs for playback

**Why Chosen:** No server management, automatic scaling, pay-per-use pricing

### 5.5 S3 (Audio Storage)
**Purpose:** Durable, scalable storage for audio files
**Responsibilities:**
- Store audio files with encryption (SSE-S3)
- Emit event notifications on upload
- Lifecycle policies for cost optimization (Intelligent-Tiering)

**Why Chosen:** 99.999999999% durability, automatic scaling, event-driven triggers

### 5.6 Amazon Transcribe
**Purpose:** Speech-to-text conversion
**Responsibilities:**
- Convert Hindi/English audio to text
- Generate confidence scores
- Support async job processing

**Why Chosen:** Production-ready ASR for Hindi/English, managed service, async processing

**Phase 1 Limitation:** Gondi not supported by Amazon Transcribe

**Manual Review Workflow:**
1. Lambda (Transcription Orchestrator) detects unsupported language
2. Updates RDS status to "needs_manual_review"
3. Admin dashboard displays flagged recordings
4. Administrator downloads audio, transcribes manually, uploads transcript via admin portal
5. Lambda (Manual Transcript Handler) stores transcript in RDS
6. Triggers translation workflow
7. Updates status to "complete"

**Phase 2 Enhancement:** SageMaker custom model training for Gondi with community-approved datasets

### 5.7 Amazon Translate
**Purpose:** Neural machine translation
**Responsibilities:**
- Translate Gondi → Hindi
- Translate Gondi → English
- Translate Hindi → English

**Why Chosen:** Supports 75+ languages, neural translation quality, pay-per-character pricing

### 5.8 RDS (PostgreSQL)
**Purpose:** Relational database for structured data
**Responsibilities:**
- Store metadata (speaker, language, location, date, topic, status)
- Store transcripts and translations
- Store consent records (immutable)
- Support full-text search on transcripts

**Why Chosen:** ACID compliance, relational integrity, full-text search, familiar SQL interface

**Why Not DynamoDB:** MVP requires complex queries (full-text search, multi-field filters). PostgreSQL provides better query flexibility. DynamoDB considered for Phase 2 if access patterns become predictable.

### 5.9 CloudWatch
**Purpose:** Monitoring, logging, and alerting
**Responsibilities:**
- Aggregate Lambda logs
- Track metrics (invocation count, duration, errors)
- Alarms for error rates, processing failures
- Dashboard for operational visibility

**Why Chosen:** Native AWS integration, centralized observability

### 5.10 EventBridge
**Purpose:** Event-driven orchestration
**Responsibilities:**
- Listen for Transcribe job state changes (COMPLETED, FAILED)
- Trigger Lambda (Result Processor) on job completion
- Route events to appropriate Lambda functions

**Why Chosen:** Native AWS event bus, serverless, enables decoupled architecture

### 5.11 IAM
**Purpose:** Identity and access management
**Responsibilities:**
- Lambda execution roles with least-privilege permissions
- S3 bucket policies restricting public access
- API Gateway authorization

**Why Chosen:** Fine-grained access control, security best practices

### 5.12 CloudTrail
**Purpose:** Audit logging
**Responsibilities:**
- Log all API calls (S3 access, RDS queries, Lambda invocations)
- Compliance and security auditing

**Why Chosen:** Required for sensitive indigenous knowledge data governance

---

## 6. API Design

### 6.1 Endpoints

**POST /api/recordings/initiate**
- **Purpose:** Generate pre-signed S3 URL for audio upload
- **Request Body:**
  ```json
  {
    "speaker_name": "string",
    "language": "hindi|english|gondi",
    "location": "string",
    "topic": "string",
    "consent": {
      "audio_url": "string",
      "access_level": "public|research_only",
      "timestamp": "ISO8601"
    }
  }
  ```
- **Response:**
  ```json
  {
    "recording_id": "uuid",
    "upload_url": "pre-signed S3 URL",
    "expires_in": 900
  }
  ```

**POST /api/recordings/{id}/confirm**
- **Purpose:** Confirm upload completion
- **Response:** `{ "status": "uploaded" }`

**GET /api/recordings/search**
- **Purpose:** Search recordings
- **Query Parameters:** `q` (text), `language`, `topic`, `date_from`, `date_to`, `page`, `limit`
- **Response:**
  ```json
  {
    "results": [
      {
        "recording_id": "uuid",
        "speaker_name": "string",
        "language": "string",
        "topic": "string",
        "date": "ISO8601",
        "transcript_snippet": "string"
      }
    ],
    "total": 100,
    "page": 1
  }
  ```

**GET /api/recordings/{id}**
- **Purpose:** Get recording details
- **Response:**
  ```json
  {
    "recording_id": "uuid",
    "speaker_name": "string",
    "language": "string",
    "location": "string",
    "topic": "string",
    "date": "ISO8601",
    "status": "complete",
    "transcript": "string",
    "translations": {
      "hindi": "string",
      "english": "string"
    },
    "consent": {
      "access_level": "public",
      "timestamp": "ISO8601"
    }
  }
  ```

**GET /api/recordings/{id}/audio-url**
- **Purpose:** Generate pre-signed URL for audio playback
- **Response:**
  ```json
  {
    "audio_url": "pre-signed S3 URL",
    "expires_in": 3600
  }
  ```

**GET /api/admin/recordings**
- **Purpose:** Admin view of all recordings with status
- **Response:** Paginated list with processing status

### 6.2 Authentication

**Phase 1 (MVP):** API key authentication via API Gateway
**Phase 2:** AWS Cognito user pools with role-based access control

---

## 7. Database Design

### 7.1 Schema

**Table: recordings**
```sql
CREATE TABLE recordings (
  id UUID PRIMARY KEY,
  speaker_name VARCHAR(255) NOT NULL,
  language VARCHAR(50) NOT NULL,
  location VARCHAR(255),
  topic VARCHAR(100),
  status VARCHAR(50) NOT NULL, -- uploading, uploaded, transcribing, translating, complete, error
  s3_audio_key VARCHAR(500) NOT NULL,
  transcribe_job_id VARCHAR(255),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_recordings_language ON recordings(language);
CREATE INDEX idx_recordings_topic ON recordings(topic);
CREATE INDEX idx_recordings_status ON recordings(status);
CREATE INDEX idx_recordings_created_at ON recordings(created_at);
```

**Table: transcripts**
```sql
CREATE TABLE transcripts (
  id SERIAL PRIMARY KEY,
  recording_id UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
  transcript_text TEXT NOT NULL,
  confidence_score DECIMAL(5,4),
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transcripts_recording_id ON transcripts(recording_id);
CREATE INDEX idx_transcripts_fulltext ON transcripts USING GIN(to_tsvector('english', transcript_text));
```

**Table: translations**
```sql
CREATE TABLE translations (
  id SERIAL PRIMARY KEY,
  recording_id UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
  target_language VARCHAR(50) NOT NULL,
  translated_text TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_translations_recording_id ON translations(recording_id);
CREATE INDEX idx_translations_fulltext ON translations USING GIN(to_tsvector('english', translated_text));
```

**Table: consent_records**
```sql
CREATE TABLE consent_records (
  id SERIAL PRIMARY KEY,
  recording_id UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
  consent_audio_s3_key VARCHAR(500),
  access_level VARCHAR(50) NOT NULL, -- public, research_only
  commercial_use_allowed BOOLEAN NOT NULL DEFAULT FALSE,
  timestamp TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consent_recording_id ON consent_records(recording_id);
```

### 7.2 Why RDS (PostgreSQL)

**Advantages for MVP:**
1. **Relational Integrity:** Foreign key constraints ensure data consistency
2. **Full-Text Search:** PostgreSQL GIN indexes enable efficient text search across transcripts
3. **Complex Queries:** Support for multi-field filters, joins, aggregations
4. **ACID Compliance:** Critical for immutable consent records
5. **Familiar Tooling:** Standard SQL, mature ecosystem

**DynamoDB Consideration:**
- Phase 2 may introduce DynamoDB for high-throughput read workloads if access patterns become predictable
- Current MVP requires query flexibility that PostgreSQL provides better

---

## 8. Consent & Data Governance Architecture

### 8.1 Consent Recording

**Workflow:**
1. User records verbal consent statement (audio)
2. User selects access level (public / research-only)
3. User confirms no commercial use (default)
4. Frontend uploads consent audio to S3 (separate bucket or prefix)
5. Lambda stores consent record in `consent_records` table with timestamp
6. Consent record is immutable (no UPDATE operations allowed)

### 8.2 Access Control

**Phase 1 (MVP):**
- All recordings with "public" access level are searchable and playable
- "research_only" recordings require admin approval (manual process)

**Phase 2 (Community Governance):**
- AWS Cognito user pools with custom attributes (community_id, role)
- Lambda authorizers check user permissions before generating pre-signed URLs
- Community leaders can approve/deny access requests via admin portal

### 8.3 Pre-Signed URLs

**Security Model:**
- S3 buckets have no public access
- Audio playback requires pre-signed URL generation
- Lambda verifies consent access_level before generating URL
- URLs expire after 1 hour (configurable)
- CloudTrail logs all URL generation events for audit

### 8.4 Immutable Consent Storage

- `consent_records` table has no UPDATE or DELETE operations in application code
- Database triggers prevent modification (Phase 2)
- Consent withdrawal requires new record with status "withdrawn" (soft delete)

---

## 9. Security Architecture

### 9.1 IAM Least Privilege

**Lambda Execution Roles:**
- **Upload Handler:** S3 PutObject (audio bucket), RDS INSERT (recordings, consent_records)
- **Transcription Orchestrator:** S3 GetObject (audio bucket), Transcribe StartTranscriptionJob, RDS UPDATE (recordings)
- **Result Processor:** S3 GetObject (transcript bucket), Translate TranslateText, RDS INSERT/UPDATE (transcripts, translations, recordings)
- **Search Handler:** RDS SELECT (read-only)
- **Audio URL Handler:** S3 GetObject (pre-signed URL generation), RDS SELECT (consent verification)

**S3 Bucket Policies:**
- Deny all public access
- Allow Lambda execution roles only
- Require encryption in transit (TLS)

### 9.2 Encryption

**At Rest:**
- S3: Server-side encryption (SSE-S3 or SSE-KMS)
- RDS: Encryption enabled at database creation
- CloudWatch Logs: Encrypted by default

**In Transit:**
- CloudFront: HTTPS only
- API Gateway: TLS 1.2+
- RDS: SSL/TLS connections enforced

### 9.3 API Gateway Security

**Throttling:**
- Default: 1000 requests/second per account
- Burst: 2000 requests
- Prevents abuse and DDoS

**Authentication:**
- Phase 1: API keys
- Phase 2: AWS Cognito authorizer

**CORS:**
- Restrict origins to CloudFront distribution domain

### 9.4 Audit Logging

**CloudTrail:**
- Log all S3 object access (data events)
- Log all Lambda invocations
- Log all RDS API calls
- Retain logs for 90 days (configurable)

**CloudWatch Logs:**
- Lambda function logs (INFO, WARN, ERROR levels)
- API Gateway access logs
- Retain logs for 30 days

### 9.5 No Public S3 Access

- Block all public access at bucket level
- Pre-signed URLs are the only mechanism for audio playback
- URLs are time-limited and auditable

---

## 10. Scalability Strategy

### 10.1 S3 Auto-Scaling

- S3 scales automatically to handle unlimited objects
- No capacity planning required
- Supports 3,500 PUT requests/second per prefix

### 10.2 Lambda Concurrency Scaling

- Lambda scales to 1000 concurrent executions (default account limit)
- Can request limit increase to 10,000+
- Each recording processes independently (no shared state)

### 10.3 Async Job Isolation

- Amazon Transcribe jobs run independently
- Failed job doesn't impact other recordings
- Retry logic handles transient failures

### 10.4 RDS Scaling

**Vertical Scaling:**
- Start with db.t3.micro (free tier)
- Scale to db.t3.small, db.t3.medium as needed

**Horizontal Scaling:**
- Read replicas for search queries (Phase 2)
- Write operations remain on primary instance

**Connection Pooling (Critical for Lambda):**
- RDS Proxy recommended to prevent connection exhaustion under concurrent Lambda scaling
- Without RDS Proxy, Lambda functions can exhaust database connections (default PostgreSQL max_connections = 100)
- RDS Proxy pools connections and enables thousands of concurrent Lambda executions
- Phase 1: Implement connection limits in Lambda code (max 5 connections per function)
- Phase 2: Deploy RDS Proxy for production-grade connection management

### 10.5 Horizontal Scale to 100k+ Recordings

**Current Architecture Supports:**
- S3: Unlimited storage
- Lambda: 1000+ concurrent executions
- Transcribe: 100 concurrent jobs (default), can request increase
- RDS: Scales to millions of rows with proper indexing

**Bottleneck Mitigation:**
- RDS read replicas for search queries
- ElastiCache for frequently accessed metadata (Phase 3)
- DynamoDB for high-throughput access patterns (Phase 3)

---

## 11. Cost Optimization Strategy

### 11.1 S3 Intelligent-Tiering

- Automatically moves objects between access tiers based on usage
- Frequent Access: First 30 days
- Infrequent Access: 30-90 days
- Archive: 90+ days
- Reduces storage costs by 40-60% for older recordings

### 11.2 Lambda Pay-Per-Use

- No charges for idle time
- Charged per invocation and GB-second
- Estimated cost: $0.20 per 1 million requests + $0.0000166667 per GB-second

### 11.3 Transcribe Billing Model

- $0.024 per minute of audio (Hindi/English)
- 30-minute recording: $0.72
- 100 recordings (30 min each): $72/month

### 11.4 Translate Billing Model

- $15 per million characters
- Average transcript: 5,000 characters
- 100 recordings × 2 translations: 1 million characters = $15/month

### 11.5 RDS Free Tier

- db.t3.micro: 750 hours/month free (first 12 months)
- 20 GB storage free
- Post-free-tier: ~$15-30/month for db.t3.micro

### 11.6 Estimated MVP Cost Projection

**Light Usage (10-20 recordings/month):**

| Service | Cost |
|---------|------|
| S3 (Audio Storage) | $0.50/month (10 GB) |
| S3 (Frontend Hosting) | $0.50/month |
| CloudFront | $1/month (low traffic) |
| Lambda | $2/month (estimated invocations) |
| API Gateway | $1/month (100k requests) |
| Transcribe | $7/month (10 recordings × 30 min) |
| Translate | $2/month |
| RDS | Free tier |
| CloudWatch | $1/month |
| **Total** | **~$15/month** (with free tier) |

**Heavy Usage (100 recordings/month, 30 minutes each):**

| Service | Cost |
|---------|------|
| S3 (Audio Storage) | $2/month (100 GB) |
| S3 (Frontend Hosting) | $0.50/month |
| CloudFront | $1/month (low traffic) |
| Lambda | $5/month (estimated invocations) |
| API Gateway | $3.50/month (1 million requests) |
| Transcribe | $72/month (100 recordings) |
| Translate | $15/month |
| RDS | Free tier (or $15/month post-free-tier) |
| CloudWatch | $2/month |
| **Total** | **~$50/month** (with free tier) or **~$100/month** (post-free-tier) |

**Cost Scaling:** Transcribe is the primary variable cost. Storage and compute costs remain low due to serverless architecture.

---

## 12. Reliability & Fault Handling

### 12.1 Lambda Retries

- Automatic retry on failure (2 retries by default)
- Exponential backoff
- Failed invocations logged to CloudWatch

### 12.2 Async Job Recovery

**Transcribe Job Failure:**
- Lambda (Result Processor) checks job status
- If failed: updates RDS status to "error"
- CloudWatch alarm triggers notification
- Admin can manually retry via admin portal

**Translation Failure:**
- Lambda catches exception
- Stores partial results (transcript without translation)
- Updates status to "partial_complete"
- Admin can manually trigger translation retry

### 12.3 CloudWatch Alarms

**Critical Alarms:**
- Lambda error rate > 5%
- Transcribe job failure rate > 10%
- RDS CPU utilization > 80%
- S3 4xx/5xx error rate > 1%

**Notification:** SNS topic → Email/SMS to administrators

### 12.4 Future DLQ Strategy

**Phase 2 Enhancement:**
- Dead Letter Queue (SQS) for failed Lambda invocations
- Separate Lambda function processes DLQ messages
- Retry logic with exponential backoff
- Manual intervention for persistent failures

---

## 13. Deployment Strategy

### 13.1 AWS Region

**Primary Region:** ap-south-1 (Mumbai)

**Rationale:**
- Data sovereignty (all data remains in India)
- Low latency for Indian users
- Compliance with indigenous data protection requirements

**Future Multi-Region:** Not planned for MVP; Phase 3 may add disaster recovery region

### 13.2 Infrastructure as Code

**Tool:** AWS CloudFormation

**Resources Defined:**
- S3 buckets (frontend, audio, transcripts)
- CloudFront distribution
- API Gateway REST API
- Lambda functions (code uploaded separately)
- RDS instance
- IAM roles and policies
- CloudWatch alarms

**Benefits:**
- Reproducible deployments
- Version-controlled infrastructure
- Easy rollback
- Environment parity (dev, staging, prod)

### 13.3 Version Control

**Repository Structure:**
```
vaani-infrastructure/
├── cloudformation/
│   ├── storage.yaml (S3, RDS)
│   ├── compute.yaml (Lambda, API Gateway)
│   ├── security.yaml (IAM roles)
│   └── monitoring.yaml (CloudWatch)
├── lambda/
│   ├── upload-handler/
│   ├── transcription-orchestrator/
│   ├── result-processor/
│   ├── search-handler/
│   └── audio-url-handler/
├── frontend/
│   └── react-app/
└── README.md
```

### 13.4 CI/CD (Optional Future)

**Phase 2 Enhancement:**
- GitHub Actions or AWS CodePipeline
- Automated testing (unit, integration)
- Automated CloudFormation stack updates
- Blue-green deployments for Lambda functions

**MVP Deployment:** Manual CloudFormation stack creation and Lambda function uploads

---

## 14. Future Architecture Evolution

### 14.1 SageMaker Fine-Tuning for Gondi

**Challenge:** Amazon Transcribe does not support Gondi language

**Solution:**
- Collect community-approved Gondi audio datasets (100+ hours)
- Fine-tune Wav2Vec2 or Whisper model using SageMaker
- Deploy custom model as SageMaker endpoint
- Lambda calls SageMaker endpoint instead of Transcribe for Gondi recordings

**Cost:** SageMaker endpoint hosting ~$50-100/month (ml.t3.medium instance)

### 14.2 Amazon Comprehend Topic Extraction

**Enhancement:** Automatic topic tagging

**Implementation:**
- Lambda (Result Processor) calls Amazon Comprehend after translation
- Comprehend extracts key phrases and topics
- Store topics in `recordings` table
- Enable topic-based search and filtering

**Cost:** $0.0001 per unit (100 characters)

### 14.3 Mobile App Integration

**Architecture Changes:**
- AWS Amplify for mobile backend
- Cognito for mobile authentication
- AppSync (GraphQL) for real-time sync (optional)
- S3 Transfer Utility for mobile uploads

**Offline Support:**
- Record audio locally on mobile device
- Queue uploads for when connectivity available
- Background sync using AWS Amplify DataStore

### 14.4 Knowledge Graph Layer

**Enhancement:** Semantic relationships between recordings

**Implementation:**
- Amazon Neptune (graph database) stores relationships
- Lambda extracts entities (people, places, plants, rituals) from transcripts
- Build graph: Recording → mentions → Entity
- Enable graph-based exploration (e.g., "all recordings mentioning turmeric")

**Cost:** Neptune db.t3.medium ~$70/month

### 14.5 Community Governance Portal

**Enhancement:** Community leaders manage access

**Implementation:**
- Cognito user pools with custom attributes (community_id, role)
- Lambda authorizers enforce community-level access control
- Admin portal (React.js) for community leaders
- Approval workflows for research access requests

---

## 15. Closing Architecture Statement

Vaani's architecture demonstrates how AWS serverless services can build national-scale voice-first infrastructure for India's linguistic diversity. The design prioritizes:

1. **Serverless-first:** Eliminates operational overhead, enables automatic scaling, optimizes costs
2. **Event-driven orchestration:** Asynchronous processing ensures fault isolation and horizontal scalability
3. **Data sovereignty:** All infrastructure in AWS Mumbai region protects indigenous knowledge
4. **Security by design:** Encryption, least-privilege access, audit logging at every layer
5. **Cost efficiency:** Pay-per-use pricing makes pilot deployment affordable (<$100/month)

This architecture is production-ready for MVP deployment, scales to 100,000+ recordings without redesign, and provides a foundation for future enhancements (custom ASR models, knowledge graphs, mobile apps).

Vaani is not just a preservation tool—it is digital public infrastructure that enables climate resilience, language revitalization, and ethically governed research. AWS provides the technical foundation to build this responsibly at scale, serving communities that Digital India has yet to reach.

---

## 16. AWS Well-Architected Framework Alignment

This architecture aligns with the five pillars of the AWS Well-Architected Framework:

### 16.1 Operational Excellence
- **Infrastructure as Code:** CloudFormation templates enable reproducible deployments
- **Monitoring & Logging:** CloudWatch provides centralized observability for all components
- **Automated Responses:** EventBridge triggers Lambda functions based on state changes
- **Continuous Improvement:** Modular architecture enables incremental enhancements

### 16.2 Security
- **Identity & Access Management:** IAM roles enforce least-privilege access for all services
- **Data Protection:** Encryption at rest (S3, RDS) and in transit (TLS) for all data
- **Audit Logging:** CloudTrail tracks all API calls for compliance and security auditing
- **Network Security:** No public S3 access; pre-signed URLs provide time-limited, auditable access
- **Consent Governance:** Immutable consent records ensure indigenous data sovereignty

### 16.3 Reliability
- **Fault Isolation:** Asynchronous processing ensures one failed job doesn't impact others
- **Automatic Retries:** Lambda and Transcribe handle transient failures with exponential backoff
- **Monitoring & Alarms:** CloudWatch alarms notify administrators of critical failures
- **Data Durability:** S3 provides 99.999999999% durability; RDS automated backups enable point-in-time recovery

### 16.4 Performance Efficiency
- **Serverless Compute:** Lambda scales automatically to handle variable workloads
- **Managed Services:** Transcribe, Translate, and RDS eliminate infrastructure tuning
- **Event-Driven Architecture:** Non-blocking async workflows maximize throughput
- **Content Delivery:** CloudFront reduces latency for users across India

### 16.5 Cost Optimization
- **Pay-Per-Use Pricing:** Lambda and Transcribe charge only for actual usage
- **Storage Tiering:** S3 Intelligent-Tiering reduces costs for infrequently accessed audio
- **Right-Sizing:** Start with free-tier RDS, scale only when needed
- **No Idle Resources:** Serverless architecture eliminates costs for idle capacity

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Document Owner:** Technical Architecture Team  
**Target:** AWS AI for Bharat Architecture Submission
