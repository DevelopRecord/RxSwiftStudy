# RxSwiftStudy
RxSwift, RxCocoa를 사용한 다양한 예제를 다룹니다.   
기초 예시는 [곰튀김](https://www.youtube.com/watch?v=w5Qmie-GbiA&ab_channel=%EA%B3%B0%ED%8A%80%EA%B9%80)님의 RxSwift 영상을 보면서 정리하였습니다.
   
먼저 기존의 Sync, Async 방식의 처리는 아래와 같습니다. 

불러올 사진의 링크는 https://picsum.photos/ 입니다.
## Sync
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

## Async
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

## PromiseKit의 Async
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

## RxSwift
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

하지만 외부에 영향을 주는 코드를 do, subscribe가 아닌 다른 곳에서도 사용할 수는 있다고는 합니다만 권장하지는 않는다고 합니다.

## RxCocoa
**RxCocoa**란?
> 애플 환경(iOS/macOS/watchOS 및 tvOS)의 애플리케이션을 개발하기 위한 Cocoa Framework 전용 기능을 제공하는 라이브러리

#### 예시
예를 들어 ID를 입력하는 텍스트필드, Password를 입력하는 텍스트필드가 있다고 가정하고 각 인풋값이 올바른지 판단해주기 위해 빨간 점으로 된 bullet이 있다고 가정합니다. 그리고 각각의 인풋값을 올바르게 입력하면 로그인 버튼이 활성화가 됩니다.
만약 RxCocoa를 사용하지 않고 작성하려면 어떻게 해야 할까요?

1. 각 텍스트필드마다 Selector함수를 만들고 UIControl Event를 생성합니다.
2. <code>UITextFieldDelegate</code>를 상속하고 <code>textFieldDidChange</code>로 키보드의 입력에 대한 이벤트를 실시간으로 받습니다.
3. 각 텍스트필드(id, pw)에 반드시 들어가야할 문자 혹은 문자열 카운트의 조건 함수를 생성합니다.
4. 조건에 부합할 시 로그인 버튼을 활성화합니다.

이런식으로 많이 했던 과정들을 사용하는데, RxCocoa를 사용하면 아래 방법으로 이용 가능합니다.
```
idField.rx.text
    .orEmpty // nil인지 판별
    .map(checkEmailValid(_:)) // 이메일 형식이 올바른지 판단
    .subscribe(onNext: { bool in // 구독하여 Boolean 형태의 값을 받고
    self.idValidView.isHidden = bool //  bullet의 Hidden 여부를 판단
})
    .disposed(by: disposeBag)
    
pwField.rx.text
    .orEmpty
    .map(checkPasswordValid(_:))
    .subscribe(onNext: { bool in
    self.pwValidView.isHidden = bool
})
    .disposed(by: disposeBag)
    
private func checkEmailValid(_ email: String) -> Bool {
    return email.contains("@") && email.contains(".")
}

private func checkPasswordValid(_ password: String) -> Bool {
    return password.count > 5
}
```

위와 같이 작성하면 id와 pw의 bullet이 각 올바른 형식에 부합하면 사라지고 올바르지 않으면 그대로 존재합니다.
이제 우리는 위 두개의 Observable을 하나로 결합해서 두 조건 모두 true일 때 로그인 버튼이 활성화 되게 만들어야 합니다.
그럴 때 사용하는 메서드가 <code>CombineLatest</code>입니다. 이것은 아래와 같은 기능을 제공합니다.
> 두 개의 Observable 중 하나가 항목을 배출할 때 배출된 마지막 항목과 다른 한 Observable이 배출한 항목을 결합한 후 함수를 적용하여 실행 후 실행된 결과를 배출한다

그럼 여기서 Observable을 하나 새로 만들고 CombineLatest 메서드를 이용해 위 두 Observable을 결합해 각각의 Source들을 넣고 모든 조건이 맞을 때 true를 반환합니다.
하나라도 조건에 부합하지 않을 시 false를 반환하고요.
```
Observable.combineLatest(
    idField.rx.text.orEmpty.map(checkEmailValid(_:)), // 조건 판별하여 나오는 bool
    pwField.rx.text.orEmpty.map(checkPasswordValid(_:))) // 동일
    { id, pw in id && pw } // id, pw의 Source가 조건에 맞을시 true, 아니면 false
    .subscribe { bool in
    self.loginButton.isEnabled = bool // 로그인 버튼의 활성화 유무 결정
}
    .disposed(by: disposeBag)
```
이제 실행해보면 정상적으로 잘 작동합니다. 하지만 중복되는 코드가 많아요. 리팩토링을 해야 합니다.

```
let idInputOb = idField.rx.text.orEmpty.asObservable() // 텍스트필드의 텍스트가 nil이 아닌지 판별후 nil 아니면 언래핑. asObservable()은 ControlProperty 타입을 Observable 타입으로
let idValidOb = idInputOb.map(checkEmailvalid(_:))
let pwInputOb = pwField.rx.text.orEmpty.asObservable()
let pwValidOb = pwInputOb.map(checkPasswordvalid(_:))

idValidOb.subscribe(onNext: { bool in
    self.idValidView.isHidden = bool
}).disposed(by: disposeBag)

pwValidOb.subscribe(onNext: { bool in
    self.pwValidView.isHidden = bool
}).disposed(by: disposeBag)

Observable.combineLatest(idValidOb, pwValidOb) { id, pw in id && pw }
    .subscribe(onNext: { bool in
    self.loginButton.isEnabled = bool
}).disposed(by: disposeBag)
```
이렇게 작성해도 같은 결과를 얻을 수 있습니다.
여기서 다시 리팩토링을 하면 Input(id, pw 입력)과 Output(Input에서 나온 결과(Boolean)을 가지고 불릿 Hidden 여부, 로그인버튼 활성화 여부)을 구분해서 리팩토링할 수도 있습니다.
먼저 전역변수로 <code>BehaviorSubject(value:_)</code>를 사용합니다. BehaviorSubject는 기본적으로 PublishSubject와 유사하지만 기본값을 갖는다는 차이점이 존재합니다.
따라서 아래와 같이 작성할 수 있습니다.
```
let idValid: BehaviorSubject<Bool> = BehaviorSubject(value: false)
let idInputText: BehaviorSubject<String> = BehaviorSubject(value: "")
let pwValid: BehaviorSubject<Bool> = BehaviorSubject(value: false)
let pwInputText: BehaviorSubject<String> = BehaviorSubject(value: "")

private func bindInput() { // Input
   idField.rx.text.orEmpty
      .bind(to: idInputText)
      .disposed(by: disposeBag)

   idInputText
      .map(checkEmailValid(_:))
      .bind(to: idValid)
      .disposed(by: disposeBag)

   pwField.rx.text.orEmpty
      .bind(to: pwInputText)
      .disposed(by: disposeBag)

   pwInputText
      .map(checkPasswordValid(_:))
      .bind(to: pwValid)
      .disposed(by: disposeBag)
}

private func bindOutput() { // Output
   idValid.subscribe(onNext: { bool in
      self.idValidView.isHidden = bool
   }).disposed(by: disposeBag)

pwValid.subscribe(onNext: { bool in
      self.pwValidView.isHidden = bool
   }).disposed(by: disposeBag)

Observable.combineLatest(idValid, pwValid) { id, pw in id && pw }
    .subscribe(onNext: { bool in
      self.loginButton.isEnabled = bool
   }).disposed(by: disposeBag)
}
```
먼저 valid, inputText를 BehaviorSubject 타입으로 만들고 각각 false, "" 라는 기본값을 줍니다.
이후부터는 크게 다를만한게 없지만 <code>bind(to:)</code>를 살펴보면, bind는 값을 관찰하다가 변경되면 <code>bind(to:)</code>에 있는 대상으로 값을 넘겨줍니다.
그리고 <code>bind(to:)</code>의 리턴타입을 보면 <code>Disposable</code>입니다. 그래서 disposeBag에 담아주고요.
이렇게 Input, Output을 명확하게 구분해서 작성 가능하고 훨씬 안정적이고 간결하게 표현 가능합니다.


