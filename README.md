# replworks/apt

[![publish](https://github.com/replworks/apt/actions/workflows/publish.yml/badge.svg)](https://github.com/replworks/apt/actions/workflows/publish.yml)

`https://apt.repl.net` 로 서비스되는 replworks APT 저장소.
각 프로젝트가 GitHub Release에 붙인 `.deb`를 모아서 서명하고 GitHub Pages로 배포한다.

## 동작 방식

1. 각 프로젝트가 goreleaser로 `.deb`를 만들어 자기 Release에 붙인다.
2. 릴리스가 끝나면 프로젝트가 이 저장소로 `repository_dispatch`(`release`)를 보낸다. 이때 자기 저장소 이름(`client_payload.repo`)을 같이 실어 보낸다.
3. 이 저장소의 `publish.yml`이 그 저장소가 `packages.txt`에 없으면 **자동으로 추가하고 커밋**한다. (`replworks/` 아래 저장소만 허용)
4. `packages.txt`에 적힌 저장소들의 최신 Release에서 `.deb`를 받는다.
5. `Packages` / `Release` 메타데이터를 만들고 GPG로 서명한 뒤 Pages에 배포한다.

## 구조

```
.
├── packages.txt                  # 배포할 저장소 목록 (owner/repo 한 줄씩, 자동으로 채워짐)
└── .github/workflows/publish.yml # 자동 등록 + 메타데이터 생성 + 서명 + Pages 배포
```

## 새 저장소(패키지) 추가하는 법

이 저장소는 건드릴 필요 없다. 추가할 프로젝트 쪽에서 아래 3가지만 하면 첫 릴리스 때 `packages.txt`에 자동 등록된다.
추가할 프로젝트를 `replworks/repo`라고 하자.

### 1. `.goreleaser.yaml`에 `nfpms` 추가

```yaml
nfpms:
  - package_name: <패키지 이름>
    homepage: "https://github.com/replworks/<repo>"
    maintainer: "cable8mm <you@example.com>"
    description: "<설명>"
    license: MIT
    formats: [deb]
```

`goreleaser release --snapshot --clean` 으로 로컬에서 돌려서 `dist/`에 `.deb`가 생기는지 확인한다.

### 2. `APT_DISPATCH_TOKEN` 시크릿 등록

- organization 시크릿으로 한 번 등록해뒀으면 그 저장소에 접근 권한만 열어주면 된다.
- 토큰을 새로 만들 때: fine-grained PAT, Repository access는 `replworks/apt`만, Permissions는 Contents → Read and write.

### 3. `release.yml`의 goreleaser 스텝 바로 뒤에 알림 스텝 추가

```yaml
- name: Notify apt repo
  env:
    GH_TOKEN: ${{ secrets.APT_DISPATCH_TOKEN }}
  run: |
    gh api repos/replworks/apt/dispatches \
      -f event_type=release \
      -f "client_payload[repo]=$GITHUB_REPOSITORY"
```

`$GITHUB_REPOSITORY`가 자동으로 들어가니까 프로젝트마다 고칠 곳은 없다.

### 4. 확인

- 프로젝트에 새 태그를 푸시한다.
- 이 저장소 Actions 탭에서 `publish`가 돌았는지, `packages.txt`에 `Add replworks/<repo>` 커밋이 생겼는지 본다.
- `https://apt.repl.net/dists/stable/InRelease` 가 열리는지 확인한다.

> 이미 만들어진 Release로 바로 등록하고 싶으면 `packages.txt`에 `replworks/<repo>` 한 줄을 직접 추가하고 `publish`를 **Run workflow**로 수동 실행해도 된다.
> 수동 실행(`workflow_dispatch`)은 payload가 없어서 자동 등록은 일어나지 않는다.

## 사용자 설치 방법

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.repl.net/replworks.gpg | sudo tee /etc/apt/keyrings/replworks.gpg >/dev/null
echo "deb [signed-by=/etc/apt/keyrings/replworks.gpg] https://apt.repl.net stable main" | sudo tee /etc/apt/sources.list.d/replworks.list
sudo apt update && sudo apt install <패키지 이름>
```

## 한 번만 해둔 설정 (참고)

- **GPG 키**: 이 저장소 Actions 시크릿 `APT_GPG_PRIVATE_KEY`, `APT_GPG_PASSPHRASE`
- **Pages**: Settings → Pages → Source는 **GitHub Actions**, Custom domain은 `apt.repl.net`, Enforce HTTPS 켜둠
- **DNS**: `apt` CNAME → `replworks.github.io`
- **워크플로 권한**: `publish.yml`의 `contents: write` (`packages.txt` 자동 커밋용)

## 문제 생겼을 때

| 증상                                  | 확인할 것                                                               |
| ------------------------------------- | ----------------------------------------------------------------------- |
| `publish`의 다운로드 단계 실패        | 대상 저장소가 public인지, Release에 `.deb`가 붙어 있는지                |
| dispatch를 보냈는데 `publish`가 안 돎 | `publish.yml`이 이 저장소 **기본 브랜치**에 있는지, 토큰 권한과 만료일  |
| 자동 등록 스텝에서 `Invalid repo`     | 저장소 이름이 `replworks/`로 시작하는지                                 |
| 자동 등록 스텝의 `git push` 실패      | main 브랜치 보호 규칙(PR 필수 등)이 봇 push를 막는지                    |
| `apt update`에서 서명 오류            | `replworks.gpg`를 다시 받았는지, 시크릿의 GPG 키가 맞는지               |
| 새 패키지가 안 보임                   | `packages.txt`에 등록됐는지, 해당 Release에 amd64/arm64 `.deb`가 있는지 |

## 제한

- 각 저장소의 **최신 Release**의 `.deb`만 배포한다. 옛날 버전을 고르게 하려면 구조를 바꿔야 한다.
- GitHub Pages 소프트 제한: 용량 1GB, 대역폭 월 100GB.
