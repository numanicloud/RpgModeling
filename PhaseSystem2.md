# フェーズシステム

このフェーズシステムなるシステムでは、キャンセル可能かつ段階を持つメニューの制御を直感的に扱えることを目指す。

---

重要な項目

- フェーズとは？
- フェーズの概念から見出せるモデルとは？
- フェーズコマンドパターンの紹介
- 上記3つについて更に詳しく
    - フェーズでは無いが近い概念
    - なぜこのモデルが必要か

## キャンセル可能なメニュー

RPGではキャラクターの行動を決めるために何段階かのメニューを操作することになる。例えば：

1. 「たたかう」「アイテム」などの方針の選択
2. 実際に使用するアイテムの選択
3. アイテムを使用する対象の選択

プレイヤーが1,2,3の順で選択を進めていったとき、途中で選択をキャンセルしたい場合がある。
もし3の選択肢にあるときにキャンセル操作をしたならば、2の選択をやり直してほしい。

その実装は単純ではなく、例外的な状況もいくつかある。

まず、方針の選択をしている時は、それより前の段階というのが無いのでキャンセルはできない。
だが、もし現在の選択肢が2人目のキャラクターに対するものであれば、
方針の選択肢であってもキャンセルできるべきで、
その際には1人目のキャラクターがアイテムを使う対象を選択をする段階へ戻って欲しい。

## 直感的な書き方

理想としてはフェーズ管理をあまり意識せずに、
ifなどの制御構文と同じ感覚でフェーズの制御をしたい。

## フェーズ

フェーズとは、キャンセル命令によって巻き戻すことのできるような、ひとまとまりの処理のこと。
フェーズには入力と出力があってよく、フェーズの処理が全て完了すると出力を得られる。
フェーズの出力となる値は、その処理がキャンセルされたかどうかによって異なる。
特に、もし出力の値が`Retry`であれば、そのフェーズをもう一度実行する必要がある。

---

今回の実装では、フェーズはメソッドとして表現される。
フェーズごとにキャンセル処理をサポートするには直前に処理していたフェーズを覚えておく必要があるが、
この方法ならそもそも呼び出しスタックにフェーズが積んであるため、簡単に処理を戻せる。
また、特定のフェーズが次に実行するフェーズを条件で分岐するような流れが把握しやすいのも利点である。

各フェーズがプログラムの他の部分へ返す出力の型などは、プログラムの仕様による。
一方で、キャンセルが起きた時にそのフェーズをやり直す、という制御は実装も薄く小さく、使いまわしがきく。
この制御の部分を、フェーズハンドラーに任せる。

そこで、フェーズが役目を果たし出力を得られた場合には `Finished<T>` のインスタンスを返す。
この型はフェーズに要求した戻り値をコンストラクタを通じて保持しており、後で取り出せる。

一方、フェーズがキャンセルされた場合には `Cancelled<T>` のインスタンスを返す。
キャンセルされたがためにフェーズは役割を果たせておらず、それゆえにこの型はメンバーを持たない。

あるフェーズが別のフェーズを呼び出したとき、子（呼び出される側）フェーズがキャンセルされると、
処理が親（呼び出した側）フェーズに戻される。
この場合、子フェーズがそのハンドラーに `Cancelled` を返す想定だが、
この状況下で子ハンドラーは親フェーズに `NeedsRetry` を返す。
そして親フェーズが `NeedsRetry` をそのまま親ハンドラーへ返すと、
親ハンドラーの責任で親フェーズを最初から実行しなおしてくれる。

---

子ハンドラーが親フェーズに `Cancelled` ではなく `NeedsRetry` を返すのは、
子フェーズから子ハンドラーへの戻り値と、親フェーズから親ハンドラーの戻り値が同じであるのでは、
キャンセルされたのが子フェーズなのか親フェーズなのか区別がつかないからである。

---

フェーズは、ゲームの仕様として期待される処理をする部分（フェーズ制御）と、
その処理を適切に呼び出す処理をする部分に分かれる（フェーズ実装）。

