[日本語版 README はこちら](/README_ja.md)

# docker_laravel_vue_redis_echo
docker-compose for Laravel with vue.js, redis, and echo-server

## Summary
You can create minimum Laravel(LEMP) environment for broadcasting with vue.js, redis, laravel-echo-server using docker-compose.

## Demo
(insert a video later)

## Prerequisites
(I recommend you try https://github.com/Asano-Naoki/docker_laravel_minimum first.)

Before using this repository, prepare your Laravel project directory having broadcasting function.

If you'd like to create a new one, follow the directions below:

### Base set up

1. Create a brand new Laravel project directory.
```
docker run --rm -v $PWD:/app composer create-project laravel/laravel example-broadcast-app
```
2. Edit .env file.
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
NOTE:
Adding "REDIS_PREFIX=" is important. Laravel automatically sets redis prefix and it is confusing, so I set REDIS_PREFIX blank. You can achieve this by editing config/database.php, too.

3. Copy all the files and directory in this repository to the Laravel project directory with git command(fetch and merge) or manually.
```
git init
git add .
git commit -m "first commit"
git remote add origin git@github.com:Asano-Naoki/docker_laravel_vue_redis_echo.git
git fetch
git merge origin/main --allow-unrelated-histories
```
4. Start docker-compose.
```
docker-compose up -d
```
5. Create users table.
```
docker-compose exec php sh
...
(after entering sh of php)
...
php artisan migrate
```
NOTE:
At the first setup, php artisan migrate command may fail because not all the mysql files are created. In this situation, you have just to wait for a few minutes.

### Vue.js

1. Install laravel/ui.
```
docker run --rm -v $PWD:/app composer require laravel/ui
```
NOTE:
It is not necessary to install laravel/ui, however, it is the easiest way to use vue.js.

2. Publish the vue.js related files.
```
docker-compose exec php sh
...
(after entering sh of php)
...
php artisan ui vue
```

3. Edit resources/views/welcome.blade.php file.
```
(before </head>)
+<script src="/js/app.js" defer></script>

(around line 50)
-<div class="mt-8 bg-white dark:bg-gray-800 overflow-hidden shadow sm:rounded-lg">
+<div id="app" class="mt-8 bg-white dark:bg-gray-800 overflow-hidden shadow sm:rounded-lg">
+<example-component></example-component>
```
NOTE:
Withouse "defer", "Cannot find element: #app" occurs.

4. Compile the assets.
```
docker run --rm  -v $PWD:/usr/src/app -w /usr/src/app node npm install && npm run dev
```
NOTE:
If "npm ERR! code ELIFECYCLE" occurs, simply execute the same command again.


5. Visit welcome page(http://localhost).

The string  

"Example Component  
I'm an example component."  

should appear. This is the proof which shows vue.js works properly.

### Redis

1. Check redis connection.
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
Session information like "laravel_database_laravel_cache:xxxxxxxxxxxxxxxxxxxx" should be displayed. This means you connect laravel to redis and session information is stored there.

NOTE:
To store session information in redis, SESSION_DRIVER is need to be set "redis" instead of "file". [See Base](#base-set-up) set up section above.

### laravel-echo-server

1. Create laravel-echo-server.json file.
```
docker-compose run --rm echo-server laravel-echo-server init
```
You will be asked some question. Answer them like this:
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

2. Edit laravel-echo-server.json file.
```
-"redis": {},
+"redis": {"host": "redis", "port": 6379},
```

3. Start docker-compose.
```
docker-compose up -d
```

4. Confirm laravel-echo-server is ready.
```
docker-compose logs echo-server
```
If laravel-echo-server is ready, you can see the logs below:
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

1. Uncomment Broadcast Service Provider in config/app.php.
```
-// App\Providers\BroadcastServiceProvider::class,
+App\Providers\BroadcastServiceProvider::class,
```

2. Install laravel-echo and socket.io-client.
```
docker run --rm  -v $PWD:/usr/src/app -w /usr/src/app node npm install laravel-echo socket.io-client@2.4.0
```
NOTE:
Latest socke.io-client fails to establish websocket connection. Version 2.4.0 is fine.

3. Add Echo setting to resources/js/bootstrap.js.
```
+import Echo from 'laravel-echo';
+window.io = require('socket.io-client');
+window.Echo = new Echo({
+    broadcaster: 'socket.io',
+    host: window.location.hostname + ':6001'
+});
```
NOTE:
Echo setting for pusher is prepared in the state of comment out, but there are some differences between them(for pusher and for redis).

4. Add route for message data post to routes/web.php.
```
+ Route::post('/chat', [App\Http\Controllers\ChatController::class, 'index']);
```

5. Make the controller and event.
```
docker-compose exec php sh
...
(after entering sh of php)
...
php artisan make:controller ChatController
php artisan make:event MessageCreated
```

6. Edit app/Http/Controllers/ChatController.php.
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

7. Edit app/Events/MessageCreated.php.
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

8. Edit resources/js/components/ExampleComponent.vue
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

9. Compile the assets.
```
docker run --rm  -v $PWD:/usr/src/app -w /usr/src/app node npm run dev
```

10. Run two browsers and chat with each other.

Visit welcome page(http://localhost) and you can chat with each other as shown in the [Demo](#demo)


## Usage
This repository is an infraastructure for real-time interactions. There is no special usage. Just use laravel broadcasting.


## Author
[Asano Naoki](https://asanonaoki.com/blog/)


## License
Under the MIT License. See [LICENSE](/LICENSE) for details.



