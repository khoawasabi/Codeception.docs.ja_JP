# 高度な使用法

この章ではあなたのテストのエクスペリエンスを向上させ、プロジェクトをより良い組織に保つためのいくつかのテクニックとオプションについて取り上げます。

## Cestクラス

もしCept形式に対して、クラスのような構造を持たせたい場合、プレーンなPHPの替わりにCest形式を利用することができます。
Cest形式はとても単純でCept形式のシナリオと完全な互換性があります。これはつまり、テストが非常に長くなり分割したいと感じた場合、簡単にクラスに移動することができることを意味します。

次のコマンドによりCestを作成することができます。

```bash
$ php codecept generate:cest suitename CestName
```

生成されたファイルは次のようになります。

```php
<?php
class BasicCest
{
    public function _before(\AcceptanceTester $I)
    {
    }

    public function _after(\AcceptanceTester $I)
    {
    }

    // tests
    public function tryToTest(\AcceptanceTester $I)
    {    
    }
}
```

**それぞれのpublicなメソッド（「_」で始まるものを除く）はテストとして実行され**、最初の引数としてアクタークラスを、2つ目の引数して`$scenario`変数を受け取ります。

`_before`と`_after`メソッドはそのクラスのテストに対する共通のセットアップとティアダウンのために使用することができます。このことが、類似の処理をヘルパークラスのみに依存するCept形式と比較して、Cest形式をより柔軟にしています。

ご覧のように、`tryToTest`メソッドにアクターオブジェクトを渡しています。このオブジェクトを使ってこれまでにやってきた方法でシナリオを描くことができます。

```php
<?php
class BasicCest
{
    // test
    public function checkLogin(\AcceptanceTester $I)
    {
        $I->wantTo('log in to site');
        $I->amOnPage('/');
        $I->click('Login');
        $I->fillField('username', 'john');
        $I->fillField('password', 'coltrane');
        $I->click('Enter');
        $I->see('Hello, John');
        $I->seeInCurrentUrl('/account');
    }
}
```

このように、Cestクラスは`\Codeception\Test\Unit`や`PHPUnit_Framework_TestCase`といった親クラスを持ちません。意図的にそうしています。これにより、子クラスで使用することができる共通の振る舞いや処理を伴ってクラスを拡張することができます。ただし、それらのメソッドはテストとして実行されないよう`protected`なメソッドとして定義することを忘れないでください。

また、テストが`error`となったり失敗した時に呼ばれる`_failed`メソッドについてもCestクラスに定義することができます。

## 依存性の注入

CodeceptionはCest形式と`\Codeception\TestCase\Test`クラスについて、簡単なDIをサポートしています。これはつまり、あなたは`_inject()`という特別なメソッドのパラメーターに必要とするクラスを指定することができ、Codeceptionは自動的にそれぞれのオブジェクトを作成し、すべての依存関係を引数としてそのメソッドを呼び出す、ということです。この仕組みはヘルパーを利用する際に便利です。例：

```php
<?php
class SignUpCest
{
    /**
     * @var Helper\SignUp
     */
    protected $signUp;

    /**
     * @var Helper\NavBarHelper
     */
    protected $navBar;

    protected function _inject(\Helper\SignUp $signUp, \Helper\NavBar $navBar)
    {
        $this->signUp = $signUp;
        $this->navBar = $navBar;
    }

    public function signUp(\AcceptanceTester $I)
    {
        $I->wantTo('sign up');

        $this->navBar->click('Sign up');
        $this->signUp->register([
            'first_name'            => 'Joe',
            'last_name'             => 'Jones',
            'email'                 => 'joe@jones.com',
            'password'              => '1234',
            'password_confirmation' => '1234'
        ]);
    }
}
```

テストクラスの例：

```php
<?php
class MathTest extends \Codeception\TestCase\Test
{
   /**
    * @var \UnitTester
    */
    protected $tester;

    /**
     * @var Helper\Math
     */
    protected $math;

    protected function _inject(\Helper\Math $math)
    {
        $this->math = $math;
    }

    public function testAll()
    {
        $this->assertEquals(3, $this->math->add(1, 2));
        $this->assertEquals(1, $this->math->subtract(3, 2));
    }
}
```

とはいえ、DIはこれに限定されるものではありません。Codeceptionが把握している引数とともに作成できる、**どのようなクラスでも注入** することができます。

この作業を自動化するためには、必要な引数とともに`_inject()`メソッドを実装する必要があります。Codeceptionがどのオブジェクトを渡せばよいか推測するために、引数の型を指定することが重要となります。`_inject()`メソッドはテストケースオブジェクト（CestまたはTest）の作成後に1度のみ呼ばれます。同じ方法でヘルパークラスとアクタークラスについてもDIは働きます。

