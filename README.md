![Laravel best practices](/images/logo-english.png?raw=true)

Translations:

- [Русский : Russian](russian.md)
- [임영록(cherrypick) : Korean](korean.md)

이 문서가 도움이 되셨다면 현재 레퍼지토리뿐만 아니라, 번역 전 레퍼지토리도 Star 한 번씩 눌러주시면 감사하겠습니다. :D
<br>
레퍼지토리: https://github.com/alexeymezenin/laravel-best-practices

이 문서는 라라벨 프레임워크에서 객체지향 디자인의 5원칙(SOLID), 패턴 등을 적용한 내용이 아닙니다. 라라벨 프레임워크로 프로젝트를 진행하면서 놓칠 수 있는 Best practice에 대해 정리한 글입니다.



## Contents
[단일 책임 원칙](#단일-책임-원칙)

[모델은 무겁게, 컨트롤러는 가볍게](#모델은-무겁게-컨트롤러는-가볍게)

[Validation-유효성 검사](#validation-유효성-검사)

[비즈니스 로직은 서비스 클래스에 있어야 합니다.](#비즈니스-로직은-서비스-클래스에-있어야-합니다)

[중복 배제(Don't repeat yourself)](#중복-배제dont-repeat-yourself)

[Query Builder, raw SQL 쿼리보다 Eloquent를 사용하는 것이 좋습니다.](#query-builder-raw-sql-쿼리보다-eloquent를-사용하는-것이-좋습니다)

[Mass assignment-대량 할당](#mass-assignment-대량-할당)

[블레이드 템플릿에서 쿼리를 실행하지 않습니다. 그리고 즉시 로딩을 사용합니다.(N + 1 문제)](#블레이드-템플릿에서-쿼리를-실행하지-않습니다-그리고-즉시-로딩을-사용합니다n--1-문제)

[코드에 주석을 작성합니다. 그러나 주석보다 의미있는 메서드 이름과 변수 이름을 사용하는 것이 더 좋습니다.](#코드에-주석을-작성합니다-그러나-주석보다-의미있는-메서드-이름과-변수-이름을-사용하는-것이-더-좋습니다)

[블레이드 템플릿에 js와 css를 작성하지 말고 php 클래스에 html을 작성하지 않습니다.](#do-not-put-js-and-css-in-blade-templates-and-do-not-put-any-html-in-php-classes)

[Use config and language files, constants instead of text in the code](#use-config-and-language-files-constants-instead-of-text-in-the-code)

[Use standard Laravel tools accepted by community](#use-standard-laravel-tools-accepted-by-community)

[Follow Laravel naming conventions](#follow-laravel-naming-conventions)

[Use shorter and more readable syntax where possible](#use-shorter-and-more-readable-syntax-where-possible)

[Use IoC container or facades instead of new Class](#use-ioc-container-or-facades-instead-of-new-class)

[Do not get data from the `.env` file directly](#do-not-get-data-from-the-env-file-directly)

[Store dates in the standard format. Use accessors and mutators to modify date format](#store-dates-in-the-standard-format-use-accessors-and-mutators-to-modify-date-format)

[Other good practices](#other-good-practices)

### **단일 책임 원칙**

클래스와 메서드는 하나의 책임만 있어야 합니다.

나쁜 예:

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

좋은 예:

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[🔝 목차로 돌아가기](#contents)

### **모델은 무겁게, 컨트롤러는 가볍게**

DB와 관련된 로직은 Eloquent 모델이나 Repository 클래스에 작성되어야 합니다. 

나쁜 예:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

좋은 예:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝 목차로 돌아가기](#contents)

### **Validation-유효성 검사**

유효성 검사 로직을 컨트롤러에서 Request 클래스로 옮깁니다.

나쁜 예:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

좋은 예:

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[🔝 목차로 돌아가기](#contents)

### **비즈니스 로직은 서비스 클래스에 있어야 합니다.**

컨트롤러는 하나의 책임만 가지기 때문에 비즈니스 로직은 서비스 클래스에 있어야 합니다.


나쁜 예:

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

좋은 예:

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[🔝 목차로 돌아가기](#contents)

### **중복 배제(Don't repeat yourself)**

코드를 재사용합니다. 단일 책임 원칙뿐만 아니라 블레이드 템플릿, Eloquent 스코프 등은 코드의 중복을 피할 수 있도록 도와줍니다.

나쁜 예:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

좋은 예:

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[🔝 목차로 돌아가기](#contents)

### **Query Builder, raw SQL 쿼리보다 Eloquent를 사용하는 것이 좋습니다.**

Eloquent를 사용하면 읽기 쉽고 유지 보수할 수 있는 코드를 작성할 수 있습니다. Eloquent는 소프트 삭제, 이벤트, 스코프 등 좋은 기능이 있습니다.

나쁜 예:


```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

좋은 예:

```php
Article::has('user.profile')->verified()->latest()->get();
```

[🔝 목차로 돌아가기](#contents)

### **Mass assignment-대량 할당**

나쁜 예:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Add category to article
$article->category_id = $category->id;
$article->save();
```

좋은 예:

```php
$category->article()->create($request->all());
```

[🔝 목차로 돌아가기](#contents)

### **블레이드 템플릿에서 쿼리를 실행하지 않습니다. 그리고 즉시 로딩을 사용합니다.(N + 1 문제)**

나쁜예 (유저 전체를 가져오는 쿼리(1번) + 해당 유저의 프로필을 가져오는 쿼리(100번) = 101번 실행):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

좋은 예 (유저 전체를 가져오는 쿼리(1번) + 해당 유저의 프로필을 가져오는 쿼리(1번) = 2번 실행):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[🔝 목차로 돌아가기](#contents)

### **코드에 주석을 작성합니다. 그러나 주석보다 의미있는 메서드 이름과 변수 이름을 사용하는 것이 더 좋습니다.**

나쁜 예:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

조금 더 나은 예:

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

좋은 예:

```php
if ($this->hasJoins())
```

[🔝 목차로 돌아가기](#contents)

### **블레이드 템플릿에 JS와 CSS를 작성하지 말고 PHP 클래스에 HTML을 작성하지 않습니다.**

나쁜 예:

```php
let article = `{{ json_encode($article) }}`;
```

조금 더 나은 예:

```php
<input id="article" type="hidden" value="{{ json_encode($article) }}">

Or

<button class="js-fav-article" data-article="{{ json_encode($article) }}">{{ $article->name }}<button>
```

자바스크립트 파일:

```php
let article = $('#article').val();
```

The best way is to use specialized PHP to JS package to transfer the data.

[🔝 목차로 돌아가기](#contents)

### **코드에 텍스트로 작성하지 말고, 설정 파일, 언어 파일, 상수 등을 사용합니다.**

나쁜 예:

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

좋은 예:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[🔝 목차로 돌아가기](#contents)

### **라라벨 커뮤니티에서 수용하는 표준 라라벨 도구를 사용합니다.**

써드파티 패키지 및 도구 대신 내장되어있는 라라벨 기능과 커뮤니티 패키지를 사용합니다. 프로젝트에 참여하게 되는 개발자는 새로운 도구에 대해 학습을 해야합니다. 또한 써드파티 패키지나 도구를 사용할 때 라라벨 커뮤니티의 도움을 받을 수 있는 기회가 줄어듭니다. 
Prefer to use built-in Laravel functionality and community packages instead of using 3rd party packages and tools. Any developer who will work with your app in the future will need to learn new tools. Also, chances to get help from the Laravel community are significantly lower when you're using a 3rd party package or tool. Do not make your client pay for that.

Task | Standard tools | 3rd party tools
------------ | ------------- | -------------
Authorization | Policies | Entrust, Sentinel and other packages
Compiling assets | Laravel Mix | Grunt, Gulp, 3rd party packages
Development Environment | Homestead | Docker
Deployment | Laravel Forge | Deployer and other solutions
Unit testing | PHPUnit, Mockery | Phpspec
Browser testing | Laravel Dusk | Codeception
DB | Eloquent | SQL, Doctrine
Templates | Blade | Twig
Working with data | Laravel collections | Arrays
Form validation | Request classes | 3rd party packages, validation in controller
Authentication | Built-in | 3rd party packages, your own solution
API authentication | Laravel Passport | 3rd party JWT and OAuth packages
Creating API | Built-in | Dingo API and similar packages
Working with DB structure | Migrations | Working with DB structure directly
Localization | Built-in | 3rd party packages
Realtime user interfaces | Laravel Echo, Pusher | 3rd party packages and working with WebSockets directly
Generating testing data | Seeder classes, Model Factories, Faker | Creating testing data manually
Task scheduling | Laravel Task Scheduler | Scripts and 3rd party packages
DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[🔝 목차로 돌아가기](#contents)

### **라라벨 네이밍 규칙을 따릅니다.**

 [PSR 표준](http://www.php-fig.org/psr/psr-2/)을 따릅니다.
 
또한 라라벨 커뮤니티에서 수용하고 있는 네이밍 규칙을 따릅니다:

What | How | Good | Bad
------------ | ------------- | ------------- | -------------
Controller | singular | ArticleController | ~~ArticlesController~~
Route | plural | articles/1 | ~~article/1~~
Named route | snake_case with dot notation | users.show_active | ~~users.show-active, show-active-users~~
Model | singular | User | ~~Users~~
hasOne or belongsTo relationship | singular | articleComment | ~~articleComments, article_comment~~
All other relationships | plural | articleComments | ~~articleComment, article_comments~~
Table | plural | article_comments | ~~article_comment, articleComments~~
Pivot table | singular model names in alphabetical order | article_user | ~~user_article, articles_users~~
Table column | snake_case without model name | meta_title | ~~MetaTitle; article_meta_title~~
Model property | snake_case | $model->created_at | ~~$model->createdAt~~
Foreign key | singular model name with _id suffix | article_id | ~~ArticleId, id_article, articles_id~~
Primary key | - | id | ~~custom_id~~
Migration | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Method | camelCase | getAll | ~~get_all~~
Method in resource controller | [table](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Method in test class | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Variable | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
Collection | descriptive, plural | $activeUsers = User::active()->get() | ~~$active, $data~~
Object | descriptive, singular | $activeUser = User::active()->first() | ~~$users, $obj~~
Config and language files index | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
View | snake_case | show_filtered.blade.php | ~~showFiltered.blade.php, show-filtered.blade.php~~
Config | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
Contract (interface) | adjective or noun | Authenticatable | ~~AuthenticationInterface, IAuthentication~~
Trait | adjective | Notifiable | ~~NotificationTrait~~

[🔝 Back to contents](#contents)

### **될 수 있으면 짧고 읽기 쉬운 문법을 사용합니다.**

나쁜 예:

```php
$request->session()->get('cart');
$request->input('name');
```

좋은 예:

```php
session('cart');
$request->name;
```

더 많은 예시:

Common syntax | Shorter and more readable syntax
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? $object->relation->id : null }` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[🔝 목차로 돌아가기](#contents)

### **new Class 대신 IoC 컨테이너 또는 파사드를 사용합니다.**

new Class 문법은 클래스 간의 결합도를 높이고 테스트를 복잡하게 만듭니다. new Class 문법 대신에 IoC 컨테이너 또는 파사드를 사용합니다.

나쁜 예:

```php
$user = new User;
$user->create($request->all());
```

좋은 예:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->all());
```

[🔝 목차로 돌아가기](#contents)

### **.env 파일에서 직접 데이터를 가져오지 않습니다.**

데이터를 설정 파일에 전달한 다음 `config()` helper 함수를 통해 애플리케이션에서 데이터를 사용합니다.

나쁜 예:

```php
$apiKey = env('API_KEY');
```

좋은 예:

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

[🔝 목차로 돌아가기](#contents)

### **날짜를 표준 형식으로 저장합니다. accessors(get), mutators(set)을 사용해 날짜 형식을 수정합니다.**

나쁜 예:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

좋은 예:

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at']
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

[🔝 목차로 돌아가기](#contents)

### **다른 좋은 사례**

라우트 파일에 로직을 작성하지 않습니다.

블레이드 템플릿에 바닐라 PHP의 사용을 최소화합니다.

[🔝 목차로 돌아가기](#contents)
