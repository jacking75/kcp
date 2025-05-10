# Chapter 1: KCP 핵심 동작 엔진 (주요 API 및 업데이트 루프)


KCP 프로토콜을 사용한 통신은 마치 자동차를 운전하는 것과 비슷합니다. 데이터를 보내고 싶을 때는 가속 페달을 밟고(`ikcp_send`), 상대방이 보낸 데이터를 받을 때는 계기판을 확인하듯 데이터를 가져옵니다(`ikcp_recv`). 외부에서 자동차(KCP)로 전달되는 정보(예: 다른 차의 접근 알림, 도로 상태)는 `ikcp_input`을 통해 처리됩니다. 그리고 가장 중요한 것은 자동차의 엔진처럼 주기적으로 `ikcp_update` 함수를 호출하여 KCP 내부 상태를 점검하고, 필요한 경우 실제로 바퀴를 굴려 데이터를 네트워크로 내보내는(`ikcp_flush`) 것입니다.

이번 장에서는 KCP 통신을 실제로 구동하고 제어하는 이 핵심 부품들, 즉 주요 API 함수들과 업데이트 루프에 대해 알아보겠습니다. 이 함수들이 어떻게 상호작용하며 안정적인 데이터 전송을 가능하게 하는지 이해하는 것이 KCP를 효과적으로 사용하는 첫걸음입니다.

## KCP 통신의 기본 흐름: 메시지 주고받기

가장 기본적인 사용 사례는 한쪽에서 메시지를 보내고 다른 쪽에서 그 메시지를 받는 것입니다. 예를 들어, "안녕하세요 KCP!"라는 메시지를 보내고, 상대방이 "반갑습니다!"라고 답장하는 상황을 생각해 봅시다. 이 간단한 통신을 위해 KCP는 여러 핵심 함수들을 조화롭게 사용합니다.

## 주요 KCP API 함수 살펴보기

KCP의 핵심 기능은 몇 가지 중요한 API 함수들을 통해 제공됩니다. 이 함수들은 KCP 엔진의 주요 부품과 같습니다.

### 1. `ikcp_send()`: 데이터 전송 요청 (가속 페달)

애플리케이션이 KCP를 통해 데이터를 보내고 싶을 때 사용하는 함수입니다.

```c
// ikcp.h
int ikcp_send(ikcpcb *kcp, const char *buffer, int len);
```

*   `kcp`: [KCP 연결 제어기 (ikcpcb)](02_kcp_연결_제어기__ikcpcb__.md)의 포인터입니다. 어떤 KCP 연결을 사용할지 지정합니다.
*   `buffer`: 보낼 데이터가 담긴 버퍼입니다.
*   `len`: 보낼 데이터의 길이입니다.

`ikcp_send`를 호출하면 데이터는 즉시 네트워크로 전송되는 것이 아니라, KCP 내부의 "보내기 큐(send queue)"에 추가됩니다. 이는 마치 편지를 우체통에 넣는 것과 같습니다. 실제 발송은 나중에 `ikcp_update` (내부적으로 `ikcp_flush` 호출)가 처리합니다.

**예시:**

```c
// "안녕하세요 KCP!" 메시지 전송 요청
const char *message = "안녕하세요 KCP!";
int length = strlen(message);
ikcp_send(kcp_object, message, length);
// 아직 실제 전송은 안 됨, KCP의 보내기 큐에만 추가됨
```
이 코드는 "안녕하세요 KCP!"라는 문자열을 `kcp_object`가 관리하는 KCP 연결을 통해 보내도록 요청합니다.

### 2. `ikcp_recv()`: 수신된 데이터 가져오기 (목적지 도착 알림)

상대방으로부터 도착하여 KCP 내부에 잘 정돈된 데이터를 애플리케이션이 가져갈 때 사용하는 함수입니다.

```c
// ikcp.h
int ikcp_recv(ikcpcb *kcp, char *buffer, int len);
```

*   `kcp`: [KCP 연결 제어기 (ikcpcb)](02_kcp_연결_제어기__ikcpcb__.md)의 포인터입니다.
*   `buffer`: 수신된 데이터를 저장할 버퍼입니다.
*   `len`: 버퍼의 최대 크기입니다.

