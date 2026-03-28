# Code Mate 서비스 대시보드 구현 계획

## 1. 개요

DS부문의 AI Coding 도구 "Code Mate" 전반에 대한 서비스 사용 현황을 모니터링하는 대시보드를 구축합니다.

---

## 2. 요구사항 정리

### 2.1 사용자 정보 (User Profile)
| 항목 | 설명 |
|------|------|
| 소속 | 사용자가 속한 조직/팀 |
| 직군 | S직군 등 직군 분류 |
| 개발자 유형 | Pro 개발자 (구현/비구현) 또는 Citizen 개발자 |

### 2.2 개발자 유형 분류

```
전체
├── Pro 개발자
│   ├── 구현 개발자 (pro-impl)
│   └── 비구현 개발자 (pro-non-impl)
└── Citizen 개발자 (citizen)
```

| 대시보드 필터 | 대상 |
|---|---|
| 전체 | 모든 사용자 |
| Pro 개발자 | 구현 + 비구현 개발자 |
| Citizen 개발자 | Citizen 개발자 |
| 구현 개발자 | Pro 개발자 중 구현 개발자만 |

### 2.3 사용자 모수 (User Population)
| 항목 | 설명 |
|------|------|
| DS부문 인원 수 | DS부문 전체 인원 |
| Pro 개발자 인원 수 | Pro 개발자 전체 (구현 + 비구현) |
| 구현 개발자 인원 수 | Pro 개발자 중 구현 개발자 |
| 비구현 개발자 인원 수 | Pro 개발자 중 비구현 개발자 |
| Citizen 개발자 인원 수 | Citizen 개발자 인원 |

### 2.4 서비스 분류
| 서비스 | 하위 도구 |
|--------|-----------|
| **Code Mate** | Continue, Roo Code, OpenCode |
| **Code Pearl** | - |
| **Code Search** | - |

### 2.5 지원 모델
- GLM 4.7
- Qwen 3.5

### 2.6 데이터 수집 주기 및 세분화

| 항목 | 내용 |
|------|------|
| **수집 주기** | 주별 배치 (예: 2026.01, 2026.13) |
| **Raw 데이터** | 주별 배치 내 개별 레코드 (timestamp 포함) |
| **세분화** | Raw 레코드의 timestamp 기반으로 시간별/일별 Summary 자동 생성 |

```
주별 배치 데이터 (2026.13)
  │  개별 레코드에 timestamp 포함
  │
  ├──→ 시간별 집계 (summary-hourly): 168개/주
  ├──→ 일별 집계 (summary-daily): 7개/주
  └──→ 주별/월별: 일별 집계를 합산하여 계산
```

| 대시보드 granularity | 데이터 출처 |
|---|---|
| 시간(hour) | summary-hourly |
| 일(day) | summary-daily |
| 주(week) | summary-daily 합산 |
| 월(month) / 년(year) | summary-daily 합산 |

### 2.7 요청 관련 지표
| 지표 | 설명 |
|------|------|
| 요청 수 | 서비스별/모델별 총 요청 건수 |
| 요청별 응답 시간 | 평균/P50/P95/P99 응답 시간 |
| 요청별 Token | Input Token, Output Token, Cache Hit Token |

### 2.8 코드 관련 지표
| 지표 | 설명 |
|------|------|
| 코드 적용 라인 수 | AI가 생성한 코드 중 실제 적용된 라인 수 |

### 2.9 시스템 관련 지표
| 지표 | 설명 |
|------|------|
| GPU 전체 개수 | 클러스터 내 총 GPU 수 |
| GPU당 모델 탑재 현황 | 각 GPU에 배포된 모델 정보 |
| 모델과 서비스 연결 현황 | 어떤 모델이 어떤 서비스에 연결되어 있는지 |

---

## 3. 기술 스택

| 구분 | 선택 | 사유 |
|------|------|------|
| **Framework** | React 18 + TypeScript | 컴포넌트 기반, 타입 안전성 |
| **빌드 도구** | Vite | 빠른 HMR, 간단한 설정 |
| **UI 라이브러리** | Ant Design 5 | 대시보드에 최적화된 테이블/카드/레이아웃 |
| **차트 라이브러리** | Apache ECharts (echarts-for-react) | 풍부한 차트 유형, 한글 지원 |
| **상태관리** | Zustand | 경량, 간결한 API |
| **API 통신** | Axios + React Query | 캐싱, 자동 재요청, 로딩 상태 관리 |
| **라우팅** | React Router v6 | SPA 라우팅 |
| **스타일링** | CSS Modules + Ant Design Token | 일관된 디자인 시스템 |

