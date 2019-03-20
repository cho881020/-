# 통합근무 관련 이슈

## 전제사항

- 통합근무 관련된 api 작업은 별개의 브런치에서 진행합니다.
  - 브런치 이름은 topic/intergrated-orderwork
- 관련 api의 주소는 전부 /v3/ 으로 시작합니다.
  - 기존의 이름에서 /v3/을 붙이거나, /v2/의 숫자를 바꾸거나 하는 식으로 진행합니다.
  - 이전 api와의 호환을 위해 (업데이트 고려) 새 api에 작업합니다.
- 리프트 예약 / 진행 관련
  - order_work_lift_process를 기반으로 움직인다. 

## 예약 ~ 근무 상세

1. 예약현황

   1. 요구사항

      - 날짜별로 예약을 한 아파트/업무를 목록으로 보여 줍니다.

      - ~~~sql
        SELECT DISTINCT ap_id, user_id, reservation_date, work_type FROM reservations ORDER BY created_at DESC;
        ~~~

      - 진행중인 솔트 업무는 reservations.py 에서 contract를 확인했을때 내가 잡은거라면 is_processing = True 로 담아서 보내주시고, 잡은게 없다면 False로 담아주세요

        - 진행중인 리프트 업무?
          - contract -> lift_prcoess 목록을 돌아보다가, 내가 진행중인게 있다면  is_processing = True

      - 수락가능?

        - SORT
          - order_work => contract가 null 이면 수락가능. is_contractable = True
        - LIFT
          - 솔트는 끝났는데, contract_id가 없는 order_work_lift_process 가 있다면 True
          - 어렵다. 같이 해야할듯.

      - 기존에는 예약 목록을 먼저 보여줬는데, 아파트 별로 가져오는 방식으로 해야할듯 합니다.

        - 가능하면 모델 에서 처리하는 식으로..!

   2. Endpoint

      - 현재 : /my_reservation
      - 변경 : /v3/my_reservation 834

   3. 파라미터

      - 유져토큰 - 헤더
      - 날짜(date) - query

   4. 응답형태

      ~~~json
      {
        "reservations":[
          {
            "apartname":"갈매 5단지 와이시티",
            "work_type":"SORT",
            "is_processing":true,
            "is_contractable":false
          },
          {
            "apartname":"갈매 5단지 와이시티",
            "work_type":"LIFT",
            "is_processing":true,
            "is_contractable":true
          }
        ]
      }
      ~~~

      

2. 근무 상세

   1. 요구사항

      - 지금은 하나의 order_work에 대해서 상세정보를 보여주지만,

      - 아파트의 정보에 수락한 업무목록이 달려있는 형식을 자세히 표현하는 방향이어야 합니다.

      - ~~~json
        {
            "아파트":{
                "이름":"갈매 5단지 와이시티",
                "업무목록":[
                   	{
                      	"택배사":"CJ",
                      	"업무종류":"SORT"
                  	},
                    {
                      	"택배사":"CJ",
                      	"업무종류":"SORT"
                  	}
                ]
            }
        }
        ~~~

   2. Endpoint

      - 현재 : /v2/order/order_work
      - 변경 : /v3/aprtment/today_orders - GET

   3. 파라미터

      - 유저토큰 - Header
      - 어느 아파트 - apartment_id
      - 어느날짜 - date
        - null일 경우 오늘날짜로

   4. 동작 내용

      - 유져 / 아파트 / 날짜 => contract 조회.
        - 내가 수락한 업무 목록을 Array로
      - 결과 : 내가 수락한 order_works가 array로 들어가게됨.
        - 오더워크의 부가정보는 예약현황과 같은 방식으로.

## Sort 근무 관련

1. 동 목록 화면 - 근무현황

   1. 요구사항

      - 아파트의 의 동 목록을 보여주자
      - 배송 수량을 보여주자
        - 모든 order_work에서 수행된 박스 카운트의 합
      - 완료 보고 여부도.
        - 모든 order_work가 전부 완료되었을때만 된걸로.
        - 매번 모든 order_work에 대해 다 검사해야한다.
          - 왜냐면 일하던 중간에 다른 아파트를 다시 콜을 받을 수도 있으니까!
      - 여러개의 order_work를 일괄처리.
        - 씨리얼 발급 절차처럼 파라미터를 order_work_ids 로 (배열로)
   2. 엔드포인트

      - 현재 : /v2/order/order_work/process
      - 변경 : /v3/apartment/sort_order_process - GET
   3. 파라미터
      - order_work_ids 로 (배열로)
   4. 응답 형테

   ~~~json
   {
       "동목록":[
           {
               "동이름":"1동",
          		"수량":100,
               "보고여부":true/false
           }
       ]
   }
   ~~~

2. 바코드 등록

   1. 요구사항
      - 바코드 등록은 당장은 변할게 없다.
      - 어차피 박스를 생성하는거고, 생성시에 order_id를 같이 던져주니까.
      - 다만 클라이언트에서 어떤 order인지 구별해서 그 아이디를 보내줘야할 필요가있다.
        - 송장번호에 따른 회사 구별 양식을 **이지훈팀장님** 에게 받아야한다.

3. 쏠트 동별 완료 보고

   1. 요구사항
      - 진행중인 모든 order_work 들에 대해 전부 완료처리를 한다.
        - 수량이 없는 경우 전부가 갯수가 0개일때로 처리하면 된다.
        - 수량이 있는경우 각각의 order_work/동 에 따라 상태를 다르게 처리.
          - 201동에 대해 배송을 하는데
            - 롯데 : 100개
            - 한진 : 30개
            - CJ : 0개
            - 이런 경우도 있을 수 있으니까.

   2. 엔드포인트
      - 현재 : /v2/order/order_work/process
      - 변경 : /v3/apartment/sort_order_process - POST

   3. 파라미터

      - order_work_ids : 어느 업무들을 처리하는지 (배열로)
        - 하나씩 기록하던걸  for문으로 등록하는 방식이면 충분할 듯.
      - dong_id : 어느 동에 대한 처리 할듯.

   4. 응답 형태

      ~~~json
      {
      	'code': 200,
      	'message': 'SORT 동별 완료보고 성공',
      	'data': {
              'order_works': [
                  {
                      "업무정보":"업무1번"
                  },
                  {
                      "업무정보":"업무2번"
                  }
              ]
      	}
      }
      ~~~

      