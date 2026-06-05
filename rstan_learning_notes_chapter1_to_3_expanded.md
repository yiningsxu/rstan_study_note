---
title: "RStan学習ノート：Chapter 1〜3（Chapter 3詳細版）"
author: ""
date: "2026-06-05"
lang: ja
---

# RStan学習ノート：Chapter 1〜3

本ノートは、RStan公式リファレンス、Stan Reference Manual、Stan User's Guideを確認した上で、RStanを用いた統計モデリングの導入から基本的な回帰モデルの実装までをまとめたものです。Chapter 3では、感染症分野で実際に頻出するデータ構造を意識したサンプルデータとRStanコードを提示します。サンプルデータは再現可能なシミュレーションデータであり、実患者データではありません。今回の詳細版では、回帰モデルの数式の仕組み、出力結果の実務的解釈、Stanコードのブロック別意図を追加しています。

参照した主な公式資料は、[RStan Reference](https://mc-stan.org/rstan/reference/rstan.html)、[RStan `stan()` Reference](https://mc-stan.org/rstan/reference/stan.html)、[Stan Reference Manual](https://mc-stan.org/docs/reference-manual/)、[Stan User's Guide](https://mc-stan.org/docs/stan-users-guide/)、[Stan Diagnostics and Warnings](https://mc-stan.org/learn-stan/diagnostics-warnings.html)です。

## 目次

- [Chapter 1　統計モデリングとStanの概要](#chapter-1統計モデリングとstanの概要)
  - [1.1　統計モデリングとは](#11統計モデリングとは)
  - [1.2　統計モデリングの目的](#12統計モデリングの目的)
  - [1.3　確率的プログラミング言語](#13確率的プログラミング言語)
  - [1.4　なぜStanなのか？](#14なぜstanなのか)
  - [1.5　なぜRStanなのか？](#15なぜrstanなのか)
  - [補足と文献案内](#chapter-1補足と文献案内)
- [Chapter 2　StanとRStanをはじめよう](#chapter-2stanとrstanをはじめよう)
  - [4.1　StanとRStanの準備](#41stanとrstanの準備)
  - [4.2　Stanの基本的な文法](#42stanの基本的な文法)
  - [補足と文献案内](#chapter-2補足と文献案内)
- [Chapter 3　基本的な回帰とモデルのチェック](#chapter-3基本的な回帰とモデルのチェック)
  - [5.1　単回帰](#51単回帰)
  - [5.2　重回帰](#52重回帰)
  - [5.3　二項ロジスティック回帰](#53二項ロジスティック回帰)
  - [5.4　ロジスティック回帰](#54ロジスティック回帰)
  - [5.5　ポアソン回帰](#55ポアソン回帰)
  - [補足と文献案内](#chapter-3補足と文献案内)

---

# Chapter 1　統計モデリングとStanの概要

## この章の到達目標

この章の目的は、RStanの使い方に入る前に、そもそも**統計モデリングとは何か**、**Stanは何をする言語なのか**、**RStanはStanをRからどう使えるようにするのか**を整理することです。

Stan Reference Manualは、Stanを「確率モデルをコード化するためのプログラミング言語、モデルを当てはめ予測するための推論アルゴリズム、結果を評価するための事後解析ツール」を扱う体系として位置づけています。RStanは、そのStanをRから使うためのインターフェースです。

この章では、次の考え方を順に確認します。

1. 統計モデルとは、観測データを確率的に説明するための仮定体系である。
2. 統計モデリングの目的は、推定だけでなく、予測、説明、不確実性評価、モデルチェック、意思決定支援まで含む。
3. Stanは、統計モデルをプログラムとして書き、その対数密度に基づいて推論する確率的プログラミング言語である。
4. RStanは、StanモデルをRからコンパイル・実行・要約するためのインターフェースである。

## 1.1　統計モデリングとは

### 1.1.1　統計モデルの基本的な考え方

**統計モデリング**とは、観測されたデータがどのような確率的仕組みによって生じたのかを、数式やプログラムで表現する作業です。たとえば、身長、売上、クリック率、故障件数、感染者数、顧客離脱、試験点数などのデータに対して、次のような問いを扱います。

- このデータはどのような確率分布から発生したと考えるのが妥当か。
- どの変数が結果に影響しているのか。
- 未知の量をどの程度の不確実性で推定できるのか。
- まだ観測されていない将来の値をどの程度予測できるのか。

統計モデルでは、通常、次の3つを区別します。

**観測データ**  
すでに観測された値です。Stanでは主に`data`ブロックに入ります。たとえば、観測数`N`、応答変数`y`、説明変数`x`などです。

**未知パラメータ**  
データから推定したい量です。平均、分散、回帰係数、階層モデルの群間ばらつき、潜在変数などが含まれます。Stanでは主に`parameters`ブロックに書きます。

**確率構造**  
データとパラメータの関係を確率分布で書いたものです。たとえば、`y ~ normal(mu, sigma)`は「観測値`y`は平均`mu`、標準偏差`sigma`の正規分布に従う」という仮定を表します。ただしStanの`~`記法は、乱数を発生させる命令ではなく、モデルの対数密度に項を追加するための記法です。

### 1.1.2　統計モデルは「真実」ではなく「仮定の明示」

統計モデルは現実そのものではなく、現実を分析可能な形に簡略化した**仮定の集合**です。モデルは、現実の全要素を完全に再現するものではありません。むしろ、分析目的に照らして重要な構造を取り出し、不確実性を含めて表現するための道具です。

たとえば、正規分布モデル

```stan
y ~ normal(mu, sigma);
```

は、「データのばらつきは正規分布で近似できる」という仮定を置いています。この仮定が妥当かどうかは、推定後に残差、事後予測、モデル比較、外部データでの予測性能などを通じて検討します。

Stan User's Guideには、回帰モデル、時系列モデル、欠測データ、打ち切り・切断データ、混合モデル、測定誤差、ガウス過程、ODE、サバイバルモデルなど、多様なモデル例が含まれています。これは、統計モデリングが単なる「回帰分析」だけではなく、データ生成過程を明示的に記述する広い枠組みであることを示しています。

### 1.1.3　ベイズ統計における統計モデル

Stanはベイズ推論に強いツールです。ベイズ統計では、未知パラメータを固定された未知量として扱いつつ、その不確実性を確率分布で表します。基本形は次のようになります。

\[
p(\theta \mid y) \propto p(y \mid \theta)p(\theta)
\]

ここで、\(\theta\)は未知パラメータ、\(y\)は観測データです。

**尤度 \(p(y \mid \theta)\)** は、パラメータが与えられたときに観測データがどれくらい起こりやすいかを表します。

**事前分布 \(p(\theta)\)** は、データを見る前にパラメータについて持っている知識や制約を表します。

**事後分布 \(p(\theta \mid y)\)** は、データを観測した後のパラメータの不確実性を表します。

Stan言語は、データで条件づけられた対数密度関数を定義します。ベイズモデルでは、この対数密度は典型的には事後分布に比例する非正規化対数密度です。

### 1.1.4　最小例：正規分布モデル

観測値`y`が正規分布に従うと仮定し、平均`mu`と標準偏差`sigma`を推定するモデルは、Stanでは次のように書けます。

```stan
data {
  int<lower=1> N;
  vector[N] y;
}

parameters {
  real mu;
  real<lower=0> sigma;
}

model {
  mu ~ normal(0, 10);
  sigma ~ exponential(1);
  y ~ normal(mu, sigma);
}
```

このモデルでは、`data`ブロックがRから渡される観測データ、`parameters`ブロックが推定対象、`model`ブロックが事前分布と尤度を定義しています。

## 1.2　統計モデリングの目的

### 1.2.1　目的1：未知量を推定する

統計モデリングの第一の目的は、観測データから未知の量を推定することです。たとえば、商品の平均購入金額、広告効果、薬剤効果、地域ごとのリスク、顧客の離脱確率、機械の故障率などです。

頻度論的な推定では点推定値や信頼区間を使うことが多いですが、ベイズモデリングでは未知量の**事後分布**を直接扱います。これにより、「平均効果はどの程度か」だけでなく、「その効果が正である確率はどれくらいか」「実務上意味のある大きさを超える確率はどれくらいか」といった問いを表現しやすくなります。

### 1.2.2　目的2：不確実性を定量化する

統計モデリングでは、推定値そのものよりも、推定の不確実性が重要になることがあります。たとえば、2つの施策AとBの平均効果が近い場合、単に平均値だけを比較しても意思決定には不十分です。どの程度の確信を持ってAがBより良いと言えるのか、効果のばらつきはどれくらいか、最悪の場合にどの程度悪化しうるかを評価する必要があります。

StanのMCMC出力では、各パラメータについて多数の事後ドローが得られます。RStanの`stanfit`オブジェクトは、Stanモデルのフィット結果を保持し、`print`、`plot`、`summary`などのメソッドで要約や診断情報を取り出せます。

### 1.2.3　目的3：予測する

統計モデルは、まだ観測されていないデータの予測にも使います。ベイズモデルでは、パラメータの事後分布を通じて、予測値の不確実性も同時に扱えます。

Stan User's Guideでは、事後予測分布を「過去の観測データが与えられたときの新しい観測値の分布」として説明し、`generated quantities`ブロックを使って事後予測量を生成する方法を示しています。

```stan
generated quantities {
  real y_new;
  y_new = normal_rng(mu, sigma);
}
```

この`y_new`の分布を見ることで、「次に観測される値はどの範囲に入りそうか」を評価できます。

### 1.2.4　目的4：モデルの妥当性を確認する

統計モデルは作って終わりではありません。モデルがデータの重要な特徴を再現できているかを確認する必要があります。Stan User's Guideでは、事後予測チェックを、フィット済みモデルのパラメータから複製データをシミュレーションし、その複製データと元データの統計量を比較する方法として説明しています。

たとえば、モデルが平均はよく再現しているが外れ値の頻度を過小評価している場合、そのモデルは平均値の推定には使えても、リスク評価や異常検知には不適切かもしれません。統計モデリングでは、モデルの仮定、推定結果、予測性能、診断結果を総合的に見る必要があります。

### 1.2.5　目的5：意思決定を支援する

統計モデリングの最終目的は、多くの場合、意思決定です。意思決定には、単なる推定値ではなく、不確実性、リスク、コスト、便益、制約条件が関係します。たとえば、広告予算配分、医療介入、在庫管理、価格設定、品質管理などでは、「期待値が最大の選択肢」だけでなく、「失敗時の損失が許容範囲か」も考える必要があります。

Stan User's Guideには、事後予測サンプリング、シミュレーションベースのキャリブレーション、事後・事前予測チェック、ホールドアウト評価、クロスバリデーション、意思決定分析などが含まれています。

## 1.3　確率的プログラミング言語

### 1.3.1　確率的プログラミング言語とは

**確率的プログラミング言語**とは、確率モデルをプログラムとして記述し、そのモデルに対して推論アルゴリズムを適用するための言語です。通常のプログラムが「入力から出力を計算する手続き」を書くのに対して、確率的プログラムでは「データがどのような確率構造から発生したか」を書きます。

Stanの場合、ユーザーはモデルをStan言語で書きます。Stan言語は、確率モデルをコード化するためのプログラミング言語であり、データに条件づけられた対数密度関数を定義します。

### 1.3.2　モデル記述と推論を分離する

確率的プログラミングの大きな利点は、**モデル記述**と**推論アルゴリズム**をある程度分離できることです。ユーザーは、まずモデルを次のように書きます。

```stan
data {
  int<lower=1> N;
  vector[N] y;
}

parameters {
  real mu;
  real<lower=0> sigma;
}

model {
  mu ~ normal(0, 10);
  sigma ~ exponential(1);
  y ~ normal(mu, sigma);
}
```

その後、Stanがこのモデルから対数密度と勾配を計算し、NUTS/HMCなどの推論アルゴリズムを用いてパラメータの事後分布を探索します。Stan Reference Manualでは、Stanで使われるMCMCアルゴリズムとしてHamiltonian Monte Carlo、HMC、およびその適応版であるNo-U-Turn Sampler、NUTSが説明されています。

### 1.3.3　Stanプログラムの基本ブロック

Stanプログラムは、役割ごとにブロックを分けて書きます。主要ブロックは次の通りです。

```stan
functions {
  // ユーザー定義関数
}

data {
  // 外部から渡されるデータ
}

transformed data {
  // データだけから計算できる定数や変換
}

parameters {
  // 推定対象の未知パラメータ
}

transformed parameters {
  // パラメータから導かれる保存対象の量
}

model {
  // 事前分布と尤度、つまり対数密度
}

generated quantities {
  // 事後予測、派生量、乱数生成
}
```

Stan公式リファレンスでは、`data`ブロックはモデルに必要なデータ、`parameters`ブロックはモデルのパラメータ、`model`ブロックは対数確率関数、`generated quantities`ブロックはパラメータやデータに基づく派生量や乱数生成を扱う場所として説明されています。

### 1.3.4　Stanは「サンプリング文」を実行しているわけではない

Stanでは、次のような記法を使います。

```stan
y ~ normal(mu, sigma);
```

一見すると「`y`を正規分布からサンプリングする」ように見えますが、`model`ブロック内ではそうではありません。この記法は、モデルの対数密度に正規分布の対数確率を加えるための簡略記法です。

```stan
target += normal_lpdf(y | mu, sigma);
```

これはStanを理解する上で非常に重要です。Stanは、モデル中の分布文を使って**非正規化事後密度**を構成し、その密度に比例する分布からMCMCでサンプルを生成します。

### 1.3.5　Stanで直接扱いにくいもの

Stanは連続パラメータの推論に強く設計されています。一方で、Stanは離散パラメータを直接サンプリングしません。Stan User's Guideでは、Stanは離散パラメータのサンプリングをサポートせず、離散潜在変数を含む多くのモデルでは周辺化によってStanで書ける形にする、と説明されています。

たとえば、変化点モデル、潜在クラスモデル、HMM、混合モデルなどでは、離散状態をそのまま`parameters`ブロックに置くのではなく、可能な場合は`log_sum_exp`などを使って離散状態を周辺化します。この点は、BUGS/JAGS系のモデル表現とStanの大きな違いです。

## 1.4　なぜStanなのか？

### 1.4.1　理由1：柔軟なベイズモデリングができる

Stanは、単純な線形回帰から、階層モデル、時系列モデル、欠測データモデル、測定誤差モデル、混合モデル、ガウス過程、ODE、サバイバルモデルまで、幅広い統計モデルを書けます。

これは、既製の関数にモデルを合わせるのではなく、分析対象に合わせてモデルを設計できるという意味です。たとえば、次のような状況ではStanの柔軟性が重要になります。

- 観測誤差と過程誤差を分けたい。
- 個人差、店舗差、地域差を階層構造で表現したい。
- 欠測値を単に削除せず、モデル内で扱いたい。
- 標準的な回帰では表現しにくい独自の尤度を使いたい。
- 事後予測や意思決定に必要な派生量をモデル内で生成したい。

### 1.4.2　理由2：NUTS/HMCによる効率的なサンプリング

Stanの中心的なMCMC手法はHMCとNUTSです。HMCは、サンプリング対象の密度関数の微分情報を使って、事後分布内を効率よく移動するMCMC手法です。

NUTSは、HMCの調整を自動化する適応的なバリアントです。RStan公式ページでも、RStanはNUTS、つまりHMCの一種であるNo-U-Turn Samplerによるフルベイズ推論を提供すると説明されています。

実務的には、StanのNUTS/HMCは、パラメータ間に強い相関があるモデルや高次元の連続パラメータを含むモデルで有用です。ただし、すべてのモデルが自動的にうまくいくわけではありません。モデルの識別性が弱い、事前分布が不適切、パラメータ化が悪い、事後分布の幾何が複雑な場合には、divergent transitions、R-hat悪化、ESS低下などの診断問題が出ます。

### 1.4.3　理由3：診断が重視されている

Stanを使う利点の一つは、MCMCの結果に対する診断情報が重視されていることです。RStanやStanでは、推定値の平均や信用区間だけでなく、R-hat、ESS、divergence、treedepthなどを見る必要があります。

Stan公式の診断ガイドでは、R-hatはチェーン間・チェーン内の推定量を比較する収束診断であり、最終的には少なくとも4チェーンを走らせ、一般にはR-hatが1.01未満の場合にサンプルを十分信頼するという目安が示されています。また、bulk-ESSとtail-ESSは分布の中心部と裾の推定効率を評価するための指標です。

### 1.4.4　理由4：モデルを明示的に書ける

Stanでは、モデルの仮定をコードとして明示します。これは、便利な一方で、責任も伴います。`brms`や`rstanarm`のような高水準インターフェースでは、Rの数式構文に近い形でモデルを指定できますが、Stanを直接書く場合は、尤度、事前分布、パラメータ制約、生成量を自分で設計します。

Stanを直接使うと、次のような利点があります。

- モデルの仮定がコード上で明確になる。
- 標準パッケージにない尤度や構造を書ける。
- 事前分布を明示的に設計できる。
- 事後予測や派生量を`generated quantities`で管理できる。
- モデルの問題点を診断しやすくなる。

### 1.4.5　理由5：R、Python、Julia、Shellなど複数環境から使える

Stanは単独のRパッケージではなく、複数言語から利用できる統計モデリング基盤です。Rから使う場合には、RStanとCmdStanRが主な選択肢になります。Stan Toolkitでは、CmdStanRはRユーザー向けの推奨インターフェース、RStanはRインターフェースの一つとして掲載されています。

## 1.5　なぜRStanなのか？

### 1.5.1　RStanはStanをRから使うためのインターフェース

**RStan**は、StanをRから使うためのインターフェースです。パッケージ名は小文字で`rstan`です。RStan公式ページでは、RStanはStan C++パッケージへのRインターフェースであり、NUTS/HMC、ADVI、L-BFGS最適化を提供すると説明されています。

RStanを使うと、Stanモデルを`.stan`ファイルとして書き、Rからデータを渡し、推定を実行し、結果をRオブジェクトとして扱えます。

```r
library(rstan)

stan_data <- list(
  N = length(y),
  y = y
)

fit <- stan(
  file = "model.stan",
  data = stan_data,
  chains = 4,
  iter = 2000,
  warmup = 1000,
  seed = 123
)

print(fit)
```

RStanの`stan()`関数は、Stanモデルをフィットし、その結果を`stanfit`オブジェクトとして返します。公式リファレンスでは、`stan()`はStanモデルをC++コードに変換し、共有オブジェクトとしてコンパイルし、サンプルを生成して`stanfit`オブジェクトにまとめる、と説明されています。

### 1.5.2　Rの分析環境と統合しやすい

RStanの強みは、Rの分析環境と統合しやすいことです。Rでは、データ前処理、可視化、統計解析、レポート作成、パッケージ化までのワークフローが成熟しています。RStanを使うと、Stanで推定した事後ドローをRのオブジェクトとして扱い、`summary()`、`plot()`、`as.data.frame()`、`as.matrix()`などで加工できます。

また、Stan開発チーム関連のRパッケージとして、`bayesplot`、`shinystan`、`loo`、`posterior`、`rstanarm`などがあります。`bayesplot`は推定値・診断・事後予測チェックの可視化、`loo`は予測性能評価やモデル比較、`rstanarm`は応用回帰モデリング向けのR数式インターフェースです。

### 1.5.3　RStanの実行プロセス

RStanで`stan()`を実行すると、内部的には次の処理が行われます。

1. Stanコードを読み込む。
2. StanコードをC++コードへ変換する。
3. C++コードをコンパイルし、Rセッション内に読み込む。
4. Rのリストなどで渡したデータをStanに渡す。
5. NUTS/HMCなどでサンプリングする。
6. 結果を`stanfit`オブジェクトとして返す。

`stan()`のデフォルトは`chains = 4`、`iter = 2000`、`warmup = floor(iter/2)`です。`iter`はウォームアップを含む各チェーンの反復数、`warmup`はステップサイズ適応が行われる期間でもあり、ウォームアップサンプルは推論に使いません。既定かつ推奨アルゴリズムはNUTSです。

### 1.5.4　RStanの基本設定

RStanでは、よく次の設定を使います。

```r
library(rstan)

options(mc.cores = parallel::detectCores())
rstan_options(auto_write = TRUE)
```

`options(mc.cores = ...)`は、複数チェーンを並列実行するためのR側の設定です。`rstan_options(auto_write = TRUE)`は、コンパイル済みの`stanmodel`を保存するかどうかを制御するオプションです。同じStanコードを繰り返し使う場合、再コンパイルの負担を減らせます。

### 1.5.5　RStanを使うべき場面

RStanは、次のような場合に適しています。

**R中心の分析ワークフローを維持したい場合**  
データ処理、可視化、レポート、既存コードがRに集約されているなら、RStanは自然に組み込めます。

**Stanコードを直接書いて柔軟なモデルを構築したい場合**  
`brms`や`rstanarm`では表現しにくい独自の尤度、制約、潜在構造、事後予測量を扱う場合、RStanでStanコードを直接書く価値があります。

**既存のRStan資産がある場合**  
過去に作成された`.stan`ファイル、`stanfit`オブジェクトを前提にした解析コード、RStan依存の社内パッケージや研究コードがある場合、RStanを継続利用する合理性があります。

**CRANバイナリや従来のR構文に依存したい場合**  
CmdStanR公式ドキュメントでは、RStanの利点として、CRANバイナリがMacとWindows向けに利用できること、R6クラスを避けられるため多くのRユーザーにとって構文がなじみやすいこと、事前コンパイル済みStanプログラムを含むRパッケージを配布しやすいことが挙げられています。

### 1.5.6　RStanとCmdStanRの関係

現在のStanエコシステムでは、RStanとCmdStanRの両方があります。CmdStanRはCmdStanを背後で使う軽量なRインターフェースで、Stan ToolkitではRユーザー向けの推奨インターフェースとして掲載されています。

一方、RStanはRセッション内からStanのC++コードを呼び出すインメモリ型のインターフェースです。CmdStanR公式ドキュメントでは、RStanはRcppやinlineを使ってRからC++コードを呼び出すのに対し、CmdStanRはRからC++を直接呼び出さず、CmdStanを通じてコンパイル・実行・出力ファイルへの書き込みを行うと説明されています。

実務上は、新規プロジェクトではCmdStanRも検討対象になります。ただし、RStanは現在でもStanのRインターフェースとして公式に維持されており、R中心の既存資産がある場合には十分に有用です。

## Chapter 1　補足と文献案内

### まず読むべき公式ドキュメント

最初に確認すべきなのは、[RStan公式リファレンスのトップページ](https://mc-stan.org/rstan/reference/rstan.html)です。ここで、RStanがStan C++パッケージへのRインターフェースであること、NUTS/HMC、ADVI、L-BFGSを提供すること、関連Rパッケージとして`bayesplot`、`shinystan`、`loo`、`rstanarm`などがあることを確認できます。

次に読むべきなのは、[RStanの`stan()`関数リファレンス](https://mc-stan.org/rstan/reference/stan.html)です。ここには、`file`、`model_code`、`data`、`chains`、`iter`、`warmup`、`cores`、`thin`などの主要引数、StanコードからC++への変換、コンパイル、サンプリング、`stanfit`生成までの流れがまとまっています。

Stan言語そのものを学ぶ場合は、[Stan Reference Manual](https://mc-stan.org/docs/reference-manual/)を参照します。モデル例と実践的な書き方を学ぶ場合は、[Stan User's Guide](https://mc-stan.org/docs/stan-users-guide/)を使います。

### 学習順序のおすすめ

1. 正規分布モデル、ベルヌーイモデル、ポアソンモデルを書く。
2. RStanで`stan()`を実行し、`print(fit)`で平均、標準偏差、信用区間、R-hat、ESSを見る。
3. `generated quantities`で事後予測を作る。
4. 事後予測チェックを行う。
5. 回帰モデル、階層モデルへ進む。
6. divergence、R-hat、ESS、treedepthなどの診断を学ぶ。
7. 必要に応じて再パラメータ化、事前分布設計、効率化を学ぶ。

併読に向く文献としては、Gelman et al. の *Bayesian Data Analysis*、McElreath の *Statistical Rethinking: A Bayesian Course with Examples in R and Stan* が代表的です。

---

# Chapter 2　StanとRStanをはじめよう

## この章の到達目標

この章では、RStanを実際に動かすための環境準備と、Stanコードを読むための最小限の文法を整理します。Stanは、モデルをStan言語で書き、RStanがStanコードをC++に変換・コンパイルし、Rからサンプリングを実行する、という流れで使います。

> 注：この章の節番号は、ご指定の目次に合わせて「4.1」「4.2」としています。

## 4.1　StanとRStanの準備

### 4.1.1　必要なもの

RStanを使うには、少なくとも次のものが必要です。

- R
- RStudioまたは任意のR実行環境
- `rstan`パッケージ
- C++コンパイル環境
- Stanコードを書くためのテキストエディタ

Stanモデルをコンパイルして実行するにはC++17コンパイラとビルド用ユーティリティが必要です。WindowsではRtools、macOSではXcode Command Line Tools、Linuxでは`build-essential`系のツールが関係します。

### 4.1.2　RStanのインストール

通常はR上で次を実行します。

```r
install.packages(
  "rstan",
  repos = "https://cloud.r-project.org/",
  dependencies = TRUE
)
```

Linuxでnodejs関連のライブラリがない場合に限り、RStan Getting Startedでは次の環境変数を設定する例が示されています。

```r
Sys.setenv(DOWNLOAD_STATIC_LIBV8 = 1)
install.packages(
  "rstan",
  repos = "https://cloud.r-project.org/",
  dependencies = TRUE
)
```

推定結果の可視化と事後予測チェックまで行うなら、次も入れておきます。

```r
install.packages(c("bayesplot", "posterior"), dependencies = TRUE)
```

`bayesplot`はパラメータ推定値、診断、事後予測チェックの可視化に使えます。

### 4.1.3　読み込みと基本設定

RStanを使うRスクリプトの冒頭には、通常、次を書きます。

```r
library(rstan)

options(mc.cores = parallel::detectCores())
rstan_options(auto_write = TRUE)
```

`options(mc.cores = parallel::detectCores())`は、複数チェーンを並列実行するための設定です。`rstan_options(auto_write = TRUE)`は、コンパイル済みStanプログラムを保存し、同じStanコードを再実行するときの再コンパイルを減らすための設定です。

### 4.1.4　インストール確認

RStanのインストール確認には、公式Getting Startedで次のテストコードが示されています。

```r
example(stan_model, package = "rstan", run.dontrun = TRUE)
```

このコードでStanモデルがコンパイルされ、サンプリングが始まれば、RStan、C++コンパイラ、Rとの連携は概ね動作しています。

### 4.1.5　最小のRStan実行例

以下は、StanコードをR文字列として渡す最小例です。実務では`.stan`ファイルに分けるほうが管理しやすいですが、初学段階では`model_code`を使うと動作確認が簡単です。

```r
library(rstan)

options(mc.cores = parallel::detectCores())
rstan_options(auto_write = TRUE)

stan_code_minimal <- "
data {
  int<lower=1> N;
  vector[N] y;
}

parameters {
  real mu;
  real<lower=0> sigma;
}

model {
  mu ~ normal(0, 10);
  sigma ~ exponential(1);
  y ~ normal(mu, sigma);
}

generated quantities {
  real y_new;
  y_new = normal_rng(mu, sigma);
}
"

set.seed(123)
y <- rnorm(30, mean = 2, sd = 1.5)

stan_data <- list(
  N = length(y),
  y = y
)

fit_minimal <- stan(
  model_code = stan_code_minimal,
  data = stan_data,
  chains = 4,
  iter = 2000,
  warmup = 1000,
  seed = 123,
  refresh = 0
)

print(fit_minimal, pars = c("mu", "sigma", "y_new"))
```

## 4.2　Stanの基本的な文法

### 4.2.1　Stanプログラムの基本ブロック

Stanコードは、役割ごとにブロックを分けて書きます。

```stan
functions {
  // ユーザー定義関数
}

data {
  // Rから渡すデータ
}

transformed data {
  // データだけから作る前処理済み変数
}

parameters {
  // 推定したい未知パラメータ
}

transformed parameters {
  // パラメータから導く量。モデルでも使い、保存もしたい量
}

model {
  // 事前分布と尤度
}

generated quantities {
  // 事後予測、派生量、log_lik、乱数生成
}
```

`model`ブロックは対数確率関数を定義する場所、`generated quantities`ブロックはパラメータやデータに基づく派生量や疑似乱数生成を行う場所です。乱数生成関数は、基本的に`transformed data`、`generated quantities`、または`_rng`で終わるユーザー定義関数の中で使います。

### 4.2.2　`data`ブロック

`data`ブロックには、Rから渡す値を宣言します。

```stan
data {
  int<lower=1> N;
  vector[N] y;
}
```

これは、R側で次のようなリストを渡すことを意味します。

```r
stan_data <- list(
  N = length(y),
  y = y
)
```

`data`ブロックの変数は外部入力、たとえばRのデータ構造から読み込まれます。制約に反する値を渡すと、読み込み時点でエラーになります。

### 4.2.3　`parameters`ブロック

`parameters`ブロックには、推定対象を宣言します。

```stan
parameters {
  real alpha;
  real beta;
  real<lower=0> sigma;
}
```

`real<lower=0> sigma;`は、`sigma`が0以上の実数であることを示します。標準偏差、分散、率、濃度など、負にならない量には制約を付けます。

Stanでは、`parameters`ブロックに宣言した量がHMC/NUTSでサンプリングされます。

### 4.2.4　`model`ブロック

`model`ブロックには、事前分布と尤度を書きます。

```stan
model {
  alpha ~ normal(0, 5);
  beta ~ normal(0, 2);
  sigma ~ exponential(1);

  y ~ normal(alpha + beta * x, sigma);
}
```

`alpha ~ normal(0, 5)`は切片の事前分布、`beta ~ normal(0, 2)`は傾きの事前分布、`y ~ normal(...)`は観測データの尤度です。

Stanの`~`記法は、厳密には「サンプリングする」という命令ではありません。`y ~ normal(mu, sigma)`のような分布文は、`target += normal_lpdf(y | mu, sigma)`のように対数密度へ加算するための記法上の便利表現です。

### 4.2.5　`generated quantities`ブロック

`generated quantities`ブロックは、事後予測、派生量、オッズ比、相対リスク、log likelihoodなどを計算する場所です。

```stan
generated quantities {
  array[N] real y_rep;

  for (n in 1:N) {
    y_rep[n] = normal_rng(alpha + beta * x[n], sigma);
  }
}
```

ここで`y_rep`は、推定されたパラメータに基づいて生成される複製データです。事後予測チェック用の複製データは`generated quantities`ブロックで生成できます。

### 4.2.6　型の基本

Stanで頻出する型は次です。

```stan
int n;                     // 整数
real x;                    // 実数
vector[N] y;               // 長さNの列ベクトル
row_vector[K] z;           // 長さKの行ベクトル
matrix[N, K] X;            // N行K列の行列
array[N] int y_bin;        // 長さNの整数配列
array[N] real y_rep;       // 長さNの実数配列
```

Stanの添字はRと同じく1始まりです。

```stan
for (n in 1:N) {
  y[n] ~ normal(mu[n], sigma);
}
```

### 4.2.7　ベクトル化

Stanでは、ループで書ける処理をベクトル化して書けることがあります。

```stan
y ~ normal(alpha + beta * x, sigma);
```

これは概念的には次と同じです。

```stan
for (n in 1:N) {
  y[n] ~ normal(alpha + beta * x[n], sigma);
}
```

Stan User's Guideでは、単回帰モデルの`y ~ normal(alpha + beta * x, sigma)`はベクトル化された形であり、同等のループ版より簡潔で高速だと説明されています。

### 4.2.8　RStanへ渡すデータ

RStanでは、Stanの`data`ブロックで宣言した名前と同じ名前を持つRのリストを渡します。

```r
stan_data <- list(
  N = nrow(df),
  K = 3,
  X = X,
  y = df$y
)
```

注意点として、RStan公式リファレンスでは、Rの長さ1ベクトルはStan側ではスカラーとして扱われるため、`array[1] real y;`のようなStanデータには、R側で`array(1.0, dim = 1)`のように配列として渡す必要があると説明されています。

## Chapter 2　補足と文献案内

この章で最初に読むべき公式資料は、[RStan公式リファレンス](https://mc-stan.org/rstan/reference/rstan.html)、[RStan Getting Started](https://github.com/stan-dev/rstan/wiki/RStan-Getting-Started)、[Stan Reference ManualのProgram Blocks](https://mc-stan.org/docs/reference-manual/blocks.html)、[Stan User's GuideのRegression Models](https://mc-stan.org/docs/stan-users-guide/regression.html)です。

RStanで実行する関数の仕様はRStan公式リファレンス、Stanコードの仕様はStan Reference Manual、モデル例はStan User's Guideで確認する、という使い分けが実務的です。

---


# Chapter 3　基本的な回帰とモデルのチェック

## この章の到達目標

この章では、RStanで基本的な回帰モデルを実装し、その出力を感染症分野の実務的な問いに結びつけて解釈します。単に「数式を書いてStanで推定する」だけでなく、次の点を重視します。

1. 回帰モデルが何を仮定しているのかを、数式と日本語で理解する。
2. 線形予測子、リンク関数、尤度、事前分布、事後分布の役割を区別する。
3. Stanコードの各ブロックが何のために存在するかを理解する。
4. `print(fit)`や事後ドローから得られる結果を、感染症監視、リスク評価、介入評価、予測にどう使うかを説明できるようにする。
5. 事後予測チェックとHMC診断を使い、モデルを盲目的に信用しない習慣を身につける。

Stan User's Guideでは、回帰モデルをStanで書く方法、`generated quantities`を使った予測、事後予測チェックが整理されています。Stanでは、モデルの当てはまりだけでなく、MCMC診断、事後予測、モデル仮定の確認まで含めて分析します。

本章のサンプルデータはすべてRで生成するシミュレーションデータです。実患者データではありませんが、感染症分野でよく出るデータ構造に合わせています。

| 節 | モデル | データ単位 | 感染症分野での典型例 | 主な解釈量 |
|---|---|---|---|---|
| 5.1 | 単回帰 | 週単位・地域単位 | 下水中ウイルス濃度と症例率 | 傾き、予測値、しきい値超過確率 |
| 5.2 | 重回帰 | 週単位・地域単位 | 入院率と症例数、接種率、移動量 | 調整済み係数、シナリオ予測 |
| 5.3 | 二項ロジスティック回帰 | 集計データ | 検査陽性数 / 検査数 | 陽性確率、オッズ比 |
| 5.4 | ロジスティック回帰 | 個人単位 | 感染有無、曝露、接種歴 | 個人リスク、オッズ比、VE風指標 |
| 5.5 | ポアソン回帰 | 地域・週のカウント | 症例数、人口オフセット | 発生率比、予測症例数 |

---

## 3.0　回帰モデルを読むための共通枠組み

### 3.0.1　回帰モデルの基本形

回帰モデルは、応答変数 $y$ を説明変数 $x$ で説明するモデルです。ただし、統計モデリングでは、$y$ を単なる決定論的な値として扱うのではなく、確率分布から発生した観測値として扱います。

多くの回帰モデルは次の3層で理解できます。

**第1層：観測モデル**

$$
y_n \sim \text{何らかの確率分布}
$$

これは、観測値がどのような確率分布から出たと仮定するかを決める部分です。連続値なら正規分布、陽性数なら二項分布、症例数ならポアソン分布を使うことが多いです。

**第2層：平均・確率・率のモデル**

$$
\mu_n,\ p_n,\ \lambda_n = f(\alpha + X_n\beta)
$$

ここでは、観測分布の中心的な量を説明変数で表します。正規回帰では平均 $\mu_n$、ロジスティック回帰では確率 $p_n$、ポアソン回帰では期待カウント $\lambda_n$ をモデル化します。

**第3層：事前分布**

$$
\alpha \sim \text{prior}, \quad \beta_k \sim \text{prior}
$$

ベイズモデルでは、未知パラメータに事前分布を置きます。これは「データを見る前の知識」または「非現実的な値を抑制するための正則化」として機能します。

### 3.0.2　線形予測子とは

回帰モデルでは、次の形が頻繁に出てきます。

$$
\eta_n = \alpha + \beta_1 x_{n1} + \beta_2 x_{n2} + \cdots + \beta_K x_{nK}
$$

この $\eta_n$ を**線形予測子**と呼びます。

- $\alpha$ は切片です。説明変数が0のときの基準値を表します。
- $\beta_k$ は係数です。$x_k$ が1単位増えたとき、$\eta$ がどれだけ変わるかを表します。
- $X_n\beta$ は、複数の説明変数の寄与を合計したものです。

ただし、$\eta_n$ がそのまま平均や確率になるとは限りません。ロジスティック回帰やポアソン回帰では、リンク関数を通して確率や率に変換します。

### 3.0.3　リンク関数とは

感染症データでは、応答変数の種類によって自然な範囲が異なります。

- 確率 $p$ は0から1の間でなければなりません。
- 症例数の期待値 $\lambda$ は0以上でなければなりません。
- 連続値の平均 $\mu$ は実数全体を取り得ます。

線形予測子 $\eta$ は $-\infty$ から $+\infty$ まで取り得るため、必要に応じて変換します。

ロジスティック回帰では、

$$
p_n = \frac{1}{1 + \exp(-\eta_n)}
$$

とします。これはStanでは`inv_logit(eta)`または`bernoulli_logit`、`binomial_logit`として扱えます。

ポアソン回帰では、

$$
\lambda_n = \exp(\eta_n)
$$

とします。Stanでは`poisson_log(eta)`を使うと、$\eta$をログ期待値として直接渡せます。

### 3.0.4　RStanの出力で見るもの

`print(fit)`で表示される代表的な列は次のように読みます。

| 列 | 意味 | 実務上の読み方 |
|---|---|---|
| `mean` | 事後平均 | 推定量の平均的な値 |
| `sd` | 事後標準偏差 | 推定の不確実性 |
| `2.5%`, `50%`, `97.5%` | 事後分布の分位点 | 95%信用区間と中央値 |
| `n_eff` | 有効サンプルサイズ | MCMCサンプルの実効的な情報量 |
| `Rhat` | 収束診断 | 1.00に近いほどよい。最終推論では概ね1.01未満を目安にする |

感染症分野では、係数そのものだけでなく、次のような派生量のほうが実務的に使いやすいことが多いです。

- オッズ比：`exp(beta)`
- 発生率比：`exp(beta)`
- ワクチン有効性風の値：`(1 - odds_ratio) * 100`
- しきい値超過確率：`mean(predicted_cases > threshold)`
- 予測陽性率：`posterior mean of p`
- 予測症例数：`posterior mean of lambda`または`y_rep`

### 3.0.5　共通Rコード

以下は本章の各節で共通して使うRコードです。

```r
library(rstan)
library(bayesplot)

options(mc.cores = parallel::detectCores())
rstan_options(auto_write = TRUE)

set.seed(2026)

stan_sampling_args <- list(
  chains = 4,
  iter = 2000,
  warmup = 1000,
  seed = 2026,
  refresh = 0,
  control = list(adapt_delta = 0.95)
)

check_fit <- function(fit, pars) {
  print(fit, pars = pars, probs = c(0.025, 0.5, 0.975))
  rstan::check_hmc_diagnostics(fit)
}
```

`adapt_delta = 0.95`はdivergenceを減らす目的で使うことがあります。ただし、警告が出たときは、単に`adapt_delta`を上げるだけでは不十分です。事前分布、スケーリング、パラメータ化、尤度、外れ値、過分散、識別性を確認します。

---

# 5.1　単回帰

## 5.1.1　感染症分野での問い

ここでは、週別の下水中ウイルスRNA濃度と、同じ地域の週別COVID-19症例率の関係を考えます。下水監視では、地域の感染状況を個人検査とは別の経路で把握できます。実務上の問いは次のようになります。

> 下水中ウイルス濃度が高い週ほど、地域の症例率も高いと言えるか。さらに、下水指標から次週または同週の症例率をどの程度予測できるか。

この節では簡単のため、下水中ウイルス濃度を対数変換して標準化し、対数症例率を説明します。

## 5.1.2　数式

$$
y_n \sim \mathrm{Normal}(\mu_n, \sigma)
$$

$$
\mu_n = \alpha + \beta x_n
$$

$$
\alpha \sim \mathrm{Normal}(0, 5), \quad
\beta \sim \mathrm{Normal}(0, 2), \quad
\sigma \sim \mathrm{Exponential}(1)
$$

ここで、

- $y_n$：週 $n$ の対数症例率、たとえば`log(cases_per_100k)`
- $x_n$：週 $n$ の標準化済み下水ウイルス指標
- $\mu_n$：モデルが予測する平均的な対数症例率
- $\alpha$：下水指標が平均的な週の対数症例率
- $\beta$：下水指標が1標準偏差高いときの対数症例率の変化
- $\sigma$：モデルでは説明しきれない週ごとのばらつき

## 5.1.3　数式の仕組み

このモデルは、「観測された対数症例率は、下水指標から予測される平均値 $\mu_n$ の周りに正規分布でばらつく」と仮定します。

下水指標 $x_n$ が1増えると、平均対数症例率 $\mu_n$ は $\beta$ だけ変化します。アウトカムが対数症例率なので、元の症例率スケールでは $\exp(\beta)$ 倍として解釈できます。

たとえば、$\beta = 0.70$ なら、

$$
\exp(0.70) \approx 2.01
$$

です。つまり、下水指標が1標準偏差高い週では、症例率が約2倍高い、という読み方になります。ただし、これは相関・予測の解釈であり、下水中ウイルス濃度が症例発生を原因として増やすという意味ではありません。

## 5.1.4　サンプルデータ作成

```r
set.seed(101)

N <- 40
week <- 1:N

# 下水中ウイルスRNA濃度を想定した指標。
# 実データでは、コピー数、流量補正値、人口補正値などを使うことがある。
wastewater_raw <- exp(
  2.5 +
    0.03 * week +
    sin(2 * pi * week / 12) * 0.5 +
    rnorm(N, 0, 0.35)
)

# 解析では対数化して標準化する。
log_wastewater <- log(wastewater_raw)
x <- as.numeric(scale(log_wastewater))

# 真の生成過程。下水指標が高いほど症例率も高いように作る。
alpha_true <- 2.0
beta_true <- 0.75
sigma_true <- 0.35

log_cases_per_100k <- rnorm(
  N,
  mean = alpha_true + beta_true * x,
  sd = sigma_true
)

cases_per_100k <- exp(log_cases_per_100k)

d1 <- data.frame(
  week = week,
  wastewater_raw = wastewater_raw,
  log_wastewater_z = x,
  cases_per_100k = cases_per_100k,
  log_cases_per_100k = log_cases_per_100k
)

head(d1)
```

## 5.1.5　Stanコード

```r
stan_code_simple_lm <- "
data {
  // Rから渡される観測データを宣言する。
  // Nは週数、xは標準化済み下水指標、yは対数症例率。
  int<lower=1> N;
  vector[N] x;
  vector[N] y;
}

parameters {
  // alphaは切片、betaは下水指標の係数。
  // sigmaは観測誤差の標準偏差なので0以上に制約する。
  real alpha;
  real beta;
  real<lower=0> sigma;
}

model {
  // 事前分布。
  // xとyを標準化または対数化しているため、過度に広すぎない弱情報事前分布を使う。
  alpha ~ normal(0, 5);
  beta ~ normal(0, 2);
  sigma ~ exponential(1);

  // 尤度。
  // 対数症例率yは、平均 alpha + beta * x、標準偏差sigmaの正規分布に従うと仮定する。
  y ~ normal(alpha + beta * x, sigma);
}

generated quantities {
  // 推定後に計算したい量を置く。
  // muは各週の平均予測値、y_repは事後予測チェック用の複製データ。
  vector[N] mu;
  array[N] real y_rep;
  real rate_ratio_per_1sd;

  rate_ratio_per_1sd = exp(beta);

  for (n in 1:N) {
    mu[n] = alpha + beta * x[n];
    y_rep[n] = normal_rng(mu[n], sigma);
  }
}
"
```

## 5.1.6　Stanコードのブロック別意図

**`data`ブロック**  
Stanの外から渡す値を宣言します。ここでは週数`N`、説明変数`x`、応答変数`y`を渡します。Stanは`data`ブロックの値を固定値として扱い、推定対象にはしません。

**`parameters`ブロック**  
データから推定したい未知量を宣言します。`alpha`と`beta`は実数全体を取り得ます。`sigma`は標準偏差なので負にならないよう`real<lower=0>`とします。

**`model`ブロック**  
事前分布と尤度を書きます。Stanの`~`記法は乱数生成ではなく、対数密度への加算です。ここでは、パラメータに事前分布を置き、観測された`y`が正規分布から生じたという仮定を入れています。

**`generated quantities`ブロック**  
MCMCで得られた各事後ドローに対して、追加の量を計算します。`mu`は平均構造の推定値、`y_rep`は同じモデルから生成される仮想データです。`rate_ratio_per_1sd = exp(beta)`は、対数症例率を元の率スケールに戻した解釈用の量です。

## 5.1.7　RStanで推定

```r
stan_data1 <- list(
  N = nrow(d1),
  x = d1$log_wastewater_z,
  y = d1$log_cases_per_100k
)

fit1 <- do.call(
  rstan::stan,
  c(
    list(model_code = stan_code_simple_lm, data = stan_data1),
    stan_sampling_args
  )
)

check_fit(fit1, pars = c("alpha", "beta", "sigma", "rate_ratio_per_1sd"))
```

## 5.1.8　出力結果の読み方

`beta`が正で、95%信用区間の大部分が0より上にあれば、下水指標と対数症例率の間に正の関連があると解釈します。

`rate_ratio_per_1sd`は実務上わかりやすい量です。たとえば中央値が2.1なら、下水指標が1標準偏差高い週では、症例率が約2.1倍高いという意味です。

`sigma`は、下水指標だけでは説明できない週ごとのばらつきです。`sigma`が大きい場合、下水指標は関連していても、単独では症例率の予測に不十分かもしれません。

## 5.1.9　モデルチェックと実応用

```r
post1 <- rstan::extract(fit1)
y_rep1 <- post1$y_rep

# 観測データと複製データの分布比較
bayesplot::ppc_dens_overlay(
  y = d1$log_cases_per_100k,
  yrep = y_rep1[sample(seq_len(nrow(y_rep1)), 50), ]
)

# 予測平均と観測値の比較
mu_mean1 <- apply(post1$mu, 2, mean)

plot(
  mu_mean1,
  d1$log_cases_per_100k,
  xlab = "Posterior mean fitted log case rate",
  ylab = "Observed log case rate"
)
abline(0, 1)
```

実応用では、次のような使い方ができます。

```r
# 例：症例率が100人/10万人を超える確率を週ごとに推定する。
threshold_log <- log(100)
prob_above_100 <- colMeans(post1$y_rep > threshold_log)

result1 <- data.frame(
  week = d1$week,
  observed_cases_per_100k = d1$cases_per_100k,
  prob_cases_above_100_per_100k = prob_above_100
)

head(result1)
```

この確率は、下水監視に基づく警戒レベル設定や追加検査体制の判断に使えます。ただし、実務では、報告遅れ、曜日効果、検査行動の変化、下水処理区の人口変動などを考慮する必要があります。

---

# 5.2　重回帰

## 5.2.1　感染症分野での問い

単回帰では1つの説明変数だけを使いました。しかし、感染症データでは複数の要因が同時に関係します。たとえば、入院率は症例数だけでなく、ワクチン接種率、年齢構成、変異株、医療アクセス、行動変化、季節性に影響されます。

この節では、週別の呼吸器感染症入院率を、症例数指標、ワクチン接種率、移動量、変異株流行期で説明します。

## 5.2.2　数式

$$
y_n \sim \mathrm{Normal}(\mu_n, \sigma)
$$

$$
\mu_n = \alpha + \beta_1 x_{n1} + \beta_2 x_{n2} + \beta_3 x_{n3} + \beta_4 x_{n4}
$$

行列で書くと、

$$
\mu = \alpha + X\beta
$$

です。

ここで、

- $y_n$：週 $n$ の対数入院率
- $x_{n1}$：標準化済み症例数指標
- $x_{n2}$：標準化済みワクチン接種率
- $x_{n3}$：標準化済み移動量
- $x_{n4}$：変異株流行期の指標、0または1
- $\beta_k$：他の変数を固定したときの各説明変数の関連

## 5.2.3　数式の仕組み

重回帰のポイントは、係数が**調整済みの関連**を表すことです。

たとえば、$\beta_2$がワクチン接種率の係数だとします。この係数は、「症例数指標、移動量、変異株流行期が同じ条件だとしたとき、ワクチン接種率が高い週では入院率がどの程度違うか」を表します。

ただし、これは自動的に因果効果を意味しません。ワクチン接種率が高い地域は年齢構成、医療アクセス、感染歴、行動、検査体制も異なる可能性があります。重回帰は交絡調整の基本ですが、必要な交絡因子が入っていなければ、係数はバイアスを含みます。

アウトカムが対数入院率の場合、$\exp(\beta_k)$は入院率比として読めます。たとえば、ワクチン接種率の標準化係数が$-0.30$なら、

$$
\exp(-0.30) \approx 0.74
$$

なので、接種率が1標準偏差高い週では、調整後の入院率が約26%低い、という読み方になります。

## 5.2.4　サンプルデータ作成

```r
set.seed(102)

N <- 60
week <- 1:N

log_cases_z <- as.numeric(scale(
  1.5 + sin(2 * pi * week / 16) + 0.03 * week + rnorm(N, 0, 0.4)
))

vacc_coverage <- pmin(0.9, pmax(0.2, 0.35 + 0.008 * week + rnorm(N, 0, 0.03)))
vacc_z <- as.numeric(scale(vacc_coverage))

mobility_index <- 0.2 * sin(2 * pi * week / 10) + rnorm(N, 0, 0.4)
mobility_z <- as.numeric(scale(mobility_index))

variant_period <- ifelse(week >= 35, 1, 0)

X2 <- cbind(
  log_cases_z = log_cases_z,
  vacc_z = vacc_z,
  mobility_z = mobility_z,
  variant_period = variant_period
)

beta_true <- c(
  log_cases_z = 0.65,
  vacc_z = -0.35,
  mobility_z = 0.20,
  variant_period = 0.45
)

alpha_true <- 1.0
sigma_true <- 0.30

log_hosp_per_100k <- rnorm(
  N,
  mean = as.vector(alpha_true + X2 %*% beta_true),
  sd = sigma_true
)

d2 <- data.frame(
  week = week,
  log_hosp_per_100k = log_hosp_per_100k,
  hosp_per_100k = exp(log_hosp_per_100k),
  log_cases_z = log_cases_z,
  vacc_coverage = vacc_coverage,
  vacc_z = vacc_z,
  mobility_z = mobility_z,
  variant_period = variant_period
)

head(d2)
```

## 5.2.5　Stanコード

```r
stan_code_multiple_lm <- "
data {
  // Nは観測数、Kは説明変数の数。
  // XはN行K列の説明変数行列、yは対数入院率。
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  vector[N] y;
}

parameters {
  // alphaは切片。
  // betaはK個の説明変数に対応する係数ベクトル。
  // sigmaは正規分布の残差標準偏差。
  real alpha;
  vector[K] beta;
  real<lower=0> sigma;
}

model {
  // 事前分布。
  // 説明変数を標準化しているため、beta ~ normal(0, 2)はかなり広い。
  alpha ~ normal(0, 5);
  beta ~ normal(0, 2);
  sigma ~ exponential(1);

  // 尤度。
  // X * betaで各観測の線形予測子をまとめて計算する。
  y ~ normal(alpha + X * beta, sigma);
}

generated quantities {
  // muは各観測の平均予測値。
  // y_repは事後予測チェック用。
  // rate_ratioは各係数を元の率比スケールに変換した量。
  vector[N] mu;
  array[N] real y_rep;
  vector[K] rate_ratio;

  mu = alpha + X * beta;
  rate_ratio = exp(beta);

  for (n in 1:N) {
    y_rep[n] = normal_rng(mu[n], sigma);
  }
}
"
```

## 5.2.6　Stanコードのブロック別意図

**`data`ブロック**  
複数の説明変数を`matrix[N, K] X`として渡します。R側で`cbind()`した列の順番が、Stanの`beta[1]`、`beta[2]`、...に対応します。

**`parameters`ブロック**  
`beta`を`vector[K]`として宣言することで、説明変数の数が増えても同じStanコードを使えます。

**`model`ブロック**  
`X * beta`は、各観測について $\beta_1x_1 + \cdots + \beta_Kx_K$ を計算します。ベクトル化して書くことで、コードが短くなり、効率も良くなります。

**`generated quantities`ブロック**  
`rate_ratio = exp(beta)`により、対数スケールの係数を率比として保存します。R側で毎回計算してもよいですが、Stan側に書くと`print(fit, pars = "rate_ratio")`で直接確認できます。

## 5.2.7　RStanで推定

```r
stan_data2 <- list(
  N = nrow(d2),
  K = ncol(X2),
  X = X2,
  y = d2$log_hosp_per_100k
)

fit2 <- do.call(
  rstan::stan,
  c(
    list(model_code = stan_code_multiple_lm, data = stan_data2),
    stan_sampling_args
  )
)

check_fit(fit2, pars = c("alpha", "beta", "sigma", "rate_ratio"))
```

## 5.2.8　出力結果の読み方

```r
summary2_beta <- summary(fit2, pars = "beta")$summary
rownames(summary2_beta) <- colnames(X2)
summary2_beta[, c("mean", "sd", "2.5%", "50%", "97.5%", "Rhat", "n_eff")]

summary2_rr <- summary(fit2, pars = "rate_ratio")$summary
rownames(summary2_rr) <- colnames(X2)
summary2_rr[, c("mean", "sd", "2.5%", "50%", "97.5%", "Rhat", "n_eff")]
```

`beta`は対数入院率スケールの係数です。`rate_ratio`は入院率比です。

- `log_cases_z`の`rate_ratio`が1より大きい：症例数指標が高い週ほど入院率が高い。
- `vacc_z`の`rate_ratio`が1より小さい：接種率が高い週ほど、調整後の入院率が低い。
- `variant_period`の`rate_ratio`が1より大きい：変異株流行期では、他の変数を調整しても入院率が高い。

## 5.2.9　モデルチェックと実応用

```r
post2 <- rstan::extract(fit2)
y_rep2 <- post2$y_rep

bayesplot::ppc_dens_overlay(
  y = d2$log_hosp_per_100k,
  yrep = y_rep2[sample(seq_len(nrow(y_rep2)), 50), ]
)

mu_mean2 <- apply(post2$mu, 2, mean)

plot(
  mu_mean2,
  d2$log_hosp_per_100k,
  xlab = "Posterior mean fitted log hospitalization rate",
  ylab = "Observed log hospitalization rate"
)
abline(0, 1)
```

実応用では、シナリオ比較が重要です。たとえば「接種率が1標準偏差高い場合、入院率はどの程度下がるか」を事後ドローで計算できます。

```r
post2 <- rstan::extract(fit2)
j_vacc <- which(colnames(X2) == "vacc_z")

# 接種率が1標準偏差高い場合の入院率比。
rr_vacc_1sd <- exp(post2$beta[, j_vacc])

quantile(rr_vacc_1sd, probs = c(0.025, 0.5, 0.975))
```

この結果は、感染症対策会議などで「接種率の高い地域・期間では、調整後入院率がどの程度低い傾向にあるか」を説明する材料になります。ただし、観察データなので、因果効果として報告するには設計と交絡調整が不十分でないかを別途検討します。

---

# 5.3　二項ロジスティック回帰

## 5.3.1　感染症分野での問い

この節では、週別の検査陽性数と検査数を扱います。感染症監視では、単に陽性率`positive / tested`を計算するだけでなく、検査数の違いを考慮した上で、陽性確率がどの要因と関係しているかをモデル化します。

問いは次のようになります。

> ILI活動、ワクチン接種率、季節性を考慮したとき、週別の検査陽性確率はどう変わるか。

このような集計データでは、陽性率を正規分布で近似するよりも、陽性数を二項分布で扱うほうが自然です。検査数が少ない週では陽性率の不確実性が大きく、検査数が多い週では不確実性が小さいことを、二項分布が自動的に反映します。

## 5.3.2　数式

$$
y_n \sim \mathrm{Binomial}(m_n, p_n)
$$

$$
\mathrm{logit}(p_n) = \eta_n = \alpha + X_n\beta
$$

$$
p_n = \frac{1}{1 + \exp(-\eta_n)}
$$

ここで、

- $y_n$：週 $n$ の陽性数
- $m_n$：週 $n$ の検査数
- $p_n$：週 $n$ の陽性確率
- $\eta_n$：陽性オッズの対数、つまりlog odds
- $\beta_k$：説明変数が1単位増えたときのlog oddsの変化

## 5.3.3　数式の仕組み

二項分布は、「$m_n$回の検査のうち、$y_n$回陽性になる」というデータに合っています。陽性率 $y_n/m_n$ を直接モデル化するのではなく、陽性数そのものをモデル化します。

ロジット変換は、確率 $p$ を実数全体に変換します。

$$
\mathrm{logit}(p) = \log\frac{p}{1-p}
$$

$\frac{p}{1-p}$ はオッズです。したがって、係数 $\beta$ を指数変換すると、オッズ比になります。

$$
\mathrm{odds\ ratio} = \exp(\beta)
$$

たとえば、ILI活動の係数が$1.1$なら、

$$
\exp(1.1) \approx 3.0
$$

です。つまり、ILI活動指標が1標準偏差高い週では、検査陽性のオッズが約3倍になるという読み方です。

## 5.3.4　サンプルデータ作成

```r
set.seed(103)

N <- 52
week <- 1:N

tested <- sample(150:450, N, replace = TRUE)

ili_activity <- 0.5 + sin(2 * pi * week / 52) + rnorm(N, 0, 0.4)
ili_z <- as.numeric(scale(ili_activity))

vacc_coverage <- pmin(0.75, pmax(0.25, 0.40 + 0.004 * week + rnorm(N, 0, 0.03)))
vacc_z <- as.numeric(scale(vacc_coverage))

season_sin <- sin(2 * pi * week / 52)
season_cos <- cos(2 * pi * week / 52)

X3 <- cbind(
  ili_z = ili_z,
  vacc_z = vacc_z,
  season_sin = season_sin,
  season_cos = season_cos
)

beta_true <- c(
  ili_z = 1.10,
  vacc_z = -0.45,
  season_sin = 0.60,
  season_cos = -0.20
)

alpha_true <- -2.0

eta <- as.vector(alpha_true + X3 %*% beta_true)
p <- plogis(eta)

positive <- rbinom(N, size = tested, prob = p)

d3 <- data.frame(
  week = week,
  tested = tested,
  positive = positive,
  positivity = positive / tested,
  ili_z = ili_z,
  vacc_coverage = vacc_coverage,
  vacc_z = vacc_z,
  season_sin = season_sin,
  season_cos = season_cos
)

head(d3)
```

## 5.3.5　Stanコード

```r
stan_code_binomial_logit <- "
data {
  // 集計された検査データを受け取る。
  // positive[n]は陽性数、tested[n]は検査数。
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  array[N] int<lower=0> tested;
  array[N] int<lower=0> positive;
}

parameters {
  // alphaは基準log odds。
  // betaは各説明変数がlog oddsに与える影響。
  real alpha;
  vector[K] beta;
}

model {
  // 事前分布。
  // ロジットスケールでは極端な係数が非常に大きな確率変化を意味するため、
  // normal(0, 1.5)程度の弱情報事前分布を置く。
  alpha ~ normal(0, 3);
  beta ~ normal(0, 1.5);

  // 尤度。
  // binomial_logitは、成功確率pではなくlogit(p)を直接受け取る。
  // inv_logitを明示的に書くより数値的に安定である。
  positive ~ binomial_logit(tested, alpha + X * beta);
}

generated quantities {
  // etaはlog odds、pは陽性確率。
  // y_repは陽性数の事後予測、log_likはモデル比較用。
  // odds_ratioは解釈しやすいようにexp(beta)で保存する。
  vector[N] eta;
  vector[N] p;
  array[N] int<lower=0> y_rep;
  vector[N] log_lik;
  vector[K] odds_ratio;

  eta = alpha + X * beta;
  odds_ratio = exp(beta);

  for (n in 1:N) {
    p[n] = inv_logit(eta[n]);
    y_rep[n] = binomial_rng(tested[n], p[n]);
    log_lik[n] = binomial_lpmf(positive[n] | tested[n], p[n]);
  }
}
"
```

## 5.3.6　Stanコードのブロック別意図

**`data`ブロック**  
各週の検査数`tested`と陽性数`positive`を渡します。陽性率ではなく、陽性数と検査数を分けて渡す点が重要です。これにより、検査数が多い週ほど情報量が多いことをモデルが反映できます。

**`parameters`ブロック**  
推定するのは`alpha`と`beta`だけです。陽性確率`p`はパラメータとして直接サンプリングするのではなく、`alpha + X * beta`から導かれる量として扱います。

**`model`ブロック**  
`positive ~ binomial_logit(tested, alpha + X * beta)`が中心です。これは、各週の陽性数が、検査数`tested[n]`、陽性確率`p[n]`の二項分布に従い、その`p[n]`のロジットが線形予測子で表される、という意味です。

**`generated quantities`ブロック**  
`p`を保存することで、週別の予測陽性率を直接確認できます。`y_rep`はモデルが再現する陽性数、`log_lik`は後でLOO-CVなどを使う場合の準備、`odds_ratio`は係数の実務的な解釈用です。

## 5.3.7　RStanで推定

```r
stan_data3 <- list(
  N = nrow(d3),
  K = ncol(X3),
  X = X3,
  tested = d3$tested,
  positive = d3$positive
)

fit3 <- do.call(
  rstan::stan,
  c(
    list(model_code = stan_code_binomial_logit, data = stan_data3),
    stan_sampling_args
  )
)

check_fit(fit3, pars = c("alpha", "beta", "odds_ratio"))
```

## 5.3.8　出力結果の読み方

```r
summary3_or <- summary(fit3, pars = "odds_ratio")$summary
rownames(summary3_or) <- colnames(X3)
summary3_or[, c("mean", "sd", "2.5%", "50%", "97.5%", "Rhat", "n_eff")]
```

- `ili_z`のオッズ比が1より大きい：ILI活動が高い週ほど陽性になりやすい。
- `vacc_z`のオッズ比が1より小さい：接種率が高い週ほど陽性オッズが低い傾向がある。
- 季節性項は、年間周期の波を表すために入れている。

二項ロジスティック回帰の結果は、陽性率監視の平滑化、異常週の検出、流行期の判定、検査体制の調整に使えます。

## 5.3.9　モデルチェックと実応用

```r
post3 <- rstan::extract(fit3)

p_mean3 <- apply(post3$p, 2, mean)

plot(
  d3$positivity,
  p_mean3,
  xlab = "Observed positivity",
  ylab = "Posterior mean positivity"
)
abline(0, 1)

# 複製陽性率を作る。
p_rep3 <- sweep(post3$y_rep, 2, d3$tested, "/")

bayesplot::ppc_stat(
  y = d3$positivity,
  yrep = p_rep3[sample(seq_len(nrow(p_rep3)), 100), ],
  stat = "mean"
)

bayesplot::ppc_stat(
  y = d3$positivity,
  yrep = p_rep3[sample(seq_len(nrow(p_rep3)), 100), ],
  stat = "sd"
)
```

しきい値を使うと、監視業務に近い出力になります。

```r
# 例：各週の陽性確率が10%を超える事後確率。
prob_positivity_above_10 <- colMeans(post3$p > 0.10)

result3 <- data.frame(
  week = d3$week,
  tested = d3$tested,
  positive = d3$positive,
  observed_positivity = d3$positivity,
  posterior_mean_positivity = p_mean3,
  prob_positivity_above_10 = prob_positivity_above_10
)

head(result3)
```

このような結果は、「陽性率が10%を超えている可能性が高い週」を検出し、検査能力、医療機関への注意喚起、公衆衛生メッセージの発出判断に使えます。

---

# 5.4　ロジスティック回帰

## 5.4.1　感染症分野での問い

5.3は週単位の集計データでした。ここでは、個人単位の感染有無を扱います。たとえば、接触者調査、症例対照研究、検査陰性デザイン、血清疫学調査では、個人ごとに感染あり/なし、ワクチン接種あり/なし、曝露あり/なし、年齢、マスク着用などを持ちます。

問いは次のようになります。

> 年齢、ワクチン接種、同居内曝露、マスク着用を考慮したとき、個人の感染確率はどう変わるか。

## 5.4.2　数式

$$
y_i \sim \mathrm{Bernoulli}(p_i)
$$

$$
\mathrm{logit}(p_i) = \eta_i = \alpha + \beta_1\text{age}_i + \beta_2\text{vaccinated}_i + \beta_3\text{exposure}_i + \beta_4\text{mask}_i
$$

$$
p_i = \frac{1}{1 + \exp(-\eta_i)}
$$

ここで、

- $y_i$：個人 $i$ の感染有無、0または1
- $p_i$：個人 $i$ が感染している確率
- $\beta_2$：ワクチン接種ありとなしのlog odds差
- $\exp(\beta_2)$：接種者と非接種者の感染オッズ比

## 5.4.3　数式の仕組み

ベルヌーイ分布は、0/1の個人アウトカムに対応します。二項分布が「集計された成功数」を扱うのに対し、ベルヌーイ分布は「1人ごとの成功/失敗」を扱います。

ロジスティック回帰では、説明変数の効果は確率に対して直接足し算されるのではなく、log oddsに対して足し算されます。このため、係数の指数変換がオッズ比になります。

ワクチン接種の係数が $-0.7$ なら、

$$
\exp(-0.7) \approx 0.50
$$

です。これは、他の変数が同じなら、接種者の感染オッズが非接種者の約0.5倍であることを意味します。

検査陰性デザインなどの文脈では、単純化して、

$$
\mathrm{VE} = (1 - \mathrm{OR}) \times 100\%
$$

と表すことがあります。ただし、この解釈には研究デザインと交絡調整の前提が必要です。

## 5.4.4　サンプルデータ作成

```r
set.seed(104)

N <- 800

age <- round(runif(N, 18, 85))
age_z <- as.numeric(scale(age))

vaccinated <- rbinom(N, 1, 0.65)
household_exposure <- rbinom(N, 1, 0.25)
mask_regular <- rbinom(N, 1, 0.55)

X4 <- cbind(
  age_z = age_z,
  vaccinated = vaccinated,
  household_exposure = household_exposure,
  mask_regular = mask_regular
)

beta_true <- c(
  age_z = 0.20,
  vaccinated = -0.70,
  household_exposure = 1.30,
  mask_regular = -0.45
)

alpha_true <- -1.6

eta <- as.vector(alpha_true + X4 %*% beta_true)
p <- plogis(eta)

infected <- rbinom(N, 1, p)

d4 <- data.frame(
  age = age,
  age_z = age_z,
  vaccinated = vaccinated,
  household_exposure = household_exposure,
  mask_regular = mask_regular,
  infected = infected
)

head(d4)
table(d4$infected)
```

## 5.4.5　Stanコード

```r
stan_code_bernoulli_logit <- "
data {
  // 個人単位のデータを受け取る。
  // yは感染有無で、0または1。
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  array[N] int<lower=0, upper=1> y;
}

parameters {
  // alphaは基準log odds。
  // betaは個人属性や曝露が感染log oddsに与える効果。
  real alpha;
  vector[K] beta;
}

model {
  // 事前分布。
  // ロジスティック回帰では係数が大きいと確率が0または1に張り付きやすい。
  // そのため、弱情報だが極端すぎない事前分布を使う。
  alpha ~ normal(0, 3);
  beta ~ normal(0, 1.5);

  // 尤度。
  // bernoulli_logitはlogit(p)を直接受け取る。
  y ~ bernoulli_logit(alpha + X * beta);
}

generated quantities {
  // pは個人ごとの感染確率。
  // y_repは事後予測チェック用の感染有無。
  // odds_ratioは係数の指数変換。
  // vaccine_effectiveness_likeは接種係数が2列目であることを前提にしたVE風指標。
  vector[N] eta;
  vector[N] p;
  array[N] int<lower=0, upper=1> y_rep;
  vector[K] odds_ratio;
  real vaccine_effectiveness_like;
  vector[N] log_lik;

  eta = alpha + X * beta;
  odds_ratio = exp(beta);
  vaccine_effectiveness_like = (1 - exp(beta[2])) * 100;

  for (n in 1:N) {
    p[n] = inv_logit(eta[n]);
    y_rep[n] = bernoulli_rng(p[n]);
    log_lik[n] = bernoulli_lpmf(y[n] | p[n]);
  }
}
"
```

## 5.4.6　Stanコードのブロック別意図

**`data`ブロック**  
個人単位の説明変数行列`X`と、感染有無`y`を渡します。`y`には`<lower=0, upper=1>`の制約を付け、0/1以外が渡された場合にエラーになるようにしています。

**`parameters`ブロック**  
感染確率そのものではなく、log oddsを決める`alpha`と`beta`を推定します。ロジスティック回帰では、確率は`inv_logit(alpha + X * beta)`から導きます。

**`model`ブロック**  
`bernoulli_logit`を使い、個人ごとの感染有無をモデル化します。`bernoulli(inv_logit(...))`と書くこともできますが、`bernoulli_logit`のほうが数値的に安定です。

**`generated quantities`ブロック**  
個人ごとの感染確率`p`、複製データ`y_rep`、オッズ比`odds_ratio`、VE風指標`vaccine_effectiveness_like`、モデル比較用`log_lik`を保存します。ここでの`vaccine_effectiveness_like`は、`X4`の2列目が`vaccinated`であることを前提にしています。

## 5.4.7　RStanで推定

```r
stan_data4 <- list(
  N = nrow(d4),
  K = ncol(X4),
  X = X4,
  y = d4$infected
)

fit4 <- do.call(
  rstan::stan,
  c(
    list(model_code = stan_code_bernoulli_logit, data = stan_data4),
    stan_sampling_args
  )
)

check_fit(fit4, pars = c("alpha", "beta", "odds_ratio", "vaccine_effectiveness_like"))
```

## 5.4.8　出力結果の読み方

```r
summary4_or <- summary(fit4, pars = "odds_ratio")$summary
rownames(summary4_or) <- colnames(X4)
summary4_or[, c("mean", "sd", "2.5%", "50%", "97.5%", "Rhat", "n_eff")]

summary(fit4, pars = "vaccine_effectiveness_like")$summary
```

- `vaccinated`のオッズ比が1より小さい：接種者では感染オッズが低い傾向。
- `household_exposure`のオッズ比が1より大きい：同居内曝露がある人では感染オッズが高い。
- `mask_regular`のオッズ比が1より小さい：マスク着用習慣がある人では感染オッズが低い傾向。

VE風指標の中央値が50なら、「調整済みオッズ比に基づくと、接種により感染オッズが約50%低い」という表現になります。ただし、これは検査陰性デザインなどの適切な研究デザインを仮定した場合に近い解釈であり、単純な観察データでは慎重に扱います。

## 5.4.9　モデルチェックと実応用

```r
post4 <- rstan::extract(fit4)
p_mean4 <- apply(post4$p, 2, mean)

hist(
  p_mean4,
  breaks = 30,
  xlab = "Posterior mean infection probability",
  main = "Predicted infection probabilities"
)

bayesplot::ppc_stat(
  y = d4$infected,
  yrep = post4$y_rep[sample(seq_len(nrow(post4$y_rep)), 100), ],
  stat = "mean"
)
```

実応用では、特定の条件を持つ人の感染確率を予測できます。

```r
# 例：50歳、接種あり、同居内曝露あり、マスク習慣ありの人。
age_mean <- mean(d4$age)
age_sd <- sd(d4$age)
new_age_z <- (50 - age_mean) / age_sd

new_x <- c(
  age_z = new_age_z,
  vaccinated = 1,
  household_exposure = 1,
  mask_regular = 1
)

eta_new <- post4$alpha + as.vector(post4$beta %*% new_x)
p_new <- plogis(eta_new)

quantile(p_new, probs = c(0.025, 0.5, 0.975))
```

このような予測は、個人リスク評価、接触者調査の優先順位付け、検査推奨、隔離・待機期間の意思決定補助に使えます。ただし、個人レベルの予測では、欠測、選択バイアス、測定誤差、集団差、データ取得時点の流行状況が大きく影響します。

---

# 5.5　ポアソン回帰

## 5.5.1　感染症分野での問い

ポアソン回帰は、症例数、死亡数、入院数、集団発生件数などのカウントデータを扱う基本モデルです。感染症分野では、地域ごとに人口が異なるため、単純な症例数比較ではなく、人口を調整した発生率を扱うことが多くあります。

問いは次のようになります。

> 地域・週ごとの人口を考慮した上で、移動量、接種率、時間傾向は症例発生率とどう関係しているか。

## 5.5.2　数式

$$
y_n \sim \mathrm{Poisson}(\lambda_n)
$$

$$
\log(\lambda_n) = \log(\mathrm{population}_n) + \alpha + X_n\beta
$$

または、発生率 $r_n$ を使って、

$$
\lambda_n = \mathrm{population}_n \times r_n
$$

$$
\log(r_n) = \alpha + X_n\beta
$$

と書けます。

ここで、

- $y_n$：地域・週 $n$ の症例数
- $\lambda_n$：期待症例数
- $\mathrm{population}_n$：対象人口
- $\log(\mathrm{population}_n)$：オフセット
- $\exp(\beta_k)$：発生率比、incidence rate ratio

## 5.5.3　数式の仕組み

ポアソン分布は、一定の時間・地域で発生するカウントを扱う基本分布です。期待症例数 $\lambda_n$ は0以上でなければならないため、ログリンクを使います。

人口オフセットの役割は重要です。人口が2倍の地域では、同じ発生率でも期待症例数は約2倍になります。そこで、

$$
\log(\lambda_n) = \log(\mathrm{population}_n) + \log(r_n)
$$

と分解し、人口差を既知の補正項として入れます。

係数 $\beta$ を指数変換すると発生率比になります。たとえば、接種率の係数が$-0.5$なら、

$$
\exp(-0.5) \approx 0.61
$$

なので、接種率が1標準偏差高い地域・週では、発生率が約39%低いという読み方になります。

## 5.5.4　サンプルデータ作成

```r
set.seed(105)

n_area <- 20
n_week <- 8

d5 <- expand.grid(
  area = 1:n_area,
  week = 1:n_week
)

N <- nrow(d5)

area_population <- sample(50000:250000, n_area, replace = TRUE)
d5$population <- area_population[d5$area]

mobility <- rnorm(N, 0, 1)
vacc_coverage <- runif(N, 0.35, 0.85)
vacc_z <- as.numeric(scale(vacc_coverage))
mobility_z <- as.numeric(scale(mobility))
week_z <- as.numeric(scale(d5$week))

X5 <- cbind(
  mobility_z = mobility_z,
  vacc_z = vacc_z,
  week_z = week_z
)

beta_true <- c(
  mobility_z = 0.35,
  vacc_z = -0.50,
  week_z = 0.20
)

alpha_true <- -9.0

eta <- log(d5$population) + as.vector(alpha_true + X5 %*% beta_true)
lambda <- exp(eta)

d5$cases <- rpois(N, lambda)
d5$mobility_z <- mobility_z
d5$vacc_coverage <- vacc_coverage
d5$vacc_z <- vacc_z
d5$week_z <- week_z
d5$incidence_per_100k <- d5$cases / d5$population * 100000

head(d5)
summary(d5$cases)
```

## 5.5.5　Stanコード

```r
stan_code_poisson_regression <- "
data {
  // 地域・週ごとのカウントデータを受け取る。
  // log_offsetは人口の対数。yは症例数。
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  vector[N] log_offset;
  array[N] int<lower=0> y;
}

parameters {
  // alphaは基準ログ発生率。
  // betaは説明変数がログ発生率に与える効果。
  real alpha;
  vector[K] beta;
}

model {
  // 事前分布。
  // alphaは人口オフセット込みの症例数スケールではなく、ログ発生率側の切片。
  // ここではデータ生成に合わせてnormal(-9, 2)としている。
  alpha ~ normal(-9, 2);
  beta ~ normal(0, 1);

  // 尤度。
  // poisson_logはlog(lambda)を直接受け取る。
  // log_offset + alpha + X * beta が各観測のlog(lambda)。
  y ~ poisson_log(log_offset + alpha + X * beta);
}

generated quantities {
  // etaはlog(lambda)、lambdaは期待症例数。
  // y_repは予測症例数、log_likはモデル比較用。
  // incidence_rate_ratioはexp(beta)で発生率比を保存する。
  vector[N] eta;
  vector[N] lambda;
  array[N] int<lower=0> y_rep;
  vector[N] log_lik;
  vector[K] incidence_rate_ratio;

  eta = log_offset + alpha + X * beta;
  incidence_rate_ratio = exp(beta);

  for (n in 1:N) {
    lambda[n] = exp(eta[n]);
    y_rep[n] = poisson_log_rng(eta[n]);
    log_lik[n] = poisson_log_lpmf(y[n] | eta[n]);
  }
}
"
```

## 5.5.6　Stanコードのブロック別意図

**`data`ブロック**  
症例数`y`、説明変数`X`、人口オフセット`log_offset`を渡します。人口は推定するのではなく、既知の曝露量として扱うため、`parameters`ではなく`data`に置きます。

**`parameters`ブロック**  
基準ログ発生率`alpha`と、説明変数の係数`beta`を推定します。ポアソン回帰では、期待症例数`lambda`を直接パラメータにするのではなく、ログリンクを通じて導きます。

**`model`ブロック**  
`poisson_log(log_offset + alpha + X * beta)`により、ログ期待症例数を直接指定します。`poisson(exp(...))`と書くより、`poisson_log`のほうが数値的に安定で、ログ線形モデルとして読みやすいです。

**`generated quantities`ブロック**  
期待症例数`lambda`、複製症例数`y_rep`、発生率比`incidence_rate_ratio`を保存します。`log_lik`は、後で`loo`パッケージを使ってモデル比較を行う場合に必要になります。

## 5.5.7　RStanで推定

```r
stan_data5 <- list(
  N = nrow(d5),
  K = ncol(X5),
  X = X5,
  log_offset = log(d5$population),
  y = d5$cases
)

fit5 <- do.call(
  rstan::stan,
  c(
    list(model_code = stan_code_poisson_regression, data = stan_data5),
    stan_sampling_args
  )
)

check_fit(fit5, pars = c("alpha", "beta", "incidence_rate_ratio"))
```

## 5.5.8　出力結果の読み方

```r
summary5_irr <- summary(fit5, pars = "incidence_rate_ratio")$summary
rownames(summary5_irr) <- colnames(X5)
summary5_irr[, c("mean", "sd", "2.5%", "50%", "97.5%", "Rhat", "n_eff")]
```

- `mobility_z`の発生率比が1より大きい：移動量が高い地域・週ほど症例発生率が高い。
- `vacc_z`の発生率比が1より小さい：接種率が高い地域・週ほど症例発生率が低い。
- `week_z`の発生率比が1より大きい：時間とともに発生率が上昇する傾向がある。

## 5.5.9　モデルチェックと実応用

```r
post5 <- rstan::extract(fit5)
lambda_mean5 <- apply(post5$lambda, 2, mean)

plot(
  lambda_mean5,
  d5$cases,
  xlab = "Posterior mean expected cases",
  ylab = "Observed cases"
)
abline(0, 1)

bayesplot::ppc_stat(
  y = d5$cases,
  yrep = post5$y_rep[sample(seq_len(nrow(post5$y_rep)), 100), ],
  stat = "mean"
)

bayesplot::ppc_stat(
  y = d5$cases,
  yrep = post5$y_rep[sample(seq_len(nrow(post5$y_rep)), 100), ],
  stat = "sd"
)
```

症例数が医療対応能力を超える確率を計算することもできます。

```r
# 例：各地域・週で症例数が30を超える確率。
prob_cases_above_30 <- colMeans(post5$y_rep > 30)

result5 <- data.frame(
  area = d5$area,
  week = d5$week,
  population = d5$population,
  observed_cases = d5$cases,
  posterior_mean_cases = lambda_mean5,
  prob_cases_above_30 = prob_cases_above_30
)

head(result5)
```

このような出力は、地域別の警戒レベル設定、検査資源配分、外来・病床需要の見積もり、保健所支援の優先順位付けに使えます。

ただし、感染症カウントデータでは過分散がよく起こります。ポアソン分布では平均と分散が等しいため、観測データのばらつきが大きすぎる場合は、負の二項回帰、地域ランダム効果、週ランダム効果、時系列構造などを検討します。

---

# Chapter 3　モデルチェックのまとめ

## 3.A　MCMC診断

RStanでモデルを推定したら、まずサンプリングが正常に行われたかを確認します。

```r
print(fit5)
rstan::check_hmc_diagnostics(fit5)
traceplot(fit5, pars = c("alpha", "beta[1]", "beta[2]"))
```

見るべき点は次です。

| 診断 | 望ましい状態 | 問題がある場合の意味 |
|---|---|---|
| R-hat | 1.00に近い。目安として1.01未満 | チェーンが同じ分布に収束していない可能性 |
| n_eff / ESS | 十分大きい | 事後分布の推定が不安定 |
| divergence | 0が望ましい | 事後分布の幾何が難しく、推定が偏る可能性 |
| treedepth | 上限到達が少ない | サンプリング効率が悪い可能性 |
| BFMI | 低すぎない | エネルギー方向の探索が悪い可能性 |

警告が出た場合、反復数を増やすだけでは根本解決にならないことがあります。感染症データでは、外れ値、過分散、ゼロ過剰、未調整の地域差・週差、強すぎる相関、弱すぎる事前分布が原因になることが多いです。

## 3.B　事後予測チェック

事後予測チェックでは、モデルから生成した複製データ`y_rep`と観測データ`y`を比較します。

基本的な考え方は次です。

$$
p(\tilde{y} \mid y) = \int p(\tilde{y} \mid \theta) p(\theta \mid y) d\theta
$$

ここで、$\tilde{y}$はモデルから生成した仮想データです。観測データが、複製データの典型的な範囲から大きく外れているなら、モデルは重要な特徴を再現できていない可能性があります。

例として、ポアソン回帰では平均だけでなく標準偏差を確認します。

```r
bayesplot::ppc_stat(
  y = d5$cases,
  yrep = post5$y_rep[sample(seq_len(nrow(post5$y_rep)), 100), ],
  stat = "mean"
)

bayesplot::ppc_stat(
  y = d5$cases,
  yrep = post5$y_rep[sample(seq_len(nrow(post5$y_rep)), 100), ],
  stat = "sd"
)
```

感染症データで`sd`が再現できない場合、過分散が疑われます。

## 3.C　係数ではなく、意思決定量まで変換する

Stanの出力には`beta`のようなモデル内部の係数が含まれます。しかし、実務で重要なのは、しばしば次のような変換後の量です。

| モデル | Stan上の係数 | 実務上の量 | 変換 |
|---|---|---|---|
| 線形回帰、対数アウトカム | `beta` | 率比 | `exp(beta)` |
| ロジスティック回帰 | `beta` | オッズ比 | `exp(beta)` |
| ワクチン効果風解釈 | `beta_vaccinated` | VE風指標 | `(1 - exp(beta)) * 100` |
| ポアソン回帰 | `beta` | 発生率比 | `exp(beta)` |
| 予測モデル | `y_rep` | しきい値超過確率 | `mean(y_rep > threshold)` |

報告書や論文では、`beta = -0.50`と書くだけでは伝わりにくいことがあります。感染症対策の文脈では、「発生率比0.61」「症例数30超過確率80%」「陽性率10%超過確率90%」のように、意思決定につながるスケールに変換して示すほうが有用です。

## 3.D　感染症データで特に注意する点

この章のモデルは基本形です。実データに適用する際は、少なくとも次の点を検討します。

- 報告遅れ：発症日、検査日、報告日がずれる。
- 曜日効果：週末や休日で検査数・報告数が変わる。
- 検査数の変化：陽性数だけを見ると流行と検査体制の変化が混ざる。
- 検査対象者の選択：症状の強い人だけが検査されると陽性率が高くなる。
- 地域差：人口密度、医療アクセス、年齢構成が異なる。
- 年齢構成：重症化・入院・死亡のモデルでは特に重要。
- 変異株の交代：感染性・免疫逃避・重症度が変化する。
- ワクチン接種選択：接種者と非接種者の背景が異なる。
- 過分散：症例数のばらつきがポアソン分布より大きい。
- ゼロ過剰：症例が0の地域・週が多い。
- 時系列自己相関：今週の症例数は前週の症例数と強く関係する。

次に進むべき拡張は、負の二項回帰、階層モデル、時系列モデル、曜日効果、地域ランダム効果、遅れ効果モデル、測定誤差モデル、事後予測に基づく意思決定分析です。

---

## Chapter 3　補足と文献案内

### RStan・Stan公式資料

- [RStan Reference](https://mc-stan.org/rstan/reference/rstan.html)
- [RStan `stan()` Reference](https://mc-stan.org/rstan/reference/stan.html)
- [RStan `stanfit` class](https://mc-stan.org/rstan/reference/stanfit-class.html)
- [RStan `check_hmc_diagnostics()`](https://mc-stan.org/rstan/reference/check_hmc_diagnostics.html)
- [Stan Reference Manual: Program Blocks](https://mc-stan.org/docs/reference-manual/blocks.html)
- [Stan Reference Manual: Statements](https://mc-stan.org/docs/reference-manual/statements.html)
- [Stan User's Guide: Regression Models](https://mc-stan.org/docs/stan-users-guide/regression.html)
- [Stan User's Guide: Posterior Predictive Checks](https://mc-stan.org/docs/stan-users-guide/posterior-predictive-checks.html)
- [Stan Diagnostics and Warnings](https://mc-stan.org/learn-stan/diagnostics-warnings.html)
- [Stan Functions Reference: Binary Distributions](https://mc-stan.org/docs/functions-reference/binary_distributions.html)
- [Stan Functions Reference: Bounded Discrete Distributions](https://mc-stan.org/docs/functions-reference/bounded_discrete_distributions.html)
- [Stan Functions Reference: Unbounded Discrete Distributions](https://mc-stan.org/docs/functions-reference/unbounded_discrete_distributions.html)

### 感染症分野の参考

感染症監視データを例にするなら、下水監視、検査陽性率、症例数、入院数、ワクチン有効性評価のデータ構造を理解しておくと、Stanモデルに落とし込みやすくなります。

- [CDC Wastewater Surveillance](https://www.cdc.gov/wastewater/index.html)
- [CDC Respiratory Viruses Data](https://www.cdc.gov/respiratory-viruses/data/activity-levels.html)
- [CDC MMWR: COVID-19 Vaccine Effectiveness examples](https://www.cdc.gov/mmwr/)
- [CDC Emerging Infectious Diseases Journal](https://wwwnc.cdc.gov/eid/)

### 次に進むべき内容

Chapter 3までで、RStanによる基本的な回帰モデルは実装できます。次に進むなら、感染症分野では特に次の拡張が重要です。

1. 負の二項回帰：症例数の過分散に対応する。
2. 階層モデル：地域差、施設差、年齢群差を扱う。
3. 時系列モデル：曜日効果、季節性、自己相関、報告遅れを扱う。
4. 遅れ効果モデル：曝露から発症、検査、報告までの遅延を扱う。
5. 測定誤差モデル：検査感度・特異度、下水濃度測定誤差を扱う。
6. 事後予測と意思決定：病床需要、アウトブレイク警戒、介入効果を評価する。
