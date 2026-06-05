---
title: "RStan学習ノート：Chapter 1〜3"
author: ""
date: "2026-06-05"
lang: ja
---

# RStan学習ノート：Chapter 1〜3

本ノートは、RStan公式リファレンス、Stan Reference Manual、Stan User's Guideを確認した上で、RStanを用いた統計モデリングの導入から基本的な回帰モデルの実装までをまとめたものです。Chapter 3では、感染症分野で実際に頻出するデータ構造を意識したサンプルデータとRStanコードを提示します。サンプルデータは再現可能なシミュレーションデータであり、実患者データではありません。

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

## この章の前提

この章では、感染症疫学でよく出るデータ形式を使って、RStanで基本的な回帰モデルを実装します。使うデータは**再現可能なサンプルデータ**です。実患者データや公開監視データそのものではありませんが、列構造とモデルの考え方は、感染症分野で実際に使われる分析に近づけています。

| 節 | モデル | 感染症分野での例 |
|---|---|---|
| 5.1 | 単回帰 | 下水中ウイルスRNA濃度と地域感染者率 |
| 5.2 | 重回帰 | 入院率と症例数、ワクチン接種率、移動量 |
| 5.3 | 二項ロジスティック回帰 | 検査陽性数 / 検査数、検査陽性率 |
| 5.4 | ロジスティック回帰 | 個人単位の感染有無、ワクチン接種、曝露 |
| 5.5 | ポアソン回帰 | 地域別・週別症例数、人口オフセット |

CDCは感染症を下水で監視するプログラムを運用しており、下水監視は感染拡大に早く対応するためのデータ源として位置づけられています。CDCはCOVID-19、インフルエンザ、RSVなどについて週次の検査陽性割合も監視指標として示しています。また、COVID-19ワクチン有効性評価では、多変量ロジスティック回帰でオッズ比を推定する解析が使われています。感染症の症例数解析では、人口をオフセットに入れたポアソン回帰で率を扱う例もあります。

## 共通準備

以下をRスクリプトの最初に実行します。

```r
library(rstan)
library(bayesplot)

options(mc.cores = parallel::detectCores())
rstan_options(auto_write = TRUE)

set.seed(2026)
```

共通のサンプリング設定も作っておきます。

```r
stan_sampling_args <- list(
  chains = 4,
  iter = 2000,
  warmup = 1000,
  seed = 2026,
  refresh = 0,
  control = list(adapt_delta = 0.95)
)
```

`adapt_delta = 0.95`は、divergenceを減らすために使われることがあります。ただし、Stanの診断ガイドでは、`adapt_delta`や`max_treedepth`を上げることは最後の手段に近く、まずはモデル、事前分布、パラメータ化、数値安定性を確認することが推奨されています。

共通の診断関数も用意します。

```r
check_fit <- function(fit, pars) {
  print(fit, pars = pars, probs = c(0.025, 0.5, 0.975))
  rstan::check_hmc_diagnostics(fit)
}
```

RStanの`check_hmc_diagnostics()`は、divergence、treedepth、E-BFMIなどのHMC診断をまとめて表示する関数です。Stanの診断では、R-hat、ESS、divergent transitions、maximum treedepth、BFMIを見る必要があります。

## 5.1　単回帰

### 5.1.1　問題設定

例として、週別の下水中ウイルスRNA濃度と、同じ地域の週別COVID-19症例率の関係を考えます。下水中ウイルス濃度は地域内感染の早期指標として使われることがあり、ここでは簡略化して、標準化した下水指標から対数症例率を説明します。

モデルは次です。

\[
y_n \sim \mathrm{Normal}(\alpha + \beta x_n, \sigma)
\]

ここで、

- \(y_n\)：週\(n\)の対数症例率
- \(x_n\)：週\(n\)の標準化済み下水ウイルス指標
- \(\alpha\)：切片
- \(\beta\)：下水指標1標準偏差あたりの対数症例率の変化
- \(\sigma\)：観測誤差

