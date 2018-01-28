## 14. 測定誤差とメタアナリシス

統計モデルで使われる量のほとんどは測定で得られたものです。こうした測定にはほとんどの場合、何らかの誤差があります。測定された量に対して測定誤差が小さければ、モデルには通常あまり影響しません。測定された量に対して測定誤差が大きいときや、測定された量について非常に精密な関係を推定する可能性があるときは、測定誤差を明示的に取り入れたモデルが役に立ちます。測定誤差の一種には丸め誤差もあります。

メタアナリシスは統計的には、測定誤差モデルと非常によく似ています。メタアナリシスでは、複数のデータセットから導かれた推定が、それら全体についての推定にまとめられます。各データセットについての推定は、真のパラメータの値から、一種の測定誤差があるものとして扱われます。

### 14.1. ベイズ測定誤差モデル

ベイズ統計の手法では、真の量を欠測データとして扱うことにより、測定誤差を直接的に定式化できます(Clayton, 1992; Richardson and Gilks, 1993)。これには、真の値から測定値がどのように導かれるのかというモデルが必要です。

#### 測定誤差のある回帰

測定誤差のある回帰を考える前にまず、予測変数$x_n$と結果変数$y_n$を含む、$N$回の観測データの線形回帰モデルを考えましょう。Stanでは、傾きと切片のある、$x$についての$y$の線形回帰は以下のようにモデリングされます。

```
data {
  int<lower=0> N;       // 観測回数
  real x[N];            // 予測変数（共変量）
  real y[N];            // 結果変数（変量）
}
parameters {
  real alpha;           // 切片
  real beta;            // 傾き
  real<lower=0> sigma;  // 結果変数のノイズ
} model {
  y ~ normal(alpha + beta * x, sigma);
  alpha ~ normal(0,10);
  beta ~ normal(0,10);
  sigma ~ cauchy(0,5);
}
```

ここで、予測変数$x_n$の真値が既知ではないとします。ただし、各$n$について、$x_n$の測定値$x_{n}^\mathrm{meas}$は分かっています。測定誤差をモデリングできるならば、測定値$x_{n}^\mathrm{meas}$は、真値$x_n$に測定ノイズを加算したものとモデリングできます。真値$x_n$は欠測データとして扱われ、モデル中の他の量と同時に推定されます。非常に単純な方法としては、測定誤差が既知の偏差$\tau$で正規分布すると仮定する方法があります。以下のような、測定誤差が一定の回帰モデルになります。

```
data {
  ...
  real x_meas[N];     // xの測定値
  real<lower=0> tau;  // 測定ノイズ
}
parameters {
  real x[N];          // 未知の真値
  real mu_x;          // 事前分布の位置
  real sigma_x;       // 事前分布のスケール
  ...
}
model {
  x ~ normal(mu_x, sigma_x);   // 事前分布
  x_meas ~ normal(x, tau);   // 測定モデル
  y ~ normal(alpha + beta * x, sigma);
  ...
}
```

回帰係数の`alpha`と`beta`、回帰ノイズのスケール`sigma`は前と同じですが、データではなくパラメータとして`x`が新たに宣言されています。データは`x_meas`になり、真の`x`の値からスケール`tau`のノイズを含めて測定されているとなっています。そしてこのモデルは、真値`x[n]`についての測定誤差`x_meas[n]`が偏差`tau`の正規分布に従うと指定しています。さらに真値`x`にはここでは階層事前分布が与えられています。

測定誤差が正規分布でない場合には、もっと複雑な測定誤差モデルを指定することもできます。真値の事前分布も複雑にできます。例えば、Clayton (1992)は、既知の（測定誤差のない）リスク要因$c$に対する、未知の（ただしノイズ込みで測定された）リスク要因$x$についての暴露モデルを紹介しています。単純なモデルでは、共変量$c_n$とノイズ項$\upsilon$から$x_n$を回帰するというようになるでしょう。

![$$x_{n} \sim \mathsf{Normal}(\gamma^{\top}c,\upsilon)$$](fig/fig01.png)

これはStanでは、ほかの回帰とまったく同様にコーディングできます。もちろん、さらにほかの暴露モデルも使えます。

#### 丸め

測定誤差でよくあるのは、測定値を丸めることに由来するものです。丸めのやり方はたくさんあります。重さをもっとも近いミリグラムの値に丸めたり、もっとも近いポンドの値に丸めたりします。もっとも近い整数に切り下げるという丸めもあります。

Gelman et al. (2013)の演習3.5(b)に以下の例題があります。

> 3.5 ある物体の重さを5回はかるとします。測定値はもっとも近いポンドの値に丸められ、10, 10, 12, 11, 9となりました。丸める前の測定値は正規分布するとして、$\mu$と$\sigma^2$には無情報事前分布を使います。
> (b) 測定値が丸められていることを考慮して、$(\mu, \sigma^2)$についての正しい事後分布を求めなさい。

$y_n$の丸められていない測定値を$z_n$とします。すると、述べられている問題は以下の尤度を仮定することになります。

