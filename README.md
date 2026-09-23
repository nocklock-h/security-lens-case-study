# Security Lens

> AI-assisted privacy risk analyzer for images and documents.

이미지 속 개인정보와 EXIF 메타데이터를 분석해  
업로드 전에 개인정보 노출 위험을 확인할 수 있도록 만든 Spring Boot 기반 MVP입니다.

**Java 17 · Spring Boot · OCR · EXIF · Local LLM · Railway**

> Source code is kept private.  
> This repository documents the product, architecture, implementation decisions, and lessons learned.

---

## Problem

사진이나 문서를 온라인에 업로드할 때 사용자가 눈으로 확인하기 어려운 개인정보가 함께 노출될 수 있습니다.

예를 들어 이미지 안에는 다음 정보가 포함될 수 있습니다.

- 전화번호
- 이메일 주소
- 이름 등 텍스트 개인정보
- GPS 위치
- 촬영 시간
- 기기 정보
- 기타 EXIF metadata

특히 EXIF 정보는 이미지 화면만 보고는 확인하기 어렵습니다.

Security Lens는 업로드 전에 이런 정보를 한 번에 분석하고  
사용자가 **“이 파일을 그대로 공유해도 되는가?”**를 판단할 수 있도록 만드는 것을 목표로 했습니다.

---

## What I Built

Security Lens는 이미지를 업로드하면 여러 분석 단계를 거쳐 개인정보 노출 위험을 보여줍니다.

### Core flow

```text
Image Upload
      ↓
OCR Text Extraction
      ↓
Personal Information Detection
      ↓
EXIF Metadata Analysis
      ↓
Risk Score Calculation
      ↓
Local LLM Analysis
      ↓
Risk Level + Explanation
```

---

## Key Features

### 1. OCR-based personal information detection

이미지에 포함된 텍스트를 OCR로 추출한 뒤 개인정보 패턴을 분석합니다.

현재 주요 탐지 대상:

- Phone number
- Email address
- Name-like information
- Sensitive text patterns

---

### 2. EXIF metadata analysis

이미지 파일에 포함된 metadata를 분석합니다.

주요 분석 정보:

- GPS coordinates
- Capture date and time
- Device / camera information
- Other available EXIF metadata

화면에서 보이지 않는 정보까지 분석 대상으로 포함했습니다.

---

### 3. Privacy Risk Score

각 탐지 결과를 단순히 나열하는 대신  
사용자가 빠르게 위험도를 이해할 수 있도록 Risk Score와 Risk Level을 설계했습니다.

```text
Detected privacy signals
        ↓
Weighted risk calculation
        ↓
Risk Score
        ↓
Risk Level
```

목표는 완벽한 보안 판정이 아니라,

> 사용자가 추가 확인이 필요한 파일을 빠르게 식별하는 것

입니다.

---

### 4. Local LLM analysis

OCR / EXIF / rule-based 결과를 기반으로  
Local LLM이 추가적인 위험 설명을 생성하도록 구성했습니다.

사용 모델:

- Ollama
- Gemma 3

Local LLM을 선택한 이유는 개인정보 분석 서비스 특성상  
가능한 한 민감한 데이터를 외부 AI API로 보내지 않는 구조를 실험하고 싶었기 때문입니다.

---

## Architecture

```text
Browser
   ↓
Spring Boot API
   ├── OCR Analyzer
   ├── Personal Info Detector
   ├── EXIF Analyzer
   ├── Risk Scoring
   └── Local LLM Analyzer
             ↓
          Ollama
             ↓
          Gemma 3
```

Application deployment:

```text
Client
  ↓
Railway
  ↓
Spring Boot Application
```

---

## Tech Stack

| Area | Technology |
|---|---|
| Backend | Java 17, Spring Boot |
| Image analysis | OCR |
| Metadata | EXIF / metadata-extractor |
| AI | Ollama, Gemma 3 |
| Frontend | HTML, CSS, JavaScript |
| Deployment | Railway |
| Build | Gradle |
| Version Control | Git / GitHub |

---

## Engineering Decisions

### Why Spring Boot?

기존 Java/Spring 개발 경험을 활용하면서  
이미지 분석, API, AI integration을 하나의 서비스 흐름으로 구현하기 위해 선택했습니다.

---

### Why Local LLM?

Security Lens가 다루는 데이터 자체가 개인정보일 가능성이 있기 때문에  
외부 API 호출만 사용하는 구조보다 로컬 추론 구조도 직접 실험해보고 싶었습니다.

다만 현재 MVP에서는 성능과 운영 환경의 한계도 존재합니다.

---

### Why combine rules with LLM analysis?

전화번호나 이메일처럼 명확한 패턴은 deterministic rule이 더 안정적입니다.

따라서 LLM 하나에 모든 판단을 맡기기보다:

```text
Rules / OCR / Metadata
          +
        LLM
```

형태로 역할을 나누었습니다.

LLM은 최종 판정자가 아니라 **추가 분석과 설명을 제공하는 보조 계층**으로 사용했습니다.

---

## Challenges & Solutions

### Local LLM performance

초기에는 GPU / CUDA 환경 문제를 경험했습니다.

개발 과정에서 CPU 실행으로 전환해  
기능 검증을 우선 진행하고 MVP를 완성했습니다.

이 경험을 통해 모델 성능뿐 아니라  
실제 실행 환경과 배포 제약도 제품 설계의 일부라는 점을 배웠습니다.

---

### Port conflicts

Spring Boot와 Ollama를 함께 실행하면서  
8080 / 11434 포트 충돌 및 환경 설정 문제를 해결했습니다.

이를 통해 애플리케이션 코드 외에도  
실행 환경을 재현하고 문제를 추적하는 과정의 중요성을 경험했습니다.

---

### Risk scoring

단순히 “개인정보가 발견되었습니다”라고 표시하는 것보다  
사용자에게 어느 정도 주의가 필요한지를 표현할 방법이 필요했습니다.

그래서 각 탐지 결과를 하나의 Risk Score / Risk Level로 통합하는 구조를 설계했습니다.

---

## What I Learned

Security Lens를 만들면서 가장 크게 배운 점은  
AI 기능 하나를 추가하는 것과 실제 제품 흐름에 AI를 넣는 것은 다르다는 점이었습니다.

특히 다음을 경험했습니다.

- 기존 backend와 AI inference 연결
- OCR 결과의 불확실성 처리
- rule-based detection과 LLM 역할 분리
- 개인정보를 다루는 제품의 데이터 흐름 고민
- 로컬 모델의 실행 환경 제약
- MVP를 실제 URL까지 배포하는 과정

또한 완벽한 기능을 기다리기보다

> **작동하는 MVP를 먼저 만들고 실제 문제를 확인한 뒤 개선한다**

는 개발 방식을 직접 경험했습니다.

---

## Next

현재 MVP 이후 개선하고 싶은 기능입니다.

- OCR accuracy improvements
- Bounding box visualization
- Automatic masking / redaction
- QR / barcode detection
- EXIF removal
- Better risk scoring
- Automated tests
- CI/CD
- AWS deployment architecture

---

## Project Status

Security Lens was developed as an MVP for the **Wanted AI Championship 2026**.

The original implementation repository remains private.

This public repository is maintained as a technical case study for the project.

---

## About Nocklock

I build and document small products around:

- Java / Spring Boot
- AI-assisted development
- privacy & security tooling
- deployment
- developer automation

- [Nocklock Blog](https://nocklock.tistory.com/)
- [GitHub Profile](https://github.com/nocklock-h)

---

Built by **Nocklock**

> From idea → implementation → deployment → iteration.
