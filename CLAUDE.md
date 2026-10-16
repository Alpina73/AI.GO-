# CLAUDE.md — 느린 지도

> 이 파일은 매 요청마다 먼저 읽는 프로젝트 앵커입니다.

## 프로젝트 한 줄
은퇴한 시니어가 빈 시간에 할 일과 걷기 좋은 길을 찾아 주고, 시니어가 걸은 길의 만족도를 모아 '만족도 높은 길'을 추천하는 용산구 동네 지도 서비스. (팀 AIgo, 아립앤위립 2팀)

## 매 요청 복기 순서
작업 전에 필요한 문서를 이 순서로 확인합니다.
1. **brd.md** — 무엇을, 누구를 위해 만드는가 / MVP 5개 / 범위 밖
2. **cps.md** — 왜 이 솔루션인가
3. **design.md** — 데이터 모델·API·화면·핵심 로직
4. **plan.md** — 지금 어느 Phase인가
5. **todo.md** — 다음 할 일과 진행률
6. **test.md** — 이 작업의 완료 기준
- 보조: **stack.md**(스택·환경변수), **adr.md**(설계 결정 이유), **log.md**(개발 기록)

## 기술 스택
Next.js (App Router, TypeScript) · Tailwind CSS · Supabase (Postgres + PostGIS, Auth, RLS) · 카카오맵 JS SDK · TMAP 보행자 경로 API · Vercel · PWA
상세와 버전은 stack.md를 따릅니다.

## 폴더 구조
```
app/
  (tabs)/today, map, schedule, me    화면 (S03, S05, S04, S10)
  facilities/[id]                     S06 시설 상세
  route/[facilityId]                  S07 길 고르기
  trip/[id], trip/[id]/rate           S08 걷는 중, S09 평가
  onboarding                          S02 처음 설정
  api/                                Route Handlers (design.md 5장)
lib/
  supabase/                           서버·브라우저 클라이언트
  domain/                             순수 로직: getFreeSlots, 추천 점수, 경로 거르기
  tmap/                               TMAP 래퍼 (서버 전용)
components/                           공용 UI (큰 버튼, 칩, 카드)
supabase/migrations/                  SQL 마이그레이션
scripts/                              데이터 import
```

## 코딩 원칙
- **TypeScript strict.** `any` 금지. DB 타입은 Supabase 타입 생성으로.
- **순수 로직은 `lib/domain/`에** 두고 단위 테스트를 먼저 쓴다 (빈 시간 계산, 추천 점수, 경로 거르기).
- **시니어 접근성은 기본값이다.** 본문 18px+, 버튼 56px+, 아이콘에는 항상 글자. 공용 컴포넌트를 써서 지킨다.
- **문구는 쉬운 존댓말.** "오류가 발생했습니다" 대신 "길 안내를 잠시 못 불러왔어요".
- **모든 화면에 빈 상태·오류 상태**가 있어야 한다 (design.md 4.4).
- 작은 단위로 커밋하고, 커밋 메시지에 todo 항목을 적는다.

## 확인 후 진행할 것 (먼저 물어보기)
- DB 스키마 변경 (마이그레이션 추가·수정)
- 새 외부 API·라이브러리 추가
- brd.md의 MVP 기능 범위를 바꾸는 작업
- 추천·경로 점수 공식의 가중치 변경
- 개인정보·위치 데이터를 다루는 방식 변경

## 절대 금지
- brd.md 7장 **범위 밖 기능 구현** (매칭, 제보, 음성 기록, 포인트 등)
- **TMAP API 키를 클라이언트 코드에** 넣기
- **사용자 위치를 요청 외 목적으로 저장**하거나, 3건 미만 집계 점수·타인의 trip·평가를 노출하기
- RLS 끄기, service role 키를 브라우저에서 쓰기
- test.md 기준을 통과하지 않은 작업을 todo.md에서 완료로 표시하기

## 자주 쓰는 패턴
- **반경 검색:** `ST_DWithin(location, ST_MakePoint(lng, lat)::geography, radius)` + `ST_Distance`로 정렬
- **도보 시간:** `round(distance_m * 1.3 / (walk_speed_kmh * 1000 / 60))` 분 (경로 확정 전)
- **API 오류:** `{ error: { code, message } }` 형식, 화면은 `code`로 분기
- **위치 없음:** profiles.home_area의 동 중심 좌표로 대체

## 작업 요청 예시
- "plan.md Phase 2의 1번(`GET /api/facilities`)을 구현해줘. 끝나면 todo.md 업데이트하고 다음 작업 알려줘."
- "design.md 3.3 경로 추천 로직을 `lib/domain/`에 순수 함수로 만들고 EC-5.1, 5.2 테스트부터 써줘."

## 개발 로그
주요 작업·결정·문제는 log.md에 날짜와 함께 남깁니다. 설계 결정은 adr.md에 번호를 붙여 추가합니다.
