# Unityでライブラリを使うには
- - -

本チュートリアルは今までと少し異なる用途を想定したものです。本チュートリアルでは`Windows`のみをターゲットとした.NETアプリケーションではなく、ゲームエンジンであるUnityEngine上で`Baku.LibqiDotNet`を利用する方法を紹介します。

<br/><br/>
##1. Unityが利用できる環境
- - -
`Windows`で**32bitの**Unity Editorを用いるか、`Mac`のUnity Editorを用いて`Baku.LibqiDotNet`と連携した開発が可能です。

本記事以外の話題は基本的に`Windows`に焦点を絞ってきましたが、`Unity`の場合はターゲットとするプラットフォームが広く取れるため、原理的にはラップ元である`libqi`がサポートされる全てのプラットフォームで`Baku.LibqiDotNet`とUnityを連携させたアプリケーションを動作させられます。

とはいうものの、パッケージの体裁として現状では`Windows`と`Mac`のみがサポートされています。これら以外のOS(デスクトップの`Linux`や`Android`など)上で`Baku.LibqiDotNet`を用いたUnityアプリケーションを動かしたい場合でも、マネージライブラリである`Baku.LibqiDotNet.dll`はそのまま利用できます。ただし、動作させたいOSに対応した`libqi`のネイティブライブラリをUnityプロジェクトのAssetへ正しく配置する必要があることに注意してください。


<br/><br/>
## 2. Unity用ライブラリの利用
- - - 

Unity用のパッケージファイル(`.unitypackage`)はNuGetでは提供していません。筆者が`Unity`向けにビルドしたパッケージを**[Google Drive](https://drive.google.com/folderview?id=0BzVgwIMLJboJeWJmaFZ3Q25ENjQ&usp=sharing)**に公開しているので、こちらからパッケージを取得してください。なるべく新しいバージョンを利用することを推奨しています。

Unitypackageの利用上のポイントは以下の3点です。

- Windowsの場合**32bitの**Unity Editorを使わないとデバッグ実行が出来ません。
- パッケージの導入方法についてはzipファイル内の`readme.txt`に記載しています。
- ライブラリの内容は基本的にNuGetパッケージとほぼ同一ですが、Macに対応するためにNuGetパッケージに含まれないフォルダを追加で持っています。

<br/>
.NETアプリケーションと同様のライブラリを移植しているため、Unityに特有な機能は一切提供していません。


<br/>
Unity向けパッケージに含まれる`Baku.LibqiDotNet.dll`はNuGetパッケージに含まれる`Baku.LibqiDotNet.dll`とほぼ一致していますが、下記の2点のみが異なります。

- NuGetパッケージ版ではターゲットアーキテクチャが`x86`とされているのに対し、Unity向けパッケージでは`AnyCPU`がターゲット
- NuGetパッケージ版では`DllImport`で指定するアンマネージライブラリ名が`qic.dll`であるのに対し、Unity向けパッケージでは拡張子のない`qic`という名前指定が使われている

<br/>
上記の差異が存在する理由も簡単に補足しておきます。

第一にUnity向けパッケージでターゲットアーキテクチャが`AnyCPU`となっている理由ですが、これはWindows/Macの両方に対応するためです。`Baku.LibqiDotNet`がラップしているアンマネージライブラリがWindowsでは32bit、Mac版では64bitのみのサポートとなっているため、マネージドライブラリは`AnyCPU`ターゲットであることが必須となっています。これに対してNuGetパッケージ版はWindowsでの利用のみを想定しているため、動作しない`x64`での実行を防ぐ目的で`x86`がターゲットとなっています。

第二に`DllImport`の参照先が変わっている理由ですが、これについては[Unityマニュアル](http://docs.unity3d.com/ja/current/Manual/PluginsForDesktop.html)にUnityに特有な`DllImport`の利用法が記載されているので、それに従っているだけです。


<br/><br/>
## 3. 把握済みの問題

`Windows`以外のOSで`Baku.LibqiDotNet`とUnityを連携させたアプリケーションを作成する場合、バイナリデータの送受信は(多分)できません。この制約はラップ元のアンマネージライブラリである[`libqi-capi`](https://github.com/aldebaran/libqi-capi)の未実装部分を利用していることに由来します(`Windows`に関してはソースからビルドし直して対策しました。詳細は[GitHubのissue](https://github.com/malaybaku/BakuLibQiDotNet/issues/1)に掲載しているので、他のOSでも`libqi-capi`のビルド環境さえ構成できれば対策は可能だと思います)。



<br/><br/>
## 補足: GitHubの最新ソースをUnity用にビルドする
- - - 
上で書いたようにUnity向けのパッケージはビルドして配布されていますが、アンマネージライブラリ群をUnityパッケージから取得し、ライブラリ本体である`Baku.LibqiDotNet.dll`についてだけ最新のソースを反映することも可能です。手順は次のようになります。

1. [Baku.LibqiDotNet](https://github.com/malaybaku/BakuLibQiDotNet)のソースを取得する。あるいは手元でソースコードを適宜書き換える。
2. ターゲットアーキテクチャを`AnyCPU`にし、構成を`Release`にする(※`Debug`構成でもいいですが動作速度などの理由からオススメできません)
3. `Baku.LibqiDotNet`プロジェクトのプロパティでコンパイルシンボルとして`UNITY`を定義
4. ビルドする
5. ビルド結果のうち`Baku.LibqiDotNet.dll`および`Baku.LibqiDotNet.xml`をUnityのAssetに含まれる既存のファイルから置き換える

<br/>
`UNITY`というシンボルを定義する理由ですが、これはソース中の[**`DllImportSetting.cs`**](https://github.com/malaybaku/BakuLibQiDotNet/blob/master/Baku.LibqiDotNet/Baku.LibqiDotNet/QiApi/DllImportSettings.cs)でコンパイルシンボルに応じて`DllImport`属性に渡すファイル名を切り替えているためです。推奨はしませんが、シンボルによる分岐ではなく上記のソースファイル部分を手作業で適切に書き換えても問題ありません。

