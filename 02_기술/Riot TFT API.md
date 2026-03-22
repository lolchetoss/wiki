---
분류: 기술
---

## 사용 가능한 API 목록

| API | 라우팅 | 설명 |
| --- | --- | --- |
| Account (account-v1) | 리전 | Riot ID로 puuid 조회 — 모든 API의 시작점 |
| Match History (tft-match-v1) | 리전 | 매치 ID 목록 + 매치 상세 데이터 |
| Summoner (tft-summoner-v1) | 플랫폼 | 소환사 프로필 (아이콘, 레벨) |
| League (tft-league-v1) | 플랫폼 | 랭크/리그 정보, 티어별 플레이어 목록 |
| Spectator (spectator-tft-v5) | 플랫폼 | 진행 중 게임 관전 데이터 (제한적) |
| Status (tft-status-v1) | 플랫폼 | TFT 서버 상태, 점검/장애 정보 |
| Data Dragon | 없음 (정적) | 챔피언, 아이템, 증강, 특성 이미지 및 기본 정보 |

## 라우팅 체계

API마다 **리전(Regional)** 또는 **플랫폼(Platform)** 라우팅을 사용한다. 잘못된 라우팅으로 요청하면 404가 반환되므로 주의.

### 리전 라우팅 (account-v1, tft-match-v1)

`{region}.api.riotgames.com` 형식으로 요청

| 리전 | 포함 플랫폼 |
| --- | --- |
| americas | NA1, BR1, LA1, LA2 |
| asia | KR, JP1 |
| europe | EUW1, EUN1, TR1, RU |
| sea | OC1, SG2, TW2, VN2, TH2, PH2 |

### 플랫폼 라우팅 (tft-league-v1, Tft-summoner-v1, Spectator-tft-v5, tft-status-v1)

`{platform}.api.riotgames.com` 형식으로 요청

사용 가능 플랫폼: `na1`, `br1`, `eun1`, `euw1`, `jp1`, `kr`, `la1`, `la2`, `me1`, `oc1`, `ru`, `sg2`, `tr1`, `tw2`, `vn2`

## Account API (account-v1)

