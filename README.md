# 연구(프로젝트) 진행

날짜: 2025년 12월 30일 → 2026년 2월 27일
상태: 세미나 완료
설명: 연구 진행 방식 및 상황 요약

<br>

# 인턴 연구 진행


## (1) CISA Minimal Elements (11개의 field 분석)

- 아래의 11개의 항목을 포함하고 있는 지를 중점적으로 조사한다.
    - NTIA에서 제공한 최소 필드가 7개였는데 4개가 추가된 것을 확인할 수 있다.

| **Data Fields 명** | 내용 |
| --- | --- |
| **SBOM Author** | 해당 컴포넌트의 SBOM 데이터를 생성한 주체 (작성자) |
| **Tool Name** | SBOM 생성에 사용한 도구(들)과 데이터 소스 유형(예: hybrid/enriched) |
| **Timestamp** | SBOM 데이터의 최종 갱신 시각(ISO 8601) |
| **Generation Context** | SBOM이 생성된 생명주기 시점(빌드 전/중/후) |
| **Dependency Relationship** | “포함(includes)” 관계 및 포크/백포트 등 *유래(pedigree)* 까지 표현 |
| **Software Producer** | 컴포넌트를 만들고 정의/식별하는(실제 제작) 주체 |
| **Component Name** | 제작자가 부여한 컴포넌트 이름 |
| **Component Version** | 해당 컴포넌트 버전을 식별하는 값(없으면 파일 생성일 등으로 대체 가능) |
| **Software Identifiers** | DB 조회키가 될 수 있는 식별자(예: CPE, purl 등) 최소 1개 이상 |
| **Component Hash** | 컴포넌트 아티팩트 해시값(+사용 알고리즘) |
| **License** | 컴포넌트가 제공되는 라이선스(가능하면 기계처리형, proprietary 포함) |

---
<br>

### 새로운 필드 4개

### 1) Tool Name (New)

- **무엇**: SBOM Author가 SBOM을 생성/보강(enrich/augment)하는 데 사용한 **도구 이름(복수 가능)** 및 도구의 데이터 소스 특성(예: hybrid/enriched).
- **왜 중요(해석)**: SBOM 데이터의 신뢰도/커버리지/생성방식(정적 분석 vs 빌드 파이프라인 기반 등)을 판단할 때 “어떤 도구로 만들었는지”가 핵심 단서가 되기 때문이다(문서도 *생성 방식에 대한 정보 제공*을 추가 이유로 들고 잇다.).

### 2) Generation Context (New)

- **무엇**: SBOM이 **빌드 전(pre-build)** / **빌드 중(during build)** / **빌드 후(post-build)** 중 **어느 시점의 데이터**로 생성되었는지.
- **왜 중요**: 시점에 따라 관측 가능한 데이터가 달라 SBOM 내용도 달라지며(예: 빌드 산출물 기반 vs 소스/레포 기반), 이를 알면 수신자가 **리스크 평가에 필요한 맥락**을 더 정확히 잡을 수 있다.(개발 주기에 따라 프로젝트의 구조가 상이하기 때문이다.)

### **3) Component Hash (New)**

