# 크로스 프로모션 배너 – 신규 8개 앱 적용 조사

> **리뷰완료** (2026-09-05, 사용자 승인)

작성일: 2026-09-05
대상: `cross_promo.json`의 `rules`에 소스로 등록된 신규 8개 앱

---

## 1. 목표

`cross_promo.json`에 앱 정보와 룰은 등록 완료(커밋 `807267b`)됐으나,
아래 8개 앱은 **클라이언트에 배너 표시 기능이 없어 실제로는 배너가 뜨지 않는다.**
참조 구현을 이식해 8개 앱 모두 배너를 노출하도록 한다.

---

## 2. 대상 앱 ↔ 디렉터리 매핑

| # | 앱 키 | 프로젝트 경로 (`~/Documents/`) | 표시할 대상 |
|---|---|---|---|
| 1 | `kr_briefing` | `stock_briefing/apps/kr_briefing` | us_briefing, financial_radar |
| 2 | `us_briefing` | `stock_briefing/apps/us_briefing` | kr_briefing, fire_calculator |
| 3 | `envelope_screener` | `stock_briefing/apps/envelope_screener` | kr_briefing |
| 4 | `financial_radar` | `financial_radar` | fire_calculator, kr_briefing |
| 5 | `fire_calculator` | `fire_calculator/fire_calculator` | us_briefing, financial_radar |
| 6 | `remtime` | `remtime` | focusleep |
| 7 | `mlb_stadiums_app` | `mlb_stadiums_ive_been` | national_parks_app |
| 8 | `national_parks_app` | `national_parks_ive_been` | mlb_stadiums_app |

> 디렉터리명과 앱 키가 다른 3건 주의: `fire_calculator/fire_calculator`(중첩),
> `mlb_stadiums_ive_been`, `national_parks_ive_been`

---

## 3. 참조 구현 분석

구현이 완료된 앱: `serie_light`, `nba_light`, `epl_light`, `rush_hour`,
`jhmyung6225/section_eight/section8_guide_app`, `jhmyung6225/highlights_*`

### 3.1 파일 구조 (4파일)

| 파일 | 줄수 | 역할 |
|---|---|---|
| `lib/models/cross_promo_config.dart` | 28 | `CrossPromoApp` 모델 + 로케일 폴백 |
| `lib/services/cross_promo_service.dart` | 77 | JSON fetch·파싱·필터링, 싱글턴 + 메모리 캐시 |
| `lib/widgets/cross_promo_banner.dart` | 136 | `StatefulWidget` 배너 UI |
| (화면 파일) | — | `const CrossPromoBanner()` 삽입 |

### 3.2 핵심 발견 — 앱별 차이가 `_appId` 한 줄뿐

`serie_light`와 `section8_guide_app`의 `cross_promo_service.dart`를 diff한 결과:

```
8c8
< const _appId = 'serie_light';
---
> const _appId = 'section8_guide';
```

**모델·서비스·위젯 3파일은 그대로 복사하고 `_appId` 한 줄만 바꾸면 된다.**
이식 난이도가 매우 낮다.

### 3.3 서비스 동작

- 설정 URL: `https://raw.githubusercontent.com/jinhom6225/app-config/main/cross_promo.json`
- `HttpClient`, connectionTimeout 5초
- `rules[_appId]` 로 대상 ID 목록을 얻어 순회
- 필터: `enabled != true` → skip
- **플랫폼 필터: `stores[ios|android]`가 없으면 `continue`로 건너뜀**
- 전 구간 `try/catch`로 감싸 실패 시 빈 리스트 반환 → 배너 미표시(앱 크래시 없음)
- `_cached`로 앱 실행 중 1회만 네트워크 호출

### 3.4 위젯 동작

- `initState`에서 fetch → `setState`
- `_apps.isEmpty` 이면 `SizedBox.shrink()` (공간 차지 안 함)
- 다크 카드(`0xFF1a1a2e`) + 아이콘 50x50 + 제목/설명 + 초록 다운로드 버튼(`0xFF01875f`)
- 버튼 라벨 하드코딩 7개 로케일(en/ko/ja/es/de/fr/it)
- 아이콘 로드 실패 시 `errorBuilder`로 회색 플레이스홀더
- 탭 시 `url_launcher`의 `launchUrl(externalApplication)`

### 3.5 삽입 패턴 (`serie_light/lib/screens/results_screen.dart`)

```dart
// 데이터 없을 때도 배너는 계속 보이게
const CrossPromoBanner(),
const SizedBox(height: 32),
...
// 리스트 최상단에 배치
if (index == 0) {
  return const CrossPromoBanner();
}
```

