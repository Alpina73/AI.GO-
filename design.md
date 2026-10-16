# design.md — 느린 지도: Application Design

> cps.md의 솔루션을 시스템으로 옮긴 문서입니다.
> 입력: cps.md · 다음 문서: test.md, plan.md
> `(가정)`은 팀 결정 전 기본값입니다. 바꾸면 stack.md와 adr.md도 함께 고칩니다.

---

## 1. 아키텍처 개요

```
[시니어 폰 브라우저 (PWA)]
        │  HTTPS
        ▼
[Next.js App Router on Vercel]
   ├─ 화면 (React Server/Client Components)
   ├─ Route Handlers  /api/*   ← 추천·경로 계산
   │        │
   │        ├──▶ [TMAP 보행자 경로 API]   도보 경로·시간
   │        └──▶ [Supabase]
   │                ├─ Postgres + PostGIS   시설·길·평가
   │                ├─ Auth (카카오 로그인)
   │                └─ RLS                  내 데이터는 나만
   └─ 지도 표시 ──▶ [카카오맵 JavaScript SDK] (브라우저)
```

| 영역 | 선택 | 이유 |
|---|---|---|
| 플랫폼 | 모바일 웹앱(PWA) | 설치 없이 링크로 시작, 현장테스트 배포가 쉽다 (ADR-001) |
| 프론트·서버 | Next.js (App Router) + TypeScript | 화면과 API를 한 프로젝트로 |
| DB·인증 | Supabase (Postgres + PostGIS, Auth, RLS) | 위치 계산을 DB에서 처리 (ADR-002) |
| 지도 표시 | 카카오맵 JavaScript SDK | 국내 지도·장소 정보가 익숙함 (가정) |
| 도보 경로 | TMAP 보행자 경로 API | 국내 도보 경로·경유지 지원 (ADR-003, 가정) |
| 배포 | Vercel | Next.js 기본 배포 |

---

## 2. 데이터 모델

Supabase Postgres, PostGIS 확장 사용. 좌표는 `geography(Point/LineString, 4326)`.

### 2.1 테이블

**profiles** — 사용자 (auth.users와 1:1)
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | uuid PK, FK → auth.users | |
| nickname | text | 화면에 보일 이름 |
| interests | text[] | 관심사 태그: `walk`, `exercise`, `culture`, `learning`, `social`, `volunteer` |
| walk_speed_kmh | numeric default 3.0 | 예상 이동시간 계산용 (가정: 시니어 평균 3km/h) |
| home_area | text | 동 단위 (예: 이촌동). 정확한 주소는 받지 않음 |
| location_consent_at | timestamptz | 위치정보 이용 동의 시각 |
| created_at | timestamptz | |

**schedule_blocks** — 시간표 (기능 ①)
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | uuid PK | |
| user_id | uuid FK → profiles | |
| weekday | smallint 0–6 | 0=월요일 |
| start_time / end_time | time | 30분 단위 |
| title | text | 예: "병원", "손주 하원" |
| kind | text | `busy`(일정) / `routine`(반복 활동) |

빈 시간 = 하루 활동 시간대(기본 07:00–21:00, 가정) 중 `schedule_blocks`가 없는 30분 이상 구간. 저장하지 않고 계산한다.

**facilities** — 시설 (기능 ②③)
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | uuid PK | |
| name | text | |
| category | text | `welfare`(복지관·경로당) / `sports` / `culture` / `education` / `park` |
| interest_tags | text[] | 관심사와 매칭할 태그 |
| location | geography(Point) | |
| address, phone | text | |
| open_hours | jsonb | `{ "mon": ["09:00","18:00"], ... }` |
| description | text | 상세정보 |
| source, source_id | text | 공공데이터 출처 (중복 방지 unique) |

**activities** — 시설의 프로그램 (선택, 기능 ②)
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | uuid PK | |
| facility_id | uuid FK | |
| title | text | 예: "건강 체조" |
| weekday, start_time, end_time | | 운영 시간 |
| interest_tags | text[] | |

공공데이터에 프로그램 정보가 없을 수 있다. 없으면 시설 자체를 추천 단위로 쓴다. (가정)

**path_segments** — '길' 단위 (기능 ④⑤)
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | uuid PK | |
| name | text | 예: "효창공원 둘레길 북쪽" |
| geom | geography(LineString) | 100–500m 길이의 구간 |
| tags | text[] | 특징: `shade`(그늘), `rest`(쉼터), `flat`(평지), `view`(경치), `safe`(안전) |
| source | text | `seed`(팀 초기 데이터) / `user` |

**trips** — 한 번의 이동
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | uuid PK | |
| user_id | uuid FK | |
| destination_facility_id | uuid FK null | |
| route_geom | geography(LineString) | 선택한 경로 |
| segment_ids | uuid[] | 경로가 지나간 path_segments |
| started_at, arrived_at | timestamptz | |

