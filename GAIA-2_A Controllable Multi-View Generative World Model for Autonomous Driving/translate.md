###### 요약

생성 모델은 복잡한 환경을 시뮬레이션하기 위한 확장 가능하고 유연한 패러다임을 제공하지만, 현재 접근 방식은 다중 에이전트 상호 작용, 정밀 제어, 다중 카메라 일관성과 같은 자율 주행의 도메인 특정 요구 사항을 해결하는 데 부족합니다. 우리는 이러한 기능을 단일 생성 프레임워크 내에서 통합하는 잠재 확산 세계 모델인 GAIA-2, Generative AI for Autonomy를 소개합니다. GAIA-2는 자율 주행 차량 역학, 에이전트 구성, 환경 요인 및 도로 의미론과 같은 풍부한 구조화된 입력에 따라 제어 가능한 비디오 생성을 지원합니다. 지리적으로 다양한 운전 환경(영국, 미국, 독일)에 걸쳐 고해상도, 시공간적으로 일관된 다중 카메라 비디오를 생성합니다. 이 모델은 구조화된 조건화와 외부 잠재 임베딩(예: 독점 주행 모델에서 가져온 임베딩)을 모두 통합하여 유연하고 의미론적으로 근거 있는 장면 합성을 용이하게 합니다. 이러한 통합을 통해 GAIA-2는 일반 및 희귀 운전 시나리오의 확장 가능한 시뮬레이션을 가능하게 하여 자율 시스템 개발의 핵심 도구로서 생성 세계 모델의 사용을 발전시킵니다. 비디오는 [https://wayve.ai/thinking/gaia-2](https://wayve.ai/thinking/gaia-2)에서 볼 수 있습니다.

## 1 서론

운전 시나리오의 현실적인 시뮬레이션은 자율 주행 시스템의 개발, 훈련 및 평가를 위한 기본적인 요구 사항입니다. 생성 세계 모델은 확장 가능하고 다양한 합성 데이터 생성을 가능하게 하여 고가의 실제 데이터 수집에 대한 의존도를 줄이고 안전하고 반복 가능한 환경에서 강력한 평가를 용이하게 합니다. 주로 시각적 사실성 및 시간적 일관성에 중점을 두는 일반적인 텍스트-비디오 또는 이미지-비디오 모델과 달리, 자율 주행 애플리케이션은 장면의 도메인 특정 측면에 대한 정밀한 제어를 요구합니다.

자율 주행을 위한 생성 모델은 자율 주행 차량의 행동, 다른 에이전트(예: 차량, 보행자, 자전거 운전자)의 위치 및 움직임, 그리고 그들의 상호 작용과 같은 요소를 정확하게 시뮬레이션해야 합니다. 더욱이, 이 모델은 지리적 위치, 날씨, 시간, 도로 구성(예: 속도 제한, 차선 수, 횡단보도, 신호등, 교차로) 및 드물지만 중요한 엣지 케이스 시나리오와 같은 상황별 속성을 기반으로 조건부 생성을 허용해야 합니다. 자율 주행 차량은 지리적으로나 시간적으로 일관된 여러 관점의 입력에 의존하여 인지 및 의사 결정을 수행하므로 일관된 다중 카메라 비디오 스트림을 생성할 수 있는 능력도 필수적입니다. 이러한 도메인 특정 제약을 충족하는 것은 기존 비디오 생성 모델로는 적절하게 해결되지 않는 고유한 과제를 제시합니다.

잠재 비디오 생성 분야의 최근 발전, 특히 연속 [^1] 또는 양자화 [^2] 잠재 공간을 활용하는 발전은 효율성 및 시각적 품질 향상으로 이어졌지만, 현재 접근 방식은 여전히 범위가 제한적입니다. 자율 주행의 맥락에서 여러 모델이 작업별 조건화 메커니즘 [^3]을 도입하여 시나리오 요소에 대한 일부 제어를 가능하게 했습니다. 그러나 기존 모델은 일반적으로 제한된 조건화 유형, 다중 카메라 생성 부족 또는 열악한 시공간 일관성과 같은 필요한 기능의 일부만 지원합니다. 따라서 포괄적이고 통합된 생성 프레임워크는 해당 분야에서 여전히 미해결 과제로 남아 있습니다.

이러한 격차를 해소하기 위해, 우리는 자율 주행을 위한 비디오 생성 세계 모델링에서 상당한 진전을 이룬 도메인 전문 잠재 확산 모델인 GAIA-2를 소개합니다. GAIA-2는 자율 주행 차량 행동, 에이전트 행동, 장면 기하학 및 환경 요인에 대한 정밀 제어를 통해 고해상도, 다중 카메라 주행 장면의 조건부 생성을 지원함으로써 이전 작업을 발전시킵니다. 이 모델은 최대 5개의 시간적 및 공간적으로 일관된 카메라 스트림을 448times960 해상도로 생성할 수 있으며, 실제 자율 주행 차량 시스템에서 사용되는 다양한 다중 카메라 리그 구성을 수용합니다.

![](./figure/1.png)

그림 1: GAIA-2로 가능한 합성 시나리오의 다양성을 보여주는 '처음부터' 생성 선택.

GAIA-2는 자율 주행 차량 운동학(예: 속도, 곡률), 지리적 지역(영국, 미국, 독일), 시간, 날씨, 그리고 차선 수 및 유형(예: 주행 가능, 버스, 자전거), 횡단보도, 신호등, 교차로 위상(예: 일방통행 도로, 원형 교차로)과 같은 풍부한 도로 레이아웃 기능 분류를 포함하여 광범위한 장면 속성을 조건화할 수 있습니다. 또한 장면 내 동적 에이전트의 위치, 방향 및 치수에 대한 직접적인 제어를 허용합니다. 이러한 광범위한 조건화 매개변수 세트는 생성된 시나리오에 대한 정밀한 제어를 가능하게 하여 일반적인 운전 조건과 강력한 시스템 평가에 중요한 엣지 케이스 시나리오를 모두 시뮬레이션할 수 있도록 합니다.

유연성과 상호 운용성을 보장하기 위해 GAIA-2는 CLIP 임베딩 및 주행 특정 잠재 표현을 생성하도록 훈련된 독점 모델을 포함한 외부 잠재 공간에 대한 조건화를 가능하게 합니다. 이 기능은 장면 내용에 대한 의미론적 제어를 허용하고 다운스트림 계획 또는 인식 모듈과의 통합을 용이하게 합니다. GAIA-2는 또한 처음부터 생성, 과거 맥락에서 예측, 인페인팅을 통한 선택적 내용 편집을 포함한 여러 비디오 생성 모드를 지원합니다.

이러한 기능들을 단일 프레임워크에 통합함으로써 GAIA-2는 자율 주행을 위한 비디오 생성 세계 모델에서 새로운 기준을 제시합니다. 이는 다양한 조건에서 상세하고 제어 가능하며 현실적인 시뮬레이션을 가능하게 하여 자율 주행 시스템의 확장 가능한 훈련 및 강력한 평가를 위한 강력한 도구를 제공합니다.

![](./figure/2.png)

Figure 2: GAIA-2 world model. The architecture schematic shows full surround camera views independently encoded by a video tokenizer. The combined multi-camera latents are used as past context and linearly interpolated with sampled noise. We add camera parameters along with positional encodings before passing the latents through a space-time factorized transformer. Conditioning is implemented via cross-attention and adaptive layer norm. The latents are denoised into future latents, conditioned on various inputs, including actions, 3D bounding boxes, metadata, and scenario embeddings. The denoised latents are then decoded back to pixel space by the video tokenizer. Below, we depict the various tasks GAIA-2 is trained on, including from scratch generations, context rollouts, and spatial inpainting. At inference, we can generate beyond the window duration GAIA-2 was trained on by taking overlapping strides to generate new frames conditioned on previously generated frames.

## 2 모델

GAIA-2는 구조화된 조건화, 다중 카메라 일관성, 높은 시공간 해상도를 갖춘 서라운드 뷰 비디오 생성 세계 모델입니다. 아키텍처([그림 2](https://arxiv.org/html/2503.20523v1#S1.F2))는 비디오 토크나이저와 잠재 세계 모델의 두 가지 주요 구성 요소로 구성됩니다. 이 모듈들은 함께 GAIA-2가 풍부한 조건부 제어를 통해 여러 시점에서 사실적이고 의미론적으로 일관된 비디오를 생성할 수 있도록 합니다.

비디오 토크나이저는 원본 고해상도 비디오를 압축하고, 의미론적 및 시간적 구조를 보존하면서 압축된 연속 잠재 공간으로 변환합니다. 이 압축된 표현은 규모에 따른 효율적인 학습 및 생성을 가능하게 합니다. 그런 다음 세계 모델은 과거 잠재 상태, 행동 및 다양한 도메인 특정 제어 신호를 기반으로 조건화된 미래 잠재 상태를 예측하는 것을 학습합니다. 또한 처음부터 새로운 상태를 생성하고 인페인팅을 통해 비디오 내용을 수정하는 데 사용될 수 있습니다. 예측된 잠재 상태는 비디오 토크나이저 디코더를 사용하여 픽셀 공간으로 다시 디코딩됩니다.

대부분의 잠재 확산 모델과 달리, GAIA-2는 훨씬 더 높은 공간 압축률(예: 32times 대 일반적인 8times)을 사용합니다. 이는 잠재 공간의 채널 차원 증가(예: 16채널 대신 64채널)로 상쇄됩니다. 이는 더 적지만 의미론적으로 더 풍부한 잠재 토큰을 생성합니다. 결과적인 이점은 두 가지입니다. (1) 더 짧은 잠재 시퀀스는 더 빠른 추론과 더 나은 메모리 효율성을 가능하게 하며, (2) 모델은 비디오 내용과 시간 역학을 포착하는 능력을 향상시킵니다. 이 매개변수화 전략은 압축 잠재 표현에 대한 이전 작업 [^4]에서 영감을 받았습니다.

이전 모델인 GAIA-1 [^3]이 이산 잠재 변수에 의존했던 것과 달리, GAIA-2는 연속 잠재 공간을 사용하여 시간적 부드러움과 재구성 충실도를 향상시킵니다. 또한 GAIA-2는 자율 주행 차량 행동, 동적 에이전트 상태(예: 3D 바운딩 박스), 구조화된 메타데이터, CLIP 및 시나리오 임베딩, 카메라 기하학을 지원하는 유연한 조건화 인터페이스를 도입합니다. 이 설계는 생성 중에 장면 의미 및 에이전트 행동에 대한 강력한 제어를 가능하게 하는 동시에 교차 뷰 일관성 및 시간적 일관성을 보장합니다.

### 2.1 Video Tokenizer

The video tokenizer compresses pixel-space video into a compact latent space that is continuous and semantically structured. It is composed of a space-time factorized transformer with an asymmetric encoder-decoder architecture (with 85M and 200M parameters, respectively). The encoder extracts spatiotemporally downsampled latents that are temporally independent; the decoder reconstructs full-frame video from these latents using temporal context for temporal consistency.

![](./figure/3.png)

Figure 3: GAIA-2 video tokenizer. The architecture schematic depicts input frames spatiotemporally downsampled into temporally independent latents. A high spatial compression rate is compensated for by an increased latent channel dimension. Latents are decoded back to video frames, leveraging full spatiotemporal attention to ensure temporal consistency. To autoencode long sequences, we employ a rolling inference mechanism where current latents are decoded using past and future context in a sliding window fashion. The examples show input frames, decoded frames, and a visualization of the latent space. The first 3 principal components of the latent features are mapped to RGB values. Notably, the latent space is semantically stable across frames and across samples.

#### 2.1.1 인코더

입력 비디오 $({\mathbf{i}}{1},...,{\mathbf{i}}{T_{v}})$가 주어지면, 인코더 $e_{\phi}$는 잠재 토큰 $({\mathbf{z}}{1},...,{\mathbf{z}}{T_{L}})=e_{\phi}({\mathbf{i}}{1},...,{%

\mathbf{i}}{T_{v}})$를 계산합니다. 여기서 $T_{v}$는 비디오 프레임 수에 해당하고 $T_{L}$은 잠재 토큰 수에 해당합니다. 비디오 프레임의 공간 해상도를 $H_{v}\times W_{v}$로, 잠재 토큰의 공간 해상도를 HtimesW로 나타냅니다. 인코더는 공간적으로 H_v/H=32배, 시간적으로 T_v/T_L=8배 다운샘플링합니다. 잠재 차원은 L=64이며, 이는 (T_v,H_v,W_v)=(24,448,960) 및 $(T_{L},H,W)=(3,14,30)$일 때 총 압축률이 $384$($\frac{T_{v}\times H_{v}\times W_{v}\times 3}{T_{L}\times H\times W\times L}=384)$가 됩니다.

이 다운샘플링은 입력 프레임을 시간적으로 2times 스트라이딩하고 다음 모듈을 통해 달성됩니다.

1. 보폭 2times8times8 (시간, 높이, 너비)을 가진 다운샘플링 컨볼루션 블록, 이어서 보폭 2times2times2 (둘 다 임베딩 차원 512에서 작동)를 가진 또 다른 다운샘플링 컨볼루션 블록.
    
2. 차원 512 및 16개 헤드를 가진 24개의 공간 트랜스포머 블록 시리즈.
    
3. 보폭 1times2times2를 가진 최종 컨볼루션, 이어서 잠재 토큰에 대한 가우스 분포를 모델링하기 위해 2L 채널로 선형 투영. 잠재 차원은 인코더가 가우스 분포의 평균과 표준 편차를 예측하므로 두 배가 된다는 점에 유의하십시오. 훈련 및 추론 중에 잠재 토큰은 이 결과 분포에서 샘플링됩니다.
    

#### 2.1.2 디코더

디코더 아키텍처는 다음과 같습니다.

1. 잠재 차원에서 임베딩 차원으로의 선형 투영, 이어서 보폭 1times2times2를 가진 첫 번째 업샘플링 컨볼루션 블록(업샘플링은 깊이-공간 모듈 [^5]로 달성됨).
    
2. 차원 512 및 16개 헤드를 가진 16개의 시공간 분해 트랜스포머 블록 시리즈.
    
3. 보폭 2times2times2를 가진 업샘플링 컨볼루션 블록, 이어서 차원 512 및 16개 헤드를 가진 8개의 시공간 분해 트랜스포머 블록.
    
4. 보폭 2times8times8 및 픽셀 RGB 채널에 해당하는 차원 3을 가진 최종 업샘플링 컨볼루션 블록.
    

인코더와 디코더의 주요 차이점은 인코더가 8개의 연속적인 비디오 프레임을 단일 시간 잠재로 독립적으로 매핑하는 반면, 디코더는 T_L=3개의 시간 잠재를 T_v=24개의 비디오 프레임으로 공동으로 디코딩하여 시간적 일관성을 유지한다는 것입니다. 추론 중에는 비디오 프레임이 슬라이딩 윈도우를 사용하여 디코딩됩니다. 로직은 [그림 3](https://arxiv.org/html/2503.20523v1#S2.F3)의 '훈련' 및 '추론' 다이어그램에 표시됩니다.

#### 2.1.3 훈련 손실

비디오 토크나이저는 픽셀 재구성 및 잠재 공간 손실의 조합으로 훈련됩니다.

- L_1 , L_2 및 지각 손실 [^6]을 사용한 이미지 재구성.
    
- 사전 훈련된 표현과의 의미론적 정렬을 장려하는 코사인 유사성 손실을 통한 잠재 특징에 대한 DINO [^7] 증류. 비디오 프레임은 DINO 모델로 인코딩되고 시간적으로 선형 보간으로 다운샘플링되어 잠재 특징의 차원과 일치합니다.
    
- 잠재 공간을 정규화하기 위한 표준 가우스 분포 [^1]에 대한 Kullback-Leibler 발산 손실.
    

시각적 품질을 향상시키기 위해, 3D 컨볼루션 디스크리미네이터와 이미지 재구성 손실을 사용하여 GAN 손실 [^8]로 디코더를 추가로 미세 조정하면서 인코더는 고정 상태로 유지했습니다. 디스크리미네이터는 기본 채널 64, 시간 및 공간에서 보폭 2, 3D 블러 풀링 [^9], 채널 승수 [2, 4, 8, 8], 3D 인스턴스 정규화 및 슬로프 0.2의 LeakyReLU를 가진 일련의 잔류 3D 컨볼루션 블록입니다. 스펙트럼 정규화 [^10]와 softplus 활성화 함수로 구현된 원본 GAN 손실을 사용합니다.

### 2.2 세계 모델

잠재 세계 모델은 과거 잠재, 행동 및 풍부한 조건화 입력을 기반으로 미래 잠재 상태를 예측합니다. 이 모델은 8.4B 매개변수를 가진 시공간 분해 트랜스포머로 구현되었으며 안정성 및 샘플 효율성을 위해 흐름 일치 [^11]를 사용하여 훈련되었습니다.

입력 잠재 mathbfx∗1:TinmathbbRTtimesNtimesHtimesWtimesL 이 주어졌을 때, T 는 시간 창을 나타내고 $ N 은카메라수를나타냅니다.입력잠재는각카메라뷰를인코더 e*{\phi} 로독립적으로인코딩하여얻습니다.각시간단계 t $ 에는 행동 벡터 mathbfa∗t 와 조건화 벡터 mathbfc∗t 도 제공합니다.

#### 2.2.1 아키텍처

세계 모델은 은닉 차원 C를 가진 시공간 분해 트랜스포머입니다. 각 동작 ${\mathbf{a}}_{t}$는 $\mathbb{R}^{C}$로 임베딩되고, 조건화 벡터 ${\mathbf{c}}_{t}$는 K가 조건화 변수의 수를 나타내는 $\mathbb{R}^{K\times C}$로 임베딩됩니다. 흐름 일치 시간 $\tau\in[0,1]$도 시노이달 인코딩 [^12]을 사용하여 $\mathbb{R}^{C}$로 매핑됩니다.

흐름 일치 시간 `τ`와 동작 `𝐚_t`는 적응형 레이어 정규화 [^13]를 통해 각 트랜스포머 블록에 주입되며, 다른 조건 변수 `𝐜_t`에는 교차 주의(cross-attention)를 사용합니다. 교차 주의보다 적응형 레이어 정규화를 사용할 때 동작 조건이 더 정확하다는 것을 발견했습니다. 동작은 모든 공간 토큰에 영향을 미치기 때문에 적응형 레이어 정규화는 학습 가능한 주의 메커니즘에 의존할 필요 없이 명시적인 정보 게이트웨이를 제공합니다.

위치 인코딩과 관련하여, 우리는 독립적으로 인코딩합니다. (i) 정현파 임베딩을 사용한 공간 토큰 위치, (ii) 작은 MLP가 뒤따르는 정현파 임베딩을 사용한 카메라 타임스탬프, (iii) 학습 가능한 선형 레이어를 사용한 카메라 지오메트리(왜곡, 내장 및 외장). 이 모든 위치 인코딩은 [^14]과 유사하게 각 트랜스포머 블록 시작 부분에 입력 잠재에 추가됩니다.

세계 모델은 은닉 차원 C=4096 및 32개 헤드를 가진 22개의 시공간 분해 트랜스포머 블록을 포함합니다. 각 트랜스포머 블록에는 공간 주의(공간 및 카메라에 대한), 시간 주의, 교차 주의 및 적응형 레이어 정규화를 가진 MLP 레이어가 포함됩니다. 훈련 안정성을 높이기 위해 각 주의 레이어 앞에 쿼리-키 정규화 [^15]를 사용합니다.

#### 2.2.2 손실

훈련 시에는 컨텍스트 프레임 수 $t\in{0,...,T-1}$를 무작위로 샘플링합니다(t=0은 처음부터 생성을 의미). 또한 미리 정의된 분포에 따라 흐름 일치 시간 $\tau\in[0,1]$를 샘플링합니다([섹션 2.2.4](https://arxiv.org/html/2503.20523v1#S2.SS2.SSS4)에서 자세한 내용 확인). 컨텍스트 잠재 ${\mathbf{x}}_{1:t}$는 변경되지 않고, 미래 잠재 ${\mathbf{x}}_{t+1:T}$는 무작위 가우스 노이즈 ${\bm{\epsilon}}_{t+1:T}\sim\mathcal{N}({\bm{0}},{\bm{I}})$와 선형적으로 보간됩니다.

|   |   |   |   |
|---|---|---|---|
||_{t+1:T}^{\tau}=\tau{\mathbf{x}}_{t+1:T}+(1-\tau){\bm{\epsilon}}_{% t+1:T}||(1)|

속도 목표 벡터 ${\mathbf{v}}_{t+1:T}$는 목표 잠재와 무작위 노이즈의 차이입니다:

|   |   |   |   |
|---|---|---|---|
||_{t+1:T}={\mathbf{x}}_{t+1:T}-{\bm{\epsilon}}_{t+1:T}||(2)|

세계 모델 $f_{\theta}$는 컨텍스트 잠재 $ \\mathbf{x}*{1:t} $, 행동 $\\mathbf{a}*{1:T}$ 및 조건화 변수 mathbfc∗1:T 에 따라 목표 속도 mathbfv∗t+1:T 를 예측합니다.

|   |   |   |   |
|---|---|---|---|
||_{t+1:T}=f_{\theta}({\mathbf{x}}_{t+1:T}^{\tau}\|{\mathbf{x}}% _{1:t},{\mathbf{a}}_{1:T},{\mathbf{c}}_{1:T},\tau)||(3)|

모델은 예측된 속도와 목표 속도 간의 L_2 손실로 훈련됩니다.

|   |   |   |   |
|---|---|---|---|
||_{\text{world-model}}=\mathbb{E}_{t\sim P(t),\tau\sim p(\tau)}[\|{% \mathbf{v}}_{t+1:T}-\hat{{\mathbf{v}}}_{t+1:T}\|^{2}]||(4)|

여기서 $t\sim P(t)$는 컨텍스트 프레임의 분포를 나타내고 $\tau\sim p(\tau)$는 흐름 일치 시간의 분포를 나타냅니다.

#### 2.2.3 조건화

GAIA-2는 생성된 장면에 대한 정밀 제어를 가능하게 하는 풍부하고 구조화된 조건화 입력을 지원합니다. 이러한 입력에는 자율 주행 차량 동작, 동적 에이전트 속성, 장면 수준 메타데이터, 카메라 구성, 타임스탬프 임베딩, 그리고 CLIP 또는 독점 시나리오 임베딩과 같은 외부 잠재 표현이 포함됩니다. 조건화 메커니즘은 적응형 레이어 정규화(동작용), 가산 모듈(카메라 지오메트리 및 타임스탬프용), 교차 주의(다른 모든 변수용)의 조합을 통해 세계 모델에 통합됩니다.

##### 카메라 매개변수.

우리는 내장, 외장 및 왜곡에 대한 별도의 임베딩을 계산하고, 이들을 합산하여 통합된 카메라 인코딩을 형성합니다. 내장 매개변수의 경우, 내장 행렬에서 초점 거리와 주점 좌표를 추출하고, 정규화한 다음, 공유 잠재 공간으로 투영합니다. 외장 및 왜곡 계수는 각각의 인코더를 통해 유사하게 처리되어 압축된 표현을 산출합니다. 이 구성은 모델이 실제 카메라 변화를 효과적으로 통합할 수 있도록 합니다. [그림 5](https://arxiv.org/html/2503.20523v1#S5.F5)는 훈련 데이터 세트에서 가장 흔한 세 가지 카메라 리그 구성의 상위 세 가지를 보여줍니다.

##### 비디오 주파수.

가변 비디오 프레임 속도를 설명하기 위해 GAIA-2는 타임스탬프 조건화를 사용합니다. 각 타임스탬프는 (i) 현재 시간에 상대적으로 정규화되고 [−1,1] 범위로 스케일링됩니다. (ii) 시노이달 함수(푸리에 특징 인코딩)를 사용하여 변환됩니다. (iii) 공유 잠재 공간에 벡터를 생성하기 위해 MLP를 통과합니다. 이 인코딩은 저주파 및 고주파 시간 변화를 모두 포착하며, 모델이 다른 속도로 기록된 비디오에 대해 효과적으로 추론할 수 있도록 합니다.

##### 행동.

자율 주행 차량의 행동은 속도와 곡률로 매개변수화됩니다. 이러한 양은 여러 차수 크기에 걸쳐 있으므로, 정규화를 위해 대칭 로그 변환 `symlog` [^16]을 사용합니다.

|   |   |   |   |
|---|---|---|---|
||||(5)|

여기서 y는 속도 또는 곡률을 나타내고 s는 스케일 인자입니다. 곡률(단위: m−1, 범위: 0.0001 ~ 0.1)의 경우 낮은 값을 증폭하기 위해 s=1000을 사용합니다. 속도(단위: m/s, 범위: 00 ~ 75)의 경우 값을 km/h로 표현하기 위해 s=3.6을 사용합니다. 결과는 $[-1,1]$로 스케일링된 압축 표현으로 훈련 안정성을 향상시킵니다.

##### 동적 에이전트.

주변 에이전트를 표현하기 위해, 우리 데이터셋에서 재훈련된 3D 객체 감지기 [^17]에 의해 예측된 3D 바운딩 박스를 사용합니다. 각 박스는 에이전트의 3D 위치, 방향, 크기, 및 카테고리를 인코딩합니다. 3D 박스는 2D 이미지 평면에 투영되고 정규화되어 f_iinmathbbRTtimesNtimesBtimes13 조건화 특징을 산출합니다. 여기서 T는 시간 잠재의 수, N은 카메라 수, B는 최대 3D 바운딩 박스 수입니다(필요에 따라 제로 패딩). 각 특징 차원은 독립적으로 임베딩되고 단일 레이어 MLP를 통해 집계됩니다.

모델의 견고성과 일반화 가능성을 높이기 위해 훈련 중 특징 차원과 인스턴스 수준 모두에 드롭아웃을 구현합니다. 특히, 특징 차원은 p=0.3의 확률로 드롭아웃되어 추론 시 모델이 불완전한 정보로 작동할 수 있도록 합니다. 이 설정은 예를 들어, 인스턴스의 3D 위치를 지정하지 않고 2D 투영 박스에 대한 조건화를 허용하거나, 다른 조건에 따라 가장 그럴듯한 방향을 모델이 예측하도록 방향을 생략할 수 있도록 합니다.

인스턴스 수준에서 각 카메라에 대해 프레임 `t` ( tin1,...,T )를 샘플링하고 감지된 인스턴스 수 `N_instances`를 계산합니다. 그런 다음 조건화할 인스턴스 수 `n` ( nin0,...,min(B,N_instances) )를 샘플링하고 이 샘플 크기를 초과하는 인스턴스에 드롭아웃을 적용합니다. 이를 통해 모델은 추론 중에 가변적인 수의 동적 에이전트에 적응할 수 있습니다. `n`을 시간 동안 일정하게 유지하면서 인스턴스 추적은 사용하지 않습니다. 모델이 프레임 간의 조건화 특징이 동일한 인스턴스에 속하는지 또는 다른 인스턴스에 속하는지를 독립적으로 결정하도록 허용합니다.

##### 메타데이터.

메타데이터 기능은 범주형이며 전용 학습 가능 임베딩 레이어를 사용하여 임베딩됩니다. 여기에는 다음이 포함됩니다. 국가, 날씨, 시간; 속도 제한; 차선 수 및 유형(예: 버스, 자전거); 횡단보도, 교통 신호 및 그 상태; 일방통행 도로 표시 및 교차로 유형. 이러한 임베딩을 통해 GAIA-2는 장면 수준 기능과 행동에 미치는 영향 간의 미묘한 관계를 학습할 수 있어 일반적인 시나리오와 드문 시나리오를 모두 시뮬레이션할 수 있습니다.

##### CLIP 임베딩.

의미론적 장면 조건화를 가능하게 하기 위해, GAIA-2는 CLIP 임베딩 [^18]에 대한 조건화를 지원합니다. 훈련 중에는 비디오 프레임에 대한 이미지 인코더를 사용하여 CLIP 특징을 추출합니다. 추론 시에는 이러한 특징을 자연어 프롬프트에서 얻은 CLIP 텍스트 인코더 출력으로 대체할 수 있습니다. 모든 CLIP 임베딩은 학습 가능한 선형 투영을 사용하여 모델의 잠재 공간으로 투영됩니다. 이를 통해 자연어 또는 시각적 유사성을 통해 장면 의미에 대한 제로 샷 제어가 가능합니다.

##### 시나리오 임베딩.

GAIA-2는 주행 특정 정보를 인코딩하도록 훈련된 내부 독점 모델에서 얻은 시나리오 임베딩을 통해 조건화될 수도 있습니다. 이러한 임베딩은 도로 레이아웃 및 에이전트 구성과 같은 자율 주행 차량 행동 및 장면 컨텍스트를 압축적으로 캡처합니다. 시나리오 벡터는 트랜스포머에 통합되기 전에 학습 가능한 선형 레이어를 통해 잠재 공간으로 투영됩니다. 이를 통해 압축된 추상 표현에서 고수준 시나리오 생성이 가능합니다.

#### 2.2.4 흐름 일치 시간 분포

흐름 일치 프레임워크에서 세계 모델을 훈련하는 데 중요한 요소는 흐름 일치 시간 tau의 분포입니다. 이 분포는 모델이 실제 잠재에 가까운 잠재 입력을 얼마나 자주 보는지 또는 크게 교란된 잠재 입력을 얼마나 자주 보는지를 결정합니다.

우리는 두 가지 모드가 있는 이봉성 로짓-정규 분포를 사용합니다.

- 주 모드: mu=0.5, sigma=1.4에 중심을 두고 확률 p=0.8로 샘플링됩니다. 이는 모델이 낮은 수준에서 중간 수준의 노이즈로 학습하도록 편향시킵니다. 경험적으로, 이는 유용한 그라디언트 학습을 장려하는데, 심지어 적은 양의 노이즈도 고용량 잠재를 크게 교란할 수 있기 때문입니다.
    
- 보조 모드: mu=−3.0, sigma=1.0에 중심을 두고 확률 p=0.2로 샘플링됩니다. 이는 모델이 공간 구조와 자율 주행 차량 움직임이나 객체 궤적과 같은 저수준 역학을 학습하는 데 도움이 되도록 tau=0 주변의 거의 순수 노이즈 영역에 훈련을 집중시킵니다.
    

이 이봉성 전략은 낮은 노이즈 및 높은 노이즈 레짐 모두에서 훈련이 효과적이도록 보장하여 일반화 및 샘플 품질을 향상시킵니다.

또한, 입력 잠재 ${\mathbf{x}}_{t}$는 [^19]에 따라 평균 $\mu_{x}$와 표준 편차 $\sigma_{x}$로 정규화되어 그 크기가 추가된 가우스 노이즈의 크기와 일치하도록 합니다. 이는 신호와 교란 간의 스케일 불일치를 방지하여 훈련 역학을 저하시킬 수 있습니다.

## 3 데이터

GAIA-2는 자율 주행을 위한 비디오 생성의 다양한 요구를 지원하기 위해 특별히 큐레이션된 대규모 내부 데이터 세트로 훈련되었습니다. 이 데이터 세트는 약 2,500만 개의 비디오 시퀀스로 구성되며, 각 시퀀스는 2초 길이로 2019년에서 2024년 사이에 수집되었습니다. 지리적 및 환경적으로 다양한 운전 조건을 보장하기 위해 영국, 미국, 독일 세 국가에 걸쳐 녹화가 이루어졌습니다.

실제 자율 주행의 복잡성을 포착하기 위해 데이터 수집에는 세 가지 다른 자동차 모델과 두 가지 밴 유형을 포함한 여러 차량 플랫폼이 사용되었습니다. 각 차량에는 포괄적인 360도 서라운드 뷰 커버리지를 제공하도록 구성된 5개 또는 6개의 카메라가 장착되었습니다. 카메라 시스템은 20Hz, 25Hz, 30Hz의 다양한 캡처 주파수를 사용하여 다양한 시간 해상도를 도입했습니다. 이러한 가변성은 실제 자율 주행 차량에 배포된 센서 설정의 이질적인 특성을 반영하며, GAIA-2가 다양한 입력 속도 및 하드웨어 사양에 걸쳐 일반화할 수 있도록 지원합니다.

데이터셋의 중요한 특징은 데이터 수집 기간 동안 카메라 배치의 가변성입니다. 시간이 지남에 따라 카메라의 위치와 보정이 플랫폼에 따라 조정되어 광범위한 공간 구성을 도입했습니다. 이러한 다양성은 자율 주행 도메인에서 확장 가능한 합성 데이터 생성을 위한 핵심 요구 사항인 다양한 카메라 리그에 걸쳐 일반화하기 위한 강력한 훈련 신호를 제공합니다.

이 데이터 세트는 또한 다양한 날씨 조건, 시간, 도로 유형 및 교통 환경을 포함한 광범위한 운전 시나리오를 포괄합니다. 이러한 복잡성 전반에 걸쳐 커버리지를 보장하기 위해, 우리는 개별 특징뿐만 아니라 공동 확률 분포에 대해서도 훈련 데이터의 균형을 명시적으로 맞춥니다. 이 접근 방식은 특정 지리적 위치에서의 특정 조명 및 날씨 조건 또는 특정 도로 유형에 고유한 행동과 같은 현실적인 동시 발생을 모델링하여 더 대표적인 학습 신호를 가능하게 합니다. 중복된 훈련 샘플을 방지하기 위해, 우리는 선택된 시퀀스 간에 최소 시간적 간격을 강제하여 중복 위험을 줄이면서 자연스러운 분포를 유지합니다.

평가를 위해 지리적으로 보류된 검증 전략을 구현합니다. 특정 검증 지오펜스는 훈련 세트에서 특정 영역을 완전히 제외하도록 정의됩니다. 이는 모델 평가가 보이지 않는 위치에서 수행되도록 보장하여 다양한 환경에 걸친 일반화 성능에 대한 더 엄격한 평가를 제공합니다.

요약하면, 이 데이터 세트는 GAIA-2 훈련을 위한 견고한 기반을 제공합니다. 광범위한 시간 및 공간 범위, 차량 및 카메라 구성의 다양성, 그리고 원칙적인 검증 설정은 다양하고 실제적인 조건에서 현실적이고 제어 가능한 운전 비디오를 생성할 수 있는 생성 세계 모델을 개발하는 데 특히 적합합니다.

## 4 훈련 절차

이 섹션에서는 GAIA-2의 두 구성 요소, 즉 비디오 토크나이저와 세계 모델에 대한 훈련 절차를 설명합니다. 각 구성 요소는 대규모 컴퓨팅 인프라와 맞춤형 손실 구성을 사용하여 독립적으로 훈련되어 각 목표를 최적화합니다.

##### 비디오 토크나이저.

비디오 토크나이저는 128개의 H100 GPU를 사용하여 128 배치 크기로 300scriptstyle,000 단계 동안 훈련되었습니다. 입력 시퀀스는 원래 캡처 주파수(20, 25 또는 30 Hz)로 샘플링된 T_v=24개의 비디오 프레임으로 구성되었습니다. 448times960 크기의 무작위 공간 크롭이 프레임에서 추출되었습니다. 각 훈련 샘플에 대해 사용 가능한 N=5개의 관점 중에서 카메라 뷰가 무작위로 선택되었습니다. 특히 각 카메라 스트림은 독립적으로 인코딩되었습니다.

토크나이저는 시간적으로 8times, 공간적으로 32times 다운샘플링을 수행하여 잠재 차원 L=64를 가진 압축된 표현을 생성하며, 효과적인 총 압축률은 약 400입니다(frac24times448times960times33times14times30times64simeq400).

토크나이저의 손실 함수는 이미지 재구성, 지각 및 의미 정렬 항의 조합으로 구성됩니다. (1) 잠재 공간에서 0.1의 가중치를 가진 DINO v2 (Large) [^7] 증류. (2) 부드러움을 장려하기 위해 $1\mathrm{e}{-6}$의 낮은 가중치를 가진 잠재 분포와 단위 가우스 사이의 KL 발산 [^1]. (3) 픽셀 수준 손실: L_1 손실 (가중치 0.2), L_2 손실 (가중치 2.0), 그리고 LPIPS 지각 손실 [^20] (가중치 0.1).

토크나이저 매개변수 `φ`의 지수 이동 평균(EMA)은 훈련 내내 유지되었으며, 감쇠 인자는 0.9999이고 매 훈련 단계마다 업데이트되었습니다. EMA 가중치는 추론에 사용되었습니다.

훈련은 AdamW를 사용하여 최적화되었습니다. 기본 학습률 $1\mathrm{e}{-4}$까지 2scriptstyle,500 웜업 단계, 최종 학습률 $1\mathrm{e}{-5}$까지 5scriptstyle,000 쿨다운 단계, Adam 베타 [0.9,0.95], 가중치 감쇠 0.1, 그라디언트 클리핑 1.0을 사용했습니다. 초기 훈련 후, 토크나이저 디코더는 이전 재구성 손실과 결합하여 GAN 손실(가중치 0.1)을 사용하여 추가 20scriptstyle,000 단계 동안 미세 조정되었습니다. 디스크리미네이터는 $1\mathrm{e}{-5}$의 학습률로 최적화되었습니다.

##### 세계 모델.

잠재 세계 모델은 256개의 H100 GPU에서 256 배치 크기로 460scriptstyle,000 단계 동안 훈련되었습니다. 입력은 원래 캡처 주파수(20, 25 또는 30 Hz), 공간 해상도 448times960 및 N=5개 카메라에 걸쳐 48개 비디오 프레임으로 구성되었습니다. 이 비디오들을 잠재 공간으로 인코딩한 후, 이는 TtimesNtimesHtimesW=6times5times14times30=12scriptstyle,600개의 입력 토큰에 해당합니다.

일반화를 장려하기 위해, 우리는 다음과 같이 다양한 훈련 작업을 샘플링했습니다: 70%는 처음부터 생성, 20%는 문맥적 예측, 10%는 공간 인페인팅. 모델을 정규화하고 분류기 없는 안내를 가능하게 하기 위해, 조건화 변수는 무작위로 제거되었습니다. 각 개별 조건화 변수는 80% 확률로 독립적으로 제거되었고, 모두 10% 확률로 동시에 제거되었습니다.

부분 가시성에 대한 견고성을 높이기 위해 입력 카메라 뷰는 10% 확률로 무작위로 제거되었습니다. 잠재 토큰은 토크나이저 훈련 중에 경험적으로 결정된 고정된 평균 mu_x=0.0과 표준 편차 sigma_x=0.32를 사용하여 정규화되었습니다.

토크나이저와 마찬가지로, 세계 모델 매개변수 theta의 EMA는 감쇠 인자 0.9999와 매 훈련 단계마다 업데이트를 통해 유지되었습니다. EMA 가중치는 추론에 사용되었습니다. 최적화 도구는 AdamW를 사용했으며, 초기 학습률 $5\mathrm{e}{-5}$까지 2scriptstyle,500 웜업 단계와 전체 훈련 기간 동안 코사인 감쇠를 통해 최종 학습률 $6.5\mathrm{e}{-6}$으로 조정되었습니다. Adam 베타 [0.9,0.99], 가중치 감쇠 0.1, 그라디언트 클리핑 1.0을 사용했습니다.

## 5 추론

GAIA-2 모델은 잠재 공간에서 작동하는 공유된 노이즈 제거 프로세스에 이어 비디오 토크나이저를 통해 픽셀 공간으로 디코딩되는 다양한 추론 작업을 지원하여 비디오 생성 시나리오 전반에 걸쳐 유연성과 제어 가능성을 보여줍니다.

![](./figure/4.png)

그림 4: CLIP 조건화. CLIP 텍스트 인코더를 통해 인코딩된 텍스트 프롬프트로 GAIA-2 생성을 조건화하면 환경 특징에 대한 미묘한 제어가 가능합니다.

##### 추론 작업.

우리는 모델의 고유한 기능을 보여주는 네 가지 주요 추론 모드를 고려합니다.

- '처음부터 생성'은 순수 가우스 노이즈를 샘플링하고 [섹션 2.2.3](https://arxiv.org/html/2503.20523v1#S2.SS2.SSS3)에서 정의된 조건화 변수로부터의 안내를 통해 노이즈를 제거하는 것을 포함합니다. 결과 잠재는 비디오 토크나이저 디코더를 사용하여 비디오 프레임으로 디코딩되며, [그림 3](https://arxiv.org/html/2503.20523v1#S2.F3)에 설명된 롤링 윈도우 디코딩 메커니즘을 통해 시간적으로 일관된 출력이 생성됩니다.
    
- 자기회귀 예측은 과거 컨텍스트 잠재 시퀀스로부터 미래 잠재를 예측할 수 있게 합니다. k=3개의 시간 잠재로 구성된 초기 컨텍스트 윈도우가 주어지면, 모델은 다음 잠재 세트를 예측하고, 이를 컨텍스트에 추가한 다음, 슬라이딩 윈도우를 사용하여 프로세스를 반복합니다. 이 접근 방식은 자율 주행 차량 움직임과 같은 조건화 신호를 통합하면서 장기적인 롤아웃을 가능하게 합니다. 예시는 [그림 9](https://arxiv.org/html/2503.20523v1#S6.F9)에 나와 있습니다.
    
- 인페인팅은 비디오 콘텐츠의 선택적 수정을 허용합니다. 잠재 입력에 시공간 마스크를 적용하고, 마스크된 영역은 조건부 노이즈 제거를 통해 재생성됩니다. 동적 에이전트 조건화(예: 에이전트 위치)로부터의 선택적 안내는 마스크된 영역 내에서 생성을 유도할 수 있습니다. 예시는 [그림 12](https://arxiv.org/html/2503.20523v1#S6.F12)에 나와 있습니다.
    
- 장면 편집은 실제 비디오에서 추출한 잠재를 부분적으로 노이즈를 추가한 후, 변경된 조건화로 노이즈를 제거하는 방식으로 이루어집니다. 이를 통해 날씨, 시간, 도로 레이아웃과 같은 목표 지향적인 의미론적 또는 스타일적 변환을 전체 장면을 다시 생성하지 않고도 가능하게 합니다. [그림 6](https://arxiv.org/html/2503.20523v1#S5.F6)은 이 기능을 보여줍니다.
    

이러한 모드들은 GAIA-2가 노이즈, 컨텍스트 또는 기존 비디오에서 시작하든 다양한 장면 조작 작업을 위한 범용 시뮬레이터 역할을 할 수 있음을 보여줍니다.

##### 추론 노이즈 스케줄.

모든 추론 작업에 대해 [^14]에서 소개된 선형-2차 노이즈 스케줄을 채택합니다. 이 스케줄은 거친 장면 레이아웃 및 움직임 패턴을 포착하는 데 효과적인 선형 간격 노이즈 수준으로 시작합니다. 후기 단계에서는 고주파 시각적 세부 사항을 보다 효율적으로 미세 조정할 수 있는 2차 간격 단계로 전환됩니다. 이 하이브리드 접근 방식은 생성 품질과 계산 효율성을 모두 향상시킵니다. 우리 실험에서는 고정된 50개의 노이즈 제거 단계를 사용합니다.

##### 분류기 없는 안내 (Classifier-Free Guidance).

분류기 없는 안내(CFG)는 추론 중에 기본적으로 사용되지 않습니다. 그러나 희귀한 엣지 케이스 또는 특이한 에이전트 구성(예: 그림 [11](https://arxiv.org/html/2503.20523v1#S6.F11))과 같이 도전적이거나 분포를 벗어난 시나리오의 경우, 장면의 복잡성에 따라 2에서 20까지의 안내 스케일로 CFG를 활성화합니다.

동적 에이전트 조건화가 포함된 시나리오에서, 에이전트 특정 영역과 관련된 잠재 토큰이 사전에 알려진 경우, 공간 선택적 CFG를 적용합니다. 이 경우, 조건화에 의해 영향을 받는 공간 위치(예: 3D 바운딩 박스)에만 안내가 적용되어 장면의 나머지 부분에 불필요하게 영향을 주지 않으면서 대상 영역에서 생성 품질을 향상시킵니다. 이 목표 지향적 접근 방식은 전역적 일관성을 유지하면서 장면 요소에 대한 더 정밀한 제어를 가능하게 합니다.

![](./figure/5.png)

그림 5: 멀티-리그 비디오 생성. GAIA-2는 다양한 차량 플랫폼 및 카메라 설정을 지원하며, 공간 및 시간적 일관성을 유지합니다. 표시된 두 예시에는 스포츠카, SUV 및 대형 밴의 카메라 배열이 포함됩니다.

![](./figure/6.png)

그림 6: 부분 노이즈 제거를 통한 증강. 비디오 프레임을 부분적으로 노이즈 제거하여, GAIA-2는 원본의 의미론적 내용과 자율 주행 차량 행동을 보존하면서 실제 비디오를 날씨 및 시간과 같은 다양한 환경 설정의 다양한 버전으로 변환합니다.

![](./figure/7.png)

Figure 7: Scenario embedding conditioning. Scenario embeddings from a proprietary driving model enable GAIA-2 to generate diverse yet semantically consistent variations from real-world scenarios. For each scenario type the top row shows the real data the embedding is derived from, and the bottom three rows show synthetic variants of it, guided by the embedding as conditioning. We also optionally apply additional conditioning such as country, weather, time of day, or camera parameters to control diversity.

## 6 결과

우리는 GAIA-2를 정성적 예시와 정량적 지표를 통해 평가하여 그 생성 충실도, 제어 가능성, 그리고 자율 주행에서의 합성 데이터 생성 적합성을 강조합니다. 생성된 비디오 예시의 포괄적인 세트는 [https://wayve.ai/thinking/gaia-2](https://wayve.ai/thinking/gaia-2)에서 볼 수 있습니다.

##### 실제 데이터 증강.

GAIA-2는 실제 시퀀스의 시각적 및 상황별 다양화를 허용하는 다양한 기술을 통해 데이터 세트 증강을 가능하게 합니다.

### 6.1 정성적 예시

##### 다양한 시나리오 생성.

GAIA-2는 여러 국가, 날씨 조건, 시간대 및 도로 레이아웃에 걸쳐 매우 다양한 운전 시나리오 생성을 지원합니다. 메타데이터를 통한 구조화된 조건화 외에도 GAIA-2는 CLIP 임베딩에 의해 안내될 수 있으며, 레이블에 명시적으로 캡처되지 않은 장면 의미론에 대한 제어를 허용합니다. 여기에는 도시, 산악 또는 해안 장면과 같은 지리적 및 환경적 컨텍스트가 포함됩니다([그림 4](https://arxiv.org/html/2503.20523v1#S5.F4)).

게다가, 훈련 중 다양한 카메라 구성에 노출되었기 때문에 GAIA-2는 다양한 차량 구현에 걸쳐 운전 비디오를 시뮬레이션할 수 있습니다. 카메라 매개변수에 대한 조건화를 통해 스포츠카, SUV 및 대형 밴에 장착된 리그에 대한 시점 전반에 걸쳐 공간 및 시간적 일관성을 유지합니다([그림 5](https://arxiv.org/html/2503.20523v1#S5.F5)).

- **부분 노이즈 및 노이즈 제거**: 실제 비디오에서 파생된 잠재는 부분적으로 노이즈가 추가된 다음, 다른 날씨 또는 조명과 같은 변경된 조건에서 노이즈가 제거됩니다. 이 접근 방식은 의미와 자율 주행 차량 움직임을 보존하면서 다양한 시각적 출력을 산출합니다(그림 [6](https://arxiv.org/html/2503.20523v1#S5.F6)). 우리는 핵심 의미 및 기능 구성 요소를 원본대로 유지하면서 장면의 환경적 측면을 수정하는 능력을 보여줍니다. 본질적으로 단일 실제 예시를 여러 새로운 시나리오로 전환합니다. 동일한 자율 주행 차량 궤적이지만 햇빛, 비, 일몰, 밤, 눈 속에서 발생하도록 시각적으로 다양화됩니다.
    
- **시나리오 임베딩**: 독점 주행 모델에서 파생된 시나리오 임베딩은 GAIA-2에게 단일 실제 예시로부터 여러 그럴듯한 변형을 생성하는 능력을 부여하여 다양하면서도 상황적으로 일관된 합성 데이터의 많은 새로운 예시를 제공합니다([그림 7](https://arxiv.org/html/2503.20523v1#S5.F7)). 임베딩 공간이 주행 모델에 의해 의미 있게 조직화되어 있기 때문에, 우리는 임베딩 공간 내 위치(예: 가속, 감속 또는 다른 에이전트 추월)를 고려하여 의미론적으로 해석 가능한 시나리오를 안정적으로 생성할 수 있습니다. 추가적으로 환경 요인에 대한 조건화를 통해 다양한 조건에 걸쳐 커버리지를 증가시킴으로써 데이터셋을 합성적으로 확장할 수 있습니다. 다른 카메라 매개변수에 대한 조건화를 통해 어떤 주어진 차량 플랫폼에서도 코퍼스를 효과적으로 복제할 수 있습니다.
    
- **행동 기반 생성**: GAIA-2는 자율 주행 차량 행동 궤적만으로 새로운 장면을 합성할 수 있습니다. 속도 및 곡률 프로필이 주어지면, 교통 신호 변화, 제동 행동 또는 회전 기동과 같은 해당 역학과 일치하는 그럴듯한 시각적 컨텍스트를 생성합니다([그림 8](https://arxiv.org/html/2503.20523v1#S6.F8)). 우리는 세 가지 예시를 보여줍니다. (1) '정지 상태에서 시작'하는 속도 프로필에 대한 조건화를 통해 GAIA-2는 해당 행동에 맞는 그럴듯한 시각적 관찰을 생성합니다. 이 경우 영국 신호등이 빨간색과 노란색에서 초록색으로 바뀝니다. (2) '정지까지 천천히' 속도 궤적에 대한 조건화를 통해 GAIA-2는 자율 주행 차량이 런던 택시 뒤에서 속도를 늦추는 시나리오를 생성합니다. (3) 강한 왼쪽 곡률과 느린 속도 증가에 대한 조건화를 통해 GAIA-2는 미국 교차로에서 U턴을 생성합니다.
    

![](./figure/8.png)

그림 8: 행동 기반 생성. GAIA-2는 지정된 속도 및 곡률 프로필로부터 다양한 장면을 합성합니다. 각 시나리오는 원본 비디오 입력이 제거되었음에도 불구하고 상황적으로 그럴듯합니다.

##### 안전 필수 시나리오 생성.

GAIA-2는 실제 세계에서는 드물게 발생할 수 있는 드물지만 안전 필수적인 시나리오를 생성할 수 있습니다. 우리는 두 가지 유형의 안전 필수 상황을 고려합니다. 자율 주행 차량 에이전트에 의해 발생하는 상황과 다른 에이전트에 의해 발생하는 상황입니다.

- **자율 주행 차량 유도 시나리오**: 극단적이거나 안전하지 않은 자율 주행 차량 행동(예: 반대편 차선으로 갑작스러운 조향)에 대한 조건화를 통해 GAIA-2는 위험한 조건에서 자율 시스템의 탄력성을 테스트하는 데 중요한 현실적인 시나리오를 생성합니다([그림 9](https://arxiv.org/html/2503.20523v1#S6.F9) 및 [그림 10](https://arxiv.org/html/2503.20523v1#S6.F10)).
    
- **다른 에이전트 유도 시나리오**: 3D 바운딩 박스 조건화를 사용하여 GAIA-2는 다른 에이전트의 배치 및 움직임을 정밀하게 제어하여, 자율 시스템의 반응 능력을 테스트하는 데 중요한 공격적인 운전 행동, 비상 제동 또는 위험한 횡단과 관련된 시나리오를 생성합니다([그림 11](https://arxiv.org/html/2503.20523v1#S6.F11)).
    

![](./figure/9.png)

그림 9: 자율 주행 차량 유도 안전 필수 시나리오. 안전하지 않은 행동 조건화를 적용하여, GAIA-2는 마주오는 차량으로 조향하는 것과 같은 위험한 상황을 생성할 수 있습니다. 우리는 실제 비디오 프레임을 컨텍스트로 제공하고, GAIA-2는 새로운 행동 조건화가 주어지면 이러한 시작 프레임에서 롤아웃합니다. 이 예시에서는 우향 곡률을 조건화하여 자율 주행 차량의 조향을 제어합니다. 속도를 제거하여 GAIA-2가 여러 다양한 결과를 생성할 수 있도록 합니다. 자율 주행 차량이 마주오는 차량으로 급하게 방향을 틀면서 속도를 늦추는 방식에 주목하십시오. 마주오는 차량은 자율 주행 차량을 피하기 위해 방향을 바꿉니다.

![](./figure/10.png)

Figure 10: Extreme generalization behavior. When conditioned on high speed and strong curvature, GAIA-2 extrapolates off-road trajectories, capturing rare behaviors like driving into fields or forests.

![](./figure/11.png)

그림 11: 다른 에이전트 유도 안전 필수 시나리오. 3D 바운딩 박스 조건화는 동적 에이전트에 대한 정밀 제어를 가능하게 하여, 급제동, 추월, 교차로에서의 과속과 같은 위험한 행동을 시뮬레이션할 수 있습니다.

##### 인페인팅 (Inpainting).

GAIA-2는 시공간 인페인팅을 지원하여 선택적 콘텐츠 편집을 가능하게 합니다. 마스크된 영역과 해당 에이전트 수준 조건화가 제공되면, GAIA-2는 관련 없는 영역을 방해하지 않고 에이전트를 기존 컨텍스트에 원활하게 삽입합니다([그림 12](https://arxiv.org/html/2503.20523v1#S6.F12)).

![](./figure/12.png)

그림 12: 에이전트 인페인팅. GAIA-2는 3D 바운딩 박스 조건화에 따라 비디오의 마스크된 영역에 동적 에이전트를 삽입할 수 있습니다. 배경 일관성과 의미론적 사실성이 보존됩니다.

이러한 정성적 결과는 GAIA-2가 정밀하고 제어 가능한 비디오 생성을 수행할 수 있음을 강조하며, 자율 시스템을 위한 확장 가능한 시뮬레이션 및 다양한 시나리오 생성을 지원합니다.

### 6.2 지표

GAIA-2를 정량적으로 평가하기 위해, 시각적 충실도, 시간적 일관성, 조건화 정확도를 평가하는 일련의 지표를 사용합니다.

##### 시각적 충실도.

[^21]와 유사하게, 생성된 비디오와 실제 비디오의 분포 간 거리를 측정하기 위해 Frechet DINO Distance(FDD) 및 Frechet Inception Distance(FID) [^22]를 사용합니다. 특징 인코더는 DINOv2 [^7] ViT-L/14이며, 입력 이미지는 FID에서 InceptionV3 [^23]의 299times299에 비해 더 높은 해상도 448times952로 양선형 재스케일링됩니다. FDD가 FID보다 노이즈가 적고 훈련 후반에 포화된다는 것을 관찰했습니다.

##### 시간적 일관성.

세계 모델의 시간적 일관성을 평가하기 위해 Frechet Video Motion Distance (FVMD) [^24]를 사용합니다. 이 지표는 생성된 비디오와 실제 비디오의 명시적 키포인트 모션 특징 분포를 비교합니다. 우리는 이 지표가 FVD [^25]에 비해 시간적 일관성에 대한 인간의 선호도와 더 잘 일치한다는 것을 발견했습니다.

![](./figure/13.png)

그림 13: 검증 손실 및 기타 지표(n=1024 𝑛 n=1024 samples에서 평가). 검증 손실이 인간의 지각 선호도와 잘 상관 관계를 보인다는 것을 발견했습니다.

##### 동적 에이전트 조건화.

우리는 생성된 비디오에서 OneFormer [^26]를 통해 얻은 분할 마스크와 3D 바운딩 박스(우리의 조건부 신호)의 투영을 비교하여 동적 에이전트 충실도를 평가합니다. 그런 다음 클래스 기반 IoU(Intersection-over-Union)를 계산하여 모델이 생성 중에 동적 에이전트의 공간 및 범주적 사양을 얼마나 잘 준수하는지 측정합니다.

[그림 13](https://arxiv.org/html/2503.20523v1#S6.F13)에서 지표와 검증 손실을 보고합니다. 검증 손실 및 기타 지표는 n=1024 샘플로 처음부터 생성 작업에서 평가됩니다. 작은 샘플 크기로 인한 일부 노이즈에도 불구하고, 모든 지표는 훈련 내내 긍정적인 추세를 보입니다. 특히, 검증 손실은 인간 선호도와 가장 강한 상관 관계를 보여 지각 품질에 대한 귀중한 대리 지표가 됩니다.

## 7 관련 연구

##### 비디오 생성 모델.

비디오 생성 모델의 최근 발전은 주로 차원 축소를 통해 파생된 잠재 표현을 활용하는 모델에 의해 주도되었습니다. 이러한 표현은 연속적 [^1]이거나 양자화될 수 있으며 [^27], 비디오 생성의 성공은 이러한 표현이 시공간 정보를 얼마나 효율적으로 캡처하는지에 크게 좌우됩니다. 주요 과제 중 하나는 잠재 공간으로의 매핑 및 잠재 공간으로부터의 매핑이 시각적 충실도와 의미적 일관성을 모두 보존하도록 보장하는 것입니다. 대규모 데이터 세트로 확장하면 일반화가 향상되지만 [^2], 시각적 사실성과 시간적 일관성을 모두 보존하는 데 어려움이 남아 있습니다.

양자화된 토큰을 사용하는 자기회귀 및 마스킹된 생성 모델은 시간적 의존성 및 모션 역학의 강력한 모델링을 보여주었습니다 [^28]. 그러나 순차적 생성 프로세스로 인해 느린 추론 및 오류 누적이 발생하여 긴 시퀀스 또는 복잡한 다중 에이전트 장면에 대한 확장성을 제한할 수 있습니다. 이와 대조적으로, 잠재 확산 모델은 반복적인 노이즈 제거를 사용하여 전체 비디오 시퀀스를 생성하는 더 병렬화 가능한 대안을 제공합니다. Stable Video Diffusion [^29], DIAMOND [^30], Sora [^31], MovieGen [^14], Gen-3 [^32], Cosmos [^2]를 포함한 이러한 모델은 텍스트-비디오 및 이미지-비디오 작업, 특히 개방 도메인 설정에서 인상적인 품질을 달성했습니다.

그러나 이러한 연구의 대부분은 시각적 미학과 도메인 불가지론적 생성을 우선시하며, 장면 요소 또는 에이전트 행동에 대한 구조화된 제어를 제한적으로 지원합니다. 이로 인해 자율 주행과 같은 애플리케이션에 덜 적합합니다. 자율 주행은 공간과 시간에 걸쳐 정밀하고 의미론적이며 다중 모드 장면 제어를 요구하기 때문입니다. 이러한 맥락에서 GAIA-2는 안전 필수 시스템에 필요한 고충실도 생성과 도메인별 제어 가능성 간의 격차를 해소하는 것을 목표로 합니다.

##### 자율 주행 분야의 생성형 월드 모델.

범용 비디오 생성과 달리, 자율 주행을 위한 생성형 세계 모델은 더 엄격한 요구 사항을 충족해야 합니다. 다중 에이전트 상호 작용, 자율 주행 차량 행동 제어, 환경 다양성, 다중 카메라 일관성 등이 있습니다. 이 분야의 초기 연구에는 GAIA-1 [^3]이 포함됩니다. 이 모델은 시간적 일관성을 개선하기 위해 비디오 확산 디코더를 갖춘 이산 세계 모델에 텍스트 및 자율 주행 차량 행동 조건화를 도입했습니다. CommaVQ [^33]도 유사하게 자율 주행 차량 행동 제어를 위해 이산 토큰에 대한 인과 관계 트랜스포머를 활용했습니다. 이러한 방법들은 제어 가능성을 향한 중요한 단계를 밟았지만, 종종 단일 카메라 설정과 행동 또는 텍스트 조건화를 통한 부분적인 제어 가능성을 목표로 하여 포괄적인 운전 시뮬레이션에 대한 적용 가능성을 제한했습니다.

이후의 접근 방식은 더 높은 충실도 생성을 위해 잠재 확산을 채택하고 더 유연한 제어를 도입했습니다. DriveDreamer [^34]는 3D 바운딩 박스, HD 지도 및 자율 주행 차량 행동에 조건화된 잠재 확산 모델을 채택하고, 미래 자율 주행 차량 행동을 예측하는 행동 디코더를 추가로 통합합니다. Drive-WM [^35]은 이 접근 방식을 다중 카메라 설정(자율 주행 차량 주변 최대 6개 카메라)으로 확장하고, 자율 주행 차량 행동, 동적 에이전트 및 날씨, 조명과 같은 환경 조건에 대한 제어 가능성을 도입합니다. UniMLVG [^36]도 유사하게 텍스트, 카메라 매개변수, 3D 바운딩 박스 및 HD 지도에 조건화된 다중 뷰 생성을 가능하게 하는 반면, MaskGWM [^37]은 UniMLVG를 확장하여 더 긴 비디오 생성을 지원합니다. Vista [^38]는 고해상도, 장시간 비디오 생성을 위해 잠재 확산을 사용하고, Delphi [^39]는 데이터 기반 접근 방식을 사용하여 비디오 생성을 운전 모델의 실패 사례로 유도하여 합성 데이터의 훈련 유용성을 향상시킵니다. 최근의 연구는 GEM [^40]이 자율 주행 차량 행동, 객체 역학, 심지어 인간 자세를 포함하도록 장면 제어를 더욱 일반화하여 인간 모션 합성 및 드론 탐색과 같은 도메인 전반에 걸쳐 더 넓은 적용 가능성을 제시합니다. 최근의 연구는 4D 기하학 인식 생성을 탐구했습니다. DriveDreamer4D [^41]는 비디오 생성 모델을 사용하여 새로운 궤적을 따라 비디오를 합성하고, 실제 및 생성된 영상을 결합하여 폐쇄 루프 시뮬레이션을 위한 4D 가우스 스플래팅 모델을 최적화합니다. 마찬가지로 DreamDrive [^42]는 생성적 사전 학습을 활용하여 실제 운전 데이터에서 고품질 4D 장면을 구성합니다.

이러한 발전에도 불구하고, 이러한 솔루션의 대부분은 현실적인 운전 시뮬레이션 요구 사항의 일부만 다룹니다. 일부는 단일 카메라 설정으로 제한되고, 다른 일부는 구조화된 에이전트 수준 제어가 부족하며, 다중 뷰 생성 및 에이전트 수준 의미론을 단일 프레임워크에 통합하는 경우는 거의 없습니다. 또한, 많은 모델이 인페인팅 또는 목표 지향적 장면 수정과 같은 정밀 편집 작업을 지원하지 않습니다. 이는 시뮬레이션 루프에서 데이터 증강 및 시나리오 변형에 필수적입니다.

이러한 한계에 따라, GAIA-2는 다음을 통합하는 통합 잠재 확산 프레임워크로 설계되었습니다.

- 다중 카메라 생성(최대 5개 뷰);
    
- 자율 주행 차량 행동, 동적 에이전트 및 장면 수준 메타데이터에 대한 구조화되고 의미론적 조건화;
    
- 처음부터 생성, 인페인팅, 장면 편집 및 장기 롤아웃을 포함한 유연한 추론 모드; 그리고
    
- CLIP 및 운전 시나리오 임베딩과 같은 외부 잠재 공간 지원.
    

연속 잠재 표현과 지리적 및 환경적으로 다양한 데이터 세트에 대한 대규모 훈련을 결합함으로써 GAIA-2는 시각적 충실도와 시나리오 제어 가능성을 모두 제공합니다. 이는 일반적인 확산 모델과 자율 주행의 도메인 특정 요구 사항, 즉 강력하고 확장 가능한 훈련 및 평가를 위한 현실적이고 제어 가능한 다중 뷰 운전 장면을 시뮬레이션할 수 있는 능력 사이의 중요한 격차를 메웁니다.

## 8 결론 및 향후 연구

우리는 자율 주행 시뮬레이션의 다양하고 까다로운 요구 사항을 충족하도록 설계된 도메인 전문 잠재 확산 모델인 GAIA-2를 소개했습니다. GAIA-2는 다중 카메라 일관성, 구조화된 의미론적 조건화, 그리고 정밀한 제어를 확장 가능한 아키텍처 내에서 통합함으로써 생성 세계 모델링의 새로운 기준을 제시합니다. 이 모델은 자율 주행 차량 역학, 환경 특징, 도로 레이아웃 및 동적 에이전트에 따라 제어 가능한 생성을 지원하며, 최대 5개의 카메라 뷰에 걸쳐 공간 및 시간적 일관성을 유지합니다.

이 아키텍처는 시공간 분해 잠재 확산 세계 모델과 압축되고 의미론적으로 의미 있는 비디오 토크나이저를 결합하여 다양한 운전 환경에서 고충실도 생성을 가능하게 합니다. GAIA-2는 처음부터 생성, 자기회귀 롤아웃, 공간 인페인팅, 실제 장면 편집을 포함한 여러 추론 모드를 지원하며, 각 모드는 운전 장면 공간의 체계적인 탐색을 가능하게 합니다. 이 모델은 일반적인 시나리오뿐만 아니라 드물고 안전 필수적인 시나리오를 생성하는 데 탁월하여 자율 주행 시스템에서 데이터 다양성, 시나리오 커버리지 및 검증 엄격성을 크게 향상시킵니다.

시뮬레이션에서 제어 가능성과 장면 다양성을 확장함으로써 GAIA-2는 실제 데이터 제한과 강력한 AI 주행 모델 훈련 및 평가의 증가하는 요구 사이의 격차를 해소합니다. 그렇게 함으로써, 반복 주기를 가속화하고, 분포 외 조건에 대한 일반화를 개선하며, 복잡하고 롱테일 시나리오에서 AI 에이전트를 스트레스 테스트하는 강력한 도구 역할을 합니다.

##### 향후 연구.

GAIA-2는 상당한 진전을 나타내지만, 몇 가지 연구 방향은 여전히 열려 있으며 이 작업의 향후 반복을 안내할 것입니다. (i) 모든 생성 모델과 마찬가지로 GAIA-2는 특히 장시간 또는 복잡한 시나리오에서 때때로 시간적 또는 의미적 불일치를 생성합니다. 더 나은 실패 감지, 정제 모델 또는 제약 인식 샘플링을 통해 비디오 생성의 신뢰성과 일관성을 향상시키는 것은 핵심 과제입니다. (ii) GAIA-2는 병렬 생성을 가능하게 하지만, 실시간 또는 거의 실시간 비디오 합성은 여전히 계산 집약적입니다. 향후 연구에서는 모델 증류, 효율적인 트랜스포머 변형 및 추론 시간 가속 기술을 탐구하여 실제 파이프라인에서 배포 가능성을 향상시킬 것입니다. (iii) 풍부한 조건화 세트를 지원함에도 불구하고, 새로운 노력은 더 풍부한 에이전트 행동 모델링, 더 미묘한 환경 조건 및 개방형 자연어 제어를 목표로 할 것입니다. 인간 에이전트, 복잡한 인프라 또는 사회적 상호 작용과 관련된 희귀하고 안전 필수적인 이벤트로 훈련 데이터 세트를 더욱 풍부하게 하는 것은 현실적인 시뮬레이션의 한계를 뛰어넘을 것입니다.

이러한 방향들이 함께, GAIA-2 및 그 후속 모델이 안전하고 견고하며 일반화 가능한 자율 시스템 개발의 핵심 인프라로서 생성 모델의 역할을 계속 발전시킬 것입니다.

## 감사 및 자금 공개

저자들은 이 작업을 가능하게 한 수많은 개인들의 귀중한 기여에 감사드립니다. Alex Persin, Ana-Maria Marcu, Aniruddha Kembhavi, Benoit Hanotte, Evgenii Kashin, Francesca Iovu, Giacomo Gallino, Giulio D’Ippolito, Jeremy Plassmann, Lorenzo von Ritter, Mohamed Nabil, Nikhil Mohan, Oleh Chernov, Pragya Kale, Remi Tachet, Rudi Rankin, Sahana Vankatesh, Sarah Belghiti, Sofía Dudas, Tilly Pielichaty, Vassia Simaiaki, Victor Delépine, Vincent Micheli, Zak Murez.

## 참고 문헌

[^1]:

D. P. Kingma and M. Welling.Auto-encoding variational bayes.Proceedings of the International Conference on Learning Representations (ICLR), 2014.

[^2]:

N. Agarwal, A. Ali, M. Bala, Y. Balaji, E. Barker, T. Cai, P. Chattopadhyay, Y. Chen, Y. Cui, Y. Ding, et al.Cosmos world foundation model platform for physical ai.arXiv preprint arXiv:2501.03575, 2025.

[^3]:

A. Hu, L. Russell, H. Yeo, Z. Murez, G. Fedoseev, A. Kendall, J. Shotton, and G. Corrado.Gaia-1: A generative world model for autonomous driving.Technical Report arXiv:2309.17080, 2023.

[^4]:

J. Chen, H. Cai, J. Chen, E. Xie, S. Yang, H. Tang, M. Li, Y. Lu, and S. Han.Deep compression autoencoder for efficient high-resolution diffusion models.Proceedings of the International Conference on Learning Representations (ICLR), 2025.

[^5]:

W. Shi, J. Caballero, F. Huszar, J. Totz, A. P. Aitken, R. Bishop, D. Rueckert, and Z. Wang.Real-time single image and video super-resolution using an efficient sub-pixel convolutional neural network.In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2016.

[^6]:

J. Johnson, A. Alahi, and L. Fei-Fei.Perceptual losses for real-time style transfer and super-resolution.In Proceedings of the European Conference on Computer Vision (ECCV), 2016.

[^7]:

M. Oquab, T. Darcet, T. Moutakanni, H. V. Vo, M. Szafraniec, V. Khalidov, P. Fernandez, D. Haziza, F. Massa, A. El-Nouby, R. Howes, P.-Y. Huang, H. Xu, V. Sharma, S.-W. Li, W. Galuba, M. Rabbat, M. Assran, N. Ballas, G. Synnaeve, I. Misra, H. Jegou, J. Mairal, P. Labatut, A. Joulin, and P. Bojanowski.DINOv2: Learning robust visual features without supervision.Transactions on Machine Learning Research (TMLR), 2024.

[^8]:

P. Esser, R. Rombach, and B. Ommer.Taming transformers for high-resolution image synthesis.In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2021.

[^9]:

R. Zhang.Making convolutional networks shift-invariant again.In Proceedings of the International Conference on Machine Learning (ICML), 2019.

[^10]:

T. Miyato, T. Kataoka, M. Koyama, and Y. Yoshida.Spectral normalization for generative adversarial networks.In Proceedings of the International Conference on Learning Representations (ICLR), 2018.

[^11]:

Y. Lipman, R. T. Q. Chen, H. Ben-Hamu, M. Nickel, and M. Le.Flow matching for generative modeling.Proceedings of the International Conference on Learning Representations (ICLR), 2023.

[^12]:

J. Ho, A. Jain, and P. Abbeel.Denoising diffusion probabilistic models.In Advances in Neural Information Processing Systems (NeurIPS), 2020.

[^13]:

W. Peebles and S. Xie.Scalable diffusion models with transformers.In Proceedings of the IEEE International Conference on Computer Vision (ICCV), 2023.

[^14]:

A. Polyak, A. Zohar, A. Brown, A. Tjandra, A. Sinha, A. Lee, A. Vyas, B. Shi, C.-Y. Ma, C.-Y. Chuang, et al.Movie gen: A cast of media foundation models.arXiv preprint, 2024.

[^15]:

M. Dehghani, J. Djolonga, B. Mustafa, P. Padlewski, J. Heek, J. Gilmer, A. Steiner, M. Caron, R. Geirhos, I. Alabdulmohsin, R. Jenatton, L. Beyer, M. Tschannen, A. Arnab, X. Wang, C. Riquelme, M. Minderer, J. Puigcerver, U. Evci, M. Kumar, S. van Steenkiste, G. F. Elsayed, A. Mahendran, F. Yu, A. Oliver, F. Huot, J. Bastings, M. P. Collier, A. Gritsenko, V. Birodkar, C. Vasconcelos, Y. Tay, T. Mensink, A. Kolesnikov, F. Pavetić, D. Tran, T. Kipf, M. Lučić, X. Zhai, D. Keysers, J. Harmsen, and N. Houlsby.Scaling vision transformers to 22 billion parameters.In Proceedings of the International Conference on Machine Learning (ICML), 2023.

[^16]:

J. B. W. Webber.A bi-symmetric log transformation for wide-range data.Measurement Science and Technology, 2012.

[^17]:

S. Wang, Y. Liu, T. Wang, Y. Li, and X. Zhang.Exploring object-centric temporal modeling for efficient multi-view 3d object detection.In Proceedings of the IEEE/CVF International Conference on Computer Vision, pages 3621–3631, 2023.

[^18]:

A. Radford, J. W. Kim, C. Hallacy, A. Ramesh, G. Goh, S. Agarwal, G. Sastry, A. Askell, P. Mishkin, J. Clark, G. Krueger, and I. Sutskever.Learning transferable visual models from natural language supervision.Proceedings of the International Conference on Machine Learning (ICML), 2021.

[^19]:

R. Rombach, A. Blattmann, D. Lorenz, P. Esser, and B. Ommer.High-resolution image synthesis with latent diffusion models.In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2022.

[^20]:

R. Zhang, P. Isola, A. A. Efros, E. Shechtman, and O. Wang.The unreasonable effectiveness of deep features as a perceptual metric.In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2018.

[^21]:

G. Stein, J. C. Cresswell, R. Hosseinzadeh, Y. Sui, B. L. Ross, V. Villecroze, Z. Liu, A. L. Caterini, J. E. T. Taylor, and G. Loaiza-Ganem.Exposing flaws of generative model evaluation metrics and their unfair treatment of diffusion models.In Advances in Neural Information Processing Systems (NeurIPS), 2023.

[^22]:

M. Heusel, H. Ramsauer, T. Unterthiner, B. Nessler, and S. Hochreiter.Gans trained by a two time-scale update rule converge to a local nash equilibrium.In Advances in Neural Information Processing Systems (NeurIPS), 2017.

[^23]:

C. Szegedy, V. Vanhoucke, S. Ioffe, J. Shlens, and Z. Wojna.Rethinking the inception architecture for computer vision.In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2016.

[^24]:

J. Liu, Y. Qu, Q. Yan, X. Zeng, L. Wang, and R. Liao.Fréchet video motion distance: A metric for evaluating motion consistency in videos.In Proceedings of the International Conference on Machine Learning, workshop (ICMLw), 2024.

[^25]:

T. Unterthiner, S. Steenkiste, K. Kurach, R. Marinier, M. Michalski, and S. Gelly.Towards accurate generative models of video: A new metric & challenges.arXiv preprint, 2018.

[^26]:

J. Jain, J. Li, M. Chiu, A. Hassani, N. Orlov, and H. Shi.Oneformer: One transformer to rule universal image segmentation.In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR), 2023.

[^27]:

A. van den Oord, O. Vinyals, and K. Kavukcuoglu.Neural discrete representation learning.In Advances in Neural Information Processing Systems (NeurIPS), 2017.

[^28]:

W. Yan, Y. Zhang, P. Abbeel, and A. Srinivas.VideoGPT: Video generation using vq-vae and transformers.In arXiv preprint, 2021.

[^29]:

A. Blattmann, T. Dockhorn, S. Kulal, D. Mendelevitch, M. Kilian, D. Lorenz, Y. Levi, Z. English, V. Voleti, A. Letts, et al.Stable video diffusion: Scaling latent video diffusion models to large datasets.arXiv preprint, 2023.

[^30]:

E. Alonso, A. Jelley, V. Micheli, A. Kanervisto, A. Storkey, T. Pearce, and F. Fleuret.Diffusion for world modeling: Visual details matter in atari.In Advances in Neural Information Processing Systems (NeurIPS), 2024.

[^31]:

OpenAI.Video generation models as world simulators.https://openai.com/index/video-generation-models-as-world-simulators, 2024.

[^32]:

Runway.Introducing gen-3 alpha: A new frontier for video generation.https://runwayml.com/research/introducing-gen-3-alpha, 2024.

[^33]:

comma.ai.commavq.https://github.com/commaai/commavq, 2023.

[^34]:

X. Wang, Z. Zhu, G. Huang, X. Chen, J. Zhu, and J. Lu.Drivedreamer: Towards real-world-driven world models for autonomous driving.Proceedings of the European Conference on Computer Vision (ECCV), 2024a.

[^35]:

Y. Wang, J. He, L. Fan, H. Li, Y. Chen, and Z. Zhang.Driving into the future: Multiview visual forecasting and planning with world model for autonomous driving.In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pages 14749–14759, 2024b.

[^36]:

R. Chen, Z. Wu, Y. Liu, Y. Guo, J. Ni, H. Xia, and S. Xia.Unimlvg: Unified framework for multi-view long video generation with comprehensive control capabilities for autonomous driving.arXiv preprint arXiv:2412.04842, 2024.

[^37]:

J. Ni, Y. Guo, Y. Liu, R. Chen, L. Lu, and Z. Wu.Maskgwm: A generalizable driving world model with video mask reconstruction.arXiv preprint arXiv:2502.11663, 2025.

[^38]:

S. Gao, J. Yang, L. Chen, K. Chitta, Y. Qiu, A. Geiger, J. Zhang, and H. Li.Vista: A generalizable driving world model with high fidelity and versatile controllability.Advances in Neural Information Processing Systems (NeurIPS), 2024.

[^39]:

E. Ma, L. Zhou, T. Tang, Z. Zhang, D. Han, J. Jiang, K. Zhan, P. Jia, X. Lang, H. Sun, et al.Unleashing generalization of end-to-end autonomous driving with controllable long video generation.arXiv preprint arXiv:2406.01349, 2024.

[^40]:

M. Hassan, S. Stapf, A. Rahimi, P. Rezende, Y. Haghighi, D. Brüggemann, I. Katircioglu, L. Zhang, X. Chen, S. Saha, et al.Gem: A generalizable ego-vision multimodal world model for fine-grained ego-motion, object dynamics, and scene composition control.arXiv preprint arXiv:2412.11198, 2024.

[^41]:

G. Zhao, C. Ni, X. Wang, Z. Zhu, X. Zhang, Y. Wang, G. Huang, X. Chen, B. Wang, Y. Zhang, et al.Drivedreamer4d: World models are effective data machines for 4d driving scene representation.arXiv preprint arXiv:2410.13571, 2024.

[^42]:

J. Mao, B. Li, B. Ivanovic, Y. Chen, Y. Wang, Y. You, C. Xiao, D. Xu, M. Pavone, and Y. Wang.Dreamdrive: Generative 4d scene modeling from street view images.arXiv preprint arXiv:2501.00601, 2024.

}

Based on the provided research paper, what are the different approaches for generating safety-critical scenarios for autonomous vehicles? Explain each approach, including its advantages and disadvantages. Do not provide any information outside of the provided text.