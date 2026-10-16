# stack.md — 느린 지도: Tech Stack Reference

> 고정 스택입니다. 바꾸려면 adr.md에 결정을 먼저 남깁니다.
> 버전은 Phase 0에서 설치한 실제 버전으로 고쳐 적습니다.

## 런타임·프레임워크
| 항목 | 선택 | 메모 |
|---|---|---|
| 언어 | TypeScript (strict) | |
| 프레임워크 | Next.js (App Router) | Route Handlers로 API |
| 스타일 | Tailwind CSS | 접근성 토큰(글씨·버튼 크기)을 theme에 정의 |
| 패키지 매니저 | pnpm (가정) | |
| 테스트 | Vitest (단위) · Playwright (유저 플로우) (가정) | test.md의 TC·UF와 연결 |

## 백엔드·데이터
| 항목 | 선택 | 메모 |
|---|---|---|
| DB | Supabase Postgres + PostGIS | `create extension postgis;` |
| 인증 | Supabase Auth — 카카오 로그인 | |
| 권한 | Row Level Security | design.md 2.3 |
| 마이그레이션 | Supabase CLI (`supabase/migrations`) | |

## 외부 서비스
| 항목 | 선택 | 쓰는 곳 | 메모 |
|---|---|---|---|
| 지도 표시 | 카카오맵 JavaScript SDK | 브라우저 | 도메인 제한 필수 |
| 주소→좌표 | 카카오 로컬 API | import 스크립트 | 공공데이터에 좌표 없을 때 |
| 도보 경로 | TMAP 보행자 경로 API | 서버 전용 | 일일 호출 한도 확인, 10분 캐시 |
| 시설 데이터 | 공공데이터포털 / 용산구 | import 스크립트 | 출처를 facilities.source에 기록 |

## 배포
| 항목 | 선택 |
|---|---|
| 호스팅 | Vercel |
| 형태 | PWA (manifest + 홈 화면 추가) |

## 환경변수
| 이름 | 공개 여부 | 설명 |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | 공개 | |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | 공개 | RLS로 보호 |
| `SUPABASE_SERVICE_ROLE_KEY` | **서버 전용** | import 스크립트·집계 갱신만 |
| `NEXT_PUBLIC_KAKAO_MAP_JS_KEY` | 공개 | 도메인 제한 걸기 |
| `KAKAO_REST_API_KEY` | **서버 전용** | 주소→좌표 |
| `TMAP_APP_KEY` | **서버 전용** | 도보 경로 |

`.env.local`은 커밋하지 않습니다. `.env.example`에 이름만 적어 둡니다.