**route_ratings** — 경로 평가 (기능 ④)
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | uuid PK | |
| trip_id | uuid FK unique | 이동 1번에 평가 1번 |
| user_id | uuid FK | |
| score | smallint 1–5 | 얼굴 5단계 |
| tags | text[] | 좋았던 점 (`shade`, `rest`, `flat`, `view`, `safe`) |
| issues | text[] | 불편했던 점 (`steep`, `no_rest`, `crowded`, `unsafe`) |
| created_at | timestamptz | |

### 2.2 집계 뷰 (개인정보 원칙)

**segment_scores** (materialized view, 평가가 들어올 때 갱신)
- 각 path_segment가 포함된 trip의 평가를 모아 `avg_score`, `rating_count`, 상위 태그를 계산.
- **`rating_count < 3`인 구간은 사용자 평가 점수를 노출하지 않는다.** seed 점수만 쓴다. 개인 한 명의 경로·평가가 드러나지 않게 하기 위함.

### 2.3 RLS 정책
| 테이블 | 읽기 | 쓰기 |
|---|---|---|
| profiles, schedule_blocks, trips, route_ratings | 본인만 | 본인만 |
| facilities, activities, path_segments, segment_scores | 로그인 사용자 전체 | 관리자(service role)만 |

---

## 3. 핵심 로직

### 3.1 빈 시간 활동 추천 (기능 ②)
입력: 빈 시간 구간, 사용자 관심사, 현재 위치(없으면 home_area 중심)

1. **후보:** 반경 1.5km(가정) 안의 시설 중, 빈 시간 동안 운영 중인 곳.
2. **시간 필터:** `이동시간 × 2 + 최소 체류 30분 ≤ 빈 시간 길이`인 곳만 남긴다.
3. **점수:** `관심사 일치(0.5) + 가까움(0.3) + 길 만족도(0.2)` — 시설로 가는 길 주변 segment의 평균 점수.
4. 상위 3개 반환. 각 항목에 "왜 추천했는지" 한 줄 표시 (예: "운동을 좋아하셔서 · 걸어서 12분").

MVP는 규칙 기반이다. 사용자 특성을 학습하는 개인화는 범위 밖(brd 7장).

### 3.2 거리·이동시간 (기능 ③)
- **목록·지도:** PostGIS `ST_Distance` 직선거리 × 1.3(도보 보정, 가정) ÷ 사용자 걷는 속도.
- **경로 선택 후:** TMAP 보행자 경로 결과의 실제 거리·시간을 사용자 속도로 다시 계산.

### 3.3 만족도 높은 길 추천 (기능 ⑤)
입력: 출발지(현재 위치), 목적지(시설)

1. **기본 경로:** TMAP 보행자 경로(경유지 없음) → 후보 A ("가장 빠른 길").
2. **경유 후보 찾기:** 출발–목적지 직선을 기준으로 폭 300m(가정) 안에 있는 segment 중 점수 상위 구간을 고른다.
3. **대안 경로:** 상위 구간을 경유지로 넣어 TMAP을 다시 호출 → 후보 B, C.
4. **거르기:** 기본 경로보다 **50% 이상 길어지면 제외**(가정). 서로 70% 이상 겹치는 경로는 하나만 남긴다.
5. 각 경로에 이름표를 붙여 2~3개 반환: "가장 빠른 길", "그늘 많은 길", "쉼터 있는 길" 등 — 지나는 segment의 상위 태그로 정한다.

TMAP 호출 결과는 같은 출발·목적지에 대해 10분간 캐시해 호출 수를 줄인다.

### 3.4 평가 반영 (기능 ④)
- 도착 버튼 또는 목적지 50m 안 도착 감지 → 평가 화면.
- 평가 저장 시, trip의 `segment_ids` 각각에 점수를 반영하고 `segment_scores`를 갱신.
- 평가를 건너뛰면 trip만 남고 점수에는 영향 없음.

---

## 4. UI/UX

### 4.1 디자인 원칙 (시니어 접근성)
- 본문 글씨 **18px 이상**, 주요 버튼 높이 **56px 이상**, 명도 대비 WCAG AA 이상.
- 한 화면에 **주요 행동 1개**. 버튼 문구는 동사로 ("길 찾기", "다녀왔어요").
- 아이콘만 쓰지 않고 항상 글자를 붙인다.
- 하단 탭 4개: **오늘 · 지도 · 시간표 · 내 정보**.
- 키보드 입력이 필요한 곳은 닉네임 1곳뿐. 나머지는 선택·탭으로.

