---
layout: post
title: "UICollectionViewでのCellごとのページング"
date: 2014-08-31 16:20:35 +0900
comments: true
categories: iOS
---

iOS端末の画面の上部分に横スクロールメニューを作りたいって場合、UICollectionViewを使うと割と簡単に作れる。  

これがただスクロールするだけのもので良いなら楽なんだけど、ユーザがスクロールする毎にセル毎にページングさせて、そのセルを画面の中心に持ってきたい、って場合があった。  

って言葉で書くと分かりにくいので画面例を出す。以下のように横に選択肢が並んでるイメージ。  

{% img center /images/uicollectionvew-paging-cell-1.png 320 'title text' 'alt text' %}

<!-- more -->

この時に、メニュー部分を横にスクロールすると、途中で止まらずに勝手にセル毎にページング（ページングって言うのかあやしいけど）され、セルの中心が画面の中心に来るようになってほしい。ストアに出てるアプリを例に挙げるとMercariさんのような感じ。  

Pressoさんとかはセルをタップしたらそのセルが中心になるように動くけど、それは``didSelectItemAtIndexPath``内で座標指定してスクロールするようにすればいいだけ（だと思ってる）のでまた別の話。  

``UICollectionView``は``UIScrollView``を継承しているので、``pagingEnabled``も設定できる。
最初はこれをいじって何かしらのパラメータ変えれば良いのかなくらいに思っていたんだけど、これだとうまくいかない。
``YES``にしても画面の横幅分のスクロールするようになるだけ。
例えば上記画面をスクロールすると、以下の状態にページングしてしまう。  

{% img center /images/uicollectionview-paging-cell-2.png 320 'title text' 'alt text' %}

丁度さっきの画面で見切れていたMenu2の部分の途中から画面が始まる形になった。  
つまり、画面の横幅分 = 320px分右に移動した形。  

ScrollView側で移動幅制御できないかなとも思って色々試したんだけどそれもうまくいかず、stackoverflowでの次の質問に行き着いた。  

- [targetContentOffsetForProposedContentOffset:withScrollingVelocity without subclassing UICollectionViewFlowLayout](http://stackoverflow.com/questions/13492037/targetcontentoffsetforproposedcontentoffsetwithscrollingvelocity-without-subcla)

基本はUICollectionViewFloaLayoutのサブクラスを作って、その``targetContentOffsetForProposedContentOffset``をオーバーライドする形で実装する。  
このメソッドは``UICollectionViewLayout.h``で以下のように説明されている。  

> _return a point at which to rest after scrolling - for layouts that want snap-to-point scrolling behavior_

つまり、スクロール後にどこに留まるかを指定するようなコード書けばよい。これは``UIScrollView``のプロパティである``contentOffset``を指定する形で書くことになる。  

stackoverflowの回答の場合、常に対象セルを画面左端に5px空けた上で置くようにしているけれど、今回は画面中央に置きたいので、例えば次のように実装する。  

```obj-c SomeChildCollectionViewFlowLayout
const float kMenuCellWidth  = 80.;
const float kWindowWidth = 320.;

// 中略

- (CGPoint)targetContentOffsetForProposedContentOffset:(CGPoint)proposedContentOffset withScrollingVelocity:(CGPoint)velocity
{
    CGFloat offsetAdjustment = MAXFLOAT;
    CGFloat horizontalOffset = proposedContentOffset.x + (kWindowWidth - kMenuCellWidth) / 2.;
    CGRect targetRect = CGRectMake(proposedContentOffset.x, 0, self.collectionView.bounds.size.width, self.collectionView.bounds.size.height);
    NSArray *array = [super layoutAttributesForElementsInRect:targetRect];
    for (UICollectionViewLayoutAttributes *layoutAttributes in array) {
        CGFloat itemOffset = layoutAttributes.frame.origin.x;
        if (ABS(itemOffset - horizontalOffset) < ABS(offsetAdjustment)) {
            offsetAdjustment = itemOffset - horizontalOffset;
        }
    }
    return CGPointMake(proposedContentOffset.x + offsetAdjustment, proposedContentOffset.y);
}
```

上記のコードだと、``proposedContentOffset``がユーザが手を離した瞬間のcontentOffsetを指す。入力としてこれ（と``velocity``）を受け取り、遷移完了時に指定したい``contentOffset``を出力として返す、という流れを作っている。  
（ブレークポイントを打つなりなんなりするとスクロールして手を離した瞬間にこのメソッドが呼ばれていることが分かる。）  

``kWindowWidth``に画面横幅、``kMenuCellWidth``にセルの横幅が入っているという想定でまず``horizontalOffset``を計算している。  
この値は "最終的に画面真ん中に位置して欲しいセルのoffset" を表している。  
（そのため個人的には別な名称の方が良い気がするんだけれどあまりいいのが思いつかないし参照したstackoverflowに合わせる。）  

後は、現状画面に表示されている各セルをその``horizontalOffset``の位置まで移動させた時に、どれくらいの移動があるかを見て、その移動が最も小さくなるような（= 自然なページングとなるような）セルを選択する。  

具体的には、``UICollectionViewFloaLayout``の``layoutAttributesForElementsInRect``で画面上に表示されている部分の``layoutAttributes``、すなわち各セルに関してのレイアウトの情報を、配列としてまず取得する。このレイアウト情報にはoffsetも含まれているので、``horizontalOffset``との差分を移動距離（``offsetAdjustment``）として計算し最小のものを選択する。後はその差分を使って最終的なoffsetを返してやれば良い。  

（注意点として、この記述をしたとしても、collectionViewに対してさっきの``pagingEnabled``を``YES``にしたままだとそちらの設定が優先されるようで、正しくセル毎のページングができない。）  

結果として、さっきのやつをセルごとにページングできるようになる。  

{% img center /images/uicollectionview-paging-cell-3.png 320 'title text' 'alt text' %}

こうなって  

{% img center /images/uicollectionview-paging-cell-4.png 320 'title text' 'alt text' %}

こうと。静止画じゃ分かりにくいけれど。 

