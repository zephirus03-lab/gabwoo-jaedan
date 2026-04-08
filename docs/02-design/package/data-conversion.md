# Data Conversion Design — 패키지 생산현황 조회 서비스

**Feature**: package-production-viewer
**Phase**: 02-design
**작성일**: 2026-04-08
**대상 파일**: `convert.py` (루트)
**입력**: 3개 엑셀 (읽기 전용, 수정 절대 금지)
**출력**: `web/data.json`

---

## 1. 설계 원칙

1. **원본 엑셀은 절대로 수정/이동/덮어쓰기 하지 않는다** (CLAUDE.md 최우선 규칙).
2. **Fail loud, preserve prior state**: 변환 중 오류 발생 시 즉시 종료하고 기존 `web/data.json`은 그대로 유지한다. 부분적으로 잘못된 JSON을 덮어쓰는 것이 최악의 시나리오.
3. **수주관리 = 진실의 원천**. 인쇄종합/출고리스트는 배지(badge) 부착 용도의 보조 데이터.
4. **느슨한 조인**: 수주번호(관리번호) 우선, 실패 시 품명 기반 fuzzy, 그래도 실패 시 배지 생략 (데이터는 별도 리스트로 살림).
5. **읽기 모드 최적화**: 18.7MB 인쇄종합 파일은 반드시 `read_only=True, data_only=True` + `iter_rows()` 스트리밍.
6. **실제 업무 데이터**: 매 실행마다 샘플 카운트 로그 출력 → 운영자가 육안 대조 가능.

---

## 2. 입력 파일 요약

| 파일 | 경로 | 역할 | 파싱 대상 |
|---|---|---|---|
| 수주관리 | `생산진행현황/수주관리 2026년도.xlsx` | 진실의 원천 | `수주관리 2026. NN월` 패턴 모든 월별 시트 |
| 인쇄종합 | `생산진행현황/인쇄종합_260326_REV01-*.xlsx` | 오늘 인쇄 배지 | `인쇄일정관리` 시트 1개만 |
| 출고리스트 | `생산진행현황/출고품목리스트 (4월).xlsx` | 오늘 출고 배지 | `MMDD` 패턴 중 최근 시트 1개만 |

> 인쇄종합 파일명은 한글/기호 포함하므로 `glob.glob('생산진행현황/인쇄종합_*.xlsx')`로 와일드카드 매칭한다.

---

## 3. Target JSON Schema

