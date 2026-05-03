# Factory Atlas — 보안 강화 작업 계획서

폐쇄형 멤버십 · 워터마킹 · 행동 로그 단계적 도입

---

## 배경과 목적

Factory Atlas는 디자이너 풀이 함께 가공 업체 정보를 축적하는 비공식 협업 플랫폼입니다. 현재는 누구나 URL만 알면 접근 가능한 상태이며, 작성자 식별은 사용자가 직접 입력한 이름(alias)에만 의존합니다. 가공 업체 정보는 실질적으로 상업적 가치가 있는 자산이므로, 외부 유출 방지와 멤버십 통제가 필요합니다.

본 계획은 다음 세 가지를 단계적으로 도입합니다.

1. 폐쇄형 멤버십 — 운영자가 초대한 사람만 접근
2. 워터마킹 — 화면에 사용자 식별자를 표시해 유출 추적 가능
3. 행동 로그 — 누가 언제 무엇을 했는지 기록

기술적 차단이 아닌 사회적·법적 억제력을 만드는 것이 핵심 방향입니다.

---

## 전제 조건

- 현재 GitHub Pages에 배포된 Factory Atlas가 정상 작동 중
- Firebase 프로젝트 `factory-atlas-1b5a8`에 Firestore 활성화 상태
- 카카오 지도 API 정상 작동 (atlas 키)
- 사용자분이 운영자(admin) 역할을 단독으로 수행

---

## Phase 1 — 폐쇄형 멤버십 (인증 + 화이트리스트)

### 1.1 Firebase Authentication 활성화

Firebase Console에서 다음 작업이 선행되어야 합니다.

- Authentication 메뉴 진입 → Sign-in method 탭
- Google 로그인 활성화 (가장 마찰이 적음)
- 선택사항: 이메일/비밀번호 로그인 추가 활성화 (Google 계정 없는 디자이너 대비)

### 1.2 Firestore에 화이트리스트 컬렉션 생성

새 컬렉션 `members`를 생성하고, 각 문서를 다음 구조로 정의합니다.

```
members/{email}
  - email: string
  - name: string
  - role: "admin" | "member"
  - invitedAt: timestamp
  - invitedBy: string
  - status: "active" | "suspended"
```

문서 ID는 이메일 주소를 사용 (예: `members/jh@example.com`). 운영자(사용자분)의 이메일은 role: admin으로 직접 입력해 시드합니다.

### 1.3 Firestore 보안 규칙 강화