![$$z_{n} \sim \mathsf{Normal}(\mu,\sigma)$$](fig/fig02.png)

丸めの過程により、$z_n \in (y_n - 0.5, y_n + 0.5)$となります。離散値の観測値$y$の確率質量関数は、丸められていない測定値を周辺化消去することにより与えられ、以下の尤度が得られます。

![$$p(y_{n}\mid\mu,\sigma)=\int_{y_{n}-0.5}^{y_{n}+0.5}\mathsf{Normal}(z_{n}\mid\mu,\sigma)\mathrm{d}z_{n}=\Phi\left(\frac{y_{n}+0.5-\mu}{\sigma}\right)-\Phi\left(\frac{y_{n}-0.5-\mu}{\sigma}\right)$$](fig/fig03.png)

この問題についてのGelmanの解答では、分散$\sigma^2$について対数スケールで一様分布の無情報事前分布を使いました。このときの事前密度は（ヤコビアンの調整により）以下のようになります。

![$$p(\mu,\sigma^2) \propto \frac{1}{\sigma^2}$$](fig/fig04.png)

$y=(10,10,12,11,9)$を観測した後の事後分布は、ベイズの定理により計算できます。

![$$\begin{array}{ll}p(\mu,\sigma^2\mid y) &\propto p(\mu,\sigma^2)p(y\mid\mu,\sigma^2)\\ &\propto \frac{1}{\sigma^2}\prod_{n=1}^{5}\left(\Phi\left(\frac{y_{n}+0.5-\mu}{\sigma}\right)-\Phi\left(\frac{y_{n}-0.5-\mu}{\sigma}\right)\right) \end{array}$$](fig/fig05.png)

Stanのコードは単純に数学的定義に従っており、確率関数をそのまま定義したものを、割合にするまでの例となっています。

```
data {
  int<lower=0> N;
  vector[N] y;
}
parameters {
  real mu;
  real<lower=0> sigma_sq;
}
transformed parameters {
  real<lower=0> sigma;
  sigma <- sqrt(sigma_sq);
}
model {
  increment_log_prob(-2 * log(sigma));
  for (n in 1:N)
    increment_log_prob(log(Phi((y[n] + 0.5 - mu) / sigma)
                           - Phi((y[n] - 0.5 - mu) / sigma)));
}
```

別のやり方として、丸められていない測定値$z_n$についての潜在パラメータを使ってモデルを定義する方法もあるでしょう。この場合のStanのコードは、制約$z_n \in (y_n - 0.5, y_n + 0.5)$を満たしつつ$z_n$の尤度をそのまま使います。Stanでは、ベクトル（あるいは配列）の要素についての上下限は不変でなくてなりませんので、そのパラメータは丸め誤差$y - z$として宣言され、$z$は`transformed parameter`として定義されます。

```
data {
  int<lower=0> N;
  vector[N] y;
}
parameters {
  real mu;
  real<lower=0> sigma_sq;
  vector<lower=-0.5, upper=0.5>[5] y_err;
}
transformed parameters {
  real<lower=0> sigma;
  vector[N] z;
  sigma <- sqrt(sigma_sq);
  z <- y + y_err;
}
model {
  increment_log_prob(-2 * log(sigma));
  z ~ normal(mu, sigma);
}
```

丸められていない測定値$z$を明示したこのモデルは、$z$を周辺化消去した前のモデルと、$\mu$と$\sigma$について同じ事後分布を生成します。どちらのやり方とも連鎖が良く混ざりますが、潜在パラメータを使うバージョンは、iterationあたりの有効サンプルの点で2倍ほど効率的です。また、丸められていないパラメータの事後分布も得られます。

### 14.2. メタアナリシス

メタアナリシスは、いくつかの学校での指導プログラムの利用や、いくつかの臨床試験での薬を使った治療といった、いくつかの研究からのデータをプールすることを目的としています。

ベイズの枠組みはメタアナリシスには特に便利です。というのは、興味ある真の量をノイズ込みで測定したものとして、各先行研究を扱うことができるからです。このとき、モデルはそのまま2つの部分から成ります。つまり、興味ある真の量についての事前分布と、解析する研究のそれぞれについての測定誤差タイプのモデルです。

#### 対照群と比較する臨床試験における治療の効果

治療群と対照群について対応のある二項分布のデータが得られている研究が全部で$M$個あるところから問題のデータが得られているとします。例えばデータは、イブプロフェン投与下での外科手術後の痛みの軽減や(Warn et al., 2002)、ベータブロッカー投与下での心筋梗塞後の死亡率(Gelman et al, 2013, Section 5.6)といったものでしょう。

##### データ

この臨床データは$J$個の臨床試験から成り立っています。それぞれの臨床試験について、治療群に割り付けられた人数は$n^t$、対照群に割り付けられた人数は$n^c$、治療群で改善した（成功した）人数は$r^t$、対照群で改善した（成功した）人数は$r^c$です。このデータはStanでは以下のように宣言できます。<sup>1</sup>