`ikcp_recv`는 KCP의 "수신 큐(receive queue)"에서 완성된 데이터를 가져옵니다. 반환값은 읽어온 데이터의 크기이며, 읽을 데이터가 없으면 음수 값을 반환합니다.

**예시:**

```c
char receive_buffer[1024];
int received_bytes = ikcp_recv(kcp_object, receive_buffer, sizeof(receive_buffer));

if (received_bytes > 0) {
    receive_buffer[received_bytes] = '\0'; // 문자열로 사용하기 위해 널 종료
    printf("수신된 메시지: %s\n", receive_buffer);
} else if (received_bytes == -1) {
    // 아직 수신된 데이터가 없음
}
```
이 코드는 `kcp_object`로부터 데이터를 수신하려고 시도하고, 성공하면 메시지를 출력합니다.

### 3. `ikcp_input()`: 하위 계층으로부터 패킷 입력 (외부 정보 수신)

KCP는 UDP 같은 하위 프로토콜 위에서 동작합니다. UDP 소켓 등으로부터 실제 네트워크 패킷(바이트 덩어리)을 수신했을 때, 이 데이터를 KCP에게 전달하는 함수입니다.

```c
// ikcp.h
int ikcp_input(ikcpcb *kcp, const char *data, long size);
```

*   `kcp`: [KCP 연결 제어기 (ikcpcb)](02_kcp_연결_제어기__ikcpcb__.md)의 포인터입니다.
*   `data`: 수신된 원시 패킷 데이터입니다.
*   `size`: 수신된 원시 패킷 데이터의 크기입니다.

`ikcp_input`은 KCP 프로토콜 헤더를 분석하여 이 패킷이 실제 데이터인지, ACK인지, 아니면 다른 제어 정보인지를 판단하고 내부 상태를 업데이트합니다. 데이터 패킷이라면 수신 버퍼로 옮겨 정렬하고, ACK 패킷이라면 해당 데이터가 잘 전달되었음을 기록합니다.

**예시:**

```c
// UDP 소켓으로부터 raw_packet_data (크기: packet_length)를 수신했다고 가정
// 이 데이터를 KCP에게 전달
ikcp_input(kcp_object, raw_packet_data, packet_length);
// KCP는 이 패킷을 분석하여 내부 상태를 업데이트함
```
애플리케이션은 UDP 소켓 등에서 데이터를 읽은 후, 그 내용을 그대로 `ikcp_input`에 넣어주어야 합니다.

### 4. `ikcp_update()`: KCP 상태 업데이트 (엔진 구동)

KCP의 심장과 같은 함수입니다. 주기적으로 호출되어야 KCP가 제대로 동작합니다.

```c
// ikcp.h
void ikcp_update(ikcpcb *kcp, IUINT32 current);
```

*   `kcp`: [KCP 연결 제어기 (ikcpcb)](02_kcp_연결_제어기__ikcpcb__.md)의 포인터입니다.
*   `current`: 현재 시간 (밀리초 단위). KCP는 타이머 기반 동작을 위해 현재 시간을 알아야 합니다.

`ikcp_update`는 다음과 같은 중요한 작업들을 수행합니다:
1.  지정된 시간이 되면 `ikcp_flush`를 호출하여 쌓여있는 ACK나 데이터 패킷을 실제로 네트워크로 내보냅니다.
2.  타이머를 확인하여 재전송해야 할 패킷이 있는지 검사하고, 있다면 재전송합니다 ([KCP 신뢰성 보장 장치 (ACK 및 재전송)](04_kcp_신뢰성_보장_장치__ack_및_재전송__.md)에서 자세히 다룹니다).
3.  윈도우 크기 업데이트 등 기타 내부 상태를 갱신합니다.

이 함수는 보통 10ms ~ 100ms 간격으로 애플리케이션의 메인 루프나 별도의 타이머 스레드에서 호출됩니다.

**예시:**

```c
// 현재 시간을 밀리초 단위로 가져오는 함수 (시스템마다 다름)
IUINT32 current_time_ms = get_current_milliseconds();
ikcp_update(kcp_object, current_time_ms);
// KCP 내부 상태가 갱신되고, 필요시 패킷이 전송됨
```

