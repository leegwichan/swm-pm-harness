# Java Virtual Thread — 개념 정리 및 디베이트 트래커 도입 근거

- 작성일: 2026-05-20
- 작성 목적: 본 프로젝트(**디베이트 트래커 / Debate Tracker**, 팀 KFC)에 **Java Virtual Thread(가상 스레드, Project Loom)** 를 도입해야 하는 이유를 기술적·운영적·취업 임팩트 관점에서 정리.
- 전제 문서:
  - `02-main.md` (2026-05-15 기획서 초안)
  - `synthesis.md` (2026-05-15 종합 리포트)
  - `F1-realtime-summary-nfr-v0.2.md` (F1 비기능 요구사항 v0.2)
  - `26-05-20-plan.md` (최신 기획서, **권위본**)

---

## 1. Virtual Thread란 무엇인가

**Virtual Thread**는 JVM이 직접 스케줄링하는 경량 스레드로, OS 스레드(=Platform Thread)와 1:1로 묶이지 않는다. JEP 444로 표준화되었고 **Java 21(2023-09 LTS)** 에서 정식 도입되었다.

### 핵심 정의

- `java.lang.Thread` 의 서브타입이다. 기존 API와 그대로 호환된다 (`Thread.ofVirtual().start(...)`, `Executors.newVirtualThreadPerTaskExecutor()`).
- 하나의 **Platform Thread(=Carrier Thread)** 위에서 **다수의 Virtual Thread**가 mount/unmount 되며 실행된다.
- I/O 블로킹 호출(`socket.read`, `JDBC`, `HttpClient.send`, `Thread.sleep` 등)이 발생하면 **JVM이 자동으로 unmount**시켜 carrier thread를 다른 virtual thread가 사용하게 한다. 블로킹은 "OS 스레드 점유"가 아니라 "JVM 컨티뉴에이션의 일시 정지"가 된다.
- Carrier thread pool은 기본적으로 `ForkJoinPool#commonPool` 과 별개로 운영되며, 워커 수는 기본 `Runtime.getRuntime().availableProcessors()`.

### 등장 배경 — Thread-per-Request 모델의 한계

전통적인 Spring MVC(Tomcat) Thread-per-Request 모델은:

- 요청 1개 = OS 스레드 1개 점유. OS 스레드는 메모리(스택 ~1MB)와 컨텍스트 스위칭 비용이 크다.
- 외부 I/O를 기다리는 동안 스레드가 **블로킹 상태로 그대로 잡혀 있다.** CPU는 놀지만 스레드 풀은 고갈된다.
- 동시 처리량을 늘리려면 스레드 풀을 키우거나, 비동기/리액티브(WebFlux, Reactor)로 전환해야 했다.
- 리액티브는 학습 곡선이 가파르고 디버깅·스택 트레이스가 어려우며 JDBC 등 블로킹 라이브러리와 정합이 나쁘다.

Virtual Thread는 **"동기 코드 스타일을 유지하면서 비동기 수준의 처리량"** 을 목표로 한다. `webClient.get().block()` 같은 패턴이 더 이상 안티패턴이 아니다.

---

## 2. 동작 원리 (간단)

```
┌─────────────────────────────────────────────────────────┐
│   Virtual Thread 1   Virtual Thread 2   Virtual Thread N│  (수만~수십만 개)
│        │                  │                  │           │
│        └──── mount ───────┘                  │           │
│                  │                           │           │
│        Carrier Thread (= Platform Thread, OS Thread)     │  (CPU 코어 수만큼)
└─────────────────────────────────────────────────────────┘
```

- **Mount**: virtual thread가 carrier 위에 올라가서 실제 코드 실행.
- **Unmount**: 블로킹 I/O 호출 시 virtual thread의 스택을 힙으로 옮기고 carrier에서 내려옴. Carrier는 즉시 다른 virtual thread를 mount.
- **Park/Unpark**: I/O 응답이 도착하면 JVM이 해당 virtual thread를 다시 스케줄링.

