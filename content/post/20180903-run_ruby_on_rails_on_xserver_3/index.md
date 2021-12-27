---
title: XSERVERでRuby on Railsを動かすその3(完結編)
description: XSERVERでRuby on Railsを動かしてみた記録第三話です。すべての工程を最終話にまとめていますので、この記事を見る必要はありません。
date: 2018-09-03
slug: run_ruby_on_rails_on_xserver_3
tags:
  - "Ruby"
  - "Web service"
  - "XSERVER"
series:
  - "run ruby on rails on xserver"
---
> この記事は試行錯誤の記録です！railsを動かすには、[その4（結論編）](https://blog.pinfort.me/posts/run_ruby_on_rails_on_xserver_4)だけを見てもらえれば解決します！

前回まででrails sが動いたので見えるようにします。

ポートふさがれてるのでそのまま外からは見えません。Apacheやnginxのvirtual host設定もできません。ですが、PHPは置くところに置けば見えるので、PHPをプロキシのために使います。

> [https://qiita.com/tadsan/items/f76e3c62af82c98efcfc](https://qiita.com/tadsan/items/f76e3c62af82c98efcfc)

ほぼこぴぺ。

```
$cat index.php
<?php

namespace zonuexe\ZoProxy;

require_once(dirname(__FILE__).'/../../../php/vendor/autoload.php');

use GuzzleHttp\Client as HttpClient;
use GuzzleHttp\Psr7;

// require __DIR__ . '/../vendor/autoload.php';
// ここはいい感じにやってね
$host_table = [
    'localhost' => [
        'host' => 'localhost',
        'port' => railsのぽーと,
        'scheme' => 'http',
    ],
    'おもてのドメイン名' => [
        'host' => 'localhost',
        'port' => railsのぽーと,
        'scheme' => 'http',
    ]
];

$request = Psr7\ServerRequest::fromGlobals();
$new_uri = $request->getUri()->withPort(80);

$key = $new_uri->getHost();
$port = $new_uri->getPort();

//if ($port !== null && !Psr7\Uri::isDefaultPort($new_uri)) {
//    $key .= ":{$port}";
//}
// コメントアウトしないとSSLがうまくいかなかった。

if (isset($host_table[$key])) {
    if (isset($host_table[$key]['host'])) {
        $new_uri = $new_uri->withHost($host_table[$key]['host']);
    }
    if (isset($host_table[$key]['port'])) {
        $new_uri = $new_uri->withPort($host_table[$key]['port']);
    }
    if (isset($host_table[$key]['scheme'])) {
        $new_uri = $new_uri->withScheme($host_table[$key]['scheme']);
    }
}

$client = new HttpClient;
$response = $client->send($request->withUri($new_uri), [
    'http_errors' => false,
]);

foreach ($response->getHeaders() as $key => $values) {
    if($key !== 'Transfer-Encoding'){
        // Transfer-Encodingを除外しないと内部リダイレクトループになった。なんでだろう？
        foreach ($values as $value) {
            header("{$key}:{$value}");
        }
    }
}

echo $response->getBody();
```

```
$cat .htaccess
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On

    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://どめいん/$1 [R=302,L]

    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Handle Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>
```

できた。
