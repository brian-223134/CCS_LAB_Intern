# Transformers

Github URL: https://github.com/huggingface/transformers

# SBOM 출력 결과 분석

## 1. Hatbom (hmark hash)

- 출력 결과
    
    [transformers_hmark_SBOM_hatbom.json](transformers_hmark_SBOM_hatbom.json)
    
- 출력 내용 분석
    - 다른 SBOM generator과 다르게 JSON의 최상위 필드에 실제 sbom 내용인 ‘sbom’ 필드와 발견된 컴포넌트 개수인 ‘file_count’ 필드가 있다. (file_count의 값은 components 필드의 list elements 개수이다.)
    - **metadata 필드**: 중요한 내용은 component에 있다.
    
    ```json
    "metadata": {
          "timestamp": "2026-01-09T04:02:22.861685+00:00",
          "authors": [
            {
              "name": "IoTcube - https://iotcube.net"
            }
          ],
          "component": {
            "group": "",
            "name": "transformers",
            "version": "",
            "type": "application",
            "bom-ref": "pkg:generic/transformers",
            "purl": "pkg:generic/transformers"
          }
        }
    ```
    
    - group: 네임스페이스가 들어가야 하지만 출력 결과 비어 있는 것을 확인
    - name: root 패키지의 이름으로 주로 프로젝트의 명칭이 들어감
    - version: 해당 컴포넌트의 버전이다. (비어있는 것을 확인)
    - type: 컴포넌트의 성격이다. application으로 나와있으나 library 같은 것이 더 자연스럽다고 평가할 수 있다.
    - bom-ref: BOM 문서 내부에서 해당 컴포넌트를 참조하기 위한 ‘유일한 참조키’
    - purl: package를 URL로 표시하여 식별하는 것(bom-ref와 동일하게 구성)

- **dependencies** 필드

```json
"dependencies": [
      {
        "ref": "transformers",
        "dependsOn": [
          "transformers v4.44.0"
        ]
      }
    ]
```

- ref: “의존하는 주체(부모 노드)” 컴포넌트를 가리키는 참조값. transformers라는 github project directory가 root directory인데 이를 최상위 부모 컴포넌트로 인식하는 것을 알 수 있음.
- dependsOn: ref가 의존하는 자식 컴포넌트 들이다. ‘transformers v4.44.0’을 사용하고 있는 것을 볼 수 있는데, 해당 내용이 ref로 올라와야 할 것 같다.

- components 필드

```json
{
        "group": "transformers",
        "name": "optimum_benchmark_wrapper",
        "version": "0.0.0-aaf60cce010e",
        "type": "file",
        "bom-ref": "pkg:generic/transformers/optimum_benchmark_wrapper@0.0.0-aaf60cce010e#benchmark/optimum_benchmark_wrapper.py",
        "purl": "pkg:generic/transformers/optimum_benchmark_wrapper@0.0.0-aaf60cce010e#benchmark/optimum_benchmark_wrapper.py",
        "hashes": [
          {
            "alg": "MD5",
            "content": "aaf60cce010ecac1d2bf843bba568768"
          }
        ]
      },
```

- group: 상위 논리 그룹의 이름으로 transformers project가 해당 원소의 상위 그룹임을 알 수 있다.
- name: 해당 의존성의 타입이 file이기 때문에 해당 name 필드는 파일 이름이 된다.
    - 아래는 type이 파일이 아닌 경우 주로 나타나는 name의 형태
    - **패키지/모듈 이름**: `requests`, `numpy`, `transformers`
    - **아티팩트 이름**(Maven 등): `log4j-core`
    - **컨테이너/이미지 이름**: `nginx`, `my-service`
    - **내부 서비스/앱 이름**: `billing-api` 같은 식
    - PyPI: `name=transformers` / 버전은 `version=4.44.0`
    - npm: `name=react`
    - Maven: `group`(네임스페이스) + `name`(artifactId) 조합이 자주 쓰임
- version: 0.0.0-hash 값의 형태를 가지는데 file의 경우 특정 버전을 가지지 않으므로 고유성을 위한 hash 값을 같이 넣어서 표현
- bom-ref: 유일한 참조키이다. 해당 필드를 분석해보면 아래의 형식을 지니고 있다.

> { component의 bom-ref}/{ 해당 elements의 name }@{ 해당 elements의 version }#{ 해당 file의sub directory }
> 

---

## 2. syft

- 출력 결과
    
    [transformers_SBOM.json](transformers_SBOM.json)
    
- $schema: syft가 어떠한 출력 규범을 따르고 있는 지를 나타낸다.  아래를 보면 해당 syft가 cyclonedx의 1.6 버전을 따르는 것을 알 수 있다.

> "[http://cyclonedx.org/schema/bom-1.6.schema.json](http://cyclonedx.org/schema/bom-1.6.schema.json)"
> 
- metadata 필드: tools와 component 출력 결과가 나왔다.

```json
"metadata": {
    "timestamp": "2026-01-09T05:24:01Z",
    "tools": {
      "components": [
        {
          "type": "application",
          "author": "anchore",
          "name": "syft",
          "version": "1.39.0"
        }
      ]
    },
    "component": {
      "bom-ref": "dfcd457ff0b35452",
      "type": "file",
      "name": "/src"
    }
  }
```

