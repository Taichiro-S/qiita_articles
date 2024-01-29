---
title: Flutterのurl_lanucherとuni_linksでOAuth
tags:
  - Android
  - iOS
  - OAuth
  - Flutter
  - Notion
private: false
updated_at: '2024-01-29T22:06:48+09:00'
id: b1551eb0dbbef83918c9
organization_url_name: null
slide: false
ignorePublish: false
---
[この記事](https://qiita.com/meique/items/c1933247803d92829020)の補足記事になります。

認証部分は読んだ前提で話を進めるので、読んでください！

webviewを使ったGoogleのOAuthがセキュリティ的にちょっとアレなため、url_launcherとuni_linksを使用した正攻法(?)で実装してみます。

サンプルのリポジトリは同じです。
https://github.com/Taichiro-S/notion_sample

## 使用するFlutterパッケージ

- [url_launcher](https://pub.dev/packages/url_launcher) : 端末のデフォルトブラウザでWebページを開く
- [uni_links](https://pub.dev/packages/uni_links) : deeplinkするやつ

## 実装

修正箇所は以下の2点です。

1. **「連携」ボタンクリック時にnotionの認証ページをurl_launcherで開くようにする**
2. **uni_liksの`linkStream.listen`で自分のアプリへのリクエストをlistenする**

修正が必要なのは`main.dart`のみです。

まず連携ボタンを修正。

```main.dart
child: ElevatedButton(
    onPressed: () async {
        if (await canLaunchUrl(Uri.parse(Env.notionOauthUrl))) {
         launchUrl(Uri.parse(Env.notionOauthUrl));
        }
    },
    child: const Text('Notionと連携する'),
)
```

次にlistenerを設定
MyAppをConsumerWidgetからConsumerStatefulWidgetに変更します。
`linkStream.listen`内で、自分のアプリに対するリクエストで、URLに`code`パラメータを含むものを検知してauthenticateメソッドを実行し、providerを更新します。

```dart
class MyApp extends ConsumerStatefulWidget {
  const MyApp({super.key});

  @override
  ConsumerState<MyApp> createState() => _MyApp();
}

class _MyApp extends ConsumerState<MyApp> {
  StreamSubscription? _sub;
  @override
  void initState() {
    super.initState();
    initUniLinks();
  }

  Future<void> initUniLinks() async {
    _sub = linkStream.listen((link) async {
      if (link != null &&
          link.startsWith('notionsample://oauth/callback?code')) {
        await ref.read(notionOauthApiProvider).authenticate(link);
        ref.invalidate(notionAuthProvider);
      }
    }, onError: (e) {
      print(e.toString());
    });
  }

// build以降はそのまま
}
```

これで一応実装できました。

動かしてみるとこんな感じ。

https://youtube.com/shorts/L57JhLel_y4

2つほど気になるところが...

### 1. 二度目以降の連携では最初に連携をしたアカウントの連携ページに飛ばされてしまう

ブラウザに初回ログイン時の情報が残っているため？
webviewでは毎回ページを開く前にキャッシュを削除していた(`clearCache: true`に設定)が、url_launcherではできないみたい。
Githubに同様のissueがあったが、

>url_launcher is intended to have a minimal API for launching, and the API of SFViewController is extremely limited so this couldn't be implemented on iOS anyway. Use cases that need detailed control of the webview should build on webview_flutter instead.
>
>Closing as out of scope.

webviewを使わないと無理っぽい。

https://github.com/flutter/flutter/issues/29976

### 2. 認証完了後に自動でブラウザを閉じることができない

ので、ユーザーに手動で閉じてもらう必要がある。
こちらちょっとめんどい程度の問題なのでまあ許せるが、1つ目は今のところ解決策が思い浮かばない。

**ということで、いい解決方法知ってる人、教えてください！**

