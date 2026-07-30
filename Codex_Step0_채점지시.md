# Codex 지시 — Step 0 신뢰도 측정용 40건 채점 (그대로 전달)

> 저장소: `skycloud-maker/youtube-comment-analyzer` · 브랜치: `codex/voc-engine-rebuild`
> 목적: 1차 추출 신뢰도를 오염 없이 측정. 이 40건은 **사람이 라벨한 ground truth와 대조**할 대상 — 너는 **정답을 보지 말고** 원문만 분석해라.

## 입력
- 첨부: `VoC Step0 채점입력 (40 원문).xlsx` (컬럼: `source_id`, `raw_row_index`, `original_voc`) — **원문만, 사람 라벨 없음.**
- 이 댓글들은 **식기세척기·틔운·냉장고 등(로봇청소기 아님)**. 캐노니컬 topic 택소노미는 로봇청소기 전용이라 **여기 topic은 안 맞는 게 정상** — 억지로 로봇청소기 topic을 붙이지 말고, 범위 밖이면 규칙대로 N/A/scope_excluded로 두면 된다. **이번 측정은 제품 무관 필드(유효성·감성방향·문의·CEJ)만 본다.**

## 할 일
1. 현재 `codex/voc-engine-rebuild` 엔진(Round-5 상태)으로 40건을 **fresh 1회 분석**(저장 재사용 금지).
2. 60컬럼 출력 CSV를 그대로 저장(최소 다음 필드 포함): `source_id, raw_row_index, original_voc, status, topic, topic_score, inquiry_flag, inquiry_type, cej, tee_emotion_score, tee_success_score, tee_effort_score, tee_time_score, confidence_rating, atomic_voc_summary`.
3. 출력만 반환. **정답 라벨을 만들지 말 것**(우리가 사람 라벨과 대조한다).

## 산출물
- `outputs/voc_step0_scoring_40/voc_engine_output.csv` (+ jsonl)
- 같은 브랜치 커밋 OK, **main merge/deploy/repoint 금지.**

---

### 참고: 우리(Cowork)가 대조에 쓸 파생 규칙 (너는 구현 안 해도 됨 — 우리가 계산)
- **유효성**: `status==scope_excluded` 또는 `topic=="N/A"` → 무효, 아니면 유효.
- **감성(긍/부/중)**: `tee_emotion_score` 부호 우선(>0 긍정 / <0 부정), 0이면 `tee_success_score` 부호, 둘 다 0이면 중립.
- **문의**: `inquiry_flag`(Y/N).
- **CEJ**: `cej`(해당없음 라벨 포함).
- **채점 게이팅**: 유효성은 40건 전부 채점. 감성·문의·CEJ는 **사람이 ‘유효’로 본 행에서만** 채점(무효 행은 제외).