- **tools**: 해당 SBOM을 생성한 주체이다. Hatbom의 경우 단순하게 authors 필드 만을 작성하였는데 이는 저자의 느낌이 강하다. 조금 더 완벽하게 생성 주체를 나타내고 싶다면 tools 필드를 따로 빼는 것이 좋다고 본다.
- **component:** 해당 프로젝트의 최상위 패키지이다. Hatbom과 다르게 bom-ref를 hash 처리한 것을 볼 수 있다. type의 경우  file로 나와있기 때문에 Hatbom과 마찬가지로 name에는 directory가 나와있는 것을 볼 수 있다. 다른 점이라면 Hatbom은 최상위 프로젝트 디렉토리를 나타냈으며 syft는 해당 root 디렉토리를 src라는 이름으로 alias를 한 것을 볼 수 있다. 또한 syft는 Hatbom과 다르게 값이 채워지지 않는 필드의 경우 따로 표기하지 않은 것을 확인할 수 있다.
- syft 출력물을 보면 dependencies 필드가 없고 바로 components가 출력 되는 것을 볼 수 있다. 이는 의존관계를 확인할 때 불편할 수 있다.

```json
{
      "bom-ref": "dbed8a6341fe7748",
      "type": "library",
      "name": "./.github/workflows/benchmark_v2.yml",
      "version": "UNKNOWN",
      "cpe": "cpe:2.3:a:.\\/.github\\/workflows\\/benchmark-v2.yml:.\\/.github\\/workflows\\/benchmark-v2.yml:*:*:*:*:*:*:*:*",
      "properties": [
        {
          "name": "syft:package:foundBy",
          "value": "github-action-workflow-usage-cataloger"
        },
        {
          "name": "syft:package:type",
          "value": "github-action-workflow"
        },
        {
          "name": "syft:package:metadataType",
          "value": "github-actions-use-statement"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:.\\/.github\\/workflows\\/benchmark-v2.yml:.\\/.github\\/workflows\\/benchmark_v2.yml:*:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:.\\/.github\\/workflows\\/benchmark_v2.yml:.\\/.github\\/workflows\\/benchmark-v2.yml:*:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:.\\/.github\\/workflows\\/benchmark_v2.yml:.\\/.github\\/workflows\\/benchmark_v2.yml:*:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:.\\/.github\\/workflows\\/benchmark:.\\/.github\\/workflows\\/benchmark-v2.yml:*:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:.\\/.github\\/workflows\\/benchmark:.\\/.github\\/workflows\\/benchmark_v2.yml:*:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:location:0:path",
          "value": "/.github/workflows/benchmark_v2_a10_caller.yml"
        }
      ]
    },
```

- Hatbom과 다르게 하나의 패키지가 의존하고 있는 다른 패키지를 하나의 components element로 묶은 것을 볼 수 있다. purl이 없는 것처럼 보이지만 일부는 없고 일부 components의 element에는 아래와 같이 purl도 있는 것을 볼 수 있다.

```json
{
      "bom-ref": "pkg:pypi/git-python@1.0.3?package-id=36c778a7cb79dad4",
      "type": "library",
      "name": "git-python",
      "version": "1.0.3",
      "cpe": "cpe:2.3:a:python-git-python:python-git-python:1.0.3:*:*:*:*:*:*:*",
      "purl": "pkg:pypi/git-python@1.0.3",
      "properties": [
        {
          "name": "syft:package:foundBy",
          "value": "python-package-cataloger"
        },
        {
          "name": "syft:package:language",
          "value": "python"
        },
        {
          "name": "syft:package:type",
          "value": "python"
        },
        {
          "name": "syft:package:metadataType",
          "value": "python-pip-requirements-entry"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-git-python:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git_python:python-git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git_python:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git-python:python-git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git-python:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git_python:python-git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git_python:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-git-python:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-git-python:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-git:python-git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-git:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git:python-git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git_python:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git_python:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python:python-git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git-python:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git-python:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git:python-git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git:python_git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git_python:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git_python:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-git:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-git:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_git:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git:git-python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:git:git_python:1.0.3:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:location:0:path",
          "value": "/examples/pytorch/_tests_requirements.txt"
        }
      ]
    },
    {
      "bom-ref": "pkg:pypi/gpustat@1.1.1?package-id=610746c2b5daefcd",
      "type": "library",
      "name": "gpustat",
      "version": "1.1.1",
      "cpe": "cpe:2.3:a:python-gpustat:python-gpustat:1.1.1:*:*:*:*:*:*:*",
      "purl": "pkg:pypi/gpustat@1.1.1",
      "properties": [
        {
          "name": "syft:package:foundBy",
          "value": "python-package-cataloger"
        },
        {
          "name": "syft:package:language",
          "value": "python"
        },
        {
          "name": "syft:package:type",
          "value": "python"
        },
        {
          "name": "syft:package:metadataType",
          "value": "python-pip-requirements-entry"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-gpustat:python_gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_gpustat:python-gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_gpustat:python_gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:gpustat:python-gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:gpustat:python_gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python-gpustat:gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python_gpustat:gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python:python-gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python:python_gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:gpustat:gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:cpe23",
          "value": "cpe:2.3:a:python:gpustat:1.1.1:*:*:*:*:*:*:*"
        },
        {
          "name": "syft:location:0:path",
          "value": "/benchmark/requirements.txt"
        }
      ]
    }
```

- syft의 경우 dependsOn 등과 같이 그래프 기반의 탐색 형태를 띄고 있진 않지만 해당 파일의 의존성을 어느 부분에서 찾았다는 내용을 properties의 하위 항목에서 역추적할 수 있다.

---

## 3. trivy

- 출력 결과
    
    [transformers_SBOM.json](transformers_SBOM%201.json)
    

