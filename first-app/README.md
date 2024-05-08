# Laravel sailを使用したLaravelの開発環境の構築方法

## sec01 laravel sail 起動
- 参照：https://fadotech.com/mac-laravel-sail/

1\. プロジェクト作成
- Dockerを事前に起動させておく。
```
cd ~/Desktop
mkdir laravel_sail
cd laravel_sail
composer create-project laravel/laravel first-app "10.*" --prefer-dist
```
- first-app：任意のアプリ名でOK
- Laravelのバージョンは10で作成しているがこれも任意でOK

2\. docker-compose.ymlファイルを作成する
```
cd first-app
touch docker-compose.yml
```

3\. docker-compose.ymlファイルを編集する
```yml:docker-compose.yml
services:
    laravel.test:
        build:
            context: ./vendor/laravel/sail/runtimes/8.3
            dockerfile: Dockerfile
            args:
                WWWGROUP: '${WWWGROUP}'
        image: sail-8.3/app
        extra_hosts:
            - 'host.docker.internal:host-gateway'
        ports:
            - '${APP_PORT:-80}:80'
            - '${VITE_PORT:-5173}:${VITE_PORT:-5173}'
        environment:
            WWWUSER: '${WWWUSER}'
            LARAVEL_SAIL: 1
            XDEBUG_MODE: '${SAIL_XDEBUG_MODE:-off}'
            XDEBUG_CONFIG: '${SAIL_XDEBUG_CONFIG:-client_host=host.docker.internal}'
            IGNITION_LOCAL_SITES_PATH: '${PWD}'
        volumes:
            - '.:/var/www/html'
        networks:
            - sail
        depends_on:
            - mysql
            - redis
            - meilisearch
            - mailpit
            - selenium
    mysql:
        image: 'mysql/mysql-server:8.0'
        ports:
            - '${FORWARD_DB_PORT:-3306}:3306'
        environment:
            MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
            MYSQL_ROOT_HOST: '%'
            MYSQL_DATABASE: '${DB_DATABASE}'
            MYSQL_USER: '${DB_USERNAME}'
            MYSQL_PASSWORD: '${DB_PASSWORD}'
            MYSQL_ALLOW_EMPTY_PASSWORD: 1
        volumes:
            - 'sail-mysql:/var/lib/mysql'
            - './vendor/laravel/sail/database/mysql/create-testing-database.sh:/docker-entrypoint-initdb.d/10-create-testing-database.sh'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - mysqladmin
                - ping
                - '-p${DB_PASSWORD}'
            retries: 3
            timeout: 5s
    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        platform: linux/amd64
        links:
            - mysql:mysql
        ports:
            - 8080:80
        environment:
            MYSQL_USERNAME: '${DB_USERNAME}'
            MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
            PMA_HOST: mysql
        networks:
            - sail
    redis:
        image: 'redis:alpine'
        ports:
            - '${FORWARD_REDIS_PORT:-6379}:6379'
        volumes:
            - 'sail-redis:/data'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - redis-cli
                - ping
            retries: 3
            timeout: 5s
    meilisearch:
        image: 'getmeili/meilisearch:latest'
        ports:
            - '${FORWARD_MEILISEARCH_PORT:-7700}:7700'
        environment:
            MEILI_NO_ANALYTICS: '${MEILISEARCH_NO_ANALYTICS:-false}'
        volumes:
            - 'sail-meilisearch:/meili_data'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - wget
                - '--no-verbose'
                - '--spider'
                - 'http://localhost:7700/health'
            retries: 3
            timeout: 5s
    mailpit:
        image: 'axllent/mailpit:latest'
        ports:
            - '${FORWARD_MAILPIT_PORT:-1025}:1025'
            - '${FORWARD_MAILPIT_DASHBOARD_PORT:-8025}:8025'
        networks:
            - sail
    selenium:
        image: seleniarm/standalone-chromium
        extra_hosts:
            - 'host.docker.internal:host-gateway'
        volumes:
            - '/dev/shm:/dev/shm'
        networks:
            - sail
networks:
    sail:
        driver: bridge
volumes:
    sail-mysql:
        driver: local
    sail-redis:
        driver: local
    sail-meilisearch:
        driver: local
```

4\. Laravel sailを起動する
```
cd first-app
./vendor/bin/sail up -d
```
- 時間がかかるが、完了後
http://localhost/
へアクセスする。表示されていればOK

3\. Laravel sailを停止する
```
./vendor/bin/sail stop
```

## sec02 laravel sail エイリアスの作成
0\. エイリアスの作成
- vimで編集する
```
vim ~/.zshrc
```

- ファイルに追記する（追記場所はどこでも）
- 該当場所までカーソルを合わせたらコマンド：i 
```
alias sail='bash vendor/bin/sail'
```
- 追記したらコマンド 「esc」→「:」→「w」→「q」→Enter
- 「:wq」となっている状態でEnter押せば抜けられる。

- 設定を反映
```
source ~/.zshrc
```
- 反映後、ターミナルを再起動する