이 메커니즘 덕분에 OS 스레드 ≪ Virtual Thread 수가 되어, 수만 개 동시 작업을 적은 OS 스레드로 처리할 수 있다.

### Pinning(고정) — 가장 큰 함정

다음 경우에는 virtual thread가 carrier에서 **unmount되지 못하고 고정(pinning)** 된다.

1. **`synchronized` 블록 안의 블로킹 호출**: Java 21 시점 한계. (JEP 491에서 해소 예정 — 확인 필요)
2. **JNI 네이티브 코드** 안의 블로킹.

Pinning이 빈번하면 virtual thread의 효과가 사라지고 carrier 고갈로 성능이 오히려 나빠진다. **대안: `synchronized` → `ReentrantLock` 전환.**

JFR 이벤트 `jdk.VirtualThreadPinned` 로 측정 가능 (`-Djdk.tracePinnedThreads=full`).

### ThreadLocal 영향

- Virtual Thread는 ThreadLocal을 지원하지만, 수십만 개 생성 시 ThreadLocal 누수가 심각한 메모리 문제로 번질 수 있다.
- 권장: `ScopedValue`(JEP 446/464, preview/incubator — 확인 필요) 또는 ThreadLocal 사용량 최소화.

---

## 3. Platform Thread vs Virtual Thread 비교

| 항목 | Platform Thread | Virtual Thread |
|------|-----------------|----------------|
| 1개당 메모리 | 스택 ~1MB | 초기 수백 바이트~수 KB (스택은 힙에서 동적) |
| 생성 비용 | 비싸다 (OS 콜) | 매우 싸다 (객체 생성 수준) |
| 동시 개수 한계 | 수천 (스레드 풀로 제한) | **수십만~수백만** |
| 스케줄러 | OS 커널 | JVM (ForkJoinPool 기반) |
| 블로킹 I/O 비용 | 스레드 점유 (낭비) | unmount → carrier 해방 |
| 디버깅/스택 트레이스 | 익숙함 | 동기 코드 그대로, 익숙함 |
| 적합 워크로드 | CPU-bound | **I/O-bound (네트워크·DB·외부 API)** |
| 부적합 워크로드 | — | CPU-bound (이득 없음), pinning 빈번 코드 |

---

## 4. 디베이트 트래커 워크로드 분석 — 왜 I/O-bound인가

`26-05-20-plan.md` 의 시스템 구성도·F1 데이터 파이프라인과 `F1-realtime-summary-nfr-v0.2.md` 의 QAS를 종합하면, BFF가 처리하는 단일 토론 세션은 다음 외부 I/O를 **동시에 다수** 발생시킨다.

| I/O 종류 | 호출 빈도 | 블로킹 특성 |
|----------|----------|------------|
| WebSocket 클라이언트 ↔ BFF | 세션당 viewer 수 × 양방향 | 장시간 keep-alive, 짧은 메시지 |
| BFF → STT API (AWS Transcribe / Azure Speech) | 발화 청크당 1회, 1~3초 청크 단위 | 응답 2~5초 (QAS-PE-01) |
| BFF → Redis Pub/Sub | 발화·쟁점 갱신마다 | 짧지만 빈번 (QAS-PE-04) |
| BFF → Postgres (JPA) | 발화 INSERT, 보정 UPDATE, 쟁점 트리 저장 | DB 응답 대기 |
| BFF → S3 | 음성 청크 영속, PDF 리포트(F3) | 큰 파일 업로드 |
| AI Worker → LLM API (OpenAI/Anthropic) | 보정 10~20초 주기, 쟁점 추출 10~20초 주기 (QAS-PE-02/03) | 응답 수 초~수십 초 |
| BFF → SQS | 작업 enqueue | 짧음 |

