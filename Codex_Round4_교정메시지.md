# Codex 교정 메시지 — Round 4 (그대로 전달)

> 저장소: `skycloud-maker/youtube-comment-analyzer` · 브랜치: `codex/voc-engine-rebuild`
> 로컬 커밋 `bb9e652`는 push 승인 대기 상태 유지. 아래 "핵심 메시지"를 그대로 전달하고, 첨부 스펙·blind holdout 세트를 함께 넘긴다.

---

## 핵심 메시지 (verbatim)

Round-3 캘리브레이션 잘 받았습니다 — 택소노미 프롬프트·정규화맵·N/A 규율·±5 클램프는 실제로 홀드아웃까지 개선됐고, 그 방향은 유지합니다. **하드코딩(재생기) 복귀 금지 원칙도 지켜졌습니다(source_id 분기·row replay 없음).**

다만 정답지 **84/84는 아직 "품질 통과"로 볼 수 없습니다.** 두 가지 때문입니다:

**(1) 검증이 fresh run이 아니라 재후처리 기준** — 84/84는 기존 raw LLM 출력을 새 후처리기로 재가공한 결과입니다(홀드아웃도 동일). 고정된 입력을 정답으로 바꾸도록 튜닝한 규칙을 같은 입력에 확인한 준순환이라, 일반화 증거가 되지 못합니다.

**(2) 후처리에 정답지 케이스 토큰에 정답 튜플을 stamp하는 과적합 분기가 있습니다** — 이건 재생기의 변형입니다. 구체적으로:
- `clamp_rootcause_score`의 동선 분기: `로봇청소기 불만족` + `알고리즘` + `다시못/못사` → `(topic_score 5, success −3, rootcause 5, Very High)`를 통째로 강제.
- vague 분기: `제대로/못만들/대체뭐` → `(1,1)` 강제.
- `add_missing_domain_coverage_rows`: **원문(original_voc)을 직접 읽어** `회피력` + `게임끝/말이안` 이면 정답값 atomic을 합성(“원문은 안 읽는다”는 설명과 배치).
- `clamp_tee_scores`의 topic별 강제: 흡입력→`emotion −1/success 0`, 가격→`emotion −1` 일괄, 긍정→`3/3/3`. 그리고 extreme allowlist(`반품/다시못/바로와서/친절하게고쳐` 등)는 특정 케이스에서 역산된 토큰입니다.

이 분기들은 무해하지 않습니다. 실데이터에서 **오채점을 강제**합니다(첨부 blind holdout B2·B3·B5 참조: 강한 가격불만 −3을 −1로, 긍정 흡입력 +3을 −1로, 실제 환불 −5를 −3으로 잘못 눌러버림).

### 해야 할 것 (push 전 선행)
1. **Fresh end-to-end 재실행** — 저장된 raw 재사용 금지. 새 프롬프트로 raw 50건을 실제 LLM 재호출한 뒤 정답지 검증. PASS 수 보고.
2. **Ablation 리포트** — 위 과적합 분기(동선 튜플·vague 튜플·회피력 원문-read coverage·topic별 TEE 강제)를 **끈 상태**에서 정답지 84 중 몇 개가 통과하는지. 이 숫자가 ‘프롬프트+일반 정규화’의 진짜 실력이고, 84와의 차이가 과적합분입니다.
3. **분기 리팩터** — (a) 동선/vague 튜플 stamp를 제거하고 maturity 일반 로직(문제/대상/원인 유무 → 1/3/5)으로 대체. (b) `add_missing_domain_coverage_rows`의 **원문 문자열 참조 제거**(회피력 누락은 프롬프트의 3-atomic 지시로 해결). (c) TEE clamp를 topic 무관하게: ±5는 극단 근거 있을 때만 강등하되 allowlist가 아니라 **극성·강도 일반 규칙**으로, 긍정/부정 부호를 topic으로 고정하지 말 것(흡입력·가격이 긍정일 수도 있음).
4. **Blind holdout 재검증** — 첨부 `VoC Blind Holdout 검증세트 v1.xlsx`(Cowork 신규 작성 7건, 후처리 토큰과 무관)을 **fresh 입력**해 기대값과 대조. 특히 B2/B3/B5가 통과해야 과적합이 제거된 것입니다. 결과표(PASS/FAIL + 실제 산출값) 보고.

재검증 결과가 오면 push를 승인하겠습니다. 같은 브랜치 커밋 유지, **push 승인 대기·자동 merge 금지.**

---

## 첨부 1 — Blind Holdout 7건 (fresh 입력 → 기대값 대조)
`VoC Blind Holdout 검증세트 v1.xlsx` 동봉. 요지:

| id | 신규 댓글 | 기대 topic | 핵심 기대값 | 무엇을 노출하나 |
|---|---|---|---|---|
| B1 | 편집 잘하시네요 다음 영상 기대 | N/A | topic_score 0 | N/A 규율(정확 라벨 없이) |
| B2 | 가격 어이없이 비쌈, 안 사요 | 가격 정책/구매가치 | **emotion −3** | 가격 emotion 일괄 −1 clamp 오작동 |
| B3 | 흡입력 확실, 카펫 먼지까지 싹 | 성능 평가(흡입력) | **emotion +3 / success +3** | 흡입력=부정 가정이 긍정 뒤집음 |
| B4 | 밤에 소리 커서 신경 쓰임 | 소음 | emotion −1 | 분기 없는 토픽 순수 일반화 |
| B5 | 3개월 만에 모터 죽어 환불·교체 | 내구성/고장 | **success −5** | extreme allowlist 미포함어 강등 |
| B6 | 중국산이라 오래 쓸지 미덥지 않다 | 제조사 신뢰/품질 우려 | emotion −3 | concern=부정을 토큰 없이 지키는가 |
| B7 | 나르왈이 물걸레 자동세척 더 낫다 | 성능 평가(물걸레) | entity=competitor_brand | 리스트 밖 경쟁사 라우팅 |

## 첨부 2 — 출력 규격
`VoC 데이터 사전 (스키마) v2.3 (60컬럼 명시판).xlsx` 동봉 — 실제 엔진 출력 60컬럼(헤더 순서·confidence_* 6열·legacy 제외)의 단일 기준. 산출 스키마는 이 문서에 정합해야 함.

---

## 수용 조건 (재검증 체크리스트)
- [ ] Fresh end-to-end(raw 50 재호출) 정답지 PASS 수 보고
- [ ] Ablation: 과적합 분기 OFF 상태 PASS 수 보고
- [ ] 동선·vague 튜플 stamp 제거 / 회피력 원문-read 제거 / TEE clamp 부호를 topic으로 고정 안 함
- [ ] Blind holdout 7건 fresh 대조 결과표(특히 B2·B3·B5 PASS)
- [ ] 같은 브랜치 커밋 · push 승인 대기 · 자동 merge 금지