---

## 4. 대상 앱 조사 결과

### 4.1 공통 — `url_launcher` 의존성이 8개 앱 전부 없음

```
kr_briefing NO · us_briefing NO · envelope_screener NO · shared NO
financial_radar NO · fire_calculator NO · remtime NO
mlb_stadiums_ive_been NO · national_parks_ive_been NO
```

→ 8개 앱(또는 shared 패키지) `pubspec.yaml`에 `url_launcher` 추가 필요.

### 4.2 `stock_briefing`은 공유 패키지를 가진 모노레포

```
stock_briefing/
├── apps/{kr_briefing, us_briefing, envelope_screener}
└── packages/shared/lib/src/{ads, briefing, cache, extensions,
                             heatmap, network, paper_trading,
                             rankings, screener, theme, widgets}
```

→ 3개 앱에 3벌 복사하지 말고 **`packages/shared/lib/src/cross_promo/`에 1벌만
구현**하고, `_appId`만 앱별로 주입하는 구조가 맞다.
(`_appId` 상수를 `CrossPromoService(appId: ...)` 파라미터로 바꾸는 최소 변경 필요)

### 4.3 앱별 첫 화면 (배너 삽입 위치) — 코드 확인 완료

"첫 번째 탭 = 앱 실행 시 최초로 보이는 화면"으로 확정. 배너는 그 화면 **최상단**.

| 앱 | 첫 화면 | 파일 | 근거 |
|---|---|---|---|
| kr_briefing | `HomeScreen` | `lib/features/home/home_screen.dart` | `app.dart:87` 탭 배열 0번 |
| us_briefing | `HomeScreen` | `lib/features/home/home_screen.dart` | `app.dart:87` 탭 배열 0번 |
| envelope_screener | **`SearchScreen`** | `lib/features/search/…` | `app.dart:97` 탭 배열 0번 (Home 없음) |
| financial_radar | `HomeScreen` | `lib/features/home/home_screen.dart` | `app.dart:29` splash 이후 진입 |
| fire_calculator | `HomeScreen` | `lib/features/home/home_screen.dart` | `app.dart:29` splash 이후 진입 |
| remtime | `HomeTab` | `lib/features/home/presentation/screens/home_tab.dart` | `shell_screen.dart:20` 배열 0번 |
| mlb_stadiums_app | **`MapTab`** | `lib/screens/map_tab.dart` | `main.dart:76` `_screens` 0번 |
| national_parks_app | **`MapTab`** | `lib/screens/map_tab.dart` | `home_screen.dart:31` `_tabs` 0번 |

> 주의 3건:
> - `envelope_screener`의 첫 탭은 Home이 아니라 `SearchScreen`
> - 여행 앱 2개의 첫 탭은 전체화면 `MapTab` → 지도 위 최상단 배치 필요
> - `national_parks_app`은 하단에 AdMob 배너가 이미 있음 (`home_screen.dart:92`)

---

## 5. 리스크 / 미결 사항

| # | 항목 | 내용 |
|---|---|---|
| ~~R1~~ | ~~삽입 위치 미확정~~ | **해결** — 첫 탭 최상단으로 확정, 4.3 표 참조 |
| R2 | shared 리팩터링 범위 | `_appId` 상수 → 생성자 파라미터 변경 시 기존 구현 앱과 코드가 갈라짐 |
| ~~R3~~ | ~~디자인 톤 불일치~~ | **해결** — 다크 고정 대신 `Theme.of(context)` 기반으로 재작성 (참조 구현 그대로 복사 불가) |
| ~~R4~~ | ~~광고와의 간섭~~ | **해결** — 크로스 배너는 항상 최상단, AdMob 배너보다 위 |
| R7 | MapTab 위 배치 | 여행 앱 2개는 전체화면 지도라 배너가 지도를 가림. overlay vs 지도 축소 결정 필요 |
| R5 | 노출 0회 | `envelope_screener`, `remtime`은 현재 아무도 추천해주지 않음(설정 이슈, 구현과 무관) |
| R6 | 테스트 | 참조 구현에 테스트가 없음. TDD 규칙(80%+)과 충돌 → 서비스 로직 단위 테스트 필요 |

---

## 6. 참고

- 설정 저장소: `~/Documents/app-config` (커밋 `807267b`)
- 배너는 앱 재배포 없이 `cross_promo.json` push만으로 내용 변경 가능
- 단, **배너 표시 기능 자체는 앱 배포가 필요** (이번 작업 대상)
