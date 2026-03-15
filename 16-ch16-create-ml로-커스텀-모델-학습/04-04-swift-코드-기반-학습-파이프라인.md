# Swift 코드 기반 학습 파이프라인

> Create ML 프레임워크와 Create ML Components로 코드 기반 ML 학습 파이프라인을 구축하고, MLJob으로 학습을 모니터링하며, 하이퍼파라미터 튜닝과 데이터 증강을 자동화합니다.

## 개요

이 섹션에서는 Create ML의 코드 기반 학습을 한 단계 끌어올립니다. 앞서 [이미지 분류 모델 학습](16-ch16-create-ml로-커스텀-모델-학습/02-02-이미지-분류-모델-학습.md)에서 `MLImageClassifier`, [텍스트 분류와 표 형식 모델](16-ch16-create-ml로-커스텀-모델-학습/03-03-텍스트-분류와-표-형식-모델.md)에서 `MLTextClassifier`를 Swift 코드로 학습하는 방법을 배웠습니다. 이번에는 거기서 더 나아가 **MLJob을 통한 실시간 학습 모니터링**, **Create ML Components의 모듈형 파이프라인 조합**, 그리고 **하이퍼파라미터 튜닝 자동화**를 다룹니다.

[Create ML 개요와 워크플로](16-ch16-create-ml로-커스텀-모델-학습/01-01-create-ml-개요와-워크플로.md)에서 소개한 4단계 워크플로를 완전히 프로그래매틱하게 제어하는 방법을 익히게 됩니다.

**선수 지식**: Create ML의 기본 워크플로(Ch16.1~16.3), Swift Concurrency(async/await)
**학습 목표**:
- MLJob을 사용하여 학습 진행률과 메트릭을 실시간 모니터링한다
- Create ML Components의 Transformer/Estimator 패턴으로 커스텀 파이프라인을 조합한다
- 프로그래매틱 데이터 증강과 하이퍼파라미터 튜닝을 구현한다
- 체크포인트 기반 학습 중단/재개를 활용한다

