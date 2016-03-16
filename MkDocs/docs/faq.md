# FAQ
- - -
質問や不具合の症状報告と、それに対する解決策や回避策について紹介しています。


<br/><br/>
## 1. オーバーロード解決の不具合
- - -
NuGetパッケージver1.0.1相当のバージョンに関する不具合として、下記の条件を同時に満たした場合に`QiObject.Call`を用いた関数呼び出しが失敗する事を確認しています。

- 同一サービス内に関数オーバーロードが(つまり同名の関数が)複数ある
- 実引数としてタプル(`QiTuple`)または辞書(`QiMap`)型の変数を渡す

この問題は具体[ALAnimatedSpeech::say](http://doc.aldebaran.com/2-1/naoqi/audio/alanimatedspeech-api.html#ALAnimatedSpeechProxy::say__ssCR)関数などで生じます。

<br/>
この不具合は`QiObject.Call`関数の内部で行っているオーバーロードの解決処理が失敗しているのが原因で生じています。対策は2通り提供しています。

- NuGetパッケージver2.0.0相当以降のバージョンではこの不具合は修正しました。
- NuGetパッケージver1.0.1相当バージョンでこの問題を回避するには、オーバーロードを手動で選択する`QiObject.CallDirect`関数が利用できます。

両方のバージョンに関して、`ALAnimatedSpeech::say`を利用するためのサンプルコードを[GitHub](https://github.com/malaybaku/LibqiDotNetAnimatedSaySample)に掲載しています。


