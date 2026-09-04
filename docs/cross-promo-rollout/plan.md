# 크로스 프로모션 배너 – 신규 8개 앱 구현 계획

> **리뷰완료** (2026-09-05, 사용자 승인)

작성일: 2026-09-05
근거 문서: [research.md](./research.md)

---

## 1. 최초 계획

### Phase 0 — 사전 결정 (완료)

| 항목 | 결정 |
|---|---|
| P0-1 배너 위치 | **첫 번째 탭(앱 실행 시 최초 화면)에만**, 화면 **최상단** |
| P0-2 stock_briefing 구조 | `packages/shared`에 1벌 구현 후 `appId` 주입 (모노레포 기존 구조를 따름) |
| P0-3 배너 디자인 | 다크 고정 폐기, **앱 테마(`Theme.of(context)`) 기반**으로 재작성 |
| P0-4 AdMob 배너와 순서 | 크로스 배너가 **항상 위**, AdMob은 그 아래 유지 |

**P0-3의 파급효과:** 참조 구현의 위젯을 그대로 복사할 수 없다.
`cross_promo_banner.dart`는 테마 대응으로 새로 작성해야 하며, 하드코딩된
`0xFF1a1a2e`(배경)·`Colors.white`(제목)·`0xFFCCCCCC`(설명)·`0xFF01875f`(버튼)를
`colorScheme.surfaceContainer` / `onSurface` / `onSurfaceVariant` / `primary` 계열로 치환한다.
모델·서비스 2파일은 기존대로 복사 가능.

### 배너 삽입 대상 (첫 화면, 코드 확인 완료)

| 앱 | 삽입 파일 |
|---|---|
| kr_briefing | `lib/features/home/home_screen.dart` |
| us_briefing | `lib/features/home/home_screen.dart` |
| envelope_screener | `SearchScreen` (첫 탭이 Home이 아님) |
| financial_radar | `lib/features/home/home_screen.dart` |
| fire_calculator | `lib/features/home/home_screen.dart` |
| remtime | `lib/features/home/presentation/screens/home_tab.dart` |
| mlb_stadiums_app | `lib/screens/map_tab.dart` |
| national_parks_app | `lib/screens/map_tab.dart` |

### Phase 1 — `stock_briefing` 3개 앱 (공유 패키지 방식)

`packages/shared`에 1벌 구현 후 3개 앱에서 `appId`만 주입.

1. `packages/shared/pubspec.yaml`에 `url_launcher` 추가
2. `packages/shared/lib/src/cross_promo/cross_promo_config.dart` 생성 (참조 구현 복사)
3. `packages/shared/lib/src/cross_promo/cross_promo_service.dart` 생성
   - `const _appId` → `CrossPromoService({required this.appId})` 로 변경
   - 싱글턴 → 앱별 인스턴스 또는 `appId` 기준 캐시 맵
4. `packages/shared/lib/src/cross_promo/cross_promo_banner.dart` **신규 작성**
   - `CrossPromoBanner({required this.appId})`
   - 테마 기반 색상 (P0-3)
5. `packages/shared/lib/shared.dart`에 export 추가
6. 첫 화면 최상단에 삽입
   - kr_briefing / us_briefing → `features/home/home_screen.dart`
   - envelope_screener → `SearchScreen` (첫 탭이 Home이 아님)

### Phase 2 — 독립 앱 3개 (`financial_radar`, `fire_calculator`, `remtime`)

앱마다 참조 구현 3파일 복사 + `_appId` 한 줄 수정.

1. 각 앱 `pubspec.yaml`에 `url_launcher` 추가
2. `lib/models/cross_promo_config.dart` 복사
3. `lib/services/cross_promo_service.dart` 복사 후 `_appId` 수정
4. `lib/widgets/cross_promo_banner.dart` 신규 작성 (테마 대응, P0-3)
5. 삽입
   - `financial_radar` → `features/home/home_screen.dart` 최상단
   - `fire_calculator` → `features/home/home_screen.dart` 최상단
   - `remtime` → `features/home/presentation/screens/home_tab.dart` 최상단

> `financial_radar`와 `fire_calculator`는 `lib/shared/` 구조를 쓰므로
> `lib/shared/cross_promo/` 하위로 배치할지 검토

### Phase 3 — 여행 앱 2개 (`mlb_stadiums_app`, `national_parks_app`)

`screens/` + `widgets/` 구조. 3파일 복사 후 삽입.

