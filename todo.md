# todo.md — 느린 지도: Task List & Progress

> plan.md를 작업 단위로 나눈 체크리스트입니다.
> 완료 기준은 test.md의 해당 ID입니다. 끝나면 `[x]`로 바꾸고 log.md에 한 줄 남깁니다.

**현재 단계:** Phase 0 — 프로젝트 세팅
**진행률:** 0 / 56

---

## Phase 0 — 프로젝트 세팅
- [ ] Next.js(App Router, TS) + Tailwind 프로젝트 생성
- [ ] ESLint·Prettier 설정, 폴더 구조 정리 (CLAUDE.md 참고)
- [ ] Supabase 프로젝트 생성, PostGIS 확장 켜기
- [ ] Supabase 클라이언트(서버·브라우저) 설정
- [ ] 카카오 개발자 앱 등록, Supabase Auth에 카카오 Provider 연결
- [ ] 로그인/로그아웃 동작 (S01)
- [ ] Vercel 연결·배포, 환경변수 등록
- [ ] PWA manifest·아이콘
- [ ] 하단 탭 4개 껍데기 화면
- [ ] ✅ 확인: 폰에서 로그인 → 탭 이동 / NF-4

## Phase 1 — 데이터 준비
- [ ] 마이그레이션: profiles, schedule_blocks
- [ ] 마이그레이션: facilities, activities
- [ ] 마이그레이션: path_segments, trips, route_ratings
- [ ] RLS 정책 (design.md 2.3)
- [ ] segment_scores 집계 뷰 (3건 미만 비공개)
- [ ] 용산구 시설 공공데이터 수집 (복지관·경로당·공원·체육·문화·교육)
- [ ] `scripts/import-facilities.ts` 작성·실행
- [ ] 좌표 없는 시설 주소→좌표 변환
- [ ] 🚶 팀 현장 작업: seed 구간 30개 이상 걷고 태그·점수 기록
- [ ] `scripts/import-segments.ts` 작성·실행
- [ ] ✅ 확인: 시설 100곳+, seed 구간 30개+

## Phase 2 — 내 주변 시설 지도 (기능 ③)
- [ ] `GET /api/facilities` (반경 검색, 거리·도보 시간)
- [ ] `GET /api/facilities/[id]`
- [ ] 위치 권한 요청·거부 처리 (동 중심 대체)
- [ ] S05 지도: 카카오맵 + 마커
- [ ] S05 카테고리 칩 + 가까운 순 목록
- [ ] S06 시설 상세 (전화 `tel:` 링크 포함)
- [ ] 빈 상태·운영시간 없음 처리
- [ ] ✅ 확인: TC-3.1~3.5, EC-3.1~3.3

## Phase 3 — 빈 시간 시간표 (기능 ①)
- [ ] `getFreeSlots()` 함수 + 단위 테스트 (TC-1.4, EC-1.1)
- [ ] S04 시간표 화면 (요일별, 30분 블록)
- [ ] 일정 추가·수정·삭제
- [ ] 겹침·시간 역전 검증 (EC-1.2, 1.3)
- [ ] S02 처음 설정 (닉네임·관심사·동·위치 동의)
- [ ] ✅ 확인: TC-1.1~1.4, EC-1.1~1.4

## Phase 4 — 빈 시간 활동 추천 (기능 ②)
- [ ] `GET /api/recommendations` (후보·시간 필터·점수)
- [ ] 추천 이유 문구
- [ ] S03 홈: 오늘 시간표 요약 + 다음 빈 시간 + 추천 3개
- [ ] 추천 없음·관심사 없음 처리
- [ ] ✅ 확인: TC-2.1~2.3, EC-2.1~2.5, UF-3

## Phase 5 — 만족도 높은 길 추천 (기능 ⑤)
- [ ] TMAP 보행자 경로 API 키 발급, 서버 전용 래퍼
- [ ] 경로 결과 캐시 (10분)
- [ ] `POST /api/routes`: 기본 경로 + 경유 대안 + 거르기 + 이름표
- [ ] S07 길 고르기 (경로 색깔별 + 카드)
- [ ] `POST /api/trips`, `POST /api/trips/[id]/arrive`
- [ ] S08 걷는 중 (내 위치, 남은 시간, 다녀왔어요)
- [ ] API 실패 시 대체 표시
- [ ] ✅ 확인: TC-5.1~5.4, EC-5.1~5.5

## Phase 6 — 경로 만족도 평가 + 마무리 (기능 ④)
- [ ] S09 평가 화면 (얼굴 5개, 칩)
- [ ] `POST /api/trips/[id]/rating` + segment_scores 갱신
- [ ] 중복 평가·타인 trip 차단 (EC-4.1, 4.4)
- [ ] S10 내 정보 + 회원 탈퇴 (NF-6)
- [ ] 오프라인 캐시 (시간표·시설 목록)
- [ ] 접근성 점검 (NF-1~3)
- [ ] 위치기반서비스사업 신고 필요 여부 확인
- [ ] ✅ 확인: TC-4.1~4.4, EC-4.1~4.4, UF-1~UF-4, NF 전체

---

## 회고

> Phase가 끝날 때마다 3줄로 남깁니다.

### Phase 0
- 잘된 점:
- 아쉬운 점:
- 다음에 바꿀 점:
