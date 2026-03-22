# Solana Orca Swap Indexer

솔라나 메인넷 Orca Whirlpool SOL-USDC 풀의 스왑 트랜잭션을 실시간 수집하고, TimescaleDB에 적재한 뒤 REST API로 조회하는 인덱서입니다.

## 아키텍처

```
Solana Mainnet
     │
     │ Yellowstone gRPC (실시간 스트림)
     ▼
┌──────────┐     INSERT      ┌──────────────┐
│ Indexer   │ ──────────────▶ │ TimescaleDB  │
│ (indexer) │                 │ (PostgreSQL)  │
└──────────┘                 └──────┬───────┘
                                    │ SELECT
                              ┌─────▼──────┐
                              │ API Server │
                              │ (Express)  │
                              └────────────┘
                               :3000
```

## 사전 요구사항

- Node.js 18+
- Docker & Docker Compose
- Yellowstone gRPC 엔드포인트 및 토큰 (Triton, Helius 등)

## 환경변수 설정

`.env.example`을 복사하여 `.env` 파일을 생성합니다.

```bash
cp .env.example .env
```

`.env` 파일을 열어 아래 값들을 설정합니다.

```dotenv
# Yellowstone gRPC 접속 정보
# Triton, Helius, Shyft 등의 RPC 제공자에서 gRPC 엔드포인트를 발급받습니다.
GRPC_URL=https://grpc.mainnet.solana.blockdaemon.tech
GRPC_TOKEN=your_token_here

# PostgreSQL / TimescaleDB 접속 정보
# docker-compose.yml의 설정과 일치해야 합니다.
DB_HOST=localhost
DB_PORT=5432
DB_USER=solana
DB_PASSWORD=solana
DB_NAME=solana_indexer

# API 서버 포트
API_PORT=3000

# 최소 필터 금액 (USDC 기준, 이 금액 미만 스왑은 무시)
MIN_USDC_AMOUNT=10
```

| 변수 | 설명 | 기본값 |
|------|------|--------|
| `GRPC_URL` | Yellowstone gRPC 엔드포인트 URL | - |
| `GRPC_TOKEN` | gRPC 인증 토큰 | - |
| `DB_HOST` | TimescaleDB 호스트 | `localhost` |
| `DB_PORT` | TimescaleDB 포트 | `5432` |
| `DB_USER` | DB 사용자명 | `solana` |
| `DB_PASSWORD` | DB 비밀번호 | `solana` |
| `DB_NAME` | 데이터베이스 이름 | `solana_indexer` |
| `API_PORT` | REST API 서버 포트 | `3000` |
| `MIN_USDC_AMOUNT` | 저장할 최소 USDC 금액 | `10` |

## 데이터베이스 실행

Docker Compose로 TimescaleDB 컨테이너를 실행합니다.

```bash
# 데이터베이스 시작 (백그라운드)
docker compose up -d

# 로그 확인
docker compose logs -f timescaledb

# 데이터베이스 중지
docker compose down

# 데이터베이스 중지 + 볼륨 삭제 (데이터 초기화)
docker compose down -v
```

컨테이너가 시작되면 `init.sql`이 자동 실행되어 다음이 생성됩니다:

- `orca_swaps` 테이블 (TimescaleDB 하이퍼테이블)
- `five_min_volume` Continuous Aggregate 뷰 (5분 단위 거래량 집계)

DB가 정상 실행 중인지 확인하려면:

```bash
docker exec -it solana-timescaledb psql -U solana -d solana_indexer -c '\dt'
```

## 애플리케이션 실행

```bash
# 의존성 설치
npm install

# 인덱서 + API 서버 동시 실행
npm run dev

# 또는 개별 실행
npm run dev:indexer   # 인덱서만
npm run dev:server    # API 서버만
```

## API 엔드포인트

### `GET /api/swaps`

최근 스왑 내역 20건을 시간 역순으로 반환합니다.

```bash
curl http://localhost:3000/api/swaps
```

```json
{
  "success": true,
  "data": [
    {
      "time": "2026-03-23T12:00:00.000Z",
      "signature": "5K8t...",
      "wallet_address": "7xKX...",
      "sol_amount": 1.5432,
      "usdc_amount": 234.56,
      "is_buy": true
    }
  ],
  "count": 20
}
```

### `GET /api/volume`

최근 1시간의 5분 단위 누적 거래량을 반환합니다.

```bash
curl http://localhost:3000/api/volume
```

```json
{
  "success": true,
  "data": [
    {
      "bucket": "2026-03-23T11:00:00.000Z",
      "total_usdc_volume": 12345.67,
      "tx_count": 42
    }
  ],
  "count": 12
}
```

### `GET /health`

서버 상태 확인용 헬스체크 엔드포인트입니다.

```bash
curl http://localhost:3000/health
```

## 프로젝트 구조

```
solana-bootcamp/
├── docker-compose.yml     # TimescaleDB 컨테이너 정의
├── init.sql               # DB 스키마 초기화 (테이블, 하이퍼테이블, CA 뷰)
├── .env.example           # 환경변수 템플릿
├── package.json
├── tsconfig.json
└── src/
    ├── config.ts          # 환경변수 로딩 및 상수 정의
    ├── db.ts              # PostgreSQL 연결 풀 및 쿼리 함수
    ├── parser.ts          # Whirlpool 스왑 트랜잭션 파싱 로직
    ├── indexer.ts         # Yellowstone gRPC 스트림 구독 및 필터링
    └── server.ts          # Express REST API 서버
```