이 모든 호출은 **블로킹 I/O**다. Thread-per-Request 모델로는 다음과 같은 처리량 목표(QAS)에서 OS 스레드가 부족해진다.

- **QAS-PE-04**: 세션 1개당 viewer 30명 동시 구독. 동시 운영 세션이 K개라면 활성 WebSocket = 30K개. 각 WebSocket의 메시지 펌핑·세션 상태 유지에 스레드가 묶이면 평범한 m5.large 인스턴스에서도 수천 viewer를 못 받친다.
- **QAS-RE-01/02**: 재접속·중간 합류 시 즉시 직전 발화 N개 캐시 응답 → 동시 합류 burst 처리 필요.
- **AI Worker**: LLM 호출이 분당 수십~수백 회 발생하면서 각각 수 초~수십 초 블로킹. Worker가 동기 호출 시 풀이 빠르게 고갈된다.
- **F3 산출물(Sprint 5-6) 부담**: 카드뉴스·세특 초안 생성 시 1건당 LLM 호출 + S3 업로드 + PDF 렌더가 묶여 발생. 학생 30명 단위 세특 일괄 생성은 동시 호출 수가 burst로 튄다.

**결론: 디베이트 트래커의 BFF·AI Worker는 정확히 Virtual Thread가 가장 큰 이득을 주는 워크로드 형태다.**

---

## 5. 본 프로젝트에 Virtual Thread를 도입해야 하는 이유

### 5.1 워크로드 정합 — 가장 직접적인 이유

위 4절 분석대로 BFF·AI Worker 모두 I/O-bound이며 외부 호출 응답 대기 시간이 전체 처리 시간의 대부분이다. Virtual Thread는 carrier OS 스레드를 점유하지 않고 unmount되므로, **동일 하드웨어로 처리 가능한 동시 세션·동시 viewer 수가 큰 폭으로 증가**한다.

### 5.2 NFR 달성 마진 확보

`F1-realtime-summary-nfr-v0.2.md` 에 명시된 NFR 중 Virtual Thread가 마진을 확보해 주는 항목:

| QAS ID | NFR | Virtual Thread 도입 효과 |
|--------|-----|--------------------------|
| QAS-PE-01 | STT 응답 P95 ≤ 5초 | 블로킹 호출 동안 carrier 해방 → 동시 처리 가능 청크 수 증가 |
| QAS-PE-04 | viewer 30명, broadcast 평균 ≤ 300ms, 손실률 0% | WebSocket 1개당 스레드 1개를 부담 없이 할당 가능 |
| QAS-RE-01 | 재접속 후 손실 0, 재동기화 ≤ 3초 | 재접속 burst를 큐잉 없이 흡수 |
| QAS-RE-02 | 중간 합류 초기 화면 ≤ 2초 | 캐시 조회·DB 조회 병렬화 시 스레드 부담 없음 |
| QAS-PE-05 | LLM 토큰 비용 상한 | (간접) 동시 호출 패턴 단순화로 비용 분석·캐싱 도입이 쉬워짐 |

### 5.3 시스템 아키텍처와의 정합 — Reactive 회피의 정당화

`26-05-20-plan.md` §개발환경에 명시된 스택은 **Java & Kotlin / Spring**이다. JPA(블로킹) 기반 Spring 스택에서 처리량을 늘리려고 WebFlux/Reactor로 전환하면 JPA를 못 쓰고(R2DBC 학습), 디버깅이 어려워지며, 신입 개발자에게는 학습 리스크가 크다. **Virtual Thread는 "Spring MVC + JPA를 유지하면서 처리량을 얻는" 유일한 표준 경로**다.

이는 `synthesis.md` 의 핵심 권고 — "줄임 = Sprint 3, 깊이 = 백엔드 1개" — 와도 정합한다. 리액티브 전환은 깊이가 아니라 "전선 확장"이며, Virtual Thread는 깊이로 환전 가능한 결정이다.