---

フェーズシステムでいうフェーズとは、キャンセルによって巻き戻される処理のかたまりである。
たとえばアイテム選択をするフェーズと対象選択をするフェーズがあったならば、
対象選択をする際にキャンセルしたときにはアイテム選択をやり直す。
このとき、プレイヤーの意図としては、対象選択を放棄したというよりも
アイテム選択の決定を取り消す意図が強いだろう。

各フェーズは「結果」を返す。
例えば、そのフェーズが完了したことで知ることができたプレイヤーの作戦だとか、
そのフェーズが完了したことでバトル全体が特定の結末で終了したとか、
そういったメッセージをメソッドの戻り値として返すことになる。

ただ、フェーズシステムを制御するための戻り値も必要である。
フェーズの終わり方にはいくつかある。

* 「完了」。フェーズの連鎖が終わり、ユーザーの入力を欲しがっている呼び出し元に処理が返る。
* 「キャンセル」。直前のフェーズの結果を取り消すために、まず現在のフェーズを脱出する必要がある。
* 「リトライ」。後続のフェーズがキャンセルされたということは、主体のフェーズの入力し直しが要求されたということである。

これらの結果を表現できるように、 `IPhaseResult` というジェネリック型を用意する。
型引数はユーザー定義の型を使うことを想定していて、
つまりユーザーがフェーズごとにどのような情報を集めたいかは型引数で表現し、
フェーズシステム側がどのような制御をするべきかは `IPhaseResult` の実装が決める。

## 制御の種類

フェーズ制御ではいくつかのシナリオが考えられる。

* そのフェーズが最初の選択であり、キャンセルするものがまだ無い場合
* 選択結果を射影して使いたい場合
* foreachループの各繰り返しをフェーズとして扱いたい場合

## いくらかの前提

フェーズシステムを設計する際に前提にしたことがある。

### キャンセルが意味すること

プレイヤーが選択肢をキャンセルすることというのは、
直前に表明したプレイヤーの意志に基づいてプログラムが進行することを拒否することと同じと見なす。

### フェーズを意識しない場合のコードの形状

フェーズを意識せずにコードを書いた場合のコードを、
簡単にフェーズシステムに修正できるようにしたいところ。

以下のような手順で書き換える：

* キャンセル可能な非同期メソッドの戻り値は `Task<IPhaseResult<T>>` となる
* 選択が確定すると、戻り値は `new Finished<T>()` で包んで返す
* キャンセルされると、戻り値として `new Cancelled<T>()` を返す。従来はnullを返すのが自然だった
* 次のフェーズの呼び出しは `PhaseMethod(args)` ではなく `PhaseHandler.HandlePhase(() => PhaseMethod(args))` といった感じになる
* foreachは `PhaseHandler.ForEachPhase(item => PhaseMethod(item), items)` といった感じになる
* フェーズ階層のルートを呼び出す部分は `PhaseHandler.HandleEntryPhase` を利用する

この他、`while`や`using`などにもフェーズシステムの支援が必要な制御が存在するのかもしれないが、
今はそういった使い道には出会っていないのでとりあえず無視した。

### フェーズの扱い

各フェーズどうしの間に、処理の流れを制御するメソッド呼び出しが挟まる形になる。

以下のような呼び出し階層を考える：

```
Handler1 -> Phase1 -> Handler2 -> Phase2
```

Phase2の入力がキャンセルされると、Phase1でなされた入力は無効となり、
もはやPhase2は、今提示している選択肢の尤もらしさを失うため、入力を続けるわけにはいかない。
そこで関数を抜け、 `Cancelled` という結果をHandler2に知らせる。

Handler2はCancelledという結果を見て、Phase2の処理を行うための根拠が期限切れになったことを知る。
そこで、Phase1をもう一度実行する必要がある。
しかしPhase1を実行する権限を持っているのはHandler1なので、
ここは一旦 `NeedsRetry` という結果をPhase1へ返す。