```jsonc
{
  "updated_at": "2026-04-08",           // 가장 최근 데이터 기준일
  "generated_at": "2026-04-08 10:15",   // convert.py 실행 시각

  "summary": {
    "in_progress": 212,                 // orders 중 completed_at == null
    "printing_today": 12,               // printings_today 개수
    "shipping_today": 18,               // shipments_today 개수
    "completed_7d": 87,                 // 최근 7일(오늘 포함) 완료된 orders
    "recent_7d_chart": {                // 완료일별 카운트 (7개 키 보장)
      "2026-04-02": 11,
      "2026-04-03": 13,
      "2026-04-04": 0,
      "2026-04-05": 0,
      "2026-04-06": 15,
      "2026-04-07": 20,
      "2026-04-08": 9
    },
    "clients_in_progress": {            // 진행중 거래처별 건수 내림차순
      "코스맥스": 34,
      "코스메카코리아": 22
    }
  },

  "orders": [
    {
      "mgmt_no": "260402-0005",
      "manager": "김병주",
      "registered_at": "2026-04-02",
      "client": "(주)비앤에프엘엔씨",
      "product": "리아_AO클렌저100G_단상자",
      "qty": 10000,                     // 수주량 (int, null 허용)
      "net_qty": 10500,                 // 정미량 (int, null 허용)
      "deadline": "2026-04-14",         // 납기예정 (YYYY-MM-DD 또는 null)
      "process_spec": "옵셋인쇄,톰슨,단면접착",  // 원문 공정 텍스트
      "applicable_stages": ["인쇄", "톰슨", "접착"],  // 공정 텍스트에서 추출한 표준 스테이지
      "progress": {                     // 7개 키 고정, 날짜(YYYY-MM-DD) 또는 null
        "인쇄": "2026-04-07",
        "코팅": null,
        "실크": null,
        "박":   null,
        "형압": null,
        "톰슨": null,
        "접착": null
      },
      "completed_at": null,             // 생산일 값 (null이면 진행중)
      "rework": null,                   // 재작업 텍스트
      "note": "",                       // 비고
      "source_month": "2026.04",        // 어떤 월별 시트에서 왔는지
      "badges": {
        "printing_today": true,         // 인쇄일정관리.계획일 == 오늘
        "shipping_today": false         // 출고리스트 최신 시트와 매칭
      }
    }
  ],

  "printings_today": [                  // 인쇄일정관리 필터 결과 (원본 보존)
    {
      "plan_date": "2026-04-08",
      "day": "화",
      "manager": "김병주",
      "mgmt_no": "260402-0005",
      "client": "코스맥스",
      "process": "옵셋인쇄",
      "process_detail": "4도 양면",
      "product": "리아_AO클렌저100G_단상자",
      "deadline": "2026-04-14",
      "qty": 10000,
      "sets": 5,
      "net_qty": 10500,
      "note": "CTP 완료"
    }
  ],

  "shipments_today": [                  // 출고리스트 최신 시트 원본
    {
      "sheet_date": "2026-04-08",
      "client": "알래스카애드",
      "product": "오로라 PN 프로",
      "qty": 2300,
      "destination": "본사창고",
      "status": "톰슨대기"
    }
  ]
}
```

### 스키마 규칙
- **날짜**: 모두 `YYYY-MM-DD` 문자열 또는 `null`.
- **수량**: `int` 또는 `null`. (문자열/콤마는 파싱 단계에서 정수화)
- **progress**: 7개 키(`인쇄/코팅/실크/박/형압/톰슨/접착`) 항상 존재. 값은 날짜 또는 null.
- **applicable_stages**: `process_spec`을 공정 매핑 테이블로 정규화한 리스트. 매칭 실패 시 빈 리스트가 아니라 7개 전체(기본 표시) — fallback 정책은 `extract_progress` 참조.

---

## 4. 공정 매핑 테이블

수주관리 `공정` 컬럼은 자유 텍스트(`"옵셋인쇄,톰슨,단면접착"`, `"UV인쇄 + 무광코팅 + 금박"` 등) → 7개 표준 스테이지로 정규화.

| 표준 스테이지 | 매칭 키워드 (부분 문자열, 소문자 변환 후 비교) |
|---|---|
| 인쇄 | `인쇄`, `옵셋`, `uv인쇄` |
| 코팅 | `코팅`, `라미`, `ir`, `무광`, `유광` |
| 실크 | `실크`, `부분uv`, `부분 uv` |
| 박   | `박`, `먹박`, `금박`, `홀로박` (단, "박스"는 제외) |
| 형압 | `형압`, `양각`, `음각`, `엠보` |
| 톰슨 | `톰슨`, `타발` |
| 접착 | `접착`, `이면접착`, `삼면접착`, `단면접착`, `날개접착` |

### 매칭 알고리즘
1. 입력 텍스트를 소문자 변환 (`text.lower()`).
2. 각 표준 스테이지에 대해 해당 키워드가 하나라도 포함되면 스테이지 활성.
3. **"박" 특수 케이스**: `"박스"` 같은 오탐지를 막기 위해 `"박"` 키워드는 앞뒤가 한글 영역이거나 단독 토큰일 때만 매치. (구현 힌트: `"박" in text`가 아니라 토큰 분리 후 비교, 또는 키워드 블랙리스트 `["박스"]` 검사)
4. 매칭 결과가 공백(`[]`)이면 **fallback**: 7개 스테이지 전부를 `applicable_stages`로 반환. 이유는 원문이 비어있거나 형식 파괴일 때 UI에서 최소한 전체 공정 바라도 보여주기 위함.