### 5.4 Kotlin Coroutines와의 관계

`26-05-20-plan.md` 에서 Kotlin은 **"Null Safety가 필요한 부분에 한하여 도입"** 으로 한정되어 있다. 주언어는 Java다. 이 전제 위에서 Kotlin Coroutines와 Virtual Thread의 관계는 다음과 같이 정리된다.

- **Kotlin Coroutines**: `suspend` 함수와 `Dispatchers` 기반의 협조적(cooperative) 비동기. 컴파일러가 컨티뉴에이션을 변환하며, Java 8+ 환경에서도 동작한다.
- **Virtual Thread**: JVM 런타임이 블로킹 호출을 자동으로 unmount. 일반 동기 코드 그대로 사용 가능.

**팀의 전제(Java 주언어, Kotlin은 부분 도입)에서는 Virtual Thread가 답이다.** 이유:

1. 백엔드 코드 대부분이 Java로 작성되므로 Coroutines를 강제할 수 없다.
2. Coroutines를 쓰려면 RestTemplate·JdbcTemplate 같은 블로킹 API를 모두 `suspend` 친화 API로 갈아끼워야 한다(학습·전환 비용 큼).
3. Kotlin Coroutines는 **Dispatcher가 Virtual Thread Executor를 쓰도록 설정 가능** — 두 패러다임이 충돌하지 않고 결합된다(`Dispatchers.IO` 대안으로 `Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()` 패턴 — 확인 필요).

결론: Kotlin이 부분 도입되더라도 Virtual Thread 결정은 영향받지 않는다.

### 5.5 백엔드 면접 카드(취업 임팩트)로의 환전

`synthesis.md` 의 백엔드 면접 카드 후보:

- **옵션 A**: WebSocket partial transcript 정합성 + 끊김·재연결 시 상태 이어쓰기
- **옵션 B**: Bedrock + LLM 호출 큐 + 백프레셔 + 토큰 비용/지연 제어 + 캐싱

**Virtual Thread는 옵션 A·B 둘 다와 자연스럽게 결합**한다.

- 옵션 A에서: 동시 WebSocket 세션 수·재접속 burst 처리량 측정 → "Platform Thread 풀 N개 한계에서 Virtual Thread로 전환 시 Y배 처리" 같은 정량 서사 가능.
- 옵션 B에서: LLM 호출 큐의 worker가 Virtual Thread 기반일 때 동시성 제어·백프레셔 설계가 단순해진다. Worker 풀 사이즈를 동적으로 조절하지 않아도 자연스럽게 동시 호출 수가 늘어남.

이 카드는 본 프로젝트의 BE Lead 2인(김OO — Backend/Product Lead, 이OO — Backend/System Architecture Lead)이 모두 취업 카드로 환전 가능하다. 특히 이OO은 `26-05-20-plan.md` §팀 구성에서 **"실시간 통신(WebSocket)·동시성·AWS 인프라"** 의 최종 결정권자로 명시되어 있어 Virtual Thread 의사결정 오너로 가장 자연스럽다.

면접관에게 30분 떠들 서사:

> "초기에는 Tomcat 기본 풀(200)로 시작했는데 viewer 동시 접속 N에서 응답 지연 P95가 X초로 무너졌습니다. WebFlux 전환은 JPA·디버깅 비용이 커서 배제했고, Java 21 Virtual Thread + `spring.threads.virtual.enabled=true` 로 전환했습니다. 전환 후 동일 워크로드에서 P95가 X→Y로 개선되었고, `synchronized`로 묶인 캐시 락 1곳에서 pinning이 발생해 JFR `jdk.VirtualThreadPinned` 로 추적, `ReentrantLock` 으로 교체했습니다."

이런 서사는 `peer-competitor` 가 강조한 "측정 수치 + 실패 로그" 패턴에 정확히 해당한다. **단순 "Virtual Thread 켜봤어요" 가 아니라, pinning 진단·교체 의사결정까지 가져가야 면접 카드로 작동한다.**