### 4.2 화면 목록
| ID | 화면 | 내용 | 기능 |
|---|---|---|---|
| S01 | 시작·로그인 | 서비스 소개 1줄, 카카오 로그인 | — |
| S02 | 처음 설정 | 닉네임 → 관심사 고르기(큰 카드) → 사는 동 → 위치 동의 | ② |
| S03 | 오늘 (홈) | 오늘 시간표 요약, "다음 빈 시간 14:00–16:00" + 추천 3개 | ①② |
| S04 | 시간표 | 요일별 세로 시간표, 빈 칸 탭 → 일정 추가 / 일정 탭 → 수정·삭제 | ① |
| S05 | 지도 | 내 위치 주변 시설 마커, 카테고리 필터 칩, 아래 목록(거리·도보 시간) | ③ |
| S06 | 시설 상세 | 이름, 분류, 운영시간, 주소, 전화, 설명, 거리·도보 시간, [길 찾기] | ③ |
| S07 | 길 고르기 | 지도 위 경로 2~3개 색깔별 표시, 카드: 이름표·시간·거리·특징 | ⑤ |
| S08 | 걷는 중 | 선택 경로, 내 위치, 남은 시간, [다녀왔어요] | ⑤ |
| S09 | 길 평가 | 얼굴 5개 → 좋았던 점 칩 → 불편했던 점 칩 → [보내기] / [건너뛰기] | ④ |
| S10 | 내 정보 | 관심사·동·걷는 속도 수정, 위치 동의 철회, 로그아웃 | — |

### 4.3 주요 흐름
```
S03 오늘 ─ 추천 탭 ─▶ S06 시설 상세 ─ 길 찾기 ─▶ S07 길 고르기 ─ 선택 ─▶ S08 걷는 중
                                                                       │ 다녀왔어요
                                                                       ▼
S03 오늘 ◀──────────────── "고맙습니다! 다음 사람에게 도움이 돼요" ◀── S09 길 평가
```

### 4.4 상태 처리
| 상황 | 표시 |
|---|---|
| 위치 권한 거부 | "사는 동네 기준으로 보여드릴게요" + 동 중심으로 계속. 길 찾기에서는 출발지를 시설 목록에서 고르게 안내 (test.md UF-4) |
| 주변 시설 없음 | 반경을 넓혀 다시 찾기 버튼 |
| 빈 시간 없음 | "오늘은 일정이 꽉 찼네요" + 내일 빈 시간 보기 |
| 경로 API 실패 | 직선 거리·예상 시간만 보여 주고 "길 안내를 잠시 못 불러왔어요" |
| 오프라인 | 마지막으로 본 시간표·시설 목록을 캐시로 보여 줌 |

---

## 5. API 설계

Next.js Route Handlers. 인증은 Supabase 세션 쿠키. 단순 CRUD(시간표, 내 정보)는 클라이언트에서 Supabase를 직접 호출하고 RLS로 보호한다.

| 메서드 | 경로 | 입력 | 출력 | 기능 |
|---|---|---|---|---|
| GET | `/api/facilities` | `lat, lng, radius=1500, category?` | 시설 목록 + `distance_m`, `walk_min` | ③ |
| GET | `/api/facilities/[id]` | `lat?, lng?` | 시설 상세 + 거리·시간 + 오늘 프로그램 | ③ |
| GET | `/api/recommendations` | `date, lat?, lng?` | 빈 시간 구간별 추천 3개 + 이유 | ①② |
| POST | `/api/routes` | `from{lat,lng}, facility_id` | 경로 2~3개 `{label, distance_m, walk_min, geom, tags}` | ⑤ |
| POST | `/api/trips` | `facility_id, route` | `trip_id` | ⑤ |
| POST | `/api/trips/[id]/arrive` | — | `arrived_at` | ⑤ |
| POST | `/api/trips/[id]/rating` | `score, tags[], issues[]` | 저장 결과 | ④ |

**오류 형식:** `{ "error": { "code": "ROUTE_API_FAILED", "message": "..." } }` — 화면은 `code`로 상태 처리(4.4)를 고른다.

---

## 6. 데이터 준비 (초기 데이터)

| 데이터 | 출처 | 방법 |
|---|---|---|
| 노인복지관·경로당 | 용산구 / 공공데이터포털 (가정) | CSV → `scripts/import-facilities.ts` |
| 공원·산책로·운동시설 | 용산구 공원 데이터 (가정) | 같은 스크립트 |
| 문화·교육시설 | 공공데이터포털 (가정) | 같은 스크립트 |
| 길(path_segments) seed | **팀이 직접 걷고 기록** | 용산가족공원·효창공원·이촌한강공원 주변 30구간 이상 (가정), GeoJSON으로 정리 후 import |

---

## 7. 보안·개인정보

- 위치는 **요청할 때만** 쓰고 저장하지 않는다. 저장되는 위치는 trip의 경로뿐이다.
- 다른 사용자에게는 `segment_scores`(3건 이상 집계)만 보인다. 개별 trip·평가는 본인만 본다.
- 회원 탈퇴 시 trips·route_ratings·schedule_blocks를 삭제한다. 단, 이미 집계된 segment 점수는 익명 통계로 남는다.
- 위치기반서비스사업 신고 필요 여부는 현장테스트 전에 확인한다 (brd 6장).
- API 키: TMAP 키는 서버(Route Handler)에서만 쓴다. 카카오맵 JS 키는 도메인 제한을 건다.
