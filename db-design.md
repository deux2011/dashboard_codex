# Code Mate 대시보드 - Elasticsearch DB 설계

## 1. 개요

| 항목 | 내용 |
|------|------|
| **DB** | Elasticsearch 8.x |
| **선정 사유** | 스키마 유연성 (dynamic mapping), 시계열 집계, 백분위 내장, 다차원 필터링 |
| **클라이언트** | @elastic/elasticsearch (Node.js) |
| **데이터 수집** | 주별 배치 (예: 2026.01, 2026.02, ...) |
| **설계 원칙** | **심플한 구조 + 유연한 확장**. 인덱스 수 최소화, Summary 차원 최소화 |

---

## 2. 아키텍처: 주별 배치 → Raw → Summary

### 2.1 데이터 흐름

```
[주별 데이터 수집]                    2026.01, 2026.02, ...
     │                               (개별 레코드에 timestamp 포함)
     │  배치 적재 (Bulk Index)
     ▼
[Raw 인덱스]                         [Summary 인덱스]
codemate-raw-2026.01       ──┐
codemate-raw-2026.02       ──┼─→ ES Transform (배치 후 실행)
codemate-raw-2026.13       ──┘      ├── codemate-summary-hourly       (서비스 차원)
     │                                ├── codemate-summary-daily         (서비스 차원)
     │                                └── codemate-summary-user-weekly   (사용자 차원)
     │
[프론트엔드] → [API] → Summary  ← 대시보드 일반 조회 (빠름)
                  └──→ Raw      ← 백분위, 드릴다운, User Analytics
```

### 2.2 배치 파이프라인

```
① 주별 데이터 파일 수신 (예: 2026.13)
     │
② 파일 파싱 → ES Bulk API로 codemate-raw-2026.13에 적재
     │
③ 적재 완료 → Transform 자동 감지 → Summary 갱신
     │
④ 대시보드 조회 가능
```

| 단계 | 설명 | 빈도 |
|------|------|------|
| ① 데이터 수신 | 주별 배치 파일 도착 | 주 1회 |
| ② Raw 적재 | Bulk Index로 주별 인덱스에 적재 | 주 1회 |
| ③ Summary 갱신 | Transform이 새 Raw 데이터 감지 후 자동 집계 | 자동 (Transform sync) |
| ④ 조회 | 대시보드에서 Summary/Raw 조회 | 상시 |

> **핵심**: 주별 배치로 Raw에 적재하면, 각 레코드의 `timestamp`를 기반으로 Transform이 시간별/일별 Summary를 자동 생성합니다. 주 단위 데이터에서 시/일 단위로 세분화되는 구조입니다.

### 2.3 조회 라우팅 전략

| 대시보드 요청 | 조회 대상 | 사유 |
|---|---|---|
| Overview KPI / 추이 차트 | **Summary** | 사전 집계, 즉시 응답 |
| 서비스별 상세 지표 | **Summary (hourly/daily)** | 서비스/모델 차원으로 사전 집계됨 |
| 인별 사용량 목록/랭킹 | **Summary (user-weekly)** | 사용자별 사전 집계 |
| 부서별/직군별/개발자유형별 집계 | **Summary (user-weekly)** | 사용자 차원 terms 집계 |
| P50/P95/P99 백분위 | **Raw** | 개별 값 필요, 사전 집계 불가 |
| 특정 사용자 상세 로그 드릴다운 | **Raw** | 개별 건 조회 |
| GPU/시스템 현황 | 일반 인덱스 | 소량 데이터 |

### 2.4 세분화 구조

주별 Raw 데이터 내 개별 레코드의 `timestamp`를 기준으로 Summary를 생성합니다.