### 5.1.2　サンプルデータ作成

```r
set.seed(101)

N <- 40
week <- 1:N

# 下水中ウイルスRNA濃度を想定した指標
wastewater_raw <- exp(
  2.5 +
    0.03 * week +
    sin(2 * pi * week / 12) * 0.5 +
    rnorm(N, 0, 0.35)
)

# 解析では対数化して標準化する
log_wastewater <- log(wastewater_raw)
x <- as.numeric(scale(log_wastewater))

# 真の生成過程：下水指標が高いほど症例率も高い
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

### 5.1.3　Stanコード

```r
stan_code_simple_lm <- "
data {
  int<lower=1> N;
  vector[N] x;
  vector[N] y;
}

parameters {
  real alpha;
  real beta;
  real<lower=0> sigma;
}

model {
  alpha ~ normal(0, 5);
  beta ~ normal(0, 2);
  sigma ~ exponential(1);

  y ~ normal(alpha + beta * x, sigma);
}

generated quantities {
  vector[N] mu;
  array[N] real y_rep;

  for (n in 1:N) {
    mu[n] = alpha + beta * x[n];
    y_rep[n] = normal_rng(mu[n], sigma);
  }
}
"
```

### 5.1.4　RStanで推定

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

check_fit(fit1, pars = c("alpha", "beta", "sigma"))
```

### 5.1.5　モデルチェック

```r
# 事後予測ドロー
post1 <- rstan::extract(fit1)
y_rep1 <- post1$y_rep

# 観測値と事後予測の分布比較
bayesplot::ppc_dens_overlay(
  y = d1$log_cases_per_100k,
  yrep = y_rep1[sample(seq_len(nrow(y_rep1)), 50), ]
)

# 予測平均と観測値の比較
mu_mean1 <- apply(post1$mu, 2, mean)

plot(
  mu_mean1,
  d1$log_cases_per_100k,
  xlab = "Posterior mean of fitted log case rate",
  ylab = "Observed log case rate"
)
abline(0, 1)
```

事後予測チェックでは、観測データと、モデルから生成した複製データ`y_rep`が似た特徴を持つかを確認します。

### 5.1.6　解釈

`beta`の事後中央値が正で、95%信用区間の大部分が0より大きければ、「下水指標が高い週ほど対数症例率も高い」という関係を支持します。

ただし、この単回帰は因果効果を示すものではありません。検査体制、報告遅れ、季節性、免疫状態、変異株、行動変化などを調整していないため、あくまで監視指標間の関連を見るモデルです。

## 5.2　重回帰

### 5.2.1　問題設定

次に、週別の呼吸器感染症入院率を、複数の説明変数で説明するモデルを考えます。感染症疫学では、単一の説明変数だけでなく、症例数、ワクチン接種率、移動量、変異株流行期、季節性などを同時に調整したい場面が多くあります。

モデルは次です。

\[
y_n \sim \mathrm{Normal}(\alpha + X_n \beta, \sigma)
\]

ここで、

- \(y_n\)：週別の対数入院率
- \(X_n\)：説明変数ベクトル
- \(\beta\)：各説明変数の係数

### 5.2.2　サンプルデータ作成

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

### 5.2.3　Stanコード

```r
stan_code_multiple_lm <- "
data {
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  vector[N] y;
}

parameters {
  real alpha;
  vector[K] beta;
  real<lower=0> sigma;
}

model {
  alpha ~ normal(0, 5);
  beta ~ normal(0, 2);
  sigma ~ exponential(1);

  y ~ normal(alpha + X * beta, sigma);
}

generated quantities {
  vector[N] mu;
  array[N] real y_rep;

  mu = alpha + X * beta;

  for (n in 1:N) {
    y_rep[n] = normal_rng(mu[n], sigma);
  }
}
"
```

### 5.2.4　RStanで推定

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