---

## 5. Parsing Strategy

### 5.1 `parse_orders(filepath)` — 수주관리

**입력**: `생산진행현황/수주관리 2026년도.xlsx`
**출력**: `list[dict]` (각 dict는 위 schema의 `orders[]` 요소, `badges`는 이 단계에서 `{printing_today: false, shipping_today: false}` 기본값)

알고리즘:
1. `openpyxl.load_workbook(path, read_only=True, data_only=True)`.
2. `wb.sheetnames`에서 정규식 `r"수주관리\s*2026\.\s*(\d{1,2})월"` 매치 시트만 선별. 월 번호 오름차순 정렬.
3. 각 시트에 대해:
   a. `ws.iter_rows(min_row=1, values_only=True)`로 전체 순회.
   b. 헤더 탐지: 1행이 "No", "담당", "등록일"... 등인지 확인. 헤더 행이 2행일 수도 있으므로 `"관리번호"` 단어가 등장하는 첫 행을 `header_row`로 동적 탐지.
   c. `header_row` 다음 행부터 데이터. 각 행에 대해 20개 컬럼 인덱스 고정 매핑:

      | idx | 컬럼 | 필드 |
      |---|---|---|
      | 0 | No | (무시 / 행 번호만 확인) |
      | 1 | 담당 | `manager` |
      | 2 | 등록일 | `registered_at` |
      | 3 | 관리번호 | `mgmt_no` (필수 키, 없으면 행 skip) |
      | 4 | 매출처 | `client` |
      | 5 | 품명 | `product` |
      | 6 | 수주량 | `qty` |
      | 7 | 정미량 | `net_qty` |
      | 8 | 납기예정 | `deadline` |
      | 9 | 공정 | `process_spec` |
      | 10 | 인쇄 | `stage_dates["인쇄"]` |
      | 11 | 코팅 | `stage_dates["코팅"]` |
      | 12 | 실크,부분uv | `stage_dates["실크"]` |
      | 13 | 박 | `stage_dates["박"]` |
      | 14 | 형압 | `stage_dates["형압"]` |
      | 15 | 톰슨 | `stage_dates["톰슨"]` |
      | 16 | 접착 | `stage_dates["접착"]` |
      | 17 | 생산일 | `completed_at` |
      | 18 | 재작업 | `rework` |
      | 19 | 비고 | `note` |

   d. Skip 조건:
      - `mgmt_no`(관리번호) 비어 있음
      - `client`와 `product` 둘 다 비어 있음
      - 헤더/소제목 행 (`mgmt_no`가 "관리번호" 같은 텍스트)
   e. 각 값 정규화:
      - 날짜: `datetime → strftime("%Y-%m-%d")`, 이외 `None`.
      - 수량: `int(value)` 시도, 실패 시 `None`.
      - 텍스트: `str(value).strip()`, 빈 문자열은 보존(`""`).
   f. `extract_progress(process_spec, stage_dates)` 호출 → `{progress, applicable_stages}`.
   g. `source_month = "2026.{MM:02d}"`.
4. 월별 결과 병합. 중복 `mgmt_no`가 나오면 **뒤에 오는(더 최근 월) 레코드가 이긴다** + 경고 로그.
5. 반환.

### 5.2 `parse_printings_today(filepath)` — 인쇄종합

**입력**: `생산진행현황/인쇄종합_*.xlsx` (glob으로 해결)
**출력**: 오늘 인쇄 예정 행 리스트

