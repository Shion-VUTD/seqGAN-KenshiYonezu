# SeqGAN
実践プロジェクト 

## 目的 
米津玄師さんの歌詞データを元に、「米津さんっぽい歌詞」を自動的に生成すること。

## 設計 

* LSTM

[この](a1d2d159c614258828ab) コードを元に、LSTMによる歌詞生成を試みたが、epochが進むに つれ、入力データと全く同じ歌詞が生成されてしまうことが分かった。コードを調べると、 すでに歌詞中にある単語を一番目に持ってきて、そこから次の単語を予測していることが原 因であることが分かった。この状態を避け、かつ入力データ以外の単語も用いて生成するに は、LSTM では限界があると考えた。

* SeqGAN

方針としては、[この本](https://www.amazon.co.jp/現場で使える！Python深層強化学習入門-強化学習と深層学習による探索と制御-AI-TECHNOLOGY-伊藤/dp/4798159921)を参考にし、この資料に参考資料と して添付されているコードを改変して利用した。元コードでは生成器に学習させるデータと 識別器に学習させるデータを同一のものにしていたが、LSTMでの結果を踏まえて、今回は生成器には通 常のJーPOPの歌詞、識別器には米津玄師さんの曲の歌詞を学習させることで、米津さんが使 っていない単語からでも、米津さんっぽい歌詞が生成されることを目指した。

## データセット

生成器への学習データは、 [アイネクライネ](https://www.uta-net.com/search/?Aselect=6&Bselect=3&Keyword=あ&sort=&pnum=1) 以降、曲名の平仮名表 記に「あ」がつく曲 10006 曲分の歌詞データを、スクレイピングにより利用した(歌ってい る歌手が異なるなど、同一の歌詞データは一部削除している。また、英語の曲も排除して収 集している)。これらのデータは一曲毎に改行されており、これらを単語毎に分かち書きした ものを kashi_wakati.txt とした。識別器への学習データは、 [うたまっぷサイト](https://www.utamap.com/searchkasi.php?searchname=artist&word=%95%C4%92%C3%8C%BA%8Et&act=search&search_by_keyword=%8C%9F%26%23160%3B%26%23160%3B%26%23160%3B%8D%F5&sortname=1&pattern=1)から、2020年6月以前に、米津玄師名義で発表された楽曲80曲分の 歌詞を、スクレイピングにより利用した。これも同様に分かち書きを行い、yonezu_wakati.txt とした。さらに、この 2 つのデータを連結させたものを wakati.txt とし、これを元に単語コ ーパスを作成し、登場した各単語に ID を割り振ることで、曲の歌詞を数値に置き換えた。こ のデータを米津玄師さんの楽曲の ID 列とその他の楽曲の ID 列に分け、各々yonezu_id.txt 及びkashi_id.txtとした。これらのデータも、分かち書きのデータとともにそれぞれ識別器 と生成器に学習させた。([参考1](https://qiita.com/Senple/items/1ad08b1a7ac9560bef62) [参考2](https://techacademy.jp/magazine/30591、pytorch-dataset-transforms))
         
## 本体
(改変部分のみ一部抜粋)

```
#generator へのデータを追加
input_data2 = os.path.join('data','/content/kashi_wakati.txt')
id_input_data2 = os.path.join('data','/content/kashi_id.txt')
```
生成器へのデータを、識別器のデータとは別に組み込んだ。
```
#追加データのベクトル化
vocab2 = Vocab(input_data2)
vocab_size2 = vocab2.vocab_num
pos_sentence_num2 = vocab2.sentence_num vocab2.write_word2id(input_data2, id_input_data2) sampling_num2 = vocab2.data_num

```
追加データのベクトル化を行った。
```
#元コードに戻る
env = Environment(batch_size, vocab_size, emb_size, d_hidden, T, dropout, d_lr)
agent = Agent(sess, vocab_size2, emb_size, g_hidden, T, g_lr)
#vocab_size -> vocab_size2

```
生成器のデータサイズ(データの次元量)に合わせるように改変した。
```
def pre_train():
g_data = DataForGenerator(
　　 id_input_data2, #id_input_data -> id_input_data2 batch_size, T,
    vocab2 #vocab ->vocab2
)
agent.pre_train(
    g_data,
    g_pre_episodes,
    g_pre_weight,
    g_pre_lr
)
agent.generate_id_samples(
    agent.generator, T,
    sampling_num2, #sumpling_num -> sumpling_num2 pre_id_output_data
)

vocab2.write_id2word(pre_id_output_data, pre_output_data)
d_data = DataForDiscriminator( id_input_data,
    pre_id_output_data, batch_size, T,
    vocab  #vocab -> vocab2
)
env.pre_train(d_data, d_pre_episodes, d_pre_weight, d_pre_lr)
```
事前学習の部分を、生成器と識別器に違うデータが入るように改変した。g_data の部分を 各々
id_input_data2、 vocab2 に改変する他に、train 段階で生成器に入力されるデータ部分も、 id_input_data2、 vocab2 と改変した。

```
def d_train(): agent.generate_id_samples( 
    agent.generator, T,
    sampling_num2, #sumpling_num -> sumpling_num2
    id_output_data,
)
vocab2.write_id2word(id_output_data, output_data) #vocab -> vocab2
d_data = DataForDiscriminator(
    id_input_data, id_output_data,
    batch_size, T, vocab)
env.discriminator.fit_generator(d_data, steps_per_epoch=None, epochs=1)

```
識別器の訓練は、正解データに加え、生成器が作った偽のデータが追加されることで学習が進むことから、生成器のデータに関わる部分を改変する必要があった。
## 結果
* SeqGAN による結果

なにか 忘れ られ ない よ だろ う ウッフウッフッフ 合っ たら 道 に も 遠く を つな ぐ 前 に 行こ う 君 の 君 を 信じ
愛 の 山 に しよ う 世界 の 町 を 包む 破滅 へ と 追いかけ て く 置き去り の 雲 街 ばかり 一瞬 も 水

古い 何 か あ そう やっ て よ フィールド の 片隅 で 鍵 で 思え ば わたし に 感じ て 涙 を 振っ て も
こんなに 昼過ぎ に キレイ な 強い もっと いつも いつ まで も 泣き たい や 影 の ほ ろにがい ずっと 離れ て も 今 でも ~ 泣い
夕暮れ たち を 運ぶ ここ から あなた 愛さ れ たい 何 も 無い 恋 は ない けど 限ら ず 笑っ たり 笑っ たり 傷つけ たい
晴れ た 髪 を 見かけ た 雫 手の平 に 咲く 島 へ と 浮かべ た ベイビー 鏡 に 遠い 星の春の中、
何 も 見え なく て それでも 本当は あなた の グレー を 胸 が 明ける の 日 も 忘れ なかっ た の 心 を 見 た
遠く 遠く 溢れる この 街 に も きっと 教え て 泣き たく なる ただ 大事 な あなた が 倒れ た カケラ の 中 に ひとつ
ねぇ に 問いかける 壁 僕ら は はじまり 子供 達 は いつ に なる ん じゃ ない の か 理由 が あっ た じゃ 傷つく 為
優し さ の 好き に 君 この 町 から さよなら は 切ない さ そこ に 続く 街 から こん なに 叫ぶ 時 は ない 信じ て
君 の そば で くれ た の 跡形 も 中 に つながる の なら 感じる 心 に 沈ま せる 銀 色 の 愛 俺 の 全て
どうして こんなに 私 」 なんて 言え ない よ ウニ つい ちゃっ て 涙し たり ひと 吹 きで やわ な 双子 の 今年 の こと ばかり で
あなた の 心 を 呼び リボン 海 に ゆく の 下 過ぎ て 夢 に 私 だけ 生まれ て 外 が ある あなた は 見
伝え たい 熱い 恋 が 素知らぬ 振り ゃないくっついちゃってもいいんじゃない 僕 の 為 俺 も 幻 に 繋がる くらい 君 を 見 て そう な の は
愛 の 数 まで 咲き乱れ て 震え て いる の よ 背中 が 切ない う 早く も うち 楽しい 笑い声 流し て しまう の 待っ
ジンジャー で 恐怖 の 《 山 の 畑 の 自分 を 異人 さん 、 お 焦げ て やり 抜く いく あの 娘 不 の 頭
## 課題 
今回の結果から、まず第一に、「米津さんっぽい」の指標をどのように数値化すべきかが
課題となる。自分の技術不足により、MNIST などの画像データで GAN を行ったときの loss に あたる関数の定義を、SeqGAN にどのように当てはめれば良いのかが分からなかった。また、 それが定義できたとしても、必ずしも、識別器が産出する損失の値が「米津さんっぽさ」の 指標になるとは言えない。この基準は非常に主観的な問題であるため、議論が必要である。
さらに、生成器に学習させたデータが約 10000 曲であったのに対し、識別器の学習データ は高々80 曲に過ぎず、このデータの少なさが精度の低下を招いたと考えられる。テキストデ ータを水増しする方法を学習すれば、改善する可能性がある。また、生成器の歌詞データを J-pop からランダムに抽出するのではなく、米津玄師さんが影響を受けたとされる BUMP OF CHICKEN や RADWIMPS、あるいは音楽性が類似する King Gnu の歌詞、あるいはハチ時代の歌 詞などを中心に抽出することで、生成器そのものの精度が上がれば、より直感的に「米津さ んっぽい」歌詞が生成されるかもしれない。また、データの前処理の段階で、どの曲にも多 く登場する単語をストップワードとして除去しておけば、さらに精度が上がったと思われる。

## 追記
以下のサイトのコードをそのまま使わせて頂いて分類器を作成し、生成データの精度を確か めてみました。ここでは、 AKB48、嵐、BUMP OF CHICKEN、いきものがかり、中島みゆき、西 野カナ、ポルノグラフティ、椎名林檎、SMAP、米津玄師 の各楽曲の歌詞から正しいアーテ ィスト名を答える分類問題を学習させました。
### データセット
SeqGAN と同じスクレイピングコードを利用して、
AKB48:200 曲
嵐:200 曲
BUMP OF CHICKEN:126 曲
いきものがかり:143 曲
中島みゆき:200 曲
西野カナ:172 曲
ポルノグラフティ:198 曲
椎名林檎:108 曲
SMAP:196 曲
米津玄師:80曲 のデータを、[アーティスト名、歌詞]を列名にとるデータフレームを作成 しました。
### 本体
python-scikit-learnで学ぶパーセプトロンによる文書分類入門 をそのまま使いました。 そのままコードを利用すると、テストデータでの分類器の正答率は6割ほどです(結果参照)。 この分類器に、SeqGAN で生成した歌詞フレーズを1行ずつ入れ、分類させました。
### 結果
* 一曲目
Precision: 0.5935960591133005

Recall: 0.5935960591133005(正答率)

F-score: 0.5935960591133005(F 値)

Tokenized: なにか 忘れ られ ない よ だろ う ウッフウッフッフ 合っ たら 道 に も 遠
く を つなぐ 前 に 行こ う 君 の 君 を 信じ 

Label: 米津玄師

* 2曲目
Precision: 0.6330049261083743

Recall: 0.6330049261083743

F-score: 0.6330049261083743

Tokenized: 愛 の 山 に しよ う 世界 の 町 を 包む 破滅 へ と 追いかけ て く 置き去 り の 雲 街 ばかり 一瞬 も 水

Label: SMAP

* 3曲目
Precision: 0.625615763546798
 
Recall: 0.625615763546798

F-score: 0.625615763546798

Tokenized: 古い何かあそうやってよフィールドの片隅で鍵で思えばわた
し に 感じ て 涙 を 振っ て も Label: 椎名林檎
Precision: 0.6083743842364532
Recall: 0.6083743842364532
F-score: 0.6083743842364532
Tokenized: こんなに 昼過ぎ に キレイ な 強い もっと いつも いつ まで も 泣き たい や 影 の ほろにがい ずっと 離れ て も 今 でも 泣い
Label: 西野カナ

* 4曲目
Precision: 0.6231527093596059
Recall: 0.6231527093596059
F-score: 0.6231527093596059
Tokenized: 夕暮れ たち を 運ぶ ここ から あなた 愛さ れ たい 何 も 無い 恋 は ない けど 限ら ず 笑っ たり 笑っ たり 傷つけ たい
Label: 椎名林檎

* 5曲目
Precision: 0.6108374384236454
Recall: 0.6108374384236454
F-score: 0.6108374384236454
Tokenized: 晴れた髪を見かけた雫手の平に咲く島へと浮かべたベイビー 鏡 に 遠い 星 の 春 の 中
Label: ポルノグラフティ


Precision: 0.6059113300492611
Recall: 0.6059113300492611
F-score: 0.6059113300492611
Tokenized: 何も見えなくてそれでも本当はあなたのグレーを胸が明けるの 日 も 忘れ なかっ た の 心 を 見 た
Label: BUMP OF CHICKEN


Precision: 0.6009852216748769 Recall: 0.6009852216748769 F-score: 0.6009852216748769
Tokenized: 遠く遠く溢れるこの街にもきっと教えて泣きたくなるただ大事
な あなた が 倒れ た カケラ の 中 に ひとつ Label: 米津玄師


Precision: 0.645320197044335
Recall: 0.645320197044335
F-score: 0.645320197044335
Tokenized: ねぇに問いかける壁僕らははじまり子供達はいつになるんじゃ ない の か 理由 が あっ た じゃ 傷つく 為
Label: 中島みゆき


Precision: 0.645320197044335
Recall: 0.645320197044335
F-score: 0.645320197044335
Tokenized: 優しさの好きに君この町からさよならは切ないさそこに続く 街 から こんなに 叫ぶ 時 は ない 信じ て
Label: SMAP


Precision: 0.6059113300492611
Recall: 0.6059113300492611
F-score: 0.6059113300492611
Tokenized: 君 の そば で くれ た の 跡形 も 中 に つながる の なら 感じる 心 に 沈 ま せる 銀色 の 愛 俺 の 全て
Label: 椎名林檎


Precision: 0.6009852216748769
Recall: 0.6009852216748769
F-score: 0.6009852216748769
Tokenized: どうして こんなに 私 なんて 言え ない よ ウニ つい ちゃっ て 涙し たり ひと 吹きで やわ な 双子 の 今年 の こと ばかり で
Label: 米津玄師


Precision: 0.6133004926108374
Recall: 0.6133004926108374
F-score: 0.6133004926108374
Tokenized: あなた の 心 を 呼び リボン 海 に ゆく の 下 過ぎ て 夢 に 私 だけ 生ま れ て 外 が ある あなた は 見
Label: 中島みゆき


Precision: 0.6108374384236454 Recall: 0.6108374384236454
F-score: 0.6108374384236454
Tokenized: 伝え たい 熱い 恋 が 素知らぬ 振り ゃないくっついちゃってもいいんじゃな
い 僕 の 為 俺 も 幻 に 繋がる くらい 君 を 見 て そう な の は Label: 嵐


Precision: 0.6280788177339901
Recall: 0.6280788177339901
F-score: 0.6280788177339901
Tokenized: 愛の数まで咲き乱れて震えているのよ背中が切ないう早くも うち 楽しい 笑い声 流し て しまう の 待っ
Label: 中島みゆき


Precision: 0.6403940886699507
Recall: 0.6403940886699507
F-score: 0.6403940886699507
Tokenized: ジンジャーで恐怖の山の畑の自分を異人さん、お焦げてやり 抜く いく あの 娘 不 の 頭
Label: 椎名林檎


特に多く挙げられたアーティストは、全 16 フレーズ中、 
* 米津玄師(3フレーズ)
* 椎名林檎(4フレーズ)
* 中島みゆき(3フレーズ) という結果になりました。

米津玄師さんの歌詞ををダイレクトに生成するには程遠い結果となりましたが、この3人の 世界観には共通点も多く、一定の成果は出せたと思われます(実際、米津さんは過去のイン タビューで、中島みゆきさんや松任谷由美さんの影響を受けたと語っています)。4、で挙げ た課題を試すことで、さらなる精度向上を行っていきたいと考えています。