- **무엇**: 컴포넌트 아티팩트의 **해시값**과 **해시 알고리즘**을 제공(단, 원본 아티팩트 접근 불가면 해시 미기재 가능.
- **왜 중요**: 동일성/무결성 검증, 아티팩트 단위 추적 등 기존 보안·공급망 프로세스에서 해시가 중요한 역할을 하기 때문이다.

### 4) License (New)

- **무엇**: 컴포넌트 사용에 적용되는 **라이선스 정보**(가능하면 기계처리형), **proprietary 조건 존재**도 포함. 모르면 “unknown”으로 표시.
- **왜 중요**: 라이선스/재라이선스 문제는 집행(enforcement) 이슈로 이어져 **보안 지원 약속 이행**까지 영향을 줄 수 있고, 따라서 기관의 **리스크 우선순위**에 반영되어야 한다.

<br>

## (2) SBOM 생성

- SBOM을 프로그래밍 언어를 기준으로 하여 출력을 진행한다. 실험에 사용되는 언어는 CPP, Java, Python, Go로 네 가지 언어이다.
- 각 언어 별 주요 프로젝트는 다음과 같다.

[프로젝트](%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%202e661b25f58980d4a4fadffc3252e19c.csv)

- 프로젝트 별로 SBOM을 생성하는 데에 사용하는 generator은 다음 네 가지이다. (생성 내용 및 분석은 위의 프로젝트 DB에서 진행)
    - **Hatbom(iotcube 2.0)**
    - **Syft**
    - **Trivy**
    - **cdxgen(CycloneDX)** → Hugo 프로젝트에 대하여 cdxgen의 SBOM 생성 과정 중 문제가 발생하여 cdxgen-gomod를 사용하였기에 해당 내용은 추후 폐기 예정

- **실험 시 발생했던 문제점**
    1. Hatbom의 경우 원본 소스 파일의 ZIP 파일을 올린 것과 Hmark를 통한 hash ZIP 파일을 올린 것과의 차이점이 존재하였다. → 해당 실험에선 Hatbom을 이용한 SBOM을 생성할 때 hmark를 통해 나온 hash zip 파일을 이용하여 생성한 것을 전제로 한다.
    2. cyclonedx의 경우 go 프로젝트인 Hugo의 SBOM을 생성할 때, @cyclonedx/cdxgen 을 이용한 경우 root 권한 문제, list 파일 인식 문제 등 여러 에러가 발생하였다. 따라서 go proejct의 경우 cyclonedx-gomod를 따로 다운로드 받아서 실행하였다. (cyclonedx/cdxgen 통합 모듈의 정합성이 의심되는 부분이다.)
    
<br>    

## (3) SBOM generator 원리 이해하기

1. Github에 공개된 SBOM generator의 코드를 직접 읽고 분석해본다.
2. parser의 기초가 되는 regex 정규 표현식의 패턴 및 문법을 학습한다.
3. 국제 표준 자원 IRI의 특성을 파악한다. (링크: https://www.ibm.com/docs/ko/cics-ts/5.5.0?topic=cics-internationalized-resource-identifiers-iris)
4. 코드를 분석하면서 SBOM generator의 parser 부분을 확인하여 해당 generator가 프로젝트 파일을 어떻게 인식하는 지 파악한다.

[Code Analysis](Code%20Analysis%202e861b25f58980c4b5c6e2fcd7cc57a5.csv)

### 실험의 중간 결과

- cdxgen의 python 코드 쪽 parser을 확인했을 때, poetry 패키지 관리자 명세서(toml) 부분에서 파싱 에러가 나는 것을 확인할 수 있었다.
    - cdxgen의 dependencies와 metadata를 생성하는 부분에선 패키지 명세서를 읽으면서 상위 pkg 필드를 선언하고 상위 객체의 하위 필드를 채워 넣는 식으로 초기화를 진행하는 것을 확인할 수 있었다.
    - 어떤 프로젝트가 11개의 외부 의존성을 끌어다 사용했다면, 패키지 명세서에는 11개에 대한 패키지의 명세서가 적혀있는 것을 볼 수 있다. 이를 index = 0 ~ index = 10까지 1차적으로 loop를 돌아 상위 pkg 객체를 11개를 생성한다. 만약 index = 0이 뒷 부분의 index의 패키지를 참조하고 있다면 뒷 부분에 대한 자세한 내용은 아직 기록되지 않은 상태이므로 set에 단순히 패키지의 이름을 넣고 2차 검증 loop에서 정보를 다시 기입하게 된다.
        - 버그는 2차 loop를 돌 때 발생하였는데, 1차 loop 시 패키지 명명을 lower case로 변환하여 넣었지만 2차 검증 loop에선 toml 파일에서 읽은 값을 직접 비교하였기 때문에 대소문자 구별로 인한 패키지 누락이 발생한 것을 확인할 수 있었다.
- syft도 cdxgen과 동일하게 dj-rest-auth의 의존성 부분에 Django 패키지를 인식하지 못한 모습이었다.
    - syft는 반면에 package.extras를 직접 찾았다는 점이 다르다. cdxgen에 비해 syft가 모두 dependencies field의 elements가 많았는데, 이러한 이유가 원인으로 파악된다.

- **부록: poetry.lock 파일의 dependencies.extras**
    
    https://stackoverflow.com/questions/60971502/python-poetry-how-to-install-optional-dependencies
    
    - 해당 항목은 스크립트에 -E 옵션을 넣었을 때 추가로 설치되는 의존성 패키지를 명시한다.
    - Optional Package로 보면 된다.


## (4) SBOM generator의 parsing 방식의 한계 분석

- package 명세서의 대소문자 처리 이슈
    - django와 Django를 동일한 패키지로 인식해야 하는가?
        - version 명세가 약간 다르기에 다른 패키지로 인식한다면 패키지 내용을 추출 시에 임시로 hash된 값을 저장하고 SBOM 생성 시에 이를 decode하는 방식으로 하는 것이 맞는가?
- package의 extras(조건부로 추가되는 패키지)는 어떻게 처리해야 하는가?
    - syft는 extras를 조건에 상관 없이 dependencies에 추가하는 것을 확인했다.
- trivy의 경우 Django를 패키지로 인식하는 것을 볼 수 있었다.


## (5) 언어 마다 사용하는 프레임워크의 명세서 양식 비교

- 목적: poetry 패키지 매니저가 사용하는 명세서 파일인 poetry.lock의 내용을 통해 외부 의존성을 탐지하는 방식이 바뀐다는 것을 인지했음. 조건부로 추가되는 패키지와 같은 외부 의존성에 추가해야 하는 지에 대한 논란이 있을만한 패키지 처리를 위해 기존 외부 의존성 명세서의 작성 방식에 대해 알아본다.


## (6) cdxgen, syft, trivy의 실행시간, 컴포넌트 커버리지, 정확도 측정

- **실행 시간**: docker run을 했을 때 기준 최종 시간으로 판단
- **커버리지**: 외부 의존성 기준으로 SBOM의 components가 얼마나 패키지를 잘 인식했는지 판단
- **정확도**: TP(진양성), FP(위양성), FP(위음성)를 측정
    - **ground truth 기준**
        - 패키지 관리 명세서
        - setup.py와 같은 패키지 사전 세팅 스크립트
    - **TP**: ground truth에 있으며 SBOM의 component(s)에 있는 경우
    - FP: ground truth에 없지만 SBOM의 component(s)에 있는 경우
    - **FN**: ground truth에 있지만 SBOM의 component(s)에 없는 경우
- 테스트 내용: https://github.com/brian-223134/Projects-s-SBOM-Results/tree/main/code/analyze

[테스트 결과](%ED%85%8C%EC%8A%A4%ED%8A%B8%20%EA%B2%B0%EA%B3%BC%202f561b25f58980728cc4c1480eb65fc5.csv)

- Syft의 경우 이전에 Bug Issue했던 내용을 피드백하여 적용한 것을 볼 수 있었다.
https://github.com/anchore/syft/releases/tag/v1.41.1
    
    ```go
    // 디렉토리 syft/pkg/cataloger/python/dependency.go
    // 디렉토리를 통해 다른 패키지 관리자 형태에서도 대문자가 있었다면
    // dependencies 누락이 되었을 것이라는 확인할 수 있다.
    func packageRef(name, extra string) string {
    	// cleanExtra := strings.TrimSpace(extra)
    	// cleanName := strings.TrimSpace(name)
    	// normalize both package name and extra to ensure case-insensitive matching per Python packaging spec
    	// https://packaging.python.org/en/latest/specifications/name-normalization/
    	cleanName := normalize(strings.TrimSpace(name))
    	cleanExtra := normalize(strings.TrimSpace(extra))
    	
    	// 이전에는 package의 Ref를 가져올때 trim을 하고 normalize를 하지 않은 것을 확인할 수 있었다.
    	// normalize는 '대문자 -> 소문자', '_,/,\ -> -'로 바꾸는 기능을 수행한다.
    ```
    

---

## (8) 통합 SBOM 생성 프로그램 (prototype)

- plantuml을 이용하여 work flow 정리
- prototype을 생성한 이후 피드백 받기

### 1) 코드의 흐름

- 사용자는 Hatbom, Syft로 생성한 CycloneDX 형식의 SBOM을 준비한다.
- 서버(프로그램)은 두 개의 SBOM을 입력 받고 다음의 process를 수행한다.
    1. 서버는 Hatbom, Syft의 필드를 모두 파싱하여 임시 저장한다.
    2. components, dependencies를 제외한 UUID, 생성자, Tool property는 Hatbom, Syft를 섞어서 값을 생성한다.
    3. components에 대하여 Hatbom은 내부 의존성을 따르고, Syft는 외부 의존성을 따르니 두 개의 내용을 섞는다. 다만 섞을 때, 각각 어느 도구에서 비롯된 것을 판별할 수 있도록 metadata 필드를 추가로 생성하여 생성한다.
    4. 최종 출력은 Syft 형식의 SBOM으로 wrapping이 되어 있지 않은 형태를 따른다.
    5. JSON으로 합친 결과물과 결과물에 대한 요약을 출력한다.

### 2) Prototype 소스 코드

[GitHub - brian-223134/Unified_SBOM: Prototype of Integrating syft and hatbom formatted SBOM in one unified SBOM. (Korea University CCS Lab)](https://github.com/brian-223134/Unified_SBOM)
