---
title: "シェルスクリプトの時間精度を向上させると急に面白くなる"
emoji: "🚴"
type: "idea"
topics: ["unix", "linux", "shellscript", "iot", "posix"]
published: true
---

# イントロダクション

シェルスクリプトはファイルの整理やコマンドの起動に便利だが、それだけにしか使っていないという人も多いだろう。しかし、高精度（秒未満）で時間が制御できるようになると急に面白くなる。それは、高精度にデータを流すタイミングを制御するとか、高精度にコマンドの起動タイミングを制御するといったことだ。

そのためのUNIXコマンドコレクションを作ったので紹介しつつ、面白いことをいくつか披露する。

# 準備（インストール）

コマンドコレクションの名前は「[tokideli（トキデリ）](https://github.com/ShellShoccar-jpn/tokideli)」で、これをインストールする。インストールの仕方は次のとおり。

## Homebrewが使える人

Homebrew tap（Homebrewで野良パッケージを作る仕組み）に対応させたので、次の二つのコマンドを叩くだけ。

```
$ brew tap ShellShoccar-jpn/tap
$ brew install tokideli
```

## その他の環境の人

GitHubからソースコードをcloneしてビルドして、パスを通す。ただし、cloneした後、ビルドとパス通しをやってくれるスクリプトは作ったので、つまり、下記のコマンドを実行すればよい。

```
# 管理者権限でインストールしたい人向け
$ git clone https://github.com/ShellShoccar-jpn/tokideli.git
$ sudo tokideli/INSTALLIN.sh /usr/local/tokideli

# 自分のディレクトリー内にインストールしたい人向け
$ git clone https://github.com/ShellShoccar-jpn/tokideli.git
$ tokideli/INSTALLIN.sh $HOME/tokideli
```

なお、アンインストールした時は、INSTALLIN.shスクリプトで指定したディレクトリーを消して、追加したPATHを消すだけ。

# とにかくいろいろ遊んでみる

能書きを垂れるよりも、とにかく使って遊んでみるとよくわかると思うので、ひたすら例を紹介する。

## 古のパソコン通信を懐かしむ

1990年代くらいまでは、パソコンは電話回線につないでテキスト主体の通信をしていたのだが、通信速度は300bpsとか1200bpsとか9600bpsとかで文字がチョロチョロ流れる世界だった。「ギガ」世代の人はピンと来ないだろうから、[`valve`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.ja.md)（valveは「弁」の意味）を使ったワンライナーでその速度を体験してもらいたい。（知ってる人は試してみてノスタルジーに浸ってもらいたい）

```sh:
$ fortune | valve 300bps
$ fortune | valve 1200bps
$ fortune | valve 9600bps
```

![fortuneメッセージをノスタルジックに表示した例](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/fortune_valve_demo.gif)

なお、[`fortune`コマンド](https://ja.wikipedia.org/wiki/Fortune_(UNIX))が無いという人は、`apt`や`brew`でインストールしてもらってもいいし、手近なテキストファイルを`cat`して[`valve`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.ja.md)に流してもらっても構わない。

## SLをノロノロ走らせる

UNIXには[`sl`](https://www.tkl.iis.u-tokyo.ac.jp/~toyoda/)というコマンドがある。`ls`コマンドをtypoして"sl"と打ってしまうと、SLが走り、通り過ぎるまで何もできないという嫌がらせコマンドだ。（これもやはり、入っていない人は`apt`や`brew`でインストールしておいてもらいたい）
![slコマンド（通常運行）](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/sl_demo.gif)

そして次のコマンドを打ち込むと、パイプの転送速度を9600bpsに制限しているおかげでSLがノロノロとまかり通ってターミナルを遮る。
```sh:
$ sl | valve 9600bps
```
![slコマンド（ノロノロ運転）](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/sl_valve_demo.gif)

「余計に鬱陶しくなっただけじゃねーか！」と思うかもしれないが、[Ctrl]+[C]による中断ができるのでウザくなったら止めてくれ。（ただし、その後`reset`コマンドを実行して画面の乱れを直すこと）

## レートリミットでBANされないようにDLする

次は少し実用的な例を紹介しよう。

何らかの理由で連番ファイルを大量にダウンロードしたいことがあるだろう（あるよな？）。だが、間髪入れずにダウンロードするとBANされてしまうかもしれないので、レートリミットをしっかり守ってダウンロードしたいと考えたとする。そんな時に役立つのがやっぱり[`valve`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.ja.md)だ。

例えば、3秒より短い間隔でファイルをダウンロードしていると、やがてレートリミットに引っかかってBANされる恐れのあるWebサイトがあったとする。そういう時は次のようなワンライナーを書けば、`while`～`done`の中のコマンドの実行時間に多少揺らぎがあっても3秒インターバルでループするようになるので、レートリミットを守ることができる。（ただし、中のコマンドの実行時間が3秒以上かかると次の周回でその分短くなるので注意😉）
```sh:
$ seq -f '%02.0f' 1 50 | valve -l 3s | while read n; do wget "https://example.com/file_${n}.jpg"; done
```
なお、[`valve`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.ja.md)に付けた、`-l`オプションは「行単位にウェイトを入れるようにせよ」という意味で、この例では3秒ごとに1行が出力される。

実際に動かしてみると、次の画像のようにきちんと3秒間隔でダウンロードが実行されていていい感じである。
![valve+wgetで何かの連番ファイルをダウンロードする](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/wget_valve_demo.gif)

ここでは、[`wget`コマンド](https://www.gnu.org/software/wget/)を使ったファイルダウンロードを例にしたが、こういうワンライナーやシェルスクリプトを書けば、どんなコマンドも正確なインターバルで連続実行できるということだ。

## `yes`コマンドの新たな使い道（パルス発生器）

「[`yes`コマンド](https://ja.wikipedia.org/wiki/Yes_(UNIX))の使い道を説明せよ」と言われたら、多くの人は「何度も繰り返される『`(y/n)?`質問』に`y`と答え続けるものぐさコマンドで、それ以外に使い道は無い」と答えることだろう。しかし、[`valve`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.ja.md)と組み合わせることによって、組み込み機器における**パルス発生器としての役割が爆誕する**のだ。

次の図のように、[Raspberry Pi](https://www.raspberrypi.com/)とオシロスコープ、そして参照用のパルスジェネレーターを接続する。
![Raspberry Piとオシロスコープ、参照用パルスジェネレーターの接続図](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/valve_as_wave_generator_on_raspi_fig01.png)
そして[Raspberry Pi](https://www.raspberrypi.com/)上で次のワンライナーを動かすと、GPIO端子から指定周波数の方形波が出てくるのだ。
```sh:
$ yes $(printf '0\n1') | valve -l TIME > /sys/class/gpio/gpio2/value
```

実際に計測した様子は、次の動画リンクを参照のこと。
| 周波数 | valveに与える値 | その様子を撮影した動画 |
|---|---|---|
| 100Hz | ≒ 5ms | <https://youtu.be/e8Qx46Gbz-s> |
| 1kHz | ≒ 0.5ms | <https://youtu.be/1WZZLyjXYvg> |
| 10kHz | ≒ 50us | <https://youtu.be/a-X0ydGcugY> |

[Raspberry Pi](https://www.raspberrypi.com/)という安いコンピューターであっても、1kHz程度までなら実用的なパルスが出せていることがわかる。**パルスの源は[`valve`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.ja.md)ではなく、Yes！そのとおり、[`yes`コマンド](https://ja.wikipedia.org/wiki/Yes_(UNIX))だ。**

なお、この実験に関する詳しいレポートは[valveコマンドをRaspberry Pi上で方形波発生器として使う](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve_as_wave_generator_on_raspi.info.ja.md)というドキュメントを参照のこと。

## "hello, world!"のタイプタイミングの記録・再生

[tokideli（トキデリ）](https://github.com/ShellShoccar-jpn/tokideli)には18個もコマンドがあるのに、[`valve`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.ja.md)の使用例ばかりじゃ勿体無いので、他のコマンド（[`typeliner`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/typeliner.man.ja.md)、[`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.ja.md)、[`tscat`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/tscat.man.ja.md)）の使用例も紹介しよう。

さて、プログラマーの間では何か新しいことを始めるといったら「[hello, world!](https://ja.wikipedia.org/wiki/Hello_world)」を表示させるのが定番である。ここでは一捻りして、キーボードから"hello, world!"とタイプするそのタイミングを含めて表示してみよう。

まずは記録から。次のワンライナーを打ち込んだらファイル（`hw.log`）にタイプする文字とそのタイミングの記録が始まる。始まったら、"hello, world!"と好きなタイミングでタイプして、最後に[Ctrl]+[D]を押してもらいたい。
```sh:
$ typeliner -e | linets -3 > hw.log
（ここで、"hello, world!"などと打ち込む）
（最後に[Ctrl]+[D]を押して記録を終了させる）
```
これら二つのコマンドについて補足をすると次のとおりだ。
* [`typeliner`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/typeliner.man.ja.md)
  * キーボードを一文字タイプするたびに標準出力へ文字を送ると同時に、改行を付け足して1文字1行化するというコマンド。このコマンドを使わないと、[Enter]キーを押すまで標準出力には1文字も送られない。
  * `-e`は、今打ち込んだ文字が目視できるようにローカルエコーするためのオプション。
* [`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.ja.md)
  * 到来した各行の先頭にタイムスタンプを付けるコマンド。（lineにタイムスタンプtsを付けることが名前の由来）
  * `-3`とは、小数点以下3桁、つまりミリ秒精度のタイムスタンプにせよというオプション。
  * 他のオプションを付けない場合、タイムスタンプは`YYYYMMDDhhmmss.nnn`の形式になる。

ちなみに、これでできる`hw.log`の中身はこんな感じだ。
```text:
20261014004947.408 h
20261014004947.559 e
20261014004947.809 l
20261014004947.956 l
20261014004948.252 o
20261014004948.527 ,
20261014004949.057
20261014004950.255 w
20261014004950.441 o
20261014004950.650 r
20261014004950.818 l
20261014004950.901 d
20261014004951.235 !
20261014004951.803
```

さて、次は再生である。次のワンライナーを実行するだけだ。
```sh:
$ tscat -yZ hw.log
```
コマンドの補足
* [`tscat`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/tscat.man.ja.md)
  * 行頭にタイムスタンプがついたテキストデータを受け取り、そのタイムスタンプの示す日時になったらタイムスタンプを外してその行を標準出力するというコマンド。[linets](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.ja.md)の逆。（タイムスタンプtsに対応したcatコマンドという意味）
  * `-Z`は、先頭行のタイムスタンプを基準に後続の行の出力タイミングを決めるという意味のオプション。記録したタイムスタンプは当然ながらすべて過去の日時なので、これを付けないと、全行が今すぐ出力されてしまう。
  * `-y`は、元々が文字単位だったデータを復元するためのオプション。これがないと、各文字が1行ごと（縦並び）に表示されてしまう。

ということで、筆者の"hello, world!"タイピングを記録・再生した動画がこれである。
![タイミング付きhello,world!](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/hw_typing_demo.gif)

## 三三七拍子の記録・再生（しかも倍速＆1/2倍速対応）

"hello, world!"は、ほんの序の口。次は三三七拍子のタイミングでタイプして、等速で再生した後、さらに倍速・1/2倍速で再生してやろうという例だ。

まずは記録。先ほどとはちょっと違うワンライナーを書いて、記録をする。
```sh:
$ typeliner -e | linets -3Z > 337.log
（ここで、三三七拍子のリズムで何かの文字キーを叩く）
（終わったら[Ctrl]+[D]を押して記録を終了させる）
```
[`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.ja.md)に今度は`-Z`というオプションを付けているが、これは到来した最初の行を0とした経過時間を示すタイムスタンプ形式にするという意味である。なぜそうするかは、トリック再生をする時にわかる。

では再生だ。まずは等速で再生するが、次のワンライナーは、三三七拍子のリズムが分かりやすいようにビープ音を付けて再生するという例だ。
```sh:
$ cat 337.log | sed $(printf 's/$/\007/') | tscat -zZ
```
[`tscat`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/tscat.man.ja.md)に今度は`-z`オプションが追加されているが、これは[`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.ja.md)で作成されたタイムスタンプ形式が、最初の行を基準とする経過時間になっていることを知らせるためのものだ。

再生の様子を収録した動画を披露する前に、倍速再生と1/2倍速再生のワンライナーも記しておこう。
```sh:
$ cat 337.log | awk '{print $1/2,$2}' | sed $(printf 's/$/\007/') | tscat -zZ
$ cat 337.log | awk '{print $1*2,$2}' | sed $(printf 's/$/\007/') | tscat -zZ
```
`awk`コマンドで1列目にあるタイムスタンプを1/2にしたり、2倍にしているのがミソ。これがやりたいから、1行目を基準とした経過秒数にしたわけだ。（年月日時分秒形式だったりUNIX時間形式である場合はできないでしょ？）

では、記録から再生までの様子を収録した動画をご覧いただこう。画面が三三七拍子のタイミングでシュパッシュパッと光っている（可視化されたビープ音による）のがわかるだろう。
![三三七拍子の記録・再生・倍速再生・スロー再生](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/typing_337_demo.gif)

## `delay`コマンドでやまびこヤッホー

標準入力からやってきたデータを一定秒数遅らせて標準出力に送る[`delay`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/delay.man.ja.md)というコマンドがある。これで手っ取り早く遊ぶべく、やまびこを鳴らしてみようと思う。

まずは、次のワンライナーを打ち込む。打ち込んだら、"yo-ho!"（「ヤッホー！」という意味）と素早く打ち込んで[Enter]キーを押してもらいたい。
```sh:
$ typeliner -t "" | (tee /dev/stderr | delay 3s) 2>&1
（ここで"yo-ho!"[Enter]と打ち込むと……）
（それがやまびこになって帰ってくる）
```

さらに、ワンライナーを伸ばすとやまびこが二重になる。
```sh:
$ typeliner -t '' | (( tee /dev/stderr | delay 3s ) 2>&1 | tee /dev/stderr | delay 6s) 2>&1
（ここで"yo-ho!"[Enter]と打ち込む（=やまびこ0号））
（やまびこ1号が返ってくる）
（やまびこ2号が返ってくる）
（やまびこ3号が返ってくる）
```

これらの様子を撮影した動画はこれだ。
![やまびこ](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/delay_echo_demo.gif)

なお、[`typeliner`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/typeliner.man.ja.md)で、`-t ''`というオプションを付けているが、これは1文字ごとに改行を挿入していたのをやめて、オプションで指定した文字を挿入せよという意味。ここではそれが空文字になっているので、つまり改行するなという意味になる。

## `stdbuf`効かない問題を解決

最後に、[`ptw`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/ptw.man.ja.md)というコマンドを紹介しておく。一言で説明するならこれは[`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html)コマンドの強化版である。

そもそも[`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html)というコマンドがなぜ作られたのかといえば、多くのコマンドは処理を効率化させるために、ある程度テキストデータが溜まってから処理をしようとするからだ。人間だってそうだろう。100通の書類に承認印を押して1通ずつ個別の封筒に入れる時、1通ずつ押印・封入するよりも、とりあえず100通全部机に並べて押印し、封入はあとでまとめて行おうとするだろう。しかし、それをコマンドにされてしまうと、せっかくタイミングを調整して送ったデータのタイミングが壊れてしまう。それを防ぐために有効というわけだ。

ところが、[`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html)が効かないケースが結構ある。例えば、みんな大好きPythonも効かないものの一つだ。`python3`コマンドにstdbufを適用しても効かない。

例えば、次のPythonスクリプト（[Hacker News](https://news.ycombinator.com/)上の高スコア記事を抜き出す）を保存し……
```python:
from datetime import datetime
import requests

top_ids = requests.get(
  "https://hacker-news.firebaseio.com/v0/topstories.json"
).json()

for story_id in top_ids[:30]:
  story = requests.get(
    f"https://hacker-news.firebaseio.com/v0/item/{story_id}.json"
  ).json()

  if story.get("score", 0) >= 100:
    score = story["score"]

    # UNIX epoch to YYYY-MM-DD HH:MM:SS
    posted_at = datetime.fromtimestamp(
      story["time"]
    ).strftime("%Y-%m-%d %H:%M:%S")

    title = story["title"]

    print(f"[{score:3d}] {posted_at} {title}")
```

次のワンライナーで、実行した時の挙動を覚えておいてもらいたい。
```
$ python3 best_articles_on_hackernews.py
```
APIにアクセスして、ジワジワと記事が表示される。
![元のPythonスクリプトの動き](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/ptw_demo_part1.gif)

この出力結果に対して、`grep`コマンドで今日の記事だけに絞り込むべく次のワンライナーを書くと、溜め込みが発生して、最後に一気に表示される動きになってしまう。
```
$ python3 best_articles_on_hackernews.py | grep $(date +%Y-%m-%d)
```

そこで溜め込みを抑制すべく、`python3`コマンドに[`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html)を適用しようとするものの、挙動は全く変わらない。
```
$ stdbuf -o L python3 best_articles_on_hackernews.py | grep $(date +%Y-%m-%d)
```

しかし、[`ptw`コマンド](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/ptw.man.ja.md)だとうまくいく。
```
$ ptw python3 best_articles_on_hackernews.py | grep $(date +%Y-%m-%d)
```

それぞれの挙動を動画に収録したので見てもらいたい。
![なし・stdbuf・ptwそれぞれの動き](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/ptw_demo_part2.gif)

便利だろぅ？なぜうまくいくかについて解説すると長くなるので、興味がある人は[PTWコマンドの役割について](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/ptw.info.ja.md)を読んでもらいたい。

# まとめ

プログラマーは速さばかり関心を持ちがちだが、タイミングが重要な場面がたくさんある。CDプレーヤーがいくら高性能だからといっても74分の音楽を1秒で再生されたらワケがわらないし、今話題のフィジカルAIロボットだって適切なタイミングを無視して手足を速く動かしすぎればコケる。

ゆえに、正確なタイミングを制御できるようになると実用性が急に増す。[tokideli（トキデリ）](https://github.com/ShellShoccar-jpn/tokideli)とはそのためのコマンドコレクションであり、すなわちこれを獲得したシェルスクリプトは急に実用性が増すのである。

この記事はエンターテインメント（面白さ）の方向に振って紹介する記事に仕上げたが、どれも基礎的な機能を持つコマンドなので、開発に、デバッグに、保守に、様々な実用的なシーンで活用してもらいたい。