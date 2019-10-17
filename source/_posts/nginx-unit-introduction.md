---
title: nginx unit introduction
date: 2019-10-17 17:43:39
tags:
    - unit
---

前几天发现NGINX又搞了个[Unit](https://unit.nginx.org/)，简单来说是一个web应用容器，特点是可以通过RESTful风格的API进行动态配置（不需要NGINX那样该配置然后重启），并且支持Python、Go、PHP、Java等等一大堆语言。不知道是不是受了[Passenger](https://www.phusionpassenger.com/docs/tutorials/what_is_passenger/)的启发。

这里记录一下试用的笔记。

### 安装

ArchLinux可以通过[PKGBUILD](https://aur.archlinux.org/packages/nginx-unitd)安装，`makepkg`之后`sudo pacman -U *.xz`即可。

### 启动

执行`sudo systemctl start unit`即可。

### 配置

写一个config.json，内容如下：

```json
{
    "listeners": {
        "*:8000": {
            "pass": "applications/phalcon-test"
        }
    },

    "applications": {
        "phalcon-test": {
            "type": "php",
            "processes": 5,
            "user": "vagrant",
            "group": "vagrant",
            "script": "index.php",
            "root": "/home/vagrant/project/mvc/multiple-volt/public"
        }
    },

    "routes": [
        {
            "action": {
                "pass": "applications/phalcon-test"
            }
        }
    ]
}
```

表示启动一个PHP类型的应用，并让Unit监听8000端口，将所有请求抛给那个PHP应用。

<!-- more -->

这里使用Phalcon的[示例工程](https://github.com/phalcon/mvc )里面的`multiple-volt`作为应用示例。

执行：

```shell
curlie -X PUT --data-binary @config.json --unix-socket /run/nginx-unit.control.sock http://localhost/config/
```

让Unix应用这个配置。不过默认会提示无法连接，这是由于/run/nginx-unit.control.sock的所属用户是root，要么改下它的用户（sudo chown vagrant:vagrant /run/nginx-unit.control.sock），要么通过sudo进行访问。

不过一开始我发现Unix返回提示`Unknown parameter "pass"`，但是它的文档就是一模一样类似配置的啊，查了一圈，又翻了下源代码，发现是AUR上面的PKGBUILD里面指定的Unit的版本太低了，这个倒好说，改下PKGBUILD，升级到1.12.0，重新makepkg然后安装。

![unit config](/images/nginx-unit-config.png)

### 访问

```shell
http localhost:8000
```

![unit access](/images/nginx-unit-access.png)

尝试改下首页：

```diff
diff --git a/multiple-volt/apps/frontend/controllers/IndexController.php b/multiple-volt/apps/frontend/controllers/IndexController.php
index a8502dc..8580675 100644
--- a/multiple-volt/apps/frontend/controllers/IndexController.php
+++ b/multiple-volt/apps/frontend/controllers/IndexController.php
@@ -6,5 +6,6 @@ class IndexController extends ControllerBase
 {
     public function indexAction()
     {
+        $this->view->hello = "world <script>";
     }
 }
diff --git a/multiple-volt/apps/frontend/views/index/index.volt b/multiple-volt/apps/frontend/views/index/index.volt
index 461223d..af590f9 100644
--- a/multiple-volt/apps/frontend/views/index/index.volt
+++ b/multiple-volt/apps/frontend/views/index/index.volt
@@ -1,3 +1,5 @@
 <h1>Congratulations!</h1>

-<p>You're now flying with Phalcon. Great things are about to happen!</p>
\ No newline at end of file
+<p>You're now flying with Phalcon. Great things are about to happen!</p>
+
+<p>{{ hello | e }}</p>
```

再次访问就能看到效果。

如果增加新的controller和action呢？我们增加一个apps/frontend/controllers/AaaaController.php。内容如下：

```php
<?php

namespace Multiple\Frontend\Controllers;

class AaaaController extends ControllerBase
{
    public function bbbbAction()
    {
        $this->view->disable();
        var_dump("1234");
    }
}
```

访问`http://localhost:8000/aaaa/bbbb`，会发现我们通过config.json配置里的script把所有请求都指向了index.php，但此时代码里的路由机制并没有识别到我们的访问路径`/aaaa/bbbb`。实际上Phalcon默认需要一个_url的参数，这在Nginx+PHP-FPM的方式下，很容易通过rewrite解决，可以参考[https://docs.phalcon.io/4.0/en/webserver-setup#phalcon-configuration](https://docs.phalcon.io/4.0/en/webserver-setup#phalcon-configuration)。然而Unit目前还不能做那样的rewrite，我们可以用过更改Phalcon里面router的行为，在config/services.php增加一行代码来解决：

```diff
--- a/multiple-volt/config/services.php
+++ b/multiple-volt/config/services.php
@@ -25,6 +25,8 @@ $di["router"] = function () {

     $router->setDefaultNamespace("Multiple\Frontend\Controllers");

+    $router->setUriSource($router::URI_SOURCE_SERVER_REQUEST_URI);
+
     return $router;
 };
```

参考自[这里](https://forum.phalcon.io/discussion/9239/problem-with-get-on-nginx config/services.php) ，目的是让Phalcon通过REQUEST_URI来识别路由，而不是\_url。

这样就可以了：

![unit access custom](/images/nginx-unit-access-custom.png)