```
[주별 Raw 데이터]                    [Summary 세분화]
2026.13 (3/24~3/30)
  ├─ 레코드: 2026-03-24T09:15:00Z ──→ summary-hourly: 2026-03-24T09:00
  ├─ 레코드: 2026-03-24T14:32:00Z ──→ summary-hourly: 2026-03-24T14:00
  ├─ 레코드: 2026-03-25T10:05:00Z ──→ summary-hourly: 2026-03-25T10:00
  └─ ...                              summary-daily:  2026-03-24, 2026-03-25, ...
```

| 데이터 수준 | granularity | 출처 | 용도 |
|---|---|---|---|
| 주별 배치 | `week` | Raw 인덱스 단위 | 데이터 관리/적재 단위 |
| 일별 집계 | `day` | **summary-daily** (Transform) | 일별 추이, KPI |
| 시간별 집계 | `hour` | **summary-hourly** (Transform) | 시간대별 상세 추이 |
| 개별 레코드 | - | **Raw** | 백분위, 드릴다운 |

---

## 3. 인덱스 목록

| 인덱스 | 용도 | 유형 |
|--------|------|------|
| `codemate-raw-YYYY.NN` | **모든 이벤트 원본** (요청 + 코드 적용) | Raw, 주별 |
| `codemate-summary-hourly` | 시간별 사전 집계 (서비스 차원) | Summary |
| `codemate-summary-daily` | 일별 사전 집계 (서비스 차원) | Summary |
| `codemate-summary-user-weekly` | **주별 인별 사용량 집계** | Summary |
| `codemate-users` | 사용자 프로필 | 일반 |
| `codemate-user-population` | 사용자 모수 스냅샷 | 일반 |
| `codemate-gpu-status` | GPU 현황 | 일반 |
| `codemate-model-service-mapping` | 모델-서비스 연결 | 일반 |

> **Raw 인덱스 네이밍**: `codemate-raw-2026.01`, `codemate-raw-2026.13` 등 주 단위로 생성. 데이터 적재/삭제가 주 단위로 관리됨.

---

## 4. 인덱스 상세 매핑

### 4.1 `codemate-raw-YYYY.NN` (주별 원본)

주별 배치 데이터를 인덱스 단위로 저장합니다. 개별 레코드에 `timestamp`가 포함되어 시/일 세분화가 가능합니다.

```json
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "timestamp":        { "type": "date" },
      "event_type":       { "type": "keyword" },
      "service":          { "type": "keyword" },
      "tool":             { "type": "keyword" },
      "model":            { "type": "keyword" },
      "user_id":          { "type": "keyword" },
      "department":       { "type": "keyword" },
      "job_group":        { "type": "keyword" },
      "developer_type":   { "type": "keyword" },
      "status":           { "type": "keyword" },
      "response_time_ms": { "type": "integer" },
      "input_tokens":     { "type": "integer" },
      "output_tokens":    { "type": "integer" },
      "cache_hit_tokens": { "type": "integer" },
      "applied_lines":    { "type": "integer" },
      "language":         { "type": "keyword" }
    }
  },
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1,
    "index.lifecycle.name": "codemate-raw-ilm"
  }
}
```

**필드 설명:**

| 필드 | 설명 | `chat` 이벤트 | `code_completion` 이벤트 |
|------|------|:-:|:-:|
| `event_type` | 이벤트 구분 | `"chat"` | `"code_completion"` |
| `service` / `tool` / `model` | 서비스 차원 | O | O |
| `user_id` / `department` / `job_group` / `developer_type` | 사용자 차원 (비정규화) | O | O |
| `developer_type` | 개발자 유형: `pro-impl`, `pro-non-impl`, `citizen` | O | O |
| `status` | 요청 결과 | O | - |
| `response_time_ms` | 응답 시간 | O | - |
| `input_tokens` / `output_tokens` / `cache_hit_tokens` | 토큰 수 | O | - |
| `applied_lines` | 적용된 코드 라인 수 | - | O |
| `language` | 프로그래밍 언어 | - | O |