1. `pubspec.yaml`에 `url_launcher` 추가
2. `lib/models/`, `lib/services/`, `lib/widgets/`에 각각 복사 + `_appId` 수정
3. 삽입 — 두 앱 모두 첫 탭이 전체화면 `MapTab`
   - `national_parks_app` → `screens/map_tab.dart` 최상단
   - `mlb_stadiums_app` → `screens/map_tab.dart` 최상단
   - 지도를 가리지 않도록 배치 방식 결정 필요 (R7)
   - `national_parks_app`은 하단 AdMob 배너 유지, 크로스 배너는 그 위

### Phase 4 — 테스트 (R6)

참조 구현에 테스트가 없으므로 신규 작성.

1. `CrossPromoService` 단위 테스트
   - 정상 파싱 / `rules`에 앱 키 없음 / `enabled: false` 필터
   - **플랫폼 스토어 URL 없을 때 건너뛰기** (핵심 분기)
   - HTTP 500 / 타임아웃 / 잘못된 JSON → 빈 리스트 반환
2. `CrossPromoApp` 로케일 폴백 테스트 (`ko` → `en` → 첫 값)
3. 위젯 테스트: 빈 리스트일 때 `SizedBox.shrink()` 렌더
4. 위젯 테스트: 라이트/다크 테마에서 색상이 테마를 따르는지 (P0-3)

### Phase 5 — 검증 및 배포

1. 앱별 `flutter analyze` 통과
2. 에뮬레이터에서 배너 노출 육안 확인 (앱당 1회)
3. 탭 시 스토어 이동 확인
4. iOS 대상이 없는 앱(Android 전용 대상)에서 빈 배너 확인
5. 스토어 배포는 사용자가 수행

---

## 2. 최초 구현

| Phase | 대상 | 상태 | 검증 |
|---|---|---|---|
| Phase 0 | 사전 결정 4건 | ✅ 완료 | — |
| Phase 1 | stock_briefing 3개 앱 (shared) | ✅ 완료 | analyze 0건, 기존 테스트 393개 통과 |
| Phase 2 | financial_radar, fire_calculator, remtime | ✅ 완료 | analyze 신규 0건, 기존 테스트 65개 통과 |
| Phase 3 | mlb_stadiums_app, national_parks_app | ✅ 완료 | analyze 신규 0건 |
| Phase 4 | 테스트 13개 × 6개 패키지 | ✅ 완료 | 전부 통과 |
| Phase 5 | 에뮬레이터 스크린샷 검증 | ✅ 완료 | 8개 앱 전부 배너 노출 확인 |

### Phase 5 검증 결과 (Pixel 8 / Android 16 에뮬레이터, 2026-09-05)

스크린샷: [`screenshots/`](./screenshots/) — `00_contact_sheet.png`이 전체 요약

| 앱 | 설정된 룰 | 실제 노출된 카드 | 테마 |
|---|---|---|---|
| kr_briefing | us_briefing, financial_radar | US Stock AI Briefing / Stock Chart Game | 다크·보라 ✅ |
| us_briefing | kr_briefing, fire_calculator | Korea Stock AI Briefing / US Stock Chart Game | 다크·보라 ✅ |
| envelope_screener | kr_briefing | Korea Stock AI Briefing | 다크·민트 ✅ |
| financial_radar | fire_calculator, kr_briefing | US Stock Chart Game / Korea Stock AI Briefing | 다크·초록 ✅ |
| fire_calculator | us_briefing, financial_radar | US Stock AI Briefing / Stock Chart Game | 다크·초록 ✅ |
| remtime | focusleep | Focusleep | 다크·앰버 ✅ |
| mlb_stadiums_app | national_parks_app | US National Parks | 라이트·파랑 ✅ |
| national_parks_app | mlb_stadiums_app | MLB Stadiums | 라이트·초록 ✅ |

**8/8 모두 설정된 룰과 정확히 일치.** 배너 색상이 앱 테마를 따라가는 것도 확인
(라이트 2개 / 다크 6개, Download 버튼이 각 앱 primary 색상).

지도 앱 2개(A3)는 배너가 지도 위에 자연스럽게 얹혔고 지도 동작에 문제 없음 → 조정 불필요.

### 구현 방식 (최초 계획 대비 변경)

계획의 "참조 구현 3파일 복사 + `_appId` 한 줄 수정" 대신 **서비스를 `fetchApps(appId)`
파라미터 방식으로 통일**했다. 앱마다 상수를 고칠 필요가 없고 shared 패키지에서도
같은 코드를 쓸 수 있다. 캐시는 `Map<String, List<CrossPromoApp>>`로 appId별 분리.

테스트를 위해 파싱 로직을 `@visibleForTesting static parseConfig(body, appId, {isIOS})`로
분리했다. 네트워크 목 없이 플랫폼 스킵·`enabled` 필터·잘못된 JSON 처리를 검증할 수 있다.

