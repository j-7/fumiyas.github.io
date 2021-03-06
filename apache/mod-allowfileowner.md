---
title: "Apache HTTPD: mod_allowfileowner"
tags: [apache, security]
layout: default
---

## mod_allowfileowner って何?

Apache HTTPD 用のフィルターモジュールです。
静的コンテンツファイルの所有者をチェックして、
指定のユーザーが所有していればアクセスを許可し、
そうでなければ拒否します。

静的コンテンツファイルをオープンした後、それを指すファイル記述子に対して
`fstat`(2) を行ないファイル所有者をチェックする実装になっているため、
TOCTOU (Time Of Check to Time Of Use) 問題はありません。

## 何が嬉しいの?

共有 Web サーバーサービスにおいて、一部のシンボリックリンク攻撃を防ぐことができます。
この攻撃は `Options -FollowSymLinks` や `Options SymLinksIfOwnerMatch`
設定では完全には防ぐことはできません。

参考:

  * Apache HTTPD: `Options -FollowSymLinks` は不完全
      * {{ site.url }}/2013/09/03/no-followsymlinks-is-not-safe.html

攻撃の例:

```console
$ ln -s ~target/public_html/wordpress/wp-config.php ~/public_html/wp-config.txt
$ wget -q -O - http://www.example.jp/~attacker/wp-config.txt |grep DB_
define('DB_NAME', 'wordpress');
define('DB_USER', 'user');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');
```

## 使用例

### モジュールのビルドとインストール

ソースコードは GitHub で公開しています。

  * <https://github.com/fumiyas/apache-mod-allowfileowner>

モジュールのビルドには Apache HTTPD の開発環境が必要です。
Debian であれば `apache2-devel` (バージョン 2.4 以降の場合)、
`apache2-prefork-dev` (バージョン 2.2 以前で prefork MPM の場合)、
もしくは `apache2-threaded-dev` (バージョン 2.2 以前で worker や event MPM の場合)、
RHEL であれば `httpd-devel` パッケージ。

```console
$ git clone https://github.com/fumiyas/apache-mod-allowfileowner.git
$ cd apache-mod-allowfileowner
$ make
...
$ sudo make install
...
```

### 設定例

```conf
## モジュールのロード
## 先の `sudo make install` で自動的に足されているはず
LoadModule allowfileowner_module modules/mod_allowfileowner.so

<VirtualHost *:80>
  ## ...

  ## 既定の出力フィルターに設定
  SetOutputFilter ALLOWFILEOWNER

  ## 既存の出力フィルターに追加
  AddOutputFilter ALLOWFILEOWNER;INCLUDES .shtml

  DocumentRoot /var/www
  <Directory /var/www>
    ## 静的コンテンツファイルへのアクセスは所有者が指定のユーザーと
    ## グループの場合だけ許可する (注意: CGI, PHP などは対象外)
    AllowFileOwner webadmin apache
    AllowFileOwnerGroup webusers
    ## ...
  </Directory>

  UserDir public_html
  <Directory /home/*/public_html>
    ## 静的コンテンツファイルへのアクセスは所有者がホームディレクトリの
    ## 所有ユーザーの場合だけ許可する (注意: CGI, PHP などは対象外)
    AllowFileOwnerInUserDir On
    ## ...
  </Directory>

  ## ...
</VirtualHost>
```

## 制限

出力フィルター対象の大本のソースがファイルでない場合
([`default-handler`](http://httpd.apache.org/docs/2.2/handler.html#definition)
が利用されない場合) や、
出力フィルターが利用されないファイルアクセス手段が利用された場合は、
当モジュールによる制限は効きません。

具体的には下記が該当します。

  * CGI や PHP などのサーバーサイドスクリプト
      * suEXEC などを利用して制限ユーザー権限でスクリプトを実行し回避すること。
  * <del>SSI (mod_include) の `<!--#include file="..." -->`</del>
      * SSI によるインクルードも制限されることを確認。
      * <del>調査、試験をしていないが、恐らく駄目。</del>
      * <del>`<!--#include virtual="..." -->` は問題ないと思われる。(未確認)</del>
      * <del>SSI を無効にする、もしくは mod_include を改造して</del>
        <del>`<!--#include file="..." -->` を無効化するなどして対処すること。</del>
  * そのほか?

## 参照

  * 人間とウェブの未来 - 共有WebホスティングでVitrualHost設定に手を入れずにシンボリックリンクのTOCTOU問題を解決するApacheモジュールを作った
      * <http://blog.matsumoto-r.jp/?p=4405>
      * <https://github.com/matsumoto-r/mod_fileownercheck>