### 5. `ikcp_flush()`: 쌓인 데이터 실제 전송 (바퀴 굴리기)

`ikcp_flush`는 KCP 내부에 쌓여있는 패킷들(데이터, ACK, 윈도우 업데이트 등)을 실제로 네트워크를 통해 전송하는 역할을 합니다. 대부분의 경우 사용자가 직접 `ikcp_flush`를 호출할 필요는 없습니다. `ikcp_update`가 내부적으로 적절한 시점에 `ikcp_flush`를 호출하기 때문입니다.

```c
// ikcp.h
void ikcp_flush(ikcpcb *kcp);
```

`ikcp_flush`가 호출되면, KCP는 내부적으로 `output` 콜백 함수(KCP 객체 생성 시 설정)를 사용하여 패킷을 하위 계층(예: UDP 소켓)으로 보냅니다.

## 간단한 통신 시나리오

Alice가 Bob에게 "안녕하세요!" 메시지를 보내는 과정을 KCP API와 함께 살펴보겠습니다.

1.  **Alice 측 설정:**
    *   KCP 객체(`kcp_A`) 생성 ([KCP 연결 제어기 (ikcpcb)](02_kcp_연결_제어기__ikcpcb__.md) 참고).
    *   UDP 패킷을 실제로 보낼 `output_callback_A` 함수를 `ikcp_setoutput(kcp_A, output_callback_A)`로 설정.

2.  **Alice 메시지 전송:**
    *   Alice: `ikcp_send(kcp_A, "안녕하세요!", strlen("안녕하세요!"));`
        *   메시지는 `kcp_A`의 보내기 큐에 들어갑니다.

3.  **Alice 업데이트 루프:**
    *   Alice는 주기적으로 `current_time = get_current_milliseconds(); ikcp_update(kcp_A, current_time);`를 호출합니다.
    *   `ikcp_update`는 때가 되면 `ikcp_flush(kcp_A)`를 호출합니다.
    *   `ikcp_flush`는 "안녕하세요!" 메시지가 담긴 [KCP 데이터 조각 (IKCPSEG)](03_kcp_데이터_조각__ikcpseg__.md)을 만들고, `output_callback_A`를 통해 UDP 패킷으로 Bob에게 전송합니다.

4.  **Bob 측 설정:**
    *   KCP 객체(`kcp_B`) 생성.
    *   UDP 패킷을 실제로 보낼 `output_callback_B` 함수를 `ikcp_setoutput(kcp_B, output_callback_B)`로 설정.

5.  **Bob 패킷 수신 및 처리:**
    *   Bob의 애플리케이션이 UDP 소켓에서 Alice가 보낸 패킷(`raw_data`, `raw_data_len`)을 수신합니다.
    *   Bob: `ikcp_input(kcp_B, raw_data, raw_data_len);`
        *   KCP는 패킷을 분석하여 "안녕하세요!" 데이터를 `kcp_B`의 수신 버퍼에 넣고, 이 데이터에 대한 ACK를 보낼 준비를 합니다.

6.  **Bob 업데이트 루프 및 데이터 수신:**
    *   Bob은 주기적으로 `current_time = get_current_milliseconds(); ikcp_update(kcp_B, current_time);`를 호출합니다.
    *   `ikcp_update`는 때가 되면 `ikcp_flush(kcp_B)`를 호출하여 Alice에게 ACK 패킷을 보냅니다.
    *   Bob: `char buffer[100]; int len = ikcp_recv(kcp_B, buffer, 100);`
        *   `len > 0` 이면, `buffer`에 "안녕하세요!" 메시지가 담겨 있습니다.

7.  **Alice ACK 수신:**
    *   Alice의 애플리케이션이 UDP 소켓에서 Bob이 보낸 ACK 패킷을 수신합니다.
    *   Alice: `ikcp_input(kcp_A, ack_packet_data, ack_packet_len);`
        *   `kcp_A`는 ACK를 처리하여 "안녕하세요!" 메시지가 Bob에게 잘 전달되었음을 인지합니다.

이것이 KCP 핵심 API들을 사용한 기본적인 데이터 송수신 흐름입니다.

## KCP 엔진 내부 동작 엿보기