> **`dynamic: true`**: 새 필드 추가 시 마이그레이션 불필요. 코드에서 바로 인덱싱하면 자동 매핑.
>
> **플랫 구조**: 중첩 객체(tokens.input) 대신 `input_tokens`처럼 플랫하게 유지. 쿼리/집계가 간결해짐.
>
> **이벤트 통합**: 한 인덱스에서 `event_type`으로 구분. 인덱스 수를 줄이고, 필요 시 새로운 event_type 추가만으로 확장 가능.

**Document 예시:**

```json
// chat 이벤트
{
  "timestamp": "2026-03-28T14:32:10Z",
  "event_type": "chat",
  "service": "code-mate",
  "tool": "continue",
  "model": "glm-4.7",
  "user_id": "user-001",
  "department": "AI플랫폼팀",
  "job_group": "S직군",
  "developer_type": "pro-impl",
  "status": "success",
  "response_time_ms": 1250,
  "input_tokens": 512,
  "output_tokens": 1024,
  "cache_hit_tokens": 128
}

// code_completion 이벤트
{
  "timestamp": "2026-03-28T14:33:05Z",
  "event_type": "code_completion",
  "service": "code-mate",
  "tool": "continue",
  "user_id": "user-001",
  "department": "AI플랫폼팀",
  "job_group": "S직군",
  "developer_type": "pro-impl",
  "applied_lines": 45,
  "language": "python"
}
```

---

### 4.2 `codemate-summary-hourly` / `codemate-summary-daily` (Summary)

**Summary의 group_by는 서비스 차원만 포함** (service, tool, model). 사용자 차원은 포함하지 않습니다.

```json
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "timestamp":          { "type": "date" },
      "service":            { "type": "keyword" },
      "tool":               { "type": "keyword" },
      "model":              { "type": "keyword" },

      "chat_count":            { "type": "long" },
      "code_completion_count": { "type": "long" },
      "error_count":           { "type": "long" },
      "response_time_sum":     { "type": "long" },
      "input_tokens_sum":      { "type": "long" },
      "output_tokens_sum":     { "type": "long" },
      "cache_hit_tokens_sum":  { "type": "long" },
      "applied_lines_sum":     { "type": "long" },
      "unique_users":          { "type": "integer" }
    }
  },
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}
```

**Document 예시 (hourly):**

```json
{
  "timestamp": "2026-03-28T14:00:00Z",
  "service": "code-mate",
  "tool": "continue",
  "model": "glm-4.7",
  "chat_count": 65,
  "code_completion_count": 22,
  "error_count": 3,
  "response_time_sum": 116623,
  "input_tokens_sum": 44032,
  "output_tokens_sum": 89100,
  "cache_hit_tokens_sum": 11264,
  "applied_lines_sum": 320,
  "unique_users": 23
}
```

**문서 수 비교 (1시간, 서비스 3 x 도구 4 x 모델 2 기준):**

| 방식 | group_by | 시간당 문서 수 |
|------|----------|---------------|
| 이전 (7차원) | + 부서 10 x 직군 3 x 개발자여부 2 | ~1,440 |
| **이후 (4차원)** | service x tool x model | **~24** |

---

### 4.3 `codemate-summary-user-weekly` (인별 주간 사용량)

주별 배치 단위로 사용자별 사용량을 집계합니다. 인별 사용량 조회, 랭킹, 부서/직군/개발자유형별 분석에 활용합니다.

```json
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "week":                 { "type": "keyword" },
      "user_id":              { "type": "keyword" },
      "department":           { "type": "keyword" },
      "job_group":            { "type": "keyword" },
      "developer_type":       { "type": "keyword" },

      "chat_count":            { "type": "long" },
      "code_completion_count": { "type": "long" },
      "error_count":           { "type": "long" },
      "response_time_sum":     { "type": "long" },
      "input_tokens_sum":      { "type": "long" },
      "output_tokens_sum":     { "type": "long" },
      "cache_hit_tokens_sum":  { "type": "long" },
      "applied_lines_sum":     { "type": "long" }
    }
  },
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 }
}
```

**필드 설명:**