### 5.6 STT/LLM 어댑터 계층 단순화

QAS-CO-02(벤더 교체 가능성)는 STT/LLM 어댑터를 모듈화한다. Virtual Thread 환경에서는 어댑터 내부에서 `HttpClient.send(...)` 동기 호출을 그대로 사용하면 된다. `CompletableFuture` 체인이나 Reactor `Mono` 변환 없이 동기 코드 스타일로 단순하게 유지 가능 → 코드 가독성·디버깅 모두 개선.

---

## 6. 도입 시 고려사항

### 6.1 Java 버전

- `02-main.md` 초안에는 Java 17이 명시되어 있으나, `26-05-20-plan.md` 권위본은 **"Java & Kotlin / Spring"** 으로만 표기되어 버전 미명시. **Java 21+ 로 확정 필요**.
- Spring Boot도 3.2+ 권장 (Virtual Thread 자동 설정 지원: `spring.threads.virtual.enabled=true`).
- 영향 범위: Gradle/Maven 버전 설정, CI 이미지(`temurin:21`), AWS 런타임(Corretto 21).
- Kotlin이 부분 도입되므로 Kotlin도 JVM target 21로 빌드 필요.

### 6.2 Pinning 회피

- 코드베이스에서 `synchronized` 사용 지점을 사전 식별해 `ReentrantLock` 으로 전환.
- Kotlin 코드는 `@Synchronized` 어노테이션·`synchronized {}` 블록이 동일하게 pinning을 유발한다. Kotlin 영역에서도 `Mutex`(Coroutines) 또는 `ReentrantLock` 사용 권장.
- 외부 라이브러리(JDBC 드라이버·HTTP 클라이언트 등)의 pinning 이력 확인 — Postgres JDBC, Lettuce(Redis), AWS SDK v2 등 주요 라이브러리 호환성은 **확인 필요**(2026-05 기준 대부분 지원).
- 운영 환경에 `-Djdk.tracePinnedThreads=short` 옵션을 dev/stg에만 켜서 pinning 발생 지점을 사전 탐지.

### 6.3 ThreadLocal·SecurityContext

- Spring Security `SecurityContextHolder`(기본 ThreadLocal 전략), MDC 로깅 컨텍스트가 Virtual Thread에서 누수되지 않도록 사용 패턴 점검.

### 6.4 측정(필수)

면접 카드로 환전하려면 **측정 수치가 필수**다. 다음 항목을 캘린더에 박을 것:

- Platform Thread 기본 풀(예: Tomcat 200) vs Virtual Thread 활성화 상태에서:
  - 동시 viewer N명일 때 broadcast P50/P95/P99 (QAS-PE-04와 연결)
  - 동시 세션 K개일 때 STT 응답 P95 (QAS-PE-01)
  - JVM heap·GC 시간·thread count
- 측정 도구: `k6` (WebSocket 부하), `JFR` (pinning), `Micrometer + Prometheus + Grafana` (BE 관측 스택 — 26-05-20-plan.md §개발환경에 이미 명시됨). Pinpoint로 분산 트레이싱 시 Virtual Thread 호환성도 함께 검증.

### 6.5 운영 함정

- Virtual Thread는 무제한으로 생성 가능하지만, **외부 자원(DB connection pool, 외부 API rate limit)은 유한**하다. Bulkhead 패턴(Resilience4j Bulkhead)이나 세마포어로 동시 호출 수 상한을 두어야 한다.
- 무제한 동시성 ≠ 무제한 처리량. "BFF 안에서 무한 생성"이 외부 DB·API에 부하 폭탄으로 전달될 수 있음을 인지.
- LLM API rate limit(QAS-PE-05 토큰 비용 상한)은 Virtual Thread 도입과 무관하게 별도 제어가 필요하다.

---