KCP 엔진이 주기적으로 `ikcp_update`를 통해 어떻게 동작하는지 간단한 흐름도로 표현하면 다음과 같습니다.

```mermaid
graph TD
    A[사용자 App: ikcp_update(kcp, 현재시간) 호출] --> B{내부 타이머: flush 시간인가?};
    B -- 예 --> C[KCP 내부: ikcp_flush(kcp) 호출];
    C --> D[ACK 목록 확인 및 ACK 패킷 생성];
    D --> E[output 콜백: 생성된 ACK 패킷 UDP로 전송];
    C --> F[보내기 큐(snd_queue)에서 보내기 버퍼(snd_buf)로 데이터 이동];
    C --> G[보내기 버퍼(snd_buf) 확인];
    G --> H{전송/재전송할 데이터 패킷 있는가?};
    H -- 예 --> I[데이터 패킷 생성];
    I --> J[output 콜백: 생성된 데이터 패킷 UDP로 전송];
    H -- 아니오 --> K[윈도우 프로브 등 기타 제어 패킷 확인/생성/전송];
    J --> K;
    E --> K;
    B -- 아니오 --> L[다음 ikcp_update 호출까지 대기];
    K --> L;
```

위 흐름도에서 볼 수 있듯이 `ikcp_update`는 KCP의 여러 내부 상태를 점검하고 필요한 동작을 수행하는 중심축입니다. 사용자가 `ikcp_send`로 보낸 데이터는 보내기 큐에 대기하다가, `ikcp_update`가 적절한 시점에 이 데이터들을 보내기 버퍼로 옮기고, `ikcp_flush`를 통해 실제 패킷으로 만들어 `output` 콜백 함수로 전달하여 네트워크로 내보냅니다. 마찬가지로 `ikcp_input`으로 들어온 패킷에 대한 ACK도 `ikcp_flush`를 통해 전송됩니다.

### `ikcp_update`와 `ikcp_flush`의 관계

`ikcp_update`는 KCP의 전체적인 상태를 관리하고 타이밍을 제어하는 "관리자" 역할입니다. 이 관리자는 특정 조건(예: 설정된 `interval` 경과)이 만족되면 `ikcp_flush`라는 "실무자"를 호출하여 실제 패킷 전송 작업을 지시합니다.

`ikcp_update`의 일부 코드입니다 (`ikcp.c`):
```c
// ikcp.c
void ikcp_update(ikcpcb *kcp, IUINT32 current)
{
	IINT32 slap;

	kcp->current = current; // 현재 시간 업데이트

	if (kcp->updated == 0) { // 처음 호출된 경우
		kcp->updated = 1;
		kcp->ts_flush = kcp->current; // 다음 flush 시간 초기화
	}

	slap = _itimediff(kcp->current, kcp->ts_flush); // 다음 flush 시간까지 남은 시간

    // 다음 flush 시간이 너무 멀거나 가깝지 않도록 조정
	if (slap >= 10000 || slap < -10000) {
		kcp->ts_flush = kcp->current;
		slap = 0;
	}

	if (slap >= 0) { // flush 할 시간이 되었거나 지났다면
		kcp->ts_flush += kcp->interval; // 다음 flush 시간 설정
		if (_itimediff(kcp->current, kcp->ts_flush) >= 0) {
			kcp->ts_flush = kcp->current + kcp->interval;
		}
		ikcp_flush(kcp); // 실제 패킷 전송 및 내부 처리 함수 호출
	}
}
```
여기서 `kcp->interval`은 `ikcp_update`가 `ikcp_flush`를 호출하는 기본 주기를 결정합니다 (기본값 100ms). `ikcp_nodelay` 함수를 통해 이 간격을 조절할 수 있습니다.

`ikcp_flush` 함수는 크게 세 가지 종류의 패킷을 처리하여 `output` 콜백으로 넘깁니다:
1.  **ACK 패킷**: `ikcp_input`을 통해 데이터 패킷을 수신했을 때, 해당 데이터에 대한 확인 응답(ACK)을 `acklist`에 저장해 둡니다. `ikcp_flush`는 이 `acklist`를 보고 ACK 패킷을 만들어 보냅니다.
2.  **데이터 패킷**: `ikcp_send`로 요청된 데이터는 `snd_queue`를 거쳐 `snd_buf`로 이동합니다. `ikcp_flush`는 `snd_buf`에 있는 [KCP 데이터 조각 (IKCPSEG)](03_kcp_데이터_조각__ikcpseg__.md)들을 패킷으로 만들어 보냅니다. 재전송이 필요한 패킷도 여기서 처리됩니다.
3.  **제어 패킷**: 윈도우 크기를 묻거나(`IKCP_CMD_WASK`), 알리는(`IKCP_CMD_WINS`) 등의 제어 패킷도 필요에 따라 `ikcp_flush`에서 전송됩니다.