| 필드 | 설명 |
|------|------|
| `week` | 주차 (예: `2026.13`) |
| `user_id` | 사용자 ID (group_by 키) |
| `department` / `job_group` / `developer_type` | 사용자 속성 (비정규화) |
| `chat_count` / `code_completion_count` | 이벤트 유형별 건수 |
| `error_count` ~ `applied_lines_sum` | 해당 주 사용량 집계 |

> 사용자가 어떤 서비스/도구/모델을 사용했는지는 Raw에서 terms 집계로 조회합니다.

**Document 예시:**

```json
{
  "week": "2026.13",
  "user_id": "user-001",
  "department": "AI플랫폼팀",
  "job_group": "S직군",
  "developer_type": "pro-impl",
  "chat_count": 245,
  "code_completion_count": 97,
  "error_count": 5,
  "response_time_sum": 428500,
  "input_tokens_sum": 175000,
  "output_tokens_sum": 350000,
  "cache_hit_tokens_sum": 42000,
  "applied_lines_sum": 1280
}
```

**문서 수**: 사용자 수/주 (예: 활성 사용자 3,000명 → 주당 3,000 docs)

---

### 4.4 `codemate-users` (사용자 프로필)

```json
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "user_id":       { "type": "keyword" },
      "name":          { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "department":    { "type": "keyword" },
      "job_group":     { "type": "keyword" },
      "developer_type": { "type": "keyword" },
      "created_at":    { "type": "date" },
      "updated_at":    { "type": "date" }
    }
  },
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 }
}
```

### 4.5 `codemate-user-population` (사용자 모수 스냅샷)

```json
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "snapshot_date":              { "type": "date", "format": "yyyy-MM-dd" },
      "total_count":                { "type": "integer" },
      "pro_developer_count":        { "type": "integer" },
      "pro_impl_developer_count":   { "type": "integer" },
      "pro_non_impl_developer_count": { "type": "integer" },
      "citizen_developer_count":    { "type": "integer" }
    }
  },
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 }
}
```

### 4.6 `codemate-gpu-status` (GPU 현황)

```json
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "gpu_id":      { "type": "keyword" },
      "model":       { "type": "keyword" },
      "status":      { "type": "keyword" },
      "utilization": { "type": "float" },
      "updated_at":  { "type": "date" }
    }
  },
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 }
}
```

### 4.7 `codemate-model-service-mapping` (모델-서비스 연결)

```json
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "model":      { "type": "keyword" },
      "service":    { "type": "keyword" },
      "tools":      { "type": "keyword" },
      "is_active":  { "type": "boolean" },
      "updated_at": { "type": "date" }
    }
  },
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 }
}
```

---

## 5. ES Transform (주별 배치 → 시/일 세분화)

주별 배치 적재 후 Transform이 자동으로 시간별/일별 Summary를 생성합니다.
Transform은 `continuous` 모드로 동작하여, 새 Raw 인덱스가 추가되면 자동 감지합니다.

### 5.1 Hourly Transform

```json
PUT _transform/codemate-summary-hourly
{
  "source": { "index": ["codemate-raw-*"] },
  "dest":   { "index": "codemate-summary-hourly" },
  "pivot": {
    "group_by": {
      "timestamp": { "date_histogram": { "field": "timestamp", "calendar_interval": "1h" } },
      "service":   { "terms": { "field": "service" } },
      "tool":      { "terms": { "field": "tool", "missing_bucket": true } },
      "model":     { "terms": { "field": "model", "missing_bucket": true } }
    },
    "aggregations": {
      "chat_count":             { "filter": { "term": { "event_type": "chat" } } },
      "code_completion_count": { "filter": { "term": { "event_type": "code_completion" } } },
      "error_count":          { "filter": { "terms": { "status": ["error", "timeout"] } } },
      "response_time_sum":    { "sum": { "field": "response_time_ms" } },
      "input_tokens_sum":     { "sum": { "field": "input_tokens" } },
      "output_tokens_sum":    { "sum": { "field": "output_tokens" } },
      "cache_hit_tokens_sum": { "sum": { "field": "cache_hit_tokens" } },
      "applied_lines_sum":    { "sum": { "field": "applied_lines" } },
      "unique_users":         { "cardinality": { "field": "user_id" } }
    }
  },
  "frequency": "15m",
  "sync": { "time": { "field": "timestamp", "delay": "60s" } }
}
```

