---
title: "Github TAG & Release AutomationTAG & Release Automation"
date: "2023-06-26"
tags: ["conventional commits", "worflows", "github", "github actions", "node.js"]
---

## 깃허브 워크플로우를 사용한 앱 버젼 관리

`.github/workflows` 경로의 들어가는 `yaml`, `yml` 파일은 깃허브 레포지토리에 업로드 되는 커밋의 내역을 통해 CI (Continuous Integration)을 진행할 수 있다.

1. Tags
2. Release

---

### 1. Tags
- 특정 커밋을 기록하는 역할
- 작업하다 보면 수백개의 커밋이 쌓이면, 그 중 중요한 커밋의 "save point"를 기록한다
- Tag는 읽기전용 (readonly) 이머 수정이 불가능하다

##### Semantic Versioning
- 주로 소프트웨어의 버전을 "Release" 할 때 사용
- `v` 이후 `.`으로 분리된 형식 `v<MAJOR>.<MINOR>.<PATCH>` 형태로 저장되먄 각 숫자는 주로 아래와 같이 커밋 내용에 따라 관리한다 (Semantic Versioning)

	**MAJOR**
	- 새로운 & 큰 변화가 있는 경우
	- `v1.0.0` -> `v2.0.0`
	
	**MINOR**
	- 새로운 부가적인 기능이 추가되는 경우
	- `v1.0.0` -> `v1.1.0`
	
	**PATCH**
	- 버그 수정 및 유지보수
	- `v1.0.0` -> `v1.0.1`


#### Tag Automation

❗️ 프로젝트의 규모가 커질수록 본인을 포함한 팀원들의 커밋이 많아지며 이를 수작업으로 관리하기 힘들다
이를 해결하기 위해 커밋 내역에 따라 버젼관리를 하고 태그를 생성시켜주게 되면 관리가 수월해진다.

##### [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
해당 스펙에서 제시하는 safelist 된 prefix를 사용하하여 커밋 컨벤션을 맞추면 많은 장점이 생긴다
- 컨벤션이며 커밋 내용을 카테고리화해준다
- 많은 개발자들이 사용하는 글로벌한 컨벤션이며 이를 활용하여 더 쉽게 자동화된 도구를 만들 수 있다
- 🚀 더 좋은 점은 이미 잘 만들어진 자동화 도구가 많다


**[TriPS/conventioanl-changelog-action](https://github.com/TriPSs/conventional-changelog-action)**
- `Conventional Commit`을 확인하여 `CHANGELOG.md` 파일을 만든다
- `CHANGELOG.md` 를 확인하여 태그 버전을 `Semantic Versioning`에 맍게 업데이트
- 업데이트된 버젼을 프로젝트의 버전 정보를 가지고 있는 파일(default: `package.json`) 업데이트 후 커밋
- 해당 액션은 `node.js`환경을 기준으로 기본 설정값을 가지고 있기 때문에 특수 케이스가 아닌 이상 추가 설정을 크게 건드릴 필요가 없다


**🤯 특수케이스 예시 :**
```yml
name: Create Tag

on:
	pull_request:
		branches:
			- dev
		types:
			- synchronize
			- opened
  
jobs:
	create-tag:
		runs-on: ubuntu-latest
  
steps:
	- name: Checkout Pull Request Branch
		uses: actions/checkout@v3
		with:
		ref: ${{ github.event.pull_request.head.ref }}

  
	- name: Conventional Changelog Action
		id: changelog
		uses: TriPSs/conventional-changelog-action@v3.19.0
		with:
			github-token: ${{ secrets.GITHUB_TOKEN }}
			git-branch: ${{ github.event.pull_request.head.ref }}
```

- `git-branch` 설정은 태그 업데이트 후 버전 업데이트의 대한 커밋을 할 브랜치를 지정한다
- `pull_request` 경우 `default`브랜치는 `base` 브랜치인 `dev` 가 되는데 `dev`브랜치가 `protection rule`이 설정되어있다면 액선 내부에서 바로 커밋을 진행할 수 없다.
- 그렇기 때문에 `head`브랜치에 커밋을 하고 `pull_request`가 머지될 때 `dev`브랜치에 커밋이 된다.

---

<br>

### 2. Release
기록의 특정 지점을 표시하는 Git 태그를 기반으로 공실 릴리즈 내역을 만든다

**[release-drafter](https://github.com/release-drafter/release-drafter)**
이전 릴리즈까지의 Pull Request중 라벨링을 사용하여 버젼관리 및 내역을 정리할 수 있다


```yml
name: Create Tag on Push Dev

on:
	push:
		branches:
			- dev
  
jobs:
	create-tag:
		runs-on: ubuntu-latest

steps:
	uses: actions/checkout@v3
	
	 - name: Get Version
		id: get_version
		run: |
			VERSION=$(node -p "require('./package.json').version")
			echo $VERSION
			echo "::set-output name=version::$VERSION"
  
	- name: Create Draft
		id: release_drafter
		uses: release-drafter/release-drafter@v5
		with:
			config-name: release-drafter.yml
			version: ${{ steps.changelog.outputs.tag }}
			tag: ${{ steps.changelog.outputs.tag }}
			publish: false
			prerelease: false
		env:
			GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- 나같은 경우 `dev`브랜치로 merge를 하기 전에 버젼 업데이트와 태그를 생성했기 때문에 업데이트 된 버젼을 `package.json` 안에서 가져온다
- 구체적인 release 내역 (`release.body`), 버젼관리, 사용될 라벨 설정 등은 내용이 길기 때문에 `release-drafter.yml`파일을 분리해서 `config-name`인풋으로 가져와 관리한다 (문서에서도 이러한 방식을 추천)
- 문서가 친철히 설명을 하니 직접 보는 것을 추천

**release-drafter.yml**
```yml
name-template: 'v$RESOLVED_VERSION 🌈'
tag-template: 'v$RESOLVED_VERSION'
categories:
  - title: '🚀 Features'
    labels:
      - 'feature'
      - 'enhancement'
  - title: '🐛 Bug Fixes'
    labels:
      - 'fix'
      - 'bugfix'
      - 'bug'
  - title: '🧰 Maintenance'
    label: 'chore'
change-template: '- $TITLE @$AUTHOR (#$NUMBER)'
change-title-escapes: '\<*_&' # You can add # and @ to disable mentions, and add ` to disable code blocks.
version-resolver:
  major:
    labels:
      - 'major'
  minor:
    labels:
      - 'minor'
  patch:
    labels:
      - 'patch'
  default: patch
template: |
  ## Changes

  $CHANGES
```
