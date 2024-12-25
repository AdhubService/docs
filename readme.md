# 애드허브 API 연동

## 개요

### 주요 기능

- 캠페인 목록 조회
- 캠페인 참여 요청
- 캠페인 완료시 매체사 서버로 콜백

### 매체사에서 전달해야 하는 정보

- 캠페인 목록 조회 및 참여 요청시 사용하는 매체사 서버 IP (최대 5개)
- 콜백 받을 매체사 서버의 URL 주소

## API 상세

### 서버 주소

```
https://api.adhub.site
```

### 인증

모든 API 요청에 매체사 식별을 위해 `publisher key`를 사용합니다.

### 매체사 Secret Key

매체사별로 발급되는 `secret key`는 무결성을 검증하기 위해 사용됩니다.

이 값은 절대로 외부에 노출되어서는 안되며, 서버 사이드에서만 안전하게 보관되어야 합니다. 유출된 경우 변경을 요청해야 합니다.

### 에러 응답

API 호출 시 에러가 발생하면 HTTP 코드가 200이 아닌 값과 함께 body에 다음과 같은 형식으로 에러코드를 반환합니다. 에러 코드 목록은 아래 '에러 코드' 섹션을 참고하세요.

```json
{
  "error_code": "COMM001"
}
```

### 1. 캠페인 목록 조회

캠페인의 목록을 페이징으로 반환합니다. 캠페인의 전체 개수가 많은 경우 페이징 방식으로 여러번 호출하세요.

캠페인의 이름 및 가격, 남은 수량, 비활성화 여부 등이 실시간으로 수정될 수 있기 때문에 API의 최신 결과로 기존 캠페인 데이터를 업데이트해야 합니다.

기존 목록에 있었으나 최신 목록에 더 이상 없는 캠페인은 비활성화 됐거나 수량이 소진된 캠페인입니다.

> [!NOTE]
> 캠페인 목록은 5분 정도의 간격으로 주기적으로 호출하여 캠페인 정보를 최신으로 유지해주세요.

#### 엔드포인트

```http
GET /campaigns
```

#### 요청 파라미터

| 파라미터            |   타입   | 필수 | 설명                   |
|-----------------|:------:|:--:|----------------------|
| `publisher_key` | String | O  | 매체사 식별 키             |
| `page`          |  Int   | O  | 페이지 번호 (0부터 시작)      |
| `limit`         |  Int   | O  | 페이지당 아이템 수 (최대 1000) |

#### 응답

| 필드명         |  타입   | 필수 | 설명             |
|-------------|:-----:|:--:|----------------|
| `total`     |  Int  | O  | 전체 캠페인 수       |
| `campaigns` | Array | O  | 현재 페이지의 캠페인 목록 |

#### campaigns 배열 내 항목의 필드들

| 필드명                       |   타입    | 필수 | 설명                                                                                                                                                                                                                         |
|---------------------------|:-------:|:--:|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `campaign_id`             | String  | O  | 캠페인 ID                                                                                                                                                                                                                     |
| `type`                    | String  | O  | 캠페인 타입. '캠페인 타입' 섹션 참조                                                                                                                                                                                                     |
| `title`                   | String  | O  | 캠페인 제목                                                                                                                                                                                                                     |
| `description`             | String  | O  | 캠페인 상세 설명                                                                                                                                                                                                                  |
| `icon_url`                | String  | O  | 캠페인 아이콘 이미지 URL                                                                                                                                                                                                            |
| `price`                   |   Int   | O  | 매체사에 지급되는 광고비                                                                                                                                                                                                              |
| `adid_mandatory`          | Boolean | X  | ADID(Android) 또는 IDFA(iOS) 필수 여부. 기본적으로 필수이며, false인 경우에만 이 값이 존재함                                                                                                                                                         |
| `repeatable`              | Boolean | X  | 동일 사용자의 반복 수행 가능 여부. true인 경우에만 값이 존재함                                                                                                                                                                                     |
| `repeat_interval_in_days` |   Int   | X  | 반복 수행이 가능한 경우 재수행이 가능한 최소 간격(일 단위)<br>- `repeatable`이 true인 경우에만 값이 존재함<br>- `repeatable`이 true인데, 이 값이 없거나 0이면 기간 제한없이 계속 재수행이 가능함<br>- 시간은 고려하지 않고 날짜만 계산함<br>예) 이 값이 1이면, 23:59에 수행했어도 날짜가 바뀌는 그 이튿날에는 00:00부터 재수행이 가능함 |
| `start_time`              |  Long   | X  | 캠페인 시작 시간 (Unix timestamp in milliseconds). 시작 시간 제한이 있는 경우에만 값이 존재함                                                                                                                                                       |
| `end_time`                |  Long   | X  | 캠페인 종료 시간 (Unix timestamp in milliseconds). 종료 시간 제한이 있는 경우에만 값이 존재함                                                                                                                                                       |
| `remaining_quantity`      |   Int   | X  | 남은 참여 가능 수량. 수량 제한이 있는 경우에만 값이 존재함                                                                                                                                                                                         |
| `target_os_list`          |  Array  | X  | 참여 가능한 OS 목록. OS 제한이 있는 경우에만 값이 존재함<br>- `A`: Android, `I`: iOS, `W`: Web                                                                                                                                                  |
| `target_min_age`          |   Int   | X  | 참여 가능한 최소 나이. 최소 나이 제한이 있는 경우에만 값이 존재함                                                                                                                                                                                     |
| `target_max_age`          |   Int   | X  | 참여 가능한 최대 나이. 최대 나이 제한이 있는 경우에만 값이 존재함                                                                                                                                                                                     |
| `target_gender`           | String  | X  | 참여 가능한 성별. 성별 제한이 있는 경우에만 값이 존재함<br>- `M`: 남성, `F`: 여성                                                                                                                                                                     |

