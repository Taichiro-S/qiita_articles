---
title: Flutterの画像読み込み中の表示をイイ感じにしよう！
tags:
  - Flutter
private: false
updated_at: '2024-02-10T13:15:28+09:00'
id: 6a8e46afdacfc996d88b
organization_url_name: null
slide: false
ignorePublish: false
---


## この記事を読んでほしい人

FLutterの画像読み込みで「**くるくる🌀**」させてる人

## やること

- [cached_network_image](https://pub.dev/packages/cached_network_image)で画像をキャッシュして表示する
- [skelton_text](https://pub.dev/packages/skeleton_text)で画像読み込み時にキラッ✨とさせる
- （おまけ）[lottie](https://pub.dev/packages/lottie)でイイ感じのアニメーションをつける

これらのパッケージはインストールしておいてください！

**動作イメージ**
ZennのRss Feedを取得して表示しています（アニメーションをしっかり表示するために3秒間待っています）
https://youtu.be/zEpzqLV-0B8

サンプルリポジトリ公開しています！
https://github.com/Taichiro-S/flutter_image_loading_sample

## 環境・バージョン等

macOS Sonoma 14.1.1
iOS(Simulator) 17.0
Xcode 15.0.1
Dart SDK 3.2.2
Flutter 3.16.2

## 画像読み込み時にキラッ✨とさせる

cached_network_image で画像（本サンプルではZennのRss Feedのenclosure）を表示します

```dart
SizedBox(
    width: MediaQuery.of(context).size.width,
    child: article.enclosureUrl != ''
        ? CachedNetworkImage(
            imageUrl: article.enclosureUrl,
            placeholder: (context, url) => SkeltonContainerWidget(
                width: MediaQuery.of(context).size.width),
            errorWidget: (context, url, error) =>
                const Image(image: AssetImage('assets/no_image.png')))
        : const Image(image: AssetImage('assets/no_image.png')),
)
```

placeholder に SkeltonAnimationを設定します
height, width等を表示する画像に合わせて調節してください

```dart
import 'package:flutter/material.dart';
import 'package:skeleton_text/skeleton_text.dart';

class SkeltonContainerWidget extends StatelessWidget {
  const SkeltonContainerWidget({
    super.key,
    required this.width,
  });
  final double width;
  @override
  Widget build(BuildContext context) {
    return SkeletonAnimation(
        child: Padding(
            padding: const EdgeInsets.symmetric(vertical: 8, horizontal: 10),
            child: Container(
              width: width,
              height: 180,
              decoration: BoxDecoration(
                  color: Colors.grey[300],
                  borderRadius: BorderRadius.circular(10)),
            )));
  }
}

```

## イイ感じのアニメーションをつける

[ここ](https://lottiefiles.com/featured)からよさげなのをjson形式でダウンロードします
サンプルで使用させていただいたのは[こちら](https://app.lottiefiles.com/animation/be46b59d-8aab-4f78-8878-033b8c47bd11?channel=web&source=public-animation&panel=download)です

ダウンロードしたjsonを`loading.json`として`assets`フォルダに入れ、`pubspec.yml`に以下のように追記します

```yml
flutter:
  assets:
  - assets/loading.json
```

あとはloading時に呼び出すだけです！
以下ではRiverpodの`AsyncValue.when`の`loading`内で表示しています

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:lottie/lottie.dart';

class RssFeedPage extends ConsumerWidget {
  const RssFeedPage({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final selectedTopic = ref.watch(topicProvider);
    final articlesAsync = ref.watch(articlesProvider(topic: selectedTopic));
    return DefaultTabController(
        length: topics.length,
        child: Scaffold(
            // 略
            body: articlesAsync.when(
                loading: () => Lottie.asset('assets/loading.json'),
                error: (error, stackTrace) =>
                    Center(child: Text(error.toString())),
                data: (articles) {
                  // 略
                })));
  }
}
```

## 参考にさせていただいた記事

https://blog.pentagon.tokyo/3216/
