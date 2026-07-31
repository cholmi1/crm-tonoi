# 씨엔브이텍 CRM — 정식 서버 · 로그인 · 다중사용자 전환 설계서

작성일: 2026-07-31 · 대상: 현재 단일 파일 CRM 웹앱의 실운영 전환

---

## 1. 왜 전환이 필요한가

현재 웹앱은 데이터를 **브라우저 저장소(팀 공유)**에 두고 있습니다. 링크만 공유하면 팀이 같은 데이터를 볼 수 있어 초기 운영에는 충분하지만, 실운영으로 가면 다음이 한계로 드러납니다.

| 항목 | 현재(브라우저 저장) | 정식 서버 전환 후 |
|---|---|---|
| 로그인 | 없음(링크 아는 사람 누구나) | 계정·비밀번호 로그인 |
| 권한 | 전원 동일 | 관리자/직원/조회자 구분 |
| 데이터 보관 | 브라우저 환경에 종속, 유실 위험 | 클라우드 DB에 안전 보관·자동 백업 |
| 동시 편집 | 마지막 저장이 덮어씀 | 충돌 관리·이력 추적 |
| 접근 기기 | 저장소가 묶인 환경 | 어느 기기·브라우저에서나 동일 |
| 감사·이력 | 없음 | 누가 언제 무엇을 바꿨는지 기록 |

---

## 2. 권장 아키텍처

**결론: 프런트엔드는 지금 만든 단일 파일을 거의 그대로 쓰고, 저장소만 Supabase로 교체합니다.**

```
[사용자 브라우저]  ──HTTPS──►  [Supabase]
  현재 CRM 화면                   ├─ Auth  (로그인·역할)
  (HTML/JS 그대로)                ├─ Postgres DB (고객/판매/상담/AS)
  supabase-js 로 데이터 읽고쓰기   └─ Row Level Security (권한 규칙)
```

Supabase를 권장하는 이유:
- **관리형 PostgreSQL + 로그인 + 권한(RLS)**이 한 세트로 제공되어 별도 서버를 직접 운영할 필요가 없습니다.
- 서울 리전(AWS `ap-northeast-2`) 선택이 가능해 데이터가 국내에 위치합니다.
- 자바스크립트 SDK(`supabase-js`)로 현재 코드의 저장 함수만 바꾸면 됩니다. 화면·기능은 그대로 재사용합니다.
- 소규모 팀 기준 **무료~월 25달러** 선에서 시작할 수 있습니다.

대안 비교(참고):

| 방식 | 초기 난이도 | 운영 부담 | 월 비용(소규모) | 적합도 |
|---|---|---|---|---|
| **Supabase(권장)** | 낮음 | 낮음 | 0~25달러 | ★★★★★ |
| 자체 서버(Node + Postgres, 예: NCP/카페24) | 높음 | 높음(보안·백업 직접) | 3~10만원 | ★★★☆ |
| 노코드(Airtable 등) | 매우 낮음 | 낮음 | 인원당 과금 | ★★★(맞춤화 한계) |

---

## 3. 데이터베이스 스키마 (PostgreSQL)

현재 앱의 데이터 구조를 그대로 테이블로 옮긴 것입니다. Supabase SQL 편집기에 붙여넣으면 생성됩니다.

```sql
-- 사용자 프로필 및 역할
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  name text,
  role text not null default 'staff' check (role in ('admin','staff','viewer')),
  created_at timestamptz default now()
);

-- 품목
create table products (
  code text primary key,
  name text not null,
  warranty_months int not null default 12
);

-- 고객
create table customers (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  phone text,
  address text,
  memo text,
  created_at timestamptz default now(),
  created_by uuid references profiles(id)
);
create index on customers (phone);
create index on customers (name);

-- 판매(시리얼 단위)
create table sales (
  id uuid primary key default gen_random_uuid(),
  customer_id uuid references customers(id) on delete cascade,
  sold_at date,
  channel text,
  product_code text references products(code),
  serial text,
  qty int default 1,
  note text,
  status text default 'normal'
    check (status in ('normal','return_req','return_done','exchange_req','exchange_done','cancelled')),
  status_date date,
  status_reason text,
  status_note text,      -- 엑셀 원문 보존
  reviewed boolean default false,
  created_at timestamptz default now()
);
create index on sales (serial);
create index on sales (customer_id);
create index on sales (status);

-- 상담 이력
create table consultations (
  id uuid primary key default gen_random_uuid(),
  customer_id uuid references customers(id) on delete cascade,
  consulted_at date,
  type text,
  content text,
  handler text,
  follow_up text,
  created_at timestamptz default now(),
  created_by uuid references profiles(id)
);
create index on consultations (customer_id);

-- AS 이력
create table as_records (
  id uuid primary key default gen_random_uuid(),
  serial text,
  customer_id uuid references customers(id) on delete set null,
  received_at date,
  status text,
  symptom text,
  action text,
  cost text,
  handler text,
  created_at timestamptz default now()
);
create index on as_records (serial);
```

