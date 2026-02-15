# VĀṆĪ (वाणी)
### India’s Living Knowledge Infrastructure

> **"VĀṆĪ ensures that when the last elder speaks, India listens — and remembers."**

VĀṆĪ is a voice-first AI platform that transforms vanishing indigenous oral knowledge into structured, searchable, and ethically governed digital intelligence.

---

## 🔷 The Problem
* **Rapid Loss:** India loses an indigenous language every two weeks.
* **The Orality Gap:** 70% of indigenous wisdom exists only in oral form; it is digitally invisible and unindexed.
* **Modern AI Bias:** Current LLMs focus on urban, text-heavy data, ignoring 19,500+ Indian dialects.

## 🔷 The Solution
VĀṆĪ treats **audio as the primary truth**. It captures natural speech from elders and layers it with AI-generated metadata, creating a "structured civilizational memory" rather than just a media archive.

---

## 🔷 Global Alignment: UNSDG Impact
* **SDG 4 (Quality Education):** Integrating indigenous knowledge into modern pedagogy.
* **SDG 10 (Reduced Inequalities):** Empowering digitally excluded tribal populations.
* **SDG 11 (Sustainable Communities):** Safeguarding intangible cultural heritage.
* **SDG 13 & 15 (Climate & Life on Land):** Preserving traditional ecological knowledge (TEK).

---

## 🔷 Core Features
* **Zero-Literacy Interface:** One-tap voice recording for elders.
* **Multilingual AI:** Speech-to-Text and auto-translation for low-resource dialects.
* **Thematic Tagging:** Automatic classification (Medicine, Farming, Rituals, Climate).
* **Ethical Consent:** Built-in workflow to ensure community data sovereignty.
* **Low-Bandwidth Optimized:** Designed for the "last mile" of rural connectivity.

---

## 🔷 Technical Architecture
* **Frontend:** React Native (Mobile App)
* **Backend:** FastAPI / Node.js
* **AI Layer:** AWS Transcribe (STT), AWS Translate, AWS Bedrock (Semantic Tagging)
* **Database:** PostgreSQL (Metadata/Consent) & AWS S3 (High-fidelity Audio)
* **Process:** `Voice Capture` → `AI Processing` → `Consent Flow` → `Searchable Index`

---

## 🔷 Why VĀṆĪ?
| Traditional Archives | VĀṆĪ Infrastructure |
| :--- | :--- |
| Text-centric | **Voice-native** |
| Static media files | **Searchable, structured datasets** |
| Extractive | **Consent-based & Community-owned** |
| Requires literacy | **Universal accessibility** |

---

## 🔷 Implementation Setup
1. **Clone:** `git clone https://github.com/org/vani.git`
2. **Backend:** `pip install -r requirements.txt` | `uvicorn main:app`
3. **Frontend:** `npm install` | `npx expo start`
4. **Env:** Configure `AWS_ACCESS_KEY` for Transcribe and S3 services.

---
**VĀṆĪ: Preserving the past to power an inclusive future.**
