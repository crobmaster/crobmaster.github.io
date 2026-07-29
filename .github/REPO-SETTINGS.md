# 저장소 접근 권한 설정 체크리스트

이 사이트는 GitHub Free 플랜의 사용자 Pages 사이트라 **저장소가 public이어야만** 운영된다.
public을 유지하면서 "사이트 열람은 누구나, 나머지는 나만" 상태를 만들기 위한 설정 기록.

설정 기준일: 2026-07-29 / 저장소: `crobmaster/crobmaster.github.io`

> 이 파일은 `.github/` 아래에 있어 Jekyll 기본 규칙상 사이트에 게시되지 않는다.
> (점으로 시작하는 디렉터리는 빌드에서 제외됨)

---

## 전제: 이미 안전한 것

Collaborator가 `crobmaster` 한 명(admin)뿐이므로 **push / merge / 브랜치 삭제 / 설정 변경은
public이든 아니든 본인만 가능**하다. public이 외부에 열어주는 것은 다음 네 가지뿐이다.

1. 소스·커밋·이슈 **열람**
2. 이슈 **생성**
3. fork 후 PR **생성**
4. 이슈/PR **댓글**

아래 설정은 2~4번을 닫는 작업이다. 1번은 public인 이상 닫을 수 없다.

---

## 체크리스트

### 1. Settings → General → Features

- [ ] **Issues** 드롭다운 → `Collaborators only`
      (외부인은 열람·댓글만 가능, 신규 생성 불가. 본인은 그대로 사용)
- [ ] **Pull requests** 드롭다운 → `Collaborators only`
      (fork 자체는 막을 수 없지만 PR 제출은 차단됨)
- [ ] **Wikis** 체크 해제 — 미사용
- [ ] **Projects** 체크 해제 — 미사용
- [x] Discussions — 원래 꺼져 있음

개인 저장소에서 "collaborator" = 초대받은 사용자. 현재 본인 1명뿐이다.

### 2. Settings → Rules → Rulesets — `main` 브랜치

새 branch ruleset, target = `main`:

- [ ] **Restrict deletions** 켜기
- [ ] **Block force pushes** 켜기
- [ ] Require a pull request — **켜지 않음**. 1인 운영에 마찰만 늘고,
      "브랜치에서 작업 → 본인이 머지"하는 현재 흐름이면 불필요하다.
- [ ] **Bypass list는 비워둘 것.** admin을 우회 목록에 넣으면 실수 방지라는 목적이 사라진다.

### 3. Settings → Actions → General

- [ ] *Fork pull request workflows from outside collaborators*
      → `Require approval for all external contributors`
      (public 저장소 기본값은 "first-time contributors"라 더 느슨하다)
- [ ] *Workflow permissions* → `Read repository contents and packages permissions` (읽기 전용)
- [ ] ⚠️ **Actions 자체를 Disable 하지 말 것** — Pages 배포(`pages-build-deployment`)가
      Actions 인프라 위에서 돌기 때문에 사이트 배포가 깨진다.

### 4. Settings → Moderation → Interaction limits — 지금은 보류

`Limit to repository collaborators`는 외부인의 **기존 이슈 댓글**까지 막는다.
다만 최대 6개월이라 주기적 갱신이 필요하다.
현재 외부 유입이 0건이므로 켜지 않고, 스팸이 실제로 들어오면 그때 적용한다.

- [ ] (필요 시) Interaction limits 적용 — 적용했다면 만료일 기록: `____-__-__`

### 5. 계정 보안 — 저장소 설정보다 상위 위험

저장소를 아무리 조여도 계정이 뚫리면 무의미하다.

- [ ] Settings → Password and authentication → **2FA** 활성 확인
- [ ] Settings → Developer settings → **Personal access tokens** 점검, 미사용 토큰 폐기
- [ ] Settings → Applications → **Authorized OAuth Apps / GitHub Apps** 점검
- [ ] 저장소 Settings → **Deploy keys** 비어 있는지 확인
- [ ] 저장소 Settings → Pages → **Enforce HTTPS** 켜짐 확인

---

## 할 수 없는 것 (다시 시도하지 말 것)

- **Fork 차단 불가** — public 개인 저장소에는 fork 비활성화 옵션이 없다.
  (org 저장소이거나 private일 때만 가능)
- **소스 열람 차단 불가** — public이므로 당연하지만, private로 바꿔도 이 사이트는 정적 HTML이라
  브라우저 "소스 보기"로 `index.html`·`privacy-policy.html` 내용이 그대로 보인다.
  숨겨서 얻는 실익은 커밋 메시지와 이슈 내용 정도다.
- **사이트 자체에 접근 제한 불가** — Pages 사이트에 인증을 거는 기능은
  GitHub Enterprise Cloud 전용. Pro로 private 전환해도 사이트는 여전히 공개된다.
- **참고**: private 저장소로 Pages를 운영하려면 GitHub Pro(월 $4) 이상이 필요하다.
  위 이유로 현재는 public 유지를 선택했다.

---

## 적용 후 확인

1. **로그아웃 상태(시크릿 창)로** `https://github.com/crobmaster/crobmaster.github.io` 접속:
   - Issues 탭의 **New issue** 버튼이 사라졌는가
   - **Wiki / Projects** 탭이 사라졌는가
   - 기존 이슈는 여전히 **열람 가능**한가 (의도된 동작)
2. 사이트 정상 동작 — `https://crobmaster.github.io` 및
   `https://crobmaster.github.io/tap-tap-picture-book/privacy-policy.html` 로드 확인
3. 이 문서가 노출되지 않는지 —
   `curl -s -o /dev/null -w "%{http_code}" https://crobmaster.github.io/.github/REPO-SETTINGS.md`
   → **404**여야 정상. 200이면 이 파일을 저장소에서 지우고 로컬 메모로만 관리할 것.
4. 다음 커밋 후 Actions 탭에서 `pages-build-deployment`가 성공하는지 확인