---

## 4. 디렉토리 구조

```
dashboard/
├── public/
├── src/
│   ├── api/                    # API 인터페이스
│   │   ├── client.ts           # Axios 인스턴스
│   │   ├── userApi.ts          # 사용자 관련 API
│   │   ├── metricsApi.ts       # 요청/코드 지표 API
│   │   └── systemApi.ts        # 시스템 지표 API
│   │
│   ├── components/             # 공통 컴포넌트
│   │   ├── Layout/
│   │   │   ├── AppLayout.tsx   # 전체 레이아웃 (Sider + Header + Content)
│   │   │   └── Sidebar.tsx     # 사이드바 네비게이션
│   │   ├── Charts/
│   │   │   ├── RequestChart.tsx       # 요청 수 추이 차트
│   │   │   ├── ResponseTimeChart.tsx  # 응답 시간 차트
│   │   │   ├── TokenUsageChart.tsx    # 토큰 사용량 차트
│   │   │   ├── CodeLineChart.tsx      # 코드 적용 라인 차트
│   │   │   └── GpuStatusChart.tsx     # GPU 현황 차트
│   │   ├── Cards/
│   │   │   ├── StatCard.tsx           # 통계 카드 (KPI)
│   │   │   └── ServiceCard.tsx        # 서비스별 요약 카드
│   │   └── Tables/
│   │       ├── UserTable.tsx          # 사용자 목록 테이블
│   │       ├── GpuTable.tsx           # GPU 모델 탑재 현황 테이블
│   │       └── ServiceModelTable.tsx  # 서비스-모델 연결 테이블
│   │
│   ├── pages/                  # 페이지 컴포넌트
│   │   ├── Overview/
│   │   │   └── OverviewPage.tsx       # 전체 요약 대시보드
│   │   ├── ServiceMetrics/
│   │   │   └── ServiceMetricsPage.tsx # 서비스별 상세 지표
│   │   ├── UserAnalytics/
│   │   │   └── UserAnalyticsPage.tsx  # 사용자 분석
│   │   └── SystemStatus/
│   │       └── SystemStatusPage.tsx   # 시스템 현황
│   │
│   ├── stores/                 # Zustand 상태 관리
│   │   ├── filterStore.ts      # 필터 상태 (기간, 서비스, 모델)
│   │   └── userStore.ts        # 사용자 정보 상태
│   │
│   ├── types/                  # TypeScript 타입 정의
│   │   ├── user.ts
│   │   ├── metrics.ts
│   │   └── system.ts
│   │
│   ├── utils/                  # 유틸리티
│   │   └── formatters.ts       # 숫자/날짜 포맷터
│   │
│   ├── App.tsx
│   ├── main.tsx
│   └── router.tsx
│
├── package.json
├── tsconfig.json
├── vite.config.ts
└── index.html
```

---

## 5. 페이지 구성

### 5.1 Overview (전체 요약 대시보드)
메인 화면. 한눈에 전체 서비스 현황을 파악할 수 있는 페이지.

```
┌─────────────────────────────────────────────────────────────┐
│  [필터 바] 기간 선택 | 시간 단위(연/월/주/일/시) | 서비스 | 모델 │
├──────────┬──────────┬──────────┬──────────┬─────────────────┤
│ KPI 카드 │ KPI 카드 │ KPI 카드 │ KPI 카드 │ KPI 카드        │
│ 총 요청수│ 평균응답 │ 총 Token │ 코드적용 │ 활성사용자      │
│          │ 시간     │ 사용량   │ 라인수   │                 │
├──────────┴──────────┴──────────┴──────────┴─────────────────┤
│  [요청 수 추이 차트]              │  [서비스별 요청 비율]    │
│  Line Chart (연/월/주/일/시 전환) │  Pie/Donut Chart        │
├───────────────────────────────────┼─────────────────────────┤
│  [모델별 Token 사용량]            │  [응답 시간 분포]       │
│  Stacked Bar Chart                │  Box Plot / Histogram   │
├───────────────────────────────────┴─────────────────────────┤
│  [사용자 모수 현황]                                         │
│  전체 | Pro 개발자 | 구현 개발자 | Citizen 개발자 (Progress) │
└─────────────────────────────────────────────────────────────┘
```