```json
"metadata": {
    "timestamp": "2026-01-09T05:35:10+00:00",
    "tools": {
      "components": [
        {
          "type": "application",
          "manufacturer": {
            "name": "Aqua Security Software Ltd."
          },
          "group": "aquasecurity",
          "name": "trivy",
          "version": "0.68.2"
        }
      ]
    },
    "component": {
      "bom-ref": "fffbbf4e-8a48-4965-9160-fa57170e8bb5",
      "type": "application",
      "name": "/src",
      "properties": [
        {
          "name": "aquasecurity:trivy:SchemaVersion",
          "value": "2"
        }
      ]
    }
  }
```

- Hatbom, syft보다 더 정교한 형태의 metadata를 제공한다. tools 필드도 syft와 다르게 기업 명칭을 사용한 것을 볼 수 있다. component의 경우  최상위 디렉토리로 syft와 동일하게 /src를 alias한 것을 확인할 수 있다. properties 필드는 필수 사항은 아니지만 추가 네임스페이스를 사용하거나 확장자를 생성할 필요 없이 표준에서 공식적으로 지원되지 않는 데이터를 유연하게 포함할 수 있다. properties에서의 name은 동일한 명칭을 가져도 되지만 이를 구분하기 위해 value 값은 고유한 값이어야 한다.

```json
-- components 필드의 element --
{
      "bom-ref": "019d81f4-0937-496e-9e38-1ca8e7633437",
      "type": "application",
      "name": "benchmark/requirements.txt",
      "properties": [
        {
          "name": "aquasecurity:trivy:Class",
          "value": "lang-pkgs"
        },
        {
          "name": "aquasecurity:trivy:Type",
          "value": "pip"
        }
      ]
    },
```

- 이전과 다르게 type이 application이지만 name에서 파일이 나온 것을 볼 수 있다. properties의 경우 aquasecurity:trivy:Class, aquasecurity:trivy:Type로 나눌 수 있다. 전자는 trivy가 이 컴포넌트를 “어떤 스캔 클래스(큰 분류)”로 취급했는지를 나타내는 **분류 라벨**이다. 후자는 Class로 분류된 대상이 어떤 생태계/패키지 매니저 타입으로 스캔되었는 지를 나타내는 라벨이다.

```json
-- dependencies 필드의 element --
{
      "ref": "019d81f4-0937-496e-9e38-1ca8e7633437",
      "dependsOn": [
        "pkg:pypi/gpustat@1.1.1",
        "pkg:pypi/psutil@6.0.0",
        "pkg:pypi/psycopg2@2.9.9"
      ]
      
    ...
    
    
    {
      "ref": "pkg:pypi/datasets@1.8.0",
      "dependsOn": []
    },
```

- 해당 패키지가 어떤 패키지를 의존하고 있는지를 나타낸다. dependsOn 필드를 보면 pkg:를 통해 어떤 패키지로 관리 하는지를 알 수 있으며 패키지 이름과 버전을 볼 수 있다.  ref의 이름은 BOM 내부에서 해당 패키지를 구분할 수만 있으면 된다. 따라서 외부에서 다운 받은 패키지의 경우 ‘pkg:{ 패키지 관리자 명 } / { 패키지 이름 } @ 버전’과 같은 형태로 나타낸다. (`benchmark/requirements.txt` 와 같은 의존성 선언 파일의 경우 대상이 패키지로 볼 수 없어 버저닝을 하기 애매한 경우가 있다. 이러한 경우 UUID를 이용하여 해시 값을 ref로 사용한다.)

```json
  "vulnerabilities": []
```

- trivy의 고유 특징이라면 해당 SBOM에서의 취약점을 찾는 다는 것이다. 해당 프로젝트에서 취약점이 보이지 않는다면 해당 필드 값은 비어있다.

---

## 4. cdxgen

- 출력 결과
    
    [transformers_SBOM.json](transformers_SBOM%202.json)
    

```json
"metadata": {
    "timestamp": "2026-01-09T06:50:35Z",
    "tools": {
      "components": [
        {
          "group": "@cyclonedx",
          "name": "cdxgen",
          "version": "12.0.0",
          "purl": "pkg:npm/%40cyclonedx/cdxgen@12.0.0",
          "type": "application",
          "bom-ref": "pkg:npm/@cyclonedx/cdxgen@12.0.0",
          "publisher": "OWASP Foundation",
          "authors": [
            {
              "name": "OWASP Foundation"
            }
          ]
        }
      ]
    },
    "authors": [
      {
        "name": "OWASP Foundation"
      }
    ],
    "lifecycles": [
      {
        "phase": "build"
      }
    ],
    "component": {
      "group": "",
      "name": "app",
      "version": "latest",
      "type": "application",
      "bom-ref": "pkg:pypi/app@latest",
      "purl": "pkg:pypi/app@latest"
    },
    "properties": [
      {
        "name": "cdx:bom:componentTypes",
        "value": "github\\npypi"
      },
      {
        "name": "cdx:bom:componentNamespaces",
        "value": "actions\\nconda-incubator\\ndocker\\nhuggingface\\nhuggingface/hf-workflows/.github/actions\\npypa\\nslackapi\\ntrufflesecurity"
      },
      {
        "name": "cdx:bom:componentSrcFiles",
        "value": ".github/workflows/add-model-like.yml\\n.github/workflows/assign-reviewers.yml\\n.github/workflows/benchmark.yml\\n.github/workflows/benchmark_v2.yml\\n.github/workflows/build-ci-docker-images.yml\\n.github/workflows/build-docker-images.yml\\n.github/workflows/build-nightly-ci-docker-images.yml\\n.github/workflows/build-past-ci-docker-images.yml\\n.github/workflows/check_failed_tests.yml\\n.github/workflows/check_tiny_models.yml\\n.github/workflows/circleci-failure-summary-comment.yml\\n.github/workflows/collated-reports.yml\\n.github/workflows/doctest_job.yml\\n.github/workflows/doctests.yml\\n.github/workflows/get-pr-info.yml\\n.github/workflows/model_jobs.yml\\n.github/workflows/model_jobs_intel_gaudi.yml\\n.github/workflows/new_model_pr_merged_notification.yml\\n.github/workflows/pr-repo-consistency-bot.yml\\n.github/workflows/pr_slow_ci_suggestion.yml\\n.github/workflows/push-important-models.yml\\n.github/workflows/release-conda.yml\\n.github/workflows/release.yml\\n.github/workflows/self-comment-ci.yml\\n.github/workflows/self-nightly-caller.yml\\n.github/workflows/self-scheduled-caller.yml\\n.github/workflows/self-scheduled-flash-attn-caller.yml\\n.github/workflows/self-scheduled-intel-gaudi.yml\\n.github/workflows/self-scheduled.yml\\n.github/workflows/slack-report.yml\\n.github/workflows/ssh-runner.yml\\n.github/workflows/stale.yml\\n.github/workflows/trufflehog.yml\\n.github/workflows/update_metdata.yml\\nPyPI package: django-git\\nbenchmark/requirements.txt\\nbenchmark_v2/requirements.txt\\ncdxgen-venv-x5HuYY\\npyproject.toml\\ntests/sagemaker/scripts/pytorch/requirements.txt"
      }
    ]
  }
```