알고리즘:
1. **성능 중요**: `load_workbook(path, read_only=True, data_only=True)`. 18.7MB이므로 다른 시트는 건드리지 않는다.
2. `wb["인쇄일정관리"]` 획득. `wb.close()`는 함수 종료 시.
3. `ws.iter_rows(values_only=True)`:
   - row 1: 섹션 타이틀 → skip
   - row 2: 헤더 → 컬럼 인덱스 매핑 구축 (`"계획일"` 등장 인덱스 탐색)
   - row 3+: 데이터
4. 헤더 매핑 (예상 컬럼):
   `계획일, Day, 영업자, 특이사항, 수주번호, 매출처, 공정, 제품명, 공정상세, 납품예정, 수주량, 벌수, 정미량, 정미+여분, 용지입고, CTP출력, 지류명, 비고, 공정`
   (마지막 "공정"이 중복되는 경우 첫 번째만 사용)
5. 필터: `계획일`이 `datetime`이고 `.date() == today`인 행만 채택.
6. 각 행 → `printings_today` dict로 정규화 (위 스키마 참조).
7. 계획일이 전혀 안 맞으면 빈 리스트 반환 (오류 아님).

### 5.3 `parse_shipments_today(filepath)` — 출고리스트

**입력**: `생산진행현황/출고품목리스트 (4월).xlsx`
**출력**: 최신 `MMDD` 시트의 출고 행 리스트

알고리즘:
1. `load_workbook(path, read_only=True, data_only=True)`.
2. `wb.sheetnames` 중 `re.fullmatch(r"\d{4}", name)` 매치 시트만 후보.
3. `int(name)`이 가장 큰 시트를 최신으로 선택 (월 내에선 단순 큰 수가 최신).
4. 행 구조:
   - row 1: "출고 품목 리스트" 타이틀 → skip
   - row 2: 날짜 (셀에 문자열/날짜 가능)
   - row 3: 헤더 `업체명|제품명|출고수량|출고처|비고`
   - row 4+: 데이터
5. 각 데이터 행 → 정규화. `qty`는 int. 빈 `업체명` 행은 skip.
6. `sheet_date`는 row 2에서 추출 (datetime이면 포맷, 문자열이면 `str` 그대로, 실패시 시트명 `MMDD`로 fallback → `2026-MM-DD`).

### 5.4 `extract_progress(gongjeong_text, stage_dates)`

**입력**:
- `gongjeong_text`: `"옵셋인쇄,톰슨,단면접착"`
- `stage_dates`: `{"인쇄": datetime or None, "코팅": None, ...}` (7개 키 고정)

**출력**:
```python
{
  "progress": {"인쇄": "2026-04-07", "코팅": null, ..., "접착": null},
  "applicable_stages": ["인쇄", "톰슨", "접착"]
}
```

알고리즘:
1. 7개 스테이지 각각에 대해 매핑 테이블로 해당 스테이지가 `gongjeong_text`에 언급됐는지 판정 → `applicable_stages` 산출.
2. `applicable_stages`가 비면 fallback으로 7개 전체 리턴.
3. `progress[stage]`는 `stage_dates[stage]`를 `YYYY-MM-DD` 문자열로 변환 (None 유지).
4. 주의: `applicable_stages`에 없더라도 `progress` 7개 키는 모두 포함 (UI가 "이 공정은 N/A"로 흐리게 표시하게 함).

### 5.5 `merge_badges(orders, printings, shipments)`

**입력**: 3개 파싱 결과
**출력**: `orders` (badges 필드가 갱신된 동일 리스트)

알고리즘:
1. **printing_today 부착**:
   - `printing_mgmt_nos = {p["mgmt_no"] for p in printings if p["mgmt_no"]}`
   - 각 `order`에 대해 `order["mgmt_no"] in printing_mgmt_nos` → `badges.printing_today = True`.
