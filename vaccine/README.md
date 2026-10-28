# vaccine — 제때접종 원격 데이터

앱 `제때접종`(`~/Documents/vaccine_schedule`, `com.joe.vaccineschedule`)이 받는 파일입니다.
앱은 `https://raw.githubusercontent.com/jinhom6225/app-config/main/vaccine/<파일>`만 받습니다(`lib/core/config/app_links.dart`).

| 파일 | 내용 | 앱 번들 원본 |
|---|---|---|
| `schedule_rules.json` | 2026 국가예방접종 지침의 표준·지연접종 규칙, 인플루엔자 절기 | `assets/data/schedule_rules.json` |
| `vaccine_info.json` | 백신별 설명(왜 맞는지, 늦어지면) | `assets/data/vaccine_info.json` |
| `hospitals_meta.json` | 병원 데이터 version·dataDate·bytes·sha256 (**아직 없음**, Phase 3 뒤) | — |
| `hospitals.json.gz` | 접종 병원 목록 (**아직 없음**, Phase 3 뒤) | — |

- 앱은 `version`이 지금 쓰는 데이터보다 **클 때만** 바꿉니다. 같으면 "변경 없음", 작으면 거부합니다. **version은 절대 내리지 않습니다.**
- 커밋 전에 앱 repo에서 검증 CLI가 오류 0이어야 합니다(`dart run tool/validate_rules.dart` 등).
- 수정·커밋: 명박사. **push: 진호님.**
- 절차 전체는 앱 repo의 `data_runbook.md`를 봅니다.
