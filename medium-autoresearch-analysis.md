# Autoresearch: AI가 `program.md`를 읽고 밤새 `train.py`를 바꾸는 연구실

부제: Karpathy의 `autoresearch`와 `nanochat`를 통해 본 "AI-native research loop"의 설계 원리

## TL;DR

`autoresearch`는 거대한 에이전트 프레임워크가 아니다. 오히려 반대다. Karpathy는 연구 자동화를 위해 시스템을 더 복잡하게 만들지 않고, 연구 루프를 극단적으로 작게 압축했다. 고정된 데이터 준비(`prepare.py`), 단 하나의 수정 대상 파일(`train.py`), 그리고 에이전트의 행동 규약을 담은 자연어 운영체제(`program.md`)만 남겨 두었다.

이 설계는 중요하다. 이유는 "에이전트가 무엇이든 할 수 있게 만드는 것"보다 "에이전트가 잘할 수 있는 문제만 남기는 것"이 실제 성능 개선으로 더 빨리 이어지기 때문이다. 실제로 `nanochat` 저장소에는 `autoresearch`로 약 2일간 자율 탐색해 얻은 개선이 반영되었고, README leaderboard 기준 GPT-2 급 성능 도달 시간이 2.02시간에서 1.80시간으로 단축되었다. 같은 기준에서 보면 약 11% 개선이다.

이 글은 `autoresearch` 저장소, `nanochat` 저장소, round 1 결과를 알린 X 포스트를 함께 읽고, 이 프로젝트가 왜 단순한 장난감이 아니라 "AI가 연구 워크플로 자체를 먹어 들어가는 방식"의 초기 프로토타입인지 코드 수준까지 내려가 분석한 글이다.

## 분석 기준과 소스

이 문서는 2026-03-14 기준 아래 버전을 로컬로 클론해 읽고 작성했다.

- `autoresearch` HEAD: `c2450ad`
- `nanochat` HEAD: `f068604`
- `nanochat`의 autoresearch round 1 결과 반영 커밋: `6ed7d1d`

주요 소스:

- X 포스트: [karpathy/status/2031135152349524125](https://x.com/karpathy/status/2031135152349524125)
- 텍스트 미러: [xcancel.com/karpathy/status/2031135152349524125](https://xcancel.com/karpathy/status/2031135152349524125)
- `autoresearch` 저장소: [github.com/karpathy/autoresearch](https://github.com/karpathy/autoresearch)
- `nanochat` 저장소: [github.com/karpathy/nanochat](https://github.com/karpathy/nanochat)
- round 1 결과 반영 커밋: [nanochat commit 6ed7d1d](https://github.com/karpathy/nanochat/commit/6ed7d1d82cee16c2e26f45d559ad3338447a6c1b)

## 왜 이 프로젝트가 중요한가

LLM 에이전트 데모의 대부분은 "브라우저 조작", "툴 호출", "코드 수정" 같은 능력 시연에 머문다. 하지만 `autoresearch`는 다르다. 이 프로젝트는 에이전트를 연구자처럼 보이게 하는 것이 아니라, 연구라는 작업을 에이전트가 최적화하기 쉬운 형태로 재설계한다.

핵심은 세 가지다.

1. 연구 목표를 하나의 숫자로 고정한다. 여기서는 `val_bpb`다. 이는 validation 텍스트를 바이트 기준으로 얼마나 잘 예측하는지 나타내는 `bits per byte`이며, 낮을수록 좋다.
2. 실험 시간을 하나의 고정 예산으로 묶는다. 여기서는 벽시계 기준 5분이다.
3. 수정 가능한 코드를 하나의 파일로 제한한다. 여기서는 `train.py`다.

이 세 조건이 만들어내는 효과는 강력하다. 에이전트는 더 이상 "연구를 어떻게 할까?"를 고민하지 않는다. 대신 "이 5분 안에 더 낮은 `val_bpb`를 만들기 위해 무엇을 바꿀까?"라는 좁고 빠른 피드백 루프 안에서 탐색한다.

즉, `autoresearch`는 범용 지능을 연구에 투입한 사례라기보다, 연구 문제를 에이전트 친화적으로 재구성한 사례다. 그리고 바로 이 점이 실전적이다.

## 소스에서 확인되는 사실

먼저 사실관계를 분리해 보자.

- `autoresearch` README는 이 프로젝트를 "작지만 실제적인 LLM 학습 셋업을 에이전트에게 주고, 밤새 자율 실험하게 하는 구조"라고 설명한다.
- 같은 README는 이 저장소가 `nanochat`의 단순화된 single-GPU 구현이라고 명시한다.
- `program.md`는 에이전트가 따라야 하는 실험 운영 규약을 제공한다. 브랜치 생성, baseline 측정, git commit, 5분 학습, 성능 비교, keep/discard, `results.tsv` 기록까지 모두 자연어로 정의한다.
- `nanochat` README는 leaderboard에 `autoresearch round 1` 항목을 추가하며, GPT-2 수준 도달 시간이 2.02시간에서 1.80시간으로, 같은 기준에서 약 11% 개선되었다고 적고 있다.
- `nanochat`의 `6ed7d1d` 커밋 메시지는 더 직접적이다. "이 개선들은 Claude가 약 2일 동안 `autoresearch`를 사용해 자율적으로 개발했으며, Karpathy 본인은 손대지 않았다"라고 밝힌다.
- 다만 2026년 3월 9일 X 답글에서 Karpathy는, 그 주말에 돌린 earlier version 중 일부는 아직 완전히 time-controlled가 아니었고 loss와 training time을 함께 보고 reject했다고 설명했다. 즉, 현재 공개 저장소의 "고정 5분 budget" 설계와 round 1 당시의 내부 실험 버전을 완전히 동일시하면 안 된다.

이 여섯 문장을 연결하면 `autoresearch`의 본질은 꽤 명확하다. 이것은 "에이전트를 위한 훈련 코드 편집 샌드박스"이며, 동시에 "자율 연구 프로토콜"이다.

## Karpathy의 의도는 무엇이었나

여기서부터는 README, `program.md`, 그리고 `nanochat`의 round 1 반영 커밋을 바탕으로 한 해석이다. 다만 이 해석은 꽤 강한 근거를 가진다.

내가 보기에는 Karpathy의 의도는 최소 네 층위로 나뉜다.

### 1. 연구자의 역할을 코드 작성자에서 연구 조직 설계자로 이동시키는 것

`autoresearch` README는 사람이 Python 파일을 직접 만지는 대신 `program.md`를 프로그래밍한다고 설명한다. 이 표현은 상징적이다. 그는 연구자의 핵심 노동을 "매 실험마다 직접 코드를 고치고 돌리는 일"에서 "에이전트가 따를 연구 프로토콜을 설계하는 일"로 옮기고 있다.

즉, 사람은 이제 모델 내부를 매번 손으로 고치는 존재가 아니라, 연구실 운영 체계를 설계하는 존재가 된다.

### 2. 에이전트가 잘할 수 있도록 연구 문제 자체를 재설계하는 것

Karpathy는 에이전트에게 거대한 자유도를 주지 않았다. 수정 파일은 사실상 `train.py` 하나, metric은 `val_bpb` 하나, 예산은 5분 하나다. 이건 "AI 연구자를 만들었다"기보다 "에이전트가 높은 확률로 성공할 수 있게 연구 루프를 재구성했다"에 가깝다.

이 설계는 매우 실용적이다. agentic system이 실패하는 가장 흔한 이유는 능력이 부족해서가 아니라, 문제 정의가 너무 넓고 상태 공간이 너무 커서다. `autoresearch`는 바로 그 점을 잘라냈다.

### 3. `nanochat` 같은 실제 상위 프로젝트의 개선 속도를 높이는 것

이 프로젝트는 개념 시연에 머물지 않았다. `nanochat` README는 `autoresearch round 1`이 leaderboard 개선으로 이어졌다고 적고 있고, `6ed7d1d` 커밋 메시지는 그 개선이 약 2일간의 자율 탐색에서 나왔다고 밝힌다. 즉, 의도는 "에이전트가 연구 흉내를 내는가"를 보는 것이 아니라, "실제 연구 생산성을 올릴 수 있는가"를 보는 데 있었다.

### 4. 인간을 짧은 피드백 루프의 병목에서 빼는 것

`program.md`는 baseline을 재고, 실험하고, commit하고, 성능을 읽고, keep/discard를 결정하고, 다시 반복하라고 지시한다. 그리고 시작한 뒤에는 멈추지 말라고 못 박는다. 이건 우연한 문체라기보다, 2026년 3월 9일 X 포스트에서 Karpathy가 그린 "agents collaborate"와 "humans (optionally) contribute on the edges"라는 그림과 맞닿아 있다. 따라서 이 프로젝트는 인간을 짧은 실험 루프의 중심에서 조금씩 바깥으로 밀어내는 방향의 실험으로 읽는 편이 더 정확하다.

한 줄로 압축하면 이렇다. Karpathy의 의도는 "AI가 연구를 대신한다"를 선언하는 것보다, "연구 루프를 AI가 계속 돌릴 수 있는 인터페이스로 다시 설계한다"에 가까웠다.

## `autoresearch`의 핵심 아이디어

이 프로젝트의 진짜 발명은 모델 구조 그 자체가 아니다. 진짜 발명은 연구 행위를 세 개의 층위로 분리한 데 있다.

### 1. 실행 불변층: `prepare.py`

여기는 데이터, 토크나이저, 평가 기준, 시간 예산 같은 실험의 규칙을 고정하는 층이다. 에이전트가 수정하지 않는다.

### 2. 탐색층: `train.py`

여기는 모델 구조, 초기화, 옵티마이저, 배치 크기, 스케줄 등 에이전트가 실제로 탐색하는 층이다. 실험 차이가 여기에서만 발생한다.

### 3. 조직층: `program.md`

여기는 코드를 직접 실행하는 Python이 아니라, 에이전트를 실행하는 운영 체제다. 어떤 브랜치를 만들고, 어떤 로그를 남기고, 언제 revert하고, baseline을 어떻게 처리할지 정의한다.

이걸 다르게 말하면 `autoresearch`는 Python 코드와 Markdown 코드를 분리한다.

- Python은 모델을 정의한다.
- Markdown은 연구 조직을 정의한다.

Karpathy가 README에서 말한 "you are programming the `program.md` Markdown files"라는 표현은 과장이 아니다. 이 프로젝트에서는 `program.md`가 사실상 연구실의 SOP(Standard Operating Procedure)다.

## 프로젝트 구조: 작은 저장소, 큰 의도

`autoresearch`는 의도적으로 작다. 실제로 핵심 파일은 거의 네 개뿐이다.

| 파일 | 역할 | 수정 주체 |
|---|---|---|
| `prepare.py` | 데이터 다운로드, 토크나이저 학습, dataloader, BPB 평가 | 사람/고정 |
| `train.py` | 모델, 옵티마이저, 학습 루프 | 에이전트 |
| `program.md` | 실험 프로토콜, git 루프, 기록 규칙 | 사람 |
| `analysis.ipynb` | 실험 결과 후처리와 시각화 | 사람 |

이 구조가 중요한 이유는 에이전트가 수정할 수 있는 자유도를 제한하면서도, 실제 성능 향상에 필요한 대부분의 레버를 `train.py` 안에 남겨 두기 때문이다.

이건 흔한 "제약"이 아니라 매우 계산된 제품 설계다. 자유도가 너무 넓으면 에이전트는 헤매고, 자유도가 너무 좁으면 개선 여지가 없다. `autoresearch`는 그 중간 지점을 정확히 겨냥한다.

## 코드 레벨 분석 1: `prepare.py`

`prepare.py`는 단순 준비 스크립트가 아니다. 실험의 공정성을 보장하는 규칙 엔진에 가깝다.

### 1. 고정 상수

파일 상단에는 다음과 같은 핵심 상수가 있다.

- `MAX_SEQ_LEN = 2048`
- `TIME_BUDGET = 300`
- `EVAL_TOKENS = 40 * 524288`
- `VOCAB_SIZE = 8192`
- 고정 validation shard: `shard_06542.parquet`

여기서 중요한 것은 `TIME_BUDGET`과 `VAL_SHARD`다.

`TIME_BUDGET = 300`은 이 프로젝트의 목적 함수를 완전히 바꾼다. 에이전트는 더 이상 "이론적으로 더 좋은 모델"이 아니라 "내 GPU에서 5분 안에 가장 잘 학습되는 모델"을 찾는다. 다시 말해, `autoresearch`는 품질 최적화와 시스템 최적화를 동시에 수행한다.

또한 validation shard를 하나로 고정해 두었기 때문에 각 실험은 같은 분포에서 비교된다. 이는 실험 반복 속도를 위해 통계적 엄밀성을 일부 희생한 대신, 비교 가능성을 최대화한 선택이다.

### 2. 데이터 다운로드

데이터는 Hugging Face의 `karpathy/climbmix-400b-shuffle`에서 parquet shard를 받아온다. train shard 몇 개를 받을지와 별개로 validation shard는 항상 고정된 마지막 shard를 포함하도록 설계되어 있다.

이 선택은 에이전트의 실험이 데이터 샘플링 운에 흔들리는 것을 줄여 준다. 다시 말해, "오늘 validation batch가 우연히 쉬웠다"는 식의 변수를 줄이고, 같은 shard를 계속 비교 기준으로 사용한다.

### 3. 토크나이저 학습

토크나이저는 `rustbpe`로 학습되고, 이후 `tiktoken.Encoding` 형태로 저장된다. 이 단계는 중요하다.

- 토크나이저를 repo 밖 `~/.cache/autoresearch/`에 저장한다.
- 토큰별 byte 길이 lookup tensor를 별도로 만든다.
- special token은 byte length 0으로 처리한다.

이 구조는 나중의 `val_bpb` 평가를 위한 준비다. 즉, `autoresearch`는 일반적인 loss가 아니라 byte-normalized metric을 평가하기 때문에, vocab size가 달라도 비교가 가능하다.

### 4. dataloader 설계

`make_dataloader`는 BOS-aligned best-fit packing을 구현한다.

핵심 로직은 다음과 같다.

1. 각 row는 BOS로 시작한다.
2. buffer에서 현재 남은 길이에 가장 잘 맞는 문서를 찾는다.
3. 맞는 문서가 없으면 가장 짧은 문서를 잘라 남은 공간을 채운다.
4. 패딩 없이 100% utilization을 목표로 한다.

이 방식은 연구적으로 꽤 흥미롭다. 일반적인 padding 기반 배치보다 토큰 낭비가 적고, 모든 row가 문서 시작점에서 정렬되므로 학습 시 문맥 구조가 더 일관된다. 대신 일부 문서는 crop되며, 원문 분포가 훼손될 수 있다. `nanochat`에서도 같은 아이디어를 사용하고 있으므로, `autoresearch`는 단순한 장난감이 아니라 이미 성능이 검증된 입력 파이프라인을 가져와 최소화한 것이다.

### 5. 평가 함수 `evaluate_bpb`

이 함수는 사실상 연구의 "헌법"이다.

- 고정된 `MAX_SEQ_LEN`
- 고정된 validation split
- token별 cross entropy를 byte 길이로 정규화
- special token은 제외

이렇게 해서 얻는 `bits per byte`는 vocab-size invariant metric이 된다. 이 선택은 상당히 영리하다. 토크나이저가 다르거나 vocab size가 다를 때도 어느 정도 공정한 비교가 가능하기 때문이다.

단, 이 구조는 강력한 장점과 함께 분명한 한계도 가진다. validation이 단일 shard에 고정되어 있기 때문에, 에이전트가 그 shard 특성에 과적합하는 방향으로 탐색할 가능성을 완전히 배제할 수는 없다.

## 코드 레벨 분석 2: `train.py`

`train.py`는 `autoresearch`의 전장이다. 에이전트는 사실상 이 파일 하나만 읽고, 고치고, 실험하고, revert한다.

### 1. 실행 환경과 의존성

파일 시작부는 꽤 공격적이다.

- `PYTORCH_ALLOC_CONF=expandable_segments:True`
- CUDA capability를 확인해 Flash Attention 3 커널 repo를 고른다.
- `prepare.py`에서 시간 예산, 토크나이저, dataloader, 평가 함수를 import한다.

여기서 중요한 포인트는 repo 내부 코드만으로 완전히 닫힌 환경은 아니라는 점이다. `kernels` 패키지를 통해 외부 Flash Attention 구현을 끌어온다. README가 "self-contained"를 강조하긴 하지만, 엄밀히 말하면 실행 경로 일부는 repo 바깥에 있다. 이건 작지만 중요한 현실적 타협이다.

### 2. 모델 구조

모델은 `nanochat`의 pretraining GPT를 단일 파일 안으로 압축한 형태다.

핵심 특징:

- rotary embedding
- RMSNorm 기반 `norm`
- untied embedding / lm_head
- `relu^2` MLP
- alternating value embeddings
- sliding window attention
- Flash Attention 3
- residual scalar parameter (`resid_lambdas`, `x0_lambdas`)

이 모델은 단순한 GPT baseline이 아니다. 이미 여러 실전 최적화가 들어간 "강한 최소 baseline"이다. 즉, `autoresearch`는 약한 토이 모델을 에이전트에게 던져 주는 것이 아니라, 이미 높은 성능을 내는 compact research harness를 제공한다.

### 3. `GPTConfig`와 구조적 제약

`build_model_config(depth)`는 `depth * ASPECT_RATIO`로 기본 model dimension을 만든 다음, `HEAD_DIM`의 배수로 올림한다. 덕분에 에이전트는 복잡한 모양 제약을 직접 계산하지 않아도 된다.

이 선택은 작아 보이지만 매우 중요하다. 에이전트가 자주 망가뜨리는 영역은 "형상 일관성"이다. `autoresearch`는 이를 helper 함수와 상수 설계로 먼저 방어한다.

### 4. Attention 블록

`CausalSelfAttention`에는 일반 GPT 구현보다 몇 가지 눈에 띄는 요소가 있다.

- `ve_gate`를 통한 value embedding mixing
- rotary embedding 적용 후 QK norm
- sliding window attention 패턴
- FA3 기반의 고성능 attention 호출

흥미로운 점은 이 파일의 baseline 자체가 이미 실험적이라는 것이다. 다시 말해 `autoresearch`는 "AI에게 처음부터 연구를 시키는" 프로젝트가 아니다. 오히려 사람이 어느 정도 잘 설계해 둔 research playground를 만들고, 에이전트는 그 위에서 후속 미세 탐색을 수행한다.

### 5. MLP와 initialization

MLP는 `relu().square()`를 사용한다. 이는 `nanochat`와 동일한 선택으로, 최근 Karpathy 코드에서 자주 보이는 설계다.

초기화도 눈여겨볼 부분이 많다.

- embedding은 normal init
- attention / MLP의 projection 일부는 zero init
- residual 관련 scalar는 별도 초기화
- value embedding gate는 neutral behavior를 유도하도록 zero init

이런 초기화는 에이전트가 나중에 손대기 좋은 표적이다. 실제로 `nanochat`에 반영된 round 1 개선점 중 상당수가 이 초기화와 gate 구조를 조정하는 방향이었다.

### 6. 옵티마이저: `MuonAdamW`

이 파일의 기술적 밀도는 옵티마이저 부분에서 가장 높다. 이름만 보면 어렵지만, 독자 관점에서는 "파라미터 종류에 따라 다른 최적화 규칙을 섞어 쓰는 고성능 optimizer" 정도로 이해하면 충분하다. 이 구간의 핵심은 수학적 디테일보다는, 왜 `autoresearch`가 같은 5분 안에 더 많은 유효 학습을 밀어 넣도록 설계됐는가에 있다.

`train.py`는 두 종류의 parameter group을 분리한다.

- AdamW: embedding, unembedding, scalar
- Muon: 2D matrix parameter

그리고 shape별로 matrix param을 묶어 fused Muon step을 수행한다. 이 코드는 단순히 "optimizer를 바꿔볼 수 있다" 수준이 아니라, 에이전트가 optimizer hyperparameter를 건드렸을 때 실제로 성능 변화가 잘 나타나는 강한 기반을 제공한다.

또한 `adamw_step_fused`와 `muon_step_fused`는 `torch.compile`된 함수다. 즉, inner loop 성능을 상당히 밀어붙이고 있다.

용어도 짚고 가자. `fused`는 여러 optimizer 연산을 한 번에 묶어 GPU 오버헤드를 줄이는 구현이고, `torch.compile`은 반복되는 연산 경로를 더 빠르게 실행하도록 PyTorch가 컴파일해 주는 레이어다. 세부 구현을 몰라도, 여기서 중요한 것은 이 조합이 "같은 300초 동안 더 많은 step을 벌어 준다"는 점이다.

이건 두 가지 의미를 갖는다.

1. 좋은 점: 5분 예산 안에서 더 많은 토큰을 처리할 수 있다.
2. 위험한 점: 에이전트가 구조를 많이 바꾸면 `torch.compile` 관련 실패나 성능 저하를 일으킬 수 있다.

즉, `autoresearch`는 "탐색 공간"을 단순화했지만, "시스템 복잡도" 자체는 생각보다 낮지 않다.

### 7. 하이퍼파라미터 표면

`train.py`의 하이퍼파라미터는 모두 파일 상단 상수로 놓여 있다.

- `ASPECT_RATIO`
- `HEAD_DIM`
- `WINDOW_PATTERN`
- `TOTAL_BATCH_SIZE`
- 각종 learning rate
- `WEIGHT_DECAY`
- `DEPTH`
- `DEVICE_BATCH_SIZE`

이 방식은 CLI보다 훨씬 에이전트 친화적이다. 에이전트는 플래그 체계를 이해할 필요 없이 텍스트 패치를 통해 수치를 바로 수정하면 된다.

이건 단순 취향 문제가 아니다. 에이전트는 "코드 문맥 내 수치 수정"에는 강하지만, "런치 스크립트와 CLI 조합을 일관되게 관리"하는 데서는 종종 실수한다. `autoresearch`는 이 약점을 우회한다.

### 8. 시간 기반 스케줄링

여기서 가장 중요한 설계 포인트 하나를 꼭 짚어야 한다.

`autoresearch`는 step 기반이 아니라 **training time 기반**으로 스케줄을 잡는다.

- `progress = total_training_time / TIME_BUDGET`
- LR warmup / warmdown이 시간 비율에 따라 움직인다.
- 첫 10 step은 compile warmup으로 보고 budget 계산에서 제외한다.

이건 대단히 의미 있는 선택이다. 에이전트가 모델 크기나 batch size를 바꿔 step 수가 달라져도, "5분 동안의 최적점"을 같은 기준에서 비교할 수 있다.

즉, `autoresearch`의 숨은 목적 함수는 다음과 같다.

> 300초 동안 얼마나 많은 유효한 학습을 해내느냐

이 목적 함수는 순수한 모델 품질만이 아니라 throughput, kernel efficiency, memory fit, scheduler shape를 모두 하나의 숫자로 압축한다.

### 9. 학습 루프

학습 루프 자체는 매우 단순하다.

```text
prefetch batch
for 5 minutes:
  gradient accumulation
  schedule update
  optimizer step
  fail fast if NaN/exploding
end
evaluate val_bpb once
print summary
```

중요한 세부 사항:

- gradient accumulation으로 `TOTAL_BATCH_SIZE`를 맞춘다.
- loss NaN 또는 100 초과면 즉시 실패 처리한다.
- Python GC를 적극 제어해 500ms 수준의 stall을 줄이려 한다.
- 마지막에 `val_bpb`, `training_seconds`, `peak_vram_mb`, `mfu_percent`, `num_params_M`를 출력한다.

이 출력 포맷은 `program.md`의 grep 루프와 정확히 맞물린다. 즉, 코드와 Markdown protocol이 함께 설계되어 있다. 이 프로젝트는 "코드"만 잘 짠 게 아니라 "에이전트가 읽을 인터페이스"도 잘 짰다.

## 코드 레벨 분석 3: `program.md`

솔직히 말해 `autoresearch`의 핵심 혁신은 `train.py`보다 `program.md`에 더 가깝다.

이 파일은 에이전트에게 다음을 지시한다.

- 새 브랜치를 만든다.
- baseline을 먼저 측정한다.
- `train.py`만 수정한다.
- commit 후 `uv run train.py > run.log 2>&1`로 실행한다.
- `grep`으로 핵심 metric만 읽는다.
- 좋아지면 keep, 아니면 reset한다.
- `results.tsv`에 keep/discard/crash를 남긴다.
- 절대 멈추지 말고 계속 반복한다.

이건 사실상 자연어로 작성된 자율 연구 에이전트의 finite-state machine이다.

더 흥미로운 건 이 파일이 단순 프롬프트가 아니라는 점이다. 이것은 다음 네 가지를 함께 제공한다.

1. 목표 함수
2. 도구 사용 규칙
3. 상태 전이 규칙
4. 기록 체계

많은 사람들이 에이전트 시스템을 만들 때 orchestration layer를 Python 프레임워크로 생각한다. 하지만 `autoresearch`는 orchestration을 Markdown으로 옮겼다. 그 결과, 사람이 조직 설계를 더 쉽게 바꾸고, 에이전트는 그 운영 규약 안에서 일관되게 행동할 수 있다.

이건 "prompt engineering"보다 한 단계 높은 개념이다. 정확히는 "research process engineering"에 가깝다.

### 왜 하필 `program.md`라는 Markdown이 핵심이었는가

이 질문은 중요하다. 많은 팀이라면 여기서 별도의 YAML config, JSON workflow, 혹은 Python orchestration framework를 만들었을 것이다. 그런데 공개 저장소는 굳이 Markdown을 택했다. README가 `program.md`를 "a super lightweight skill"이라고 부른다는 점을 보면, 이 선택은 단순 문서 포맷 이상의 의미를 가진다.

첫째, Markdown은 인간과 모델이 동시에 읽기 좋은 형식이다. 사람에게는 문서이고, LLM에게는 긴 문맥 프롬프트다. 즉, 별도의 translation layer가 필요 없다.

둘째, Markdown은 실행 규약을 코드에서 분리한다. `train.py`는 모델 실험의 표면이고, `program.md`는 연구 조직의 표면이다. 이 둘을 분리하면, 사람은 연구 프로세스를 바꾸면서도 모델 코드를 직접 건드리지 않을 수 있다.

셋째, Markdown은 git과 궁합이 좋다. diff가 잘 보이고, 버전 관리가 쉽고, 사람 리뷰가 가능하다. 다시 말해 `program.md`는 읽기 쉬운 prompt이면서 동시에 리뷰 가능한 운영 코드다.

넷째, Markdown은 특정 에이전트 프레임워크에 종속되지 않는다. `autoresearch` README가 Claude나 Codex 같은 범용 코딩 에이전트를 바로 예시로 든다는 점을 보면, 이 저장소는 커스텀 런타임보다 "아무 에이전트나 들어와도 읽을 수 있는 프로토콜"을 우선시한 것으로 읽힌다.

다섯째, Markdown은 수정 비용이 낮다. 에이전트가 잘 안 움직이면 프롬프트 문구, 운영 규칙, 로그 규약, 판단 기준을 바로 바꿀 수 있다. 프레임워크 코드나 툴 러너를 리팩터링하는 것보다 훨씬 빠르다. 연구 자동화 초기에 가장 중요한 것은 추상화의 아름다움보다 iteration speed인데, Markdown은 그 점에서 압도적으로 유리하다.

결국 `program.md`는 문서가 아니라 인터페이스다. 더 정확히는, 인간이 연구 조직을 정의하고 에이전트가 그 조직을 실행하게 만드는 **LLM-native control plane**이다.

## 코드 레벨 분석 4: `analysis.ipynb`

많이 주목받지는 않지만, `analysis.ipynb`도 중요하다. 이 노트북은 `results.tsv`를 읽어 다음을 분석한다.

- kept experiment만 추출
- frontier가 어떻게 내려갔는지 시각화
- baseline 대비 총 개선량 계산
- 어떤 실험이 가장 큰 개선을 만들었는지 정렬

이 파일의 존재는 중요한 메시지를 준다. `autoresearch`의 목적은 "에이전트가 모든 걸 끝낸다"가 아니다. 오히려 "에이전트가 대량의 local search를 수행하고, 사람은 그 결과를 압축해 해석한다"에 가깝다.

즉, 자율 연구의 완성형이라기보다 "탐색은 자동화하고 해석은 인간이 맡는 구조"다. 그리고 당분간은 이 분업이 현실적이다.

## `nanochat`와의 관계: 무엇을 가져오고 무엇을 버렸는가

`autoresearch`는 `nanochat`를 대체하는 프로젝트가 아니다. 오히려 `nanochat`의 연구용 pretraining 서브시스템을 single-GPU, single-file, agent-editable 형태로 증류한 프로젝트다.

아래 비교가 핵심을 잘 보여 준다.

| 항목 | `nanochat` | `autoresearch` |
|---|---|---|
| 목적 | end-to-end LLM harness | 자율 pretraining 연구 루프 |
| 컴퓨트 | CPU/MPS/CUDA, DDP 포함 | 단일 NVIDIA GPU 중심 |
| 수정 표면 | 다수의 모듈과 CLI | 사실상 `train.py` 하나 |
| 평가 | `val_bpb`, CORE, 샘플링, 리포트 | `val_bpb` 단일 지표 |
| 운영 | wandb, checkpoint, resume, distributed | `run.log` + `results.tsv` |
| 인간 역할 | 훈련과 해석 모두 수행 | `program.md` 작성과 결과 해석 |
| 에이전트 역할 | 선택적 보조자 | 주된 실험 수행자 |

### `nanochat`에서 유지한 것

- GPT trunk 설계
- value embedding과 gate
- Muon + AdamW 혼합 optimizer 철학
- BOS-aligned best-fit dataloader
- `val_bpb` 중심 평가
- meta device 초기화와 `torch.compile` 기반 성능 최적화

### `nanochat`에서 제거한 것

- DDP와 분산 학습
- checkpoint/resume 복잡성
- wandb 로깅
- CORE metric 평가
- chat / SFT / RL / Web UI
- multi-platform fallback
- CLI 기반의 넓은 설정 표면

핵심은 단순하다. `autoresearch`는 기능을 덜어낸 것이 아니라, 에이전트가 다루기 어려운 차원을 제거했다.

## round 1에서 실제로 무엇이 발견되었는가

이 부분이 가장 흥미롭다. `nanochat`의 `6ed7d1d` 커밋 메시지는 `autoresearch`가 발견한 개선들을 직접 요약한다.

변경점 목록만 보면 추상적으로 느껴질 수 있으니, 실제 루프 하나를 글로 펼치면 대략 이런 그림이다. 아래는 **현재 공개된** `program.md`의 규약과 round 1에 최종 반영된 변경 항목을 바탕으로 재구성한 예시다. 즉, round 1 내부 버전을 그대로 복원한 것이 아니라, 공개 저장소가 지향하는 운영 루프를 설명하는 데 더 가깝다.

```diff
- UNEMBEDDING_LR = 0.004
+ UNEMBEDDING_LR = 0.008
```

1. 에이전트는 baseline 브랜치에서 기존 `train.py`를 5분 돌려 기준 `val_bpb`를 확보한다.
2. 그다음 `train.py` 상단 상수 한 줄을 바꿔 새 가설을 만든다.
3. `uv run train.py > run.log 2>&1`를 실행하고, 마지막 요약 줄에서 `val_bpb`와 `peak_vram_mb`를 `grep`으로 읽는다.
4. `val_bpb`가 더 낮아졌으면 commit을 유지하고 `results.tsv`에 keep으로 기록한다.
5. 나빠졌으면 reset하고 다음 가설로 넘어간다.

핵심은 개별 patch의 화려함이 아니다. 이런 작은 diff가 밤새 누적되면서, 사람이 아니라 에이전트가 local search를 계속 밀어붙인다는 점이 round 1의 본질이다.

### 최적화와 스케줄 변화

- unembedding LR: `0.004 -> 0.008`
- weight decay: `0.2 -> 0.28`
- Adam betas / weight decay를 parameter group별로 분리
- Muon `beta2`: `0.95 -> 0.9`
- Muon momentum warmup target: `0.95 -> 0.97`, 400 step으로 연장
- warmup: ratio 기반에서 absolute 40 step으로 변경
- warmdown ratio: `0.5 -> 0.65`
- final LR fraction: `0.0 -> 0.05`
- weight decay schedule: linear decay -> cosine decay
- polar express normalization factor: `1.02 -> 1.01`

### 아키텍처와 초기화 변화

- VE gate channel: `32 -> 12`
- gate scale range: `2x -> 3x`
- gate init: zero -> small positive
- post-QK-norm scaling 추가: `q, k *= 1.15`
- embedding init std: `1.0 -> 0.8`
- MLP `c_fc` init scale 0.5x 축소
- RoPE base theta: `10K -> 100K`
- short attention window: `seq_len/2 -> 약 seq_len/3`
- logit softcap: `20 -> 15`

이 리스트는 두 갈래로 읽어야 한다.

### 1. sample efficiency를 높이는 변화

- unembedding LR 증가
- parameter-group별 optimizer 설정
- Muon beta2 / momentum 수정
- QK scaling
- 초기화 안정화
- logit softcap 조정

이들은 같은 토큰 수를 봤을 때 더 잘 학습되게 하려는 변화다.

### 2. 5분 예산 안 throughput 또는 효율을 바꾸는 변화

- short attention window 축소
- 스케줄 형태 변경
- warmup/warmdown 구조 조정

특히 sliding window를 `1/2`에서 `약 1/3`로 줄이는 변화는 `autoresearch`의 목적 함수가 무엇인지 잘 보여 준다. 이 프로젝트는 "절대적으로 더 좋은 모델"을 찾는 것이 아니라 "고정된 300초 안에서 가장 좋은 결과"를 찾는다. 따라서 어떤 변화는 품질 자체를 높여서, 어떤 변화는 더 많은 step 또는 더 안정적인 step을 확보해서 승리할 수 있다.

이게 바로 `autoresearch`의 묘미다. 모델 연구와 시스템 연구가 분리되지 않는다.

## 왜 이 설계가 강력한가

### 1. 탐색 공간이 충분히 좁다

에이전트는 `train.py`만 고친다. 덕분에 변경 diff가 작고, 실패 원인 추적이 쉽고, revert가 간단하다.

### 2. 평가 함수가 충분히 명확하다

`val_bpb` 하나로 모든 실험을 비교한다. 멀티 메트릭 최적화가 아니다.

### 3. 시간 예산이 연구를 계산 자원에 맞춘다

fixed-step이 아니라 fixed-time이기 때문에, agent는 모델 구조와 시스템 효율을 동시에 탐색한다.

### 4. git이 memory system 역할을 한다

좋은 실험은 commit으로 남고, 나쁜 실험은 reset된다. `results.tsv`는 실험 로그 역할을 한다.

### 5. 자연어가 orchestration layer가 된다

복잡한 agent framework 없이도, `program.md` 하나로 상당한 수준의 연구 프로토콜을 정의할 수 있다.

## 기술적 한계와 비판적 평가

`autoresearch`는 인상적이지만, 과대평가하면 안 된다. 이 시스템은 매우 영리한 local search engine이지, 과학적 발견 기계는 아니다.

### 1. 단일 metric 최적화의 위험

`val_bpb` 하나만 보고 keep/discard를 결정한다. 이는 빠르고 좋지만, 다른 역량을 놓친다. 실제 downstream generalization이나 qualitative behavior는 별개일 수 있다.

### 2. 단일 validation shard 편향

validation shard가 고정이라 비교는 쉬워지지만, 특정 shard 특성을 향한 과적합 가능성은 남는다.

### 3. 단일 run 기반 의사결정

통계적 반복 측정이 없다. 한 번의 5분 실험으로 keep/discard를 결정한다. 노이즈가 큰 환경에서는 잘못된 선택을 할 수 있다.

### 4. 하드웨어 특화 목적 함수

README도 인정하듯, `autoresearch`는 "당신의 하드웨어에서 5분 안에 가장 좋은 모델"을 찾는다. 다른 GPU, 다른 kernel stack, 다른 메모리 조건에서는 결과가 달라질 수 있다.

### 5. portability가 낮다

현재 baseline은 사실상 단일 NVIDIA GPU 중심이다. `train.py`는 `torch.device("cuda")`를 직접 사용하고, CUDA 기반 memory metric과 FA3 경로에 강하게 묶여 있다.

### 6. repo는 작지만 실행 스택은 생각보다 날카롭다

`torch.compile`, fused optimizer, external kernel fetching, CUDA capability 분기 등은 숙련자에겐 효율이지만, 에이전트가 과격한 구조 변경을 시도할 때는 brittleness의 원인이 된다.

### 7. 이것은 완전 자율 연구실이 아니다

사람은 여전히 다음을 맡는다.

- `program.md`를 설계한다.
- baseline과 protocol을 정한다.
- 결과를 해석한다.
- 어떤 개선을 상위 프로젝트에 반영할지 결정한다.

즉, `autoresearch`는 human-out-of-the-loop가 아니라 human-above-the-loop 시스템이다.

## 그럼에도 불구하고 왜 의미가 큰가

이 프로젝트는 "AI가 연구자를 대체했다"는 증거라기보다, "연구 프로세스가 AI가 잘하는 형태로 다시 포장되기 시작했다"는 증거다.

더 정확히 말하면, `autoresearch`는 다음 전환을 보여 준다.

- 기존 방식: 사람이 코드를 바꾸고, 모델을 돌리고, 로그를 읽고, 다시 고친다.
- `autoresearch` 방식: 사람은 연구실의 운영 규약을 쓰고, 에이전트가 코드 변경과 실험 루프를 수행한다.

여기서 사람의 역할은 implementation에서 protocol design으로 이동한다.

이 변화는 매우 크다. 왜냐하면 앞으로 경쟁 우위는 "누가 더 빨리 코드를 쓰는가"보다 "누가 더 좋은 자율 탐색 환경을 설계하는가"로 이동할 가능성이 높기 때문이다.

## 내 해석: HPO·AutoML·NAS와 무엇이 다른가

겉으로 보면 `autoresearch`도 자동 탐색 시스템이다. 하지만 기존 자동화 계열과는 결이 다르다.

| 접근 | 주로 탐색하는 것 | `autoresearch`와의 차이 |
|---|---|---|
| HPO | 고정된 코드 위의 learning rate, batch size 같은 수치 | `autoresearch`는 수치뿐 아니라 `train.py` 코드 자체를 수정한다. 대신 수정 범위를 한 파일로 묶어 폭주를 막는다. |
| NAS | 레이어 수, 연결 구조 같은 아키텍처 공간 | `autoresearch`는 아키텍처만이 아니라 optimizer, initialization, scheduler, attention window처럼 시스템과 학습이 얽힌 표면을 함께 탐색한다. |
| AutoML | 모델 선택, 전처리, 평가 파이프라인 자동화 | `autoresearch`는 별도 컨트롤러보다 git, `program.md`, keep/discard 규약으로 연구실 SOP 자체를 자동화한다. |

이 비교로 보면 `autoresearch`의 진짜 산출물은 단일 모델 변경이라기보다 **연구 운영체제**에 가깝다.

- 고정된 실험 헌법: `prepare.py`
- 수정 가능한 탐색 표면: `train.py`
- 자연어 프로토콜: `program.md`
- 실험 메모리: git + `results.tsv`
- 사후 해석 도구: `analysis.ipynb`

이 패턴은 꼭 LLM pretraining에만 쓰이는 것이 아니다. inference latency tuning, compiler flag search, prompt eval optimization, retrieval pipeline tuning처럼 "하나의 지표, 하나의 시간 예산, 제한된 수정 표면"이 성립하는 작업이라면 넓게 재사용될 수 있다.

## 실무자가 여기서 배워야 할 것

이 프로젝트가 주는 실전 교훈은 명확하다.

### 1. 에이전트를 똑똑하게 만들려 하지 말고, 문제를 에이전트 친화적으로 바꿔라

수정 가능한 파일 수를 줄이고, 평가 함수를 하나로 고정하고, 실험 시간을 제한하면 에이전트의 성능이 급격히 올라간다.

### 2. 자연어 문서는 보조 문서가 아니라 실행 계층이 될 수 있다

`program.md`는 README가 아니라 orchestration layer다. 앞으로의 문서는 설명서가 아니라 운영 규약이 될 가능성이 높다.

### 3. 자율 탐색은 작은 루프에서 먼저 이긴다

거대한 end-to-end autonomous company보다, 5분짜리 연구 loop 자동화가 더 빨리 실질 성과를 낸다.

### 4. metric design이 agent design보다 중요할 수 있다

에이전트가 아무리 좋아도 metric이 흐리면 헤맨다. `autoresearch`는 이 점을 정확히 보여 준다.

## 결론

`autoresearch`는 연구 자동화의 완성형이 아니다. 하지만 방향은 놀라울 정도로 선명하다.

이 프로젝트는 에이전트에게 무한한 자유를 준 것이 아니라, 아주 제한된 자유와 아주 명확한 목적 함수를 주었다. 그 결과 에이전트는 실제로 유용한 개선을 찾아냈고, 그 개선은 상위 프로젝트인 `nanochat`의 leaderboard 향상으로 이어졌다.

내가 보기에 이 프로젝트의 가장 큰 의미는 "AI가 연구할 수 있다"가 아니라 "연구라는 일을 AI가 잘할 수 있는 인터페이스로 다시 정의할 수 있다"는 데 있다.

그리고 그 재정의의 핵심은 거대한 프레임워크가 아니라, surprisingly 작은 조합이다.

- 하나의 고정 평가 함수
- 하나의 수정 가능한 학습 파일
- 하나의 자연어 운영 프로토콜
- 하나의 keep/discard 루프

이 정도면, 연구실은 이미 절반쯤 소프트웨어가 된 셈이다.

## 참고 링크

- `autoresearch` README: [https://github.com/karpathy/autoresearch/blob/master/README.md](https://github.com/karpathy/autoresearch/blob/master/README.md)
- `autoresearch` `program.md`: [https://github.com/karpathy/autoresearch/blob/master/program.md](https://github.com/karpathy/autoresearch/blob/master/program.md)
- `autoresearch` `prepare.py`: [https://github.com/karpathy/autoresearch/blob/master/prepare.py](https://github.com/karpathy/autoresearch/blob/master/prepare.py)
- `autoresearch` `train.py`: [https://github.com/karpathy/autoresearch/blob/master/train.py](https://github.com/karpathy/autoresearch/blob/master/train.py)
- `nanochat` README: [https://github.com/karpathy/nanochat/blob/master/README.md](https://github.com/karpathy/nanochat/blob/master/README.md)
- `nanochat` round 1 반영 커밋: [https://github.com/karpathy/nanochat/commit/6ed7d1d82cee16c2e26f45d559ad3338447a6c1b](https://github.com/karpathy/nanochat/commit/6ed7d1d82cee16c2e26f45d559ad3338447a6c1b)
- leaderboard 반영 커밋: [https://github.com/karpathy/nanochat/commit/f06860494848db080c9a80a0ffa83203b042056b](https://github.com/karpathy/nanochat/commit/f06860494848db080c9a80a0ffa83203b042056b)
- X 포스트: [https://x.com/karpathy/status/2031135152349524125](https://x.com/karpathy/status/2031135152349524125)
- 텍스트 미러: [https://xcancel.com/karpathy/status/2031135152349524125](https://xcancel.com/karpathy/status/2031135152349524125)