2. **shipping_today 부착** (fuzzy):
   - 제품명 정규화 함수: `normalize(s) = s.replace(" ", "").replace("_", "").lower()`
   - `shipment_products = {normalize(s["product"]): s for s in shipments}`
   - 각 `order`에 대해 `normalize(order["product"]) in shipment_products` → `badges.shipping_today = True`.
   - 매칭 실패한 shipment는 그대로 `shipments_today` 섹션에 남는다 (UI에서 리스트로 노출).
3. orders를 그대로 반환 (in-place 수정 OK, 호출자가 명확히 받음).

### 5.6 `compute_summary(orders, printings, shipments)`

**출력**: schema의 `summary` 블록.

알고리즘:
1. `in_progress = sum(1 for o in orders if o["completed_at"] is None)`
2. `printing_today = len(printings)`
3. `shipping_today = len(shipments)`
4. `recent_7d_chart`:
   - `today = datetime.now().date()`
   - 7일치 키(`today - 6`일부터 `today`까지) 전부 0으로 초기화.
   - 각 `order["completed_at"]`이 이 범위면 카운트 증가.
5. `completed_7d = sum(recent_7d_chart.values())`
6. `clients_in_progress`: 진행중 주문만 `client`별 카운트, 내림차순 정렬한 dict.

### 5.7 `main()`

알고리즘:
1. 3개 파일 경로 확인 (`os.path.exists`, 인쇄종합은 glob).
2. 없으면 `sys.exit(1)` with 명확한 메시지.
3. `try` 블록 안에서 4개 파서 + merge + summary 호출.
4. 출력 dict 조립 (`updated_at` = 가장 최근 `completed_at` 또는 오늘 날짜 중 max).
5. **원자적 쓰기**: 먼저 `web/data.json.tmp`로 기록 → `os.replace(tmp, "web/data.json")`. 중단돼도 기존 파일 보존.
6. 단계별 카운트 로그 출력 (`print`):
   ```
   수주관리: 225건 (2026.01~04월)
   인쇄일정 오늘(2026-04-08): 12건
   출고리스트 최신(0408): 18건
   배지 부착: printing_today=12, shipping_today=16
   요약: 진행중 212, 완료(7일) 87
   → web/data.json (xxx KB)
   ```
7. 예외 발생 시: 스택트레이스 출력 후 `sys.exit(2)`. 기존 `data.json`은 손대지 않았으므로 자동 보존.

### Function Signatures (정리)

```python
def parse_orders(filepath: str) -> list[dict]: ...
def parse_printings_today(filepath: str) -> list[dict]: ...
def parse_shipments_today(filepath: str) -> list[dict]: ...
def extract_progress(gongjeong_text: str, stage_dates: dict) -> dict: ...
def merge_badges(
    orders: list[dict],
    printings: list[dict],
    shipments: list[dict],
) -> list[dict]: ...
def compute_summary(
    orders: list[dict],
    printings: list[dict],
    shipments: list[dict],
) -> dict: ...
def main() -> None: ...
```

---

## 6. Error Handling

| 상황 | 처리 |
|---|---|
| 파일 없음 | 즉시 `sys.exit(1)` + 메시지. data.json 보존. |
| 시트 없음 (`인쇄일정관리`, 월별 시트 0개) | 파서는 경고 로그 + 빈 리스트. main은 계속 진행. |
| 헤더 탐지 실패 | 예외 raise → main에서 catch → exit(2). |
| 셀 타입 불일치 (날짜 기대 위치에 문자열) | 해당 필드만 `None`, 전체 행은 유지. |
| 수주량/정미량 파싱 실패 | `None`, 전체 행 유지. |
| 관리번호(`mgmt_no`) 중복 | 뒤에 오는 월이 승리 + `print("WARN: duplicate mgmt_no ...")`. |
| 공정 텍스트 매칭 0개 | fallback으로 7개 스테이지 모두 `applicable_stages`. |
| 출고리스트 fuzzy 매칭 실패 | badge만 생략, 해당 shipment는 `shipments_today`에 남음. |
| 변환 중 임의 예외 | `traceback.print_exc()` + `sys.exit(2)`. **`web/data.json`에 쓰기 시도 전이므로 기존 파일 자동 보존**. |

