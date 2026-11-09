# CLX CI·배포 준비 가이드

> 기준: 2026-11-09. 이 문서는 팀이 사용할 CI·배포 흐름과 준비 상태를 설명합니다. **설명이나 계획을 적었다고 해서 GitHub 설정이나 배포가 자동으로 적용되지는 않습니다.** 변경 전에는 각 PR과 팀의 최신 결정을 확인하세요.

브랜치·커밋·Issue·PR 작성 방식은 [Git 협업 가이드](GIT_협업_가이드.md)를 함께 참고합니다.

## 1. 저장소와 전체 흐름

| 저장소 | 역할 | CI 변경 사항 확인 |
| --- | --- | --- |
| [clx-fe](https://github.com/SKALA-CLX/clx-fe) | 프론트엔드 코드와 이미지 | [CI PR #5](https://github.com/SKALA-CLX/clx-fe/pull/5) |
| [clx-be](https://github.com/SKALA-CLX/clx-be) | 백엔드 코드와 이미지 | [CI PR #3](https://github.com/SKALA-CLX/clx-be/pull/3) |
| [clx-ai](https://github.com/SKALA-CLX/clx-ai) | AI 서비스 코드와 이미지 | [CI PR #1](https://github.com/SKALA-CLX/clx-ai/pull/1) |
| [clx-manifest](https://github.com/SKALA-CLX/clx-manifest) | Kubernetes 배포 설정을 관리할 저장소 | [CI PR #1](https://github.com/SKALA-CLX/clx-manifest/pull/1) |

팀이 만들려는 전체 흐름은 다음과 같습니다. **Harbor 이미지 게시와 Argo CD 이후는 아직 연결과 운영 여부를 확인해야 하는 목표입니다.**

```text
앱 저장소 변경 → PR 검사·팀 리뷰 → develop 반영
              → main 대상 별도 PR·리뷰 → main 반영
              → 테스트 성공 시 이미지 빌드 → Harbor 게시
              → Image Updater가 매니페스트 참조 갱신
              → 팀이 결정할 검토·반영 절차
              → Argo CD가 Git과 EKS를 비교·동기화
              → EKS에서 이미지 실행
```

이미지는 앱 실행에 필요한 파일을 묶은 배포 단위이고, 매니페스트는 Kubernetes가 어떤 이미지를 어떻게 실행할지 적은 설정입니다. Harbor는 이미지 저장소이고, Argo CD는 Git에 기록된 설정을 Kubernetes 상태와 비교하는 도구입니다.

2026-11-09 확인 당시 네 CI PR은 모두 **미병합**이었습니다. FE·BE·AI는 Draft, 매니페스트 PR은 Open 상태였습니다. 상태는 바뀔 수 있으므로 각 PR 링크에서 최신 상태를 확인하세요. FE의 기존 GitHub Pages 워크플로는 별도로 존재합니다. 현재 준비한 새 CI가 기본 브랜치에 이미 적용되었다고 가정하지 않습니다.

## 2. 현재 작성된 CI 초안

### 앱 저장소 공통

- `develop`·`main`을 대상으로 한 PR과 두 브랜치에 대한 push에서 각 저장소의 앱 검사 워크플로가 실행되도록 작성했습니다.
- PR에서는 이미지를 게시하지 않습니다. `main` push에서 **같은 워크플로의 선행 검사 Job이 성공한 경우에만** 이미지 빌드·Harbor 게시 Job을 실행하도록 작성했습니다.
- 별도 `Commit policy / commit-policy` 워크플로는 PR 제목과 해당 PR의 비병합 커밋 제목에 `타입: 변경 내용` 형식을 검사합니다. PR 생성·커밋 추가·PR 제목 수정 등에서 다시 실행됩니다.
- 커밋 제목 검사는 PR 전용 **별도 워크플로**로, `main` push의 이미지 Job이 `needs`로 직접 기다리는 대상이 아닙니다. 팀 리뷰 및 사용 가능한 브랜치 보호 정책과 함께 운영해야 합니다.
- Harbor 이미지 경로는 `harbor.skala-gj.com/g4-4/frontend`, `backend`, `ai`이고 태그는 해당 코드의 Git 커밋 SHA를 사용하는 **CI 초안**입니다. 실제 게시 권한·네트워크·태그 정책은 아직 검증해야 합니다.

### 저장소별 검사와 확인 범위

| 대상 | 현재 PR의 Job과 상태 | 유의할 점 |
| --- | --- | --- |
| FE | Node.js 22 설치·빌드와 기존 Python 회전 가이드 테스트가 별도 Job으로 실행됩니다. 두 Job은 병렬 실행이 가능하며, `main` 이미지 Job은 둘 다 기다립니다. 당시 GitHub 검사 통과를 확인했습니다. | 기존 Python 테스트는 UI 전체나 실제 AI 모델 연동 테스트가 아닙니다. 린트와 UI 테스트는 별도 준비가 필요합니다. |
| BE | Java 21·Gradle 빌드/테스트, 실제 실행된 테스트 유무를 확인합니다. 이미지 Job은 이 검사를 기다립니다. | 기준 브랜치에 Gradle Wrapper와 앱·테스트가 없어 PR 앱 검사가 실패했습니다. 실제 백엔드 코드 준비 후 다시 확인합니다. |
| AI | Python 3.11 `unittest`와 실행된 테스트 유무를 확인합니다. 이미지 Job은 이 검사를 기다립니다. | 기준 브랜치에 `tests/`가 없어 PR 앱 검사가 실패했습니다. 별도 실험 브랜치의 테스트 성공은 기준 브랜치 서비스 검증이 아닙니다. |
| Manifest | YAML을 파싱하고 `manifests/kustomization.yaml`이 있으면 오프라인으로 Kustomize 결과와 기본 필드를 확인합니다. | `manifests/`는 **제안 경로**이며 팀 합의로 바꿀 수 있습니다. 현재 실제 파일이 없어 검사가 실패합니다. 전체 Kubernetes 스키마·클러스터·배포 확인은 아닙니다. |

각 PR 브랜치의 세부 준비 계획: [FE](https://github.com/SKALA-CLX/clx-fe/blob/ci/frontend-checks/.github/CI_PLAN.md), [BE](https://github.com/SKALA-CLX/clx-be/blob/ci/backend-ci-setup/.github/CI_PLAN.md), [AI](https://github.com/SKALA-CLX/clx-ai/blob/ci/python-checks/.github/CI_PLAN.md), [Manifest](https://github.com/SKALA-CLX/clx-manifest/blob/ci/manifest-validation/.github/CI_PLAN.md). 브랜치 병합·삭제 후에는 해당 저장소의 기본 브랜치 또는 PR에서 최신 파일을 확인하세요.

PR에 표시되는 검사 결과와 기본 브랜치에 설치된 워크플로는 구분합니다. FE에는 별도의 로컬 컨테이너 초안이 준비됐지만 아직 기준 브랜치에 반영된 것은 아닙니다. BE·AI의 임시 응답 서버는 연결 시험용이며 실제 프로젝트 코드나 테스트를 대체하지 않습니다.

## 3. Job 실행 순서와 검사 계획

독립적인 검사 Job은 병렬 실행할 수 있습니다. 이미지 생성처럼 선행 결과가 필요한 Job은 **같은 워크플로**에서 `needs`를 지정해 기다리도록 구성합니다. 예를 들어 FE의 이미지 게시 Job은 `frontend-build`와 `rotation-rag-tests` 둘 다 성공해야 실행됩니다. `needs`만으로 다른 워크플로의 완료를 직접 기다릴 수는 없습니다. [GitHub Actions Job 설명](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-jobs)

팀이 준비할 추가 검사는 아직 **제안**입니다.

| 종류 | 예시 | 현재 상태 |
| --- | --- | --- |
| 정적 코드 품질 검사 | 코드 문법·형식·규칙 점검 | 도구와 규칙 미결정. FE 린트 명령도 아직 없음 |
| 동적 단위·통합 테스트 | 실제 함수·API를 실행해 결과 확인 | 코드와 테스트 환경에 맞춰 추가 예정 |
| 정적 보안·의존성 검사 | 코드 패턴과 의존성의 알려진 취약점 확인 | 도구·범위·실패 기준 미결정 |
| 동적 보안 검사(DAST) | 실행 중인 테스트 서비스를 대상으로 보안 동작 확인 | 승인된 격리 환경·대상·인증 방식 준비 후 검토 |
| GPU·실제 모델 검증 | 모델과 GPU가 필요한 AI 동작 확인 | GPU 별도 요청과 승인, 실행 환경·비용 확인 후 검토 |

일반 린트·단위 테스트를 실행했다고 정적·동적 **보안** 분석까지 끝난 것은 아닙니다. 실행할 코드가 없는 상태에서 빈 테스트나 항상 성공하는 Job으로 준비 완료처럼 표시하지 않습니다.

파일 경로에 따라 필요한 검사만 실행하는 방식은 추후 검토할 수 있습니다. 필수 검사에 지정된 워크플로가 경로 필터로 건너뛰어지면 PR에서 대기 상태로 남을 수 있어, 조건과 브랜치 보호 정책을 함께 검증해야 합니다. [GitHub Actions 실행 조건](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow)

## 4. 리뷰·필수 검사와 이미지 게시의 관계

팀 작업은 작업 브랜치 → `develop` PR·팀원 리뷰·squash 병합 → `develop`에서 `main`으로 별도 PR·리뷰·merge commit을 기본으로 합니다. CI 검사 실행 여부, 팀원 승인과 보호 브랜치 설정은 각각 별도입니다. PR에서 검사에 실패해도 해당 검사를 필수로 지정할 수 있는 보호 정책이 실제로 적용되지 않았다면 자동 병합 차단을 보장할 수 없습니다.

이전에 확인한 BE·AI·manifest 비공개 저장소 화면에는 현재 조직 요금제에서 ruleset이 강제되지 않는다는 안내가 있었습니다. 조직에서 상속된 규칙과 현재 요금제·설정은 관리자와 다시 확인합니다. 확인 전에는 팀이 실패한 검사를 확인하고 병합을 보류합니다. PR에서 의도적으로 건너뛰는 이미지 게시 Job을 PR의 필수 검사로 지정하지 않습니다.

제목 검사에서 병합 커밋은 제외하지만 `develop → main` PR에 포함된 과거 작업 커밋은 검사할 수 있습니다. 형식이 다른 과거 이력 때문에 실패하면 팀과 해결 방법을 논의합니다. GitHub의 squash 병합 화면에서는 최종 제목을 수정할 수 있으므로 리뷰어가 그 제목도 확인합니다.

## 5. Harbor와 배포 준비

- Dockerfile·의존성·실제 테스트, Harbor 프로젝트의 push 권한, GitHub Actions 실행 환경의 네트워크 접근을 준비한 뒤 이미지 게시를 검증합니다. 현재 수동 Harbor push는 인증 후에도 권한 거부로 실패했습니다. FE 실제 화면 및 BE·AI 임시 서버의 로컬 이미지는 준비·실행 확인했지만 Harbor에 올라간 결과나 EKS 실행 성공을 뜻하지 않습니다.
- 비밀번호, 키, 토큰, 개인 정보는 이 저장소, 이슈·PR, 이미지와 로그에 넣지 않습니다. 앱 CI 초안은 GitHub Actions Secrets 이름 `HARBOR_USERNAME`, `HARBOR_PASSWORD`를 참조하며, 실제 값의 설정 여부와 담당자는 별도로 확인합니다.
- 목표 대상은 EKS `skala-gj`의 `g4-4` 네임스페이스입니다. 캠퍼스 네트워크 조건, 운영 시간·예외, 팀 쿼터와 권한은 실행 전에 확인합니다. 팀 도메인의 기존 경로와 다른 팀원이 사용하는 리소스를 덮어쓰지 않습니다.
- Argo CD Image Updater의 설치 여부·버전·Harbor 읽기·Git 쓰기·PR 방식 지원, Argo CD 앱 등록과 동기화 정책은 미확인입니다. 도구 이름이 다이어그램에 있다고 현재 작동한다고 가정하지 않습니다.
- 이미지 태그의 불변성, digest 사용 여부, 동시에 빌드된 이미지의 선택 순서와 롤백 방법을 정합니다. 먼저 업로드된 이미지가 항상 최신 코드라는 보장은 없습니다.

## 6. 다음 순서

1. 네 CI PR의 현재 상태를 확인하고 팀이 검토합니다. PR 상태와 검사 결과는 수시로 바뀔 수 있습니다.
2. FE의 빌드·기존 테스트 범위를 확인하고, BE·AI에는 실제 프로젝트와 의미 있는 테스트를 준비합니다. Manifest는 실제 서비스 이미지·포트·설정 경로를 합의한 뒤 파일과 검사 범위를 함께 추가합니다.
3. 각 서비스에 필요한 정적 검사·동적 테스트 도구와 실패 기준을 결정하고, 테스트 누락 및 의도적인 오류가 실제로 실패하는지 확인합니다.
4. Dockerfile과 Harbor push 권한·runner 네트워크를 확인하고 이미지 빌드·업로드를 실제로 검증합니다.
5. 운영 담당자와 Image Updater·Argo CD·EKS의 접근 및 동기화 방식을 확인합니다. 팀이 허용한 테스트 환경에서 단계별로 검증합니다.
6. 검사와 리뷰가 실제로 준비된 후, 사용 가능한 GitHub 보호 정책을 확인해 필요한 체크만 필수로 지정합니다.

담당자와 확인 날짜, 검증 결과가 달라지면 이 문서를 갱신하세요. 적용 결과를 기록할 때는 검증한 범위와 남은 한계를 함께 적습니다.
