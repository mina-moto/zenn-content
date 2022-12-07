---
title: "文章の前処理・分かち書きを行い文書ベクトルを算出"
emoji: "😀"
type: "tech"
topics: [Python,mecab,自然言語処理,doc2vec]
published: true
---
この記事は[「Sunrise Advent Calendar 2019」](https://adventar.org/calendars/4184)10日目の記事です．
最近，諸事情により自然言語処理を完全に理解する必要があり，文章の前処理やベクトルの算出とかを行ったので，その辺りの方法について簡単にまとめます．

# 必要ライブラリインポート

```python
import re
from gensim.models.doc2vec import Doc2Vec, TaggedDocument
import string
import unicodedata
import MeCab
```

# 文書の前処理

対象とする文章

```py
text = 'Yahoo Japanなどを\r運営するヤフーの親会社Zホールディングス（以下ZHD）とLINEは11月18日、経営統合することで基本合意したことを正式に発表した。\n11月13日に日本経済新聞などが両社の合併を報じていた。'
```

### 改行コードなど空白で置換

\n,\rなど

```py
' '.join(text.splitlines())
```

結果

```
'Yahoo Japanなどを 運営するヤフーの親会社Zホールディングス（以下ZHD）とLINEは11月18日、経営統合することで基本合意したことを正式に発表した。 11月13日に日本経済新聞などが両社の合併を報じていた。'
```

### 数字を0で置換

```py
re.sub(r'[0-9]+', "0", text)
```

結果

```
'Yahoo Japanなどを\r運営するヤフーの親会社Zホールディングス（以下ZHD）とLINEは0月0日、経営統合することで基本合意したことを正式に発表した。\n0月0日に日本経済新聞などが両社の合併を報じていた。'
```

### アルファベット削除

普通はあまりしないと思います．

```py
re.sub(r'[A-z]+', "", text)
```

```
' などを\r運営するヤフーの親会社ホールディングス（以下）とは11月18日、経営統合することで基本合意したことを正式に発表した。\n11月13日に日本経済新聞などが両社の合併を報じていた。'
```

### 記号文字削除

全角記号を一度半角に変えてから削除．

```py
unicodedata.normalize("NFKC", text).translate(str.maketrans("", "", string.punctuation  + "「」、。・"))
```

```
'Yahoo Japanなどを\r運営するヤフーの親会社Zホールディングス以下ZHDとLINEは11月18日経営統合することで基本合意したことを正式に発表した\n11月13日に日本経済新聞などが両社の合併を報じていた'
```

## 前処理まとめ

```py
def preprocess(text):
    new=' '.join(text.splitlines())
    new=re.sub(r'[0-9]+', "0", new)
    new=re.sub(r'[A-z]+', "", new)
    new=unicodedata.normalize("NFKC", new).translate(str.maketrans("", "", string.punctuation  + "「」、。・"))
    return new
```

結果

```
' などを 運営するヤフーの親会社ホールディングス以下とは0月0日経営統合することで基本合意したことを正式に発表した 0月0日に日本経済新聞などが両社の合併を報じていた'
```

# 分かち書き

Mecabにより形態素解析を行いリストにする．下記は動詞，形容詞，名詞のみを残す例．辞書には[mecab-ipadic-NEologd] (<https://github.com/neologd/mecab-ipadic-neologd/blob/master/README.ja.md)が用いられることが多いが今回は使っていない．Mecab>の呼び出しを何度も行うと重いのでメソッド外に出している．

```py
mecab = MeCab.Tagger("-Ochasen")
def split_into_words(text):
    lines = mecab.parse(text).splitlines()
    words = []
    for line in lines:
        chunks = line.split('\t')
        if len(chunks) > 3 and (chunks[3].startswith('動詞')
                                or chunks[3].startswith('形容詞')
                                or (chunks[3].startswith('名詞')
                                    )):

            word = chunks[0]
            words.append(word)
    return words
```

preprocess後のテキストの結果

```py
prepreprocess_text=preprocess(text)
split_into_words(prepreprocess_text)
```

```
['運営', 'する', 'ヤフー', '親会社', 'ホールディングス', '以下', '0', '月', '0', '日', '経営', '統合', 'する', 'こと', '基本', '合意', 'し', 'こと', '正式', '発表', 'し', '0', '月', '0', '日', '日本経済新聞', '両社', '合併', '報じ', 'い']
```

## 分かち書き後の処理

### ひらがな1文字の単語削除

```py
[w for w in words if re.compile('[\u3041-\u309F]').fullmatch(w) == None]
```

```
['運営', 'する', 'ヤフー', '親会社', 'ホールディングス', '以下', '0', '月', '0', '日', '経営', '統合', 'する', 'こと', '基本', '合意', 'こと', '正式', '発表', '0', '月', '0', '日', '日本経済新聞', '両社', '合併', '報じ']
```

### 特定の単語を削除

主に[SlothLib](http://svn.sourceforge.jp/svnroot/slothlib/CSharp/Version1/SlothLib/NLP/Filter/StopWord/word/Japanese.txt)のようなストップワードリストを利用して削除する．

```py
delwords=["月","日"]
[w for w in words if not(w in delwords)]
```

結果

```
['運営', 'する', 'ヤフー', '親会社', 'ホールディングス', '以下', '0', '0', '経営', '統合', 'する', 'こと', '基本', '合意', 'し', 'こと', '正式', '発表', 'し', '0', '0', '日本経済新聞', '両社', '合併', '報じ', 'い']
```

## 前処理・分かち書きまとめ

```py

def delete_words(words):
    # ひらがな1文字削除
    newWords = [w for w in words if re.compile('[\u3041-\u309F]').fullmatch(word) == None]
    # カタカナ1文字削除
    newWords = [w for w in newWords if re.compile('[\u30A1-\u30FF]').fullmatch(word) == None]
    # 特定の単語削除
    delwords=["月","日"]
    newWords = [w for w in newWords if not(w in delwords)]
    return newWords
```

```py
def generate_dataset(text):
    preprocess_text=preprocess(text)
    split_words=split_into_words(preprocess_text)
    return delete_words(words)
```

結果

```
['運営', 'する', 'ヤフー', '親会社', 'ホールディングス', '以下', '0', '0', '経営', '統合', 'する', 'こと', '基本', '合意', 'こと', '正式', '発表', '0', '0', '日本経済新聞', '両社', '合併', '報じ']
```

# 文書ベクトルの算出

対象とする文章のリストを用意（普通はtxtファイルから読み込むなどする）．

```py
text_list=[]
text_list.append('Yahoo Japanなどを\r運営するヤフーの親会社Zホールディングス（以下ZHD）とLINEは11月18日、経営統合することで基本合意したことを正式に発表した。\n11月13日に日本経済新聞などが両社の合併を報じていた。')
text_list.append('「パン工場」に住むパン作りの名人・ジャムおじさん。彼は“心を持ったあんパン”を作りたいと思っていたが上手くいかずに困っていた。ある夜、夜空の流れ星がパン工場のパン焼き窯に降り注ぐ。この「いのちの星」があんパンに宿り、アンパンマンが誕生したのだった。アンパンマンは、困っている人がいればどこへでも飛んで行き、お腹を空かせて泣いている人には自分の顔を食べさせてくれる正義のヒーロー。そんなアンパンマンをやっつけるために誕生したのが、「バイキン星」からやって来たばいきんまんであった。')
text_list.append('のび太がお正月をのんびりと過ごしていると、突然、どこからともなくのび太の未来を告げる声が聞こえ、机の引出しの中からドラえもんと、のび太の孫の孫のセワシが現れた。セワシ曰く、のび太は社会に出た後も沢山の不運に見舞われ、会社の倒産が原因で残った莫大な借金によって子孫を困らせているという。そんな悲惨な未来を変えるために、ドラえもんを子守用ロボットとしてのび太のもとへと連れてきたのだった。')
```

### Doc2Vecによる文章の学習

Doc2Vec自体やパラメータの説明は[gensimの公式ドキュメント](https://radimrehurek.com/gensim/models/doc2vec.html)や[論文](https://arxiv.org/abs/1405.4053)を参照ください．

```py
def train(text_list):
    dataset=list(map(generate_dataset,text_list))
    trainings = [TaggedDocument(doc, [i])
                 for i, doc in enumerate(dataset)]
    model = Doc2Vec(documents=trainings)
    return model
```

```py
model=train(text_list)
```

### モデルのセーブ・ロード

```py
model.save('doc2vec.model')
```

```py
model=Doc2Vec.load('doc2vec.model')
```

### 文書ベクトル取得

学習した文書のベクトル取得(text_listの先頭の文書のベクトル)．デフォルトパラメータで学習した場合100次元．

```py
model.docvecs[0]
```

```
array([ 2.5095379e-03, -1.2622289e-04, -1.1823229e-03,  3.3415016e-04,
       -1.9223742e-03, -2.3717529e-03, -8.4026062e-05, -1.4152353e-03,
       -3.4429017e-03,  1.3178275e-03,  1.0290239e-03,  2.2883087e-03,
       -3.0558954e-03,  2.5106908e-03, -4.1721470e-04, -1.0859852e-03,
       -2.3893034e-04, -3.7497834e-03,  3.6848460e-03,  1.9433234e-04,
       -7.2101870e-04, -2.1014472e-03,  1.6300754e-03, -8.1060972e-04,
       -3.0647446e-03, -3.6500772e-03, -3.4705214e-03,  4.7291894e-03,
       -3.9071091e-03, -4.6025636e-03,  1.2165984e-03, -3.8299439e-03,
       -4.2568599e-03, -2.1912481e-03, -2.5559247e-03, -3.9934221e-04,
       -1.0293393e-03, -2.3890976e-03, -2.5162895e-03,  4.3413509e-03,
        2.7214717e-03,  3.7324268e-03,  1.4758985e-03, -6.9151312e-05,
        3.8205485e-03, -2.1535682e-03, -7.4021268e-04, -2.9556274e-03,
       -7.1587786e-04, -6.9994084e-04,  1.2541209e-04, -4.8115538e-03,
        1.8687663e-03, -4.1160840e-03,  2.3677868e-03,  4.4390149e-03,
       -5.5469677e-04, -8.6610392e-04,  2.3333214e-03,  4.2179450e-03,
        3.2901503e-03, -1.1678069e-03, -5.5523252e-04, -6.5870601e-04,
        3.9052053e-03, -2.2258856e-03,  4.2966665e-03,  4.3981723e-03,
       -2.8700563e-03,  1.1474759e-03, -4.1666413e-03, -1.2336340e-03,
       -9.0547075e-04, -4.2380292e-05,  3.4137981e-03, -1.3768630e-03,
        4.9160537e-03,  3.1170491e-03,  2.6434229e-03,  3.4214210e-04,
        3.4710045e-03,  2.6036222e-03,  4.2710584e-03, -3.6335504e-04,
        2.7531444e-03, -2.1464955e-03,  1.7816224e-03,  4.7278586e-03,
       -1.5395511e-03, -9.3266240e-04, -3.1087976e-03, -1.7828557e-03,
       -4.7502802e-03,  4.7927028e-03, -4.5370241e-03, -3.7872749e-03,
        1.6869572e-03,  1.6862437e-03,  3.3439032e-03, -3.0570282e-03],
      dtype=float32)
```

## 未学習の文章のベクトルを計算

```py
new_text='就職情報サイト『リクナビ』を運営するリクルートキャリアの閲覧履歴にもとづく、就職活動中の学生の「内定辞退率」の予測を企業に販売していた問題で、トヨタ自動車や本田技術研究所など契約先の37社が、個人情報保護法による行政指導の処分を受けたという。'
```

文章を分かち書きしてメソッド呼び出し．

```python
new_words=generate_dataset(new_text)
model.infer_vector(new_words)
```

## 2つの文書の類似度算出

```py
words=dataset[0]
model.docvecs.similarity_unseen_docs(model, words, new_words)
```

```
0.14044614
```