### 5.2 Daily Transform

```json
PUT _transform/codemate-summary-daily
{
  "source": { "index": ["codemate-raw-*"] },
  "dest":   { "index": "codemate-summary-daily" },
  "pivot": {
    "group_by": {
      "timestamp": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d" } },
      "service":   { "terms": { "field": "service" } },
      "tool":      { "terms": { "field": "tool", "missing_bucket": true } },
      "model":     { "terms": { "field": "model", "missing_bucket": true } }
    },
    "aggregations": {
      "chat_count":             { "filter": { "term": { "event_type": "chat" } } },
      "code_completion_count": { "filter": { "term": { "event_type": "code_completion" } } },
      "error_count":          { "filter": { "terms": { "status": ["error", "timeout"] } } },
      "response_time_sum":    { "sum": { "field": "response_time_ms" } },
      "input_tokens_sum":     { "sum": { "field": "input_tokens" } },
      "output_tokens_sum":    { "sum": { "field": "output_tokens" } },
      "cache_hit_tokens_sum": { "sum": { "field": "cache_hit_tokens" } },
      "applied_lines_sum":    { "sum": { "field": "applied_lines" } },
      "unique_users":         { "cardinality": { "field": "user_id" } }
    }
  },
  "frequency": "30m",
  "sync": { "time": { "field": "timestamp", "delay": "60s" } }
}
```

### 5.3 User Weekly Transform

```json
PUT _transform/codemate-summary-user-weekly
{
  "source": { "index": ["codemate-raw-*"] },
  "dest":   { "index": "codemate-summary-user-weekly" },
  "pivot": {
    "group_by": {
      "week":           { "date_histogram": { "field": "timestamp", "calendar_interval": "1w" } },
      "user_id":        { "terms": { "field": "user_id" } },
      "department":     { "terms": { "field": "department" } },
      "job_group":      { "terms": { "field": "job_group" } },
      "developer_type": { "terms": { "field": "developer_type" } }
    },
    "aggregations": {
      "chat_count":             { "filter": { "term": { "event_type": "chat" } } },
      "code_completion_count": { "filter": { "term": { "event_type": "code_completion" } } },
      "error_count":          { "filter": { "terms": { "status": ["error", "timeout"] } } },
      "response_time_sum":    { "sum": { "field": "response_time_ms" } },
      "input_tokens_sum":     { "sum": { "field": "input_tokens" } },
      "output_tokens_sum":    { "sum": { "field": "output_tokens" } },
      "cache_hit_tokens_sum": { "sum": { "field": "cache_hit_tokens" } },
      "applied_lines_sum":    { "sum": { "field": "applied_lines" } }
    }
  },
  "frequency": "30m",
  "sync": { "time": { "field": "timestamp", "delay": "60s" } }
}
```

### 5.4 Transform 요약

| Transform | 감지 주기 | 세분화 결과 | 문서 수/주 (추정) |
|-----------|-----------|-------------|-------------------|
| Hourly | 15분 | 주 1건 배치 → **168개** 시간별 Summary | ~168 x 24 조합 ≈ 4,032 |
| Daily | 30분 | 주 1건 배치 → **7개** 일별 Summary | ~7 x 24 조합 ≈ 168 |
| **User Weekly** | 30분 | 주 1건 배치 → **인별 1건** | ~활성 사용자 수 (수천) |

> **동작 방식**: Transform은 `continuous` 모드로 상시 실행. 주별 배치 적재 시 새 Raw 데이터를 자동 감지하여 Summary를 갱신합니다. 배치 적재가 없는 기간에는 idle 상태로 리소스 소모 없음.