**KPI 카드 (5개)**:
- 총 요청 수 (전일 대비 증감률)
- 평균 응답 시간 (ms)
- 총 Token 사용량 (Input + Output)
- 코드 적용 라인 수
- 활성 사용자 수

### 5.2 Service Metrics (서비스별 상세 지표)
서비스별로 드릴다운하여 상세 지표를 확인하는 페이지.

```
┌─────────────────────────────────────────────────────────────┐
│  [탭] Code Mate | Code Pearl | Code Search                  │
├─────────────────────────────────────────────────────────────┤
│  [Code Mate 하위 탭] Continue | Roo Code | OpenCode | 전체  │
├──────────┬──────────┬──────────┬────────────────────────────┤
│ 요청 수  │ 응답시간 │ Token    │ 코드 적용 라인             │
│ (일별)   │ P50/P95  │ I/O/캐시 │ (일별 추이)               │
├──────────┴──────────┴──────────┴────────────────────────────┤
│  [요청 추이 상세]                                           │
│  Multi-line Chart (모델별 비교)                              │
├─────────────────────────────────────────────────────────────┤
│  [Token 상세 분석]                                          │
│  Stacked Area Chart: Input / Output / Cache Hit             │
├─────────────────────────────────────────────────────────────┤
│  [응답 시간 백분위]                                         │
│  P50 / P95 / P99 트렌드 라인                                │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 User Analytics (사용자 분석)
사용자 분포와 활용 현황을 분석하는 페이지.

```
┌─────────────────────────────────────────────────────────────┐
│  [필터] 전체 | Pro 개발자 | Citizen 개발자 | 구현 개발자     │
├─────────────────────────────────────────────────────────────┤
│  [사용자 모수 요약]                                         │
│  전체: N명 | Pro: N명(구현 N/비구현 N) | Citizen: N명       │
├─────────────────────────────────┬───────────────────────────┤
│  [개발자 유형별 사용 비율]      │  [소속별 사용 현황]       │
│  Pro(구현/비구현) vs Citizen    │  Horizontal Bar Chart     │
│  Stacked Bar / Donut Chart      │                           │
├─────────────────────────────────┼───────────────────────────┤
│  [개발자 유형별 서비스 이용]    │  [직군별 사용 현황]       │
│  Grouped Bar Chart              │  Bar Chart                │
├─────────────────────────────────┴───────────────────────────┤
│  [사용자 상세 테이블]                                       │
│  소속 | 직군 | 개발자유형 | 사용서비스 | 요청수 | ...       │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 System Status (시스템 현황)
GPU 및 모델 배포/연결 현황을 모니터링하는 페이지.

```
┌─────────────────────────────────────────────────────────────┐
│  [GPU 요약]  전체 GPU: N개 | 활성: N개 | 유휴: N개          │
├─────────────────────────────────────────────────────────────┤
│  [GPU당 모델 탑재 현황]                                     │
│  ┌─────────┬────────────┬────────┬───────┐                  │
│  │ GPU ID  │ 모델       │ 상태   │ 사용률│                  │
│  │ GPU-001 │ GLM 4.7    │ Active │ 78%   │                  │
│  │ GPU-002 │ Qwen 3.5   │ Active │ 65%   │                  │
│  │ ...     │ ...        │ ...    │ ...   │                  │
│  └─────────┴────────────┴────────┴───────┘                  │
├─────────────────────────────────────────────────────────────┤
│  [모델 ↔ 서비스 연결 현황]                                  │
│                                                             │
│  ┌──────────┐     ┌───────────┐     ┌──────────────┐       │
│  │ GLM 4.7  │────▶│ Code Mate │────▶│ Continue     │       │
│  │          │────▶│           │────▶│ Roo Code     │       │
│  │          │────▶│ Code Pearl│     │ OpenCode     │       │
│  ├──────────┤     ├───────────┤     └──────────────┘       │
│  │ Qwen 3.5 │────▶│ Code Mate │                            │
│  │          │────▶│ CodeSearch│                             │
│  └──────────┘     └───────────┘                             │
│  (Sankey Diagram 또는 연결 테이블)                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 데이터 모델 (TypeScript 타입)

### 6.1 사용자 관련

```typescript
// types/user.ts
type DeveloperType = 'pro-impl' | 'pro-non-impl' | 'citizen';

interface User {
  id: string;
  name: string;
  department: string;          // 소속
  jobGroup: string;            // 직군 (예: 'S직군')
  developerType: DeveloperType; // 개발자 유형
}