**핵심**: `with open(output_path, "w")` 대신 임시 파일 → `os.replace` 패턴을 써서 부분 쓰기 리스크를 원천 제거.

---

## 7. Performance

| 파일 | 크기 | 전략 | 예상 시간 |
|---|---|---|---|
| 수주관리 | ~수 MB, 4개 시트 × ~225행 | `read_only=True, data_only=True` + `iter_rows(values_only=True)` | < 5초 |
| 인쇄종합 | **18.7MB**, 18 sheets | `read_only=True, data_only=True`. `wb["인쇄일정관리"]` 한 시트만 `iter_rows` 스트리밍. 다른 시트는 절대 접근 금지. | < 20초 |
| 출고리스트 | ~수 MB | `read_only=True`. 최신 시트 1개만 | < 3초 |

**전체 목표**: < 30초. 초과 시 인쇄종합 파싱 쪽을 프로파일링.

### 주의점
- `read_only=True` 모드에서는 `ws.cell(row, col)` 랜덤 액세스가 O(n). 반드시 `iter_rows()` 사용.
- 셀 값이 `MergedCell`인 경우 `values_only=True`를 쓰면 None으로 나옴 → 헤더 병합 주의.
- `wb.close()` 를 `try/finally`로 보장해서 파일 핸들 누수 방지.

---

## 8. Test Data Examples

### 8.1 수주관리 입력 샘플

| No | 담당 | 등록일 | 관리번호 | 매출처 | 품명 | 수주량 | 정미량 | 납기예정 | 공정 | 인쇄 | 코팅 | 실크,부분uv | 박 | 형압 | 톰슨 | 접착 | 생산일 | 재작업 | 비고 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 김병주 | 2026-04-02 | 260402-0005 | (주)비앤에프엘엔씨 | 리아_AO클렌저100G_단상자 | 10000 | 10500 | 2026-04-14 | 옵셋인쇄,톰슨,단면접착 | 2026-04-07 |  |  |  |  |  |  |  |  |  |
| 2 | 이정훈 | 2026-04-01 | 260401-0012 | 코스맥스 | 퍼퓸 세트 쇼핑백 | 5000 | 5200 | 2026-04-20 | UV인쇄 + 무광코팅 + 금박 + 톰슨 | 2026-04-05 | 2026-04-06 |  | 2026-04-07 |  | 2026-04-07 |  |  |  | 급건 |
| 3 | 박소연 | 2026-03-28 | 260328-0021 | 정샘물뷰티 | 립밤 단상자 | 8000 | 8300 | 2026-04-10 | 옵셋인쇄,부분uv,톰슨,삼면접착 | 2026-04-03 |  | 2026-04-04 |  |  | 2026-04-05 | 2026-04-06 | 2026-04-06 |  |  |

### 8.2 기대 Orders JSON

