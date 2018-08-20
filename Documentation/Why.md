## Why

**Rxは宣言的な方法でアプリケーションを構築できます。**

### Bindings

```swift
Observable.combineLatest(firstName.rx.text, lastName.rx.text) { $0 + " " + $1 }
    .map { "Greetings, \($0)" }
    .bind(to: greetingLabel.rx.text)
```

これは `UITableView`sと` UICollectionView`でも使えます。

```swift
viewModel
    .rows
    .bind(to: resultsTableView.rx.items(cellIdentifier: "WikipediaSearchCell", cellType: WikipediaSearchCell.self)) { (_, viewModel, cell) in
        cell.title = viewModel.title
        cell.url = viewModel.url
    }
    .disposed(by: disposeBag)
```

**正式な提案は、単純バインディングには必要ないのに、常に `.disposed（by：disposeBag） `を使うことです。**

### Retries

APIが失敗しないならば大丈夫ですが、残念ながら失敗する可能性があります。 APIメソッドがあるとしましょう：

```swift
func doSomethingIncredible(forWho: String) throws -> IncredibleThing
```

この関数をそのまま使用している場合は、失敗した場合に再試行するのは本当に難しいです。複雑なモデリング[指数バックオフ]（https://en.wikipedia.org/wiki/Exponential_backoff）はもちろんです、確かに可能ですが、コードにはおそらく気にしない過渡状態がたくさんあり、再利用できません。 

理想的には、再試行の本質を捉え、それをどの操作にも適用できるようにするのが理想的です。 

これは、Rxで単純な再試行を行う方法です

```swift
doSomethingIncredible("me")
    .retry(3)
```

カスタムリトライ演算子も簡単に作成できます。

### Delegates

これはわかりにくい

```swift
public func scrollViewDidScroll(scrollView: UIScrollView) { [weak self] // what scroll view is this bound to?
    self?.leftPositionConstraint.constant = scrollView.contentOffset.x
}
```

... こうかこう

```swift
self.resultsTableView
    .rx.contentOffset
    .map { $0.x }
    .bind(to: self.leftPositionConstraint.rx.constant)
```

### KVO

これと

```
`TickTock` was deallocated while key value observers were still registered with it. Observation info was leaked, and may even become mistakenly attached to some other object.
```

これは

```objc
-(void)observeValueForKeyPath:(NSString *)keyPath
                     ofObject:(id)object
                       change:(NSDictionary *)change
                      context:(void *)context
```

