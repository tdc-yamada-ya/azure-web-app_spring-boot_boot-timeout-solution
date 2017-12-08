# Azure Web App で Spring Boot アプリケーションを動かした時に起動タイムアウトする場合の解決策

Microsoft Azure のドキュメントに App Service に Spring Boot アプリケーションをデプロイする方法が書かれています。

[Deploy a Spring Boot Application to the Azure App Service](https://docs.microsoft.com/en-us/java/azure/spring-framework/deploy-spring-boot-java-web-app-on-azure)

これを実際に試してみたときに発生した起動タイムアウトの問題についてまとめます。

## 発生した事象について

Jarパッケージファイルにしておよそ40MBほどのSpring Bootアプリケーションを作成してAzureのWeb Appとしてデプロイしたところ以下のような問題が発生しました。

* ローカルでは5秒から10秒程度で起動するが、Web Appでは2分以上かかる
* ログストリームを見てみるとSpring Bootの起動ログがでてからおよそ２分後に、再度、起動ログが表示される
* プロセスエクスプローラーを見てみると2分後のタイミングでJavaプロセスが2重に起動している
* 2分経過したプロセスは終了させられている

上記のようにSpring Bootアプリケーションがいつまで経っても起動しないという問題が発生していました。

Web App のパフォーマンスの問題やいつまでも起動しない問題はStackOverflowにも投稿されており、それなりに悩んでいる人は多いように思います。

[Stack Overflow](https://stackoverflow.com/questions/40496521/spring-boot-jar-on-azure-websites-performance-issues)

どこがボトルネックになっているのか2017-12-08現在はまだ調査できていませんが、どうしてもAzureでSpring Bootアプリケーションを稼動させたかったため、回避策を考えることにしました。

## Spring Bootの初期化処理が始まる前にダミーの組み込みHTTPサーバを起動して回避する

なぜかSpring Bootの初期化処理に2分以上もの時間がかかっていました。
初期化処理にかかる時間をなんとかして短縮できればいいのですが、確実に解決できそうな方法として、
あらかじめダミーの組み込みHTTPサーバを起動しておく方法を試したところ、ひとまずは上手くいきました。

Web Appのプロセスに大してHTTP_PLATFORM_PORTという環境変数にポート番号が入ります。
タイムアウトを防ぐには、なるべく早くこのポート番号でHTTPサーバを立ち上げる必要があるのですが、
Spring Bootはすべての初期化処理を完了させてからサーブレットコンテナを該当のポート番号で立ち上げるため、
どうしても初動が遅れてしまい、タイムアウトが発生しやすくなってしまいます。

そこでSpringApplicationを開始した後ApplicationEventを監視して、初期化が始まり環境設定が読み込まれた直後にダミーHTTPサーバを起動し、
サーブレットコンテナが立ち上がる前にダミーHTTPサーバを停止することで対処しました。

コードのイメージはこのようになります。
依存パッケージとしてApache HTTPComponentsを使用しています。

```java
public static void main(String[] args) throws Exception {
  SpringApplication application = new SpringApplication(Application.class);

  if (isStartupDummyHttpServerEnabled()) {
    application.addListeners(new ApplicationListener<ApplicationEvent>() {
      HttpServer httpServer = null;

      @Override
      public void onApplicationEvent(ApplicationEvent applicationEvent) {
        if (applicationEvent instanceof ApplicationEnvironmentPreparedEvent) {
          ApplicationEnvironmentPreparedEvent applicationEnvironmentPreparedEvent = (ApplicationEnvironmentPreparedEvent)applicationEvent;
          ConfigurableEnvironment configurableEnvironment = applicationEnvironmentPreparedEvent.getEnvironment();
          String serverPortString = configurableEnvironment.getProperty("server.port");

          httpServer = createStartupDummyHttpServer(Integer.parseInt(serverPortString));

          try {
            httpServer.start();
          } catch(Exception e) {
            // Ignore exception
          }
        } else if(applicationEvent instanceof ContextRefreshedEvent) {
          if (httpServer != null) {
            httpServer.shutdown(0, TimeUnit.SECONDS);
            httpServer = null;
          }
        }
      }
    });
  }

  application.run(args);
}

public static boolean isStartupDummyHttpServerEnabled() {
  final String STARTUP_DUMMY_HTTP_SERVER_ENABLED_PROPERTY_NAME = "startup.dummyHttpServerEnabled";
  final String STARTUP_DUMMY_HTTP_SERVER_ENABLED_ENV_NAME = "STARTUP_DUMMY_HTTP_SERVER_ENABLED";

  String startupDummyHttpServerEnabledString = System.getProperty(STARTUP_DUMMY_HTTP_SERVER_ENABLED_PROPERTY_NAME);

  if (startupDummyHttpServerEnabledString == null) {
    startupDummyHttpServerEnabledString = System.getenv(STARTUP_DUMMY_HTTP_SERVER_ENABLED_ENV_NAME);
  }

  return Boolean.valueOf(startupDummyHttpServerEnabledString);
}

public static HttpServer createStartupDummyHttpServer(int port) {
  HttpRequestHandler httpRequestHandler = new HttpRequestHandler() {
    @Override
    public void handle(HttpRequest request, HttpResponse response, HttpContext context) throws HttpException, IOException {
      response.setStatusCode(200);
    }
  };

  HttpServer httpServer = ServerBootstrap.bootstrap()
      .setListenerPort(port)
      .registerHandler("*", httpRequestHandler)
      .create();

  return httpServer;
}
```

ダミーHTTPサーバとしてJettyを使っても良かったのですが、上手く起動しなかったためApache HttpComponentsのHttpServerで代用しています。

上記のコードを動かすとSpring Bootが起動完了するまでは常に200 OKを返すように動くため、再起動を防止することができます。

## 他にもっと良いやり方が無いか

正直に言ってこの方法はバッドノウハウなので、もし他にいいやり方があれば教えてほしいのです。
私が候補として思いついたのは以下のとおりです。

* Azureの設定で起動タイムアウト時間を延ばす
* Spring Bootの設定でサーブレットコンテナを先に起動する

ただ上記の設定方法が見当たらず、こちらは断念しました。
もしくは初期化処理にかかる時間を短縮する手段を模索してもよいのですが、問題の起きたSpring Bootアプリケーションはスタンダードな設計であったため、こちらは優先度を落としました。
またスケールアップも手段の一つですが標準的なS1 Standardでこの現状なので、こちらも断念しています。

## サンプルコードについて

2017-12-08現在サンプルコードはこのリポジトリにないのですが、そのうち作るかもしれません。