```json
[
  {
    "mgmt_no": "260402-0005",
    "manager": "김병주",
    "registered_at": "2026-04-02",
    "client": "(주)비앤에프엘엔씨",
    "product": "리아_AO클렌저100G_단상자",
    "qty": 10000,
    "net_qty": 10500,
    "deadline": "2026-04-14",
    "process_spec": "옵셋인쇄,톰슨,단면접착",
    "applicable_stages": ["인쇄", "톰슨", "접착"],
    "progress": {
      "인쇄": "2026-04-07",
      "코팅": null, "실크": null, "박": null,
      "형압": null, "톰슨": null, "접착": null
    },
    "completed_at": null,
    "rework": null,
    "note": "",
    "source_month": "2026.04",
    "badges": { "printing_today": false, "shipping_today": false }
  },
  {
    "mgmt_no": "260401-0012",
    "manager": "이정훈",
    "registered_at": "2026-04-01",
    "client": "코스맥스",
    "product": "퍼퓸 세트 쇼핑백",
    "qty": 5000,
    "net_qty": 5200,
    "deadline": "2026-04-20",
    "process_spec": "UV인쇄 + 무광코팅 + 금박 + 톰슨",
    "applicable_stages": ["인쇄", "코팅", "박", "톰슨"],
    "progress": {
      "인쇄": "2026-04-05",
      "코팅": "2026-04-06",
      "실크": null,
      "박": "2026-04-07",
      "형압": null,
      "톰슨": "2026-04-07",
      "접착": null
    },
    "completed_at": null,
    "rework": null,
    "note": "급건",
    "source_month": "2026.04",
    "badges": { "printing_today": false, "shipping_today": false }
  },
  {
    "mgmt_no": "260328-0021",
    "manager": "박소연",
    "registered_at": "2026-03-28",
    "client": "정샘물뷰티",
    "product": "립밤 단상자",
    "qty": 8000,
    "net_qty": 8300,
    "deadline": "2026-04-10",
    "process_spec": "옵셋인쇄,부분uv,톰슨,삼면접착",
    "applicable_stages": ["인쇄", "실크", "톰슨", "접착"],
    "progress": {
      "인쇄": "2026-04-03",
      "코팅": null,
      "실크": "2026-04-04",
      "박": null,
      "형압": null,
      "톰슨": "2026-04-05",
      "접착": "2026-04-06"
    },
    "completed_at": "2026-04-06",
    "rework": null,
    "note": "",
    "source_month": "2026.03",
    "badges": { "printing_today": false, "shipping_today": false }
  }
]
```

### 8.3 인쇄일정관리 입력 → printings_today

입력 행 (오늘 = 2026-04-08):

| 계획일 | Day | 영업자 | 수주번호 | 매출처 | 공정 | 제품명 | 공정상세 | 납품예정 | 수주량 | 벌수 | 정미량 | 비고 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 2026-04-08 | 화 | 김병주 | 260402-0005 | 코스맥스 | 옵셋인쇄 | 리아_AO클렌저100G_단상자 | 4도 양면 | 2026-04-14 | 10000 | 5 | 10500 | CTP 완료 |
| 2026-04-07 | 월 | 이정훈 | 260401-0012 | 코스맥스 | 옵셋인쇄 | 퍼퓸 세트 쇼핑백 | 5도 단면 | 2026-04-20 | 5000 | 3 | 5200 |  |

기대 출력:

```json
[
  {
    "plan_date": "2026-04-08",
    "day": "화",
    "manager": "김병주",
    "mgmt_no": "260402-0005",
    "client": "코스맥스",
    "process": "옵셋인쇄",
    "product": "리아_AO클렌저100G_단상자",
    "process_detail": "4도 양면",
    "deadline": "2026-04-14",
    "qty": 10000,
    "sets": 5,
    "net_qty": 10500,
    "note": "CTP 완료"
  }
]
```

→ `merge_badges` 이후 `260402-0005` order의 `badges.printing_today = true`.

### 8.4 출고리스트 입력 → shipments_today

시트명 `0408`, row 2 = `2026-04-08`:

| 업체명 | 제품명 | 출고수량 | 출고처 | 비고 |
|---|---|---|---|---|
| 알래스카애드 | 오로라 PN 프로 | 2300 | 본사창고 | 톰슨대기 |
| (주)비앤에프엘엔씨 | 리아_AO클렌저100G_단상자 | 10000 | 코스맥스 오산 | 접착대기 |

기대 출력:

```json
[
  {
    "sheet_date": "2026-04-08",
    "client": "알래스카애드",
    "product": "오로라 PN 프로",
    "qty": 2300,
    "destination": "본사창고",
    "status": "톰슨대기"
  },
  {
    "sheet_date": "2026-04-08",
    "client": "(주)비앤에프엘엔씨",
    "product": "리아_AO클렌저100G_단상자",
    "qty": 10000,
    "destination": "코스맥스 오산",
    "status": "접착대기"
  }
]
```