### 배치 결과

| 프로젝트 | 코드 위치 | 테스트 |
|---|---|---|
| `stock_briefing/packages/shared` | `lib/src/cross_promo/` + `shared.dart` export | `test/cross_promo_test.dart` |
| `financial_radar` | `lib/shared/cross_promo/` | `test/cross_promo_test.dart` |
| `fire_calculator` | `lib/shared/cross_promo/` | `test/cross_promo_test.dart` |
| `remtime` | `lib/shared/cross_promo/` (package: import) | `test/cross_promo_test.dart` |
| `mlb_stadiums_ive_been` | `lib/{models,services,widgets}/` | `test/cross_promo_test.dart` |
| `national_parks_ive_been` | `lib/{models,services,widgets}/` | `test/cross_promo_test.dart` |

---

## 3. 추가 구현 계획

> 최초 구현 후 발견된 버그·부족한 부분·추가 기능을 기록한다.

| 번호 | 일자 | 대상 | 내용 | 상태 |
|---|---|---|---|---|
| A1 | 2026-09-05 | kr_briefing, us_briefing | 크로스 배너가 `StaleDataBanner`(데이터 오래됨 경고)보다 위에 위치. "항상 최상단" 지시를 그대로 따른 결과이나, 경고 배너를 위로 올리는 편이 UX상 나을 수 있음 | 미구현 (사용자 판단 필요) |
| A2 | 2026-09-05 | envelope_screener | `scanAsync.when`의 `data` 분기에만 배너 삽입. loading/error 화면은 스크롤 뷰가 아니라 배너 미노출. 노출하려면 `when` 전체를 `Column`으로 감싸는 재구조화 필요 | 미구현 (사용자 판단 필요) |
| A3 | 2026-09-05 | mlb_stadiums_app, national_parks_app | 첫 탭이 전체화면 지도라 배너가 지도 높이를 줄임. 실제 화면 확인 후 조정 여부 판단 | ✅ 확인 완료 — 조정 불필요 |
| A4 | 2026-09-05 | 설정 (`cross_promo.json`) | `envelope_screener`, `remtime` 노출 0회 — 추천해주는 앱이 없음. 구현과 무관한 설정 이슈 | 미구현 (사용자 판단 필요) |
| A5 | 2026-09-05 | fire_calculator | `url_launcher`가 끌어온 `androidx.browser 1.9.0`이 AGP 8.9.1+ 요구. 이 앱만 AGP 8.7.3이라 빌드 실패 | ✅ 완료 |
| A7 | 2026-09-05 | 전 앱 + 설정 | 설명이 2줄로 넘어가 배너가 커짐. 실측 결과 좁은 앱(화면 패딩 24dp)의 텍스트 가용폭은 `화면폭 − 233dp`로, 360dp 기기에서는 127dp뿐이라 설명 단축만으로는 해결 불가 → 위젯에서 설명 제거 + 아이콘 44px로 카드 높이 72dp 고정 | ✅ 완료 |
| A6 | 2026-09-05 | 전 앱 | `--dart-define=ADS_ENABLED=false`가 동작하지 않음. AdMob 테스트 전면광고가 그대로 노출됨. 스크린샷 촬영 시 방해되며, 광고 토글 플래그가 코드에 없는 것으로 보임 | 미구현 (사용자 판단 필요) |

---

## 4. 업데이트 로그

> "추가 구현 계획" 항목이 실제 코드에 반영된 기록만 남긴다.

| 일자 | 섹션 | 반영 항목 | 변경 파일 | 줄 | 변경 내용 | 검증 |
|---|---|---|---|---|---|---|
| 2026-09-05 | Phase 2 | A5 | `fire_calculator/android/settings.gradle.kts` | L(plugins 블록) | AGP `8.7.3` → `8.9.1` (다른 7개 앱과 동일하게 정렬) | `flutter build apk --debug` 성공, APK 생성·설치·실행 확인 |

| 2026-09-05 | Phase 1~3 | A7 | `cross_promo.json` | 설명 34건 | 배너 1줄에 맞춰 설명 단축 (22건 + 12건) | 실측 기반, 394dp 기기에서 2줄 0개 |
| 2026-09-05 | Phase 1~3 | A7 | `cross_promo_banner.dart` × 6 | 위젯 전체 | 설명 표시 제거, 아이콘 50→44px | analyze 신규 0건, 테스트 78개 통과, 에뮬레이터 실측 카드 높이 **83dp → 70dp (16% 감소)**, 8개 앱 재촬영 확인 |

> A1·A2·A4·A6은 사용자 판단 대기 중이라 미반영. A3는 검증 결과 조정 불필요로 종결.
