# Codex 교정 지시 — Round 6 (그대로 전달)

> 저장소: `skycloud-maker/youtube-comment-analyzer` · 브랜치: `codex/voc-engine-rebuild`
> 배경: Step 0(가전 40건) 측정에서 2가지 구조 문제 확인 — ①로봇청소기 전용 택소노미가 타 가전에서 topic을 N/A로 강제 ②그 N/A가 감성·CEJ까지 wipe. + CEJ 판정을 내용기준으로 확정. **케이스-피팅/blind-token 후처리 재도입 금지(Round-5 정직 baseline 유지).**

## A. N/A 이원화 — "노이즈 N/A" vs "미분류(substantive)"
현재 `normalize_topic_label`은 캐노니컬에 없으면 무조건 topic=N/A로 만들고, `apply_na_and_promo_discipline`이 N/A면 감성·rootcause·CEJ까지 0/해당없음으로 wipe한다. → **실질 VoC인데 토픽만 못 정한 경우까지 wipe**되는 게 문제.

**수정:**
1. `CANONICAL_TOPICS`에 **`미분류`** 추가 = *실질 VoC지만 캐노니컬 topic이 없음*(타 카테고리 제품, 아직 택소노미에 없는 주제 등).
2. `normalize_topic_label`에서 캐노니컬에 없을 때 **이원 분기**:
   - **저가치/노이즈**(인사·감탄·채널/영상 칭찬·후기요청·무의미·범위 밖 비제품) → **N/A** (기존 `LOW_VALUE_TOPIC_ALIASES` 규칙 유지).
   - **실질 VoC**(제품·구매·사용 관련 의견/경험/문의) → **`미분류`** (N/A 아님).
3. **wipe 게이트를 `topic=="N/A"`에만** 적용. **`미분류`는 일반 topic처럼 취급** — 감성·CEJ·inquiry·rootcause 정상 유지, `topic_score ≥ 1`.
4. 프롬프트에 명시: *"topic=N/A는 진짜 노이즈일 때만. 실질 VoC인데 목록에 맞는 topic이 없으면 `topic=미분류`로 두고 감성·CEJ·문의·rootcause는 정상 판정하라."*

## B. CEJ 내용기준 규칙 (확정)
프롬프트의 CEJ 지시를 아래로 교체·보강:
- **CEJ = 이 VoC의 이슈/경험이 속한 여정 단계(= 개선이 이뤄질 단계)이지, 작성자의 현재 쇼핑 위치가 아니다.**
- **실사용 vs 추정은 CEJ로 구분하지 말 것** — `success` 점수(실사용 근거 있을 때만, R1)·`confidence`로 구분.
- 예: "사용하면 불편할 듯"(비사용자 추정) → **CEJ 사용 + success 0** / "홍보·인지가 잘못" → **인지** / "살지 고민·비교 정보 헷갈림" → **탐색·결정** / "링크 안 열림"(운영) → **해당없음**.
- 독립된 두 단계 이슈가 섞이면 **atomic 분리**(각자 CEJ).
- 후처리 `normalize_cej` 토큰 규칙은 이 원칙과 충돌 않게 최소화(환불/교체=교체 정도만 유지).

## C. 유지/금지
- Round-5 정직 baseline 유지. **정답지/blind 케이스 토큰 후처리 재도입 금지.**
- `미분류` 도입이 로봇청소기 정답지(raw50)를 깨지 않는지 확인 — RV는 캐노니컬 topic이 있으므로 미분류로 빠지면 안 됨(회귀 확인).

## D. 채점 작업 — 120건 (fresh, blind)
- 첨부: `VoC Step0 채점입력 (120 원문).xlsx` (컬럼 `source_id`, `raw_row_index`, `original_voc` — **라벨 없음**).
- 위 A~C 반영 엔진으로 **fresh 1회** 분석 → 60컬럼 출력 CSV 반환.
- 식기세척기·틔운 등 타 가전 포함 → 캐노니컬 topic 없으면 **미분류**(N/A 아님)로, 감성·CEJ·문의는 정상 판정.
- **정답 라벨 만들지 말 것**(우리가 사람 라벨과 대조).

## E. 재검증 & 산출
- raw50 정답지 재실행(미분류 도입 후 회귀 없나) + 120 출력.
- 산출: `outputs/voc_round6_120/voc_engine_output.csv` (+ jsonl), raw50 validation results.
- 같은 브랜치 커밋 · **main merge / deploy / dashboard repoint 금지.**

---
### 참고: 우리(Cowork)가 0-4 대조에 쓸 규칙 (구현 불필요)
- **유효성**: `status==scope_excluded` 또는 `topic=="N/A"` → 무효; `topic==미분류` 또는 캐노니컬 → **유효**.
- **감성/문의/CEJ**: 사람 '유효' 행에서만 채점. CEJ는 **내용기준**으로 대조.
