# CISA_SBOM_Requirements

요약: CISA에서 제안하는 SBOM에서 필요한 필드 (7개에서 11개로 늘어났음)
정리: 완료

[2025_CISA_SBOM_Minimum_Elements.pdf](2025_CISA_SBOM_Minimum_Elements.pdf)

# 1) 보고서의 목차에 따른 내용 설명

### Introduction

- SBOM을 소프트웨어를 구성하는 “재료 목록(ingredients list)”처럼 **구성요소를 중첩(hierarchical) 구조로 나열한 인벤토리**로 정의하고, 2021년(기존 NTIA 최소요소) 이후 **도입 확대·툴 성숙**으로 인해 이번 문서가 **최소요소를 업데이트**한다고 밝힘.

### Why SBOM: Enhancing Transparency and Security

- SBOM은 소프트웨어 공급망을 보이게 만들어 조직이 **위험 기반 의사결정(risk-informed decisions)** 을 하도록 돕는다.
- 특히 SBOM 데이터는 **기계처리 가능(machine-processable)·확장 가능(scalable)** 해야 하며, 취약점 권고/승인DB 등 **다른 데이터 소스와 매핑**되어 보안 활동(취약점 관리 등)을 강화할 수 있다고 설명하고 있다.

### Why Minimum Elements: Driving Security at Scale

- SBOM의 Minimum Elements는 SBOM이 충족해야 할 **기본 기술·기본 실무 기준선(baseline)** 이며, 소프트웨어 규모가 커서 **수작업으로는 위험 평가/완화가 불가능**하므로 **기관 간 공유 가능한 공통 기준선**이 필요하다고 말하고 있다. 즉 SBOM을 생성한다면 해당 필드는 무조건 포함해야 한다.

### Scope

- 본 Minimum Elements는 기관이 획득/개발하는 소프트웨어 전반(오픈소스, AI, SaaS 포함)에 적용 가능하되, AI/SaaS 등은 추가 요소가 필요할 수 있으나 **본 문서 범위 밖**이라고 명시하고 있다.

### Minimum Elements

- 최소 요소는 3범주로 구성:
    1. **Data Fields(데이터 필드)**
    2. **Automation Support(자동화 지원)**
    3. **Practices and Processes(실무·프로세스)**
- 또한 SBOM은 대략 **계층적 구조**(컴포넌트/서브컴포넌트)이며, 데이터 필드는 **각 컴포넌트마다 적용**된다.

### 1) Data Fields (필수 필드 11개: 아래 2번에서 상세)

- 데이터 필드는 컴포넌트를 **공급망 전반에서 추적·식별**하고, 취약점/라이선스 DB 등과 **연계**하기 위한 “핵심”이라고 설명하고 있다.

### 2) Automation Support

- 대량의 SBOM 데이터를 조직 경계(공급망) 넘어 다루려면 **상호호환·자동화**가 필수이며, 대표 포맷으로 **SPDX, CycloneDX**를 언급하고 있다.
- 또한 새 소프트웨어에 대해 **폐기(deprecated)된 포맷 버전은 피해야** 하며, 지원 포맷은 주기적으로 재평가해야 한다.

### 3) Practices and Processes

- SBOM은 “데이터”만이 아니라, 조직이 SBOM을 **SDLC(개발 생명주기)에 통합**하는 실무·프로세스가 필요하다고 말한다.
- 이 범주에는 **Frequency / Coverage / Known Unknowns / Distribution and Delivery / Accommodation of Updates to SBOM Data**가 포함된다.

### Discussion

- 성숙 단계에서 추가 논의가 필요한 4개 영역으로 **SaaS/클라우드, AI, 검증(Validation), 보안 권고와의 연계**를 제시하고 잇다.
- SaaS는 잦은 변경으로 SBOM 전달 빈도가 과부하가 될 수 있어, **API 등 배포 메커니즘**과 함께 추가 탐색이 필요하다고 언급하고 있다.
- AI 시스템도 최소요소만으로는 안 잡히는 구성요소가 있을 수 있으나, **본 문서는 AI용 추가 요소를 도입하진 않는다**고 언급하고 있다.

### Validation

- SBOM은 무결성·진정성 보장을 위해 **서명(signatures)과 PKI** 같은 기존 소프트웨어 서명 인프라 활용을 언급한다.

### Correlating SBOM with Security Advisories