Phase1はこの結果を操作するべきではないし、読み取るのもやめた方がいい。
Phase1が健全な実装であれば、この結果をそのままHandler1へ返す。

Handler1は、自分以降のフェーズからキャンセルによって戻ってきたことを知る。
`NeedsRetry` は、主体のフェーズを再実行するよう求められていることを表していて、
これは `Cancelled` とは異なる。
`Cancelled` は、主体のフェーズによる処理を正当化していた条件が無効になったことを意味する。

### 目標はモデル化の目安になること

RpgGeneratorのフェーズシステム機能はいつでも役に立つものではないかもしれない。
むしろ重要なのはこのライブラリのAPIというよりも、
このライブラリがアーキテクチャの選択肢を増やすことである。

# 詳細な解説 (2020/12/01 14:54)

- どのようなユースケースで現れる概念であるか？
- どのようなモデルへ抽象化できるか？
- どのように記述されるべきか？

## ユースケース

プログラムの行う作業には順序に意味のある場合がある。
中でもとくに、ユーザーが複数回の入力を行うことで作業内容を決定する処理が今回の話題の対象である。
ユーザーに入力を促す際には何らかの形で選択肢を提示することになるが、
今回のシナリオでは、ユーザーが選択した内容に応じて提示する選択肢を変えなければならないものとする。

こういったものの具体例はコマンドRPGの戦闘システムなどである。
そこでは、入力を受けるまで提示すべき選択肢を決められない仕様が繰り返し現れることがある。
たとえば：

- 「たたかう」コマンドを選ぶまでは、スキル一覧を出せない。なぜなら「にげる」を選ぶかもしれないから。
- スキルを選択するまでは、ターゲット選択に進めない。なぜなら敵一体に使うスキルなのか、味方一体に使うスキルなのかが決まらないから。

こういった入力方式では、入力された値のうちどれかが正当性を失うと、
それ以降に提示された選択肢は全て正当性を失ってしまう。
ここでいう正当性とは、プレイヤーがその決定を望んでいるという事実のことである。
このとき、ユーザーが次に入力すべき値は、正当性のある最後の入力値に基づく次の選択肢となるだろう。

実際にはプレイヤーにキャンセル操作を提供して、最後に入力された値だけを無効化できるようにするだろうから、
重要なのは以下の2つの機能である：

- コマンドを入力すると、それに基づいて絞り込まれた更に詳細なコマンドを検討できる
- キャンセル操作をすることによって、最後に入力したコマンドを入力しなおせる

## モデル

複数の入力を要求するときに、ある入力が他の入力より後に来て然るべきであるとき、
そのような入力処理を**コマンドフェーズ**と呼ぶ。

コマンドフェーズどうしには必ず順序関係があり、この順序は動的に決まる。

各コマンドフェーズは、それより前のコマンドフェーズによる出力を

# アイデアのまとめ方 (2020/12/01 19:14)

まず詳細をすべて書き出す。

分からないところはブラックボックス化して、後の節でどう分からないか紹介する。

## ユースケース

`メインコマンド > スキル選択 > ターゲット選択` のように、
入力する順序が決まっているようなコマンドを実装する機会がよくある。

こういったコマンドは、途中の段階でキャンセル操作をすることで、前の段階に戻って欲しい。
当然、入力結果をプログラムの他の部分へ渡したいし、
スキルの効果範囲など状況に応じて有効なコマンドを切り替えたい。

### つまり

`メインコマンド > スキル選択 > ターゲット選択` のように、
入力する順序が決まっているようなコマンドを実装する機会がよくある。
こういったコマンドは、途中の段階でキャンセル操作をすることで、前の段階に戻って欲しい。

## モデル

複数の入力を要求するときに、ある入力処理が他の入力処理より後に来てほしいとき、
そのような入力処理を**コマンドフェーズ**と呼ぶ。

コマンドフェーズどうしには必ず順序関係があり、この順序は動的に決まる。

コマンドフェーズを通じて得られたプレイヤーからの入力は、コマンドフェーズからの出力となる。
コマンドフェーズを実行するためには、入力が必要な場合がある。
こうした入力は、そのコマンドがどういった情報を基準にUIを構築するかに主に影響する。