- 가장 정교한 metadata 필드를 생성한 것을 볼 수 있다.
- component의 경우 syft, trivy와 다르게 app이라는 명칭을 사용하였다.
- `cdx:bom:componentTypes = "github\npypi"`
    - 이 BOM에 포함된 컴포넌트 출처/유형이 `github`(GitHub Actions 등) + `pypi`(파이썬 패키지) 중심이라는 요약이다.
- `cdx:bom:componentNamespaces = "actions\nconda-incubator\n..."`
    - 컴포넌트들의 네임스페이스(예: GitHub org/namespace, actions 공식 namespace 등)를 요약한 인덱스이다.
- `cdx:bom:componentSrcFiles = ".github/workflows/... \n pyproject.toml \n ..."`
    - “어떤 소스/매니페스트/파일들을 근거로 컴포넌트를 추출했는지”의 목록(대개 입력 스코프)이다.
    - 예시를 보면 `.github/workflows/*.yml`, `pyproject.toml`, 여러 `requirements.txt`가 포함되어 있어서, cdxgen이 **GitHub Actions 의존성 + PyPI 의존성(매니페스트 기반)**을 함께 끌어온 것으로 보인다.

```json
{
      "ref": "pkg:pypi/transformers@latest",
      "dependsOn": [
        "pkg:pypi/filelock@3.20.2",
        "pkg:pypi/huggingface-hub@1.2.4",
        "pkg:pypi/numpy@2.4.0",
        "pkg:pypi/packaging@25.0",
        "pkg:pypi/pyyaml@6.0.3",
        "pkg:pypi/regex@2025.11.3",
        "pkg:pypi/requests@2.32.5",
        "pkg:pypi/safetensors@0.7.0",
        "pkg:pypi/tokenizers@0.22.2",
        "pkg:pypi/tqdm@4.67.1",
        "pkg:pypi/typer-slim@0.21.1"
      ]
    }
```

- dependencies를 볼 수 있으며 이전의 다른 generator와는 다르게 내부 의존성을 넣진 않아 모든 ref가 실제 패키지 이름과 버저닝 형태로 출력된 것을 볼 수 있다.

```json
{
      "authors": [
        {
          "name": "Seth Buntin <sethtrain@gmail.com>"
        }
      ],
      "group": "",
      "name": "django-git",
      "version": "0.1",
      "description": "django gitweb replacement",
      "hashes": [
        {
          "alg": "SHA-256",
          "content": "e2320fcea2accf78128c3520d94e9865fc938d386a9e4bf49d9973277c0a7f7d"
        }
      ],
      "licenses": [
        {
          "license": {
            "id": "0BSD",
            "url": "https://opensource.org/licenses/0BSD"
          }
        }
      ],
      "purl": "pkg:pypi/django-git@0.1",
      "externalReferences": [
        {
          "type": "vcs",
          "url": "http://code.google.com/p/django-git"
        }
      ],
      "type": "framework",
      "bom-ref": "pkg:pypi/django-git@0.1",
      "properties": [
        {
          "name": "SrcFile",
          "value": "tests/sagemaker/scripts/pytorch/requirements.txt"
        },
        {
          "name": "cdx:pypi:versionSpecifiers",
          "value": "+https://github.com/huggingface/transformers.git@main # install main or adjust it with vX.X.X for installing version specific transforms"
        }
      ],
      "evidence": {
        "identity": [
          {
            "field": "version",
            "confidence": 0.5,
            "methods": [
              {
                "technique": "source-code-analysis",
                "confidence": 0.5,
                "value": "PyPI package: django-git"
              }
            ],
            "concludedValue": "PyPI package: django-git"
          }
        ]
      },
      "tags": [
        "framework"
      ]
    },
```

- `authors`
    - 해당 컴포넌트(패키지)의 저자/관리자 정보이다. 보통 패키지 메타데이터(Python 패키지의 `Author`, `Maintainer` 등)에서 가져오거나, 도구가 추정한 값이 들어간다.
- `group`
    - 생태계별 “그룹/네임스페이스”이다.
    - PyPI는 Maven처럼 group을 강제하지 않아서 보통 `""`인 경우가 많습니다. (npm scope, GitHub org 같은 건 group/namespace로 들어가기도 한다.)
