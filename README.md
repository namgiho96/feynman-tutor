# 🎓 Feynman Tutor Agent

> **리처드 파인만의 교육 철학을 구현한 AI 튜터** — LangGraph 기반 단일 노트북 학습 에이전트

[![Python 3.13+](https://img.shields.io/badge/python-3.13+-blue.svg)](https://www.python.org/)
[![LangGraph 1.x](https://img.shields.io/badge/langgraph-1.x-orange.svg)](https://langchain-ai.github.io/langgraph/)
[![uv](https://img.shields.io/badge/pkg_manager-uv-purple.svg)](https://github.com/astral-sh/uv)

---

## 한 줄 요약

배우고 싶은 개념을 한 단어로 던지면, AI가 *5살에게 설명하듯* 비유 두 개로 풀어주고 → 4지선다 퀴즈 → 틀리면 *다른 각도/다른 비유*로 재설명. 정답 맞히거나 재시도 한도 도달 시 종료.

## 데모

```python
concept = '블록체인'
result = run_feynman_tutor(concept, max_retry=2)
```

**예상 흐름:**
1. AI: "블록체인은요, *공책 여러 권을 친구들이 나눠서 적는 거*예요. 누구 한 명이 거짓말로 고치려고 해도 다른 친구들 공책이랑 비교돼서 들켜요. 또 ___ 같아요."
2. 퀴즈 4지선다 → 사용자 입력 (1/2/3/4)
3. 정답 → 종료 / 오답 → "다르게 설명해볼게요. 이번엔 *레고 블록 끼우기*로 생각해볼까요..."

## 그래프 구조

```
                ┌─────────────────────────────────┐
                │     Feynman Tutor Graph         │
                ├─────────────────────────────────┤
START                                              
  │                                                
  ▼                                                
┌───────────────┐                                  
│ explain_node  │  ← 파인만 식 설명 (비유 ≥2, 5살 언어)  
└───────┬───────┘                                  
        │                                          
        ▼                                          
┌───────────────┐                                  
│   quiz_node   │  ← 4지선다 + input() 답변 평가      
└───────┬───────┘                                  
        │                                          
        ▼                                          
   ┌─────────┐                                     
   │ correct?│                                     
   └───┬─┬───┘                                     
   Yes │ │ No                                       
       │ ▼                                          
       │ ┌─────────────────┐                       
       │ │ re_explain_node │  ← 다른 비유/각도        
       │ └────────┬────────┘                       
       │          │                                
       │          ▼ (loop, max_retry=2)            
       │     [quiz_node]                            
       ▼                                            
      END                                           
```

**노드 3개:**
| Node | 역할 | 출력 |
|---|---|---|
| `explain_node` | 파인만 기법 설명 (비유 ≥2 + 한 문장 요약) | `state.explanation` |
| `quiz_node` | 4지선다 퀴즈 생성 + `input()` 답변 평가 | `state.is_correct`, `state.attempts++` |
| `re_explain_node` | *다른 비유, 다른 각도*로 재설명 | `state.explanation` (덮어쓰기) |

**라우팅:** `route_after_quiz(state)` — `is_correct=True OR attempts≥max_retry` → `END`, 아니면 `re_explain_node`.

## 빠른 시작

### 1. 의존성 설치

```bash
cd /Users/namgiho/PycharmProjects/feynman-tutor
uv sync
```

### 2. 환경변수 (.env)

```bash
echo "OPENAI_API_KEY=sk-..." > .env
```

### 3. Jupyter 실행

```bash
uv run jupyter lab feynman_tutor.ipynb
```

또는 VSCode/Cursor에서 직접 `feynman_tutor.ipynb` 열고 셀 순서대로 실행.

### 4. 데모 셀 (셀 23번)

```python
concept = '블록체인'   # ← 여기를 바꿔보세요
result = run_feynman_tutor(concept, max_retry=2)
```

추천 토픽: `'양자 얽힘'`, `'재귀 함수'`, `'미적분'`, `'자연 선택'`, `'엔트로피'`, `'기회비용'`

## 노트북 구성 (24 cells)

```
0  📝 헤더
1  📝 Step 1: 에이전트 설계 (이름·목적·페르소나)
2  📝 그래프 구조 다이어그램
3  📝 Step 2: 기초 구축 — 환경
4  💻 import + load_dotenv
5  📝 State 정의
6  💻 TutorState TypedDict (concept, explanation, attempts, is_correct, messages)
7  📝 LLM 설정
8  💻 ChatOpenAI(model='gpt-4o-mini')
9  📝 Node 1: explain_node 설명
10 💻 explain_node 구현
11 📝 Node 2: quiz_node 설명
12 💻 quiz_node 구현 (JSON 4지선다 + input())
13 📝 Node 3: re_explain_node 설명
14 💻 re_explain_node 구현
15 📝 조건부 엣지
16 💻 route_after_quiz
17 📝 그래프 조립
18 💻 build_tutor_graph + compile
19 📝 그래프 시각화
20 💻 draw_mermaid_png 출력
21 📝 데모 실행 안내
22 💻 run_feynman_tutor 함수
23 💻 ✏️ 데모 (concept 입력)
```

## 파인만 기법 적용 포인트

이 에이전트는 [파인만 학습기법](https://brunch.co.kr/@saetae/48) 4단계 중 **2단계 (Teach to a 12yo)** + **4단계 (Simplify & Refine)** 를 구현:

| Feynman 단계 | 노트북 구현 |
|---|---|
| 1. Choose & Study (지식 인출) | *생략* — 사용자가 토픽만 던짐 |
| **2. Teach (5살에게)** | ✅ `explain_node` system prompt: "5살 아이에게 설명하듯, 비유 2개 이상" |
| **3. Identify Gaps** | ✅ `quiz_node`: 오답 시 어디 막혔는지 파악 |
| **4. Simplify & Refine** | ✅ `re_explain_node`: 다른 비유로 다듬기 |

> *"전문용어의 남발은 자신의 이해 부족을 숨기는 것."* — 파인만

## 관련 프로젝트

이 프로젝트의 **production 확장판**: [namgiho96/tella](https://github.com/namgiho96/tella)

| | feynman-tutor (이 레포) | tella |
|---|---|---|
| 형태 | 단일 Jupyter 노트북 | 풀스택 (FastAPI + Next.js) |
| 노드 수 | 3개 (explain/quiz/re_explain) | 4개 active (intro/respond/quiz/brainstorm) + legacy |
| state | TypedDict (5 필드) | TypedDict (15+ 필드) |
| 메모리 | 없음 (단일 세션) | SQLite checkpointer (thread별 multi-turn) |
| 페르소나 | 친절한 5살 화법 선생님 | Richard Feynman 선생님 (P7 페르소나) |
| LLM provider | OpenAI 고정 | OpenAI/Anthropic 스위칭 (BYOK) |
| 적합 용도 | 학습/실습/프로토타입 | 데모데이/배포 |

이 노트북에서 핵심 메커니즘을 *체득*한 다음 tella 코드로 production 패턴 (SSE, 체크포인터, BYOK) 학습하면 좋음.

## 스택

```toml
[project]
name = "feynman-tutor"
requires-python = ">=3.13"
dependencies = [
    "langgraph>=1.1",
    "langchain>=1.2",
    "langchain-openai>=1.2",
    "python-dotenv>=1.2",
    "ipykernel>=7.2",
]
```

## 라이선스 / 제작

- 학습 목적 프로토타입. 자유롭게 fork / 수정 / 학습용으로 사용.
- 출처 영감: 노마드코더 AI Agents Masterclass `tutor-agent` 패턴 + Richard Feynman 학습기법.

## 다음 단계 (확장 아이디어)

- [ ] CLI 버전 (`__main__.py` + `run_feynman_tutor` 진입)
- [ ] `state.attempt_history`에 모든 비유 누적 → 마지막에 "어느 비유에서 가장 잘 이해됐어?" 물어보기
- [ ] `re_explain_node`에 *비유 다양성 강제* — 이전 비유 카테고리(요리/스포츠/일상물건) 피하기
- [ ] Web UI: tella 그대로 가져다 쓰기 (이 노트북이 *backend reasoning*, tella가 *delivery layer*)