## 7. 도입 단계 제안 (디베이트 트래커 일정에 맞춤)

`26-05-20-plan.md` §추진 일정 (기획 5월 / 분석 6월 / 설계 7월 / 개발 8~10월 / 테스트 10~11월 / 완성 11월) 및 Sprint 1-2(메인대시보드) → 3-4(사후 분석) → 5-6(산출물 자동화) 구조와 정렬해서 점진 도입을 제안한다.

| 단계 | 시점 | 작업 |
|------|------|------|
| **사전 결정** | 5~7월 (기획·분석·설계) | Java 21·Spring Boot 3.2+ 결정. Gradle·CI 이미지 업그레이드 사전 검토. 설계 단계에서 `synchronized` 사용 정책·Bulkhead 패턴 정책 수립. |
| **1차 도입** | Sprint 1-2 (8월, 메인대시보드) | `spring.threads.virtual.enabled=true` 1줄 활성화. 실시간 속기록·쟁점 요약 기능 회귀 테스트. WebSocket broadcast 1차 부하 측정(Platform vs Virtual). |
| **2차 보강** | Sprint 3-4 (9월, 사후 분석) | LLM 호출 큐(AI Worker)에 Virtual Thread 적용. F2 팀별·개인별 분석 동시 호출 burst를 Bulkhead로 보강. JFR pinning 분석 1차. |
| **3차 확장** | Sprint 5-6 (10월, 산출물 자동화) | F3 카드뉴스·세특 초안 일괄 생성(학생 30명 burst) 시나리오에 Virtual Thread 효과 측정. `synchronized → ReentrantLock` 교체 의사결정 마무리. |
| **검증·서사화** | 테스트 (10~11월) | 토론 동아리 사용성 테스트와 함께 부하 시나리오 재측정. 면접용 1페이지 답변 초안 작성(BE Lead 2인 각자 1장). |
| **최종 데모** | 완성 (11월 데모데이) | 동시 viewer 시연 안정성 확보. 데모 폴백 시나리오(QAS-RE-05)와 결합. |

---

## 8. 한계 및 추측 표기

- 본 문서는 **Java 21 GA 시점(2023-09)** 의 일반 공개 정보를 기반으로 작성되었다. JEP 491(`synchronized` pinning 해소)의 정식 GA 여부, 2026-05 시점 라이브러리(Postgres JDBC·Lettuce·AWS SDK 등) 최신 pinning 호환성은 **확인 필요**.
- Kotlin Coroutines의 `Dispatchers` 를 Virtual Thread Executor로 교체하는 패턴이 Coroutines 공식 API로 안정 지원되는지는 **확인 필요**.
- "Virtual Thread 도입 후 처리량 X배 개선" 같은 수치는 본 문서에 포함하지 않았다. **실제 측정 후 면접 카드와 기획서 차별점 한 줄에 반영**해야 한다 (`synthesis.md` 의 "차별점 한 줄에 숫자 2개" 요구와 정합).
- Spring Boot 3.2+의 Virtual Thread 자동 설정 범위(Tomcat request executor·`@Async` executor·`TaskScheduler` 등)는 버전별로 다르다. **도입 시 공식 릴리스 노트 재확인**.

---

## 9. 한 줄 요약

> 디베이트 트래커의 BFF·AI Worker는 외부 STT/LLM/Redis/DB I/O로 시간의 대부분을 소비하는 전형적 I/O-bound 워크로드다. Java 21 Virtual Thread는 WebFlux 전환의 학습·디버깅 부담 없이 동일 하드웨어에서 동시 viewer·동시 세션·F3 산출물 burst 처리량을 끌어올리는 **표준이자 거의 유일한 경로**이며, pinning 진단·`synchronized` 교체 의사결정까지 묶어내면 BE Lead 2인(김OO·이OO)의 백엔드 신입 면접 카드(옵션 A/B)와 정확히 결합된다.