---

## 6. 주요 쿼리 예시

### 6.1 Overview KPI — Summary 조회

```json
// codemate-summary-daily 에서 최근 30일 KPI
{
  "size": 0,
  "query": { "range": { "timestamp": { "gte": "now-30d/d" } } },
  "aggs": {
    "total_chats":       { "sum": { "field": "chat_count" } },
    "total_completions": { "sum": { "field": "code_completion_count" } },
    "total_errors":      { "sum": { "field": "error_count" } },
    "total_tokens":      { "sum": { "field": "input_tokens_sum" } },
    "total_tokens_out":  { "sum": { "field": "output_tokens_sum" } },
    "total_code_lines":  { "sum": { "field": "applied_lines_sum" } },
    "rt_sum":            { "sum": { "field": "response_time_sum" } }
  }
}
```

### 6.2 일별 요청 추이 — Summary 조회

```json
// codemate-summary-daily
{
  "size": 0,
  "query": {
    "bool": { "filter": [
      { "range": { "timestamp": { "gte": "now-30d/d" } } },
      { "term": { "service": "code-mate" } }
    ] }
  },
  "aggs": {
    "trend": {
      "date_histogram": { "field": "timestamp", "calendar_interval": "day" },
      "aggs": {
        "chats":       { "sum": { "field": "chat_count" } },
        "completions": { "sum": { "field": "code_completion_count" } },
        "tokens":      { "sum": { "field": "output_tokens_sum" } }
      }
    }
  }
}
```

### 6.3 응답 시간 P50/P95/P99 — Raw 조회

```json
// codemate-raw-* (백분위는 Raw에서만 정확)
{
  "size": 0,
  "query": {
    "bool": { "filter": [
      { "range": { "timestamp": { "gte": "now-7d/d" } } },
      { "term": { "event_type": "chat" } }
    ] }
  },
  "aggs": {
    "percentiles": {
      "percentiles": { "field": "response_time_ms", "percents": [50, 95, 99] }
    }
  }
}
```

### 6.4 인별 사용량 랭킹 (Top 20) — User Summary 조회

```json
// codemate-summary-user-weekly
{
  "size": 20,
  "query": { "term": { "week": "2026.13" } },
  "sort": [{ "chat_count": "desc" }]
}
```

### 6.5 부서별 인원수 + 평균 사용량 — User Summary 조회

```json
// codemate-summary-user-weekly
{
  "size": 0,
  "query": { "term": { "week": "2026.13" } },
  "aggs": {
    "by_dept": {
      "terms": { "field": "department", "size": 50 },
      "aggs": {
        "user_count":     { "cardinality": { "field": "user_id" } },
        "total_chats": { "sum": { "field": "chat_count" } },
        "avg_chats":   { "avg": { "field": "chat_count" } }
      }
    }
  }
}
```

### 6.6 개발자 유형별 비교 — User Summary 조회

```json
// codemate-summary-user-weekly
{
  "size": 0,
  "query": { "term": { "week": "2026.13" } },
  "aggs": {
    "by_type": {
      "terms": { "field": "developer_type" },
      "aggs": {
        "users":          { "cardinality": { "field": "user_id" } },
        "total_chats": { "sum": { "field": "chat_count" } },
        "total_tokens":   { "sum": { "field": "output_tokens_sum" } },
        "total_lines":    { "sum": { "field": "applied_lines_sum" } }
      }
    }
  }
}
```

**대시보드 필터 → ES 쿼리 매핑:**

| 필터 선택 | ES 쿼리 조건 |
|---|---|
| 전체 | 필터 없음 |
| Pro 개발자 | `{ "terms": { "developer_type": ["pro-impl", "pro-non-impl"] } }` |
| Citizen 개발자 | `{ "term": { "developer_type": "citizen" } }` |
| 구현 개발자 | `{ "term": { "developer_type": "pro-impl" } }` |

