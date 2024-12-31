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
5. [사용 방법 (간단 예시)](#사용-방법-간단-예시)  
6. [향후 확장](#향후-확장)  
7. [라이선스](#라이선스)  

---

## 프로젝트 개요

- **목표**  
  - 주가 변동률(returns)의 단기(7일) 예측  
  - 투자자의 다양한 위험 성향을 반영한 멀티에이전트 시스템 통합  
  - 상승(Up), 하락(Down), 횡보(Flat) 상태에 따라 변동률 패턴을 생성하는 GAN 모델 구현  

- **핵심 기술**  
  - **GAN (Generative Adversarial Network)**  
    - 시계열 데이터 생성(생성기) 및 진위 판별(판별기)  
  - **멀티에이전트(Multi-Agent)**  
    - 서로 다른 에이전트의 판단 결과를 종합하여 시장 상태 결정  
  - **LSTM + Attention**  
    - 시계열(시간적 의존성)과 상태(중요 시점 강조)를 함께 학습  

---

## 전체 아키텍처 소개

아래 이미지는 전체 시스템의 데이터 흐름을 보여줍니다. 왼쪽은 **멀티에이전트 시스템**이고, 오른쪽은 **GAN 모델**로 구성됩니다.

![인공지능 경영 drawio (1)](https://github.com/user-attachments/assets/c57d8942-5a5e-4b76-956d-84874e092f82)

### 2.1 멀티에이전트 시스템 구조

1. **News Scrapper**  
   - 최신 뉴스를 스크래핑하여 시장의 거시적·미시적 흐름에 대한 정보를 제공합니다.

2. **Risk Seeking Agent**  
   - 위험 추구 성향을 가진 에이전트로, 긍정적인(상승) 시그널에 민감하게 반응  
   - 뉴스 및 시장 지표를 종합하여 현재 시장을 `Up / Down / Flat / Unable to judge` 중 하나로 평가

3. **Risk Adverse Agent**  
   - 보수적 성향을 가진 에이전트로, 부정적인(하락) 시그널에 민감하게 반응  
   - 동일한 뉴스 및 지표를 바탕으로 별도의 관점으로 `Up / Down / Flat / Unable to judge` 중 하나를 도출

4. **Mediator Agent**  
   - **Risk Seeking Agent**와 **Risk Adverse Agent**의 의견을 종합하여 **최종 시장 상태**를 결정  
   - 만약 두 에이전트 간 의견이 충돌할 경우, 추가적 규칙 또는 가중치를 통해 최종 `Up / Down / Flat` 상태를 산출
  
<img width="1002" alt="스크린샷 2024-12-29 오후 5 00 17" src="https://github.com/user-attachments/assets/4c2fa6fd-cefd-4d1a-a7a1-33338f480314" />


### 2.2 GAN 구조

1. **Generator (생성기)**  
   - **LSTM** 레이어로 시간적 맥락을 학습하고, **Attention** 메커니즘으로 특정 시점의 중요도를 반영  
   - 투자자의 위험 성향과 함께 결정된 *최종 시장 상태*를 임베딩하여, 해당 시나리오에 맞는 주가 변동률을 생성

2. **Critic (비교·평가 역할)**  
   - 보통 ‘판별기(Discriminator)’와 함께 동작하며, 생성된 시계열의 품질을 평가해 모델 학습을 개선

3. **Discriminator (판별기)**  
   - **Conv1D** 기반 CNN 아키텍처를 통해, 입력된 시계열이 **진짜(실제 과거 데이터)**인지 **가짜(Generator가 생성한 데이터)**인지 판별  

---

## 시장 상태 결정 과정

본 모델에서는 시장 상태를 **상승(Up)**, **하락(Down)**, **횡보(Flat)**로 분류합니다.  
이를 위한 기준은 다음과 같은 **이동평균(MA) 기반의 ratio**이며, 간단한 예시는 아래와 같습니다:

1. **단기 이동평균 (Short MA)**  
   - 최근 20일간의 평균 주가를 산출  
2. **장기 이동평균 (Long MA)**  
   - 최근 60일간의 평균 주가를 산출  
3. **ratio 계산**  
   - (단기 이동평균 - 장기 이동평균) / 장기 이동평균  
   - 예를 들어, ratio ≥ 0.01이면 상승, ratio ≤ -0.01이면 하락, 그 사이면 횡보로 분류  

![image](https://github.com/user-attachments/assets/3fc697a9-ee1e-4f69-9b49-706e58ce439f)
![image (1)](https://github.com/user-attachments/assets/061c12ef-71a8-4c43-9e1b-e6481f70d09c)

최종적으로, **Risk Seeking Agent**와 **Risk Adverse Agent**가 이 정보를 포함한 다양한 지표를 활용하여 각각 시장 상태를 판단한 뒤, **Mediator Agent**가 종합해 **최종 상태**를 확정합니다.

---

## LSTM + Attention으로 상태 반영

### 1. 상태 정보를 학습에 반영

- **상태 임베딩**  
  - 시장 상태(Up, Down, Flat)를 정수 값으로 인코딩하여 임베딩 레이어에서 학습  
  - 각 상태에 대응되는 벡터를 생성하여, 모델이 상태 정보를 더 효과적으로 반영하도록 유도  

- **LSTM 입력 구성**  
  - 노이즈 벡터(랜덤 시드), 직전 시점의 가격 혹은 변동률 정보, 상태 임베딩 등을 하나로 합쳐 LSTM의 입력으로 사용  
  - 이를 통해 시계열 특성과 시장 상태를 동시에 고려하여 모델이 시나리오별 패턴을 학습  

### 2. Attention 메커니즘

- **시장 상태(또는 상태 임베딩)와 LSTM 출력**을 이용해, 시계열의 각 타임스텝에 대한 중요도를 계산  
- 이렇게 계산된 중요도(Attention)는 예측에 보다 중요한 시점의 정보를 강조하는 데 사용



---

## 사용 방법 (간단 예시)

1. **데이터 준비**  
   - 날짜별 종가(Close), 이동평균(Short_MA, Long_MA) 등이 포함된 시계열 데이터를 준비  
   - MA 기반 ratio를 계산해 Up/Down/Flat 등 시장 상태를 파악  

2. **멀티에이전트 판단**  
   - **Risk Seeking Agent**, **Risk Adverse Agent**에 뉴스, 지표를 입력해 각각 시장 상태를 판단  
   - **Mediator Agent**가 두 결과를 종합해 최종 상태(Up/Down/Flat)를 결정  

3. **GAN 학습**  
   - **Generator**: LSTM + Attention + 상태 임베딩  
   - **Discriminator**: Conv1D 기반 CNN  
   - 실제 시계열 vs. 생성된 시계열을 판별하며 반복 학습  

4. **예측 수행**  
   - 학습된 **Generator**에 노이즈, 과거 시점 데이터, 최종 상태를 입력  
   - 7일 후 주가 변동률을 생성하여 예측 결과를 확인  

---

## 향후 확장

- **다양한 에이전트 추가**  
  - 모멘텀 지표, 매크로 경제지표 등을 반영하는 에이전트 확장  
- **멀티 모달 데이터**  
  - 뉴스 텍스트뿐 아니라 소셜 미디어, 이벤트 캘린더 등의 정보를 함께 처리  
- **리워드 기반 강화학습**  
  - 예측 정확도뿐 아니라, 실제 투자 성과(수익)를 기준으로 에이전트의 의사결정을 강화  

---