コマンドの入力処理が終了すると、その結果は「値が得られた」か「キャンセルされた」となる。

値が得られた場合、それを入力として次のフェーズを実行するか、
あるいはそれを出力としてフェーズの連鎖を呼び出した大元に返すことになる。

キャンセルされた場合、現在のフェーズより１つ前のフェーズに戻って、それを実行しなおす必要がある。

### つまり

複数の入力を要求するときに、ある入力処理が他の入力処理より後に来てほしいとき、
そのような入力処理を**コマンドフェーズ**と呼ぶ。

入力処理を通じて値が得られた場合、それを入力として次のフェーズを実行するか、
あるいはそれを出力としてゲームの進行に利用することになる。
一方でキャンセルされた場合は、
現在のフェーズより１つ前のフェーズに戻って、それを実行しなおす必要がある。

## データ構造とアルゴリズム

キャンセル時に前のフェーズに戻るためには、前のフェーズが何であったか覚えておく必要がある。
実行するのは常に最新のフェーズで、かつキャンセル時は必ず１つ前のフェーズへ戻るので、
スタック構造を用いて考えるのが適している。

フェーズをデータ構造として見ると、以下の属性を持っていると分かる

- 入力
- 出力
- どのような処理を実行するか

これはメソッドと同様の構造である。
処理の順番をスタック構造で表現するアイデアも合わせて考えると、
フェーズを１つのメソッドで表現するのが自然と思われる。

この前提のもとキャンセルの実装を考えると、通常のメソッド呼び出しに不足している構造が見つかる。
通常のメソッド呼び出しは、メソッドがどのような結果に終わったとしても、1度しか呼ばない。
キャンセルしている限りはフェーズを繰り返し実行したいので、ここに介入が必要である。

具体的には、全てのフェーズコマンド呼び出しの直前に、別のメソッド呼び出しを挟む。
このメソッドをフェーズハンドラーといい、フェーズハンドラーがキャンセルの制御を担う。

フェーズが成功またはキャンセルで終わった場合、
その場でそのフェーズを再実行する意図は無いので、単に呼び出し元のフェーズへ返すことになる。
一方、あるフェーズが別のフェーズ（子フェーズ）を呼び出している場合、子フェーズがキャンセルで終わっていれば、
親フェーズの処理を再実行する必要がある。

このとき、子フェーズがハンドラーにキャンセルを送り、親フェーズがそれをハンドラーに送るだけだと、
親フェーズは再実行の必要があるフェーズであると区別することができない。
なので親フェーズは、子フェーズからキャンセルが送られてきたら、
ハンドラーには「再実行の必要がある」ことを示す出力を返す必要がある。

これは定型的な制御なので、実際にはハンドラーが担う。
子フェーズがハンドラーへキャンセルを返すと、親フェーズは再実行をハンドラーへ返す。

### つまり

コマンドフェーズはメソッドによって表現され、コマンドフェーズどうしの順番は、
あるフェーズメソッドの中で次のフェーズメソッドを呼び出すことで表現される。

コマンドフェーズはキャンセル操作をきっかけに最初から実行しなおしになる場合がある。
フェーズメソッドを実行しなおすという制御は、フェーズハンドラーなるクラスで共通化できる。

これらの構造を実現するために、フェーズメソッドの戻り値を3つに分類する必要がある(*1)。

- `Finished`：入力により値が得られたことを表す
- `Cancelled`：キャンセル操作によって中断されたことを表す
- `NeedsRetry`：フェーズを再実行する必要があることを表す

## 実装

フェーズハンドラーは呼び出しスタックに載っているべきなので、それ自体もメソッドである。
フェーズハンドラーの管理は現時点でステートレスだと言えるので、
静的クラス`PhaseHandler`の静的メソッド`Handle`として実装する。