현재 `allow read, write: if true` 상태를 다음 규칙으로 교체합니다.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // 멤버 본인의 정보만 자기가 읽을 수 있음. 쓰기는 admin만 가능
    match /members/{email} {
      allow read: if request.auth != null && request.auth.token.email == email;
      allow write: if false; // admin도 콘솔에서 직접만 변경
    }

    // 인증된 멤버만 facilities 읽기/쓰기 가능
    match /facilities/{docId} {
      allow read: if isMember();
      allow create: if isMember();
      allow update: if isMember();
      allow delete: if isMember();
    }

    function isMember() {
      return request.auth != null
        && exists(/databases/$(database)/documents/members/$(request.auth.token.email))
        && get(/databases/$(database)/documents/members/$(request.auth.token.email)).data.status == "active";
    }
  }
}
```

핵심: 화이트리스트(members)에 등록되어 있고 status가 active인 사용자만 facilities를 읽고 쓸 수 있습니다. 비멤버는 데이터를 아예 받지 못합니다.

### 1.4 클라이언트 코드 변경

`index.html`의 변경 범위:

- Firebase Auth SDK 추가 임포트
- 진입 시 인증 상태 확인 → 미인증이면 로그인 화면 표시
- 로그인 성공 후 화이트리스트 확인 → 멤버 아니면 차단 화면 ("초대받지 않은 계정입니다. 운영자에게 문의하세요.")
- 멤버 확인되면 기존 지도 화면 진입
- alias 시스템 제거 → 인증된 사용자의 email/name을 자동으로 작성자로 사용
- createdBy 필드를 email로 변경 (기존 데이터는 마이그레이션 필요 — 별도 처리)
- 우측 하단 DESIGNER 표시를 인증된 사용자의 name으로 자동 표시
- 로그아웃 버튼 추가 (DESIGNER 클릭 시 메뉴 표시)

### 1.5 운영자 — 멤버 초대 워크플로우

별도 admin 페이지를 만들지 않고, 초기에는 사용자분이 Firebase Console에서 직접 members 컬렉션에 문서를 추가하는 방식으로 운영합니다.

새 디자이너 추가 절차:
1. Firebase Console → Firestore → members 컬렉션
2. 문서 ID에 디자이너 이메일 입력
3. 필드 추가: email, name, role: "member", status: "active", invitedAt (현재 시각), invitedBy (사용자분 이메일)
4. 디자이너에게 사이트 URL 공유 → 본인이 Google 로그인하면 자동 진입

멤버 풀이 30명+ 넘어가면 그때 admin 페이지 신설 검토.

---

## Phase 2 — 워터마킹

Phase 1이 안정화된 후 진행. 인증 정보가 있어야 워터마크에 의미가 생기므로 순서가 중요합니다.

### 2.1 워터마크 구현 방식

`index.html`에 전역 워터마크 레이어를 추가합니다.

- 화면 전체를 덮는 `position: fixed` 오버레이
- `pointer-events: none`으로 클릭 이벤트는 통과
- 사용자 이메일 + 현재 시각을 반복 패턴으로 깔기
- opacity 0.04~0.08 수준 (육안으로는 거의 안 보이지만 스크린샷에는 박힘)
- z-index를 매우 높게 설정 (모달, 상세 패널 위에도 떠야 함)
- 색상은 다크 톤에 맞춰 흰색 또는 액센트 색상 매우 흐리게

### 2.2 워터마크 콘텐츠 예시

```
jh@example.com · 2026.05.04 · jh@example.com · 2026.05.04 ...
```

이메일과 날짜를 함께 박으면 유출 시점까지 특정 가능. 시간 단위로 동적 갱신 (분 단위 정밀도면 충분).

### 2.3 추가 보호 (선택사항, 효과 미미하지만 심리적 억제)

- 우클릭 컨텍스트 메뉴 비활성화
- 인쇄 시 빈 화면 표시 (`@media print`)
- 텍스트 드래그 선택 일부 영역만 차단 (등록 폼은 가능, 상세 패널은 차단)

이 부분은 정상 사용자에게 불편을 줄 수 있어 적용 여부는 사용자분 판단. 추천은 인쇄 차단만.

### 2.4 약관 페이지

별도 페이지(예: `/terms.html`) 또는 첫 로그인 시 모달로 표시:

> "본 플랫폼에 게시된 가공 업체 정보는 Factory Atlas 멤버 간 공유를 목적으로 하며, 외부로 무단 유출, 캡처 공유, 게시할 수 없습니다. 위반 시 멤버십 자격이 즉시 박탈되며, 손해배상의 책임을 질 수 있습니다. 화면 상의 워터마크는 유출 추적을 위해 사용자 식별자가 포함되어 있습니다."

체크박스로 "이용 약관에 동의합니다" 받고 동의해야만 진입. 동의 사실은 members 컬렉션에 acceptedTermsAt 타임스탬프로 기록.

---

## Phase 3 — 행동 로그

Phase 1, 2가 안정화된 후 진행.

### 3.1 로그 컬렉션 설계

새 컬렉션 `activity` 생성:

```
activity/{auto-id}
  - userId: string (이메일)
  - userName: string
  - action: "create_facility" | "update_facility" | "delete_facility"
            | "add_comment" | "delete_comment"
            | "view_facility" | "search" | "login" | "logout"
  - targetId: string | null (facility ID 등)
  - targetName: string | null (가독성용)
  - metadata: object (예: 검색어, 변경 필드 등)
  - timestamp: timestamp
  - userAgent: string
  - ipHash: string | null (선택)
```

### 3.2 로깅 트리거 지점

`index.html`의 변경 범위:

- saveFacility (create/update 분기) — 등록·수정 로그
- deleteFacility — 삭제 로그
- addComment — 댓글 추가 로그
- 사용자가 facility를 선택해서 detail panel을 연 시점 — view 로그 (선택사항, 양이 많아질 수 있음)
- 검색 input에 입력 시 (디바운스 후) — 검색 로그 (선택사항)
- 로그인/로그아웃 시점

각 액션 후 dbLogActivity() 함수를 호출하는 형태로 구현. 비동기 fire-and-forget으로 처리해서 UX 지연 방지.

### 3.3 보안 규칙 추가

```
match /activity/{logId} {
  allow create: if isMember();
  allow read: if isAdmin();
  allow update, delete: if false; // 로그는 불변
}