`ikcp_flush`의 간략화된 구조 (`ikcp.c`):
```c
// ikcp.c
void ikcp_flush(ikcpcb *kcp)
{
	// ... (변수 초기화) ...
    // 'ikcp_update'가 호출되지 않았다면 아무것도 안 함
	if (kcp->updated == 0) return;

	// 1. ACK 패킷 전송 준비
	// kcp->acklist에 있는 ACK들을 패킷으로 만듦
	for (i = 0; i < kcp->ackcount; i++) {
		// ... ikcp_ack_get으로 ACK 정보 (sn, ts) 가져오기 ...
		// ... 버퍼에 ACK 세그먼트 인코딩 ...
		if (/* 버퍼가 꽉 찼거나 MTU 초과 직전 */) {
			ikcp_output(kcp, buffer, (int)(ptr - buffer)); // output 콜백으로 실제 전송
			// ptr = buffer; // 버퍼 포인터 리셋
		}
	}
	// kcp->ackcount = 0; // ACK 목록 비우기

	// 2. 윈도우 프로브/알림 패킷 전송 준비 (필요한 경우)
	// if (kcp->probe & IKCP_ASK_SEND) { /* WASK 패킷 인코딩 및 전송 준비 */ }
	// if (kcp->probe & IKCP_ASK_TELL) { /* WINS 패킷 인코딩 및 전송 준비 */ }

	// 3. 데이터 패킷 전송 준비
    // snd_queue에 있는 데이터들을 snd_buf로 옮기고, 전송 정보(sn, ts 등) 설정
	while (/* 보낼 공간이 있고, snd_queue에 데이터가 있다면 */) {
		// ... snd_queue에서 newseg 가져오기 ...
		// ... newseg를 snd_buf로 이동 ...
		// ... newseg의 정보(sn, ts, rto 등) 설정 ...
	}

	// snd_buf에 있는 데이터 패킷들을 순회하며 전송 또는 재전송 결정
	for (p = kcp->snd_buf.next; p != &kcp->snd_buf; p = p->next) {
		IKCPSEG *segment = iqueue_entry(p, IKCPSEG, node);
		if (/* 처음 보내는 패킷이거나, 재전송 시간이 되었거나, 빠른 재전송 조건 만족 */) {
			// ... segment의 ts, wnd, una 등 업데이트 ...
			// ... 버퍼에 segment 인코딩 및 데이터 복사 ...
			if (/* 버퍼가 꽉 찼거나 MTU 초과 직전 */) {
				ikcp_output(kcp, buffer, (int)(ptr - buffer)); // output 콜백으로 실제 전송
                // ptr = buffer; // 버퍼 포인터 리셋
			}
		}
	}

	// 4. 버퍼에 남아있는 모든 패킷 전송
	if ((int)(ptr - buffer) > 0) {
		ikcp_output(kcp, buffer, (int)(ptr - buffer));
	}

	// ... (혼잡 제어 관련 ssthresh, cwnd 업데이트) ...
}
```

### `ikcp_input`의 역할

`ikcp_input`은 UDP 등 하위 프로토콜로부터 받은 원시(raw) 데이터를 KCP가 이해할 수 있도록 해석하는 "수문장"입니다.

