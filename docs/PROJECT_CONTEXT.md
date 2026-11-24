# TIME TO GO — 대기시간 맞춤형 스마트 주문 키오스크 (프로젝트 컨텍스트)

> 출처: 알고리즘 최종 정리 PDF(11.23), 5장 실습 매뉴얼, 주제계획발표 PPTX/대본, 실습 보드 사진.
> 기간: 약 2주. 보드: STM32_RTOS_EVAL (PS_STM32_RTOS_V10), μC/OS-II V2.8x, IAR EWARM 8.2+.

## 1. 확정 운영 규칙
| 항목 | 사양 |
|---|---|
| 주문 수량 | 손님 1명당 1잔 |
| 대기열 | 미완료 최대 10건(제조 중 1 + 대기 9), FIFO, 일반/시간제한 간 우선순위 없음 |
| 제조 | 직원 1명, 한 잔씩 순차 |
| 속도 | 가상 1분 = 실제 5초 (12배속, 시스템 틱은 그대로 두고 별도 가상시계) |
| 알림 | 문자 LCD `ORDER 003 READY` + LED + 부저, 알림 중에도 제조·타이머 계속 진행 |
| 메뉴 선택 시간 | 계산하지 않음. 고객 제한시간 카운트다운은 **주문 확정 이후** 시작 |

## 2. 입력/화면 매핑
- 숙련도(전원 ON 시): SW1/SW2 → OFF/OFF=하(1.3), 한쪽 ON=중(1.0), ON/ON=상(0.8)
- S19 일반 주문, S20 시간제한 주문
- GLCD 2×2 분할: 좌상단 카테고리명+픽셀 아이콘, 나머지 3칸 메뉴. S17 이전/S18 다음 페이지
  - 1p COFFEE: S3 아메리카노, S4 카페라떼, S5 바닐라라떼
  - 2p NON-COFFEE: S3 스무디, S4 에이드, S5 티
- 세부 종류 S15/S16: 커피류 ICE/HOT, 스무디 딸기/초코, 에이드 레몬/청포도, 티 캐모마일/페퍼민트
- 버튼은 1회 눌림만 유효(디바운스 → 처리 → 해제 확인). 초기화 대상은 입력 이벤트뿐, 주문 데이터는 유지

## 3. 시간 계산
- 제조시간 = 기계 작업 + 수작업 × 숙련도 계수

| 메뉴 | 기계 | 수작업 | 중 기준 |
|---|---|---|---|
| 아메리카노 | 0.5분 | 0.5분 | 1분 |
| 카페라떼 | 0.5분 | 1.5분 | 2분 |
| 바닐라라떼 | 0.5분 | 2분 | 2분 30초 |
| 스무디 | 1분 | 4분 | 5분 |
| 에이드 / 티 | 미정 | 미정 | 미정 |

- 예상 수령 = 현재 제조 **잔여**시간 + 앞선 대기 주문 제조시간 합 + 내 음료 제조시간
- 시간제한 판정: 예상 수령 ≤ 입력 최대 대기시간. 초과 메뉴는 숨기지 않고 비활성화, 전부 불가면 재입력 안내
- 확정 버튼 순간 재검사(수용량 + 시간제한) → 통과 시 주문번호 발급, 실패 시 경고
- 위험도: 여유 = 고객 남은 대기 가능시간 − 예상 완료까지 남은 시간
  - NORMAL > 2분, WARNING 0~2분, DANGER < 0 (단계가 바뀔 때만 부저)

## 4. 제조·상황 변경
- 타이머 0 → READY → 알림 Task에 전달 → GLCD 현황판에서 제거 → 다음 주문 즉시 시작
- 금지: Task_Kitchen 안에서 긴 지연으로 부저/LCD 대기
- UART 관리자 명령: `DELAY 60`(제조 중 음료 잔여시간에만 가상 60초 추가, 제조 중 없으면 무시),
  숙련도 변경(제조 중 음료 유지, 대기 주문부터 새 계수), 취소 승인(승인 시점에 WAITING일 때만), `STAT`

