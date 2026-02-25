# Software Bill of Materials (SBOM) Harmonization Plugfest 2024

연도: 2024
저자: David Tobar, Jessie Jamieson, Mark Priest, Sasank Vishnubhatla
출판: Special Report, TOP TIER

[SBOM_Harmonization_Plugfest_2024-1.pdf](SBOM_Harmonization_Plugfest_2024-1.pdf)

[3번주제 - plugfest 요약.pdf](3%EB%B2%88%EC%A3%BC%EC%A0%9C_-_plugfest_%EC%9A%94%EC%95%BD.pdf)

---

# 📄 보고서: SBOM Harmonization Plugfest 2024

> **개요**: 이 보고서는 Software Engineering Institute(SEI)가 CISA의 지원을 받아 수행한 **SBOM Harmonization Plugfest 2024**의 연구 결과와 권고 사항을 담고 있다.
> 

---

## 1. 서론 (Introduction)

### 🎯 연구 배경 및 목적

- **배경**: 동일한 소프트웨어에 대해 서로 다른 SBOM 도구들이 상이한 결과를 생성하는 문제가 빈번하게 발생하며, 이는 SBOM에 대한 사용자 신뢰를 저해하는 요인이 된다.
- **목표**: SBOM 생성 결과의 차이를 유발하는 근본 원인을 파악하고, 예측 가능하고 고품질의 SBOM 생성을 위한 가이드를 제시하는 것을 목표로 한다.

### 📌 과제 (Task)

- **Plugfest 개요**: 2024년 11월부터 2025년 3월까지 진행되었으며, 21개 참여 기관으로부터 총 243개의 SBOM을 제출받아 분석했다.
- **접근 방식**: 경쟁 방식이 아닌, 다양한 도구와 참여자가 생성한 SBOM의 **다양성(Variance)**과 **공통성(Commonality)**을 평가하는 데 중점을 두었다.

---

## 2. Plugfest 진행 과정 (The Plugfest Process)

### 🛠️ 대상 소프트웨어 (Software Targets)

다양한 언어와 생태계를 대변하기 위해 총 9개의 타겟 소프트웨어를 선정했다,

- **JavaScript**: NodeJS-goof
- **Python**: HTTPie
- **Java**: Mine Colonies, Dependency Track
- **C++**: OpenCV
- **Go**: Gin
- **Rust**: Hexyl
- **PHP**: PHPMailer
- **C**: jq

### 🔍 분석 방법론 (Methodology)

- **자동화 및 수동 분석**: Python과 Jupyter Notebook을 활용해 데이터를 추출하고, 전문가 검토를 병행
- **기준 SBOM (Baseline SBOMs)**: 비교 분석을 위해 오픈 소스 도구인 **Syft**, **Trivy**, **Microsoft SBOM Tool**을 사용하여 기준이 되는 SBOM을 생성했다.

### 📏 평가 기준 (Evaluation Criteria)

- **완결성(Completeness)**: 모든 컴포넌트 정보 포함 여부
- **정확성(Accuracy)**: 정보의 사실 여부
- **최소 요건(Minimum Elements)**: NTIA 권고 사항(작성자, 타임스탬프, 해시 등) 준수 여부

---

## 3. 제출된 SBOM 요약 (Summary of SBOM Submissions)

### 📊 제출 현황

- 총 21개 참여자가 243개의 SBOM을 제출했다.
- 대다수(186개)는 **JSON** 형식이었으며, **CycloneDX** 형식이 가장 많았고 **SPDX**가 그 뒤를 이었다.
- **XML** 및 **YML** 형식은 소수만 제출했다.

---

## 4. SBOM 깊이 분석 (SBOM Depth Analysis)

### 🌳 깊이(Depth)와 너비(Breadth)

- **깊이(Depth)**: 의존성 트리에서 가장 긴 경로의 길이로, 소프트웨어 구성 요소의 투명성 수준을 나타낸다.
- **너비(Breadth)**: 타겟 소프트웨어로부터 특정 거리(Distance)에 있는 컴포넌트의 최대 개수이다.

### 📉 분석 결과

- 제출된 SBOM들 사이에서 깊이의 편차가 컸으며, 일부는 전이적 의존성(Transitive Dependencies) 정보를 포함하지 않아 깊이가 매우 얕았다.
- 이는 참여자들이 SBOM이 제공해야 할 **투명성의 수준**에 대해 서로 다른 기대를 하고 있음을 시사한다.

---

## 5. 주요 연구 결과 (Findings)

### ❗ 주요 발견 사항