> 💡 **Combine 미니 입문**: 이 섹션에서 MLJob은 Combine 프레임워크의 Publisher를 사용합니다. Combine을 처음 접하더라도 걱정 마세요. 핵심만 짚으면 — **Publisher**는 시간에 걸쳐 값을 내보내는 파이프이고, **`.sink`**는 그 파이프 끝에서 값을 받아 처리하는 구독자입니다. `store(in: &subscriptions)`는 구독을 메모리에 유지하는 장치입니다. 아래 코드에서 이 세 가지만 기억하면 충분합니다. 나중에 Apple의 [Combine 공식 문서](https://developer.apple.com/documentation/combine)에서 더 깊이 배울 수 있습니다.

## 왜 알아야 할까?

앞선 섹션에서 `MLImageClassifier.train()`이나 `MLTextClassifier()`로 모델을 학습해봤죠. 그런데 학습이 돌아가는 동안 "지금 몇 퍼센트나 진행됐지?", "정확도가 올라가고 있긴 한 건가?" 궁금했던 적 없나요? 또한 수십 개의 하이퍼파라미터 조합을 일일이 바꿔가며 테스트하거나, CI/CD 파이프라인에서 자동으로 모델을 재학습해야 할 때는 단순 `train()` 호출만으로는 부족합니다.

이번 섹션에서 배우는 **MLJob**은 학습 과정에 대한 실시간 가시성과 제어를 제공하고, **Create ML Components**는 전처리-특징추출-학습을 레고 블록처럼 자유롭게 조합하는 파이프라인을 가능하게 합니다. 이 두 가지를 마스터하면 **재현 가능하고, 자동화 가능하며, 버전 관리가 되는** ML 파이프라인을 구축할 수 있습니다.

특히 [Foundation Models + Core ML 하이브리드](17-ch17-foundation-models-core-ml-하이브리드/01-01-하이브리드-아키텍처-설계-전략.md)에서 다룰 하이브리드 앱을 만들 때, 코드로 학습한 모델을 앱에 바로 통합하는 워크플로가 핵심이 됩니다.

## 핵심 개념

### 개념 1: MLJob — 학습의 리모컨

> 💡 **비유**: MLJob은 세탁기의 제어판과 같습니다. 세탁이 시작되면 남은 시간(progress), 현재 단계(checkpoint), 완료 상태(result)를 실시간으로 볼 수 있죠. 세탁 중간에 멈출 수도(cancel) 있고, 정전 후 이어서 돌릴 수도(resume) 있습니다. MLJob은 ML 학습에 대해 정확히 같은 역할을 합니다.

앞선 섹션에서는 `train()` 호출 후 결과만 받았다면, MLJob은 **학습 과정 자체**를 관찰하고 제어하는 객체입니다. `train()`에 `sessionParameters`를 추가하면 MLJob 인스턴스가 반환되며, 세 가지 Combine Publisher를 통해 학습을 관찰합니다.

> 📊 **그림 1**: MLJob의 세 가지 Publisher 구조

```mermaid
flowchart TD
    A["train() 호출"] --> B["MLJob 생성"]
    B --> C["progress Publisher<br/>실시간 진행률"]
    B --> D["checkpoints Publisher<br/>중간 저장점"]
    B --> E["result Publisher<br/>최종 모델/에러"]
    C --> F["fractionCompleted<br/>iteration, accuracy"]
    D --> G["MLCheckpoint<br/>모델 스냅샷"]
    E --> H["학습 완료 모델<br/>또는 에러"]
    G --> I["중간 모델 생성<br/>학습 재개"]
```

MLJob을 사용한 기본 학습 코드를 살펴보겠습니다:

```swift
import CreateML
import Combine

// 학습 세션 매개변수 설정
let sessionParams = MLTrainingSessionParameters(
    sessionDirectory: sessionDirectory,  // 체크포인트 저장 경로
    reportInterval: 10,                  // 10 이터레이션마다 진행 보고
    checkpointInterval: 100,             // 100 이터레이션마다 체크포인트 저장
    iterations: 1000                     // 총 학습 이터레이션
)

// 학습 시작 — MLJob 반환
let job = try MLImageClassifier.train(
    trainingData: .labeledDirectories(at: trainingURL),
    parameters: modelParams,
    sessionParameters: sessionParams
)

var subscriptions = Set<AnyCancellable>()

// result Publisher로 완료 처리
// .sink는 Publisher의 값을 받아 처리하는 구독자입니다
job.result.sink { completion in
    switch completion {
    case .finished:
        print("학습이 정상 완료되었습니다")
    case .failure(let error):
        print("학습 중 에러 발생: \(error)")
    }
} receiveValue: { model in
    // 최종 모델 저장
    try? model.write(to: outputURL)
}
.store(in: &subscriptions)  // 구독을 메모리에 유지
```

> 💡 **async/await 대안**: Combine에 익숙하지 않다면, `job.result`를 `for await`로 소비할 수도 있습니다. 다만 MLJob의 공식 API는 Combine Publisher 기반이므로, 위 패턴을 기본으로 사용하는 것을 권장합니다.

### 개념 2: 학습 모니터링 — Progress와 Checkpoint

> 💡 **비유**: 마라톤을 생각해보세요. Progress는 GPS 워치처럼 현재 페이스(accuracy), 거리(iteration), 남은 거리를 실시간으로 보여줍니다. Checkpoint는 중간 급수대처럼, 체크포인트마다 현재 상태를 저장해두어 넘어져도(크래시) 그 지점부터 다시 달릴 수 있게 해줍니다.

> 📊 **그림 2**: 학습 모니터링 흐름

```mermaid
sequenceDiagram
    participant App as Swift 앱
    participant Job as MLJob
    participant Model as 모델 학습

    App->>Job: train() 호출
    Job->>Model: 학습 시작

    loop 매 reportInterval 이터레이션
        Model-->>Job: 진행률 업데이트
        Job-->>App: progress Publisher 이벤트
        Note right of App: iteration, accuracy,<br/>loss 등 메트릭 확인
    end

    loop 매 checkpointInterval 이터레이션
        Model-->>Job: 체크포인트 생성
        Job-->>App: checkpoints Publisher 이벤트
        Note right of App: 중간 모델 저장<br/>재개 지점 확보
    end

    Model-->>Job: 학습 완료
    Job-->>App: result Publisher 이벤트
    Note right of App: 최종 .mlmodel 저장
```

Progress 모니터링은 `MLProgress` 헬퍼 타입을 통해 ML 관련 메트릭에 접근합니다:

```swift
// Progress 모니터링 — 실시간 메트릭 확인
job.progress
    .publisher(for: \.fractionCompleted)
    .sink { [weak job] fractionCompleted in
        guard let job = job,
              let progress = MLProgress(progress: job.progress) else {
            return
        }
        
        // 기본 진행률
        let percent = Int(fractionCompleted * 100)
        print("진행률: \(percent)%")
        
        // ML 관련 메트릭
        let iteration = progress.itemCount
        let total = progress.totalItemCount ?? 0
        print("이터레이션: \(iteration) / \(total)")
        
        // 태스크별 메트릭 (정확도, 손실 등)
        if let accuracy = progress.metrics[.accuracy] as? Double {
            print("정확도: \(String(format: "%.2f%%", accuracy * 100))")
        }
    }
    .store(in: &subscriptions)
```

Checkpoint Publisher는 학습 중단/재개와 중간 모델 추출에 사용됩니다:

```swift
// Checkpoint 관찰 — 중간 모델 추출 가능
job.checkpoints
    .sink { checkpoint in
        // 학습 단계의 체크포인트만 처리
        guard checkpoint.phase == .training else { return }
        
        // 체크포인트에서 중간 모델 생성
        if let model = try? MLImageClassifier(checkpoint: checkpoint) {
            let url = sessionDirectory
                .appendingPathComponent("checkpoint_\(checkpoint.iteration).mlmodel")
            try? model.write(to: url)
            print("체크포인트 모델 저장: iteration \(checkpoint.iteration)")
        }
    }
    .store(in: &subscriptions)

// 학습 취소 (필요 시)
// job.cancel()
```

학습 중단 후 재개하는 것도 간단합니다. 같은 `sessionDirectory`를 지정하면 Create ML이 자동으로 마지막 체크포인트를 찾아 이어서 학습합니다:

```swift
// 이전 세션에서 재개
let resumedJob = try MLImageClassifier.train(
    trainingData: .labeledDirectories(at: trainingURL),
    parameters: modelParams,
    sessionParameters: sessionParams  // 같은 sessionDirectory 지정
)
// 마지막 체크포인트부터 자동으로 이어서 학습
```

### 개념 3: Create ML Components — 레고 블록 파이프라인

> 💡 **비유**: Create ML Components는 레고 블록과 같습니다. 각 블록(컴포넌트)에는 입력 돌기와 출력 돌기가 있어서, 돌기 모양이 맞으면 자유롭게 연결할 수 있습니다. `ImageFeaturePrint`라는 블록 위에 `LinearRegressor` 블록을 쌓으면 이미지 회귀 모델이 되고, `SaliencyCropper` 블록을 먼저 깔면 주요 영역을 잘라낸 뒤 특징을 추출하는 파이프라인이 됩니다.

앞선 섹션에서 사용한 `MLImageClassifier`나 `MLTextClassifier`는 **완성된 태스크** — 전처리부터 학습까지 내부에서 알아서 처리하는 "올인원" 타입이었습니다. Create ML Components(WWDC 2022 도입)는 그 태스크를 구성하는 **개별 부품**을 노출하여, 개발자가 원하는 대로 파이프라인을 조합할 수 있게 합니다.

두 가지 핵심 프로토콜이 있습니다:

- **Transformer**: 입력을 즉시 변환하는 컴포넌트 (예: 이미지 크롭, 스케일링)
- **Estimator**: 데이터를 학습하여 Transformer를 생성하는 컴포넌트 (예: 회귀, 분류기)

> 📊 **그림 3**: Create ML Components의 Transformer/Estimator 패턴

```mermaid
flowchart LR
    subgraph Transformer
        T1["입력 데이터"] --> T2["변환 로직"] --> T3["출력 데이터"]
    end

    subgraph Estimator
        E1["학습 데이터"] --> E2["fitted() 학습"] --> E3["학습된 Transformer"]
    end

    subgraph 파이프라인 조합
        P1["SaliencyCropper<br/>(Transformer)"] --> P2["ImageFeaturePrint<br/>(Estimator)"] --> P3["LinearRegressor<br/>(Estimator)"]
    end
```

컴포넌트를 연결할 때는 `.appending()` 메서드를 사용합니다:

```swift
import CreateMLComponents
import CoreImage

// 파이프라인 조합: 전처리 → 특징 추출 → 회귀
let estimator = SaliencyCropper()              // 1. 주요 영역 크롭 (Transformer)
    .appending(ImageFeaturePrint())            // 2. 특징 벡터 추출 (Estimator)
    .appending(LinearRegressor())              // 3. 선형 회귀 학습 (Estimator)

// 데이터 로딩 — 파일명에서 라벨 자동 추출
// 예: "banana-5.jpg" → annotation = 5.0
let data = try AnnotatedFiles(
    labeledByNamesAt: trainingDataURL,
    separator: "-",
    index: 1,
    type: .image
)
.mapFeatures(ImageReader.read)       // CIImage로 변환
.mapAnnotations { Float($0)! }       // 문자열 → 숫자 변환

// 80/20 분할 후 학습
let (training, validation) = data.randomSplit(by: 0.8)

let transformer = try await estimator.fitted(
    to: training,
    validateOn: validation
) { event in
    // 학습 중 메트릭 콜백
    if let error = event.metrics[.validationMaximumError] {
        print("검증 최대 오차: \(error)")
    }
}

// 모델 저장 및 로드
try estimator.write(transformer, to: parametersURL)
let loaded = try estimator.read(from: parametersURL)
```

> 📊 **그림 4**: MLImageClassifier vs Create ML Components 비교

```mermaid
flowchart TD
    subgraph 올인원["MLImageClassifier (올인원)"]
        A1["train()"] --> A2["내부 전처리<br/>+ 특징 추출<br/>+ 학습"]
        A2 --> A3["완성 모델"]
    end

    subgraph 조합["Create ML Components (조합형)"]
        B1["SaliencyCropper"] --> B2["ImageFeaturePrint"]
        B2 --> B3["LinearRegressor"]
        B3 --> B4["커스텀 모델"]
        B5["커스텀 전처리<br/>원하는 컴포넌트 교체 가능"] -.-> B1
    end

    style 올인원 fill:#e3f2fd
    style 조합 fill:#e8f5e9
```

### 개념 4: 표 형식 데이터를 위한 Components 파이프라인

> 💡 **비유**: 표 형식 데이터 파이프라인은 공장의 조립 라인과 비슷합니다. 원재료(raw data)가 여러 가공 스테이션을 거치는데, 각 스테이션은 자기 담당 컬럼만 처리합니다. `ColumnSelector`는 "이 컬럼은 이 스테이션으로"라고 라우팅하는 분류 장치인 셈이죠.

> 📊 **그림 5**: 표 형식 데이터의 컬럼별 전처리 파이프라인

```mermaid
flowchart TD
    A["DataFrame<br/>원본 데이터"] --> B["ColumnSelector"]
    B --> C["숫자형 컬럼<br/>volume"]
    B --> D["범주형 컬럼<br/>type, region"]
    C --> E["OptionalUnwrapper<br/>nil 처리"]
    E --> F["StandardScaler<br/>정규화"]
    D --> G["자동 인코딩"]
    F --> H["BoostedTreeRegressor"]
    G --> H
    H --> I["예측값<br/>price"]
```

```swift
import CreateMLComponents
import TabularData

// 컬럼 ID 정의
let priceColumnID = ColumnID("price", Double.self)

// 표 형식 파이프라인 구성
var task: some SupervisedTabularEstimator {
    // 숫자형 컬럼: nil 제거 → 정규화
    let numeric = ColumnSelector(
        columns: ["volume"],
        estimator: OptionalUnwrapper()
            .appending(StandardScaler<Double>())
    )
    
    // 범주형 + 숫자형 → 회귀
    let regression = BoostedTreeRegressor<String>(
        annotationColumnName: priceColumnID.name,
        featureColumnNames: ["type", "region", "volume"]
    )
    
    return numeric.appending(regression)
}

// 학습 실행
let dataFrame = try DataFrame(contentsOfCSVFile: dataURL)
let (training, validation) = dataFrame.randomSplit(by: 0.8)

let transformer = try await task.fitted(
    to: DataFrame(training),
    validateOn: DataFrame(validation)
) { event in
    if let error = event.metrics[.validationError] as? Double {
        print("검증 오차: \(error)")
    }
}

// 저장 후 예측
try task.write(transformer, to: parametersURL)
```

### 개념 5: 프로그래매틱 데이터 증강

> 💡 **비유**: 데이터 증강은 같은 사진으로 여러 각도의 셀카를 만드는 것과 같습니다. 원본 사진 한 장에서 회전, 확대, 축소된 버전을 만들어 모델에게 "이것도 같은 대상이야"라고 학습시키면, 실전에서 다양한 조건에서도 잘 인식하게 됩니다.

Create ML Components에서는 `flatMap`을 사용하여 커스텀 증강을 적용합니다:

> 📊 **그림 6**: 데이터 증강을 통한 학습 데이터 확장

```mermaid
flowchart LR
    A["원본 이미지 1장"] --> B["증강 함수"]
    B --> C["원본"]
    B --> D["회전<br/>-pi ~ pi"]
    B --> E["스케일<br/>0.8x ~ 1.2x"]
    C --> F["학습 데이터<br/>3배 확장"]
    D --> F
    E --> F
```

```swift
import CoreImage
import CreateMLComponents

// 커스텀 증강 함수
func augment(
    _ original: AnnotatedFeature<CIImage, Float>
) -> [AnnotatedFeature<CIImage, Float>] {
    // 랜덤 회전
    let angle = CGFloat.random(in: -.pi ... .pi)
    let rotated = original.feature
        .transformed(by: .init(rotationAngle: angle))
    
    // 랜덤 스케일
    let scale = CGFloat.random(in: 0.8 ... 1.2)
    let scaled = original.feature
        .transformed(by: .init(scaleX: scale, y: scale))
    
    return [
        original,  // 원본 유지
        AnnotatedFeature(feature: rotated, annotation: original.annotation),
        AnnotatedFeature(feature: scaled, annotation: original.annotation),
    ]
}

// 학습 데이터에 증강 적용
let data = try AnnotatedFiles(
    labeledByNamesAt: trainingDataURL,
    separator: "-", index: 1, type: .image
)
.mapFeatures(ImageReader.read)
.mapAnnotations { Float($0)! }
.flatMap(augment)  // 각 이미지가 3개로 확장됨

let (training, validation) = data.randomSplit(by: 0.8)

let estimator = SaliencyCropper()
    .appending(ImageFeaturePrint())
    .appending(LinearRegressor())

let transformer = try await estimator.fitted(
    to: training,
    validateOn: validation
) { event in
    if let trainErr = event.metrics[.trainingMaximumError],
       let valErr = event.metrics[.validationMaximumError] {
        print("학습 최대 오차: \(trainErr), 검증 최대 오차: \(valErr)")
    }
}

// 검증 메트릭 계산
let validationError = try await meanAbsoluteError(
    transformer.applied(to: validation.map(\.feature)),
    validation.map(\.annotation)
)
print("평균 절대 오차(MAE): \(validationError)")
```

### 개념 6: CI/CD 자동화와 학습 스크립트

> 💡 **비유**: CI/CD에서의 모델 학습 자동화는 공장의 야간 자동 생산 라인과 같습니다. 코드가 푸시되면 자동으로 데이터를 가져오고, 모델을 학습하고, 품질 기준을 통과하면 앱 번들에 포함시키는 것이죠. 사람이 매번 개입하지 않아도 됩니다.

앞선 섹션들에서 배운 코드 기반 학습의 가장 큰 장점은 바로 **자동화 가능성**입니다. MLJob의 체크포인트, 메트릭 기반 조기 종료, 하이퍼파라미터 그리드 서치를 결합하면 완전 자동화된 학습 파이프라인을 구축할 수 있습니다.

> 📊 **그림 7**: 자동화된 모델 학습 파이프라인

```mermaid
flowchart TD
    A["코드 푸시 / 스케줄"] --> B["데이터 로딩<br/>+ 검증"]
    B --> C["하이퍼파라미터<br/>그리드 정의"]
    C --> D["순차 학습 실행<br/>MLJob + 모니터링"]
    D --> E{"정확도 기준<br/>통과?"}
    E -->|Yes| F["최적 모델 선택<br/>.mlmodel 저장"]
    E -->|No| G["알림 전송<br/>+ 로그 기록"]
    F --> H["앱 번들 포함<br/>또는 OTA 배포"]
```

```swift
import CreateML
import Foundation

// 자동화 스크립트용 — Combine 없이 동기 학습
// (CLI/CI 환경에서 사용)
func trainModelSync(
    dataURL: URL,
    outputURL: URL,
    maxIterations: Int,
    targetAccuracy: Double
) throws -> (model: MLImageClassifier, accuracy: Double) {
    
    let params = MLImageClassifier.ModelParameters(
        validation: .split(strategy: .automatic),
        maxIterations: maxIterations,
        augmentationOptions: [.crop, .rotation, .flip]
    )
    
    // 동기 학습 — sessionParameters 없이 호출하면
    // MLJob 대신 직접 모델이 반환됨
    let classifier = try MLImageClassifier(
        trainingData: .labeledDirectories(at: dataURL),
        parameters: params
    )
    
    // 학습/검증 메트릭 확인
    let trainingAccuracy = (1.0 - classifier.trainingMetrics.classificationError) * 100
    let validationAccuracy = (1.0 - classifier.validationMetrics.classificationError) * 100
    
    print("학습 정확도: \(String(format: "%.1f%%", trainingAccuracy))")
    print("검증 정확도: \(String(format: "%.1f%%", validationAccuracy))")
    
    // 기준 미달 시 경고
    guard validationAccuracy >= targetAccuracy else {
        print("⚠️ 목표 정확도(\(targetAccuracy)%) 미달")
        throw TrainingError.belowThreshold(actual: validationAccuracy, target: targetAccuracy)
    }
    
    // 모델 저장
    try classifier.write(to: outputURL)
    return (classifier, validationAccuracy)
}

enum TrainingError: Error {
    case belowThreshold(actual: Double, target: Double)
}
```

## 실습: 직접 해보기

하이퍼파라미터 조합을 자동으로 탐색하는 완전한 학습 파이프라인을 구축합니다. 이미지 분류기의 `maxIterations`와 데이터 증강 여부를 조합하여 최적의 설정을 찾습니다.

```swift
import CreateML
import Foundation
import Combine

// MARK: - 하이퍼파라미터 튜닝 파이프라인

/// 하이퍼파라미터 조합 정의
struct HyperparameterConfig: CustomStringConvertible {
    let maxIterations: Int
    let augmentationEnabled: Bool
    let validationRatio: Double
    
    var description: String {
        "iter=\(maxIterations), aug=\(augmentationEnabled), val=\(validationRatio)"
    }
}

/// 학습 결과 기록
struct TrainingResult {
    let config: HyperparameterConfig
    let accuracy: Double
    let trainingTime: TimeInterval
    let modelURL: URL
}

/// 하이퍼파라미터 그리드 정의
let configs: [HyperparameterConfig] = [
    .init(maxIterations: 10, augmentationEnabled: false, validationRatio: 0.2),
    .init(maxIterations: 25, augmentationEnabled: false, validationRatio: 0.2),
    .init(maxIterations: 25, augmentationEnabled: true,  validationRatio: 0.2),
    .init(maxIterations: 50, augmentationEnabled: true,  validationRatio: 0.15),
]

/// 단일 학습 실행 함수
func trainWithConfig(
    _ config: HyperparameterConfig,
    trainingURL: URL,
    outputDir: URL
) throws -> TrainingResult {
    let startTime = Date()
    var subscriptions = Set<AnyCancellable>()
    var finalAccuracy: Double = 0.0
    
    // 증강 옵션 설정
    var augOptions: MLImageClassifier.ImageAugmentationOptions = []
    if config.augmentationEnabled {
        augOptions = [.crop, .rotation, .blur, .exposure, .noise, .flip]
    }
    
    // 모델 파라미터 구성
    let params = MLImageClassifier.ModelParameters(
        validation: .split(strategy: .automatic),
        maxIterations: config.maxIterations,
        augmentationOptions: augOptions
    )
    
    // 세션 파라미터 — 진행 모니터링용
    let sessionDir = outputDir.appendingPathComponent("session_\(config.maxIterations)")
    let sessionParams = MLTrainingSessionParameters(
        sessionDirectory: sessionDir,
        reportInterval: 5,
        checkpointInterval: 10,
        iterations: config.maxIterations
    )
    
    // 학습 시작
    let job = try MLImageClassifier.train(
        trainingData: .labeledDirectories(at: trainingURL),
        parameters: params,
        sessionParameters: sessionParams
    )
    
    // 진행률 모니터링
    job.progress
        .publisher(for: \.fractionCompleted)
        .sink { [weak job] fraction in
            guard let job = job,
                  let progress = MLProgress(progress: job.progress) else { return }
            
            let iteration = progress.itemCount
            let total = progress.totalItemCount ?? 0
            
            if let acc = progress.metrics[.accuracy] as? Double {
                finalAccuracy = acc
                print("  [\(config)] iter \(iteration)/\(total) — 정확도: \(String(format: "%.1f%%", acc * 100))")
            }
        }
        .store(in: &subscriptions)
    
    // 동기적으로 결과 대기 (Playground/CLI 환경)
    let semaphore = DispatchSemaphore(value: 0)
    var trainedModel: MLImageClassifier?
    
    job.result.sink { completion in
        if case .failure(let error) = completion {
            print("  에러: \(error.localizedDescription)")
        }
        semaphore.signal()
    } receiveValue: { model in
        trainedModel = model
    }
    .store(in: &subscriptions)
    
    semaphore.wait()
    
    // 모델 저장
    let modelURL = outputDir.appendingPathComponent("model_iter\(config.maxIterations).mlmodel")
    try trainedModel?.write(to: modelURL)
    
    let elapsed = Date().timeIntervalSince(startTime)
    
    return TrainingResult(
        config: config,
        accuracy: finalAccuracy,
        trainingTime: elapsed,
        modelURL: modelURL
    )
}

// MARK: - 그리드 서치 실행

let trainingURL = URL(fileURLWithPath: "/path/to/training-data")
let outputDir = URL(fileURLWithPath: "/path/to/output")

var results: [TrainingResult] = []

for (index, config) in configs.enumerated() {
    print("\n📌 실험 \(index + 1)/\(configs.count): \(config)")
    print(String(repeating: "-", count: 50))
    
    let result = try trainWithConfig(config, trainingURL: trainingURL, outputDir: outputDir)
    results.append(result)
    
    print("  ✔ 완료 — 정확도: \(String(format: "%.1f%%", result.accuracy * 100)), 소요: \(String(format: "%.1f", result.trainingTime))초")
}

// MARK: - 결과 비교

print("\n" + String(repeating: "=", count: 60))
print("하이퍼파라미터 튜닝 결과 요약")
print(String(repeating: "=", count: 60))

let sorted = results.sorted { $0.accuracy > $1.accuracy }
for (rank, result) in sorted.enumerated() {
    let medal = rank == 0 ? "🏆" : "  "
    print("\(medal) #\(rank + 1) | \(result.config) | 정확도: \(String(format: "%.1f%%", result.accuracy * 100)) | \(String(format: "%.1f", result.trainingTime))초")
}

if let best = sorted.first {
    print("\n최적 모델: \(best.modelURL.lastPathComponent)")
}
```

```run:swift
// 실행 결과 시뮬레이션 (실제 학습 데이터 필요)
let configs = [
    ("iter=10, aug=false", 78.5, 12.3),
    ("iter=25, aug=false", 85.2, 28.7),
    ("iter=25, aug=true",  89.1, 35.4),
    ("iter=50, aug=true",  91.3, 68.9),
]

print("하이퍼파라미터 튜닝 결과 요약")
print(String(repeating: "=", count: 55))
for (i, (config, acc, time)) in configs.sorted(by: { $0.1 > $1.1 }).enumerated() {
    let medal = i == 0 ? "🏆" : "  "
    print("\(medal) #\(i+1) | \(config) | 정확도: \(acc)% | \(time)초")
}
print("\n최적 모델: model_iter50.mlmodel")
```

```output
하이퍼파라미터 튜닝 결과 요약
=======================================================
🏆 #1 | iter=50, aug=true | 정확도: 91.3% | 68.9초
   #2 | iter=25, aug=true | 정확도: 89.1% | 35.4초
   #3 | iter=25, aug=false | 정확도: 85.2% | 28.7초
   #4 | iter=10, aug=false | 정확도: 78.5% | 12.3초

최적 모델: model_iter50.mlmodel
```

## 더 깊이 알아보기

### Create ML의 탄생 — "ML을 모든 개발자에게"

Create ML은 2018년 WWDC에서 처음 공개되었습니다. 당시 Apple의 ML 팀 엔지니어인 Liz Danzico는 "ML 전문가가 아닌 앱 개발자도 몇 줄의 코드로 모델을 학습할 수 있어야 한다"는 비전을 제시했죠. 초기 버전은 Swift Playground에서 바로 실행할 수 있을 만큼 간결한 API가 핵심이었습니다.

그러나 실무 개발자들의 피드백은 분명했습니다 — "학습 중에 무슨 일이 일어나는지 모르겠다." 이에 Apple은 2020년 WWDC에서 `MLJob`과 `MLTrainingSessionParameters`를 추가하여 학습 과정에 대한 완전한 가시성과 제어를 제공했습니다.

2022년에는 더 나아가 Create ML Components를 도입했습니다. 이전까지의 Create ML은 "이미지 분류기", "텍스트 분류기" 같은 **완성된 태스크** 단위였다면, Components는 그 태스크를 구성하는 **개별 부품**을 노출한 것입니다. 이 철학은 Unix의 "작은 프로그램을 조합하라"는 원칙과 놀랍도록 닮아 있습니다.

### Combine과 ML의 만남

MLJob이 Combine 프레임워크를 채택한 것은 우연이 아닙니다. ML 학습은 본질적으로 **시간에 걸쳐 값을 발행하는 비동기 프로세스**입니다 — 매 이터레이션마다 메트릭이 변하고, 주기적으로 체크포인트가 생성되며, 최종적으로 결과가 나옵니다. 이것은 Combine의 Publisher-Subscriber 패턴에 완벽하게 들어맞습니다. Apple은 기존 프레임워크의 강점을 ML 영역에 자연스럽게 확장한 셈이죠.

참고로 Apple은 최근 Swift Concurrency(async/await, AsyncSequence)를 더 강조하는 추세이며, Create ML Components의 `fitted()` 메서드가 `async` 함수인 것도 이러한 흐름의 일부입니다. MLJob의 Combine Publisher는 현재 API에서 유일하게 Combine이 필요한 부분이므로, 이 패턴만 익혀두면 충분합니다.

## 흔한 오해와 팁

> ⚠️ **흔한 오해**: "Create ML Components는 Create ML 프레임워크를 완전히 대체한다"
> 아닙니다. Create ML 프레임워크(`MLImageClassifier` 등)는 검증된 기본 파이프라인을 빠르게 사용할 때, Components는 커스텀 전처리나 비표준 파이프라인이 필요할 때 사용합니다. 둘은 보완 관계이며, 프로젝트 요구사항에 따라 선택하세요.

> ⚠️ **흔한 오해**: "이번 섹션은 GUI 작업을 코드로 바꾸는 것이다"
> 16.2와 16.3에서 이미 Swift 코드로 모델을 학습했습니다. 이번 섹션이 추가하는 것은 **MLJob을 통한 실시간 모니터링**, **Components의 모듈형 파이프라인 조합**, **하이퍼파라미터 자동 탐색** 등 프로덕션 수준의 제어와 자동화입니다.

> 💡 **알고 계셨나요?**: MLJob의 체크포인트 시스템은 Xcode Cloud와 통합됩니다. CI/CD 파이프라인에서 모델 학습을 자동화할 때, 빌드가 중간에 실패해도 체크포인트에서 이어 학습할 수 있어 비용과 시간을 크게 절약합니다.

> 🔥 **실무 팁**: 하이퍼파라미터 튜닝 시 `reportInterval`을 작게(5~10) 설정하세요. 초기 이터레이션에서 학습률이 명확히 떨어지면 해당 설정을 빠르게 폐기하고 다음 조합으로 넘어갈 수 있습니다. 이 "조기 폐기(early discarding)" 전략만으로도 튜닝 시간을 절반 이상 줄일 수 있습니다.

> 🔥 **실무 팁**: Create ML Components의 `estimator.write(transformer, to:)`는 `.mlmodel`이 아닌 독자 포맷(`.pkg`)으로 저장합니다. 앱 배포용 Core ML 모델이 필요하면 별도로 Core ML 변환 과정을 거치거나, 기존 Create ML 프레임워크의 `model.write(to:)`를 사용하세요.

## 핵심 정리

| 개념 | 설명 |
|------|------|
| **MLJob** | 학습 작업을 나타내는 타입. progress, checkpoints, result 세 가지 Combine Publisher 제공 |
| **MLTrainingSessionParameters** | 세션 디렉토리, 보고 간격, 체크포인트 간격, 총 이터레이션 등 학습 세션 설정 |
| **MLProgress** | Foundation Progress에서 ML 메트릭(accuracy, loss 등)을 추출하는 헬퍼 타입 |
| **Create ML Components** | Transformer/Estimator 기반의 모듈형 ML 파이프라인 조합 프레임워크 (WWDC 2022) |
| **Transformer** | 입력 → 출력 즉시 변환 컴포넌트 (예: SaliencyCropper, ImageReader) |
| **Estimator** | 데이터를 학습하여 Transformer를 생성하는 컴포넌트 (예: LinearRegressor) |
| **.appending()** | 컴포넌트를 연결하여 파이프라인을 구성하는 메서드 |
| **ColumnSelector** | 표 형식 데이터에서 특정 컬럼에 전처리를 라우팅하는 컴포넌트 |
| **flatMap 증강** | 원본 데이터를 증강 함수로 확장하여 학습 데이터를 늘리는 패턴 |
| **체크포인트 재개** | 같은 sessionDirectory를 지정하면 마지막 체크포인트에서 자동 재개 |
| **CI/CD 자동화** | 코드 기반 학습 + MLJob 체크포인트로 자동 학습 파이프라인 구축 |

## 다음 섹션 미리보기

다음 [실습: 커스텀 모델 학습부터 앱 배포까지](16-ch16-create-ml로-커스텀-모델-학습/05-05-실습-커스텀-모델-학습부터-앱-배포까지.md)에서는 이번에 배운 코드 기반 학습 파이프라인을 활용하여 **실제 앱에 탑재할 커스텀 모델을 처음부터 끝까지** 완성합니다. 데이터 수집 → 전처리 → 학습 → 평가 → Core ML 변환 → SwiftUI 앱 통합까지, 전체 파이프라인을 하나의 프로젝트로 구현하게 됩니다.

## 참고 자료

- [Control training in Create ML with Swift — WWDC20](https://developer.apple.com/videos/play/wwdc2020/10156/) - MLJob, 체크포인트, 학습 제어의 핵심 WWDC 세션
- [Get to know Create ML Components — WWDC22](https://developer.apple.com/videos/play/wwdc2022/10019/) - Transformer/Estimator 패턴과 파이프라인 조합 입문
- [Compose advanced models with Create ML Components — WWDC22](https://developer.apple.com/videos/play/wwdc2022/10020/) - ColumnSelector, 표 형식 데이터, 고급 파이프라인 구성
- [Create ML Overview — Apple Developer](https://developer.apple.com/machine-learning/create-ml/) - Create ML 프레임워크 공식 문서와 지원 태스크 목록
- [Create ML — Apple Developer Documentation](https://developer.apple.com/documentation/createml/) - 전체 API 레퍼런스 (MLJob, MLProgress, MLCheckpoint 등)
- [Discover machine learning enhancements in Create ML — WWDC23](https://developer.apple.com/videos/play/wwdc2023/10044/) - 2023년 추가된 Components 업데이트와 새로운 태스크
- [Combine — Apple Developer Documentation](https://developer.apple.com/documentation/combine) - Combine 프레임워크 공식 문서 (MLJob Publisher 패턴 이해에 참고)

---
### 🔗 Related Sessions
- [mltextclassifier](16-ch16-create-ml로-커스텀-모델-학습/03-03-텍스트-분류와-표-형식-모델.md) (prerequisite)
- [mlregressor](16-ch16-create-ml로-커스텀-모델-학습/03-03-텍스트-분류와-표-형식-모델.md) (prerequisite)
- [mldatatable](16-ch16-create-ml로-커스텀-모델-학습/03-03-텍스트-분류와-표-형식-모델.md) (prerequisite)