## 5. 응용 Task 10개
Task_Input, Task_Order, Task_Scheduler, Task_Estimator, Task_Kitchen, Task_Clock,
Task_Display, Task_Alert, Task_UART, Task_Monitor
(+ 라이브러리 내부: KeyScanTask, TmrTask, TaskStart, Idle, Stat)

RTOS 도구: MsgQ(입력/주문/완료 이벤트), Mutex(대기열·상태), Semaphore(동기화·GLCD), Event Flags(지연·완료·교대·화면 갱신), 타이머/지연.

코딩 우선순위: ① 키패드 선택 + GLCD 동적 표시 → ② 숙련도 제조시간 + FIFO → ③ 예상시간·시간제한·확정 재검사 → ④ 12배속 제조 + 비차단 알림

## 6. 보드/실습 API 메모 (5장 매뉴얼)
- MCU STM32F103VET6, SysTick 1ms (`SysTick_Configuration()`), 기본 스택 512B (`OS_TASK_DEF_STK_SIZE`)
- 초기화: `BSP_Init(); DelayMS(300); GLCD_init(); GLCD_clear(); LCD_initialize(); OSInit(); ... OSStart();`
- GLCD(PE12864, 128×64): `GLCD_xy(row 0~7, col 0~20); printf(...)` — 공유 자원이므로 세마포어로 보호
- 문자 LCD 16×2(I2C): `LCD_string(0x80, ...)` 1행, `LCD_string(0xC0, ...)` 2행
- LED: `LED_TOGGLE(0~3)` / 부저: `GPIO_SetBits/ResetBits(GPIOD, GPIO_Pin_15)`
- S19: `GPIOD Pin10` (0 = 눌림), SW1 토글: `GPIOC Pin6`
- 4×4 키(S3~S18): `KeyInit()`(OSInit 다음), `KeyHit()`, `KeyGetKey(to)`(스캔코드, 타임아웃 시 0xFF)
  - key.c는 2초 누르면 자동 반복 → 알고리즘 2와 충돌하므로 반복 끄거나 무시 필요
- UART1 9600bps: `CommInit()`, `CommGetChar(COMM1, to, &err)`, `CommPutChar(COMM1, c, to)`, ISR `USART1_IRQHandler`
- 메시지 큐: `OSQCreate(tbl, N)`, `OSQPost`, `OSQPend`, `OSQAccept`
- 뮤텍스: `OSMutexCreate(PIP 우선순위, &err)` — 비어 있는 우선순위를 PIP로 예약해야 함
- 이벤트 플래그: `OSFlagCreate`, `OSFlagPost(..., OS_FLAG_SET)`, `OSFlagPend(..., OS_FLAG_WAIT_SET_ANY + OS_FLAG_CONSUME, to, &err)`
- 타이머(ESBB TMR.c, 0.1초 단위): `TmrInit/TmrCfgFnct/TmrSetT/TmrSetMST/TmrStart/TmrStop/TmrChk`
  - 실습 예제의 타임아웃 콜백 안 `OSTimeDly` 사용은 TmrTask를 막음 → 이 프로젝트에선 피할 것
- 통계: `OSStatInit()`, `OSCPUUsage`, `OSCtxSwCtr`, `OSTaskCtr`, `OSTaskStkChk()`(CreateExt + STK_CHK 필요)

## 7. 미결/확인 필요 사항
1. 에이드·티 제조시간, 세부 종류/HOT·ICE별 시간 차이
2. 대기 가능시간 입력 방법: 보드에 별도 숫자 키패드 없음 → S3~S18 중 남는 키(S6~S14 등)로 숫자/± 입력, 확정·취소 키 배정 필요
3. 주문 확정 키, 취소 요청 시 주문번호 입력 키 배정
4. 실제 GLCD 비트맵 출력 함수 유무(픽셀 아이콘 구현 방식)
5. key.c 스캔코드 ↔ S3~S18 실제 매핑은 보드에서 확인
