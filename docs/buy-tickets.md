# 티켓 구매 프로세스

### 유스케이스 명: 티켓 구매 (Buy Tickets)

**선행 조건**:

-   고객이 시스템에 로그인되어 있어야 한다.
-   원하는 영화와 극장의 상영 시간 및 좌석이 사용 가능해야 한다.

**기본 흐름**:

1. 고객은 극장 예매 시스템에 접속한다.
1. 시스템은 주간 인기, 월간 인기, 최신 순서로 정렬된 영화 목록을 제공한다.
1. 고객은 영화 목록에서 원하는 영화를 선택한다.
1. 시스템은 영화를 상영하는 극장 목록을 제공한다.
    - 사용자의 현재 위치를 기반으로 반경 5km의 가까운 극장을 추천
    - 특정 지역을 선택하여 해당 지역의 극장 목록을 제공
    - 없으면 가장 가까운 극장 5개
1. 고객은 상영하는 극장을 선택한다.
1. 시스템은 선택된 극장에서 영화 상영일 목록을 제공한다.
1. 고객은 원하는 상영일을 선택한다.
1. 시스템은 선택한 상영일의 상세 정보를 제공한다.
    - 상영 시간 별 남은 좌석수를 포함해야 한다.
1. 고객은 원하는 상영 시간을 선택한다.
1. 시스템은 해당 상영의 선택이 가능한 좌석을 보여준다.
    - 좌석은 로얄석, 커플석 등 등급이 있다.
    - 좌석은 층,블록,열,좌석번호로 지정된다.
1. 고객은 하나 이상의 좌석을 선택한다.
    - 선택한 좌석은 10분간 선점된다.
1. 시스템은 선택한 좌석과 총 가격을 요약하여 보여준다.
1. 고객은 결제 정보를 입력하고 구매를 확정한다.
1. 시스템은 결제를 처리하고, 티켓 구매 성공 메시지와 함께 전자 티켓을 제공한다.

**대안 흐름**:

-   A1. 원하는 상영 시간이나 좌석이 사용 불가능한 경우:
    -   시스템은 사용 불가능한 메시지를 표시하고, 다른 상영 시간이나 좌석을 선택하도록 유도한다.
-   A2. 결제가 실패한 경우:
    -   시스템은 결제 실패 메시지를 표시하고, 결제 정보를 재입력하거나 다른 결제 방법을 선택하도록 유도한다.

**후행 조건**:

-   고객은 구매한 티켓에 대한 전자 티켓을 이메일로 받는다.
-   시스템은 구매된 티켓의 좌석을 사용 불가능으로 업데이트한다.

**특별 요구 사항**:

-   시스템은 결제 처리를 위해 외부 결제 게이트웨이(PaymentGateway)와 통신해야 한다.
-   시스템은 고객이 선택한 좌석이 동시에 다른 고객에게 판매되지 않도록 동시성 관리를 해야 한다.
-   모든 통신은 보안 프로토콜을 통해 이루어져야 한다.

**비즈니스 규칙**:

-   고객은 한 번에 최대 10장의 티켓을 구매할 수 있다.
-   상영 30분 전까지만 온라인으로 티켓을 구매할 수 있다.

