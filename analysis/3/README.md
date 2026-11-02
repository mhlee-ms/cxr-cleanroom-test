# CLEAR 심화 분석: AP vs PA

근거 계획: `analysis/3/plan.md` (이슈 #3). 논문: [CLEAR, Nature Biomedical Engineering 2026](https://doi.org/10.1038/s41551-026-01741-4).
이 문서의 모든 수치는 `llm.hospital`에 대한 집계 질의 결과이며, 환자 단위 행은 포함하지 않는다. 결과는 가설 생성용이다.

## 논문이 보고한 것

출처: [논문 초록](https://doi.org/10.1038/s41551-026-01741-4)
- 239,391명, 87만 쌍 이상의 영상-보고서로 학습한 개념 기반 모델이다.
- 4개 외부 데이터셋에서 분류 성능을 검증했고, 방사선학적 교란 요인을 체계적으로 찾는 응용을 제시했다.

## 우리 데이터가 보여주는 것

### 코호트 균형표

| | AP | PA |
|---|---|---|
| 장수 / 환자 수 | 88 / 56 | 184 / 130 |
| 이상 소견 | 45 (51.1%) | 82 (44.6%) |
| 복합 소견 | 23 (26.1%) | 27 (14.7%) |
| 단독 소견 | 22 (25.0%) | 55 (29.9%) |
| 응급 | 12 (13.6%) | 40 (21.7%) |
| 입원 | 24 (27.3%) | 50 (27.2%) |
| 평균 연령 (SD) | 47.9 (15.3) | 53.3 (17.2) |
| 남 / 여 | 49 / 39 | 87 / 97 |

기관별 (AP/PA): INST01 17/45, INST02 20/36, INST03 13/41, INST04 21/27, INST05 17/35.
장비 9종이 두 자세 모두에 있다 (예: DEV09는 AP 4, PA 27).
영상 크기 종류는 AP 5가지, PA 59가지다.

방문 유형별 이상 소견 수 (AP / PA): 응급 9/12 vs 14/40, 입원 12/24 vs 28/50, 외래 9/27 vs 26/57, 원격 15/25 vs 14/37.

소견별 장수 (AP/PA): 무기폐 16/16, 삼출 17/16, 침윤 20/25, 심비대 2/8, 결절 2/10, 섬유화 1/12, 기흉 3/9, 폐렴 3/0.

짝지은 환자: AP·PA를 모두 가진 환자는 33명이다. 이 중 이상 소견이 AP에만 있는 환자 7명, PA에만 있는 환자 10명, 둘 다 있는 환자 10명, 둘 다 정상인 환자 6명이다.

### 쿼리

```sql
-- 자세별 구성
select view_position, count(*) n, count(*) filter (where findings_label<>'No Finding') abn,
 count(*) filter (where findings_label like '%|%') multi,
 count(*) filter (where findings_label<>'No Finding' and findings_label not like '%|%') single,
 count(*) filter (where visit_type='emergency') em, count(*) filter (where visit_type='inpatient') inp,
 round(avg(age),1) age, round(stddev(age),1) sd,
 count(*) filter (where sex='M') m, count(*) filter (where sex='F') f
from llm.hospital group by 1;
-- 기관, 장비, 방문유형 교차표
select view_position, institution_code, count(*) from llm.hospital group by 1,2;
select view_position, device_id, count(*) from llm.hospital group by 1,2;
select view_position, visit_type, count(*), count(*) filter (where findings_label<>'No Finding') from llm.hospital group by 1,2;
-- 소견별
select view_position,label,count(*) from (select view_position, unnest(string_to_array(findings_label,'|')) label from llm.hospital) t where label<>'No Finding' group by 1,2;
-- 환자 수, 짝지은 환자
select view_position,count(*),count(distinct patient_key) from llm.hospital group by 1;
select count(*) from (select patient_key from llm.hospital group by 1 having count(distinct view_position)=2) t;
-- 영상 크기
select view_position, count(distinct img_width||'x'||img_height) from llm.hospital group by 1;
```

## 추론 (우리의 해석)

- 두 군은 이상 비율은 비슷하지만 복합 소견은 AP에서 더 많고, 응급·입원 비율은 PA에서 더 높다. 중증도 대리 지표를 무엇으로 잡느냐에 따라 방향이 달라진다.
- 같은 환자가 두 자세로 촬영된 경우도 33명뿐이라 자세 효과를 환자 상태와 분리하기에는 부족하다.
- 플랜의 조정 로지스틱 분석(항목 5)은 AP에서 소견 표본이 극소수이고 환자 군집 보정이 필요해 이번에는 수행하지 않았다. 아래 Limitations 참조.

## Delegated to CXR Lens

| 계획 항목 | 담당 `cxr` run |
|---|---|
| 3. 자세별 AUROC·민감도·특이도와 부트스트랩 95% CI | `ap`, `pa` run의 metrics |
| 4. 중증도 층(단독/복합/없음)별 점수·오분류율 | `ap`, `pa` run의 subgroup metrics |
| 6. Grad-CAM 오류 사례 갤러리 | `ap`, `pa` run의 evidence maps (로컬 뷰어) |

## Limitations

- 개념 분해(CLEAR)는 수행하지 않았고 DenseNet만 대상이다. CLEAR 공개 가중치 사용은 확인되지 않았다.
- 개별 소견은 AP에서 1–3장 수준이 많아 소견별 비교는 불가능하다.
- 중증도 변수가 없고, 라벨 출처(영상 판독 vs 보고서)를 알 수 없다.
- 계획 항목 5(조정 로지스틱)는 `query-medical-db`로 계산할 수 없어 수행하지 않았다.
- 통계 검정은 하지 않았다. 표의 차이는 기술적 수치다.
