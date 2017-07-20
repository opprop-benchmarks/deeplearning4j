---
title: "再帰型ネットワークと長・短期記憶についての初心者ガイド"
layout: ja-default
redirect_from: ja/ja-lstm
---

# 再帰型ネットワークと長・短期記憶についての初心者ガイド

目次

* [フィードフォワードネットワークとは？](#feedforward)
* [再帰型ネットワーク](#recurrent)
* [通時的誤差逆伝播法](#backpropagation)
* [消失、爆発する勾配](#vanishing)
* [長・短期記憶ユニット（LSTM）](#long)
* [様々な時間スケールをキャプチャー](#capturing)
* [コードのサンプル＆コメント](#code)
* [リソース](#resources)

このページでは、ニューラルネットワークについて勉強されている方々のために、再帰型（リカレント）ニューラルネットワークの機能、そして主なバリエーションである長・短期記憶（LSTM）の目的と構造についてご説明します。

再帰型ニューラルネットワークは、人工的ニューラルネットワークの一種で、連続するデータからパターンを認識するよう設計されています。その扱うデータには、テキスト、ゲノム、手書き、話し言葉、さらにセンサーや株市場、政府機関などからの数値の時系列データなどがあります。

再帰型ニューラルネットワークは、最もパワフルなタイプのニューラルネットワークであると言ってもほぼ間違いありません。画像にも対応することできます。画像を入力すると連続したパッチへと分解し、シーケンスとして処理することができます。

再帰型ニューラルネットワークには、ある種の記憶があります。また、記憶は人間の状態の一部でもあります。我々は脳内の記憶に繰り返し類推を行います。<sup>[1](#one)</sup>

## <a name="feedforward">フィードフォワードネットワークとは？</a>

再帰型ネットワークを理解するには、まず最初に[フィードフォワード（順伝播型）ニューラルネットワーク](ja/restrictedboltzmannmachine)の基礎を理解する必要があります。これらのネットワークの名前の由来は、ノードでの一連の数学演算を使った情報の通過方法に関連しています。フィードフォワードは、前方への一方向のみに情報を通過させ（同じノードを再び通過することはない）、再帰型は、情報を繰り返しループ状に通過させます。

フィードフォワードニューラルネットワークの場合、入力されたexampleは、ネットワークに入り、教師付きの学習を行い、出力にラベルが付きます。入力画像に「猫」や「象」などとラベルを付けるように指示する信号を送るパターンを認識し、生のデータをカテゴリーにマッピングします。

![Alt text](../img/feedforward_rumelhart.png)

フィードフォワードネットワークは、入力情報のカテゴリーを推測する際に発生するエラーが最小限になるまで、ラベルの付いた画像を使ってトレーニングを受けます。トレーニングを受けたパラメータ、あるいは重みの一式を使って、これまで処理したことのないデータも分類していきます。トレーニングを受けたフィードフォワードネットワークは、どんなランダムな写真の集合にも対応でき、必ずしも最初に処理した写真が、次に処理する写真の分類に関連するわけではありません。猫の写真を処理したからといって、次に象を認識するわけではないのです。

つまり、このネットワークには、時間の観念がなく、現在の時点で入力された情報のことしか考慮しません。最近起こった過去のことに関しては、記憶がなく、それより前の過去に遡ってトレーニングを受けたときのことしか覚えていません。

## <a name="recurrent">再帰型ネットワーク</a>

一方、再帰型ネットワークは、ある入力情報を現在の時点でのみ対応するexampleとしてだけでなく、過去にも接したことのある情報とみなして対処します。下図は、初期の頃、[Elman氏によって提唱された簡単な再帰型ネットワーク](https://web.stanford.edu/group/pdplab/pdphandbook/handbookch8.html)です。一番下にある*BTSXPE*は、ある時点に入力されたexampleで、*CONTEXT UNIT*は、その前に出力された情報を表しています。

![Alt text](../img/srn_elman.png)

再帰型ネットワークがある時点`t-1`で行った決定は、その次の時点`t`で行う決定に影響します。従って、再帰型ネットワークは、現在と近い過去の2か所から情報を受け取り、それらを組み合わせて新しい情報をどう処理するかを決めます。ちょうど人間がするのと同じような対処方法です。

再帰型ネットワークは、このような帰還ループを使うという特徴から、フィードフォワードと異なります。再帰型ネットワークには記憶があるとよく言われます。<sup>[2](#two)</sup>なぜこのようにニューラルネットワークに記憶能力を追加させた再帰型ネットワークがあるかというと、情報が連続体であるものがあり、記憶能力を使うとフィードフォワードネットワークだと不可能なタスクを行うことができるためです。

連続した情報は、再帰型ネットワークの隠れ状態に保管されており、多くのタイムステップを超えて、新しいexampleを処理するのに使用されます。

目には見えなくとも人間には、体中に記憶があり、姿が見えなくとも我々の行動に影響を与えているように、情報は再帰ネットワーク内を隠れ状態で循環します。このように、記憶が巡り巡って何らかのフィードバックを送ってくる状態を表す言葉が英語にはたくさんあります。例えば、過去の自分の行動に苦しまされている人に対して、過去の行いの結果が災いしてバチが当たっていると言います。フランス語で、これを「*Le passé qui ne passe pas*（過ぎ去らない過去）」と言います。

では、次に記憶がこのように前方に送られる過程を数学的にご説明しましょう。

![Alt text](../img/recurrent_equation.png)

タイムステップtでの隠れ状態は`h_t`です。この関数は、同じタイムステップでの入力情報`x_t`に重み行列`W`（フィードフォワードで使用したもののような）を掛け合わせて修正した値に、その前のタイムステップの隠れ状態`h_t-1`と自身の隠れ状態から隠れ状態への遷移の行列`U`を掛けた値を追加したものです。この行列`U`は、遷移行列とも呼ばれ、マルコフ連鎖に似たものです。重み行列とは、現在の入力と過去の隠れ状態の両方に、どのくらいの程度の重要性を割り当てるかを決めるフィルターです。重み行列が生み出したエラーは、誤差逆伝播法によって返され、それ以上エラーが最低限度になるまで重みを調整するのに使用されます。

重みの入力と隠れた状態の総和は、ロジスティックシグモイド関数、あるいは双曲線正接関数（tanh）である`φ`により0と1の値域の空間に押し込まれます。この関数は、一般に非常に大きい、または非常に小さい値をロジスティック空間に入れ込むのに使用されます。また、誤差逆伝播法に使えるように[勾配](../glossary.html#gradient)を設けることができます。

連続する各タイムステップで帰還ループが発生するので、各隠れ状態には、1つ前の隠れ状態だけでなく、記憶が持続する限り、`h_t-1`以前のすべての隠れ状態の痕跡が含まれています。

連続する文字が提供されたら、再帰型ネットワークは、最初の文字を頼りに、次の文字の認識を行うことでしょう。例えば、最初の文字が`q`であると、次の文字は`u`であると推測したり、最初の文字が`t`であると、次の文字は`h`であると推測するのです。

再帰型ネットワークは時間を超えて作業するため、こちらのようなアニメーションによる説明が最適です。（最初の縦線のノードは、時が経つと共に展開されると循環するフィードフォワードネットワークであると考えてください。）

{::nomarkdown}
<blockquote class="imgur-embed-pub" lang="en" data-id="kpZBDfV"><a href="//imgur.com/kpZBDfV">how recurrent neural networks work #deeplearning4j #dl4j</a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>
{:/nomarkdown}

[上図](https://i.imgur.com/kpZBDfV)にある`x`は入力のexample、`w`は入力をフィルターする重み、`a`は隠れ層の活性化（重みを付けた入力とその前の隠れ状態の組み合わせ）、`b`はReLUやシグモイド関数を使って変形、または押し込まれた後の隠れ層の出力を表しています。

## <a name="backpropagation">通時的誤差逆伝播法（BPTT）</a>

再帰型ネットワークの目的は、連続する入力情報を正確に分類することです。我々は、誤差逆伝播法と勾配降下を使ってこれを可能にします。

フィードフォワードネットワークの誤差逆伝播法は、最後のエラーから順に出力、重み、それぞれの隠れ層の入力へ、というように後方から前方に進み、部分的な微分係数∂E/∂w、またはその変化の割合を計算し、これらの重みにエラーの分の修正を割り当てます。これらの微分係数は、重みを調整してエラーを減少させるために、学習ルールや勾配降下に使用されます。

再帰型ネットワークは、[通時的誤差逆伝播法（backpropagation through time）](https://www.cs.cmu.edu/~bhiksha/courses/deeplearning/Fall.2015/pdfs/Werbos.backprop.pdf)と呼ばれる誤差逆伝播法またはBPTTの拡張機能の助けを得ています。この場合、時間は、明確に定義され、順序付けられた一連のタイムステップ同士を繋げる計算で表されます。誤差逆伝播法の作業は、この処理を行うことです。

ニューラルネットワークは、それが再帰型であろうとなかろうと、`f(g(h(x)))`のような単なる入れ子になった合成関数に過ぎません。時間の要素を追加しても、一連の関数が拡張するだけです。これにあたって、連鎖の規則を使った微分係数の計算を行います。

### 打ち切り型通時的逆伝播

[打ち切り型通時的逆伝播（Truncated BPTT）](http://www.cs.utoronto.ca/~ilya/pubs/ilya_sutskever_phd_thesis.pdf)は、長い連続データに適しているフルの通時的誤差逆伝播法（BPTT）と近似していますが、フルの通時的誤差逆伝播法の場合、タイムステップが多くなると、パラメータの更新の際に前方/後方コストが非常に高くなるため、それを回避するために打ち切り型が使用されています。しかし、打ち切り型の欠点は、打ち切られているため、勾配が逆戻りするだけであり、ネットワークが依存関係を学習できないということです。

## <a name="vanishing">消失（爆発）する勾配</a>

多くのニューラルネットワークと同様、再帰型ネットワークの歴史は古く、1990年の初期には、*消失する勾配*が再帰ネットワークの主な障害として知られていました。

直線が、Xの値の変化に伴うYの値の変化の関連を表したものであるように、*勾配*はエラーの変化に関連したすべての重みの変化を表したものなのです。勾配が分からなければ、エラーを減少させるための重み調整ができなくなり、ネットワークは学習を止めてしまいます。

再帰型ネットワークでは、最後の出力と、かなり前のタイムステップのイベントとの関連性を確立することが困難でした。離れた入力にどの程度の重要性を割り当てればいいのかが非常に難しいからです。（ひい-ひい-*-おじいさんなどのように、数字が何度も掛け合わせられる。）

この困難さは、ニューラルネットワークを流れる情報は、数多くの段階を踏む掛け算を行うからということも一部の理由です。

複利計算を勉強したことのある人は誰でも、どんな量であれ1より少し大きな数値と何度も掛け合わせると測定不可能なほどの大きな値になることを知っています。（実際、単純な数学上の真実により、ネットワークにによる影響と必然的な社会的不平等が実証されています。）しかし、逆に、1未満の数値で掛けても同じ結果なのです。ギャンブラーが、1ドルにつきの勝ち取り額が97セントだと、破産はすぐ目の前に迫っているのです。

ディープ・ニューラル・ネットワークの層とタイムステップは、掛け合わす関係にあるため、微分係数が消失したり爆発したりしやすくなります。

勾配が爆発するときの、それぞれの重みの様子は、まるで蝶が羽をはためかせたときに遠方にまで風が揺れる状態に似ています。重みの勾配が上限まで達し、強力すぎる状態になります。しかし、勾配が爆発してしまった場合は、比較的、対処が簡単です。打ち切ったり、押し込んだりすればいいからです。勾配が減衰してしまった場合に、コンピューターが処理する、またネットワークが学習する対象としてはあまりにも小さくなってしまう、というのがもっと対処が困難となる問題なのです。

以下は、シグモイド関数が何度も適用された場合、どのようになるのかを示したグラフです。データは、傾斜が確認できなくなるほど平らになります。これは、層を数多く通過するとこのように勾配が消失するという状態を表しています。

![Alt text](../img/sigmoid_vanishing_gradient.png)

## <a name="long">長・短期記憶ユニット（LSTM）</a>

90年代の半ば、再帰型ネットワークのバリエーションである長・短期記憶ユニット、またはLSTMが、勾配の消失の問題を解決する方法としてドイツ人学者Sepp Hochreiter氏とJuergen Schmidhuber氏により提唱されました。

長・短期記憶ユニットは、時間と層を通して誤差逆伝搬できるエラーを保存します。より一定したエラーを維持することにより、再帰ネットワークが多くのタイムステップ（1000を超える）で学習し続けられるようにし、遠く離れた原因と結果を繋げるチャンネルを開設します。

長・短期記憶ユニットは、再帰型ネットワークの通常の流れの外部にある情報をゲート付きセルに含んでいます。コンピューターのメモリーにあるデータのように、情報をセルに保管、また読み書きすることができます。セルは開閉するゲートを使って、何を保管し、いつ読み込み、書き出し、消去するかを決めます。コンピューターのデジタルストレージとは異なり、これらのゲートはアナログで、0から1の間の値域で表されるシグモイド関数での要素の掛け合わせで実装されています。アナログはデジタルと比べて微分可能であるという点で有利です。このため、誤差逆伝播法に適しています。

これらのゲートは受け取る信号に従い、ニューラルネットワークのノードのように、自身の持つ重みの一式を使ってフィルタリングする情報をその強度や重要性に基づいてブロックしたり伝播したりします。それらの重みは、他の入力や隠れ状態を調節する重みのように、再帰型ネットワークの学習過程で調整されます。つまり、セルは、推測、エラーの逆伝播、勾配降下による重みの調整のサイクル過程を何度も繰り返すことにより、いつデータの入力、出力、削除を許可するかを学習するのです。

下図は、どのようにしてデータがメモリセルを流れ、ゲートの制御を受けているかを表したものです。

![Alt text](../img/gers_lstm.png)

この図の中には、移動する部分がたくさんあります。長・短期記憶についてまだよく知らない方は、慌てずに、よく観察してください。数分ほどすると様々なことが分かってきます。

まず一番下の部分から始めましょう。3つの矢印は、情報が流れる複数の箇所を示しています。現在の入力と過去のセルの状態の組み合わせは、セル自体に送られるだけでなく、どのように入力を処理するかを決める3つのゲートそれぞれにも送られます。

黒点はゲートを表したもので、新しい入力を受け入れるか、現在のセルの状態を消すか、そして/また、現在のタイムステップでの状態をネットワークの出力に影響させるかをそれぞれ決めます。`S_c`は、メモリセルの現在の状態、`g_y_in`はそのセルへの現在の入力です。各ゲートは、開閉し、それぞれの段階で開閉状態を組み合わせ直します。セルは各タイムステップで、その状態を忘れるか否か、書き込むか否か、読み込むか否かを決めることができます。それらのフローは図をご覧ください。

大きな太字は、各処理の結果を表したものです。

次にご覧いただきたい図は、簡単な再帰型ネットワーク（左）と長・短期記憶ユニットのセル（右）を比較したものです。青線はここでは無視してください。右側にあるボックス内の記号一覧表もご確認ください。

![Alt text](../img/greff_lstm_diagram.png)

長・短期記憶ユニットのメモリセルは、入力を変換させる際に、加算と乗算に別々の役割を割り当てます。両方の図の中央の**プラス記号**が長・短期記憶の本質的な部分なのです。馬鹿々々しいほどシンプルに見えるかもしれませんが、この簡単な変化によって、深部まで誤差逆伝搬する必要があるとき、繰り返すエラーが保管されます。現在の状態と新しい入力を掛けて次のセルの状態を決定するのではなく、それら2つを足します。これにより違いがもたらされます。（もちろん、忘却ゲートはそれでも乗算を頼りにします。）

異なる重みの一式は、入力、出力、忘却用に入力をフィルタリングします。忘却ゲートは、線的識別機能として表されます。というのは、ゲートが開いていると、メモリセルの現在の状態は、タイムステップを一つ進めるために1を掛けるだけだからです。

また、それぞれの長・短期記憶セルの忘却ゲートに[バイアスの1を含める](http://jmlr.org/proceedings/papers/v37/jozefowicz15.pdf)ことにより、[性能が高まる](http://www.felixgers.de/papers/phd.pdf)ことが知られています。（*Sutskeverは、バイアスの5をすすめています。*)  

長・短期記憶の目的は、離れた距離間の発生を最後の出力で繋げることなのに、なぜ忘却ゲートがあるのでしょうか。これは、忘れることが必要なこともあるからです。あるドキュメントのテキストコーパスを最後まで分析し終えたとします。そして、次のドキュメントはそのドキュメントに関連している言える理由が皆無であることがあります。このため、ネットワークは次のドキュメントの処理を開始する前にメモリセルをゼロに設定するのです。

下の図はゲートの作動状態を図にしたものです。上にマイナス記号が表示されたゲートは閉じられたものです。上に小さな白丸が表示されたゲートは開いたゲートです。隠れ層にあるマイナス記号や小さな白丸で表示されたゲート忘却ゲートです。

![Alt text](../img/gates_lstm.png)

フィードフォワードネットワークの場合はある1つの入力をある1つの出力にマッピングしますが、再帰型ネットワークの場合は、上記のように1つの入力を数多くの出力に（画像一つをの数多くの単語を含む説明文に）、数多くの入力が数多くの出力に（翻訳）、数多くの入力を1つの出力（音声を分類）にマッピングすることができます。

## <a name="time">様々な時間スケールや遠く離れた依存関係をキャプチャー</a>

新データの記憶セルへの入力を防ぐ入力ゲート、再帰型ニューラルネットワークの出力に対するメモリセルの影響を防止する出力ゲートに何の価値があるのでしょうか。これは、長・短期記憶があることにより、ニューラルネットワークは異なる時間スケールで同時に作業を行うことができるからです。

人間の場合について考えてみましょう。我々は、様々なデータの流れを時系列的に受け取っています。各タイムステップにおける地理位置情報が、次のタイムステップには非常に重要になります。したがって、時間のスケールは常に最新情報を受け付けています。

世の中には、数年おきに行われる選挙の投票に毎回出かける真面目な人もいます。民主主義の時代において、このような人々の選挙に関連した行動のみに絞って情報を集めたいこともあります。このとき、投票後の彼らの帰宅後の生活や彼らの抱えている大きな問題とは切り離した形で情報をキャプチャーすることが必要なのです。地理情報からの一定したノイズを政治的分析に入り込ませないようにしなければなりません。

また、例えば親との付き合いに律儀な女性がいるとすると、その女性の家族との時間を構築し、毎週日曜や祝日に著しく多く発生する通話のパターンを学習させることも可能です。この時間は、政治のサイクルとも地理位置情報ともほとんど関係ないのです。

これらのような類いのデータは他にもあります。例えば、音楽は、複数のリズムを同時に演奏するものです。テキストには、様々な間隔を置いて繰り返されるテーマがあります。株式市場や経済は、広い間隔の波形で変動します。異なる時間スケールで同時に作動するため、長・短期記憶がキャプチャーできるのです。

### ゲート付き再帰型ユニット（GRU）

ゲート付き再帰型ユニット（GRU）は、出力ゲートなしの長・短期記憶ユニットです。したがって、各タイムステップで、そのメモリセルから大型のネットワークにコンテンツを書き込みます。

![Alt text](../img/lstm_gru.png)

## <a name="code">コードのサンプル</a>

シェークスピアのドラマの復元を学習し、Deeplearning4jに実装したコメント付きサンプルのGraves長・短期記憶を、[こちら](https://github.com/deeplearning4j/dl4j-examples/blob/master/src/main/java/org/deeplearning4j/examples/recurrent/character/GravesLSTMCharModellingExample.java)でご覧いただけます。説明が必要と思われるAPIには、コメントが付けてあります。質問がある方は、是非[Gitter](https://gitter.im/deeplearning4j/deeplearning4j)に参加してください。

## <a name="tuning">LSTMハイパーパラメーターの調整</a>

再帰型ニューラルネットワーク用にハイパーパラメーターの最適化を手作業で行うときに覚えておくと良いことは以下の通りです。

* ニューラルネットワークが本質的にトレーニングのデータを「記憶する」ときに起こる*過剰適合（overfitting）*に注意してください。過剰適合とは、データのトレーニングに成功したが、サンプル外での予測には役に立たないという意味です。
* 正規化（Regularization）が役に立ちます。正規化の方法には、L1、L2、ドロップアウトなどがあります。
* ネットワークがトレーニングしないテストを別に準備しましょう。
* ネットワークが大きければ大きいほど、よりパワフルになります。しかし、過剰適合が発生することの方が容易です。10,000の例から百万のパラメーターを学習したくはありません。 -- `parameters > examples = trouble`.
* 常にといっていいほど、データは多ければ多いほどいいでしょう。過剰適合を回避するのに役立つからです。
* 複数のエポックで、トレーニングしましょう。（データセットを完全に通過）
* いつ止めるかを知るために、各エポックで、テストセットの性能を評価します。（早めにストップする）
* 学習係数は、最も重要な唯一のハイパーパラメータです。これを[deeplearning4j-ui](https://github.com/deeplearning4j/deeplearning4j/tree/master/deeplearning4j-ui)を使って調整しましょう。[こちらのグラフ](http://cs231n.github.io/neural-networks-3/#baby)をご覧ください。
* 一般的には、層を積み重ねるといいでしょう。
* 長・短期記憶については、tanh関数でなく、softsign（ソフトマックスではありません）活性化機能を使用するのがいいでしょう。（そちらの方が速く、飽和状態になる可能性が低下します。(~0 gradients))
* アップデート: RMSProp、AdaGrad、モメンタム（Nesterovs）がおすすめです。AdaGradも学習係数を減衰させますが、これも役立ちます。
* データ正常化、MSE損失関数、 + 回帰用のアイデンティティー活性化機能、[Xavierの重み初期化](../glossary.html#xavier)をお忘れなく。

## <a name="resources">リソース</a>
* [DRAW: A Recurrent Neural Network For Image Generation(DRAW：画像世代の再帰型ニューラルネットワーク)](http://arxiv.org/pdf/1502.04623v2.pdf); (attention models)
* [Gated Feedback Recurrent Neural Networks（ゲート付き再帰型ニューラルネットワーク）](http://arxiv.org/pdf/1502.02367v4.pdf)
* [Recurrent Neural Networks（再帰型ニューラルネットワーク）](http://people.idsia.ch/~juergen/rnn.html); Juergen Schmidhuber
* [Modeling Sequences With RNNs and LSTMs（再帰型ニューラルネットワークと長・短期記憶によるシーケンスモデリング）](https://class.coursera.org/neuralnets-2012-001/lecture/77); Geoff Hinton
* [The Unreasonable Effectiveness of Recurrent Neural Networks（想像以上に効果のある再帰型ニューラルネットワーク）](https://karpathy.github.io/2015/05/21/rnn-effectiveness/); Andrej Karpathy
* [Understanding LSTMs（長・短期記憶の説明ガイド）](https://colah.github.io/posts/2015-08-Understanding-LSTMs/); Christopher Olah
* [Backpropagation Through Time: What It Does and How to Do It（通時的誤差逆伝播法：何ができるのか、そしてその仕方について）](https://www.cs.cmu.edu/~bhiksha/courses/deeplearning/Fall.2015/pdfs/Werbos.backprop.pdf); Paul Werbos
* [Empirical Evaluation of Gated Recurrent Neural Networks on Sequence Modeling（シーケンスモデリングのゲート付き再帰型ニューラルネットワークの経験的評価）](http://arxiv.org/pdf/1412.3555v1.pdf); Cho et al
* [Training Recurrent Neural Networks（再帰型ニューラルネットワークのトレーニング）](https://www.cs.utoronto.ca/~ilya/pubs/ilya_sutskever_phd_thesis.pdf); Ilya Sutskever's Dissertation
* [Supervised Sequence Labelling with Recurrent Neural Networks（再帰型ネットワークによる監視下でのシーケンスラベリング）](http://www.cs.toronto.edu/~graves/phd.pdf); Alex Graves
* [Long Short-Term Memory in Recurrent Neural Networks（再帰型ネットワークの長・短期記憶）](http://www.felixgers.de/papers/phd.pdf); Felix Gers
* [LSTM: A Search Space Oddyssey（長・短期記憶：検索スペースオデュッセイア）](http://arxiv.org/pdf/1503.04069.pdf); Klaus Greff et al

## <a name="beginner">その他の初心者用ガイド</a>
* [Restricted Boltzmann Machines（制限付きボルツマン・マシン）](restrictedboltzmannmachine)
* [Eigenvectors, Covariance, PCA and Entropy（固有値、共分散、PCA、エントロピー）](eigenvector)
* [Word2vec](ja-word2vec)
* [Neural Networks（ニューラルネットワーク）](ja-neuralnet-overview)
* [Neural Networks and Regression（ニューラルネットワークと回帰）](linear-regression)
* [Convolutional Networks（畳み込みネットワーク）](convolutionalnets)

### 脚注

<a name="one">1)</a> *再帰型ネットワークは、一般的な人口知能とは全く異なるものに見えるかもしれませんが、知能というのものは、実際は我々が思っていたほど賢明ではないかもしれないと考えています。というのも、記憶の役割をする簡単な帰還ループがあると、これで既に意識の基本的構成要素の1つがあるということになるからです。とはいえ、これは必要であっても十分ではありません。その他に必要なものは、上記では触れていませんが、ネットワークとその状態を表す変数、データの解釈に基づく意思決定のロジックの枠組みなどがあります。後者は、理想的には、成功には報酬を与え、失敗には罰を与える大きめの問題解決ループがいいでしょう。これは強化学習に非常に似ています。そう考えていくと、[DeepMindは既にそれを構築している](https://www.cs.toronto.edu/~vmnih/docs/dqn.pdf)と言えるのです。*

<a name="two">2)</a> *ある意味で、パラメータが最適化したすべてのニューラルネットワークには記憶があるといえます。というのは、これらのパラメータは過去データのトレースだからです。しかし、フィードフォワードニューラルネットワークの場合、この記憶の時間は止まってしまっているといえます。つまり、ネットワークがトレーニングを受けた後、その学習するモデルが調整されることなく、さらに多くのデータに適用されます。また、すべての入力データに同じ記憶（または重みの一式）が適用されるという点でモノリシックです。一方、ダイナミック（「変化する」という意味）ニューラルネットワークという別名を持つ再帰型ネットワークは、記憶があるということよりも、連続するイベントに特定の重みを与えるという点でフィードフォワードニューラルネットワークとは異なります。これらの出来事は、直前、直後に繋がっている必要はありませんが、どれだけ離れていても、同じ時間のスレッドでつながっていると推定されています。フィードフォワードニューラルネットワークはそのような推定をせず、世界を順序や時間と関連しないバケツに詰めた複数の物体であるかのように扱います。この2種類のニューラルネットワークを2種類の人間の知識にマッピングすると役に立つかもしれません。我々は、子供の時に色の認識を学習しますが、その後の人生で、場所や時間に関係なく様々なコンテキストで色を正しく認識することができます。一度、学習しさえすればいいのです。この知識がフィードフォワードネットワークでは、記憶のようなものなのです。スコープや定義のない過去を頼りに認識します。5分前にどの色が入力されたかを聞いても、覚えていないし、気にもしていません。短期間における記憶を喪失してしまうようなものです。一方、我々は子供の時、音の流れから成る言語というものも学習します。そして、「toe」、「roe」、「z」などの音声から引き出す意味は、常に、その前の（そしてその次の）音との関連で理解されます。シーケンスの各段階は、前に何が起こったかに基づいており、その配列によって意味が見えてきます。実際、文はすべてその中にある各音節の意味を伝達しようとし、冗長信号は、環境雑音への対策としての役目を果たします。これは、再帰型ネットワークの記憶が、過去の一部を頼りにするというところと似ています。このように、この2種類のニューラルネットワークは、どちらも過去、あるいは異なる種類の過去を使って、異なる方法で情報を処理するのです。*