- VEX/CSAF 같은 보안 권고는 취약성 상태를 전달하며, SBOM과 결합하면 **취약 컴포넌트가 실제로 포함되는지 빠르게 확인**해 대응 시간을 크게 줄일 수 있다고 언급하고 있다.

### Conclusion

- SBOM 최소요소는 **새 유스케이스/기술 변화에 맞춰 진화**해야 하며, SBOM “데이터”를 분석해 “인사이트”로 바꿔 보안 의사결정에 활용해야 한다고 정리합니다. 2025_CISA_SBOM_Minimum_Elements

---

# 2) SBOM에서 요구하는 필수 필드(= Data Fields 11개)

| Data Fields 명 | 내용 |
| --- | --- |
| **SBOM Author** | 해당 컴포넌트의 SBOM 데이터를 생성한 주체  |
| **Software Producer** | 컴포넌트를 만들고 정의/식별하는(실제 제작) 주체 |
| **Component Name** | 제작자가 부여한 컴포넌트 이름  |
| **Component Version** | 해당 컴포넌트 버전을 식별하는 값(없으면 파일 생성일 등으로 대체 가능) |
| **Software Identifiers** | DB 조회키가 될 수 있는 식별자(예: CPE, purl 등) 최소 1개 이상  |
| **Component Hash** | 컴포넌트 아티팩트 해시값(+사용 알고리즘)  |
| **License** | 컴포넌트가 제공되는 라이선스(가능하면 기계처리형, proprietary 포함)  |
| **Dependency Relationship** | “포함(includes)” 관계 및 포크/백포트 등 *유래(pedigree)* 까지 표현  |
| **Tool Name** | SBOM 생성에 사용한 도구(들)과 데이터 소스 유형(예: hybrid/enriched)  |
| **Timestamp** | SBOM 데이터의 최종 갱신 시각(ISO 8601)  |
| **Generation Context** | SBOM이 생성된 생명주기 시점(빌드 전/중/후)  |

---

## 3) (기존 7개 → 11개) 추가된 4개 필드 설명

문서는 2021 대비 **새로 추가된 요소**로 **Component Hash, License, Tool Name, Generation Context**를 명시한다. (즉 “필수 데이터 필드” 관점에서 7개에서 11개로 확장된 핵심이 이 4개이다.)

### 1) Component Hash (New)

- **무엇**: 컴포넌트 아티팩트의 **해시값**과 **해시 알고리즘**을 제공(단, 원본 아티팩트 접근 불가면 해시 미기재 가능.
- **왜 중요**: 동일성/무결성 검증, 아티팩트 단위 추적 등 기존 보안·공급망 프로세스에서 해시가 중요한 역할을 하기 때문이다.

### 2) License (New)

- **무엇**: 컴포넌트 사용에 적용되는 **라이선스 정보**(가능하면 기계처리형), **proprietary 조건 존재**도 포함. 모르면 “unknown”으로 표시.
- **왜 중요**: 라이선스/재라이선스 문제는 집행(enforcement) 이슈로 이어져 **보안 지원 약속 이행**까지 영향을 줄 수 있고, 따라서 기관의 **리스크 우선순위**에 반영되어야 한다.

### 3) Tool Name (New)

- **무엇**: SBOM Author가 SBOM을 생성/보강(enrich/augment)하는 데 사용한 **도구 이름(복수 가능)** 및 도구의 데이터 소스 특성(예: hybrid/enriched).
- **왜 중요(해석)**: SBOM 데이터의 신뢰도/커버리지/생성방식(정적 분석 vs 빌드 파이프라인 기반 등)을 판단할 때 “어떤 도구로 만들었는지”가 핵심 단서가 되기 때문이다(문서도 *생성 방식에 대한 정보 제공*을 추가 이유로 들고 잇다.).

### 4) Generation Context (New)

- **무엇**: SBOM이 **빌드 전(pre-build)** / **빌드 중(during build)** / **빌드 후(post-build)** 중 **어느 시점의 데이터**로 생성되었는지.
- **왜 중요**: 시점에 따라 관측 가능한 데이터가 달라 SBOM 내용도 달라지며(예: 빌드 산출물 기반 vs 소스/레포 기반), 이를 알면 수신자가 **리스크 평가에 필요한 맥락**을 더 정확히 잡을 수 있다.(개발 주기에 따라 프로젝트의 구조가 상이하기 때문이다.)