check_fit(
  fit2,
  pars = c("alpha", "beta", "sigma")
)
```

### 5.2.5　係数名を付けて要約する

```r
summary2 <- summary(fit2, pars = "beta")$summary
rownames(summary2) <- colnames(X2)
summary2[, c("mean", "sd", "2.5%", "50%", "97.5%", "Rhat", "n_eff")]
```

### 5.2.6　モデルチェック

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
  xlab = "Posterior mean of fitted log hospitalization rate",
  ylab = "Observed log hospitalization rate"
)
abline(0, 1)
```

### 5.2.7　解釈

`beta[log_cases_z]`が正なら、症例数が多い週ほど入院率も高い、という関係を示します。`beta[vacc_z]`が負なら、ワクチン接種率が高い週では、他の変数を調整した上で入院率が低い傾向を示します。

ただし、このモデルは観察データの回帰です。ワクチン効果そのものを推定するには、交絡、時変交絡、接種選択、年齢構成、既感染、医療アクセス、検査行動などをより丁寧に扱う必要があります。

## 5.3　二項ロジスティック回帰

### 5.3.1　問題設定

ここでは、週別の「検査陽性数 / 検査数」を扱います。たとえば、インフルエンザ、COVID-19、RSVの監視では、検査数と陽性数から陽性率を計算します。このような集計データでは、陽性率そのものを正規分布で扱うより、陽性数を二項分布で扱うほうが自然です。

\[
y_n \sim \mathrm{Binomial}(N_n, p_n)
\]

\[
\mathrm{logit}(p_n) = \alpha + X_n \beta
\]

ここで、

- \(y_n\)：週\(n\)の陽性数
- \(N_n\)：週\(n\)の検査数
- \(p_n\)：週\(n\)の陽性確率

### 5.3.2　サンプルデータ作成

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

### 5.3.3　Stanコード

```r
stan_code_binomial_logit <- "
data {
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  array[N] int<lower=0> tested;
  array[N] int<lower=0> positive;
}

parameters {
  real alpha;
  vector[K] beta;
}

model {
  alpha ~ normal(0, 3);
  beta ~ normal(0, 1.5);

  positive ~ binomial_logit(tested, alpha + X * beta);
}

generated quantities {
  vector[N] eta;
  vector[N] p;
  array[N] int<lower=0> y_rep;
  vector[N] log_lik;

  eta = alpha + X * beta;

  for (n in 1:N) {
    p[n] = inv_logit(eta[n]);
    y_rep[n] = binomial_rng(tested[n], p[n]);
    log_lik[n] = binomial_lpmf(positive[n] | tested[n], p[n]);
  }
}
"
```

### 5.3.4　RStanで推定

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

check_fit(fit3, pars = c("alpha", "beta"))
```

### 5.3.5　陽性率の予測とモデルチェック

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

# 複製陽性率
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

### 5.3.6　解釈

二項ロジスティック回帰では、係数はロジットスケールです。`exp(beta[k])`を取ると、説明変数が1単位増えたときのオッズ比になります。

```r
beta_draws3 <- post3$beta
or_draws3 <- exp(beta_draws3)

or_summary3 <- apply(
  or_draws3,
  2,
  quantile,
  probs = c(0.025, 0.5, 0.975)
)

colnames(or_summary3) <- colnames(X3)
or_summary3
```

`ili_z`のオッズ比が1より大きければ、ILI活動が高い週ほど検査陽性確率が高いことを示します。`vacc_z`のオッズ比が1より小さければ、接種率が高い週ほど陽性率が低い傾向を示します。ただし、これは集計データの関連であり、個人単位のワクチン有効性をそのまま意味しません。

## 5.4　ロジスティック回帰

### 5.4.1　問題設定

ここでは、個人単位の感染有無を扱います。たとえば、検査陰性デザイン、症例対照研究、血清疫学調査、接触者調査などでは、個人ごとに感染あり/なし、ワクチン接種あり/なし、濃厚接触あり/なし、マスク着用、年齢などを扱います。

\[
y_i \sim \mathrm{Bernoulli}(p_i)
\]

\[
\mathrm{logit}(p_i) = \alpha + X_i \beta
\]

### 5.4.2　サンプルデータ作成

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

### 5.4.3　Stanコード

```r
stan_code_bernoulli_logit <- "
data {
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  array[N] int<lower=0, upper=1> y;
}

