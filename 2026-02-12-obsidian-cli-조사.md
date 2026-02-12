---
date: 2026-02-12
tags: [obsidian, cli, research]
---

# Obsidian CLI 및 릴리즈 노트 조사

## 개요
Discord에서 Obsidian CLI 정보 공유 및 최신 릴리즈 노트 조사

## 주요 내용

### 1. Obsidian CLI (커뮤니티 도구)
- **도구명:** `obsidian-cli` (yakitrak 개발)
- **설치:** `brew install yakitrak/yakitrak/obsidian-cli`
- **주요 기능:**
  - `obsidian-cli search "query"` - 노트 제목 검색
  - `obsidian-cli search-content "query"` - 노트 내용 검색
  - `obsidian-cli create "Folder/New note" --content "..."` - 노트 생성
  - `obsidian-cli move "old/path" "new/path"` - 링크 자동 업데이트하며 이동
  - `obsidian-cli delete "path/note"` - 노트 삭제

### 2. 공식 Obsidian CLI (v1.12.0+ 출시)
- **최신 버전:** Desktop v1.12.1
- **사용법:** `obsidian <command> [options]`
- **주요 명령어:**
  - `obsidian open <vault>` - vault 열기
  - `obsidian open <vault> <file>` - 특정 파일 열기
  - `obsidian daily <vault>` - Daily note 열기
  - `obsidian daily:prepend <vault> "<content>"` - Daily note 앞에 내용 추가
  - `obsidian search <vault> "<query>"` - vault 내 검색
  - `obsidian new <vault> "<path>"` - 새 노트 생성
  - `obsidian vault:list` - 등록된 vault 목록

### 3. v1.12.0 릴리즈 하이라이트
- **공식 CLI 출시** 🎉 - 터미널에서 Obsidian 제어 가능
- 이미지 크기 드래그 조절 (Live Preview)
- 파일 삭제 시 첨부파일 함께 삭제 옵션
- 파일 탐색기에서 Ctrl-C/Ctrl-V 지원
- Canvas 백링크 감지 및 그래프 뷰 연동
- 언어 선택기 개선

## 참고 링크
- [Obsidian 공식 CLI 문서](https://help.obsidian.md/cli)
- [obsidian-cli GitHub](https://github.com/yakitrak/obsidian-cli)
- [Obsidian Changelog](https://obsidian.md/changelog)

## 메모
- 커뮤니티 CLI는 링크 자동 업데이트가 강점
- 공식 CLI는 스크립팅/자동화에 최적화
- 두 도구 모두 vault 경로는 `~/Library/Application Support/obsidian/obsidian.json`에서 확인

---

# 세션 진행 기록

## 4. GitHub 동기화 및 토큰 관리

### 4.1 레포 생성
- 레포 URL: https://github.com/vobcodee/obsidian-vault
- 설명: Obsidian vault synced from OpenClaw workspace
- 초기 파일: README.md, obsidian-cli 조사.md, 요리 예능 문법 분석.md

### 4.2 GitHub 토큰 보안 이슈
- Discord에 PAT 토큰 공유됨 (보안 위험)
- 토큰 재발급 권장 → 사용자가 새 토큰 발급 완료
- VPS 환경변수 `GITHUB_TOKEN`으로 설정

### 4.3 권장 토큰 Scope
| Scope | 설명 | 필수 |
|-------|------|------|
| `repo` | 전체 repo 접근 | ✅ |
| `workflow` | Actions workflow 업데이트 | ✅ |
| `gist` | Gist 접근 | ⚪ |
| `pages` | Pages 관리 | ⚪ |

### 4.4 환경변수 설정 완료
```bash
export GITHUB_TOKEN="ghp_..."
GH_TOKEN=$GITHUB_TOKEN gh auth status
# → github.com ✓ Logged in
```

## 5. 향후 작업
- vault 파일 수정 시 `git commit && git push`로 동기화
- Obsidian에서 GitHub 레포 클론 후 사용
