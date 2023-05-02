# :memo: 지난 학습 기록을 통한 정오답 예측 (Knowledge Tracing)

산타토익 서비스를 운영하는 Riiid에서 제공하는 유저 학습 데이터를 이용하여 유저의 지난 학습 기록을 학습하고 아직 학습하지 않은 문제의 결과를 예측하는 머신러닝 모델 구축  

## 주제 선정 이유

코로나 이후 가상 및 원격 학습 환경이 확대되고 개인화 된 학습 서비스가 많이 출시되고 있습니다. 개인화된 학습 서비스의 기본적인 태스크는 수강생이 이전 학습 기록을 활용해 아직 풀지 않은 문제에 대한 정오답을 예측하는 지식 추적(Knowledge Tracing)입니다. 대규모 수강생들에게 교육을 제공하고 있지만 개인화 된 학습 경험을 제공할 수 있을지에 대한 기대에서 해당 프로젝트를 진행하게 되었음.

## 사용 데이터
Riiid Answer Correctness Prediction 데이터(https://www.kaggle.com/c/riiid-test-answer-prediction)
1. train : 유저데이터(101M) / 유저데이터의 크기가 너무 크기 때문에 전체 사용자의 약 10%인 4000명의 유저 데이터만 이용함(총 1,069,559개)
    - row_id 
    - timestamp : 처음 인터렉션(강의 혹은 문제풀이) 이용한 뒤 경과시간 millisecond
    - user_id : 사용자 id
    - content_id : 유저 인터렉션 컨텐츠 id(강의이거나 문제일 수 있음)
    - content_type_id : 컨텐츠가 1= 강의시청, 0=문제풀이
    - task_container_id : 컨텐츠 묶음 id
    - user_answer : 사용자 입력 답
    - answered_correctly : 0=오답 1=정답
    - prior_question_elapsed_time : 이전 문제풀이에 걸린 시간
    - prior_question_had_explanation : 이전 문제풀이 후 강의 시청 여부

2. questions : 문제 데이터(13.5K)
    - question_id : 문제 id
    - bundle_id : 문제 묶음 id
    - correct_answer : 정답
    - part : 토익 파트(1,2,3,4,5,6,7)
    - tags : 문제 태그(구체적인 내용은 없음)
 
3. lectures :강의 데이터(418)
    - lecture_id : 강의 id
    - part : 토익 파트
    - tag : 강의 태그 
    - type_of : 간단한 강의 목적


## 가설
* 어떤 특성이 정답율 예측에 가장 영향을 많이 미칠까?
  - 순차적 데이터이므로 시간과 관련된 변수가 정오답을 예측하는데 주요할 것이다.
  - 어플을 많이 사용한 유저의 정답율이 높을 것이다. 
  - 같은 유형의 문제의 정답율이 정오답을 예측하는데 주요할 것이다

## 모델링
1. 특성 공학(feature engineering)
    - user_count : 사용자의 누적 사용횟수
    - user_correctness_shift :	사용자의 이전까지의 누적 정답율
    - lag	: 이전 인터렉션과의 지연시간
    - watch_lec	: 강의를 봤는지 여부
    - content_count	: 문제가 사용된 총 횟수
    - content_mean :문제의 평균 정답율
    - tags1, tags2, tags3, tags4, tags5, tags6	list로 되어 있던 tags를 한 컬럼에 하나씩

2. 타겟 특성
    - answered_correctly : 정오답 여부(이진분류)

3. 학습에 사용한 특성
    - prior_question_elapsed_time, prior_question_had_explanation, part + 위의 특성 공학으로 생성한 특성

4. 실험  

    | |Chance level|RandomForest|XGBoost|LGBM|
    |--|--|--|--|--|
    |roc auc score|0.5|0.753|0.773|0.773|	

      * XGboost와 LGBM의 성능은 큰 차이가 없었으나 하이퍼파라미터 튜닝과 모델 검증을 고려하여 학습속도가 더 빠른 LGBM 선택

5. 모델 검증
    - Kfold CV는 순차적 데이터에 적합하지 않기 때문에 1 step ahead cross validation 이용
    - 1 step ahead cross validation : 이전 데이터들을 학습하여 바로 다음의 값을 예측하고, 예측한 값을 다시 훈련 데이터와 합하여 그 다음을 예측하는 방법
    - 총 15번의 검증에서 모델은 0.774에서 0.778 사이의 안정적인 성능을 보여줌

6. 최종 모델
    - train set : 각 유저의 타임라인 이전 85% 데이터
    - test set : 각 유저의 타임라인 이후 15% 데이터
    - 결과 : roc auc score train - 0.78 / test - 0.772

## 결과
shap value로 특성 중요도를 봤을 때  

  * contents_mean, user_corectness_shift, lag, wahtch_lec이 상위 4개의 중요한 feature로 나왔고  
  * prior_question_elapsed_time, user_count, part, tag는 상대적으로 덜 중요한 feature였음.  
  * 처음 세운 가설처럼 시간과 관련된 lag의 특성 중요도가 높게 나왔지만 prior_question_elapsed_time은 그렇지 않음. 전반적으로 가설과는 다른 결과를 보여주었음.  
    
    
<img src = "https://user-images.githubusercontent.com/75404758/134125848-f64864f3-ed5f-48da-9977-ecde90125e2b.png" width="50%" height="50%"/>

## 한계 및 발전 방향
1. lag의 중요도가 상대적으로 높게 나왔지만 데이터의 분산이 매우 큰 feature이다. 전처리를 더 했어야 정확한 영향력을 볼 수 있었을 것 같다.

2. 현재 모델에서는 part와 tag의 특성중요도가 낮게 나왔지만 경험적으로 생각해보면 문제 유형은 정오답에 영향을 미칠 것 같다.
해당 feature의 영향력을 더 반영할 수 있는 모델이 필요할 것 같다.

3. 순차적 데이터이기 때문에 딥러닝을 적용해 볼 계획이었는데 시간 상 진행하지 못했다. 향후에 딥러닝을 적용해 모델을 더 발전시킬 수 있을 것이라고 기대한다.






