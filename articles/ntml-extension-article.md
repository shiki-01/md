---
title: "自作のHTML拡張言語「NTML」をVSCodeの拡張機能として作った話" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["html","javascript","vscode"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---
:::message
この記事に書かれている内容について、正確性や安全性などは一歳保証しません。
あくまでも、私の場合はこうしたよっていうメモ感覚で読んでください。
:::

# はじめに
ほぼ初めて記事を書くので、言葉足らずだったり、わかりにくかったらすみません。
蛇足いっぱいでお届けするので、必要無いとこは飛ばしてもらって構いません。

### 使用環境
- MacBook Air(M1,2020) (OS：Sonoma 14.0)
- VSCode for MacOS (var.1.83.0)
- 頭脳：プログラム初心者〜中級者

# 拡張（メタ）言語について
### 拡張言語とは
有名なのはSass（Scss）とかです。
あまり良い記事がなかったので、私なりに軽くまとめてみます。

>プログラミング言語において拡張（メタ）言語とは、
>あるプログラミング言語に対して、**見た目的・あるいは機能的に様々な機能を付け加えて、使い勝手を向上**させた言語。
>多くの場合、コンパイラすることで、拡張言語から実際の言語に書き換えて使う。

### 拡張言語の例
| 言語名 | 拡張言語名 |
|:-:|:-:|
| HTML  | Haml  |
|  CSS |  Sass(Scss) |
|  JavaScript |  CoffeeScript |

例えばSassでは以下のように、変数が使えるようになったり、入れ子構造を簡単に作ることができます。詳しい解説は他の記事を見てください。

```scss:css -> scss
//CSS
body {
    color: #EEE;
    margin: 20px;
}
body .main {
    color: #000;
    font-size:200px;
    margin-top: 3em;
}
body .main:before {
    color: #EEE;
}
//↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
//Scss
$color: #EEE;   //変数が使える
body {
    color: #EEE;
    margin: 20px;

    .main {   //入れ子構造が簡単に作れる
        color: #000;
        font-size:200px;
        margin-top: 3em;

        &:before {    //&で入れ子の中に:beforeをつけれる
            color: $(color);
        }
    }
}
```
# NTMLについて

　NTML（ **N**ihongo Hyper**T**ext **M**arkup **L**anguage ）とは、HTMLの拡張言語です。
　この言語では`body`や`div`などのHTMLにおける「**タグ**」を、`本体`や`コンテナ`などのように、日本語で書くことができます。
　主に、教育目的に作成したプログラミング言語で「[scratch](https://scratch.mit.edu)」などの「**ビジュアルプログラミング言語**」と、実際のプログラミング言語の中間にあたるものが作りたくて作成しました。

:::message
正確にいうと、HTMLはマークアップ言語なので、プログラミング言語ではないのですが、まあご愛嬌だと思ってスルーしてください。
:::

```plantuml
ビジュアルプログラミング言語 -> NTML :ブロック表記をコード（日本語）にする
NTML -> HTML :日本語を英語（？）のタグに戻す
```

### 仕組み

主に使う仕組みは「字句解析」と「要素変換」によるコンパイラです。
この二つについても軽くまとめますね。

#### 字句解析
>字句解析とは、ある言語で書かれた文について、文字の並びを解析し、言語的に意味のある最小の単位（トークン）に分解する処理のこと。自然言語処理やプログラミング言語のコンパイルなどで行われる。

https://e-words.jp/w/%E5%AD%97%E5%8F%A5%E8%A7%A3%E6%9E%90.html#:~:text=%E5%AD%97%E5%8F%A5%E8%A7%A3%E6%9E%90%20%E3%80%90lexical%20analysis%E3%80%91,%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%AB%E3%81%AA%E3%81%A9%E3%81%A7%E8%A1%8C%E3%82%8F%E3%82%8C%E3%82%8B%E3%80%82

例えば、`<div class="main">hello!</div>`というコードがあった時、これらには以下のような要素があります。

| タグ名 | クラス名 | タグ内要素 | エンドタグ |
|:-:|:-:|:-:|:-:|
| `div`  |  `main` |  `hello!` | あり（ `/div` ）  |

要は、今ここに書き出した作業が「字句解析」です。
一文字ずつ「この文字はどれのことを言っているんだ？」というのを解析していくのが字句解析です。

#### 要素変換

実際のところの名前は知りません。HTMLにおけるJavaScriptの使い方の一つとして、要素の書き換えというものがあります。ページ数やデータ量の削減のために、同じような内容で書かれているものは、オブジェクトを作ってその要素ごとに何種類か用意した上で、それを書き換えることでコンテンツを大量に量産すると言った具合です。~~自分でも書きながら何言ってんだこいつってなってるので、ここら辺は無視でもいいです。~~

今回は、「`tagMAP`」というオブジェクトを用意して、その中に日本語のタグと、それに対応する実際のタグを入れておくことで、字句解析によって見つけた日本語のタグを、`tagMAP`で検索することで書き換えることができます。

![イメージ図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3526339/d62b6987-e992-be20-e678-d2e98eae85cd.png)

### 実装

ここからは、実際のコードを解説しながら、NTMLについてみていきましょう。実装するために使った言語は「JavaScript」です。

```javascript:JavaScript
const tagMap = {
    '見出し': 'h1',
	'強調': 'strong'
};
const attrMap = {
	'クラス': 'class',
	'ソース': 'src'
};

let output = input.replace(/<\/?([^>]+)>/g, function (match, tag) {
	let isClosingTag = match.startsWith('</');
	let [tagName, ...attrs] = tag.split(' ');
	tagName = tagMap[tagName] || tagName;
	attrs = attrs.map(attr => {
	let [attrName, attrValue] = attr.split('=');
	attrName = attrMap[attrName] || attrName;
	return `${attrName}=${attrValue}`;
});
    return `<${isClosingTag ? '/' : ''}${[tagName, ...attrs].join(' ')}>`;
});

//入力 -> <見出し クラス="main">これは見出し<強調>ここは強調</強調></見出し>
//出力 -> <h1 class="main">これは見出し<strong>ここは強調</strong></h1>

//入出力に関しては、適当に関数でも作ってやってください。
```

まずは全文です。１つづつ見ていきましょう。

```javascript:JavaScript
const tagMap = {
    '見出し': 'h1',
	'強調': 'strong'
};
const attrMap = {
	'クラス': 'class',
	'ソース': 'src'
};
```

これは、JavaScriptの「オブジェクト」と呼ばれるものです。
~~こんなもの見る人なら~~おそらく知っているとは思いますが、一応簡単に解説だけしておきましょう。

#### オブジェクト

オブジェクトとは、プロパティの集まりで、それぞれのプロパティに様々な値を代入しておくことで、データをきれいにまとめ、簡単に取り出すことができるものです。

今回でいくと、`tagMap`というオブジェクトには、`見出し`と`強調`というプロパティが入っています。また、`見出し`には`h1`が、`強調`には`strong`という値が代入されています。なので、このオブジェクトに対して`見出し`というプロパティを検索すると、`h1`という値が返ってきます。

本来は、HTMLを記述する際に要素ごと書き換えるときや、ユーザーデータなどを一つのところにまとめておくといって場合に使うのですが、プロパティの機能を使いたかったので今回はこれで実装しました。

詳しくはこちらの記事など、ご自身で調べてください。

https://qiita.com/akari_0618/items/5a9e9d565b47c5de2e4d

```javascript:JavaScript
let output = input.replace(/<\/?([^>]+)>/g, function (match, tag) {
	let isClosingTag = match.startsWith('</');
	let [tagName, ...attrs] = tag.split(' ');
```
一言で言うと、これが字句解析です。
`/<\/?([^>]+)>/g`が、字句解析のための正規表現です。何言ってるかわからないと思いますが、私もわかりません。要は「< tagName >」が開始タグだよっていうものです。

正規表現についてはこちらなどもどうぞ

https://qiita.com/iLLviA/items/b6bf680cd2408edd050f

おすすめの正規表現チェッカー

https://regex101.com

```javascript:JavaScript
tagName = tagMap[tagName] || tagName;
```
タグの名前を英語に変換しています。
`tagMap`に、対応するタグがある場合は変換し、そうでない場合はそのままにします。

```javascript:JavaScript
let [attrName, attrValue] = attr.split('=');
	attrName = attrMap[attrName] || attrName;
	return `${attrName}=${attrValue}`;
```

ここでは、タグ内の要素（classなど）を変換しています。
また、プロパティと値で分けるために、一度「`=`」の前後で区切って、手前をプロパティとして変換、後ろをプロパティの値として保存、最終的にまとめて一つの値とする。と言った感じです。