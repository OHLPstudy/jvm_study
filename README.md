# JVM 메모리 & GC 타임라인 학습 로드맵

> **최종 목표**
> 문제 있는 Java 애플리케이션에서 **Heap Dump + GC 로그를 기반으로 메모리 트러블슈팅을 수행할 수 있는 능력 확보**

---

## 🧭 학습 방식 안내 (타임라인 방식)

이 로드맵은 **"왜 이런 문제가 생겼을까?" → "그걸 해결하려다 무엇이 생겼을까?"** 흐름으로 학습합니다.

* 암기 ❌
* 역사·문제·의사결정 맥락 이해 ⭕
* 모든 학습은 **최종적으로 Heap Dump 해석에 연결**

---

## ⏳ Timeline 0 — 문제 해결자의 관점부터 갖기

### 이 단계의 질문

* 운영 중인 Java 서비스는 **어디서 죽는가?**
* 왜 메모리가 충분해 보여도 OOM이 나는가?

### 실무에서 만나는 증상

* `OutOfMemoryError: Java heap space`
* `OutOfMemoryError: Metaspace`
* Full GC 반복
* GC Pause Time 증가

📌 **핵심 인식**

> Heap Dump는 "원인"이 아니라 "결과"다. → 원인은 JVM 구조와 객체 생명주기에 있다.

---

## ⏳ Timeline 1 — JVM이 왜 만들어졌는가

### 당시의 근본 문제

* C/C++ 수동 메모리 관리

  * memory leak
  * dangling pointer
* 플랫폼 종속성
* 대규모 서비스 안정성 부족

### JVM의 목표

* Write Once, Run Anywhere
* 안전한 실행 환경
* 자동 메모리 관리

📌 **Dump 분석과의 연결**

> JVM은 메모리를 대신 관리한다 → 우리는 그 "관리 결과"를 해석해야 한다.

---

## ⏳ Timeline 2 — 초기 JVM 메모리 구조의 탄생

### Runtime Data Areas

```
Method Area (→ Metaspace)
Heap
 ├─ Young
 └─ Old
Thread Stack
Native Area
```

### 왜 이렇게 나뉘었는가?

* Stack: 빠른 실행, 스레드 독립성
* Heap: 객체 공유
* Method Area: 클래스 메타데이터

📌 **Dump 분석 핵심 포인트**

* Heap Dump는 Heap만 보여준다
* Metaspace OOM은 Heap Dump로 해결 안 됨

---

## ⏳ Timeline 3 — 객체는 어떻게 태어나고 죽는가

### 객체 생명주기

```
new → Eden
    → Survivor
    → Old Generation
    → GC 대상
```

### GC Root 개념의 등장

GC는 **Root에서 도달 가능한 객체만 생존**

**주요 GC Root**

* Thread Stack Local Variable
* Static Field
* JNI Reference
* System ClassLoader

📌 **Dump 분석의 본질**

> "이 객체를 누가 붙잡고 있는가?"

---

## ⏳ Timeline 4 — GC가 왜 필요했는가

### GC 이전의 세계

* 개발자가 직접 free
* 대규모 시스템에서 치명적 장애

### GC의 1차 목표

* 메모리 안전성

### GC의 2차 문제

* Stop-The-World
* 성능 저하

📌 GC는 편의성을 얻고 예측 불가능성을 얻었다

---

## ⏳ Timeline 5 — 가장 오래된 GC 알고리즘

### Mark & Sweep

* 살아있는 객체 마킹
* 나머지 제거

**문제점**

* STW
* 메모리 단편화

### Mark & Compact

* 단편화 해결
* STW 증가

📌 모든 GC는 Trade-off의 역사

---

## ⏳ Timeline 6 — Generational GC의 탄생

### 경험적 가설

> 대부분의 객체는 금방 죽는다

### Heap 구조 변화

```
Young Generation
 ├─ Eden
 ├─ S0
 └─ S1

Old Generation
```

📌 Dump 분석 시 Old Gen에 남아있는 객체가 핵심

---

## ⏳ Timeline 7 — 고전 GC들과 운영 문제

### Serial GC

* 단일 스레드
* 긴 STW

### Parallel GC

* Throughput 중심
* 여전히 STW

### CMS GC

* Concurrent Mark
* Fragmentation
* Floating Garbage

📌 "메모리가 남아있는데 OOM" → CMS 의심

---

## ⏳ Timeline 8 — Heap Dump의 구조 이해

### Heap Dump에 포함되는 것

* 객체
* 참조 그래프
* 클래스 정보

### 핵심 개념

| 개념            | 의미               |
| ------------- | ---------------- |
| Shallow Size  | 객체 자체 크기         |
| Retained Size | 제거 시 해제되는 전체 메모리 |

📌 Retained Size가 큰 객체가 1차 용의자

---

## ⏳ Timeline 9 — GC 로그와 Dump 연결

### GC 로그는 타임라인

* 메모리 증가 시점
* Promotion 실패
* Full GC 원인

### Heap Dump는 스냅샷

* 특정 시점의 메모리 상태

📌 반드시 함께 분석

---

## ⏳ Timeline 10 — 메모리 누수 패턴 학습

### 반드시 경험해야 할 누수 유형

* static Collection
* ThreadLocal 미정리
* Listener / Callback 누수
* ClassLoader Leak

📌 Dump에서 항상 반복되는 패턴

---

## ⏳ Timeline 11 — 실전 트러블슈팅 루틴 (최종 단계)

### 1. 증상 확인

* OOM 로그
* GC 로그

### 2. Heap Dump 확보

```bash
jmap -dump:live,format=b,file=heap.hprof <pid>
```

### 3. Dump 분석

* Dominator Tree
* Top Retained Objects
* GC Root Path

### 4. 원인 분류 & 코드 연결

---

## 🎯 최종 도달 수준

> ❌ "GC가 느린 것 같다"
>
> ⭕ "ThreadLocal에 의해 참조되는 HashMap이 Old Gen에 잔존하여 회수되지 않는다"

---

## 📌 추천 도구

* Eclipse MAT (필수)
* jmap / jcmd
* GCViewer
* VisualVM (보조)

---

## ✅ 이 로드맵 사용법

* 각 Timeline마다 **의문점 리스트 작성**
* Dump 분석 시 해당 Timeline으로 되돌아가기
* 이해 → 확인 → 재현의 반복

---

> 이 문서는 **JVM 메모리 문제를 "해석"할 수 있는 엔지니어**를 목표로 한다.