interface UserPopulation {
  totalCount: number;               // DS부문 전체 인원
  proDeveloperCount: number;        // Pro 개발자 (구현 + 비구현)
  proImplDeveloperCount: number;    // Pro 개발자 > 구현 개발자
  proNonImplDeveloperCount: number; // Pro 개발자 > 비구현 개발자
  citizenDeveloperCount: number;    // Citizen 개발자
}
```

### 6.2 요청 지표 관련

```typescript
// types/metrics.ts
type ServiceType = 'code-mate' | 'code-pearl' | 'code-search';
type CodeMateToolType = 'continue' | 'roo-code' | 'open-code';
type ModelType = 'glm-4.7' | 'qwen-3.5';

// 시계열 데이터 시간 단위 (Granularity)
type TimeGranularity = 'year' | 'month' | 'week' | 'day' | 'hour';

interface TimeSeriesQuery {
  from: string;               // ISO datetime (예: '2026-01-01T00:00:00Z')
  to: string;                 // ISO datetime
  granularity: TimeGranularity; // 집계 단위
}

// 시계열 데이터 포인트 (모든 시계열 차트에서 공통 사용)
interface TimeSeriesPoint {
  timestamp: string;          // granularity에 따라 달라짐
                              //   year:  '2026'
                              //   month: '2026-03'
                              //   week:  '2026-W13'
                              //   day:   '2026-03-28'
                              //   hour:  '2026-03-28T14'
  value: number;
}

interface RequestMetrics {
  service: ServiceType;
  tool?: CodeMateToolType;    // Code Mate 하위 도구
  model: ModelType;
  granularity: TimeGranularity; // 집계 시간 단위
  period: { from: string; to: string };
  timeSeries: TimeSeriesPoint[];  // 시간 단위별 요청 수 추이
  requestCount: number;       // 기간 내 총 요청 수
  responseTime: {
    avg: number;
    p50: number;
    p95: number;
    p99: number;
    timeSeries: TimeSeriesPoint[];  // 시간 단위별 평균 응답 시간 추이
  };
  tokens: {
    input: number;
    output: number;
    cacheHit: number;
    timeSeries: {               // 시간 단위별 토큰 사용 추이
      timestamp: string;
      input: number;
      output: number;
      cacheHit: number;
    }[];
  };
}

interface CodeMetrics {
  service: ServiceType;
  tool?: CodeMateToolType;
  granularity: TimeGranularity;
  period: { from: string; to: string };
  appliedLines: number;       // 기간 내 총 코드 적용 라인 수
  timeSeries: TimeSeriesPoint[];  // 시간 단위별 코드 적용 라인 추이
}
```

### 6.3 시스템 관련

```typescript
// types/system.ts
interface GpuInfo {
  gpuId: string;
  model: ModelType;
  status: 'active' | 'idle' | 'error';
  utilization: number;        // 0-100%
}

interface ModelServiceMapping {
  model: ModelType;
  services: {
    service: ServiceType;
    tools?: CodeMateToolType[];
  }[];
}

interface SystemOverview {
  totalGpuCount: number;
  activeGpuCount: number;
  gpuList: GpuInfo[];
  modelServiceMappings: ModelServiceMapping[];
}
```

---

## 7. API 엔드포인트 (예상)

| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/api/users/population` | 사용자 모수 조회 |
| GET | `/api/users?dept=&jobGroup=&developerType=` | 사용자 목록 (필터) |
| GET | `/api/metrics/requests?service=&model=&from=&to=&granularity=` | 요청 지표 조회 (시간 단위 지정) |
| GET | `/api/metrics/tokens?service=&model=&from=&to=&granularity=` | 토큰 사용량 조회 (시간 단위 지정) |
| GET | `/api/metrics/response-time?service=&model=&from=&to=&granularity=` | 응답 시간 조회 (시간 단위 지정) |
| GET | `/api/metrics/code-lines?service=&from=&to=&granularity=` | 코드 적용 라인 조회 (시간 단위 지정) |
| GET | `/api/system/gpus` | GPU 현황 조회 |
| GET | `/api/system/model-service-mapping` | 모델-서비스 연결 현황 |
| GET | `/api/metrics/overview?from=&to=&granularity=` | 대시보드 요약 KPI (시간 단위 지정) |