→ `merge_badges` 이후 `260402-0005` order의 `badges.shipping_today = true`
(품명 `리아_AO클렌저100G_단상자` 정규화 후 정확 일치).

### 8.5 compute_summary 기대값 (위 3건 기준)

```json
{
  "in_progress": 2,
  "printing_today": 1,
  "shipping_today": 2,
  "completed_7d": 1,
  "recent_7d_chart": {
    "2026-04-02": 0,
    "2026-04-03": 0,
    "2026-04-04": 0,
    "2026-04-05": 0,
    "2026-04-06": 1,
    "2026-04-07": 0,
    "2026-04-08": 0
  },
  "clients_in_progress": {
    "(주)비앤에프엘엔씨": 1,
    "코스맥스": 1
  }
}
```

---

## 9. Validation Checklist (구현 완료 후)

- [ ] 수주관리 원본 엑셀에서 임의 10건 골라 `mgmt_no`/`progress`/`completed_at`이 JSON과 100% 일치
- [ ] 인쇄일정관리에서 오늘 날짜 행 수가 `summary.printing_today`와 일치
- [ ] 출고리스트 최신 시트 행 수가 `summary.shipping_today`와 일치
- [ ] `orders[].badges.printing_today=true`인 건이 모두 `printings_today`에 존재하는지
- [ ] `progress` 키는 항상 7개
- [ ] 원본 엑셀 파일 mtime이 변환 전후 동일 (원본 수정 금지 검증)
- [ ] 실행 시간 < 30초
- [ ] 변환 중 예외 발생시키는 테스트 → `web/data.json` 변하지 않음 확인

---

## 10. Open Questions for CTO

1. **출고리스트 파일명 월 관리**: `출고품목리스트 (4월).xlsx` 처럼 월별 파일로 갈리는데, 5월이 되면 파일명이 바뀐다. glob으로 `출고품목리스트 (*월).xlsx` 매칭 후 최신 mtime을 고르는 방식으로 진행해도 되는가?
2. **`updated_at` 정의**: 스키마상 "가장 최근 데이터 기준일"인데, 후보가 (a) `max(order.completed_at)`, (b) 오늘 날짜, (c) 원본 엑셀 mtime 중 어느 것이 맞는지? 기본은 (b) 오늘 날짜로 두되 확정 필요.
3. **"박" 키워드 오탐지**: "박스"가 공정 텍스트에 등장하는 실제 사례가 있는지? 없다면 단순 부분매칭으로 충분.
4. **인쇄일정관리 '공정' 컬럼 중복**: 헤더에 "공정"이 2번(열 7, 19) 나온다고 명시됐는데, 실제로 둘의 의미가 다른지? 현재 설계는 첫 번째만 사용.
5. **월별 수주관리 시트 중복 관리번호**: 같은 `mgmt_no`가 여러 월 시트에 존재할 가능성이 있는지? (예: 3월 발주 → 4월 이월) 현재는 "마지막 월이 승리"로 설계.
6. **fuzzy 매칭 신뢰도 기준**: 품명 정규화 후 정확 일치만으로 충분한가? Levenshtein 등 거리 기반이 필요한가? (현재 설계는 정규화 후 완전일치만 채택 — 오탐지 위험 최소화.)

---

## 11. Out of Scope (이 문서)

- `web/index.html` 렌더링 로직 (별도 문서)
- 검색 인덱스 (클라이언트 측 문자열 매칭으로 충분, 별도 설계 불요)
- 차트 라이브러리 선택 (UI 설계 문서 `ui-design.md` 참조)
- Vercel 배포 설정 (`vercel.json`)
- 구글 드라이브 자동 동기화 (v2)
