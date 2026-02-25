# syft

Github URL: https://github.com/anchore/syft
진행도: 완료

```json
// syft를 이용하여 만든 SBOM 발췌
// Django를 인식하지 않고 extras를 인식하여 추가한 모습이다.
{
      "ref": "pkg:pypi/dj-rest-auth@7.0.1?package-id=e5ae9d3dbec01ba2",
      "dependsOn": [
        "pkg:pypi/django-allauth@65.11.2?package-id=7252146fe0fe7589",
        "pkg:pypi/djangorestframework@3.16.1?package-id=e1fa1261e5a7882a"
      ]
 }
```

```toml
[[package]]
name = "dj-rest-auth"
version = "7.0.1"
description = "Authentication and Registration in Django Rest Framework"
optional = false
python-versions = ">=3.8"
groups = ["main"]
files = [
    {file = "dj-rest-auth-7.0.1.tar.gz", hash = "sha256:3f8c744cbcf05355ff4bcbef0c8a63645da38e29a0fdef3c3332d4aced52fb90"},
]

[package.dependencies]
Django = ">=4.2,<6.0"
djangorestframework = ">=3.13.0"

[package.extras]
with-social = ["django-allauth[socialaccount] (>=64.0.0)"]
```

---

```go
// 각 패키지가 제공하는 문자열을 pkgsProvidingResource[resource]에 등록.
func allProvides(pkgsProvidingResource map[string]internal.Set[artifact.ID], id artifact.ID, spec Specification) []ProvidesRequires {
	prs := []ProvidesRequires{spec.ProvidesRequires}
	prs = append(prs, spec.Variants...)

	for _, pr := range prs {
		for _, resource := range deduplicate(pr.Provides) {
			if pkgsProvidingResource[resource] == nil {
				pkgsProvidingResource[resource] = internal.NewSet[artifact.ID]()
			}
			pkgsProvidingResource[resource].Add(id)
		}
	}

	return prs
}
```

```go
// Resolve()에서 dependant 패키지의 spec.Requires(문자열들)를 돌면서,
// 그 문자열 resource를 key로 pkgsProvidingResource[resource]를 조회
func Resolve(specifier Specifier, pkgs []pkg.Package) (relationships []artifact.Relationship) {
	pkgsProvidingResource := make(map[string]internal.Set[artifact.ID])

	pkgsByID := make(map[artifact.ID]pkg.Package)
	specsByPkg := make(map[artifact.ID][]ProvidesRequires)

	for _, p := range pkgs {
		id := p.ID()
		pkgsByID[id] = p
		specsByPkg[id] = allProvides(pkgsProvidingResource, id, specifier(p))
	}

	seen := strset.New()
	for _, dependantPkg := range pkgs {
		specs := specsByPkg[dependantPkg.ID()]
		for _, spec := range specs {
			for _, resource := range deduplicate(spec.Requires) {
				for providingPkgID := range pkgsProvidingResource[resource] {
					// prevent creating duplicate relationships
					pairKey := string(providingPkgID) + "-" + string(dependantPkg.ID())
					if seen.Has(pairKey) {
						continue
					}

					providingPkg := pkgsByID[providingPkgID]

					relationships = append(relationships,
						artifact.Relationship{
							From: providingPkg,
							To:   dependantPkg,
							Type: artifact.DependencyOfRelationship,
						},
					)

					seen.Add(pairKey)
				}
			}
		}
	}
	return relationships
}
```

```go
// 패키지 의존성을 매핑하는 함수, 딕셔너리와 같은 키-페어로 찾음
for _, dependantPkg := range pkgs {
		specs := specsByPkg[dependantPkg.ID()]
		for _, spec := range specs {
			for _, resource := range deduplicate(spec.Requires) {
				for providingPkgID := range pkgsProvidingResource[resource] {
					// prevent creating duplicate relationships
					pairKey := string(providingPkgID) + "-" + string(dependantPkg.ID())
					if seen.Has(pairKey) {
						continue
					}

					providingPkg := pkgsByID[providingPkgID]

					relationships = append(relationships,
						artifact.Relationship{
							From: providingPkg,
							To:   dependantPkg,
							Type: artifact.DependencyOfRelationship,
						},
					)

					seen.Add(pairKey)
				}
			}
		}
```

```go
// ref: To, dependsOn: From
artifact.Relationship{
  From: providingPkg,   // “의존성 제공자(= dependency package)”
  To:   dependantPkg,   // “의존하는 쪽(= dependant package)”
  Type: artifact.DependencyOfRelationship, // 의존관계 이름
}
```

- Type이 매핑되는 과정
    - CycloneDX `dependsOn` 생성에 사용 (관계 타입 필터링)
        
        CycloneDX로 내보낼 때 toDependencies()가 관계를 순회하면서, r.Type이 “표현 가능한 타입인지” 먼저 검사합니다.
        
        - isExpressiblePackageRelationship 에서 `case artifact.DependencyOfRelationship: return true`
        - 그리고 toDependencies()에서 exists := isExpressiblePackageRelationship(r.Type)로 필터링즉 **DependencyOfRelationship인 것만** CycloneDX dependencies/ref/dependsOn으로 변환됩니다.
    - SPDX 변환에 사용 (relationship type 매핑)
        
        SPDX로 내보낼 때도 artifact.RelationshipType에 따라 SPDX relationship 문자열로 바꿉니다.
        
        lookupRelationship() 에서 case artifact.DependencyOfRelationship:를 처리합니다.
        
    - Syft JSON 디코딩/호환 처리에 사용
        
        Syft JSON을 읽어서 내부 artifact.Relationship로 만들 때, relationship.Type을 artifact.RelationshipType으로 캐스팅한 뒤 허용된 타입인지 체크합니다.
        
        toSyftRelationship()
        
    - SBOM 내부에서 “특정 타입 관계만 조회”할 때 사용
        
        패키지 기준으로 관계를 조회할 때, rt(relationship types) 목록에 포함되는지 relationship.Type == r로 필터링합니다. RelationshipsForPackage()
        
- syft는 해당 값들을 구조체에 저장하고 CycloneDX encoder을 통해 직렬화를 하여 한번에 Json 포맷으로 만든다.