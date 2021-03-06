# はじめに

> 効率とは賢く怠けることである (作者不詳)

> 無精：エネルギーの総支出を減らすために，多大な努力をするように，あなたをかりたてる性質。(Larry Wall)

優れたプログラマが持つハッカー気質のひとつに 「無精」 があります。大好きなコンピュータの前から一時も離れずにどうやってジャンクフードを手に入れるか ―― 普通の人からするとただの横着に見えるかもしれませんが，ハッカー達にとってそれはいつでも大きな問題でした。たとえば，ハッカーの巣窟として有名な MIT の AI ラボにはかつて，UNIX のコマンド一発でピザを FAX 注文する `xpizza` コマンドが存在しました (「xpizza MIT」 でググると，[1991 年当時の man ページ](http://bit.ly/mYAJwZ)が読めます)。また，RFC 2325 として公開されているコーヒーポットプロトコルでは，遠隔地にあるコーヒーポットのコーヒーの量を監視したり，コーヒーを自動的に淹れたりするための半分冗談のインターフェースを定義しています。

こうした 「ソフトウェアで楽をする」 ハックのうち，もっとも大規模な例が最新鋭の巨大データセンターです。クラウドサービスの裏で動く巨大データセンターは極めて少人数の管理者によって運用されており，大部分の管理はソフトウェアによって極限まで自動化されているという記事を読んだことがある人も多いでしょう。ピザやコーヒーのようなお遊びから，巨大データセンターのように一筋縄ではいかない相手まで，プログラムで「モノ」を思いどおりにコントロールするのはもっとも楽しいハックの一種です。


# OpenFlowの登場

その中でもネットワークをハックする技術の 1 つが，本連載で取り上げる OpenFlow です。OpenFlow はネットワークスイッチの内部動作を変更するプロトコルを定義しており (OpenFlow の仕様書や標準化に関する情報は，[こちら](http://www.openflow.org/)で得られます。)，スイッチをコントロールするソフトウェア （OpenFlow の世界ではコントローラと呼ばれます） によってネットワーク全体をプログラム制御できる世界を目指しています（図1）。

![OpenFlow スイッチとコントローラ](https://github.com/trema/Programming-Trema/raw/master/images/1_001.png)

図1　OpenFlow スイッチとコントローラ

OpenFlow の登場によって，今までは専門のオペレータによって管理されていたネットワークがついにプログラマ達にも開放されました。ネットワークをソフトウェアとして記述することにより，たとえば 「アプリに合わせて勝手に最適化するネットワーク」 や 「障害が起こっても自己修復するネットワーク」 といった究極の自動化も夢ではなくなります!

本連載では，この OpenFlow プロトコルを使ってネットワークを 「ハック」 する方法を数回に渡って紹介します。職場や自宅のような中小規模ネットワークでもすぐに試せる実用的なコードを通じて，「OpenFlow って具体的に何に使えるの?」 というよくある疑問に答えていきます。OpenFlow やネットワークの基礎から説明しますので，ネットワークの専門家はもちろん，普通のプログラマもすんなり理解できると思います。

まずは，OpenFlow プログラミングのためのフレームワーク 「Trema (トレマ)」 を紹介しましょう。


# OpenFlow プログラミングフレームワーク Trema

Trema は，OpenFlow コントローラを開発するための Ruby および C 用のプログラミングフレームワークです。ノート PC 1 台でアジャイルに OpenFlow 開発をしたいなら，「OpenFlow 界の Rails」こと Trema で決まりです。GitHub 上で開発されており，GPLv2 ライセンスのフリーソフトウェアとして公開されています。公開は今年の 4 月と非常に新しいソフトウェアですが，その使いやすさから国内外の大学や企業および研究機関などですでに採用されています。

Trema の情報は次のサイトから入手できます。

* [Trema ホームページ](http://trema.github.com/trema/)
* [GitHub のページ](https://github.com/trema/)
* [メーリングリスト](https://groups.google.com/group/trema-dev)
* Twitterアカウント： [@trema_news](http://twitter.com/trema_news)

Trema を使うと，ノートPC 1 台で OpenFlow コントローラの開発とテストができます。本連載では，実際に Trema を使っていろいろと実験しながら OpenFlow コントローラを作っていきます。それでは早速 Trema をセットアップして，簡単なプログラムを書いてみましょう。


## セットアップ

Trema は Linux 上で動作し，Ubuntu 10.04 以降および Debian GNU/Linux 6.0 の 32 ビットおよび 64 ビット版での動作が保証されています。テストはされていませんが，その他の Linux ディストリビューションでも基本的には動作するはずです。本連載では，Ubuntu の最新バージョンである 11.04 (デスクトップエディション 32 ビット版)を使います。

`trema` コマンドの実行には `root` 権限が必要です。まずは，`sudo` を使って `root` 権限でコマンドを実行できるかどうか，`sudo` の設定ファイルを確認してください。

    % sudo visudo

`sudo` ができることを確認したら，Trema が必要とする `gcc` などの外部ソフトウェアを次のようにインストールします。

    % sudo apt-get install git gcc make ruby ruby-dev irb libpcap-dev libsqlite3-dev

次に Trema 本体をダウンロードします。Trema は GitHub 上で公開されており，`git` を使って最新版が取得できます。

    % git clone git://github.com/trema/trema.git

Trema のセットアップには，「`make install`」のようなシステム全体へインストールする手順は不要です。ビルドするだけで使い始めることができます。ビルドは次のコマンドを実行するだけです。

    % ./trema/build.rb

それでは早速，入門の定番 Hello, Trema! コントローラを Ruby で書いてみましょう。なお，本連載ではおもに Trema の Ruby ライブラリを使ったプログラミングを取り上げます。C ライブラリを使ったプログラミングの例については，Trema の `src/examples/` ディレクトリ以下を参照してください。本連載で使った Ruby コードに加えて，同じ内容の C コードを見つけることができます。

## Hello, Trema!

`trema` ディレクトリの中に `hello_trema.rb` というファイルを作成し，エディタでリスト 1 のコードを入力してください。

```ruby
class HelloController < Controller # (1)
  def start # (2)
    puts "Hello, Trema!"
  end
end
```

リスト1　Hello Trema! コントローラ

それでは早速実行してみましょう! 作成したコントローラは `trema run` コマンドで実行できます。この世界一短い OpenFlow コントローラ (?) は画面に 「Hello, Trema!」 と出力します。

    % cd trema
    % ./trema run ./hello_trema.rb
    Hello, Trema!  <- Ctrl + c で終了

いかがでしょうか? Trema を使うと，とても簡単にコントローラを書いて実行できることがわかると思います。えっ？ これがいったいスイッチの何を制御したかって？ 確かにこのコントローラはほとんど何もしてくれませんが，Trema でコントローラを書くのに必要な知識がひととおり含まれています。スイッチをつなげるのはちょっと辛抱して，まずはソースコードを見ていきましょう。

### コントローラクラスを定義する

Ruby で書く場合，すべてのコントローラは `Controller` クラスを継承して定義します (リスト1-1)。

`Controller` クラスを継承することで，コントローラに必要な基本機能が `HelloController` クラスにこっそりと追加されます。

### ハンドラを定義する

Trema はイベントドリブンなプログラミングモデルを採用しています。つまり，OpenFlow メッセージの到着など各種イベントに対応するハンドラを定義しておくと，イベントの発生時に対応するハンドラが呼び出されます。たとえば `start` メソッドを定義しておくと，コントローラの起動時にこれが自動的に呼ばれます (リスト1-2)。

さて，これで Trema の基本はおしまいです。次は，いよいよ実用的な OpenFlow コントローラを書いて実際にスイッチをつないでみます。今回のお題はスイッチのモニタリングツールです。「今，ネットワーク中にどのスイッチが動いているか」 をリアルタイムに表示しますので，何らかの障害で落ちてしまったスイッチを発見するのに便利です。

## スイッチモニタリングツールの概要

スイッチモニタリングツールは図 2 のように動作します。

![スイッチモニタリングツールの動作](https://github.com/trema/Programming-Trema/raw/master/images/1_002.png)

図2　スイッチモニタリングツールの動作

OpenFlow スイッチは，起動すると OpenFlow コントローラへ接続しに行きます。Trema では，スイッチとの接続が確立すると，コントローラの switch_ready ハンドラが呼ばれます。コントローラはスイッチ一覧リストを更新し，新しく起動したスイッチをリストに追加します。逆にスイッチが何らかの原因で接続を切った場合，コントローラの `switch_disconnected` ハンドラが呼ばれます。コントローラはリストを更新し，いなくなったスイッチをリストから削除します。

### 仮想ネットワーク

それでは早速，スイッチの起動を検知するコードを書いてみましょう。なんと，Trema を使えば OpenFlow スイッチを持っていなくてもこうしたコードを実行してテストできます。いったいどういうことでしょうか?

その答えは，Trema の強力な機能の 1 つ，仮想ネットワーク構築機能にあります。これは仮想 OpenFlow スイッチや仮想ホストを接続した仮想ネットワークを作る機能です。この仮想ネットワークとコントローラを接続することによって，物理的な OpenFlow スイッチやホストを準備しなくとも，開発マシン 1 台で OpenFlow コントローラと動作環境を一度に用意して開発できます。もちろん，開発したコントローラは実際の物理的な OpenFlow スイッチやホストで構成されたネットワークでもそのまま動作します!

それでは仮想スイッチを起動してみましょう。

### 仮想 OpenFlow スイッチを起動する

仮想スイッチを起動するには，仮想ネットワークの構成を記述した設定ファイルを `trema run` に渡します。たとえば，リスト 2 の設定ファイルでは仮想スイッチ (`vswitch`) を 2 台定義しています。

```ruby
vswitch { datapath_id 0xabc }
vswitch { datapath_id 0xdef }
```

リスト2　仮想ネットワークに仮想スイッチを2台追加

それぞれに指定されている `datapath_id` (`0xabc`， `0xdef`) はネットワークカードにおける MAC アドレスのような存在で，スイッチを一意に特定する ID として使われます。OpenFlow の規格によると，64 ビットの一意な整数値を OpenFlow スイッチ 1 台ごとに割り振ることになっています。仮想スイッチでは好きな値を設定できるので，かぶらないように適当な値をセットしてください。

```ruby
class SwitchMonitor < Controller
  periodic_timer_event :show_switches, 10 # (3)

  def start
    @switches = []
  end

  def switch_ready datapath_id # (1)
    @switches << datapath_id.to_hex
    info "Switch #{ datapath_id.to_hex } is UP"
  end

  def switch_disconnected datapath_id # (2)
    @switches -= [datapath_id.to_hex ]
    info "Switch #{ datapath_id.to_hex } is DOWN"
  end

  private # (3)
  def show_switches
    info "All switches = " + @switches.sort.join( ", " )
  end
end
```

リスト3　SwitchMonitor コントローラ

それでは，さきほど定義したスイッチを起動してコントローラから捕捉してみましょう。スイッチの起動イベントを捕捉するには `switch_ready` ハンドラを書きます (リスト 3-1)。

`@switches` は現在起動しているスイッチのリストを管理するインスタンス変数で，新しくスイッチが起動するとスイッチの `datapath_id` が追加されます。また，`puts` メソッドで `datapath_id` を表示します。

### スイッチの切断を捕捉する

同様に，スイッチが落ちて接続が切れたイベントを捕捉してみましょう。このためのハンドラは `switch_disconnected` です (リスト 3-2)。

スイッチの切断を捕捉すると，切断したスイッチの `datapath_id` をスイッチ一覧 `@switches` から除きます。また，`datapath_id` を `puts` メソッドで表示します。

### スイッチの一覧を表示する

最後に，スイッチの一覧を定期的に表示する部分を作ります。一定時間ごとに何らかの処理を行いたい場合には，タイマー機能を使います。リスト 3-3 のように，一定の間隔で呼びたいメソッドと間隔 (秒数) を `periodic_timer_event` で指定すると，指定されたメソッドが呼ばれます。ここでは，スイッチの一覧を表示するメソッド `show_switches` を 10 秒ごとに呼び出します。

### 実行

それでは早速実行してみましょう。仮想スイッチを 3 台起動する場合，リスト 4 の内容のファイルを `switch-monitor.conf` として保存し，設定ファイルを `trema run` の `-c` オプションに渡してください。

```ruby
vswitch { datapath_id 0x1 }
vswitch { datapath_id 0x2 }
vswitch { datapath_id 0x3 }
```

リスト4　仮想スイッチを3台定義

実行結果は次のようになります。

    % ./trema run ./switch-monitor.rb -c ./switch-monitor.conf
    Switch 0x3 is UP
    Switch 0x2 is UP
    Switch 0x1 is UP
    All switches = 0x1, 0x2, 0x3
    All switches = 0x1, 0x2, 0x3
    All switches = 0x1, 0x2, 0x3
    ……

`switch-monitor` コントローラが起動すると設定ファイルで定義した仮想スイッチ 3 台が起動し，`switch-monitor` コントローラの `switch_ready` ハンドラによって捕捉され，このメッセージが出力されました。

それでは，スイッチの切断がうまく検出されるか確かめてみましょう。スイッチを停止するコマンドは `trema kill` です。別ターミナルを開き，次のコマンドでスイッチ `0x3` を落としてみてください。

    % ./trema kill 0x3

すると，`trema run` を動かしたターミナルに次の出力が表示されているはずです。

    % ./trema run ./switch-monitor.rb -c ./switch-monitor.conf
    Switch 0x3 is UP
    Switch 0x2 is UP
    Switch 0x1 is UP
    All switches = 0x1, 0x2, 0x3
    All switches = 0x1, 0x2, 0x3
    All switches = 0x1, 0x2, 0x3
    ……
    Switch 0x3 is DOWN

うまくいきました! おわかりのとおり，このメッセージは `switch_disconnected` ハンドラによって表示されたものです。


---

### 友太郎の質問 datapath ってなに?

**Q.** 「こんにちは! 僕は最近 OpenFlow に興味を持ったプログラマ，友太郎です。スイッチに付いている ID を datapath ID って呼ぶのはわかったけど，いったい datapath ってなに? スイッチのこと?」

**A.** 実用的には 「datapath = OpenFlow スイッチ」 と考えて問題ありません。

「データパス」 でググると，「CPUは演算処理を行うデータパスと，指示を出すコントローラから構成されます」 というハードウェア教科書の記述がみつかります。つまり，ハードウェアの世界では一般に 「筋肉にあたる部分 = データパス」「脳にあたる部分 = コントローラ」 という分類をするようです。

OpenFlow の世界でも同じ用法が踏襲されています。OpenFlow のデータパスはパケット処理を行うスイッチを示し，その制御を行うソフトウェア部分をコントローラと呼びます。

---


# まとめ

すべてのコントローラのテンプレートとなる Hello, Trema! コントローラを書きました。また，これを改造してスイッチの動作状況を監視するスイッチモニタを作りました。学んだことは次の3つです。

* OpenFlow ネットワークはパケットを処理するスイッチ (datapath) と，スイッチを制御するソフトウェア (コントローラ) から構成される。Trema は，このコントローラを書くためのプログラミングフレームワークである
* Trema は仮想ネットワーク構築機能を持っており，OpenFlow スイッチを持っていなくてもコントローラの開発やテストが可能。たとえば，仮想ネットワークに仮想スイッチを追加し，任意の datapath ID を設定できる
* コントローラは Ruby の Controller クラスを継承し，OpenFlow の各種イベントに対応するハンドラを定義することでスイッチをコントロールできる。たとえば，`switch_ready` と `switch_disconnected` ハンドラでスイッチの起動と切断イベントに対応するアクションを書ける

次回はいよいよ本格的なコントローラとして，トラフィック集計機能のあるレイヤ 2 スイッチを作ります。初歩的なレイヤ 2 スイッチング機能と，誰がどのくらいネットワークトラフィックを発生させているかを集計する機能を OpenFlow で実現します。


---

### 友太郎の質問: `switch_ready` ってなに ?

**Q.** 「OpenFlow の仕様を読んでみたけど，どこにも `switch_ready` って出てこなかったよ? OpenFlow にそんなイベントが定義されてるの?」

**A.** `switch_ready` は Trema 独自のイベントで，スイッチが Trema に接続し指示が出せるようになった段階でコントローラに送られます。実は，`switch_ready` の裏では図 A の一連の処理が行われており，Trema が OpenFlow プロトコルの詳細をうまくカーペットの裏に隠してくれているのです。

![switch_ready イベントが起こるまで](https://github.com/trema/Programming-Trema/raw/master/images/1_00a.png)

図A　switch_ready イベントが起こるまで

最初に，スイッチとコントローラがしゃべる OpenFlow プロトコルが合っているか確認します。OpenFlow の `HELLO` メッセージを使ってお互いのプロトコルバージョンを確認し，うまく会話できそうか確認します。

次は，スイッチを識別するための datapath ID の取得です。datapath ID のようなスイッチ固有の情報は，スイッチに対して OpenFlow の Features Request メッセージを送ることで取得できます。成功した場合，datapath ID やポート数などの情報が Features Reply メッセージに乗ってやってきます。

最後にスイッチを初期化します。スイッチに以前の状態が残っていると，コントローラが管理する情報と競合が起こるため，初期化することでこれを避けます。これら一連の処理が終わると，ようやく `switch_ready` がコントローラに通知されます。