Cest形式のそれぞれのテストメソッドは固有の依存関係を定義し、引数からそれらを受け取ることができます：

```php
<?php
class UserCest
{
    function updateUser(\Helper\User $u, \AcceptanceTester $I, \Page\User $userPage)
    {
        $user = $u->createDummyUser();
        $userPage->login($user->getName(), $user->getPassword());
        $userPage->updateProfile(['name' => 'Bill']);
        $I->see('Profile was saved');
        $I->see('Profile of Bill','h1');
    }
}
```

さらに、Codeceptionは、再帰的な依存関係（`A`が`B`に、そして`B`が`C`に依存する等）の解決や、デフォルト値を伴うプリミティブ型のパラメーター（`$param = 'default'`のような）を扱うことができます。もちろん、*依存関係のループ* は禁止です。

### Examples

もし、あるテストシナリオを異なるデータで実行したい場合 、どうすればよいでしょうか。そのようなケースでは、異なるデータを提供するためにExampleを利用することができ、`\Codeception\Example`インスタンスとして注入することができます。データはJSONまたはDoctrine形式（単一行に限ります）を使用した`@example`アノテーションによって定義されます。

```php
<?php
 /**
  * @example ["/api/", 200]
  * @example ["/api/protected", 401]
  * @example ["/api/not-found-url", 404]
  * @example ["/api/faulty", 500]
  */
  public function checkEndpoints(ApiTester $I, \Codeception\Example $example)
  {
    $I->sendGET($example[0]);
    $I->seeResponseCodeIs($example[1]);
  }
```

<div class="alert alert-notice">
JSONを使うときは、その標準仕様に従い、すべてのキーと値をダブルクォーテーションで囲う必要があることを忘れないでください。
</div>

Exampleをキーバリューデータとして渡すこともでき、同じようにテストで使用することができます。

```php
 /**
  * @example { "url": "/", "title": "Welcome" }
  * @example { "url": "/info", "title": "Info" }
  * @example { "url": "/about", "title": "About Us" }
  * @example { "url": "/contact", "title": "Contact Us" }
  */
  public function staticPages(AcceptanceTester $I, \Codeception\Example $example)
  {
    $I->amOnPage($example['url']);
    $I->see($example['title'], 'h1');
    $I->seeInTitle($example['title']);
  }
```

これらのExampleはDoctrine形式のアノテーション文法としても記述することができます：

```php
 /**
  * @example(url="/", title="Welcome")
  * @example(url="/info", title="Info")
  * @example(url="/about", title="About Us")
  * @example(url="/contact", title="Contact Us")
  */
  public function staticPages(AcceptanceTester $I, \Codeception\Example $example)
  {
    $I->amOnPage($example['url']);
    $I->see($example['title'], 'h1');
    $I->seeInTitle($example['title']);
  }
```

### Before/Afterアノテーション

`@before`と`@after`アノテーションを使ってテストの実行フローをコントロールすることができます。共通のアクションをprotectedな（テストでない）メソッドに移動し、アノテーションを記述することによってテストの前後でそれらを呼び出すことができます。`@before`や`@after`アノテーションを複数利用することによって、複数のメソッドを呼び出すことが可能です。メソッドは上から下の順で呼び出されます。

```php
<?php
class ModeratorCest {

    protected function login(AcceptanceTester $I)
    {
        $I->amOnPage('/login');
        $I->fillField('Username', 'miles');
        $I->fillField('Password', 'davis');
        $I->click('Login');
    }

    /**
     * @before login
     */
    public function banUser(AcceptanceTester $I)
    {
        $I->amOnPage('/users/charlie-parker');
        $I->see('Ban', '.button');
        $I->click('Ban');
    }

    /**
     * @before login
     * @before cleanup
     * @after logout
     * @after close
     */
    public function addUser(AcceptanceTester $I)
    {
        $I->amOnPage('/users/charlie-parker');
        $I->see('Ban', '.button');
        $I->click('Ban');
    }
}
```

`@before`と`@after`アノテーションはインクルードされた関数に対しても使えます。ただし、1つのメソッドに対して同じ種類のアノテーションを複数記述することはできません。1つのメソッドに対して、1つの`@before`と1つの`@after`アノテーションのみです。

## 環境

別の設定を使ってテストを実行する必要がある場合、別の設定環境を定義することができます。
最も典型的なユースケースとしては、種類の違うブラウザーで受け入れテストを実行する場合や、異なるデータベースエンジンを利用してテストを実行する場合です。

ブラウザーのケースについて、環境をどう使うか、実際にやってみましょう。

`acceptance.suite.yml`に新しく行を追加する必要があります。

