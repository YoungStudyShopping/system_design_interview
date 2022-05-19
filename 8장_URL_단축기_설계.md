# 8장 URL 단축기 설계
### 1단계 - 문제 이해 및 설계 범위 확정
시스템 설계 면접 문제는 의도적으로 정해진 결말을 갖지 않는다.
질문을 통해 모호함을 줄이고 요구사항을 알아낼 것

<details>
<summary>요구사항 구체화를 위한 질의응답 예 (p.127-8)</summary>
 
> - URL 단축기가 어떻게 동작해야할지? (예제 확인)
>   - input : (원본 URL) => output : https://tinyurl.com/y7ke-ocwj (단축 URL)
> - 트래픽 규모?
>   - 매일 1억 (100million) 개의 단축 URL 생성 가능해야
> - 단축 URL 길이?
>   - 짧을 수록 good
> - 단축 URL 포함 문자 제약 사항?
>   - 숫자 [0-9] 와 영문자 ([a-z],[A-Z]) 허용
> - 단축 URL 을 시스템에서 지우거나 갱신 가능해야하는지?
>   - NO

</details>

- 요구사항
  - URL 단축
  - URL 리디렉션 (redirection)
  - 높은 가용성과 규모 확장성, 장애 감내
- 개략적 추정
  - 쓰기 연산 : 매일 1억 개 생성
    - 초당 쓰기 연산 : 1160 write/s (= 1억/24/3600)
  - 읽기 연산 : 읽기 쓰기 비율을 10:1로 가정
    - 초당 읽기 연산 : 11600 read/s
  - 저장 용량
    - 10년 운영시 3650억개 레코드 보관 필요
    - 축약 전 URL 평균 길이를 100으로 가정
    - 10년간 필요 저장 용량 : 36.5TB (= 3650억 * 100bytes)
  - *\* 계산 완료 후, 면접관과 결과를 점검하여 합의 후 진행할 것.*

### 2단계 - 개략적 설계안 제시 및 동의 구하기
- API 엔드포인트 (Endpoint)
- URL 리디렉션 (Redirection)
- URL 단축 플로