フェーズ呼び出しを繰り返し行いたい都合上、
フェーズハンドラーなるメソッドにはフェーズメソッドを実行するラムダ式を入力するのが単純。
フェーズハンドラーの内部では、以下の処理が必要：

- コマンドフェーズを実行する
- コマンドフェーズの結果を見て、再実行するかを決める
- 戻り値を返すときは出力値を適切に加工して返す

コマンドフェーズの出力となる値は次のような構造である：

- `IPhaseResult<T>`：コマンドフェーズの戻り値になれる値のインターフェース。
- `Finished<T>`：入力が完了したことを表現する。
    - コマンドフェーズにって得られた`T`型の入力値をメンバーに持つ。
- `Cancelled<T>`：入力がキャンセルされたことを表現する。
- `NeedsRetry<T>`：入力をやり直す必要があることを表現する。

そして、フェーズメソッドの戻り値は `Task<IPhaseResult<T>>` である必要がある。

また、以下のようなヘルパーメソッドは役に立つ：

- `Map`: `Finished` な出力から中身を取り出し
    - それ以外の出力は素通しする
- `ContinueWith`: `Finished` な出力に対して次のコマンドフェーズを呼び出し、新たな出力を作る。
    - それ以外の出力は素通しする
- `AsFinished`: 通常の値を `Finished<T>` で包む
- `HandleEntryPhase`: 最も最初のフェーズとしてコマンドフェーズを実行する。
    - このフェーズは `Finished` を返すまで(`Cancelled`でも)再実行され続ける。
    - 戻り値は通常 `Finished` に包まれているが、これを剥がして返してくれる。
- `ForEach`: foreach文の各ループに順序とキャンセル機能を持たせたい場合に便利。
    - プレイヤー3のコマンド選択の最初をキャンセルすると、プレイヤー2のコマンド選択の最後に戻る、といった感じ

### つまり

静的クラス`PhaseHandler`の静的メソッド`Handle`がフェーズ制御をする。
フェーズ制御のためのメソッドに対してラムダ式を入力することで、再実行を可能にする。

コマンドフェーズの出力を加工したり、コマンドフェーズの羅列を扱うためのメソッドも用意する。`Map`, `ContinueWith`, `AsFinished`, `HandleEntryPhase`, `ForEach` などである。

## 懸念点

### (*1)IPhaseResultの安全性

`Finished`, `Cancelled` はユーザーがコマンドフェーズの結果がどうであったかを申告するためのものであるのに対し、`NeedsRetry` はフェーズハンドラーがフェーズの制御に使うためだけのものであり、非対称的である。プログラマーが`NeedsRetry`を自分で生成しないよう注意しなければならないのが難点。

ファクトリーメソッドを利用してこれを解決できるかもしれない。
`IPhaseResult.Finish(value)` とか `IPhaseResult.Cancel()` などを通じて生成できるとよい。
`AsFinished` 拡張メソッドも引き続き使ってよさそう。

これにより、ユーザーの目に`Finished`,`Cancelled`が触れることはなくなる。

### フェーズメソッドの命名が混乱しやすい

`SkillSelection` フェーズの後は `TargetSelection` フェーズであるが、
`SkillSelection`メソッドが`TargetSelection`メソッドを呼び出すコードを書くとなると、
あたかも`TargetSelection`が`SkillSelection`の一部であるかのように見えて奇妙である。

命名でカバーできる話でもある。
「行動コマンド」「スキルの発動コマンド」「ターゲット選択コマンド」のような命名だと、
スキルの発動とはターゲット選択も含んでいて然るべき印象があるかもしれない。

### フェーズ制御のためのデータをフック出来てしまう

`NeedsRetry`がフェーズメソッドに戻ってきた場合、これはフェーズハンドラーにそのまま返すべきだが、
やろうと思えばここでフックして自由な処理を行い、`Finished`状態にでも変更して返すこともできる。
これは本筋から逸れる使い方だし、不具合の原因になる。