- `name`, `version`
    - 패키지 이름과 버전이다. 이 값과 `purl`이 식별의 핵심이 된다.
    - 여기서는 `django-git`의 `0.1` 버전.
- `description`
    - 패키지 설명(메타데이터 요약)이다.
- `hashes`
    - 컴포넌트를 특정 아티팩트(예: wheel/sdist 파일) 단위로 고정하기 위한 해시이다.
    - `SHA-256` + `content`는 “무결성 검증”에 유용하지만, 어떤 파일(whl? tar.gz?)의 해시인지가 별도 필드로 명시되지 않는 경우가 많아(도구별 차이) 감사 시에는 추가 근거(다운로드 URL, 파일명)가 있으면 더 좋다.
- `licenses`
    - 라이선스 정보이다.
    - `license.id: "0BSD"`처럼 SPDX 식별자를 쓰면 가장 표준적이고, `url`은 근거 링크로 유용하다.
- `purl`
    - 전역 식별자(package-url)이다. 이 컴포넌트를 “PyPI의 django-git 0.1”로 명확히 지칭한다.
    - 보통 `bom-ref`도 purl과 동일하게 맞춰 “참조키 = 전역식별자”로 쓰는 패턴이 많다.
- `externalReferences`
    - 이 컴포넌트에 대한 외부 링크들이다.
    - `type: "vcs"`는 소스 저장소 URL을 의미합니다(단, 예시 URL이 오래된 google code 링크라 현재 유효하지 않을 수도 있다).
- `type`
    - CycloneDX의 분류값이다(예: `library`, `framework`, `application` 등).
    - PyPI 패키지 대부분은 `library`로 두는 편이 일반적이고, 여기처럼 `framework`로 나오는 건 cdxgen의 휴리스틱/태그 매핑 결과일 수 있다. “틀렸다”기보다는 분류 기준이 도구마다 다르다.
- `bom-ref`
    - BOM 내부에서 이 컴포넌트를 가리키는 유일 참조키이다.
    - 여기서는 purl과 동일(`pkg:pypi/django-git@0.1`)이라서 `dependencies.ref/dependsOn` 같은 곳에서 참조하기 좋다.
- `properties`
    - 표준 필드로 담기 애매한 “도구 확장 메타데이터”이다.
    - `SrcFile`: 이 컴포넌트를 발견한 근거 파일(여기서는 `tests/sagemaker/scripts/pytorch/requirements.txt`)이다. 즉 “이 requirements에서 이 패키지를 읽었다”는 provenance.
    - `cdx:pypi:versionSpecifiers`: 원래는 매니페스트에 적힌 버전 제약(예: `>=1.0`, `~=2.1`) 같은 “원문 스펙”이 들어가는 자리인데, 예시 값이 `+https://github.com/huggingface/transformers.git@main ...`처럼 **VCS 설치 라인 형태**여서 `django-git` 컴포넌트와 내용이 어긋나 보인다. 이런 케이스는 (1) 파싱 과정에서 라인이 잘못 매핑됐거나, (2) requirements에 특이한 라인이 섞여 있고 도구가 이를 패키지 스펙으로 보관한 것일 수 있어요. “이 값은 참고용”으로 보고, 실제 의존성은 requirements 원문과 함께 교차검증하는 게 안전하다.
- `evidence.identity`
    - CycloneDX의 “이 컴포넌트 식별을 어떤 근거로 확신했는가”를 표현하는 영역이다.
    - `field`: 어떤 필드(예: `purl`, `name`, `version`)를 어떤 근거로 결론냈는지
    - `confidence`: 신뢰도(0~1)
    - `methods`: 사용한 방법(예: `manifest-analysis`, `source-code-analysis`)과 근거 값
    - 이 예시에서는 `confidence: 0.5`로 낮게 잡혀 있고, `concludedValue`가 `"PyPI package: django-git"`처럼 **버전 결론이라기엔 문구가 이상**해서, 이 evidence도 “정교한 증거체계”라기보다는 cdxgen이 남긴 힌트 수준으로 보는 게 좋다.
- `tags`
    - 도구가 붙인 분류 태그입니다(검색/필터용). 표준의 강한 의미라기보다 분류 편의 기능에 가깝다.

```json
"services": [],
```

- 해당 필드는 이 BOM이 다루는 대상 중 **네트워크로 제공되는 서비스(서비스 엔드포인트)**를 기술하는 배열이다.
- **식별/기본정보**
    - `bom-ref`: 서비스의 내부 참조키(유일)
    - `name`: 서비스 이름(예: `user-api`, `auth-service`)
    - `version`: 서비스 버전(선택)
    - `group`: 조직/도메인(선택)
    - `description`: 서비스 설명(선택)
- **엔드포인트(가장 핵심)**
    - `endpoints`: 서비스 접근 지점 목록예: `https://api.example.com/v1`, `grpc://auth.example.com:443` 같은 URI/주소들
- **인증/권한**
    - `authenticated`: 인증 필요 여부(true/false)
    - (필요 시) `properties`에 인증 방식(OAuth2, mTLS 등) 같은 확장 메타를 넣기도 함
- **데이터 분류/신뢰경계(선택)**
    - `data`: 서비스가 처리/노출하는 데이터 유형(민감정보 여부 등)조직에 따라 정책적으로 채우는 경우가 있음
- **라이선스/외부참조(선택)**
    - `licenses`: 서비스 제공/사용에 대한 라이선스(드문 편)
    - `externalReferences`: 문서/런북/오픈API 스펙/리포지토리 링크 등예: OpenAPI URL, 서비스 카탈로그 링크