#### (1) API 엔드포인트
- API Endpoint 설계
  - [RESTful API](https://www.restapitutorial.com/index.html) (Representational State Transfer) 스타일로 설계
- 기본적으로 2개의 엔드포인트 필요
  - URL 단축용 
    - `POST /api/v1/data/shorten?longUrl={longUrlString}`
    - RESPONSE : 단축 URL
  - URL 리디렉션용
    - `GET /api/v1/shoprtUrl`
    - RESPONSE : HTTP redirection 목적지가 될 원래 URL

#### (2) URL 리디렉션
<img alt="8-2" src="https://user-images.githubusercontent.com/26691216/168504892-e1702025-f600-4b74-ab2a-3347db4a31e1.jpg" width=300>

- 단축 URL (tynyurl) 방문
  - REQ : `GET https://tinyurl.com/qt5opu`
  - RES 
    ```
    HTTP/1.1 301 Permanently Moved
    Location: https://www.amazon.com/dp/B01... (원래 URL)
    ```
- Redirection 응답
  - `301 Permanently Moved` : HTTP 요청 처리 책임이 **영구적으로** Location 헤더의 URL로 이전 (브라우저가 응답 캐싱)
    - 서버 부하 감소
  - `302 Found` : 주어진 URL 로의 요청이 **일시적으로** Location 헤더의 URL이 처리 (캐싱 x)
    - 트래픽 분석 (클릭 발생률, 발생 위치 추적 등) 에 유리
- URL Redirection 구현 방법
  - <단축 URL, 원래 URL> 쌍의 해시 테이블 사용
    - => 가장 직관적인 방법


#### (3) URL 단축 플로
<img alt="8-3" src="https://user-images.githubusercontent.com/26691216/168504890-3ac183c8-488f-4b99-86a2-1bec9b6cf12a.jpg" width=500>

- 해시 함수
  - 단축 URL 생성을 위한 해시 함수 fx 필요
    - `f(원본 URL) = hashValue`
    - 단축 URL : https://tinyurl.com/ **{hashValue}**
- 요구 사항
  - input (긴 URL) 이 다른 값이면 output (해시 값) 도 달라야한다
  - 계산된 output (해시 값)은 원래 input (긴 URL) 로 복원될 수 있어야 한다.

### 3단계 - 상세 설계
\+ 데이터 모델, 해시 함수, URL 단축 & 리디렉션 관련 구체적인 설계안

#### 데이터 모델
- <단축 URL, 원래 URL> 해시 테이블
	- 초기 전략은 가능하나 실제로는 어려움 (유한한 메모리와 비용)
	- => **RDB** 에 저장
- 관계형 데이터베이스 (RDB) 테이블
	- pk : id (pk)
	- columns: id, shortURL, longURL

#### 해시 함수 (hash function)
- 해시값 길이 (n)
	- hashValue 에 사용 가능한 문자 : 62개 ([0-9, a-z, A-Z])
	- hashValue 길이 (n) 의 최솟값 : 7
		- 예상 레코드 수 3650억개
		- 가능 레코드 수 62^n 개 => **n=7**
			- ...
			- 62^6 = 약 500억
			- 62^7 = 약 3.5조
	- 즉, **원래 URL => 7글자 문자열로 줄일 수 있는 해시 함수** 필요

<img alt="8-5" src="https://user-images.githubusercontent.com/26691216/168504888-ee36be65-6540-4035-b80e-4a164929c5c2.jpg" width=500>

- 해시 함수 구현 방법
	- 해시 후 충돌 해소
		- 기존의 해시 함수 사용 후 첫 7글자만 이용 후 충돌 해소될 때까지 정한 문자열 덧붙이기
			- ex. CRC32, MD6, SHA-1 (=> 모두 결과값이 7자 이상)
		- 단점 : DB 질의로 인한 오버헤드 (=> 블룸 필터로 성능 개선 가능)
	- base-62 변환
		- 진법 변환 (base conversion) : 수의 표현 방식이 다른 두 시스템이 같은 수를 공유하여야하는 경우 유용 
		- 62진법 사용 
			- '0' ~ '9' : 0 ~ 9 
			- 'a' ~ 'z' : 10 ~ 35
			- 'A' ~ 'Z' : 36 ~ 61
			- ex. 11157 => '2TX' (62 base)


|                             | 해시 후 충돌 해소 방법     | base-62 변환 방법                                                |
|-----------------------------|----------------------------|------------------------------------------------------------------|
| 단축 URL의 길이             | 고정됨 (=n)                | 가변적 (ID값에 비례)                                             |
| 유일성이 보장되는 ID 생성기 | 필요 x                     | 필요 o                                                           |
| 충돌 발생 여부              | 충돌 가능. 해소 전략 필요. | ID 유일성이 보장되므로 충돌 불가능                               |
| 다음에 쓸 URL 예측 가능     | 불가능                     | ID가 1씩 증가하는 값이라 가정하면 다음 URL 예측 가능 (보안 이슈) |


#### URL 단축키 상세 설계
- URL 단축키 => 시스템의 핵심 컴포넌트
	- 처리 흐름은 논리적으로 단순하고 기능적으로 언제나 동작하는 상태 유지 필요 

<img alt="8-7" src="https://user-images.githubusercontent.com/26691216/168504882-c74ac046-f92a-442a-aacb-8d6e0b59ed1c.jpg" width=300>

- base-62 변환 방법을 이용한 URL 단축기 순서도 예제 (p.136)
- ID 생성기 : 단축 URL 을 만들기 위해 사용할 ID 생성
	- ID는 전역적 유일성 (globally unique) 을 보장하는 10진수
	- 고도 분산 환경에서 유일성을 보장하는 생성기를 만드는 건 어려운 일 => **[7장 참고](./7장_분산%20시스템을%20위한%20유일%20ID%20생성기%20설계.md)**

#### URL 리디렉션 상세 설계

<img alt="8-8" src="https://user-images.githubusercontent.com/26691216/168504874-bc44a84f-d412-40bd-bc68-f43f2680fd32.jpg" width=500>

- 쓰기보다 읽기가 많은 시스템 (=> 캐시 저장으로 성능 향상)


### 4단계 - 마무리

URL 단축기의 API, 데이터 모델, 해시 함수, URL 단축 및 리디렉션 절차 설계

추가로 시간이 남는다면 면접관과 이야기 해볼 것
- 처리율 제한 장치 (rate limiter) : 많은 요청에 대한 무력화 (잠재적 보안 결점) 보완 => [4장 참고](./4장_처리율_제한_장치의_설계.md)
- 웹 서버의 규모 확장 (Stateless)
- 데이터베이스의 규모 확장 (다중화, 샤딩)
- 데이터 분석 솔루션 통합
- 가용성, 데이터 일관성, 안정성 => [1장 참고](./1장_사용자_수에_따른_규모_확장성.md)