`NeedsRetry`の型自体にアクセスできなければ、少なくとも`NeedsRetry`状態を無視されることはなさそう。
`Finished`や`Cancelled`にも同じ対応をすることはできる。
特にファクトリーメソッドを利用すれば、`NeedsRetry`はファクトリーさえ提供しなければ完全に隠蔽できる。

### 呼び出しスタックを使わない方がよい？

IPhaseResultの安全性、フェーズメソッドの命名が混乱しやすい、フェーズ制御のためのデータをフック出来てしまう、
といった問題は、フェーズメソッドとフェーズハンドラーが呼び出しスタックの中に共存していることが原因と思われる。

フェーズメソッド同士が呼び出しツリー上で兄弟関係にあるなら、このような問題は起こらない。
ただしこの場合、あるフェーズメソッドの実行後、次にどのフェーズを実行するのかを決める手段が複雑になりそうだ。

たとえば、フェーズのルートを呼び出すメソッド上で、
コレクションに登録する要領でフェーズハンドラーの設定を構築していくべきだろうか？
この方法だと、あるフェーズの出力を別のフェーズに入力する際に型の保証が難しくなりそうだ。
メソッドチェーンとジェネリクスを組み合わせると、許容範囲内の複雑さで書けるかもしれない。
この設計において`ForEach`などの仕組みがどう表現できるかも試してみないと分からない。

```csharp
// なかなか悪くない
// foreachなどの仕組みはどうする？
// ラムダ式の引数に使う型はタプルにすべき？ユーザーが用意すべき？
// それに、内部はどうなっているだろう？そもそも実装可能か？

var invocation = await PhaseChainBuilder.Create()
    .AddPhase(() => InputMainCommand(battleContext))
    .AddPhase(mainCommand => InputSkill(battleContext, mainCommand))
    .AddPhase((mainCommand, skill) => InputTarget(battleContext, mainCommand, skill))
    .RunAsync();

var invocations = await PhaseChainBuilder.ForEnumerable(battleContext.Players)
    .AddPhase(player => InputMainCommand(battleContext, player))
    .AddPhase((player, mainCommand) => InputSkill(battleContext, player, mainCommand))
    .AddPhase((player, mainCommand, skill) => InputTarget(battleContext, player, mainCommand, skill))
    .RunAsync();

// ラムダ式の引数１つのバージョン
var hoge = await PhaseChainBuilder.ForEnumerable(battleContext.Players)
    .AddPhase(p => InputMainCommand(battleContext, p))
    .AddPhase(p => InputSkill(battleContext, p.Doer, p.MainCommand))
    .AddPhase(p => InputTarget(battleContext, p.Doer, p.MainCommand, p.Skill))
    .RunAsync();

// フェーズメソッドに与える型もまとめるとよさそう
var fuga = await PhaseChainBuilder.ForEnumerable(battleContext.Players)
    .AddPhase(InputMainCommand)
    .AddPhase(InputSkill)
    .AddPhase(InputTarget)
    .RunAsync();
```

```csharp
// フェーズメソッドの実装例
private async Task<IPhaseResult<SkillInvocation>> InputTarget(TargetSelectionContext context)
{
    var input = await View.InputTarget(context.Skill.Targeting);
    return input.AsFinished();
}
```

この方針なら実装方法としては、本編で解説したフェーズシステムのような呼び出し階層を、
フェーズハンドラー側が全て支配するような形で再現するとよいかもしれない。
詳細はうまく言い表せないが、フェーズどうしが担っていた、
お互いの前後関係を管理する責任を吸い出すように考えると良いかもしれない。

上のメソッドチェーンは概念としては以下のように展開されるはずだ。

```csharp
var action = await PhaseHandler.Handle(async () =>
{
    var targetContext = await InputSkill(skillContext);
    var skillInvocation = await PhaseHandler.Handle(async () =>
    {
        return (await InputTarget(targetContext)).AsFinished();
    });
    return skillInvocation.AsFinished();
});
```

そういえば、メインコマンド選択はその選択によって後続のフェーズの内容が変わるのだった……
そういうものにはメソッドチェーンは向かない

## 少し進んだ話題

`IPhaseResult`に対して