parameters {
  real alpha;
  vector[K] beta;
}

model {
  alpha ~ normal(0, 3);
  beta ~ normal(0, 1.5);

  y ~ bernoulli_logit(alpha + X * beta);
}

generated quantities {
  vector[N] eta;
  vector[N] p;
  array[N] int<lower=0, upper=1> y_rep;
  vector[K] odds_ratio;
  vector[N] log_lik;

  eta = alpha + X * beta;
  odds_ratio = exp(beta);

  for (n in 1:N) {
    p[n] = inv_logit(eta[n]);
    y_rep[n] = bernoulli_rng(p[n]);
    log_lik[n] = bernoulli_lpmf(y[n] | p[n]);
  }
}
"
```

### 5.4.4　RStanで推定

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

check_fit(fit4, pars = c("alpha", "beta", "odds_ratio"))
```

### 5.4.5　オッズ比を名前付きで要約

```r
post4 <- rstan::extract(fit4)

or_summary4 <- apply(
  post4$odds_ratio,
  2,
  quantile,
  probs = c(0.025, 0.5, 0.975)
)

colnames(or_summary4) <- colnames(X4)
or_summary4
```

### 5.4.6　モデルチェック

```r
p_mean4 <- apply(post4$p, 2, mean)

# 予測確率の分布
hist(
  p_mean4,
  breaks = 30,
  xlab = "Posterior mean infection probability",
  main = "Predicted infection probabilities"
)

# 観測感染率と事後予測感染率の比較
bayesplot::ppc_stat(
  y = d4$infected,
  yrep = post4$y_rep[sample(seq_len(nrow(post4$y_rep)), 100), ],
  stat = "mean"
)

# 簡易的な分類性能の確認。推論の主目的ではなく、確認用。
pred_class <- ifelse(p_mean4 > 0.5, 1, 0)
mean(pred_class == d4$infected)
```

### 5.4.7　解釈

`vaccinated`のオッズ比が1より小さければ、他の説明変数を調整した上で、接種者の感染オッズが低い傾向を示します。`household_exposure`のオッズ比が1より大きければ、同居内曝露が感染オッズを上げる傾向を示します。

COVID-19ワクチン有効性解析では、症例患者と対照患者における接種有無を比較し、多変量ロジスティック回帰でオッズ比を推定し、ワクチン有効性を\((1 - \mathrm{adjusted\ OR}) \times 100\%\)として計算する方法が用いられます。ベイズモデルでも、接種効果をワクチン有効性風に表すなら、事後ドローごとに次のように計算できます。

```r
# vaccinated列の位置
j_vacc <- which(colnames(X4) == "vaccinated")

ve_draws <- (1 - exp(post4$beta[, j_vacc])) * 100

quantile(ve_draws, probs = c(0.025, 0.5, 0.975))
```

ただし、このVE風の値は、ここで作った単純なサンプルデータとモデルに対する計算です。実データでワクチン有効性を推定する場合は、研究デザイン、症例定義、検査陰性デザイン、交絡調整、カレンダー時間、地域、年齢、既感染、接種からの経過時間などを明示的に扱う必要があります。

## 5.5　ポアソン回帰

### 5.5.1　問題設定

ポアソン回帰は、症例数、死亡数、入院数、アウトブレイク件数など、カウントデータを扱う基本モデルです。感染症疫学では、地域ごとの人口が異なるため、人口をオフセットとして入れ、症例数を率として解釈することがよくあります。

\[
y_n \sim \mathrm{Poisson}(\lambda_n)
\]