> **granularity 파라미터**: `year` | `month` | `week` | `day` | `hour`
> - 기본값: `day`
> - 데이터 출처: `hour` → summary-hourly, `day`/`week`/`month`/`year` → summary-daily 합산
> - 기간과 granularity 관계 예시:
>   - 최근 1주 → `day` (7개) 또는 `hour` (168개)
>   - 최근 4주 → `day` (28개) 또는 `week` (4개)
>   - 최근 1년 → `month` (12개) 또는 `week` (52개)
>   - 전체 기간 → `year` 또는 `month`

---

## 8. 구현 단계

### Phase 1: 프로젝트 초기화 및 레이아웃
1. Vite + React + TypeScript 프로젝트 생성
2. 의존성 설치 (Ant Design, ECharts, Zustand, React Query, Axios, React Router)
3. 디렉토리 구조 생성
4. AppLayout 컴포넌트 (Sider + Header + Content)
5. 라우터 설정 (4개 페이지)
6. Mock 데이터 준비

### Phase 2: 공통 컴포넌트
1. StatCard (KPI 카드) 컴포넌트
2. 필터 바 컴포넌트 (기간, 서비스, 모델 선택)
3. API 클라이언트 + React Query 훅 설정

### Phase 3: Overview 페이지
1. KPI 카드 5개 배치
2. 요청 수 추이 Line Chart
3. 서비스별 요청 비율 Pie Chart
4. 모델별 Token 사용량 Stacked Bar Chart
5. 응답 시간 분포 차트
6. 사용자 모수 현황 Progress Bar

### Phase 4: Service Metrics 페이지
1. 서비스/도구 탭 구성
2. 요청 추이 상세 차트 (모델별 비교)
3. Token 상세 분석 (Input/Output/Cache Hit)
4. 응답 시간 백분위 트렌드

### Phase 5: User Analytics 페이지
1. 사용자 모수 요약 카드
2. 직군별/소속별 사용 비율 차트
3. 개발자 유형별 비교 차트
4. 사용자 상세 테이블

### Phase 6: System Status 페이지
1. GPU 요약 카드
2. GPU당 모델 탑재 현황 테이블
3. 모델-서비스 연결 현황 (Sankey Diagram 또는 테이블)

---

## 9. 글로벌 필터

모든 페이지에서 사용 가능한 공통 필터:

| 필터 | 옵션 |
|------|------|
| **기간** | 오늘, 최근 7일, 최근 30일, 최근 1년, 커스텀 범위 |
| **시간 단위** | 연(year), 월(month), 주(week), 일(day), 시간(hour) |
| **서비스** | 전체, Code Mate, Code Pearl, Code Search |
| **Code Mate 도구** | 전체, Continue, Roo Code, OpenCode |
| **모델** | 전체, GLM 4.7, Qwen 3.5 |
| **개발자 유형** | 전체, Pro 개발자, Citizen 개발자, 구현 개발자 |
| **소속** | 전체, 부서 선택 |
| **직군** | 전체, S직군 등 |

> **시간 단위 자동 추천 로직**: 선택한 기간에 따라 적절한 기본 시간 단위를 자동 설정합니다.
> | 기간 | 기본 시간 단위 | 선택 가능 단위 |
> |------|---------------|---------------|
> | 오늘 | hour | hour |
> | 최근 7일 | day | hour, day |
> | 최근 30일 | day | hour, day, week |
> | 최근 1년 | month | day, week, month |
> | 커스텀 (≤1일) | hour | hour |
> | 커스텀 (≤90일) | day | hour, day, week, month |
> | 커스텀 (>90일) | month | week, month, year |

---

## 10. Mock 데이터 전략

실제 API가 준비되기 전까지 Mock 데이터를 사용합니다:
- `src/mocks/` 디렉토리에 JSON 형태 mock 데이터 배치
- API 클라이언트에서 환경 변수로 mock/real 전환 가능
- 현실적인 데이터 범위로 생성하여 차트 검증 용이

---

## 11. 주요 고려사항

1. **반응형 레이아웃**: Ant Design Grid 시스템 활용, 태블릿/데스크톱 대응
2. **다크 모드**: Ant Design 테마 토큰으로 라이트/다크 전환 지원
3. **데이터 갱신**: 주별 배치 적재 후 Summary 자동 갱신. React Query의 staleTime으로 캐싱 관리
4. **데이터 내보내기**: 테이블 데이터 CSV 다운로드 기능
5. **로딩/에러 상태**: Skeleton 로딩 + Error Boundary 처리