Riot ID(닉네임#태그)로 puuid를 조회하는 API. **모든 TFT API 호출의 시작점.**

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/riot/account/v1/accounts/by-riot-id/{gameName}/{tagLine}` | Riot ID → puuid |
| GET | `/riot/account/v1/accounts/by-puuid/{puuid}` | puuid → Riot ID |
| GET | `/riot/account/v1/accounts/me` | 액세스 토큰으로 본인 계정 조회 (RSO) |
| GET | `/riot/account/v1/region/by-game/{game}/by-puuid/{puuid}` | puuid의 활성 리전 조회 (`game`=`tft`) |

**AccountDto 응답:**

```json
{
  "puuid": "string",       // 암호화된 PUUID (78자)
  "gameName": "string",    // 닉네임 (없을 수 있음)
  "tagLine": "string"      // 태그 (없을 수 있음)
}
```

> 전적 검색 흐름: Riot ID → account-v1으로 puuid 조회 → puuid로 match/league/summoner API 호출

## Match History API (tft-match-v1)

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/tft/match/v1/matches/by-puuid/{puuid}/ids` | 매치 ID 목록 |
| GET | `/tft/match/v1/matches/{matchId}` | 매치 상세 데이터 |

### 매치 ID 목록 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `start` | int | 0 | 시작 인덱스 |
| `count` | int | 20 | 반환 개수 |
| `startTime` | long | - | 시작 시각 (epoch 초) |
| `endTime` | long | - | 종료 시각 (epoch 초) |

> `startTime`/`endTime` 필터는 2021년 6월 16일 이후 매치에만 동작

### 매치 상세 응답 구조

```json
{
  "metadata": {
    "data_version": "6",
    "match_id": "KR_7172265457",
    "participants": ["puuid1", "puuid2", ...]  // 8명의 puuid
  },
  "info": {
    "endOfGameResult": "GameComplete",
    "gameCreation": 1730540807000,             // 게임 생성 시각 (epoch ms)
    "gameId": 7172265457,
    "game_datetime": 1730542154493,            // 게임 종료 시각 (epoch ms)
    "game_length": 1330.865234375,             // 게임 길이 (초)
    "game_version": "Linux Version 14.21.630.3012 ...",
    "mapId": 22,
    "queue_id": 1100,
    "queueId": 1100,                           // queue_id와 동일 (중복 필드)
    "tft_game_type": "standard",               // "standard" | "turbo"
    "tft_set_core_name": "TFTSet12",
    "tft_set_number": 12,
    "participants": [...]                      // 8명의 참가자 상세
  }
}
```

**queueId 값:**

| queueId | 모드 |
| --- | --- |
| 1100 | 랭크 |
| 1130 | 하이퍼롤 |
| 6100 | 더블업 |

### Participant DTO

```json
{
  "puuid": "string",
  "riotIdGameName": "닉네임",
  "riotIdTagline": "태그",
  "placement": 1,                          // 최종 등수 (1~8)
  "level": 9,                              // 최종 레벨
  "gold_left": 2,                          // 남은 골드 (최종값만)
  "last_round": 18,                        // 마지막 라운드 (스테이지 2-1 = 라운드 5)
  "time_eliminated": 981.24,               // 탈락 시점 (초)
  "players_eliminated": 0,                 // 처치한 플레이어 수
  "total_damage_to_players": 16,           // 총 피해량
  "win": false,
  "augments": [                            // 증강 3개 (배열 순서 = 선택 순서)
    "TFT12_Augment_BlitzcrankCarry",
    "TFT6_Augment_PortableForge",
    "TFT12_Augment_HoneymancyCrown"
  ],
  "companion": {                           // 꼬마 전설이
    "content_ID": "uuid",
    "item_ID": 6001,
    "skin_ID": 1,
    "species": "PetPenguKnight"
  },
  "missions": {                            // 추가 통계
    "PlayerScore2": 49,
    "Kills": 5,
    "GoldEarned": 120,
    "DamageDealt": 3500
  },
  "partner_group_id": 0,                   // 더블업 파트너 그룹 ID
  "traits": [...],
  "units": [...]
}
```

### Trait DTO

```json
{
  "name": "TFT12_Honeymancy",        // 특성 ID (Data Dragon과 매핑)
  "num_units": 7,                     // 해당 특성을 가진 유닛 수
  "style": 3,                         // 0=미활성, 1=브론즈, 2=실버, 3=골드, 4=크로매틱
  "tier_current": 3,                  // 현재 달성 티어
  "tier_total": 3                     // 최대 티어
}
```

- `style=0`이면 미활성 (유닛은 있지만 최소 인원 미달)
- 모든 특성이 나열되므로 미활성 특성도 포함됨

### Unit DTO

```json
{
  "character_id": "TFT12_Veigar",          // 챔피언 ID (patch 9.22+)
  "itemNames": [                            // 장착 아이템 이름 (최대 3개)
    "TFT_Item_Artifact_LudensTempest",
    "TFT_Item_ArchangelsStaff",
    "TFT_Item_BlueBuff"
  ],
  "items": [99, 19, 44],                   // 장착 아이템 ID (최대 3개)
  "name": "",                               // 보통 빈 문자열
  "rarity": 2,                              // 코스트 티어 (코스트와 다름, 아래 매핑 참고)
  "tier": 3                                 // 성급 (1=1성, 2=2성, 3=3성)
}
```

**rarity 매핑** (코스트와 직접 대응하지 않음에 주의):

| rarity | 코스트 |
| --- | --- |
| 0 | 1코스트 |
| 1 | 2코스트 |
| 2 | 3코스트 |
| 4 | 4코스트 |
| 6 | 5코스트 |
| 9 | 특수 유닛 |

**아이템 접두사:**

| 접두사 | 유형 |
| --- | --- |
| `TFT_Item_` | 일반 아이템 |
| `TFT_Item_Artifact_` | 아티팩트 |
| `TFTSet_Item_` | 세트 고유 아이템 |

- 엠블렘 아이템으로 특성 부여 시 traits에 반영됨
- 같은 챔피언이 중복 등장 가능 (벤치 유닛 포함)

## Summoner API (tft-summoner-v1)

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/tft/summoner/v1/summoners/by-puuid/{puuid}` | puuid로 소환사 조회 |
| GET | `/tft/summoner/v1/summoners/me` | 액세스 토큰으로 본인 조회 (RSO) |

**SummonerDTO 응답:**

```json
{
  "puuid": "string",              // 78자
  "profileIconId": 4353,          // 프로필 아이콘 ID
  "revisionDate": 1730542154493,  // 마지막 갱신 (epoch ms)
  "summonerLevel": 150,
  "id": "string"                  // ⚠️ DEPRECATED — puuid 사용 권장
}
```

> Summoner ID(`id` 필드)는 폐기 예정. 모든 곳에서 puuid를 사용할 것

## League API (tft-league-v1)

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/tft/league/v1/challenger` | 챌린저 리그 전체 |
| GET | `/tft/league/v1/grandmaster` | 그랜드마스터 리그 전체 |
| GET | `/tft/league/v1/master` | 마스터 리그 전체 |
| GET | `/tft/league/v1/entries/{tier}/{division}` | 특정 티어/디비전 목록 (페이징) |
| GET | `/tft/league/v1/by-puuid/{puuid}` | 특정 플레이어의 리그 정보 |
| GET | `/tft/league/v1/leagues/{leagueId}` | 리그 ID로 조회 (비활성 포함) |
| GET | `/tft/league/v1/rated-ladders/{queue}/top` | 하이퍼롤 상위 랭킹 |

**파라미터:**
- `queue`: `RANKED_TFT` (기본값), `RANKED_TFT_DOUBLE_UP`
- `tier`: `IRON`, `BRONZE`, `SILVER`, `GOLD`, `PLATINUM`, `EMERALD`, `DIAMOND`
- `division`: `I`, `II`, `III`, `IV`

**LeagueEntryDTO 응답:**

```json
{
  "puuid": "string",
  "queueType": "RANKED_TFT",
  "tier": "DIAMOND",
  "rank": "II",
  "leaguePoints": 75,
  "wins": 50,           // ⚠️ TFT에서 wins = 1등 횟수
  "losses": 120,        // ⚠️ TFT에서 losses = 2~8등 횟수
  "hotStreak": false,
  "veteran": false,
  "freshBlood": true,
  "inactive": false
}
```

> TFT의 wins/losses는 LoL과 다름. wins = **1등만**, losses = **2~8등 전부**

**하이퍼롤 전용 필드:**
- `ratedTier`: `ORANGE`, `PURPLE`, `BLUE`, `GREEN`, `GRAY`
- `ratedRating`: 정수 레이팅
- `hotStreak`, `veteran` 등 일반 필드는 제외됨

## Spectator API (spectator-tft-v5)

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/lol/spectator/tft/v5/active-games/by-puuid/{puuid}` | 현재 진행 중인 게임 조회 |

- 경로가 `/lol/spectator/tft/`로 시작 (`/tft/spectator/`가 아님)
- 플레이어가 게임 중이 아니면 **404** 반환 (빈 응답이 아님)
- 제공 데이터: 게임 ID, 시작 시간, 참가자 목록, 관전 암호화 키
- 보드 상태, 챔피언 구성 등 상세 데이터는 미제공

## Status API (tft-status-v1)

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/tft/status/v1/platform-data` | TFT 서버 상태 조회 |

- 점검(`maintenances`)과 장애(`incidents`) 정보 반환
- 장애 심각도: `info`, `warning`, `critical`
- 데이터 수집 파이프라인에서 서버 상태 확인용으로 활용 가능

## Rate Limits

| 키 유형 | 제한 1 | 제한 2 |
| --- | --- | --- |
| Development Key | 20 req / 1초 | 100 req / 2분 |
| Personal Key | 20 req / 1초 | 100 req / 2분 |
| Production Key | 500 req / 10초 | 30,000 req / 10분 |

### Rate Limit 유형

| 유형 | 적용 범위 | 설명 |
| --- | --- | --- |
| Application | 키 × 리전 | 한 키의 전체 요청량 제한 |
| Method | 엔드포인트 × 키 × 리전 | 특정 엔드포인트별 제한 |
| Service | 서비스 × 리전 | 모든 앱이 공유하는 서비스 레벨 제한 |

### 키 종류

- **Development Key**: 로그인 시 자동 발급, **24시간마다 만료** — 개발/테스트용
- **Personal Key**: 제품 등록 후 발급, 만료 없음 — 소규모 개인 프로젝트용
- **Production Key**: 대규모 공개 서비스용, 동작하는 프로토타입 필요, RSO 연동 필수

### 429 응답 처리

- `Retry-After` 헤더로 재시도 대기 시간(초) 제공
- `X-Rate-Limit-Type` 헤더로 어떤 제한에 걸렸는지 확인 가능
- Rate limit은 **리전별 독립 적용** — 여러 리전 동시 요청 가능

## Data Dragon (정적 데이터)

Riot이 챔피언/아이템/증강/특성의 이미지와 기본 정보를 제공하는 공개 CDN. 게임 패치마다 업데이트된다.

> 직접 서빙하지 않고 우리 S3로 복사하는 이유: Riot CDN 장애 시 사이트 이미지가 깨지고, 응답 속도를 제어할 수 없으며, 한국어 이름 매핑 등 데이터 가공이 불가. 상세 구조는 [[시스템 디자인]]의 정적 데이터 관리 참고.

### 버전 확인

```
GET https://ddragon.leagueoflegends.com/api/versions.json
// → ["16.6.1", "16.5.1", ...] (최신 버전이 첫 번째)
```

> Data Dragon은 패치 후 수동 업데이트되므로 지연이 있을 수 있음

### TFT 정적 데이터 엔드포인트

```
// 챔피언
GET https://ddragon.leagueoflegends.com/cdn/{version}/data/en_US/tft-champion.json

// 아이템
GET https://ddragon.leagueoflegends.com/cdn/{version}/data/en_US/tft-item.json

// 특성
GET https://ddragon.leagueoflegends.com/cdn/{version}/data/en_US/tft-trait.json

// 증강
GET https://ddragon.leagueoflegends.com/cdn/{version}/data/en_US/tft-augments.json
```

### Data Dragon 제공 범위

| 데이터 | 제공 필드 | 미제공 |
| --- | --- | --- |
| 챔피언 | `id`, `name`, `tier`(코스트), `image` | 특성 목록, 스탯, 스킬 |
| 아이템 | `id`, `name`, `image` | 설명, 스탯, 조합법 |
| 특성 | `id`, `name`, `image` | 활성 조건(breakpoint), 효과 설명 |
| 증강 | `id`, `name`, `description`, `image` | — (유일하게 설명 포함) |

> Data Dragon만으로는 정보 부족 — Community Dragon으로 보완 필수

### Community Dragon (보충 데이터)

Data Dragon에 없는 상세 데이터:

- 챔피언별 특성 목록, 코스트: `https://raw.communitydragon.org/latest/plugins/rcp-be-lol-game-data/global/default/v1/tftchampions.json`
- 아이템 상세 (스탯, 조합법): `https://raw.communitydragon.org/latest/plugins/rcp-be-lol-game-data/global/default/v1/tftitems.json`
- 전체 세트 데이터 (대용량): `https://raw.communitydragon.org/latest/cdragon/tft/en_us.json`

## 대규모 데이터 수집 방법론

### 1단계: 시드 플레이어 수집

```
GET /tft/league/v1/challenger
GET /tft/league/v1/grandmaster
GET /tft/league/v1/master
```

Challenger/Grandmaster/Master 리그의 모든 소환사 puuid를 수집하여 시드 풀 구성

### 2단계: 매치 ID 폴링

```
GET /tft/match/v1/matches/by-puuid/{puuid}/ids?count=20
```

각 시드 플레이어의 최근 매치 ID를 주기적으로 폴링 (중복 제거 필수)

### 3단계: 매치 상세 수집

```
GET /tft/match/v1/matches/{matchId}
```

새로운 매치 ID에 대해 상세 데이터 수집. 1개 매치 = 8명의 전체 데이터 포함

### 4단계: 그래프 확장 (Snowball Crawling)

수집한 매치에서 새로운 puuid를 발견하면 시드 풀에 추가하여 커버리지 확대. 고랭크뿐 아니라 다양한 랭크대의 데이터를 점진적으로 확보

### Rate Limit 관리

- Production Key 기준: 30,000 req / 10분 = 초당 약 50 req
- 리전별 독립이므로 KR, NA, EUW 등 병렬 수집 가능
- 매치 1건에 8명 데이터 포함 → 매치 상세가 정보 밀도 높음
- 대규모 서비스는 여러 Production Key + 분산 워커로 운영

## 주의사항

### API 사용 시 주의할 점

- **Summoner ID 폐기 예정** — `id` 필드 대신 `puuid`를 사용할 것
- **rarity ≠ 코스트** — Unit DTO의 `rarity`는 골드 코스트와 직접 대응하지 않음 (위 매핑 테이블 참고)
- **wins/losses 의미** — TFT에서 wins = 1등만, losses = 2~8등 전부 (LoL과 다름)
- **시간 필터 제한** — 매치 목록의 `startTime`/`endTime`은 2021-06-16 이후만 동작
- **`game_variation` 폐기** — 매치 데이터의 해당 필드는 deprecated
- **`character_id`** — patch 9.22+ (data_version "2") 부터만 제공
- **queue_id 중복** — `queue_id`와 `queueId`가 동일한 값으로 중복 존재
- **Spectator 경로** — `/lol/spectator/tft/v5/`로 시작 (`/tft/spectator/`가 아님)
- **Data Dragon 지연** — 패치 후 수동 업데이트되므로 즉시 반영되지 않을 수 있음

### Riot 정책 금지사항

- 상대방 플레이 추적/예측 서비스
- 꼬마 전설이 기반 증강의 승률 표시
- 실시간 인게임 추천
- 플레이어 의사결정을 대체하는 기능
- 베팅/도박 연동
- 플레이어 익명성 해제
- 하나의 Production Key로 여러 제품 운영

### Riot 정책 필수사항

- HTTPS 필수
- "Riot Games가 보증하지 않음" 고지 표시 필수
- 무료 티어 유지 필수 (유료 콘텐츠는 부가가치가 있어야 함)
- API Key를 배포 바이너리에 하드코딩 금지

## 제약 사항

- **사후(post-game) 데이터 중심** — 인게임 실시간 데이터(상점, 보드, 골드) 미제공
- **라운드별 진행 데이터 없음** — 최종 보드 상태만 기록
  - AI 피드백에서 "N라운드에 뭘 했어야 했다" 류의 분석은 API만으로 불가
  - 우회 방안: 통계적 추론(유사 조합의 승률 비교), 최종 상태 기반 역추론, 유사 게임 패턴 매칭
- 실시간 오버레이는 Overwolf + 클라이언트 메모리 읽기 방식 (스코프 외)

## 참고 링크

- Riot Developer Portal: https://developer.riotgames.com
- TFT API 문서: https://developer.riotgames.com/apis#tft-match-v1
- API 정책: https://developer.riotgames.com/policies/general
- Data Dragon: https://developer.riotgames.com/docs/tft#data-dragon
- Data Dragon 버전: https://ddragon.leagueoflegends.com/api/versions.json
- Community Dragon: https://raw.communitydragon.org/latest/
- API Schema 예시: https://github.com/MingweiSamuel/riotapi-schema