\[
\log(\lambda_n) = \log(\mathrm{population}_n) + \alpha + X_n \beta
\]

ここで、\(\log(\mathrm{population}_n)\)はオフセットです。

### 5.5.2　サンプルデータ作成

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

### 5.5.3　Stanコード

```r
stan_code_poisson_regression <- "
data {
  int<lower=1> N;
  int<lower=1> K;
  matrix[N, K] X;
  vector[N] log_offset;
  array[N] int<lower=0> y;
}

parameters {
  real alpha;
  vector[K] beta;
}

model {
  alpha ~ normal(-9, 2);
  beta ~ normal(0, 1);

  y ~ poisson_log(log_offset + alpha + X * beta);
}

generated quantities {
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

### 5.5.4　RStanで推定

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

### 5.5.5　発生率比を要約する

```r
post5 <- rstan::extract(fit5)

irr_summary5 <- apply(
  post5$incidence_rate_ratio,
  2,
  quantile,
  probs = c(0.025, 0.5, 0.975)
)

colnames(irr_summary5) <- colnames(X5)
irr_summary5
```

`exp(beta)`は発生率比、つまりincidence rate ratioです。たとえば、`vacc_z`の発生率比が1より小さければ、ワクチン接種率が高い地域・週ほど、人口を調整した症例発生率が低い傾向を示します。

### 5.5.6　モデルチェック

```r
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

ポアソンモデルでは、平均と分散が等しいという制約があります。感染症データでは、地域差、クラスター、報告遅れ、集団発生により過分散が出やすいため、`sd`の事後予測チェックが重要です。過分散が大きい場合は、負の二項回帰、地域ランダム効果、時間効果、階層モデルへの拡張を検討します。

## Chapter 3　モデルチェックのまとめ

RStanで回帰モデルを推定したら、最低限、次を確認します。

### 1. MCMC診断

```r
print(fit5)
rstan::check_hmc_diagnostics(fit5)
traceplot(fit5, pars = c("alpha", "beta[1]", "beta[2]"))
```

見るべきものは、R-hat、ESS、divergence、maximum treedepth、BFMIです。Stanの診断ガイドでは、警告の多くはモデル上の重要な問題を示しており、警告を見た場合は意味を理解するまで推定値をそのまま信頼すべきではないと説明されています。

### 2. 事後予測チェック

各モデルで`generated quantities`に`y_rep`を入れました。これにより、観測データとモデルから生成した複製データを比較できます。

```r
post <- rstan::extract(fit5)
y_rep <- post$y_rep

bayesplot::ppc_stat(
  y = d5$cases,
  yrep = y_rep[sample(seq_len(nrow(y_rep)), 100), ],
  stat = "mean"
)

bayesplot::ppc_stat(
  y = d5$cases,
  yrep = y_rep[sample(seq_len(nrow(y_rep)), 100), ],
  stat = "sd"
)
```

もし元データを複製データの中から簡単に見分けられるなら、モデルに問題がある可能性があります。

### 3. 係数の解釈

線形回帰では、係数はアウトカムのスケールで解釈します。

```r
# 線形回帰
beta
```

ロジスティック回帰では、指数変換してオッズ比にします。

```r
# ロジスティック回帰
odds_ratio = exp(beta)
```

ポアソン回帰では、指数変換して発生率比にします。

```r
# ポアソン回帰
incidence_rate_ratio = exp(beta)
```

### 4. 感染症データで特に注意する点

感染症データでは、次の問題が頻繁に起こります。

- 報告遅れ
- 検査数の変化
- 検査対象者の選択バイアス
- 曜日効果
- 季節性
- 地域差
- 年齢構成の違い
- 変異株の交代
- ワクチン接種選択による交絡
- 過分散
- ゼロ過剰
- 時系列自己相関

この章のモデルは基本形です。実務では、階層モデル、時系列モデル、負の二項回帰、曜日効果、地域ランダム効果、遅れ効果、測定誤差モデルなどへ拡張することが多くなります。

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
