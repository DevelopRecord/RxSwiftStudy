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