- **구성요소와의 연결**
    - 서비스가 실제로 어떤 애플리케이션/컨테이너/컴포넌트로 구현되는지는 `services` 자체에 모두 들어가기보다,
    - `dependencies` 그래프에서 `ref`로 서비스 `bom-ref`를 포함시키거나, 다른 메타(예: `composition`, `properties`)로 연계하는 패턴이 많다.

```json
"annotations": [
    {
      "bom-ref": "metadata-annotations",
      "subjects": [
        "pkg:pypi/app@latest"
      ],
      "annotator": {
        "component": {
          "group": "@cyclonedx",
          "name": "cdxgen",
          "version": "12.0.0",
          "purl": "pkg:npm/%40cyclonedx/cdxgen@12.0.0",
          "type": "application",
          "bom-ref": "pkg:npm/@cyclonedx/cdxgen@12.0.0",
          "publisher": "OWASP Foundation",
          "authors": [
            {
              "name": "OWASP Foundation"
            }
          ]
        }
      },
      "timestamp": "2026-01-09T06:50:35Z",
      "text": "This Software Bill-of-Materials (SBOM) document was created on Friday, January 9, 2026 with cdxgen. The data was captured during the build lifecycle phase. The document describes an application named 'app'. There are 58 components. 2 package type(s) and 8 purl namespaces are described in the document under components."
    }
  ]
```

- `annotations[]`
    - 주석의 목록입니다. 한 BOM에 여러 개의 주석을 달 수 있다(예: 생성 요약, 리뷰 코멘트, 예외 처리 사유 등).
- `bom-ref: "metadata-annotations"`
    - 이 주석 객체 자체를 BOM 내부에서 참조하기 위한 ID이다(다른 곳에서 이 annotation을 가리킬 때 사용 가능).
- `subjects: ["pkg:pypi/app@latest"]`
    - 이 주석이 “어떤 대상을 설명/주석하는지”를 나열한다.
    - 값은 보통 **대상 요소의 `bom-ref`*를 넣는 게 정석인데, 여기서는 루트 컴포넌트(= `metadata.component`)의 `bom-ref`인 `pkg:pypi/app@latest`를 가리키고 있다,
    - 따라서 이 annotation은 “이 BOM의 주제(루트 대상) 컴포넌트에 대한 주석”이다..
- `annotator.component { ... cdxgen ... }`
    - “누가 이 주석을 달았는가”를 컴포넌트로 표현한다.
    - 여기서는 `@cyclonedx/cdxgen@12.0.0`가 주석 작성자(annotator)로 들어가 있어, **cdxgen이 자동 생성한 요약 주석**임을 의미한다.
- `timestamp`
    - 이 주석이 작성된 시각이다(대개 BOM 생성 시각과 같거나 매우 근접).
- `text`
    - 사람이 읽는 요약문입니다. 지금 내용은 cdxgen이 BOM을 생성하면서 자동으로 만든 “SBOM 생성 리포트”다.
    - 포함된 정보(예: “build lifecycle phase”, “application named 'app'”, “58 components”, “2 package type(s)”, “8 purl namespaces”)는 BOM의 여러 값을 **문장으로 재서술**한 것이고, 파서가 신뢰해야 하는 단일 근거라기보다 “요약”이다. (실제 정합성 검증은 `components`, `metadata`, `dependencies` 같은 구조화된 필드를 기준으로 하는 게 맞습니다.)

---

# Hatbom의 개선점

- **참조키 정합성(`bom-ref` ↔ `dependencies.ref/dependsOn`)**: HatBOM 결과에서 가장 치명적인 개선 포인트는 “그래프가 실제 컴포넌트를 정확히 가리키는가”이다. `dependencies[].ref`와 `components[].bom-ref`, `metadata.component.bom-ref`가 1:1로 맞고, 모든 `dependsOn[]`이 실제 존재하는 `bom-ref`만 참조하도록 보장해야 한다(미해결 ref는 SBOM 소비자에서 그래프가 깨짐).
- **루트 대상 식별(`metadata.component`) 고도화**: 현재는 `pkg:generic` + 빈 `version` 같은 형태가 나오기 쉬운데, 가능하면 실제 프로젝트 식별자(이름/버전/가능하면 purl)를 명확히 넣어 “이 SBOM이 무엇의 SBOM인지”가 한눈에 결정되게 해야 한다.
- **purl 적극 사용(특히 패키지 컴포넌트)**: HatBOM이 파일 중심(`type=file`) 인벤토리로 치우치면, 취약점 DB/라이선스 자동분석과 연계가 약해진다. 가능하면 생태계 패키지(PyPI 등)는 `purl`을 채우고, `bom-ref`도 purl과 동일하게 두는 패턴이 상호운용성이 가장 좋다. (내부 의존성이라 해당 방식의 어려움이 있다는 인식이 있다.)
- **해시 알고리즘 개선(MD5 지양)**: 파일 컴포넌트에 MD5만 있는 경우가 많으면 무결성/공급망 관점에서 품질이 떨어진다. 가능하면 `SHA-256` 이상으로 올리고, 어떤 아티팩트(소스파일/배포파일) 해시인지도 일관되게 관리하는 것이 좋다.
- **CycloneDX 버전 업(가능하면 1.6 이상)**: HatBOM이 1.4로 생성된 케이스라면, Syft/Trivy/cdxgen(1.6)과 비교 시 표현력과 호환성에서 손해가 있습니다. 1.6 이상으로 올리면 툴/증거/라이프사이클/주석 등 메타데이터 표준화 흐름에 맞추기 좋다.
- **`metadata.tools`(도구 provenance) 보강**: HatBOM 결과는 도구 정보가 약하게 들어가거나 `authors`만 남는 형태가 나오기 쉬운데, SBOM 신뢰성과 재현성을 위해 생성기 이름/버전/벤더를 `metadata.tools`로 명확히 남기는 게 좋다(Trivy/Syft/cdxgen처럼).

