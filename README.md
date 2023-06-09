# 概要
画像処理においてもよく使用されるクラスタリングアルゴリズムの一つである**K-平均法**を使って、画像分析する方法をまとめてます。
K-平均法で分析することで、その画像には「どんな種類の色が含まれているか」知ることができます。

## そもそもK-平均法とは？
**K-平均法（K-means clustering）** は、クラスタリングの手法の1つで、与えられたデータセットをK個のクラスタに分割するアルゴリズムです。

具体的には、まずK個の中心点をランダムに設定します。その後、各データポイントがそれぞれ最も近い中心点に属するようにクラスタに割り当てます。次に、各クラスタの中心を再計算し、その中心を新しい中心点として使い、再度クラスタリングを行います。このプロセスを繰り返し、各中心点が変化しなくなるまで続けます。

K-平均法はマーケティング調査や自然言語処理にも使われますが画像処理でも使うことができます。画像中のピクセル値をベクトル化し、そのベクトルを元に似た特徴を持つ画像をクラスタリングすることができます。

知っての通り、画像はRGBデータの集合体です。色自体はRGBの256×256×256=16,777,216種類あります。これをK-平均法で大体どれくらいの色が含まれているのかを知ることができます。

例えば以下のような画像があります。
![fresh-fruits](data/sample/fresh-fruits-2305192_960_720.jpg)

これを10個の色で分けたとき、以下のような絵画っぽい画像を作ることができます。
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k10_replaced.jpg)

# ソースコード
## ライブラリのインストール
必要なライブラリは以下でインストールしてください。
インストール後、`python3 color_separate_rgb.py`で実行可能です。
``` bash
pip3 install -r requirements.txt
```


## 画像のパス、名前、切り抜く範囲を指定する
設定しやすい用に、先頭に定数（厳密な意味では定数でないが）として置いている。
```Python
IMG_PATH = "data/sample/Google-Logo.jpg"
# IMG_PATH = "data/sample/sea-free-photo5.jpg"
# IMG_PATH = "data/sample/fresh-fruits-2305192_960_720.jpg"
```
`IMG_PATH`で画像ファイルを指定する。サンプルで3枚画像を用意している。
- Googleのロゴ
- 風景の写真
- フルーツの写真


```Python
IMG_NAME = IMG_PATH.split("/")[-1]
START_X, START_Y = 0, 0
RANGE_X, RANGE_Y = 1500, 1000
NUMBER_OF_CLUSTERS = 5
```
- `IMG_NAME`で、画像ファイル名を指定する。
- `START_X, START_Y`で、切り抜き位置を指定する。
- `RANGE_X, RANGE_Y`で、どのくらいのピクセル分切り取るか指定する。
- `NUMBER_OF_CLUSTERS`で、K-平均のクラスタ数を指定する。例えば、5と指定すれば5つの色に分類します。数値が大きくなるほど計算量が多くなるため、時間がかかります。

## 画像を読み込む
```Python
# 読み込む画像を指定する
class SetLoadingImage:
    def __init__(self, img_path=IMG_PATH):
        self.img = self.read_image(img_path)

    def read_image(self, img_path):
        try:
            read_img = cv2.imread(img_path)
            img = cv2.cvtColor(read_img, cv2.COLOR_BGR2RGB)
            return img
        except cv2.error:
            print("画像読み込みエラーのため、終了します")
            sys.exit()

    @property
    def return_img(self):
        return self.img
```

## 切り抜き範囲を指定する
```Python
class SpecifyAnalysisRange:
    def __init__(self, img):
        self.org_img = img
        self.drawed_img = np.copy(img)

    def clip(
        self,
        start_x=START_X,
        start_y=START_Y,
        draw_range_x=RANGE_X,
        draw_range_y=RANGE_Y,
    ):
        self.clipped = self.drawed_img[
            start_y : start_y + draw_range_y, start_x : start_x + draw_range_x
        ]
        self.include_square_img = cv2.rectangle(
            img=self.org_img,
            pt1=(start_x, start_y),
            pt2=(start_x + draw_range_x, start_y + draw_range_y),
            color=(255, 0, 0),
            thickness=2,
        )
        return self.clipped

    @property
    def overall_image(self):
        self.clip()
        return self.include_square_img

    @property
    def clipped_image(self):
        return self.clipped
```


## K-平均法