function isAdmin() {
  return request.auth != null
    && exists(/databases/$(database)/documents/members/$(request.auth.token.email))
    && get(/databases/$(database)/documents/members/$(request.auth.token.email)).data.role == "admin";
}
```

멤버는 자신의 행동 로그를 쓸 수 있지만, 읽거나 수정할 수는 없습니다. 운영자(admin)만 읽기 가능.

### 3.4 운영자 대시보드 (선택사항)

별도 페이지 `/admin.html`을 만들어서:

- 최근 활동 타임라인
- 사용자별 활동 통계 (등록 수, 댓글 수)
- 가장 자주 조회되는 업체
- 의심스러운 패턴 탐지 (단시간 다수 다운로드 시도 등)

이 부분은 Phase 3 후반부 또는 풀 운영이 익숙해진 뒤 추가.

---

## 마이그레이션 — 기존 데이터 처리

현재 등록된 8개 facility는 createdBy 필드가 alias 텍스트로 저장되어 있습니다. Phase 1 도입 시:

- 기존 facilities는 그대로 유지하되, createdBy를 운영자 이메일로 일괄 업데이트
- 또는 createdBy는 그대로 두고, 새로 추가되는 필드 `createdByEmail`만 신규 데이터에 적용
- 기존 댓글의 author 필드도 동일한 처리

Claude Code에 작업 지시할 때 어느 방식을 택할지 명시 필요.

---

## 작업 순서 요약

1. Firebase Console에서 Authentication 활성화 (사용자 작업)
2. Firestore에 members 컬렉션 생성 + 운영자 본인 시드 (사용자 작업)
3. Phase 1 코드 변경 (Claude Code 작업)
   - Firebase Auth 통합
   - 로그인 화면 / 차단 화면 구현
   - alias 시스템 제거
   - 인증된 사용자 정보를 작성자로 사용
4. Firestore 보안 규칙 업데이트 (사용자 작업, Claude가 규칙 텍스트 제공)
5. Phase 1 테스트 — 본인 + 테스트용 비멤버 계정으로 동작 확인
6. Phase 2 — 워터마킹 구현 (Claude Code 작업)
7. 약관 페이지 + 동의 플로우 추가 (Claude Code 작업)
8. Phase 2 테스트
9. Phase 3 — 행동 로그 구현 (Claude Code 작업)
10. 보안 규칙에 activity 컬렉션 룰 추가 (사용자 작업)
11. Phase 3 테스트
12. (선택) admin 대시보드 페이지 신설

---

## Claude Code 지시 예시

각 Phase별로 Claude Code 웹에 다음과 같이 지시:

**Phase 1**:
> PLAN.md의 Phase 1을 구현해줘. Firebase Authentication을 Google 로그인으로 통합하고, members 컬렉션 화이트리스트 확인 로직을 추가해. 미인증/비멤버는 차단 화면을 보여줘. 기존 alias 시스템은 제거하고, 인증된 사용자의 이메일과 이름을 작성자로 사용해. 기존 facilities의 createdBy는 운영자 이메일로 일괄 업데이트하는 별도 마이그레이션 함수도 포함해줘.

**Phase 2**:
> PLAN.md의 Phase 2를 구현해줘. 화면 전체에 사용자 이메일과 현재 시각을 반복 패턴으로 깔되 opacity 0.05 수준으로. pointer-events는 none. 그리고 약관 동의 모달을 첫 로그인 시 표시하고, 동의 시점을 members 문서에 기록해.

**Phase 3**:
> PLAN.md의 Phase 3을 구현해줘. activity 컬렉션에 로그를 쌓는 dbLogActivity 함수를 만들고, create/update/delete/comment 시점에 호출. 비동기 fire-and-forget 처리로 UX 지연 없게. view와 search 로깅도 포함하되 디바운스 처리.

---

## 주의사항

- 스크린샷 자체를 기술적으로 막을 방법은 존재하지 않음. 워터마킹은 추적 가능성을 만드는 것이 목적
- Phase 1 적용 시 이미 사이트를 사용 중인 디자이너가 있다면 미리 공지 필요 (이메일 받아서 화이트리스트에 추가)
- Firestore 보안 규칙 변경은 즉시 반영되므로 코드 배포 전에 규칙을 먼저 바꾸면 기존 사이트가 잠시 동안 작동 중단될 수 있음. 배포 후 규칙 변경 순서 권장
- Firebase Authentication 무료 한도는 월 50,000 사용자로 충분
- 워터마크는 Leaflet 지도 위에도 떠야 하므로 z-index 계층 주의 필요
- 행동 로그 양이 많아질 수 있어, 6개월 이상 된 로그는 자동 삭제하는 정책을 Firebase Functions로 추가하는 것도 검토 가치 있음