---

## 7. API ↔ ES 매핑

| API | 인덱스 | 유형 |
|-----|--------|------|
| `GET /api/metrics/overview` | `summary-daily` | Summary |
| `GET /api/metrics/requests?granularity=` | `summary-{hourly\|daily}` | Summary |
| `GET /api/metrics/tokens?granularity=` | `summary-{hourly\|daily}` | Summary |
| `GET /api/metrics/code-lines?granularity=` | `summary-{hourly\|daily}` | Summary |
| `GET /api/metrics/response-time` | `raw-*` | **Raw** |
| `GET /api/users/analytics` | `summary-user-weekly` | **User Summary** |
| `GET /api/users/ranking` | `summary-user-weekly` | **User Summary** |
| `GET /api/users/population` | `user-population` | 일반 |
| `GET /api/users` | `users` | 일반 |
| `GET /api/users/:id/logs` | `raw-*` | **Raw** (드릴다운) |
| `GET /api/system/gpus` | `gpu-status` | 일반 |
| `GET /api/system/model-service-mapping` | `model-service-mapping` | 일반 |

**Granularity 라우팅:**

| granularity | 인덱스 | 비고 |
|-------------|--------|------|
| `hour` | `summary-hourly` | |
| `day`, `week` | `summary-daily` | week는 daily를 7일 묶어 계산 |
| `month`, `year` | `summary-daily` | daily를 합산하여 계산 |

---

## 8. ILM (Index Lifecycle Management)

주별 Raw 인덱스에 적용합니다. 주 단위로 인덱스가 생성되므로 rollover 대신 age 기반으로 관리합니다.

```json
{
  "policy": {
    "phases": {
      "warm":   { "min_age": "4w",   "actions": { "shrink": { "number_of_shards": 1 }, "forcemerge": { "max_num_segments": 1 } } },
      "cold":   { "min_age": "12w",  "actions": { "freeze": {} } },
      "delete": { "min_age": "52w",  "actions": { "delete": {} } }
    }
  }
}
```

| Phase | 기간 | 설명 |
|-------|------|------|
| **Hot** | 0~4주 | 최근 4주 Raw. 활성 읽기 |
| **Warm** | 4~12주 | 샤드 축소, 세그먼트 병합 |
| **Cold** | 12~52주 | 동결. 드문 조회만 |
| **Delete** | 52주 이후 | 자동 삭제 |

> Raw 인덱스에만 ILM 적용. Summary 인덱스는 소량이므로 별도 관리 불필요.

---

## 9. Index Template

주별 Raw 인덱스 자동 생성을 위한 템플릿입니다.

```json
{
  "index_patterns": ["codemate-raw-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "codemate-raw-ilm",
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  }
}
```

> 주별 인덱스는 월별 대비 데이터량이 적으므로 샤드 수를 2로 줄임.

---

## 10. 스키마 변경 대응

| 시나리오 | 대응 | 영향 |
|----------|------|------|
| 새 필드 추가 | `dynamic: true`로 자동 매핑 | 없음 |
| 새 event_type 추가 | Raw에 바로 인덱싱 | 없음 |
| 새 서비스/모델 추가 | keyword 값만 추가 | 없음 |
| 필드 타입 변경 | 새 주별 인덱스에서 적용 | 해당 주부터 |
| Summary에 집계 필드 추가 | Transform 업데이트 → 재시작 | Summary만 재생성 (Raw 보존) |
| Summary 재구축 | Transform 중지 → Summary 삭제 → 재시작 | Raw 보존 |

---

## 11. 운영

| 항목 | 내용 |
|------|------|
| **클러스터** | 개발: 단일 노드 / 운영: 3노드 |
| **보안** | X-Pack Security, API 서버에서만 ES 접근 |
| **모니터링** | Kibana로 Cluster Health, Transform 상태 확인 |
| **백업** | Snapshot & Restore 일 1회 |
