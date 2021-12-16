# docker_laravel_vue_redis_echo
vue.js、redis、echo-serverを使えるLaravel環境を作るためのdocker-compose

## 概要
docker-composeを使って最小限のvue.js、redis、echo-server付きのLaravel(LEMP)環境を構築することができます。

## デモ
https://user-images.githubusercontent.com/63027441/146301131-a3d0d38a-9465-4105-a113-0ea6152d8c39.mp4

## 前提条件
(まず https://github.com/Asano-Naoki/docker_laravel_minimum を試すことをおすすめします。)

このリポジトリを使う前に、ブロードキャスト機能のあるLaravelプロジェクトディレクトリを用意してください。

新しいLaravelプロジェクトディレクトリを作る場合は、以下の手順に従ってください。

### 基本設定

1. まっさらなLaravelプロジェクトディレクトリを作ります。
```
docker run --rm -v $PWD:/app composer create-project laravel/laravel example-broadcast-app
```
2. .envファイルを編集します。
```
-DB_HOST=127.0.0.1
+DB_HOST=mysql
-DB_USERNAME=root
-DB_PASSWORD=
+DB_USERNAME=test_user
+DB_PASSWORD=test_pass

-BROADCAST_DRIVER=log
+BROADCAST_DRIVER=redis
-QUEUE_CONNECTION=sync
-SESSION_DRIVER=file
+QUEUE_CONNECTION=redis
+SESSION_DRIVER=redis
-REDIS_HOST=127.0.0.1
+REDIS_HOST=redis
+REDIS_PREFIX=
```
注:
"REDIS_PREFIX="を付け加えることが重要です。Laravelは自動的にredisのプレフィックスを設定します。それがややこしいので、REDIS_PREFIXを空白にします。config/database.phpを編集することも可能です。

3. このリポジトリ内のファイルとディレクトリをすべて、gitコマンド（fetchとmerge）か手動で、Laravelプロジェクトディレクトリにコピーします。
```
git init
git add .
git commit -m "first commit"
git remote add origin git@github.com:Asano-Naoki/docker_laravel_vue_redis_echo.git
git fetch
git merge origin/main --allow-unrelated-histories
```
4. docker-composeを開始します。
```
docker-compose up -d
```
5. usersテーブルを作ります。
```
docker-compose exec php sh
...
(after entering sh of php)
...
php artisan migrate
```
注: 
最初のセットアップ時には、mysqlファイルがすべて作られていないために、php artisan migrateコマンドが失敗するかもしれません。その場合は数分待ってください。

### Vue.js

1. laravel/uiをインストールします。
```
docker run --rm -v $PWD:/app composer require laravel/ui
```
注:
laravel/uiをインストールすることは必須ではありませんが、これがvue.jsを使うために一番簡単なやり方です。

2. vue.js関係のファイルを発行します。
```
docker-compose exec php sh
...
(after entering sh of php)
...
php artisan ui vue
```

3. resources/views/welcome.blade.phpを編集します。
```
(before </head>)
+<script src="/js/app.js" defer></script>

(around line 50)
-<div class="mt-8 bg-white dark:bg-gray-800 overflow-hidden shadow sm:rounded-lg">
+<div id="app" class="mt-8 bg-white dark:bg-gray-800 overflow-hidden shadow sm:rounded-lg">
+<example-component></example-component>
```
注:
"defer"がないと"Cannot find element: #app"が発生します。

4. assetsをコンパイルします。
```
docker run --rm  -v $PWD:/usr/src/app -w /usr/src/app node npm install && npm run dev
```
注:
"npm ERR! code ELIFECYCLE"が発生したら、もう一度同じコマンドを実行してください。

5. welcome page(http://localhost)に行きます。

"Example Component  
I'm an example component."  

という文字列が見えたら、vue.jsは適切に動いています。

### Redis

1. redisの接続を確認します。
```
docker-compose exec redis sh
...
(after entering sh of redis)
...
redis-cli
...
(after entering redis-cli)
...
127.0.0.1:6379> keys *
```
"laravel_database_laravel_cache:xxxxxxxxxxxxxxxxxxxx"といったセッション情報が表示されるはずです。これはlaravelがredisに接続できてセッション情報がredisに保存されているしるしです。

注:
セッション情報を保存するためには、SESSION_DRIVERがfileではなくredisに設定される必要があります。[基本設定](#基本設定)を参照。

### laravel-echo-server

1. laravel-echo-server.jsonファイルを作成します。
```
docker-compose run --rm echo-server laravel-echo-server init
```
いくつか質問されるので、以下のように答えてください。
```
? Do you want to run this server in development mode? Yes
? Which port would you like to serve from? 6001
? Which database would you like to use to store presence channel members? redis
? Enter the host of your Laravel authentication server. http://localhost
? Will you be serving on http or https? http
? Do you want to generate a client ID/Key for HTTP API? No
? Do you want to setup cross domain access to the API? No
? What do you want this config to be saved as? laravel-echo-server.json
```

2. laravel-echo-server.jsonファイルを編集します。
```
-"redis": {},
+"redis": {"host": "redis", "port": 6379},
```

3. docker-composeを開始します。
```
docker-compose up -d
```

4. laravel-echo-serverの準備ができていることを確認します。
```
docker-compose logs echo-server
```
laravel-echo-serverの準備ができていれば、以下のようなログを見ることができます。
```
echo-server_1  | 
echo-server_1  | L A R A V E L  E C H O  S E R V E R
echo-server_1  | 
echo-server_1  | version 1.6.2
echo-server_1  | 
echo-server_1  | ⚠ Starting server in DEV mode...
echo-server_1  | 
echo-server_1  | ✔  Running at localhost on port 6001
echo-server_1  | ✔  Channels are ready.
echo-server_1  | ✔  Listening for http events...
echo-server_1  | ✔  Listening for redis events...
echo-server_1  | 
echo-server_1  | Server ready!
echo-server_1  | 
```

### Laravel app

1. config/app.phpのBroadcast Service Providerのコメントアウトを外します。
```
-// App\Providers\BroadcastServiceProvider::class,
+App\Providers\BroadcastServiceProvider::class,
```

2. laravel-echoとsocket.io-clientをインストールします。
```
docker run --rm  -v $PWD:/usr/src/app -w /usr/src/app node npm install laravel-echo socket.io-client@2.4.0
```
注:
最新のsocke.io-clientではwebsocketの確立に失敗します。バージョン2.4.0ならうまくいきます。

3. resources/js/bootstrap.jsにEchoの設定を書きます。
```
+import Echo from 'laravel-echo';
+window.io = require('socket.io-client');
+window.Echo = new Echo({
+    broadcaster: 'socket.io',
+    host: window.location.hostname + ':6001'
+});
```
注:
pusherを使う場合のEchoの設定はコメントアウトの形で用意されていますが、redisを使う場合の設定とは異なります。

4. routes/web.phpにメッセージデータをpostするためのルートを加えます。
```
+ Route::post('/chat', [App\Http\Controllers\ChatController::class, 'index']);
```

5. controllerとeventを作成します。
```
docker-compose exec php sh
...
(after entering sh of php)
...
php artisan make:controller ChatController
php artisan make:event MessageCreated
```

6. app/Http/Controllers/ChatController.phpを編集します。
```
(at the top level)
+ use App\Events\MessageCreated;

(in the class ChatController)
+    public function index(Request $request) {
+        $message = $request->input('message');
+        MessageCreated::dispatch($message);
+        return $message;
+    }
```

7. app/Events/MessageCreated.phpを編集します。
```
(at the top level)
- class MessageCreated
+ class MessageCreated implements ShouldBroadcast

(in the class MessageCreated)
+ public $message;
-    public function __construct()
-    {
-        //
-    }
+    public function __construct($message)
+    {
+        $this->message = $message;
+    }
-     public function broadcastOn()
-    {
-        return new PrivateChannel('channel-name');
-    }
+    public function broadcastOn()
+    {
+        return new Channel('chat');
+    }
```

8. resources/js/components/ExampleComponent.vueを編集します。
```
-<div class="card-body">
-    I'm an example component.
-</div>
+<div class="card-body">
+    I'm an example component.
+    <form>
+        <input type="text" v-model="message" style="border:solid">
+        <button v-on:click.prevent="submit">submit</button>
+    </form>
+    <ul>
+        <li v-for="message in messages">
+            {{ message }}
+        </li>
+    </ul>
+</div>

-export default {
-    mounted() {
-        console.log('Component mounted.')
-    }
-}
+export default {
+    data() {
+        return {
+            message: '',
+            messages: [],
+        }
+    },
+    mounted() {
+        Echo.channel('chat')
+            .listen('MessageCreated', (e) => {
+                this.messages.push(e.message);
+            });
+    },
+    methods: {
+        submit() {
+            axios
+                .post('/chat', {'message' : this.message})
+                .then(response => {})
+                .catch(error => {});
+        },
+    },
+}
```

9. assetsをコンパイルします。
```
docker run --rm  -v $PWD:/usr/src/app -w /usr/src/app node npm run dev
```

10. ブラウザを２つ起動し、それらの間でチャットをします。

welcome page(http://localhost)に行くと[デモ](#demo)のようにチャットできます。


## 使い方
このリポジトリはリアルタイムインタラクションのためのインフラです。特別な使い方はありません。laravelのブロードキャストをお使いください。


## 著者
[浅野直樹](https://asanonaoki.com/blog/)


## ライセンス
MITライセンスの元にライセンスされています。詳細は[LICENSE](/LICENSE)をご覧ください。