---

# SBOM Generator 비교 (python 프로젝트 출력 결과물)

### - 준수 사항 (CISA Minimum Elements 11)

| **SBOM (generator)** | CycloneDX Version | **컴포넌트 수** (`components` 배열의 원소 개수) | **SBOM Author** | Tool Name | Timestamp | Generation Context | Dependency Relationship | Producer | Name | Version | Identifiers | Hash | License |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **hatbom(HatBom/Hmark)** | 1.4 | 29 | O | X | O | X | O | 0.0% | 100% | 100% | 100% | 100% | 0.0% |
| **syft** | 1.6 | 180 | X | O | O | X | X | 0.0% | 100% | 56.7% | 100% | 29.4% | 0.0% |
| **trivy** | 1.6 | 6 | X | O | O | X | O | 0.0% | 100% | 66.7% | 100% | 0.0% | 0.0% |
| **cdxgen** | 1.6 | 58 | O | O | O | O | O | 19.0% | 100% | 100% | 100% | 3.4% | 20.7% |
- syft, trivy, cdxgen 테스트 당시 일부는 default로 출력시 1.6이 나옴. 버전을 맞추기 위해 1.6으로 설정하고 테스트를 진행하였음.
- **판단 기준**
    
    ## 1) “컴포넌트 수”의 의미(분모 정의)
    
    - **컴포넌트 수 = `components[]` 배열의 원소 개수**
        - 일반 CycloneDX: `$.components.length`
        - Hatbom(Hmark)처럼 래퍼가 있는 경우: `$.sbom.components.length`
    - **의도적으로 제외한 것**
        - `metadata.component`(SBOM이 설명하는 “대상 소프트웨어” 1개) — 이것까지 포함하면 “의존 컴포넌트 수”와 섞여서 비교가 흔들림.
        - `dependencies[]`는 컴포넌트가 아니라 “관계(그래프의 엣지)”라서 별도.
    
    > 참고: CISA는 “모든 컴포넌트 + 전이 의존성까지 포함, 깊이 제한 없음”을 Coverage 관점에서 요구
    > 
    
    ---
    
    ## 2) 표의 O / X 와 %의 의미
    
    ### A. O / X (SBOM-레벨 필드: “문서에 존재하는가?”)
    
    다음 5개는 “컴포넌트별”이라기보다 **SBOM 문서(메타데이터)에서 확인되는지**가 중요해서 **O/X**로 표시
    
    - **SBOM Author**: SBOM 데이터를 생성한 엔터티(Producer와 구분)
    - **Tool Name**: SBOM 생성/보강에 사용된 도구 이름(복수 도구면 모두)
    - **Timestamp**: SBOM 데이터 최종 갱신 시각, ISO 8601
    - **Generation Context**: before/during/after build 중 어떤 생성 컨텍스트인지
    - **Dependency Relationship**: 의존성 그래프(포함/파생 관계)
    
    **O 판정(기본 규칙)**: 해당 JSONPath에 값이 있고 “비어있지 않음(또는 형식 조건 만족)”
    
    **X 판정**: 필드가 없거나, 빈 문자열/빈 배열이거나, 형식 조건(예: ISO8601)이 충족되지 않음
    
    ### B. % (컴포넌트-레벨 필드: “컴포넌트들 중 몇 %가 갖고 있나?”)
    
    다음 6개는 CISA가 “각 컴포넌트에 적용되는 데이터 필드”라고 명시한 핵심이라, components[] 전체를 분모로 커버리지(%)를 계산
    
    - **Producer 커버리지**
    - **Name 커버리지**
    - **Version 커버리지**
    - **Identifiers 커버리지**
    - **Hash 커버리지**
    - **License 커버리지**
    
    **커버리지 계산식**
    
    - `커버리지(%) = (조건을 만족하는 컴포넌트 개수 / 전체 components 개수) × 100`
    
    ---
    
    ## 3) 11개 필드별 “매핑 기준 + 유효성 체크 규칙”
    
    **(CISA 정의 요지 → CycloneDX 매핑 → 판정 기준)**
    
    ### 1) SBOM Author (O/X)
    
    - **CISA 의미**: “이 컴포넌트의 SBOM 데이터를 만든 엔터티 이름”이며 Software Producer와 구분
    - **CycloneDX 매핑(우선순위)**: `metadata.authors[].name`
    - **O 기준**: `metadata.authors`가 존재하고, name이 비어있지 않음
    
    ### 2) Software Producer (%)
    
    - **CISA 의미**: 컴포넌트를 “생산/정의/식별”한 원 제작자(불명확하면 unknown provenance 표시 권고)
    - **CycloneDX 매핑(현실적 후보)**: 각 component의 `manufacturer.name` / `supplier.name` / `publisher`
    - **% 기준**: 위 후보 중 하나라도 비어있지 않으면 그 컴포넌트는 “Producer 있음”으로 카운트
        - (주의) CycloneDX에 producer를 항상 넣는 관행이 없어 **대부분 0%가 흔하게 나올 수 있음**. 이건 “도구가 producer를 안 채운 것”이지, 실제 producer가 없다는 뜻은 아님.
    
    ### 3) Component Name (%)
    
    - **CISA 의미**: Producer가 부여한 컴포넌트 이름(식별자와 구분)
    - **CycloneDX 매핑**: `components[].name`
    - **% 기준**: name이 존재하고 빈 문자열이 아니면 OK
    
    ### 4) Component Version (%)
    
    - **CISA 의미**: 특정 코드 전달물(code delivery)을 식별하는 버전. Producer가 안 주면 파일 생성일로 대체 가능
    - **CycloneDX 매핑**: `components[].version`
    - **% 기준(엄격)**: version이 존재하고 빈 값/`UNKNOWN` 같은 플레이스홀더가 아니면 OK
    
    ### 5) Software Identifiers (%)
    
    - **CISA 의미**: 취약점/라이선스 DB 매핑을 위한 “최소 1개 이상의” 기계처리 가능 식별자. 여러 개면 모두 포함 권고
    - **CycloneDX 매핑(대표)**: `purl`, `cpe`, `swid`, `externalReferences`, (또는 `bom-ref`가 purl/urn 형태일 때)
    - **% 기준**: 위 후보 중 하나라도 존재하면 OK
    
    ### 6) Component Hash (%)
    
    - **CISA 의미**: 해시값 + 알고리즘 식별. 단, 원본 artifact 접근 불가면 해시를 넣지 말라고도 명시
    - **CycloneDX 매핑**: `components[].hashes[].{alg, content}`
    - **% 기준**: hashes 배열에 `alg`와 `content`가 함께 존재하면 OK
    
    ### 7) License (%)
    
    - **CISA 의미**: 가능한 기계처리(SPDX 등)로 표기, 독점 라이선스 조건 존재도 포함, 모르면 unknown 표시
    - **CycloneDX 매핑**: `components[].licenses[]` (license.id / license.name / expression)
    - **% 기준**: licenses가 존재하고 SPDX id/name/expression 중 하나라도 있으면 OK
    
    ### 8) Dependency Relationship (O/X)
    
    - **CISA 의미**: includes/derived 관계로 의존성 그래프 구성, 포크/백포트 계보까지 표현
    - **CycloneDX 매핑(기본)**: 최상위 `dependencies[]` 존재 여부
    - **O 기준(기본)**: dependencies 배열이 존재하고 ref/dependsOn 연결이 있다
    
    ### 9) Tool Name (O/X)
    
    - **CISA 의미**: SBOM 생성/보강에 사용한 도구 이름(여러 도구면 모두)
    - **CycloneDX 매핑(우선순위)**: `metadata.tools.components[].name`
    - **O 기준**: tool component name이 1개 이상 존재
        - 일부 도구는 `annotations.annotator.component.name`에도 남기므로, 보고서에서는 “표준 위치 + 보완 위치 둘 다 검사”라고 적어두면 좋음.
    
    ### 10) Timestamp (O/X)
    
    - **CISA 의미**: SBOM 데이터 최종 업데이트 시각(수정 없으면 생성 시각), ISO 8601
    - **CycloneDX 매핑**: `metadata.timestamp`
    - **O 기준**: timestamp가 존재하고 ISO 8601 형태로 파싱 가능
    
    ### 11) Generation Context (O/X)
    
    - **CISA 의미**: SBOM 생성 시점의 생명주기 컨텍스트(before/during/after build)로, 데이터 출처/가시성을 이해시키는 정보
    - **CycloneDX 매핑(현실적)**: `metadata.lifecycles[].phase` 존재 여부(예: build)
    - **O 기준(보수적)**: lifecycles.phase가 있으면 “컨텍스트 단서 존재”로 O