```Python
# K-Means法で画像を分析
class KMeansAnalyzer:
    def __init__(self, img):
        self.img = img  # 切り抜き範囲の画像を代入
        self.number_of_cluster = NUMBER_OF_CLUSTERS  # クラスタ数

    # K平均法で計算する
    def analyze(self) -> pd.DataFrame:
        colors = self.img.reshape(-1, 3).astype(
            np.float32
        )  # 画像で使用されている色一覧。(W * H, 3) の numpy 配列。

        # K-meansアルゴリズムの収束基準を設定
        criteria = cv2.TERM_CRITERIA_MAX_ITER + cv2.TERM_CRITERIA_EPS, 10, 1.0

        _, labels, rgb_value = cv2.kmeans(
            data=colors,  # クラスタリングするための入力データ
            K=self.number_of_cluster,  # クラスタ数
            bestLabels=None,  # クラスタ番号の初期値(通常はNone)
            criteria=criteria,  # アルゴリズムの収束基準
            attempts=10,  # 異なる初期値でアルゴリズムを実行する回数
            flags=cv2.KMEANS_RANDOM_CENTERS,  # クラスタリングアルゴリズムのフラグ
        )

        self.labels = labels.squeeze(axis=1)  # (N, 1) -> (N,)のように要素数が1の次元を除去する
        self.rgb_value = rgb_value.astype(np.uint8)  # float32 -> uint8

        _, self.counts = np.unique(
            self.labels, axis=0, return_counts=True
        )  # 重複したラベルを抽出し、カウント（NUMBER_OF_CLUSTERSの大きさだけラベルタイプが存在する）

        self.df = self.__summarize_result(self.rgb_value, self.counts)

        return self.df

    # 計算結果をグラフ用にDataFrame化させる
    @staticmethod
    def __summarize_result(rgb_value, counts):
        df = pd.DataFrame(data=counts, columns=["counts"])
        df["R"] = rgb_value[:, 0]
        df["G"] = rgb_value[:, 1]
        df["B"] = rgb_value[:, 2]

        # plt用に補正
        bar_color = rgb_value / 255
        df["plt_R_value"] = bar_color[:, 0]
        df["plt_G_value"] = bar_color[:, 1]
        df["plt_B_value"] = bar_color[:, 2]

        # グラフ描画用文字列
        bar_text = list(map(str, rgb_value))
        df["plt_text"] = bar_text

        # countsの個数順にソートして、indexを振り直す
        df = df.sort_values("counts", ascending=True).reset_index(drop=True)
        return df
```

### K-meansアルゴリズムの収束基準を設定する
```Python
criteria = cv2.TERM_CRITERIA_MAX_ITER + cv2.TERM_CRITERIA_EPS, 10, 1.0
```
cv2.TERM_CRITERIA_MAX_ITER：反復回数が最大値に達した場合に収束判定を行うフラグ
cv2.TERM_CRITERIA_EPS：クラスタ中心が移動する距離がしきい値以下になった場合に収束判定を行うフラグ

``` Python
criteria = cv2.TERM_CRITERIA_MAX_ITER + cv2.TERM_CRITERIA_EPS, 10, 1.0
```
であれば、最大反復回数が10で、移動量の閾値が1.0

```Python
_, labels, rgb_value = cv2.kmeans(
    data=colors,  # クラスタリングするための入力データ
    K=self.number_of_cluster,  # クラスタ数
    bestLabels=None,  # クラスタ番号の初期値(通常はNone)
    criteria=criteria,  # アルゴリズムの収束基準
    attempts=10,  # 異なる初期値でアルゴリズムを実行する回数
    flags=cv2.KMEANS_RANDOM_CENTERS,  # クラスタリングアルゴリズムのフラグ
)
```

戻り値は以下の3つ
compactness:各点とその所属するクラスタ中心との距離の総和。
labels:各データ点の所属するクラスタのラベル。
centers:クラスタの中心点の座標の配列。要は画像の場合は、RGBのリストになっている。

## 図の出力

