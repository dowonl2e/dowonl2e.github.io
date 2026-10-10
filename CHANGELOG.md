# Changelog

모든 주요 변경 사항은 이 파일에 기록됩니다.

## [2026-10-10]

### 변경 사항

- Serverless CI/CD 포스트 제목을 "Terraform을 이용한 Serverless CI/CD 구축"으로 확정
- 파일명에 `/`가 포함되어 디렉토리로 잘못 생성된 포스트를 정상 파일로 복구 (`2026-10-10-Terraform을 이용한 Serverless CICD 구축.md`)
- Serverless CI/CD 포스트 본문 작성 (레포지토리 분리 배경, 첫 파이프라인 구성, GitLab CI 개선 5가지, 개선 전후 비교)
- CI/CD 파이프라인 결과 이미지 2건을 `assets/img/Serverless-Terraform/`에 추가
- 기존 포스트 관례에 맞춰 문체를 평서체(`~이다`/`~한다`)로 통일하고 `.text-*` 강조 및 `.prompt-info` 블록 적용
- 작업 배경(서버리스 인프라, 로컬 배포로 인한 버전관리 문제)을 `개요` 섹션으로 추가
- `Artifact 만료 시간 조정`, `브랜치 규칙 도입` 섹션 제거
- `프로젝트 구조 개선` 섹션 작성 (배포 스크립트 분기 제거, 설정 환경 분기 제거, 설정 파일 단위 변경, 레포지토리 역할 경계)
- 구조 개선 후 배포 결과 이미지(`3-Restructured_CICD_Result.png`) 및 측정 표 추가
- `CDKTF에서 Terraform으로` 섹션 작성 (전환 배경, CDKTF 대비 차이 3가지, 서비스 레포 무영향)
- Terraform 전환 결과 이미지(`4-Terraform_CICD_Result.png`) 및 CDKTF/Terraform 배포 시간 비교 표 추가
- `TypeScript` 태그 대소문자 불일치로 발생하던 Jekyll 빌드 Conflict 경고 해소
- `_config.yml`에 `sass.quiet_deps`/`silence_deprecations` 추가로 Dart Sass `@import` deprecation 경고 제거
- `plan` 스테이지를 합계에서 제외하고 사전 체크 단계로 서술, 배포 시간 단축률 재계산 (약 49%)
- `정리` 섹션 작성 (배포 시간 변화, 회고 3가지, 향후 계획)