``` yaml
class_name: AcceptanceTester
modules:
    enabled:
        - WebDriver
        - \Helper\Acceptance
    config:
        WebDriver:
            url: 'http://127.0.0.1:8000/'
            browser: 'firefox'

env:
    phantom:
         modules:
            config:
                WebDriver:
                    browser: 'phantomjs'

    chrome:
         modules:
            config:
                WebDriver:
                    browser: 'chrome'

    firefox:
        # nothing changed
```

別環境は基本的にルートの`env`配下に定義し、名前（`phantom`、`chrome`など）をつけ、そして前で設定された任意の設定パラメーターを再定義することができます。

`paths`設定の`envs`パラメーターで指定されたディレクトリーに配置された設定ファイルに環境を定義することもできます：

```yaml
paths:
    envs: tests/_envs
```

設定ファイルの名前は環境名として利用されます（たとえば、`chrome.yml`や`chrome.dist.yml`は`chrome`という環境名となります）。
環境設定ファイルは`generate:environment`コマンドによって生成することができます：

```bash
$ php codecept g:env chrome
```

そしてその中でオーバーライドしたいオプションを指定してください：

```yaml
    config:
        WebDriver:
            browser: 'chrome'
```

環境設定ファイルはスイート設定がマージされる前に、メインの設定ファイルにマージされます。

設定ファイルは`--env`オプションを指定して実行することで簡単に切り替えることができます。PhantomJSのみテストを実行したい場合は、`--env phantom`を指定します：

```bash
$ php codecept run acceptance --env phantom
```

3ブラウザーすべてのテストを実行するには、ただすべての環境を列挙します：

```bash
$ php codecept run acceptance --env phantom --env chrome --env firefox
```

テストはそれぞれのブラウザーごとに3回実行されるでしょう。

セパレーターとしてカンマを使うことで、複数の環境を1つにマージすることもできます：

```bash
$ php codecept run acceptance --env dev,phantom --env dev,chrome --env dev,firefox
```

設定は与えられた順番でマージされます。この方法で環境設定の複数の組み合わせを簡単に作ることができます。

環境に応じて、実行すべきテストを選択することができます。
たとえば、いくつかはFirefoxのみで、少しだけChromeのみで、テストを実行する必要があるかもしれません。

TestとCest形式のテストでは、`@env`アノテーションを用いて実行を求められる環境を指定することができます：

```php
<?php
class UserCest
{
    /**
     * このテストは'firefox'と'phantom'環境でのみ実行されます
     *
     * @env firefox
     * @env phantom
     */
    public function webkitOnlyTest(AcceptanceTester $I)
    {
        // I do something
    }
}
?>
```

Cept形式では単純なコメントを使ってください

```php
// @env firefox
// @env phantom
```

この方法によりそれぞれの環境でどのテストを実行するのか、容易にコントロールすることができます。

### 現在の値

時としてリアルタイムにテストの振る舞いを変更する必要があるかもしれません。たとえば、同じテストの振る舞いがFirefoxとChromeとで異なるかもしれません。
`$scenario->current()`メソッドを呼べば、現在の環境名、テスト名、有効化されているモジュールの一覧をリアルタイムに知ることができます。

```php
<?php
// 現在の環境を受け取る
$scenario->current('env');

// 有効化されたモジュールを一覧する
$scenario->current('modules');

// テスト名
$scenario->current('name');

// ブラウザー名 (WebDriverモジュールが有効な場合)
$scenario->current('browser');

// ケーパビリティ  (WebDriverモジュールが有効な場合)
$scenario->current('capabilities');

```

Cept形式とCest形式で、`\Codeception\Scenario`にアクセスすることができます。Cept形式では、デフォルトで`$scenario`変数が用意されており、一方のCest形式ではディペンデンシーインジェクションによって受け取ってください。

```php
<?php
public function myTest(\AcceptanceTester $I, \Codeception\Scenario $scenario)
{
    if ($scenario->current('browser') == 'phantomjs') {
      // emulate popups for PhantomJS
      $I->executeScript('window.alert = function(){return true;}');
    }
}
```
`Codeception\Scenario`はアクタークラスやStepObjectからも利用可能です。`$this->getScenario()`を使用してアクセスできます。


### 依存関係

`@depends`アノテーションを使って現在のテストが事前にどのテストにパスしているべきか、指定することができます。もし事前のテストが失敗したら、対象のテストはスキップされます。
依存しているテストのメソッド名をアノテーションに記述してください。

```php
<?php
class ModeratorCest {

    public function login(AcceptanceTester $I)
    {
        // logs moderator in
    }

    /**
     * @depends login
     */
    public function banUser(AcceptanceTester $I)
    {
        // bans user
    }
}
```

依存関係は`Cest`形式と`Codeception\Test\Unit`形式に適用されます。依存関係は、異なるクラス間で設定することができます。異なるファイルが依存するテストを指定するためには、*テストシグネチャ* を与える必要があります。一般的なテストシグネチャは`className:methodName`という形式になります。`--steps`オプションをつけてテストを実行することで正確なテストシグネチャを確認することができます：