---

## 4. 로그인 · 권한 설계

**역할 3단계**
- `admin`(관리자) — 전무님·관리자: 모든 조회·수정·삭제, 사용자 관리
- `staff`(직원) — 고객·판매·상담·AS 등록/수정
- `viewer`(조회자) — 읽기 전용(예: 협력사·경영진 조회용)

**Row Level Security(RLS) 정책 예시** — 로그인한 직원만 접근, 삭제는 관리자만:

```sql
alter table customers enable row level security;

-- 로그인 사용자는 조회 가능
create policy "read for authenticated"
  on customers for select
  using (auth.role() = 'authenticated');

-- 직원·관리자는 등록·수정 가능
create policy "write for staff/admin"
  on customers for insert with check (
    exists (select 1 from profiles p where p.id = auth.uid() and p.role in ('admin','staff'))
  );
create policy "update for staff/admin"
  on customers for update using (
    exists (select 1 from profiles p where p.id = auth.uid() and p.role in ('admin','staff'))
  );

-- 삭제는 관리자만
create policy "delete for admin"
  on customers for delete using (
    exists (select 1 from profiles p where p.id = auth.uid() and p.role = 'admin')
  );
```

같은 방식으로 `sales`, `consultations`, `as_records`에 정책을 적용합니다. 신규 가입자는 기본 `staff`, 첫 관리자만 수동으로 `admin` 지정합니다.

---

## 5. 기존 데이터 마이그레이션

현재 앱에서 **[백업·복원] → 전체 백업(JSON)**으로 내려받은 파일을 그대로 사용합니다. 아래 순서로 옮깁니다.

1. 현재 앱에서 JSON 전체 백업 다운로드(423명·446건 포함).
2. Supabase에 위 스키마 생성.
3. 간단한 이관 스크립트(Node) 실행 — JSON을 읽어 `customers → sales → consultations → as_records` 순으로 삽입. (필드명이 1:1로 대응하므로 변환 로직이 단순합니다: `date→sold_at`, `product→product_code` 등.)
4. 건수 검증(고객 423 / 판매 446)을 확인하고 종료.

> 이관 스크립트는 전환 단계에서 실제 프로젝트 키에 맞춰 함께 작성해 드리겠습니다.

---

## 6. 프런트엔드 변경점 (최소)

화면·기능 코드는 그대로 두고, 저장 계층(`sget/sset`)만 Supabase 호출로 바꿉니다. 대응 관계:

| 현재(브라우저) | 전환 후(Supabase) |
|---|---|
| `window.storage.get('crm:customers')` | `supabase.from('customers').select('*')` |
| `window.storage.set(...)` (고객 저장) | `supabase.from('customers').upsert(row)` |
| 삭제 | `supabase.from('customers').delete().eq('id', id)` |
| 로그인 없음 | `supabase.auth.signInWithPassword(...)` + 로그인 화면 추가 |

즉, 저장 함수 6~8개와 로그인 화면 1개만 새로 작성하면 되고, 대시보드·검색·상담·반품·AS 화면은 재사용합니다.

---

## 7. 배포 · 운영 절차

1. Supabase 프로젝트 생성(서울 리전).
2. 스키마·RLS 정책 적용(3~4장).
3. 첫 관리자 계정 생성 및 `admin` 지정.
4. 기존 데이터 이관(5장).
5. 프런트엔드 저장 계층 교체(6장) 후 정적 호스팅에 배포 — Vercel/Netlify/Cloudflare Pages 중 택1(무료), 사내 도메인 연결 가능.
6. 팀원 계정 발급 및 역할 지정.
7. 정기 백업 확인(Supabase 자동 백업 + 월 1회 JSON/엑셀 수동 백업 병행 권장).

---

## 8. 예상 비용 (소규모 팀 기준)

| 항목 | 비용 |
|---|---|
| Supabase | 무료 플랜으로 시작 → 사용량 증가 시 Pro(월 25달러) |
| 프런트 호스팅(Vercel 등) | 무료 |
| 도메인(선택) | 연 1~2만원 |
| **합계** | **월 0원 시작, 확장 시 약 3만원대** |

현재 데이터 규모(고객 400여 명·판매 400여 건)는 무료 플랜 한도 내에서 충분히 운영됩니다.

---

## 9. 로드맵 · 다음 단계

- **1단계(지금)** — 브라우저 공유 버전으로 실사용 시작, 반품 사유 정리·상담 이력 축적.
- **2단계** — Supabase 프로젝트 생성 + 스키마 적용 + 데이터 이관(반나절~1일).
- **3단계** — 로그인 화면 추가 + 저장 계층 교체 + 호스팅 배포.
- **4단계** — 권한 세분화, 변경 이력(감사 로그), 재구매·해피콜 알림 등 확장.

원하시는 단계를 말씀해 주시면, 해당 단계의 실제 코드(이관 스크립트·로그인 화면·Supabase 연동 저장 계층)를 이어서 작성해 드리겠습니다.