```
data {
  int<lower=0> J;
  int<lower=0> n_t[J];  // 治療例の数
  int<lower=0> r_t[J];  // 治療群での成功数
  int<lower=0> n_c[J];  // 対照例の数
  int<lower=0> r_c[J];  // 対照群での成功数
}
```

<sup>1</sup> Stanの整数の制約は、`r_t[h]` $le$ `n_t[j]`という制約を表現できるほど強力ではありません。しかしこの制約は、`transformed data block`でチェックすることができるでしょう。

##### 対数オッズへの変換と標準誤差

この臨床試験データは、そのままでは2項分布のフォーマットですが、対数オッズ比を考えると、限度のない軸に変換できるでしょう。

![$$y_{j}=\log\left(\frac{r^{t}_{j}/(n^{t}_{j}-r^{t}_{j})}{r^{c}_{j}/(n^{c}_{j}-r^{c}_{j})}\right)=\log\left(\frac{r^{t}_{j}}{n^{t}_{j}-r^{t}_{j}}\right)-\log\left(\frac{r^{c}_{j}}{n^{c}_{j}-r^{c}_{j}}\right)$$](fig/fig06.png)

対応する標準誤差です。

![$$\sigma_{j}=\sqrt{\frac{1}{r^t_j}+\frac{1}{n^t_j-r^t_j}+\frac{1}{r^c_j}+\frac{1}{n^c_j-r^c_j}}$$](fig/fig07.png)

対数オッズと標準誤差は`transformed parameters`ブロックで定義できますが、整数除算にならないように注意が必要です（39.1節参照）。

```
transformed data {
  real y[J];
  real<lower=0> sigma[J];
  for (j in 1:J)
    y[j] <- log(r_t[j]) - log(n_t[j] - r_t[j])
            - (log(r_c[j]) - log(n_c[j] - r_c[j]);
  for (j in 1:J)
    sigma[j] <- sqrt(1.0/r_t[j] + 1.0/(n_t[j] - r_t[j])
                     + 1.0/r_c[j] + 1.0/(n_c[j] - r_c[j]));
}
```

いずれかの成功数が0だったり、試行回数と同じだったりすると、この定義には問題が発生します。そうなる場合には、2項分布で直接モデリングするか、正則化しない標本対数オッズを使うのではなく、別の変換を使う必要があります。

##### 非階層モデル

変換したデータができたら、二つのメタアナリシスの標準型を使うことができます。最初は、いわゆる「固定効果」モデルで、全体的なオッズ比について単一のパラメータを仮定します。このモデルはStanでは以下のようにコーディングされます。

```
parameters {
  real theta;  // 全体的な治療の効果, 対数オッズ
}
model {
  y ~ normal(theta,sigma);
}
```

`y`についてのサンプリング文はベクトル化されており、以下と同じ効果があります。

```
for (j in 1:J)
  y[j] ~ normal(theta,sigma[j]);
```

このモデルには`theta`の事前分布を含めるのが普通ですが、`y`が固定されており、$\mathsf{Normal}(y\mid\theta,\sigma) = \mathsf{Normal}(\theta\mid y,\sigma)$ですので、モデルを正則にするためにどうしても必要というわけではありません。

##### 階層モデル

治療の効果が臨床試験によって変動しうる、いわゆる「ランダム効果」をモデリングするには、階層モデルを使うことができます。パラメータには、試験ごとの治療の効果と、階層事前分布のパラメータを含め、ほかの未知の量をともに推定します。

```
parameters {
  real theta[J];      // 試験ごとの治療の効果
  real mu;            // 平均の治療の効果
  real<lower=0> tau;  // 治療の効果の偏差
}
model {
  y ~ normal(theta,sigma);
  theta ~ normal(mu,tau);
  mu ~ normal(0,10);
  tau ~ cauchy(0,5);
}
```

ベクトル化した`y`のサンプリング文は変化ないように見えますが、パラメータ`theta`はベクトルになっています。`theta`のサンプリング文もベクトル化されており、超パラメータ`mu`と`tau`はそれ自身が、データのスケールと比較して幅の広い事前分布を与えられています。

Rubin (1981)は、8つの学校での大学進学適性試験(Scholatic Aptitude Test, SAT)の指導の処理効果に関して、各学校での標本処理効果と標準誤差に基づいて、階層ベイズでメタアナリシスを行なっています。<sup>2</sup>

<sup>2</sup> Gelman et al. (2013) 5.5節のこのデータについてのモデルは、Stan example modelリポジトリ, http://mc-stan.org/documentation にデータとともに入っています。

##### 拡張と代替法

Smith et al. (1995)とGelman et al. (2013, Section 19.4)には、二項データそのままに基づいたメタアナリシスがあります。Warn et al. (2002)は、二項データの変換の際に対数オッズ比の代替法を使うと、モデリングにどのような影響があるかを考察しています。

試験に特有の予測変数が利用できるならば、試験ごとの治療の効果$\theta_j$として回帰モデルに直接含めることができます。