```
Signature: ModeratorCest:login`
```

Codeceptionは、常に依存元のテストが依存先のテストよりも後に実行されるよう、テストを並び替えます。

## インタラクティブ・コンソール

インタラクティブ・コンソールはCodeceptionのコマンドをテストで実行する前に試すために追加されました。

![コンソール](http://img267.imageshack.us/img267/204/003nk.png)

次のコマンドでコンソールを起動することができます。

``` bash
$ php codecept console suitename
```

これで、アクタークラスに対応したすべてのコマンドを実行することができ、結果をすぐに確認できます。特に、`WebDriver`モジュールと使用する場合に便利です。Selenuimとブラウザーの起動にはいつも長い時間がかかります。ところが、コンソールを使って別のセレクター、異なるコマンドを試すことで、実行された際に確実にパスするテストを書くことができます。

特別なヒント：コンソールとSeleniumを使ってあなたがウェブページを上手に操作できるか上司に見せましょう。この手順を自動化し、プロジェクトに受け入れテストを導入することを説得しやすくなるでしょう。

## 異なるフォルダーからテストを実行する

もしCodeceptionを使ったプロジェクトが複数あったとしても、すべてのテストを1つの`codecept`で実行することができます。
異なるディレクトリーのCodeceptionを実行するために、`bootstrap`を除くすべてのコマンドに`-c`オプションが用意されています。

```bash
$ php codecept run -c ~/projects/ecommerce/
$ php codecept run -c ~/projects/drupal/
$ php codecept generate:cept acceptance CreateArticle -c ~/projects/drupal/
```

現在の場所と異なるディレクトリー内にプロジェクトを作成するには、パラメーターとしてパスを指定するだけです。

```bash
$ php codecept bootstrap ~/projects/drupal/
```

基本的に`-c`オプションにはパスを指定しますが、使用する設定ファイルを指定することもできます。従って、テストスイートに対して複数の`codeception.yml`を持つことができます。さまざまな環境や設定を指定するためにこのオプションを使用することができます。特定の設定ファイルを使ってテストを実行するためには、`-c`オプションにファイル名を渡してください。

## グループ

テストをグルーピングして実行する方法はいくつかあります。特定のディレクトリーを指定して実行することもできます。

```bash
$ php codecept run tests/acceptance/admin
```

また、特定の（もしくは複数の）グループを指定して実行することもできます。

```bash
$ php codecept run -g admin -g editor
```

このケースでは`admin`と`editor`グループに属すすべてのテストが実行されます。グループの概念はPHPUnitから取り入れたもので、従来のPHPUnitと同様の振る舞いをします。

Test形式とCest形式のテストでは、テストをグループに追加するために`@group`アノテーションを使います。

```php
<?php
/**
 * @group admin
 */
public function testAdminUser()
{
}
```

Cest形式では、コメントで疑似アノテーションを使用します：

```php
<?php
// @group admin
// @group editor
$I = new AcceptanceTester($scenario);
$I->wantToTest('admin area');
```

フィーチャーファイル（Gherkin）ではタグを使用します：

```gherkin
@admin @editor
Feature: Admin area
```

### グループファイル

グループはグローバルまたはテストスイートの設定ファイルで定義できます。
グループに含まれるテストは、配列形式で指定するか、テストの一覧が含まれるファイルパスを指定します。

```yaml
groups:
  # add 2 tests to db group
  db: [tests/unit/PersistTest.php, tests/unit/DataTest.php]

  # add list of tests to slow group
  slow: tests/_data/slow  
```

たとえば、最も遅いテストの一覧ファイルを作成し、それらのグループ内で実行することができます。
グループファイルは各行にテスト名が記述されたプレーンテキストファイルです。

```bash
tests/unit/DbTest.php
tests/unit/UserTest.php:create
tests/unit/UserTest.php:update
```

グループファイルは手動で作成することもできますし、サードパーティー製のアプリケーションによって生成することもできます。
たとえば、XMLレポートから遅いテストを抽出してグループファイルを更新するスクリプトを記述することができるでしょう。

1つの定義で複数のグループファイルをロードするような、パターンについても指定することもできます。

```yaml
groups:
  p*: tests/_data/p*
```

これは、`tests/_data`内の`p*`パターンに一致するすべてのファイルをグループとしてロードします。

## まとめ

Codeceptionは一見シンプルなフレームワークのように見えますが、強力なテストを、単一のAPIを使って、それらをリファクタリングし、そしてインタラクティブ・コンソールを使ってより速く、構築することができます。CodeceptionのテストはグループやCestクラスを使って容易に整理することができます。