`ikcp_input`의 간략화된 처리 과정 (`ikcp.c`):
```c
// ikcp.c
int ikcp_input(ikcpcb *kcp, const char *data, long size)
{
	// ... (수신 로그 기록) ...
	if (data == NULL || (int)size < (int)IKCP_OVERHEAD) return -1; // 최소 크기 검사

	while (1) { // 하나의 UDP 패킷 안에 여러 KCP 세그먼트가 있을 수 있음
		// ... (헤더 디코딩: conv, cmd, frg, wnd, ts, sn, una, len) ...
		if (conv != kcp->conv) return -1; // 다른 연결의 패킷이면 무시
		// ... (패킷 크기 유효성 검사) ...

		// 상대방 윈도우 크기(rmt_wnd) 업데이트
		// 받은 UNA(una)까지의 패킷은 상대방이 수신했으므로, 우리 쪽 snd_buf에서 제거 (ikcp_parse_una)

		if (cmd == IKCP_CMD_PUSH) { // 데이터 패킷인 경우
			if (/* 수신 윈도우 내에 있고, 아직 받지 않은 sn 이라면 */) {
				ikcp_ack_push(kcp, sn, ts); // 이 sn에 대한 ACK를 acklist에 추가 (나중에 ikcp_flush가 보냄)
				IKCPSEG *seg = ikcp_segment_new(kcp, len); // IKCPSEG 객체 생성
                // ... (seg에 정보 채우기: conv, cmd, frg, wnd, ts, sn, una, len) ...
				// ... (seg->data에 실제 데이터 복사) ...
				ikcp_parse_data(kcp, seg); // 수신 버퍼(rcv_buf)에 순서대로 정렬하여 삽입
			} else { // 이미 받았거나 윈도우 밖의 데이터면
                // ikcp_segment_delete(kcp, newseg); // (ikcp_parse_data 내부 또는 여기서 처리)
            }
		} else if (cmd == IKCP_CMD_ACK) { // ACK 패킷인 경우
			if (_itimediff(kcp->current, ts) >= 0) { // 유효한 ACK 시간이라면
				ikcp_update_ack(kcp, _itimediff(kcp->current, ts)); // RTT (왕복 시간) 업데이트
			}
			ikcp_parse_ack(kcp, sn); // sn에 해당하는 패킷이 ACK되었음을 처리 (snd_buf에서 제거 시도)
			ikcp_shrink_buf(kcp); // snd_buf 정리 (snd_una 업데이트)
            // ... (빠른 재전송 관련 처리: ikcp_parse_fastack) ...
		}
        // ... (IKCP_CMD_WASK, IKCP_CMD_WINS 등 다른 명령어 처리) ...

		data += len; // 다음 세그먼트 데이터로 포인터 이동
		size -= len; // 남은 크기 줄이기
        if (size < (int)IKCP_OVERHEAD) break; // 남은 데이터가 헤더보다 작으면 종료
	}
	return 0;
}
```
`ikcp_input`은 받은 패킷의 종류(`cmd`)에 따라 적절한 내부 함수(`ikcp_parse_data`, `ikcp_parse_ack` 등)를 호출하여 KCP의 상태를 변경합니다.

## 결론

이번 장에서는 KCP의 핵심 동작 엔진을 구성하는 주요 API 함수들(`ikcp_send`, `ikcp_recv`, `ikcp_input`, `ikcp_update`, `ikcp_flush`)의 역할과 기본적인 사용법에 대해 알아보았습니다. 이 함수들은 마치 자동차의 주요 부품들처럼 각자의 역할을 수행하며 서로 유기적으로 연결되어 KCP의 안정적이고 효율적인 통신을 가능하게 합니다.

*   `ikcp_send`로 데이터를 KCP에 전달하고,
*   `ikcp_input`으로 네트워크에서 수신한 원시 패킷을 KCP에 넣어주며,
*   주기적인 `ikcp_update` 호출을 통해 KCP 내부 상태를 갱신하고 실제 패킷(데이터, ACK 등)을 `ikcp_flush`를 통해 네트워크로 내보내고,
*   `ikcp_recv`로 KCP가 정렬해 놓은 데이터를 애플리케이션이 가져갑니다.

이러한 핵심 함수들의 상호작용을 이해하는 것은 KCP를 효과적으로 활용하기 위한 첫 단추입니다.

다음 장에서는 이 모든 KCP 동작의 중심이 되는 데이터 구조이자, 개별 KCP 연결의 상태를 관리하는 "제어기"인 [KCP 연결 제어기 (ikcpcb)](02_kcp_연결_제어기__ikcpcb__.md)에 대해 자세히 살펴보겠습니다.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)