1\. docker-compose.ymlを編集する
- 参照：https://zenn.dev/holy0306/articles/2ba88724f5af69
```yml:docker-compose.yml
mysql:
    image: 'mysql/mysql-server:8.0'
    ports:
        - '${FORWARD_DB_PORT:-3306}:3306'
    environment:
        MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
        MYSQL_ROOT_HOST: '%'
        MYSQL_DATABASE: '${DB_DATABASE}'
        MYSQL_USER: '${DB_USERNAME}'
        MYSQL_PASSWORD: '${DB_PASSWORD}'
        MYSQL_ALLOW_EMPTY_PASSWORD: 1
    volumes:
        - 'sail-mysql:/var/lib/mysql'
        - './vendor/laravel/sail/database/mysql/create-testing-database.sh:/docker-entrypoint-initdb.d/10-create-testing-database.sh'
    networks:
        - sail
    healthcheck:
        test:
            - CMD
            - mysqladmin
            - ping
            - '-p${DB_PASSWORD}'
        retries: 3
        timeout: 5s
// 追加 start
phpmyadmin: 
    image: phpmyadmin/phpmyadmin
    platform: linux/amd64
    links:
        - mysql:mysql
    ports:
        - 8080:80
    environment:
        MYSQL_USERNAME: '${DB_USERNAME}'
        MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
        PMA_HOST: mysql
    networks:
        - sail
// 追加 end
redis:
    image: 'redis:alpine'
    ports:
        - '${FORWARD_REDIS_PORT:-6379}:6379'
    volumes:
        - 'sail-redis:/data'
    networks:
        - sail
    healthcheck:
        test:
            - CMD
            - redis-cli
            - ping
        retries: 3
        timeout: 5s
```

2\. laravel/breezeのパッケージを追加する
```
./vendor/bin/sail up -d
```

```
sail composer require laravel/breeze --dev
```

3\. larave/breezeのインストール
```
sail artisan breeze:install
```

```
 ┌ Which Breeze stack would you like to install? ───────────────┐
 │ Blade with Alpine                                            │
 └──────────────────────────────────────────────────────────────┘

 ┌ Would you like dark mode support? ───────────────────────────┐
 │ No                                                           │
 └──────────────────────────────────────────────────────────────┘

 ┌ Which testing framework do you prefer? ──────────────────────┐
 │ PHPUnit                                                      │
 └──────────────────────────────────────────────────────────────┘

   INFO  Installing and building Node dependencies.  
```

4\. マイグレーションファイルを実行（laravel/breezeのデフォルトファイルを実行）
- 実行前に.envファイルの中身を確認する
- Before
```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```
- After
```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```
- 実行
```
sail artisan migrate
```

4\. 言語・日時設定を変更する
```php:config/app.php
'timezone' => 'Asia/Tokyo',

'locale' => 'ja',

'faker_locale' => 'ja_JP',
```

## sec03 翻訳用のディレクトリを設置（laravel10より必要）
1\. 翻訳用のディレクトリを設置（laravel10より必要）
```
sail artisan lang:publish
```

2\. laravel/beezeの日本語化パッケージを使用
```
sail composer require askdkc/breezejp --dev
sail artisan breezejp
```

## sec04 記事投稿フォーム作成
- 『件名』『本文』にテキストエリアがあり、フォームの下に『送信する』ボタンがあるフォームを作成する。

1\. モデルとマイグレーションファイルの作成
```
sail artisan make:model Post -m
```
- database/migrations 配下と app/Modelsにファイルが作成される。

2\. database/migrations 配下に作成されたマイグレーションファイルに、postsテーブルを作成するためのtitleカラムとbodyカラムを設定する
- ./database/migrations/(年)_(月)_(日)_(時刻)_create_posts_table.php
```php:database/migrations/2024_05_08_232219_create_posts_table.php
public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('body');
            $table->timestamps();
        });
    }
```

- マイグレート実行し、データベースにpostsテーブルを作成
```
sail artisan migrate
```

3\. ビューファイルの作成
- resources/viewsの中にpostフォルダを作り、その中に create.blade.phpファイルを作る

```
mkdir resources/views/post
touch resources/views/post/create.blade.php
```

- create.blade.phpファイルを編集する
```
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            フォーム
        </h2>
    </x-slot>
 
    <div class="max-w-7xl mx-auto px-6">
        <form method="post" action="{{ route('post.store') }}">
        @csrf
            <div class="mt-8">    
                <div class="w-full flex flex-col">
                    <label for="title" class="font-semibold mt-4">件名</label>
                    <input type="text" name="title" class="w-auto py-2 border border-gray-300 rounded-md" id="title">
                </div>
            </div>
        
            <div class="w-full flex flex-col">
                <label for="body" class="font-semibold mt-4">本文</label> 
                <textarea name="body" class="w-auto py-2 border border-gray-300 rounded-md" id="body" cols="30" rows="5">
                </textarea>
            </div>
        
            <x-primary-button class="mt-4">
                送信する
            </x-primary-button>
        </form>
    </div>
</x-app-layout>
```

3\. ビューファイル表示用コードの追加
- PostControllerを作成する
```
sail artisan make:controller PostController
```

4\. PostControllerファイルを編集する
```php:app/Http/Controllers/PostController.php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\Post;

class PostController extends Controller
{
    public function create(){
        return view('post.create');
    }

    public function store(Request $request) {
        $post = Post::create([
            'title' => $request->title,
            'body' => $request->bosy
        ]);
        return back();
    }
}
```

5\. ルーティングの設定(ビューファイル表示用のルート設定)
- routes/web.phpの中に、下記のuse宣言とフォーム表示用のルート設定と投稿データ保存用のルート設定を追加する。
```php:routes/api.php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\PostController;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider and all of them will
| be assigned to the "api" middleware group. Make something great!
|
*/

Route::get('post/create', [PostController::class, 'create']);
Route::post('post', [PostController::class, 'store'])->name('post.store');
Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});
```

6\. ローカルサーバーを立ち上げて確認する

```
php artisan serve
```
- http://localhost/post/create
- 件名、本文に何か文字を入力して「送信する」ボタンをクリック
- 入力フォームがリセットされる。