```Python
class MakeFigure:
    def __init__(self, dataframe, overall_image, cliped_image, rgb_value, labels):
        print(dataframe)
        self.df = dataframe  # DataFrame
        self.number_of_cluster = NUMBER_OF_CLUSTERS  # クラスタ数
        self.overall_image = overall_image  # 全体画像
        self.cliped_image = cliped_image  # 切り抜き画像
        self.rgb_value = rgb_value  # RGB値
        self.labels = labels  # 図専用ラベル

    def output_overall_image(self, ax):
        ax.imshow(self.overall_image)

    def output_cliped_image(self, ax):
        ax.imshow(self.cliped_image)

    def output_histgram(self, ax):
        rgb_value_counts = (
            self.df.loc[:, ["counts"]].to_numpy().flatten().tolist()
        )  # ヒストグラム用のrgb値カウント数

        bar_color = (
            self.df.loc[:, ["plt_R_value", "plt_G_value", "plt_B_value"]]
            .to_numpy()
            .tolist()
        )  # ヒストグラム用のrgb値カウント数

        bar_text = self.df.loc[:, ["plt_text"]].to_numpy().flatten()  # ヒストグラム用x軸ラベル

        # ヒストグラムを表示する。
        ax.barh(
            np.arange(self.number_of_cluster),
            rgb_value_counts,
            color=bar_color,
            tick_label=bar_text,
        )

    def output_replaced_image(self, ax):
        # 各画素を k平均法の結果に置き換える。
        self.dst = self.rgb_value[self.labels].reshape(self.cliped_image.shape)
        ax.imshow(self.dst)
```

このクラスで結果の図の出力を行っています。
main関数内では以下のように書きます。
```Python
        # 可視化する。
        fig, [ax1, ax2, ax3, ax4] = plt.subplots(1, 4, figsize=(16, 5))
        fig.subplots_adjust(wspace=0.5)

        # 全体画像を表示する
        make_figure.output_overall_image(ax1)
        # 切り抜き画像を表示する。
        make_figure.output_cliped_image(ax2)
        # ヒストグラムを表示する
        make_figure.output_histgram(ax3)
        # クラスタ数分のRGB値で置き換え画像を生成
        make_figure.output_replaced_image(ax4)

        # 各タイトル
        ax2_title = "x:{} y:{}".format(str(START_X), str(START_Y))
        ax1.set_title("overall view " + IMG_NAME)
        ax2.set_title("cliped Image_" + ax2_title)
        ax3.set_title("histgram")
        ax4.set_title("replaced image")
```
- ax1：全体画像
- ax2：切り取り指定した後の画像
- ax3：K平均法の解析結果のヒストグラム
- ax4：各画素を k平均法の結果に置き換えた画像


# サンプル画像でK-平均をやってみる。

## Google のロゴ
### 元画像
![Google-Logo](data/sample/Google-Logo.jpg)
### K=5
![Google-Logo](sample_image_result/Google-Logo.jpg_k5_result.jpg)
![Google-Logo](sample_image_result/Google-Logo.jpg_k5_replaced.jpg)
### K=10
![Google-Logo](sample_image_result/Google-Logo.jpg_k10_result.jpg)
![Google-Logo](sample_image_result/Google-Logo.jpg_k10_replaced.jpg)
### K=20
![Google-Logo](sample_image_result/Google-Logo.jpg_k20_result.jpg)
![Google-Logo](sample_image_result/Google-Logo.jpg_k20_replaced.jpg)
### K=50
![Google-Logo](sample_image_result/Google-Logo.jpg_k50_result.jpg)
![Google-Logo](sample_image_result/Google-Logo.jpg_k50_replaced.jpg)

## フルーツの写真
### 元画像
![fresh-fruits](data/sample/fresh-fruits-2305192_960_720.jpg)
### K=5
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k5_result.jpg)
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k5_replaced.jpg)
### K=10
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k10_result.jpg)
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k10_replaced.jpg)
### K=20
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k20_result.jpg)
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k20_replaced.jpg)
### K=50
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k50_result.jpg)
![Google-Logo](sample_image_result/fresh-fruits-2305192_960_720.jpg_k50_replaced.jpg)

## 風景の写真
### 元画像
![sea-free-photo](data/sample/sea-free-photo5.jpg)
### K=5
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k5_result.jpg)
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k5_replaced.jpg)
### K=10
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k10_result.jpg)
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k10_replaced.jpg)
### K=20
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k20_result.jpg)
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k20_replaced.jpg)
### K=50
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k50_result.jpg)
![Google-Logo](sample_image_result/sea-free-photo5.jpg_k50_replaced.jpg)

# 参考
https://xtrend.nikkei.com/atcl/contents/18/00076/00008/　<br>
https://pystyle.info/opencv-kmeans/
