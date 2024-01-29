---
title: FlutterアプリをNotionと連携する
tags:
  - Android
  - iOS
  - OAuth
  - Flutter
  - Notion
private: false
updated_at: '2024-01-29T22:06:53+09:00'
id: c1933247803d92829020
organization_url_name: null
slide: false
ignorePublish: false
---
[Notion](https://www.notion.so/ja-jp)は史上最高の万能アプリなわけですが（異論は認める）、Flutterアプリ連携しちゃお〜ってやってたら案外苦労しました。というお話。

サンプルのリポジトリ公開しています！
https://github.com/Taichiro-S/notion_sample

## 前提

- notionアカウント作成済み
- flutterアプリ作成済み（fvmを使っているなら[これ](https://qiita.com/kilalabu/items/7ed07a99024e1687a8f4)が早いです）
- OAuthの仕組みなんとなくわかる（[この記事](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)がわかりやすいかも）

## 実装する機能

FlutterアプリからNotionアカウントを連携し、Databaseにデータを書き込む

### 動作イメージ

※動画はiOS simulatorですが、androidでも同じように動作します。
アカウント連携
https://youtube.com/shorts/lesGbzy4NnQ

データベース操作
https://youtube.com/shorts/AfS8vTcOgQM

## 使用するFlutterパッケージ

- [freezed](https://pub.dev/packages/freezed) : immutableなクラスを作るやつ
- [riverpod](https://pub.dev/packages/riverpod) : 状態管理のやつ
- [secure_storage](https://pub.dev/packages/flutter_secure_storage) : セキュアにストレージしてくれる
- [inappwebview](https://pub.dev/packages/flutter_inappwebview) : アプリ内でwebviewできる
- [http](https://pub.dev/packages/http) : httpリクエストするやつ
- [envied](https://pub.dev/packages/envied) : 環境変数をセキュアに扱える

**※これらのパッケージの使い方は解説しません！**
自分がちゃんと使えているのかあやしいところがあるので...

## 環境・バージョン等

macOS Sonoma 14.1.1
iOS 17.0
Xcode 15.0.1
Dart SDK 3.2.2
Flutter 3.16.2

## 大まかな流れ

[公式のドキュメント](https://developers.notion.com/docs/authorization)が割と丁寧に書いてあるので、これに沿って進めます。

1. Notion integration作成
2. リダイレクト用ページの作成
3. OAuth認証フローの実装
4. NotionのDatabaseにアクセス

長い道のりです。頑張りましょ〜😇

## 1. Notion integrationの作成

まずは[ここ](https://www.notion.so/my-integrations)から、public integrationを作成します（internal integrationを作成し、ディストリビューションからパブリックに設定します）。
リダイレクトURIは、`https://<アカウント名>.github.io/<リポジトリ名>/redirects`とします。後でGithub pagesでホスティングするURLになります。
設定が終わると、以下の3つが発行されるので、`.env`に書いておきます。

- OAuthクライアントID
- OAuthクライアントシークレット
- 認証URL

`lib/env/env.dart`を作成し、enviedで`.env`ファイルの中身を読み取ります。

```env.dart
import 'package:envied/envied.dart';

part 'env.g.dart';

@Envied(path: '.env')
abstract class Env {
  @EnviedField(varName: 'NOTION_OAUTH_CLIENT_ID')
  static const String notionOauthClientId = _Env.notionOauthClientId;
  @EnviedField(varName: 'NOTION_OAUTH_CLIENT_SECRET')
  static const String notionOauthClientSecret = _Env.notionOauthClientSecret;
  @EnviedField(varName: 'NOTION_OAUTH_URL')
  static const String notionOauthUrl = _Env.notionOauthUrl;
}
```

build runnerを実行すると、`env.g.dart`が生成します。
**`env.g.dart`は必ず`gitignore`に記載しましょう。** ← ここ忘れるとenvの中身全部githubに晒してしまいます！！（一回やらかした人）

## 2.  リダイレクト用ページの作成

Github pagesでホスティングし、notionからリダイレクトします。
直でアプリにリダイレクトできると楽なんですが、notionで設定するリダイレクトURLのスキームが`https`しか設定できませんでした。
このページではリダイレクトURLに含まれる`code`を取得し、自分のアプリに再度リダイレクトします。

プロジェクトルートに`docs/redirects/index.html`を作成します。

```index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Redirect</title>
  </head>
  <body>
    <script>
      var urlParams = new URLSearchParams(window.location.search);
      var code = urlParams.get("code");
      if (code == null){
        window.location.replace(
          "https://taichiro-s.github.io/notion_sample/redirects/fallback.html"
        );
      } else {
        window.location.replace(
          "notionsample://oauth/callback?code=" + code
        );
      }
    </script>
  </body>
</html>
```

URLに`code`が含まれないときのエラーページとして`fallback.html`も作成しておきます。

mainブランチにpushし、githubのsettings → pages でmainブランチのdocsフォルダを選択します。

ブラウザから`https://<アカウント名>.github.io/<リポジトリ名>/redirects`にアクセスして`fallback.html`にリダイレクトされれば、ホスティングできています。

## 3. OAuth認証フローの実装

認証の大まかな流れは以下のようになります。

1. 「連携」ボタンをクリックすると、integrationを作成したときの認証URLをinappwebviewで開く。
2. ログインすると、integrationで設定したページにリダイレクトし、そこからアプリにリダイレクトする。
3. リダイレクトURLに含まれる`code`を取り出して、アクセストークンを発行し、secure storageに保管する

以下の4つのファイルを作成します。

```zsh
.
├── api
│   ├── notion_oauth_api.dart  <- apiリクエストを送ったりするクラス
│   └── notion_oauth_api.g.dart
├── env
│   ├── env.dart　
│   └── env.g.dart
├── main.dart  <- ログインページ
├── provider
│   ├── notion_auth_provider.dart  <- notionとの連携状態を管理するプロバイダ
│   ├── notion_auth_provider.freezed.dart
│   ├── notion_auth_provider.g.dart
│   ├── webview_provider.dart  <- webviewの状態を管理するプロバイダ
│   ├── webview_provider.freezed.dart 
│   └── webview_provider.g.dart
└── widget
    └── notion_login_webview_widget.dart   <- webviewの設定
```

1つずつ説明していきます。

- `main.dart` : ログインページ
  webviewが開いている時はページを表示し、開いていないときはnotionとの連携状態に応じてボタンやnotionのデータ等を表示します。「連携」をクリックで認証ページを開き、「連携を解除」でsecure storageから認証情報を削除します。ただし、notionにはintegrationの連携が残ります。気になる場合は設定 → 自分のコネクトで連携を解除します。


```main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:notion_sample/provider/notion_auth_provider.dart';
import 'package:notion_sample/provider/webview_provider.dart';
import 'package:notion_sample/widget/notion_database_list_widget.dart';
import 'package:notion_sample/widget/notion_login_webview_widget.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(
    const ProviderScope(child: MyApp()),
  );
}

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final notionAuthAsync = ref.watch(notionAuthProvider);
    final notionAuthNotifier = ref.read(notionAuthProvider.notifier);
    final webview = ref.watch(webViewProvider);
    final webviewNotifier = ref.read(webViewProvider.notifier);
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (notionAuthAsync is AsyncLoading) {
        ref.read(notionAuthProvider.notifier).getNotionWorkspace();
      }
    });
    if (webview.isOpen) {
      return MaterialApp(
          home: Scaffold(
              appBar: AppBar(
                  title: Row(children: [
                IconButton(
                  icon: const Icon(Icons.close_outlined, color: Colors.black),
                  onPressed: () => webviewNotifier.hide(),
                ),
                const Center(child: Text("Notionと連携")),
              ])),
              body: Column(children: [
                Expanded(
                    child: Stack(children: <Widget>[
                  const NotionLoginWebviewWidget(),
                  webview.isError
                      ? Center(child: Text(webview.errorText))
                      : webview.isLoading
                          ? Container(
                              color: Colors.white,
                              child: const Center(
                                  child: CircularProgressIndicator()))
                          : Container()
                ]))
              ])));
    } else {
      return MaterialApp(
          home: Scaffold(
        resizeToAvoidBottomInset: false,
        appBar: AppBar(
          title: const Text('Notion Sample'),
        ),
        body: notionAuthAsync.when(
          error: (error, stackTrace) => Center(
            child: Text('Error: $error'),
          ),
          loading: () => const Center(
            child: CircularProgressIndicator(),
          ),
          data: (notionAuth) {
            if (notionAuth.isAuth) {
              return Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Text('${notionAuth.workspaceName}'),
                    const SizedBox(height: 10),
                    notionAuth.workspaceIcon != null
                        ? Image.network(notionAuth.workspaceIcon!)
                        : const SizedBox(),
                    const SizedBox(height: 10),
                    ElevatedButton(
                      onPressed: () async {
                        await notionAuthNotifier.deleteNotionWorkspace();
                      },
                      child: const Text('連携を解除'),
                    ),
                    const SizedBox(height: 10),
                    const NotionDatabaseListWidget()
                  ],
                ),
              );
            } else {
              return Center(
                child: ElevatedButton(
                  onPressed: () {
                    webviewNotifier.show();
                  },
                  child: const Text('Notionと連携する'),
                ),
              );
            }
          },
        ),
      ));
    }
  }
}
```

このページでは以下2つのproviderをwatchしています。

- `notion_auth_provider.dart` : notionとの連携状態を管理するprovider
  secure storageにアクセストークンがあれば連携しているとみなします。

```notion_auth_provider.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'notion_auth_provider.g.dart';
part 'notion_auth_provider.freezed.dart';

@freezed
class NotionAuthState with _$NotionAuthState {
  const factory NotionAuthState({
    required bool isAuth,
    required String? workspaceIcon,
    required String? workspaceName,
  }) = _NotionAuthState;
}

@riverpod
class NotionAuth extends _$NotionAuth {
  final _storage = const FlutterSecureStorage();
  @override
  AsyncValue<NotionAuthState> build() {
    return const AsyncValue.loading();
  }

  Future<void> getNotionWorkspace() async {
    state = const AsyncValue.loading();
    try {
      final isAuth =
          await _storage.read(key: 'access_token') != null ? true : false;
      final workspaceName = await _storage.read(key: 'workspace_name');
      final workspaceIcon = await _storage.read(key: 'workspace_icon');
      if (isAuth) {
        state = AsyncValue.data(
          NotionAuthState(
            isAuth: true,
            workspaceIcon: workspaceIcon,
            workspaceName: workspaceName,
          ),
        );
      } else {
        state = const AsyncValue.data(
          NotionAuthState(
            isAuth: false,
            workspaceIcon: null,
            workspaceName: null,
          ),
        );
      }
    } catch (e) {
      throw Exception('Flutter Storage error: $e');
    }
  }

  Future<void> deleteNotionWorkspace() async {
    state = const AsyncValue.loading();
    try {
      _storage.delete(key: 'access_token');
      _storage.delete(key: 'workspace_name');
      _storage.delete(key: 'workspace_icon');
      state = const AsyncValue.data(
        NotionAuthState(
          isAuth: false,
          workspaceIcon: null,
          workspaceName: null,
        ),
      );
    } catch (e) {
      throw Exception('Flutter Storage error: $e');
    }
  }
}
```

- `webview_provider.dart` : webviewの状態
  open, loading, errorの3つの状態があります。

```webview_provider.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'webview_provider.g.dart';
part 'webview_provider.freezed.dart';

@freezed
class WebViewState with _$WebViewState {
  const factory WebViewState({
    required bool isOpen,
    required bool isLoading,
    required bool isError,
    required String errorText,
  }) = _WebViewState;
}

@riverpod
class WebView extends _$WebView {
  @override
  WebViewState build() {
    return const WebViewState(
        isOpen: false, isLoading: false, isError: false, errorText: '');
  }

  void show() => state = state.copyWith(isOpen: true);
  void hide() => state = state.copyWith(isOpen: false);
  void loading() => state = state.copyWith(isLoading: true);
  void loaded() => state = state.copyWith(isLoading: false);
  void error(String msg) {
    state = state.copyWith(isError: true);
    state = state.copyWith(errorText: msg);
  }

  void clearError() {
    state = state.copyWith(isError: false);
    state = state.copyWith(errorText: '');
  }
}
```

- `notion_oauth_api.dart` : apiを送ったりするクラス
  リダイレクトURLに含まれる`code`を使用して、アクセストークンを発行するためのリクエストを行います。発行出来たらsecure storageに保存します。

```notion_oauth_api.dart
import 'dart:convert';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:http/http.dart' as http;
import 'package:notion_sample/env/env.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'notion_oauth_api.g.dart';

class NotionOauthApi {
  final _storage = const FlutterSecureStorage();

  Future<void> authenticate(String url) async {
    final code = Uri.parse(url).queryParameters['code'];
    if (code != null) {
      final accessToken = await _getAccessToken(code);
      await _storage.write(key: 'access_token', value: accessToken);
    } else {
      throw Exception('code is null');
    }
  }

  Future<String> _getAccessToken(String code) async {
    final String encoded = base64Encode(utf8
        .encode('${Env.notionOauthClientId}:${Env.notionOauthClientSecret}'));
    final response = await http.post(
      Uri.parse('https://api.notion.com/v1/oauth/token'),
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'Authorization': 'Basic $encoded',
      },
      body: jsonEncode({
        'grant_type': 'authorization_code',
        'code': code,
        'redirect_uri': 'https://taichiro-s.github.io/notion_sample/redirects',
      }),
    );

    if (response.statusCode == 200) {
      final Map<String, dynamic> data =
          jsonDecode(response.body) as Map<String, dynamic>;
      await _storage.write(
          key: 'workspace_name',
          value: data['workspace_name'] != null
              ? data['workspace_name'].toString()
              : '');
      await _storage.write(
          key: 'workspace_icon',
          value: data['workspace_icon'] != null
              ? data['workspace_icon'].toString()
              : '');
      return data['access_token'].toString();
    } else {
      throw Exception('Failed to get access token: ${response.body}');
    }
  }
}

@riverpod
NotionOauthApi notionOauthApi(NotionOauthApiRef ref) {
  return NotionOauthApi();
}
```

- `notion_login_webview_widget.dart` : webviewの設定
  初期ページとしてintegration作成時の認証URLを設定しています。
  また、webview上でのリクエストで、自分のアプリに対するもの（Github pagesのリダイレクトURLに設定した`notionsample://oauth/callback?code`で始まるもの）があった場合、apiクラスのauthenticateメソッドを実行し、認証情報をsecure storageから取得します。

```notion_login_webview_widget.dart 
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:flutter_inappwebview/flutter_inappwebview.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:notion_sample/api/notion_oauth_api.dart';
import 'package:notion_sample/env/env.dart';
import 'package:notion_sample/provider/notion_auth_provider.dart';
import 'package:notion_sample/provider/webview_provider.dart';

class NotionLoginWebviewWidget extends ConsumerWidget {
  const NotionLoginWebviewWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final notionOauthApi = ref.read(notionOauthApiProvider);
    InAppWebViewController? webViewController;
    final webViewNotifier = ref.read(webViewProvider.notifier);
    return InAppWebView(
      initialUrlRequest: URLRequest(url: Uri.parse(Env.notionOauthUrl)),
      initialOptions: InAppWebViewGroupOptions(
          crossPlatform: InAppWebViewOptions(
            useShouldOverrideUrlLoading: true,
            useOnLoadResource: true,
            clearCache: true,
            userAgent: Platform.isIOS
                ? 'Mozilla/5.0 (iPhone; CPU iPhone OS 13_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.1 Mobile/15E148 Safari/604.1'
                : 'Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Mobile Safari/537.36',
          ),
          android: AndroidInAppWebViewOptions(disableDefaultErrorPage: true)),
      onWebViewCreated: (controller) {
        webViewController = controller;
      },
      onLoadStart: (controller, url) async {
        if (url != null) {
          webViewNotifier.loading();
          webViewNotifier.clearError();
          try {
            if (url
                .toString()
                .startsWith('notionsample://oauth/callback?code')) {
              await notionOauthApi.authenticate(url.toString());
              webViewNotifier.hide();
              ref.read(notionAuthProvider.notifier).getNotionWorkspace();
            }
          } catch (e) {
            throw Exception(e);
          }
        }
      },
      onLoadStop: (controller, url) async {
        webViewNotifier.loaded();
      },
      onLoadError: (controller, url, code, message) {
        // allow redirect to notionsample://oauth/callback
        if (url.toString().startsWith('notionsample://oauth/callback?code')) {
          return;
        }
        webViewNotifier.error(message);
      },
    );
  }
}
```

webviewの設定について2点補足説明します。

### 1. userAgentの設定について

今回はnotionにgoogleアカウントでログインするんですが、inappwebviewなどを使用した埋め込みブラウザでのOAuthリクエストはセキュリティ上の理由から許可されていません。なので、デフォルト設定ではdisallowed useragent というエラーになります。7年前になりますが[公式のブログ](https://developers.googleblog.com/2016/08/modernizing-oauth-interactions-in-native-apps.html)にも書かれています。

調べてみると、inappwebviewのuserAgentプロパティを利用してこの対策を回避することができるみたいです。
https://stackoverflow.com/questions/62730993/problem-in-google-login-in-canva-through-webview-in-flutter
userAgentを以下のように設定します。

```dart
InAppWebView(
    initialOptions: InAppWebViewGroupOptions(
        userAgent: Platform.isIOS
            ? 'Mozilla/5.0 (iPhone; CPU iPhone OS 13_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.1 Mobile/15E148 Safari/604.1'
            : 'Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Mobile Safari/537.36',
    ),
) 
```

**この方法では、webviewを使いながら、リクエストを端末のデフォルトブラウザからのものに見せかけることでGoogleのセキュリティ対策を回避しているため、セキュリティ上あまりよろしくないかもしれません。また、将来的に使えなくなる可能性もあります。**

url_launcherとuni_linksを使うのが正攻法ぽいですが、認証処理完了後にブラウザ画面をプログラム的に閉じることができない（ユーザが手動で閉じるしかない）ので、こちらの方法にしました。
**もっといい方法を知っている方は教えてください！**

**2024/1/29 追記**
url_launcherとuni_linksでOAuthを実装する記事を書きました（まだ期待通りの挙動ではない）

https://qiita.com/meique/items/b1551eb0dbbef83918c9

Inappwebviewパッケージの[InAppBrowser](https://inappwebview.dev/docs/5.x.x/in-app-browsers/in-app-browser)というのも試してみたけど、結局userAgentを設定しないとはじかれるみたい。

### 2. androidの設定について

android emulatorでアプリを実行してみたところ、認証後のリダイレクトで以下のような画面が一瞬表示されました。
![スクリーンショット 2024-01-29 21.55.49.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3065616/2b83bacc-89f1-35fb-5113-f189b520bb11.png)

エラー内容的にはカスタムURLスキームが認識できていない？ようですが、すぐにアプリの画面に戻るので、どうなっているのかよくわかりません。

根本的な解決ではないですが、以下のようにエラーを非表示にしました。

```dart
initialOptions: InAppWebViewGroupOptions(
    android: AndroidInAppWebViewOptions(disableDefaultErrorPage: true)),
```

ただこれだとちゃんとしたエラーも出なくなってしまう...?
**こちらも何かご存じの方は教えて欲しいです！**

以上で認証処理の実装は完了です。

最後にGithub pagesからリダイレクトされた時にiOSとandroidアプリを開くように設定します。

### iOSとandroideでのdeeplinkの設定

Github pagesからのリダイレクトでアプリを開くように設定します。
iOSは、`iOS/Runner/Info.plist`に以下を追記します。

```xml:Info.plist
<key>FlutterDeepLinkingEnabled</key>
<true/>
<key>CFBundleURLTypes</key>
<array>
<dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLSchemes</key>
    <array>
        <string>notionsample</string> <!-- 適宜カスタムURLスキームを設定 -->
    </array>
</dict>
</array>
```

androidは`android/app/src/main/AndroidManifest.xml`のactivityタグ内に以下を追記します。

```xml:AndroidManifest.xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="notionsampley"  />  <!-- 適宜カスタムURLスキームを設定 -->
</intent-filter>
```

これで認証フローの実装は完了です。

一旦休憩しましょ〜

## 4. Notionのデータベースにアクセス

認証が実装できたので、次はNotionのデータベースの読み取りと書き込みを行います。
操作できるのは認証操作の際にアクセスを許可したデータベースのみです。また、一度integrationを連携すると、notion側の設定からアクセスを許可するデータベースやページを変更できます。
こちらも[ドキュメント](https://developers.notion.com/reference/intro)はしっかりしてますが、レスポンスの構造が中々ややこしいです...

以下のようにしてデータベースにアクセスします。

1. データベースを名前で検索しIDを取得する（今回は「サンプル」という名前のデータベースにしています）
2. データベースの「タイトル」プロパティを取得して表示する。
3. タイトルプロパティを入力して新たなデータ行を挿入する。

以下の3つのファイルを追加します。

```zsh
.
├── api
│   ├── notion_database_api.dart <- databaseエンドポイントにapiリクエストを送ったりするクラス
│   ├── notion_database_api.g.dart
│   ├── notion_oauth_api.dart
│   └── notion_oauth_api.g.dart
├── env
│   ├── env.dart
│   └── env.g.dart
├── main.dart
├── provider
│   ├── notion_auth_provider.dart
│   ├── notion_auth_provider.freezed.dart
│   ├── notion_auth_provider.g.dart
│   ├── notion_database_provider.dart　<- 取得したデータを管理するプロバイダ
│   ├── notion_database_provider.freezed.dart
│   ├── notion_database_provider.g.dart
│   ├── webview_provider.dart
│   ├── webview_provider.freezed.dart
│   └── webview_provider.g.dart
└── widget
    ├── notion_database_list_widget.dart <- 取得したデータを表示するウィジェット
    └── notion_login_webview_widget.dart
```

1つずつ説明していきます。

- `notion_database_api.dart` : databaseエンドポイントにapiリクエストを送ったりするクラス
  データベース名での検索、「タイトル」プロパティの取得、データの挿入の３つのメソッドを持つクラスです。認証時にアクセスを許可したデータベースのIDを取得するためのAPIエンドポイントがないようなので、名前で検索してIDを取得します。アクセストークンはsecure storageから取得します。

```notion_database_api.dart
import 'dart:convert';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:http/http.dart' as http;
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'notion_database_api.g.dart';

class NotionDatabaseApi {
  final _storage = const FlutterSecureStorage();
  static const databaseName = 'サンプル';
  Future<String?> searchDatabases() async {
    final apiKey = await _storage.read(key: 'access_token');
    final response = await http.post(
      Uri.parse('https://api.notion.com/v1/search'),
      headers: {
        'Authorization': 'Bearer $apiKey',
        'Notion-Version': '2022-06-28',
        'Content-Type': 'application/json',
      },
      body: jsonEncode({
        'query': databaseName,
        'filter': {
          'property': 'object',
          'value': 'database',
        },
        'sort': {
          'direction': 'ascending',
          'timestamp': 'last_edited_time',
        },
      }),
    );

    if (response.statusCode == 200) {
      final results = jsonDecode(response.body)['results'] as List<dynamic>;
      if (results.isNotEmpty) {
        for (var result in results) {
          final titles = result['title'] as List<dynamic>;
          for (var title in titles) {
            if (title['text']['content'] == databaseName) {
              return result['id'].toString();
            }
          }
        }
      }
      return null;
    } else {
      throw Exception('Failed to search database: ${response.body}');
    }
  }

  Future<List<String>> insertTitle(
      {required String databaseId, required String title}) async {
    final apiKey = await _storage.read(key: 'access_token');
    final response = await http.post(
      Uri.parse('https://api.notion.com/v1/pages'),
      headers: {
        'Authorization': 'Bearer $apiKey',
        'Notion-Version': '2022-06-28',
        'Content-Type': 'application/json',
      },
      body: jsonEncode({
        'parent': {
          "type": "database_id",
          "database_id": databaseId,
        },
        'properties': {
          'タイトル': {
            'title': [
              {
                'text': {
                  'content': title,
                },
              },
            ],
          },
        },
      }),
    );

    if (response.statusCode == 200) {
      return await fetchTitles(databaseId: databaseId);
    } else {
      throw Exception('Failed to search database: ${response.body}');
    }
  }

  Future<List<String>> fetchTitles({required String databaseId}) async {
    final apiKey = await _storage.read(key: 'access_token');
    final response = await http.post(
      Uri.parse('https://api.notion.com/v1/databases/$databaseId/query'),
      headers: {
        'Authorization': 'Bearer $apiKey',
        'Notion-Version': '2022-06-28',
        'Content-Type': 'application/json',
      },
    );

    if (response.statusCode == 200) {
      final results = jsonDecode(response.body)['results'] as List<dynamic>;

      if (results.isNotEmpty) {
        List<String> titles = [];
        for (var page in results) {
          var titleProperty =
              page['properties']['タイトル']['title'] as List<dynamic>;
          if (titleProperty.isNotEmpty) {
            titles.add(titleProperty[0]['text']['content'] as String);
          }
        }
        return titles;
      }
      return [];
    } else {
      throw Exception('Failed to search database: ${response.body}');
    }
  }
}

@riverpod
NotionDatabaseApi notionDatabaseApi(NotionDatabaseApiRef ref) {
  return NotionDatabaseApi();
}
```

- `notion_database_provider.dart` : 取得したタイトルリストの状態管理

```notion_database_provider.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:notion_sample/api/notion_database_api.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'notion_database_provider.g.dart';
part 'notion_database_provider.freezed.dart';

@freezed
class NotionDatabaseState with _$NotionDatabaseState {
  const factory NotionDatabaseState({
    required AsyncValue<List<String>> titles,
    required String newTitle,
    required bool isInputting,
    required bool isInserting,
  }) = _NotionDatabaseState;
}

@riverpod
class NotionDatabase extends _$NotionDatabase {
  @override
  NotionDatabaseState build() {
    return const NotionDatabaseState(
      titles: AsyncValue.loading(),
      isInputting: false,
      newTitle: '',
      isInserting: false,
    );
  }

  Future<void> getDatabaseTitles() async {
    state = state.copyWith(titles: const AsyncValue.loading());
    final notionDatabaseApi = ref.read(notionDatabaseApiProvider);
    try {
      final databaseId = await notionDatabaseApi.searchDatabases();
      if (databaseId == null) {
        state = state.copyWith(titles: const AsyncValue.data([]));
        return;
      }
      final results = await notionDatabaseApi.fetchTitles(
        databaseId: databaseId,
      );
      state = state.copyWith(titles: AsyncValue.data(results));
    } catch (e, s) {
      state = state.copyWith(titles: AsyncValue.error(e, s));
    }
  }

  Future<void> insertTitle({required String title}) async {
    state = state.copyWith(titles: const AsyncValue.loading());
    state = state.copyWith(isInserting: true);
    final notionDatabaseApi = ref.read(notionDatabaseApiProvider);
    try {
      final databaseId = await notionDatabaseApi.searchDatabases();
      if (databaseId == null) {
        state = state.copyWith(titles: const AsyncValue.data([]));
        return;
      }
      final results = await notionDatabaseApi.insertTitle(
        databaseId: databaseId,
        title: title,
      );
      state = state.copyWith(titles: AsyncValue.data(results));
      state = state.copyWith(isInserting: false);
    } catch (e, s) {
      state = state.copyWith(titles: AsyncValue.error(e, s));
    }
  }

  void startInput() => state = state.copyWith(isInputting: true);
  void endInput() => state = state.copyWith(isInputting: false);
}
```

- `notion_database_list_widge.dart` : 入力欄と、取得したデータをリストで表示する
  
```notion_database_list_widge.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:notion_sample/provider/notion_database_provider.dart';

class NotionDatabaseListWidget extends ConsumerWidget {
  const NotionDatabaseListWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final notionDatabaseTitles =
        ref.watch(notionDatabaseProvider.select((state) => state.titles));
    final isInputting =
        ref.watch(notionDatabaseProvider.select((state) => state.isInputting));
    final isInserting =
        ref.watch(notionDatabaseProvider.select((state) => state.isInserting));
    final notionDatabaseNotifier = ref.read(notionDatabaseProvider.notifier);
    final TextEditingController controller = TextEditingController();
    final bottomSpace = MediaQuery.of(context).viewInsets.bottom;
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (notionDatabaseTitles is AsyncLoading && !isInserting) {
        ref.read(notionDatabaseProvider.notifier).getDatabaseTitles();
      }
    });
    return SingleChildScrollView(
        reverse: true,
        child: Padding(
            padding: isInputting
                ? EdgeInsets.only(bottom: bottomSpace)
                : EdgeInsets.zero,
            child: Column(children: [
              Padding(
                  padding: const EdgeInsets.all(16),
                  child: TextField(
                    controller: controller,
                    onTap: () => notionDatabaseNotifier.startInput(),
                    decoration: InputDecoration(
                      prefixIcon: isInputting
                          ? IconButton(
                              icon: const Icon(Icons.arrow_back),
                              onPressed: () {
                                FocusScope.of(context).unfocus();
                                notionDatabaseNotifier.endInput();
                              })
                          : null,
                      border: const OutlineInputBorder(),
                      hintText: 'タイトル',
                    ),
                    onSubmitted: (value) {
                      notionDatabaseNotifier.insertTitle(title: value);
                      notionDatabaseNotifier.endInput();
                      controller.clear();
                    },
                  )),
              notionDatabaseTitles.when(
                data: (data) {
                  if (data.isEmpty) {
                    return const Center(child: Text('データがありません'));
                  }
                  return Padding(
                      padding: const EdgeInsets.all(10),
                      child: SizedBox(
                          height: 250,
                          child: ListView.builder(
                              itemCount: data.length,
                              itemBuilder: (context, index) {
                                final title = data[index];
                                return Card(
                                    child: ListTile(
                                  title: Text(title),
                                ));
                              })));
                },
                loading: () => const CircularProgressIndicator(),
                error: (error, stackTrace) => Text(error.toString()),
              )
            ])));
  }
}
```

**認証直後はなぜかデータが取得できません！**
数回リロードするとデータが読み込まれます。この挙動はﾁｮｯﾄﾜｶﾗﾝ。

### 最後に

みんなもNotionを連携して使い倒そう！

### 参考にさせていただいた記事

https://zenn.dev/matsumaru/articles/fd63cf2793638f
https://qiita.com/yufuku/items/24dac97e6052b2571386