응답 예시:

```json
{
  "total": 150,
  "campaigns": [
    {
      "campaign_id": "240325-abcd1234",
      "type": "QUESTION_AND_ANSWER",
      "title": "캠페인 1 제목",
      "description": "캠페인 1 설명",
      "icon_url": "https://example.com/icon.png",
      "price": 1000,
      "repeatable": true,
      "repeat_interval_in_days": 1,
      "start_time": 1703980800000,
      "end_time": 1704067199999,
      "remaining_quantity": 1000,
      "target_os_list": ["A", "I"],
      "target_min_age": 20,
      "target_max_age": 60,
      "target_gender": "M"
    },
    {
      "campaign_id": "240325-abcd1235",
      "type": "QUESTION_AND_ANSWER",
      "title": "캠페인 2 제목",
      "description": "캠페인 2 설명",
      "icon_url": "https://example.com/icon.png",
      "price": 1000,
      "adid_mandatory": true
    }
  ]
}
```

### 2. 캠페인 참여 요청

#### 엔드포인트

```http
GET /campaigns/{campaign_id}/join
```

#### 요청 파라미터

| 파라미터            |   타입   | 필수 | 설명                                                                          |
|-----------------|:------:|:--:|-----------------------------------------------------------------------------|
| `publisher_key` | String | O  | 매체사 식별 키                                                                    |
| `user_id`       | String | O  | 매체사의 사용자 ID                                                                 |
| `ip_address`    | String | O  | 사용자 IP 주소                                                                   |
| `os_type`       | String | X  | `A`: Android, `I`: iOS, `W`: Web                                            |
| `device_ad_id`  | String | X  | Android ADID 또는 iOS IDFA. `adid_mandatory`가 `true`인 캠페인은 이 값을 반드시 전달해야 합니다. |
| `age`           |  Int   | X  | 사용자 나이                                                                      |
| `gender`        | String | X  | `M`: 남성, `F`: 여성                                                            |
| `callback_data` | String | X  | 콜백 시 전달받을 데이터 (최대 255 bytes)                                                |

## 콜백

캠페인이 완료되면 매체사가 등록한 콜백 URL로 HTTP POST 요청을 보냅니다. 아래 파라미터들은 body에 JSON 형태로 전송됩니다.

> 요청 content type: `application/json`

| 필드명                        |   타입   | 필수 | 설명                                               |
|----------------------------|:------:|:--:|--------------------------------------------------|
| `user_id`                  | String | O  | 매체사의 사용자 ID                                      |
| `completed_transaction_id` | String | O  | 캠페인 완료 트랜잭션 ID. 이 값을 사용해서 매체사에서 자체적으로 중복을 체크해야 함 |
| `campaign_id`              | String | O  | 캠페인 ID                                           |
| `price`                    |  Int   | O  | 매체사에 지급되는 광고비 (원 단위)                             |
| `callback_data`            | String | X  | 참여 요청 시 전달받은 데이터                                 |
| `completed_time`           |  Long  | O  | 캠페인 완료 시간 (Unix timestamp in milliseconds)       |
| `signature`                | String | O  | 위변조 검증을 위한 HMAC-SHA256 서명                        |

### 콜백 서명 검증

`signature` 필드를 통해 콜백 요청을 반드시 검증해야 합니다. 이 값은 다음과 같이 생성합니다.

```
message = publisher_key + user_id + completed_transaction_id
signature = Base64(HMAC-SHA256(secret_key, message))
```

계산한 값과 콜백으로 전달한 값이 다른 경우 정상적인 콜백 요청이 아니기 때문에 성공으로 처리하면 안됩니다.

