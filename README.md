# Multi-Agent GAN 기반 주가 변동 예측 모델

이 프로젝트는 **멀티에이전트(Multi-Agent) 분석**과 **GAN(Generative Adversarial Network)** 구조를 결합하여, 주가 변동률(returns)을 보다 현실적으로 예측하는 시계열 모델을 구현합니다.  
투자자의 위험 성향과 시장 상태(상승/하락/횡보)를 반영하여, 다양한 시나리오 기반의 예측을 가능하게 하는 것이 특징입니다.

---

## 목차
1. [프로젝트 개요](#프로젝트-개요)  
2. [전체 아키텍처 소개](#전체-아키텍처-소개)  
   - [2.1 멀티에이전트 시스템 구조](#21-멀티에이전트-시스템-구조)  
   - [2.2 GAN 구조](#22-gan-구조)  
3. [시장 상태 결정 과정](#시장-상태-결정-과정)  
4. [LSTM + Attention으로 상태 반영](#lstm--attention으로-상태-반영)  
5. [적용 사례](#적용-사례)  
6. [설정값 정리](#설정값-정리)  
7. [결과](#결과)

---

## 프로젝트 개요

- **목표**  
  - **주가 변동률(returns)의 단기(7일) 예측**  
    - 단순히 종가(price)를 직접 예측하는 대신, **수익률**을 생성해 이를 (1 + 예측 수익률) 방식으로 최종 가격을 추정합니다.  
  - 투자자의 다양한 위험 성향을 반영한 멀티에이전트 시스템 통합  
  - 상승(Up), 하락(Down), 횡보(Flat) 상태에 따라 변동률 패턴을 생성하는 GAN 모델 구현  

- **핵심 기술**  
  - **GAN (Generative Adversarial Network)**  
    - 시계열 데이터(수익률) 생성(생성기) 및 진위 판별(판별기)  
  - **멀티에이전트(Multi-Agent) + LLM**  
    - 서로 다른 에이전트가 **GPT 4o-mini**를 통해 시장 상태를 결정 (Up / Down / Flat)  
  - **LSTM + Attention**  
    - 시계열(시간적 의존성)과 에이전트가 결정한 상태(중요 시점 강조)를 함께 학습  

---

## 전체 아키텍처 소개

아래 이미지는 전체 시스템의 데이터 흐름을 보여줍니다. 왼쪽은 **LLM 기반 멀티에이전트 시스템**이고, 오른쪽은 **GAN 모델**로 구성됩니다.

![인공지능 경영 drawio (1)](https://github.com/user-attachments/assets/c57d8942-5a5e-4b76-956d-84874e092f82)

### 2.1 멀티에이전트 시스템 구조

1. **News Scrapper**  
   - 최신 뉴스를 스크래핑하여 시장의 거시적·미시적 흐름에 대한 정보를 제공합니다.

2. **Risk Seeking Agent**  
   - 위험 추구 성향을 가진 에이전트로, 긍정적인(상승) 시그널에 민감하게 반응  
   - 뉴스 및 시장 지표를 종합하여 현재 시장을 Up / Down / Flat / Unable to judge 중 하나로 평가

3. **Risk Adverse Agent**  
   - 보수적 성향을 가진 에이전트로, 부정적인(하락) 시그널에 민감하게 반응  
   - 동일한 뉴스 및 지표를 바탕으로 별도의 관점에서 Up / Down / Flat / Unable to judge 중 하나를 도출

4. **Mediator Agent**  
   - **Risk Seeking Agent**와 **Risk Adverse Agent**의 의견을 종합하여 **최종 시장 상태**를 결정  
   - 만약 두 에이전트 간 의견이 충돌할 경우, 추가적 규칙 또는 가중치를 통해 최종 Up / Down / Flat 상태를 산출  

---

### 2.2 GAN 구조

1. **Generator (생성기)**  
   - **LSTM** 레이어로 시간적 맥락을 학습하고, **Attention** 메커니즘으로 특정 시점의 중요도를 반영  
   - 에이전트가 결정한 *최종 시장 상태*를 임베딩하여, 해당 시나리오에 맞는 **수익률** 시퀀스를 생성  
     - 학습 시에는 실제 수익률을 \[0,1\] 구간으로 스케일링하여 LSTM으로 입력하고,  
     - 예측 시에는 (1 + 예측 수익률) 형태로 실제 가격을 역산합니다.

2. **Critic (비교·평가 역할)**  
   - 보통 ‘판별기(Discriminator)’와 함께 동작하며, 생성된 시계열의 품질을 평가해 모델 학습을 개선  

3. **Discriminator (판별기)**  
   - **Conv1D** 기반 CNN 아키텍처를 통해, 입력된 시계열이 진짜(실제 과거 데이터)인지 가짜(Generator가 생성한 데이터)인지 판별  

---

## 시장 상태 결정 과정

본 모델에서는 시장 상태를 상승(Up), 하락(Down), 횡보(Flat)로 분류합니다.  
이를 위한 기준은 다음과 같은 이동평균(MA) 기반의 ratio이며, 간단한 예시는 아래와 같습니다:

1. 단기 이동평균 (Short MA)  
   - 최근 20일간의 평균 주가를 산출  

2. **장기 이동평균 (Long MA)**  
   - 최근 60일간의 평균 주가를 산출  

3. **ratio 계산**  
   - (단기 이동평균 - 장기 이동평균) / 장기 이동평균  
   - 예를 들어, ratio ≥ 0.01이면 상승, ratio ≤ -0.01이면 하락, 그 사이면 횡보로 분류  

결정된 시장 상태(Up/Down/Flat)는 곧 LSTM + Attention 모듈이 수익률 예측을 수행할 때 참고하는 핵심 파라미터로 활용됩니다.

---

## LSTM + Attention으로 상태 반영

1. **상태 임베딩**  
   - 시장 상태(Up, Down, Flat)를 정수 값으로 인코딩하여 임베딩 레이어에서 학습  
   - 각 상태에 대응되는 벡터를 생성하여, 모델이 상태 정보를 더 효과적으로 반영하도록 유도  

2. **LSTM 입력 구성**  
   - 노이즈 벡터(랜덤 시드), 직전 시점의 가격 또는 수익률, 상태 임베딩 등을 하나로 합쳐 LSTM의 입력으로 사용  
   - 이를 통해 시계열 특성과 시장 상태를 동시에 고려해, 시나리오별 변동률 패턴을 학습  

3. **Attention 메커니즘**  
   - **시장 상태(또는 상태 임베딩)와 LSTM 출력을 이용해, 시계열의 각 타임스텝에 대한 중요도를 계산  
   - 이렇게 계산된 중요도(Attention)는 예측에 보다 중요한 시점의 정보를 강조하는 데 사용  

4. **최종 출력(수익률)**  
   - 모델이 출력하는 최종 시퀀스는 수익률 형태이며, 이를 (1 + 예측 수익률)로 환산하여 종가를 추정합니다.

---

## 적용 사례

아래 예시에서는 삼성전자(005930.KS)를 대상으로, 2014년 12월 1일부터 2024년 12월 1일까지의 데이터를 다운로드하고, 이 중 2024년 9월 26일 이전까지의 구간을 학습 데이터로 사용하였습니다.  
2024년 9월 27일부터 1주 기간은 예측 범위로 설정했습니다.

![image](https://github.com/user-attachments/assets/3fc697a9-ee1e-4f69-9b49-706e58ce439f)
![image (1)](https://github.com/user-attachments/assets/061c12ef-71a8-4c43-9e1b-e6481f70d09c)

예측 시점 직전 일주일의 뉴스를 분석한 결과 멀티 에이전트의 판단은 '하락(Down)'이었고, 이를 바탕으로 예측을 진행하였습니다.  
모델은 하락 상태를 임베딩하여 학습한 패턴에 기반해 미래 수익률을 생성했고, 실제 종가에 (1 + 예측 수익률) 방식으로 적용하여 7일 간의 가격 추이도 예측하였습니다.

---

## 설정값 정리

1. **시계열 길이 (`seq_len = 30`)**  
   - 한 번에 볼 데이터 구간의 길이. (30일)

2. **노이즈 차원 (`noise_dim = 5`)**  
   - 생성기 입력으로 들어가는 랜덤 벡터 크기.

3. **시장 상태 임베딩 크기 (`state_dim = 3`)**  
   - `Up`, `Down`, `Flat` 3가지 상태를 벡터화하는 크기.

4. **LSTM 히든 크기 (`hidden_dim = 16`)**  
   - LSTM 내부 상태 크기. 클수록 모델 파워가 커지지만 과적합 위험도 증가.

5. **배치 크기 (`batch_size = 16`)**  
   - 한 번 학습 시 사용하는 데이터 샘플 수.

6. **학습 횟수 (`num_epochs = 50`)**  
   - 전체 데이터를 반복 학습하는 횟수.

7. **Critic 업데이트 횟수 (`n_critic = 5`)**  
   - WGAN 방식을 따라, 생성기를 1번 학습할 때 Critic을 5번 학습.

8. **Optimizer**  
   - **RMSprop** 사용, 학습률(`lr`)은 `0.00005`.

9. **Gradient Clipping (`clip_value = 0.01`)**  
   - Critic 파라미터의 폭주를 막기 위해 \[-0.01, 0.01\] 범위로 제한.

10. **MinMaxScaler**  
   - 실제 수익률(returns)을 \[0,1\]로 변환해 안정적으로 학습.
---

## 결과

![image (2)](https://github.com/user-attachments/assets/3eb5737f-4c85-498a-b141-8b0e8d9c91bf)

위 그래프에서 파란색 곡선은 실제 종가, 주황색 곡선은 (1 + 예측 수익률)을 누적 곱하여 추정한 예측 종가입니다.
당초 **주된 목표**는 멀티 에이전트의 판단(하락 상태)을 얼마나 잘 반영하여 미래 변동률을 생성하는지였고, 결과적으로 **‘하락’ 시그널을 반영한 예측 곡선**이 실제 시계열 추세와 어느 정도 유사하게 움직이는 것을 확인했습니다.

더욱 다채로운 지표 활용과 멀티 에이전트의 판단 로직을 한층 정교하게 보완하여, 실제 금융 투자자들이 따르는 의사결정 과정을 면밀히 모사한다면, 예측 성능은 물론 실질적 활용도까지 크게 향상될 것으로 기대됩니다.

---