---

### 1) hatbom(Hmark)

- `metadata.authors`와 `metadata.timestamp`는 존재 transformers_hmark_SBOM_hatbom
- **Tool Name**을 나타내는 `metadata.tools`가 없고(최소 필드 관점에서 Tool Name 미충족),
    
    **Generation Context**도 표준 필드로는 드러나지 않음
    
- 대신 컴포넌트들이 전부 **file 단위 + MD5 해시**를 갖고 있어 Hash 커버리지가 100%
- **License 필드가 컴포넌트에 없음**(0%)

### 2) syft

- `metadata.tools.components[0].name = syft`, `metadata.timestamp`는 있음 transformers_SBOM_syft
- 하지만 `metadata.authors`(SBOM Author)가 없고
    
    `components[].version = "UNKNOWN"` 같은 항목이 많아 Version 커버리지가 내려감 
    
- 일부 file 컴포넌트에는 **SHA-1/SHA-256 hashes**가 있어 Hash 커버리지가 일부 존재
- **Dependency Relationship 섹션(`dependencies`)이 SBOM 최상위에 없음**(그래프 기반 분석이 어렵다는 의미)

### 3) trivy

- `metadata.tools.components[0].name = trivy` 및 timestamp 존재 transformers_SBOM_trivy
- 그러나 `metadata.authors`(SBOM Author)가 없고
    
    `benchmark/requirements.txt` 같은 “파일(애플리케이션 타입)” 컴포넌트에는 `version`이 없어 Version 커버리지가 100%가 안 됨 
    
- Dependency Relationship(`dependencies`)는 존재
- 라이선스/해시는 컴포넌트에 안 보임(커버리지 0%)

### 4) cdxgen

- **SBOM Author / Tool / Timestamp / Generation Context**가 모두 잘 들어가 있음
    - `metadata.tools.components[0].name = cdxgen`, `metadata.authors = OWASP Foundation`, `metadata.timestamp`, `metadata.lifecycles.phase = build`
- **Dependency Relationship**도 풍부하게 존재 (ref/dependsOn 그래프)
- 컴포넌트 중 일부는 **라이선스/해시까지 제공**(예: `django-git`에 hashes+licenses+purl)
    
    다만 모든 컴포넌트에 해시/라이선스가 있는 건 아니라서 커버리지가 낮게 나옴.