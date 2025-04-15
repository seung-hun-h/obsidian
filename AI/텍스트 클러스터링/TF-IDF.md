## 1. 정의
문서 속의 단어가 얼마나 중요한지 수치화한 임베딩 방식이다.
- **TF(Term Frequency)**: 특정 문서에서 단어가 얼마나 자주 등장했는가?(한 문장에서 단어가 얼마나 흔한가?)
$$
\text{TF}(t, d) = \frac{\text{단어 } t \text{의 등장 횟수}}{\text{문서 } d \text{의 전체 단어 수}}
$$

- **IDF(Inverse Document Frequency)**: 전체 문서 중 몇 개 문서에 등장했는가?(문서들 중 단어가 얼마나 희귀한가?)
$$
\text{IDF}(t) = \log \left( \frac{N{\text{(문서의 수)}}}{1 + \text{DF}(t)} \right)
$$

- **TF-IDF**: TF와 IDF를 곱한값. 단어가 중요할 수록 점수가 높다
	- 특정 문서에만 많이 등장하는 단어 -> 점수 높음
	- 많은 문서에서 많이 등장하는 단어 -> 점수 낮음
$$
\text{TF-IDF}(t, d) = \text{TF}(t, d) \times \text{IDF}(t)
$$
## 2. 예시
```text
D1: 사과 사과 바나나
D2: 사과 바나나 바나나 바나나
D3: 포도 포도 바나나
```
- 사과(D1)
	- 단어 등장 횟수: 2
	- 전체 단어수: 3
	- TF = 2/3
	- IDF log(3/(1+2)​)=log(1)=0
 - 바나나(D1)
	- 단어 등장 횟수: 1
	- 전체 단어수: 3
	- TF = 1/3
	- IDF log(3/(1+3)​)=log(3/4)=-0.124
## 3. TF-IDF 벡터
문서마다 전체 단어의 수 만큼 차원을 가지는 벡터를 말한다. 각 문서에 대해 각 단어에 대한 TF-IDF 값을 가지고 있다.

## 4. 예제
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from sklearn.metrics.pairwise import cosine_similarity
from sklearn.feature_extraction.text import TfidfVectorizer

plt.rc('font', family='NanumBarunGothic')

# 예제 데이터 생성
data = {
    'text': [
        "나는 오늘 아침에 커피를 마셨다.",                                      # 1. Beverage
        "오늘 아침에 나는 커피를 한 잔 마셨다.",                                # 2. Beverage (매우 유사)
        "나는 오늘 아침에 차를 마셨다.",                                        # 3. Beverage (애매: 커피 vs 차)
        "오늘은 날씨가 맑아서 기분이 좋았다.",                                # 4. Weather (전혀 다른 주제)
        "인공지능 연구는 최근 빠르게 발전하고 있다.",                         # 5. Tech
        "자연어 처리 기술은 인공지능의 한 분야이다.",                         # 6. Tech (유사)
        "나는 어제 본 영화가 매우 감동적이었다.",                               # 7. Movie
        "영화를 보며 감동을 느낀 나는, 영화관에서 팝콘을 즐겼다.",               # 8. Movie (유사)
        "오늘 아침에 도서관에서 책을 읽으며 커피를 마셨다.",                    # 9. Beverage (커피 포함, 약간 다름)
        "기계 학습과 딥 러닝은 인공지능 발전에 핵심적인 역할을 한다."            # 10. Tech (유사)
    ],
    'category': [
        'Beverage', 'Beverage', 'Beverage', 'Weather',
        'Tech', 'Tech', 'Movie', 'Movie', 'Beverage', 'Tech'
    ]
}

df = pd.DataFrame(data)

# TF-IDF 벡터화 (기본 옵션으로 진행)
tfidf_vectorizer = TfidfVectorizer()
tfidf_matrix = tfidf_vectorizer.fit_transform(df['text'])

# (선택 사항) TF-IDF 행렬을 DataFrame으로 변환
tfidf_df = pd.DataFrame(tfidf_matrix.toarray(), columns=tfidf_vectorizer.get_feature_names_out())

# 문서 간 유사도 계산 (코사인 유사도)
cos_sim = cosine_similarity(tfidf_matrix)

# 유사도 행렬 시각화 - Heatmap 생성
plt.figure(figsize=(8, 6))
plt.imshow(cos_sim, interpolation='nearest', cmap='viridis')
plt.title("TF-IDF 문장 간 코사인 유사도")
plt.colorbar()

# x, y축 라벨 설정 (문장 번호 등으로 표시)
labels = [f"문장 {i+1}" for i in range(len(df))]
plt.xticks(np.arange(len(df)), labels, rotation=45)
plt.yticks(np.arange(len(df)), labels)

# 각 셀마다 유사도 값을 표시 (소수점 둘째 자리까지)
for i in range(len(df)):
    for j in range(len(df)):
        plt.text(j, i, f"{cos_sim[i, j]:.2f}", ha="center", va="center", color="w")

plt.tight_layout()
plt.show()
```


![[TF-IDF 코사인 유사도.png]]