```plantuml
@startuml
actor 고객 as customer
participant "Frontend" as front
participant "Backend" as back
participant "Movies" as movies
participant "Recommendation" as recommend
participant "Customers" as customsvc
participant "Theaters" as theaters
participant "Showtimes" as showtimes
participant "Tickets" as tickets

customer -> front : 극장 예매 시스템에 접속
front -> back : 추천 영화 목록 요청\nGET /movies/recommended?\ncustomerId={id}
back -> movies : findRecommendedMovies\n(customerId)
movies -> recommend : getMovieRecommendation\nForCustomer(customerId)
recommend -> customsvc : getViewingHistory(customerId)
recommend <-- customsvc : viewingHistory
recommend -> customsvc : getSearchHistory(customerId)
recommend <-- customsvc : searchHistory
recommend -> recommend : createMovieRecommendations\n(viewingHistory,searchHistory)
movies <-- recommend : movieRecommendations
movies -> movies : getMovies(recommendations)
back <-- movies : recommendedMovies[]
front <-- back : recommendedMovies[]
customer <-- front : 영화 목록 제공
customer -> front : 영화 선택
front -> back : 상영 극장 목록 요청\nGET /theaters/showing?\nmovieId={id}&nearby=37.123,128.678
back -> theaters: findNearByTheatersShowingMovie(movieId,location)
theaters -> showtimes: findTheaterIdsShowingMovie(movieId)
theaters <-- showtimes: theaterIds[]
theaters -> theaters: getTheatersSortedByDistance(theaterIds, location)
back <-- theaters: showingTheaters[](limited:5)
front <-- back : showingTheaters[]
customer <-- front : 상영 극장 목록 제공
customer -> front : 상영 극장 선택
front -> back : 상영일 목록 요청\nGET /showdates?movieId={id}&\ntheaterId={id})
back -> showtimes: findMovieShowdates(movieId,theaterId)
back <-- showtimes: movieShowdates[]
front <-- back : movieShowdates[]
customer <-- front : 상영일 목록 제공
customer -> front : 상영일 선택
front -> back : 상영 시간 목록 요청\nGET /showtimes?showdate={date}&\nmovieId={id}&theaterId={id}
back -> showtimes: findShowtimes(theaterId, movieId, showdate)
showtimes -> showtimes: findShowtimes(theaterId, movieId, showdate)\n: showtimes[]
showtimes -> tickets: getTicketSalesStatuses(showtimeIds[])
showtimes <-- tickets: ticketSalesStatuses[]
showtimes -> showtimes: createShowtimesWithSalesStatus\n(showtimes[], ticketSalesStatuses)
back <-- showtimes: showtimesWithSalesStatus[]
front <-- back : showtimesWithSalesStatus[]
customer <-- front : 상영일의 상세 정보 제공
customer -> front : 상영 시간 선택
front -> back : 상영 시간의 좌석 정보 요청\nGET /theaters/{theaterId}/\nshowing-seatmap?showtimeId={id}
back -> theaters : getShowingSeatmap(showtimeId)
theaters -> tickets : getTickets(showtimeId)
theaters <-- tickets : tickets[]
theaters -> theaters : generateShowingSeatmap(tickets)
back <-- theaters: showingSeatmap
front <-- back : showingSeatmap
customer <-- front : 선택 가능한 좌석 정보 제공
customer -> front : 좌석 선택
front -> back : 선택한 좌석 예약 및 가격 계산\nPOST /orders
front <-- back : order
customer <-- front : 결제화면(선택한 좌석과 총 가격 정보 제공)
customer -> front : 결제 정보 입력 및 구매 확정
front -> back : 결제 처리 요청 (POST /payments)
front <-- back : paymentResult(success)
customer <-- front : 티켓 구매 결과(성공)
customer <-- back : 전자 티켓 이메일 발송
@enduml

```

```ts
const customers = new CustomersService()
const recommendation = new RecommendationService(customersService)
const movies = new MoviesService(recommendationService)
```

```plantuml
@startuml
actor 고객 as customer
participant "Frontend" as front
participant "Backend" as back
participant "Movies" as movies
participant "Customers" as customsvc
participant "Tickets" as tickets

customer -> front : 극장 선택
front -> back : 고객이 관람한 영화 목록 요청\nGET /customers/{customerId}/history
back -> customsvc : getWatchingHistory\n(customerId)
customsvc -> tickets : getBuyingHistory\n(customerId)
customsvc <-- tickets : boughtTickets[]
customsvc -> customsvc : movieIds = boughtTickets[].movieId
customsvc -> movies : getMovies(movieIds)
customsvc <-- movies : movies
back <-- customsvc : movies
front <-- back : 고객이 관람한 영화 목록
customer <-- front : 영화 목록 제공
@enduml

```

순환 종속성 문제가 있음

```ts
const recommendation = new RecommendationService(customers)
const movies = new MoviesService(recommendation)
const tickets = new TicketsService()
const customers = new CustomersService(tickets, movies)
```

해결#1

```ts
const tickets = new TicketsService()
const recommendationmoviescustomers = new MoviesCustomersRecommendationService(recommendation)
```

해결#2

```ts
const recommendation = new RecommendationService()
const movies = new MoviesService(recommendation)
const tickets = new TicketsService()
const customers = new CustomersService(tickets, movies)

// 억지스러운 코드
recommendation.customers = customers

// 이런 부작용이 생긴다
if (this.customers) {
    console.log('')
}
```

상영 서비스, 통계 서비스를 분리하면 편하다. 그런데 이것을 movies, theaters에 통합하면 덩치가 커지고 종속성이 높아진다.
코드는 물 흐르듯이 자연스러워야 하고 이치에 맞아야 한다.
자연스러우면 기억할 필요가 없다. 부자연스러운 부분은 기억을 해야 한다. 코드가 복잡해지는 것이다.