1. **컴포넌트 수의 불일치**: 동일한 소프트웨어임에도 식별된 컴포넌트 수에 큰 차이가 있었습니다.
    - *예시*: NodeJS-goof의 경우, 어떤 SBOM은 496개, 다른 SBOM은 1,081개의 컴포넌트를 식별했다.
2. **정규화(Normalization) 부족**: 소프트웨어 버전 표기(예: "v 2.0" vs "2.0")나 이름 표기 방식의 불일치가 데이터 차이의 주요 원인이었다.
3. **의존성(Dependency) 정의의 모호함**: 일부는 빌드 도구, 테스트 도구, 문서화 도구까지 의존성으로 포함시킨 반면, 일부는 런타임 의존성만 포함했다.
4. **최소 요건 준수 미흡 (CISA에서 제공하는 기준)**:
    - **암호화 해시(Cryptographic Hash)**: 포함률이 **22%**에 불과했다.
    - **라이선스(License)**: 포함률이 **6%**로 매우 낮았다.
    - **작성자(Author)**: 포함률이 **35%**로 나타났다.

---

## 6. 권고 사항 (Recommendations)

### ✅ SBOM 최소 요건 개선 (Minimum Elements)

- **SBOM 유형(Type)**: SBOM이 생성된 생명주기 단계(Source, Build 등)를 **필수**로 명시해야 한다.
- **버전 문자열(Version String)**: 정확한 버전 표기가 중요하며, 유연성을 위해 시맨틱 버저닝(Semantic Versioning)을 따를 것을 권장한다.
    - **Semantic Versioning의 설명**: Major.Minor.Patch(주.부.수)의 형태로 존재
        - **MAJOR (주 버전)**
        • **의미**: 기존 버전과 **호환되지 않는** API 변경이 발생했을 때 올린다.
        • **예시**: 함수가 삭제되거나, 필수 파라미터가 변경되어 기존 코드가 더 이상 동작하지 않을 때 ($1.0.0 \rightarrow 2.0.0$).
        - **MINOR (부 버전)**
        • **의미**: 기존 버전과 **호환되는** 방식으로 새로운 기능이 추가되었을 때 올린다.
        • **예시**: 새로운 함수가 추가되었지만, 기존 코드는 문제없이 돌아갈 때 ($1.1.0 \rightarrow 1.2.0$).
        • *참고*: MINOR 버전이 올라가면 PATCH 버전은 0으로 초기화된다.
        - **PATCH (수 버전)**
        • **의미**: 기존 버전과 **호환되는** 버그 수정이 이루어졌을 때 올린다.
        • **예시**: 잘못된 동작을 고쳤지만 API 변경은 없을 때 ($1.1.1 \rightarrow 1.1.2$).
- **공급자 명(Supplier Name)**: 오픈 소스의 경우, 공급자 이름 대신 프로젝트 저장소 링크를 제공해야 한다.
- **암호화 해시**: 해시값이 어떤 대상(소스 파일, 바이너리 등)을 기준으로 생성되었는지 명시해야 한다.

### 🤝 SBOM 조화 (Harmonization)

- **정규화**: 컴포넌트 이름과 버전에 대한 표준화된 가이드가 필요하다.
- **용어 표준화**: 1차 공급자는 `Supplier`, 2차 제조자는 `Manufacturer`로 용어를 통일한다.
- **포함 이유 명시(Annotation)**: 컴포넌트가 왜 포함되었는지(예: 빌드 매니페스트에 포함됨, 컴파일러가 처리함 등)를 주석으로 명시해야 한다.
- **개발자 도구 지원**: 언어 및 빌드 프레임워크 자체에 SBOM 생성 기능을 내장하여 상류(Upstream)에서부터 정확한 정보가 생성되도록 지원해야 한다. (SBOM 기능이 내장 되면 해당 프레임워크를 사용하는 다른 프로젝트의 경우 동일한 알고리즘으로 SBOM이 생성되기 때문에 SBOM 간의 규격 통일성이 생김)

---

## 7. 향후 연구 과제 (Future Research)

- **취약점 분석(Vulnerability Analysis)**: SBOM이 취약점 관리 기능을 얼마나 잘 지원하는지, NVD(국가 취약점 데이터베이스)와 어떻게 연계되는지 연구가 필요하다
- **구조적 분석**: 의존성 트리를 비교하여 SBOM 간의 구조적 유사점과 차이점을 심층 분석해야 한다.
- **거짓 양성/음성(False Positives/Negatives)**: 실제 사용되지 않는 의존성이 보고되거나, 필요한 의존성이 누락되는 원인에 대한 근본적인 분석이 필요하다.
- **동적 분석(Dynamic Analysis)**: 런타임에 결정되는 의존성이 SBOM에 미치는 영향을 연구해야 한다.