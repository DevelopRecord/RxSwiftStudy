# RxSwiftStudy
RxSwift, RxCocoa를 사용한 다양한 예제를 다룹니다.   
기초 예시는 [곰튀김](https://www.youtube.com/watch?v=w5Qmie-GbiA&ab_channel=%EA%B3%B0%ED%8A%80%EA%B9%80)님의 RxSwift 영상을 보면서 정리하였습니다.
   
먼저 기존의 Sync, Async 방식의 처리는 아래와 같습니다. 

불러올 사진의 링크는 https://picsum.photos/ 입니다.
### Sync
```
@IBAction func onLoadSync(_ sender: Any) {
    let image = loadImage(from: IMAGE_URL)
    imageView.image = image
}

private func loadImage(from imageUrl: String) -> UIImage? {
    guard let url = URL(string: imageUrl) else { return nil }
    guard let data = try? Data(contentsOf: url) else { return nil }

    let image = UIImage(data: data)
    return image
}
```
네트워크 작업을 동기적으로 하면 앱이 정지된 것처럼 보입니다.
사진 아래의 숫자 카운트가 사진을 불러오는 동안 정지되는 것을 보면 알수 있습니다.

### Async
```
@IBAction func onLoadAsync(_ sender: Any) {
    // TODO: async
    DispatchQueue.global().async {
        let image = self.loadImage(from: self.IMAGE_URL)
        
        DispatchQueue.main.async {
            self.imageView.image = image
        }
    }
}
```
GCD의 DispatchQueue를 사용하여 비동기적으로 처리하는 코드블럭을 추가했습니다.
또한 반드시 메인 스레드에서 작업해야 할 내용(주로 UIKit)은 따로 메인 스레드에서 처리 해줍니다.

### PromiseKit의 Async
```
@IBAction func onLoadImage(_ sender: Any) {
    imageView.image = nil

    promiseLoadImage(from: LARGER_IMAGE_URL)
        .done { image in
            self.imageView.image = image
        }.catch { error in
            print(error.localizedDescription)
        }
}

func promiseLoadImage(from imageUrl: String) -> Promise<UIImage?> {
    return Promise<UIImage?>() { seal in
        asyncLoadImage(from: imageUrl) { image in
            seal.fulfill(image)
        }
    }
}
```
함수를 하나 만들고 반환타입은 Promise<UIImage?> 로 해줍니다.
그리고 값을 받을 수 있게 클로저를 만들어 줍니다.
이후 비동기적으로 사진을 받아올 수 있게 함수를 하나 만들고 받아온 image를 다시 fulfill 메서드로 받아옵니다.

사실 RxSwift의 사용 목적은 비동기 처리가 메인이고 그 비동기를 쓰기 간편하게 그리고 코드도 깔끔하게 한다는데 큰 목적이 있습니다.
또한 GCD를 통해서도 비동기적으로 처리가 가능합니다. 하지만 그럴 경우엔 상황에 맞게 사용해야 하는데 어디서는 OperationQueue, 다른 곳에선 DispatchQueue 이런식으로 코드의 일관성이 없어집니다.
유지보수에도 어려움이 존재합니다.

### RxSwift
> An API for asynchronous programming   
with observable streams

**비동기 프로그래밍을 하는 API. 관찰 가능한 흐름으로**
Rx의 주요기능은 다섯개가 존재합니다. 중요한 순서대로 보자면,
Observable, Operator, Scheduler, Subject, Single

1. Observable
어떠한 Observable 객체를 생성하고 그 객체에서 이벤트가 발생합니다. 이 이벤트를 관찰하고 다양한 처리(next, error, complete)를 합니다.

이후 Subscribe를 합니다. Subscribe 이전까지는 그저 값을 처리하는 방식을 작성한 것입니다. 즉, 정의한 것 뿐이고 값을 받아오는 그 자체가 아닙니다.
쉽게 생각해 보자면, 유튜브에서 어떤 채널을 구독해야 우리가 그 채널에 대한 영상과 알림 등을 받아볼 수 있어요.

```
func rxswiftLoadImage(from imageUrl: String) -> Observable<UIImage?> {
    return Observable.create { observer in
        asyncLoadImage(from: imageUrl) { image in
            observer.onNext(image)
            observer.onCompleted()
        }
        return Disposables.create()
    }
}
```
방식 자체는 PromiseKit과 유사합니다. 대신 리턴타입으로 Observable<UIImage?>를 리턴합니다. 
이후 create 메서드로 Observable을 생성합니다. 이렇게 생성하고 반환타입을 보면 Disposable이 반환타입이 됩니다.

#### Disposable
> 사용 후 버리게 되어 있는, 일회용의

Observable이 모든 행동을 다 취하였으면 필요없어지겠죠? 그러면 종료해야 합니다.
dispose 방식에는 아래와 같은 방식이 존재합니다.

```
var dispose: Disposable?

rxswiftLoadImage(from: LARGER_IMAGE_URL)
      .observeOn(...)
      .subscribe { result in
      ...
      }
      .dispose()
```

Disposable 타입의 객체를 변수로 받아서 dispose()메소드로 취소합니다.

```
var dispose: Disposable?
private var disposeBag = DisposeBag()

@IBAction func onLoadImage(_ sender: Any) {
    imageView.image = nil

    disposable = rxswiftLoadImage(from: LARGER_IMAGE_URL)
        .observeOn(MainScheduler.instance)
        .subscribe({ result in
            switch result {
            case let .next(image):
                self.imageView.image = image

            case let .error(err):
                print(err.localizedDescription)

            case .completed:
                break
            }
        })
}
```

만약 사진을 불러오는 도중 취소하려면 어떻게 구현해야 할까요?
```
@IBAction func onCancel(_ sender: Any) {
    // TODO: cancel image loading
    disposable?.dispose()
}
```
만들어놓은 Disposable 객체를 dispose()하면 취소가 됩니다.

#### map
map은 기존의 map과 사용방법이 같습니다. 기존 데이터를 새로운 데이터로 변형해서 스트림을 내보내는 것이죠.
```
@IBAction func exMap3() {
    Observable.just("800x600")
        .map { $0.replacingOccurrences(of: "x", with: "/") }
        .map { "https://picsum.photos/\($0)/?random" }
        .map { URL(string: $0) }
        .filter { $0 != nil }
        .map { $0! }
        .map { try Data(contentsOf: $0) }
        .map { UIImage(data: $0) }
        .subscribe(onNext: { image in
            self.imageView.image = image
        })
        .disposed(by: disposeBag)
}
```
1. 우선 첫줄부터, just를 이용해서 단일 데이터를 가져온 뒤에 map을 하는데 "800x600" -> "800/600"으로 변형시킵니다.
2. 다시 https://picsum.photos 이라는 사이트에서 변형한 사진 사이즈 정보로 다시 map 합니다.
3. 이후 URL? 형태로 바꿔주고요.
4. 그리고 filter 메서드를 사용해서 가져온 값이 nil이 아닌지 판별합니다. 이후 nil이 아니면,
5. 강제언래핑을 해도 괜찮기에 !를 붙여 강제언래핑을 합니다.
6. 그리고 URL을 Data 타입으로 변형하고요.
7. 변형한 데이터를 UIImage? 타입으로 변형합니다.
8. 이제 subscribe를 이용해 구독을 하고 외부에 영향을 주는(SideEffect) 코드를 작성합니다.
9. 마지막으로 disposeBag에 담습니다.

이렇게 작성하고 동작시키면 정상적으로 동작은 합니다. 다만, 동기적으로 동작되죠. 이미지를 불러오고 UI에 적용하는데 앱이 일시정지 된 것처럼 보입니다. 이는 Rx를 사용하는데 가장 큰 이유에서 벗어납니다.
이럴 때 사용하는 것이 **observeOn()** 입니다.

먼저 Main Thread와 Concurrency하게 처리할 부분을 구분지어야 합니다.
just에 들어온 데이터를 변형하고 필터링하는 과정은 비동기적으로 다시 말해, Concurrency하게 처리해야 합니다.
그리고 imageView에 image를 삽입하는 과정은 UI 처리와 관련된 부분이기 때문에 반드시 Main Thread에서 작업해야 합니다.
observeOn의 사용방법은 아래와 같습니다.
```
@IBAction func exMap3() {
    Observable.just("800x600")
        .observeOn(ConcurrentDispatchQueueScheduler(qos: .default) // subscribe 전까지 변형, 필터링하는 과정은 ConcurrentDispatchQueueScheduler
        .map { $0.replacingOccurrences(of: "x", with: "/") }
        .map { "https://picsum.photos/\($0)/?random" }
        .map { URL(string: $0) }
        .filter { $0 != nil }
        .map { $0! }
        .map { try Data(contentsOf: $0) }
        .map { UIImage(data: $0) }
        .observeOn(MainScheduler.instance) // 여기서부턴 데이터를 변형, 필터링 과정이 끝나고 UI에 삽입하는 과정이 남아있기에 MainScheduler
        .subscribe(onNext: { image in
            self.imageView.image = image
        })
        .disposed(by: disposeBag)
}
```

이렇게 작성하면 비동기적으로 처리하기에 UI가 멈추지도 않고 정상적으로 잘 작동됩니다.
단, 이렇게 작성하면 observeOn 밑으로의 코드만 ConcurrentDispatchQueueScheduler 혹은 MainScheduler로 작동합니다.
하지만 여기서 just에 불러오는 데이터는 간단한 String 타입의 데이터이지만, 만약 just에서부터 가져오는 데이터도 비동기적으로 작성해야 한다면?
그 경우엔 subscribeOn()을 사용하면 됩니다.
subscribeOn은 .subscribe 되는 순간에 즉, subscribe 이후부터 원하는 Scheduler로 Observable.just(...)부터 코드를 읽어 내려갑니다.
그래서 subscribeOn의 위치는 어디에 있든 영향을 받지 않습니다. 이제 마지막으로 코드를 보면 아래와 같습니다.
```
@IBAction func exMap3() {
    Observable.just("800x600")
        .map { $0.replacingOccurrences(of: "x", with: "/") }
        .map { "https://picsum.photos/\($0)/?random" }
        .map { URL(string: $0) }
        .filter { $0 != nil }
        .map { $0! }
        .map { try Data(contentsOf: $0) }
        .subscribeOn(ConcurrentDispatchQueueScheduler(qos: .default))
        .map { UIImage(data: $0) }
        .observeOn(MainScheduler.instance)
        .subscribe(onNext: { image in
            self.imageView.image = image
        })
        .disposed(by: disposeBag)
}
```
observeOn과 subscribeOn은 작업 처리를 MainScheduler에서 처리할지, Concurrency하게 처리할지 결정지을 메서드이기에 매우 중요하다고 생각합니다.
   
이제 마지막으로 Side-Effect의 개념에 대해 설명하자면, **외부에 영향을 주는** 입니다.
위 코드에서 외부에 영향을 주는 코드는 <code>self.imageView.image = image</code> 입니다.
말 그대로 외부의 imageView라는 UIImageView에 image를 전달해주고 세팅해주는 코드이니까요.
Side-Effect를 허용해주는 메서드는 두 개가 존재합니다. 첫번째로 <code>subscribe</code>, 두번째로 <code>do</code>가 존재합니다.
함수형 프로그래밍 개념에 따르면 외부에 영향을 주는 코드는 적합하지 않다고 합니다. 외부에 영향을 준다는 말은 극단적으로 보면 예측할 수 없는 곳에서 에러가 발생할 가능성이 있다는 얘기기도 합니다.
하지만 외부에 영향을 줘야 하는 경우도 반드시 있습니다. 예를 들면, 위처럼 UI에 적용해준다던지, 전역변수에 변형시킨 데이터를 넘겨준다던지 하는 경우입니다.

```
private var imageSize: CGSize?

@IBAction func exMap3() {
    Observable.just("800x600")
        .map { $0.replacingOccurrences(of: "x", with: "/") }
        .map { "https://picsum.photos/\($0)/?random" }
        .map { URL(string: $0) }
        .filter { $0 != nil }
        .map { $0! }
        .map { try Data(contentsOf: $0) }
        .subscribeOn(ConcurrentDispatchQueueScheduler(qos: .default))
        .map { UIImage(data: $0) }
        .do(onNext: { image in
            print("size: \(image?.size)")
            self.imageSize = image?.size
            print("imageSize: \(self.imageSize)")
        })
        .observeOn(MainScheduler.instance)
        .subscribe(onNext: { image in
            self.imageView.image = image
        })
        .disposed(by: disposeBag)
}
```
> size: Optional((800.0, 600.0))   
imageSize: Optional((800.0, 600.0))

하지만 외부에 영향을 주는 코드를 다른 곳에서 사용할 수는 있다고는 합니다만 권장하지는 않는다고 합니다.
