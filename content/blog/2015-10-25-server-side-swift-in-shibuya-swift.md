---
layout: post
title: "shibuya.swift#1でServer-Side Swift?というテーマでLTした"
date: 2015-10-25 16:50:35 +0900
comments: true
categories: Swift
---

最後にクエスチョンマーク10個くらい付けたい気持ちがあるけど、[shibuya.swift#1](http://shibuya-swift.connpass.com/event/19306/)でLTしてきた。  

<!-- more -->

スライドは以下。

<div style="width: 65%"><script async class="speakerdeck-embed" data-id="5b406d260b214fe09b55823b6ccaa490" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script></div>  
<br>
  
Swiftがオープンソースになるということで、client/serverの両方Swiftで書けるようになるんじゃ、って話がちょっと前に上がっていたので、何か周辺で進展あるかなと確認もしたくってこの話にした。  
ついでに[ISUCON](http://isucon.net/)の予選が終わったばかりで、その実装くらいなら練習にも丁度いいでしょと思ったのだけど、資料にもある通りDBI/DBD的なものを見つけられずさすがに自分で実装する時間もなく半端に終わってしまった（結局メモリ上にデータ載っけてるだけ）。。  
まだ実装途中ですが、デモに使ったレポジトリは以下。  

https://github.com/shingt/SwiftServerExamples  

使用したframework等は以下。

* HTTP Server Engine / Framework
  * https://github.com/glock45/swifter
  * https://github.com/izqui/Taylor
* Template Engine
  * https://github.com/groue/GRMustache.swift

スライドの中でも紹介したけど、[cocoapods-rome](https://github.com/neonichu/Rome/)が便利だった。  
cocoapods対応してないものでも`pod install`の際にframeworkをビルドして配置してくれる。今回のようにXcodeに依存したくない時には良い。  
元々はRealm newsの以下のプレゼンで紹介されていたもの。  

[Swift Scripting with Ayaka Nonaka](https://realm.io/news/swift-scripting/)

### 所感

思ったよりこのレイヤで書いてる人が少なかった。テンプレートエンジンもMustacheのものしか見つからなかったし。今書いてもどうにもならないから当たり前ですが。。  
楽しいは楽しいけど、ちょろっと書いて試せることと運用できるレベルに持ってくることとはまるで違うし、というかそもそも本当に今年中にオープンソースになるんですかね。 