これを使いましょう [`rx.observe` and `rx.observeWeakly`](GettingStarted.md#kvo)

使用例は以下

```swift
view.rx.observe(CGRect.self, "frame")
    .subscribe(onNext: { frame in
        print("Got new frame \(frame)")
    })
    .disposed(by: disposeBag)
```

もしくは

```swift
someSuspiciousViewController
    .rx.observeWeakly(Bool.self, "behavingOk")
    .subscribe(onNext: { behavingOk in
        print("Cats can purr? \(behavingOk)")
    })
    .disposed(by: disposeBag)
```

### Notifications

これは

```swift
@available(iOS 4.0, *)
public func addObserverForName(name: String?, object obj: AnyObject?, queue: NSOperationQueue?, usingBlock block: (NSNotification) -> Void) -> NSObjectProtocol
```

こう書けます

```swift
NotificationCenter.default
    .rx.notification(NSNotification.Name.UITextViewTextDidBeginEditing, object: myTextView)
    .map {  /*do something with data*/ }
    ....
```

### Transient state

また、非同期プログラムを書くときに過渡状態に多くの問題があります。典型的な例は、オートコンプリート検索ボックスです。

Rxを使わないでオートコンプリートコードを書いていたら、おそらく解決しなければならない最初の問題は `abc`の` c`がタイプされ、 `ab`に対する保留中のリクエストがあるときです。 
OK、解決するのは難しいことではありません。保留中のリクエストを参照する追加の変数を作成するだけです。

次の問題は、リクエストが失敗した場合、その不吉なリトライロジックを行う必要があることです。しかし、OK、クリーンアップする必要がある再試行の回数をキャプチャするいくつかのフィールド。

プログラムがしばらく待ってからサーバーにリクエストを発してしまうといいでしょう。結局のところ、誰かが非常に長い間何かを入力している場合に備えて、私たちはサーバーをスパムしたくありません。追加のタイマーフィールドはおそらく？

また、検索が実行されている間に画面上に何を表示する必要があるのか​​、すべての再試行でも失敗した場合に表示する必要があることについての質問があります。

これをすべて書き、適切にテストするのは面倒です。これは、Rxで書かれた同じロジックです。

```swift
searchTextField.rx.text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .subscribe(onNext: { results in
      // bind to ui
    })
    .disposed(by: disposeBag)
```

追加のフラグやフィールドは必要ありません。 Rxはその過渡的な混乱をすべて処理します。

### Compositional disposal

テーブルビューでぼやけた画像を表示するシナリオがあるとしましょう。まず、URLからイメージを取り出し、デコードしてからぼかしてください。

また、ブレンディングのための帯域幅とプロセッサ時間が高価であるため、セルが可視のテーブルビュー領域を出ると、そのプロセス全体がキャンセルされる可能性もあります。

ユーザーが本当に速くスワップすると、起動され、キャンセルされたリクエストが多いため、セルが可視領域に入るとすぐにイメージを取得するだけでは、すぐに開始しないとよいでしょう。

また、画像をぼかすことは高価な操作であるため、同時の画像操作の数を制限することができればいいと思います。

Rxではこうかきます

```swift
// this is a conceptual solution
let imageSubscription = imageURLs
    .throttle(0.2, scheduler: MainScheduler.instance)
    .flatMapLatest { imageURL in
        API.fetchImage(imageURL)
    }
    .observeOn(operationScheduler)
    .map { imageData in
        return decodeAndBlurImage(imageData)
    }
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { blurredImage in
        imageView.image = blurredImage
    })
    .disposed(by: reuseDisposeBag)
```

このコードはすべてのことを行い、 `imageSubscription`が破棄されると、それはすべての従属非同期操作を取り消し、不正な画像がUIにバインドされていないことを確認します。

### Aggregating network requests

ネットワーク要求の集約

両方のリクエストが発生し、両方のリクエストが完了したときに結果を集計する必要がある場合はどうなりますか？

`zip` operatorを使いましょう

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```

では、これらのAPIがバックグラウンドスレッドで結果を返し、メインUIスレッドでバインディングが発生した場合はどうなりますか？ `observeOn`があります。

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.observeOn(MainScheduler.instance)
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```

Rxが本当に輝く、もっと実用的な使用例がたくさんあります。

### State

突然変異を可能にする言語は、グローバルな状態にアクセスしてそれを突然変異させることを容易にする。共有されたグローバル状態の制御されない突然変異は、[組み合わせの爆発]（https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing）を容易に引き起こす可能性がある。

しかし一方で、スマートな方法で使用される場合、命令型言語はより効率的なコードをハードウェアに近づけることができます。

コンビナトリアル爆発に対抗する通常の方法は、状態をできるだけ単純な状態に保ち、[単方向データフロー]（https://developer.apple.com/videos/play/wwdc2014-229）を使用して派生データをモデル化することです。

これはRxが本当に輝く場所です。

Rxは機能的な世界と命令的な世界の間のすばらしい場所です。不変の定義と純粋な関数を使用して、可変状態のスナップショットを信頼性の高い合成可能な方法で処理できます。

実際の例は何ですか？

### Easy integration

あなた自身の観測可能物を作成する必要がある場合はどうなりますか？それはかなり簡単です。このコードはRxCocoaから取得したもので、HTTPリクエストを `URLSession`でラップするために必要なものです

```swift
extension Reactive where Base: URLSession {
    public func response(request: URLRequest) -> Observable<(Data, HTTPURLResponse)> {
        return Observable.create { observer in
            let task = self.base.dataTask(with: request) { (data, response, error) in
            
                guard let response = response, let data = data else {
                    observer.on(.error(error ?? RxCocoaURLError.unknown))
                    return
                }

                guard let httpResponse = response as? HTTPURLResponse else {
                    observer.on(.error(RxCocoaURLError.nonHTTPResponse(response: response)))
                    return
                }

                observer.on(.next(data, httpResponse))
                observer.on(.completed)
            }

            task.resume()

            return Disposables.create(with: task.cancel)
        }
    }
}
```

### Benefits

利点

In short, using Rx will make your code:

* Composable <- Because Rx is composition's nickname
* Reusable <- Because it's composable
* Declarative <- Because definitions are immutable and only data changes
* Understandable and concise <- Raising the level of abstraction and removing transient states
* Stable <- Because Rx code is thoroughly unit tested
* Less stateful <- Because you are modeling applications as unidirectional data flows
* Without leaks <- Because resource management is easy

- 合成可能 <- 構成物
- 再利用可能　<- 構成物だから
- 宣言的 <- データは不変である
- 簡潔に理解できる　<- 抽象化のレベルを上げ、過渡状態を除去する
- 安定、堅牢 <- Rx codeはユニットテストかきやすい
- ステートフルレス <- データフローが単方向性
- リークがない <- リソースマネジメントが簡単

### It's not all or nothing

通常、Rxを使用してできるだけ多くのアプリケーションをモデル化することをお勧めします。

しかし、すべての演算子と、特定のケースをモデル化する演算子が存在するかどうかわからない場合はどうすればよいでしょうか？

Rx演算子はすべて数学に基づいており、直感的でなければなりません。

良いニュースは、約10-15人のオペレータが最も典型的なユースケースをカバーしていることです。そしてそのリストにはすでに `map`、` filter`、 `zip`、` observeOn`などの使い慣れたものがいくつか含まれています。

There is a huge list of [all Rx operators](http://reactivex.io/documentation/operators.html).

For each operator, there is a [marble diagram](http://reactivex.io/documentation/operators/retry.html) that helps to explain how it works.

しかし、そのリストにない演算子が必要な場合はどうすればよいでしょうか？あなた自身のオペレータを作ることができます。

何らかの理由でその種の演算子を作成することが本当に困難な場合、または作業する必要のあるステートフルなコードを残しておくとどうなりますか？さて、あなたは混乱してしまいましたが、あなたは[Rxモナドから飛び出して]簡単にデータを処理して戻すことができます。