#### 서명 생성 예시

매체사 secret key 예시: `aB7cD9eF1hJ3kL5nP7rT9vX1zZ3pR5tN`

콜백 데이터 예시:

```
publisher_key = "mK9pV8zXnL4jR2wQ"
user_id = "publisher_user_12345"
completed_transaction_id = "240325-Kj8mN4pX2w"
```

서명 생성을 위한 메시지:

```
message = "mK9pV8zXnL4jR2wQpublisher_user_12345240325-Kj8mN4pX2w"
```

이 메시지를 HMAC-SHA256으로 서명 후 Base64 인코딩하면 다음과 같은 signature가 생성됩니다.

```
signature = "RWClSMyUqB+IjtHRHIh+nyMFRHdgyyU1pqYeohNdHOc="
```

### 콜백 응답

매체사에서 콜백을 처리한 이후 문제가 없으면 응답 코드 200에 body에 내용을 넣지 않은 채로 응답을 보내세요.

콜백에 대한 응답 코드가 200이 아닌 경우에는 콜백을 실패로 간주하며 48시간에 걸쳐 최대 10회 동안 콜백을 재시도 합니다. 열번째 재시도에 실패하면 그 이후로 더 시도하지 않습니다.

| 시도 횟수 | 캠페인 완료 시간부터 경과 시간 |
|:-----:|:-----------------:|
|  1회   |       2분 후        |
|  2회   |       5분 후        |
|  3회   |       10분 후       |
|  4회   |       30분 후       |
|  5회   |       90분 후       |
|  6회   |   180분 (3시간) 후    |
|  7회   |   360분 (6시간) 후    |
|  8회   |   720분 (12시간) 후   |
|  9회   |  1440분 (24시간) 후   |
|  10회  |  2880분 (48시간) 후   |

## 캠페인 타입

| 타입                             | 설명                           |
|--------------------------------|------------------------------|
| `NAVER_PLACE_SAVE_QUIZ`        | 네이버 플레이스 저장 (정답 맞추기)         |
| `NAVER_PLACE_SAVE`             | 네이버 플레이스 저장 (방식이 특정되지 않음)    |
| `NAVER_PLACE_TRAFFIC_QUIZ`     | 네이버 플레이스 트래픽 (정답 맞추기)        |
| `NAVER_PLACE_TRAFFIC`          | 네이버 플레이스 트래픽 (방식이 특정되지 않음)   |
| `NAVER_PLACE_NOTIFICATION`     | 네이버 플레이스 알림받기                |
| `NAVER_STORE_TRAFFIC_QUIZ`     | 네이버 쇼핑 트래픽 (정답 맞추기)          |
| `NAVER_STORE_TRAFFIC`          | 네이버 쇼핑 트래픽 (방식이 특정되지 않음)     |
| `NAVER_STORE_PRODUCT_FAVORITE` | 네이버 쇼핑 상품 찜하기                |
| `NAVER_STORE_NOTIFICATION`     | 네이버 스토어 알림받기                 |
| `KAKAO_PRODUCT_FAVORITE`       | 카카오톡 선물하기 상품 찜하기             |
| `GENERAL_QUIZ`                 | 일반 정답 맞추기 (대상 및 방식이 특정되지 않음) |

## 에러 코드

|    코드     | 설명                                                       |
|:---------:|----------------------------------------------------------|
| `ADHB001` | 존재하지 않는 publisher key                                    |
| `ADHB102` | 존재하지 않는 캠페인 ID                                           |
| `ADHB103` | 허용되지 않은 IP에서 접근함                                         |
| `ADHB106` | 이미 에러를 보고함                                               |
| `CAMP011` | 디바이스 AD ID가 필수인 캠페인에 AD ID 없이 참여 요청함                     |
| `CAMP101` | 종료된 캠페인                                                  |
| `CAMP102` | 이미 참여한 캠페인                                               |
| `CAMP104` | 광고 수량이 소진된 캠페인                                           |
| `CAMP105` | 타겟 사용자가 아님                                               |
| `CAMP106` | 참여할 수 있는 장비가 아님                                          |
| `CAMP107` | 캠페인 수행 후 완료 여부 확인 중인 상태임                                 |
| `CAMP109` | 사용자가 특정 제공사의 캠페인 타입에 참여할 수 있는 일일 한도에 도달함. 다음날부터 다시 시도 가능 |
| `CAMP110` | 참여 요청의 유효기간이 경과함                                         |
| `CAMP199` | 참여할 수 없는 캠페인                                             |
| `CAMP302` | 제휴사로부터 요청이 지연됨                                           |
| `COMM001` | 애드허브 서버 에러                                               |
