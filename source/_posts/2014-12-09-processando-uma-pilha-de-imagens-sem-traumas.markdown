---
layout: post
title: "Processando Uma Pilha De Imagens Sem Traumas"
date: 2014-04-04 00:09:59 -0200
comments: true
categories: ["PHP", "GD", "IO", "Imagemagic"]
---

![imagem](/../images/destaque_reduz_image.png)

Quem nunca precisou processar um número consideravel de imagens para um relatório, não tem idéia da dor de cabeça que isso pode dar. Em um mundo ideal, todo thumb ou imagem tratada, é gerada no momento do upload, porém por diversos fatores, podemos nos deparar com a necessidade de fazer isso sob demanda, e como todos sabemos, isso pode implicar em uma série de problemas, como demora ao gerar essas imagens, e em situações mais graves, podemos ter um estouro de memória, lentidão no servidor, timeout na execução, em fim, tudo que você não quer fazer.

Pensando nisso, vamos desenvolver uma solução que ataca os principais gargalos desse tipo de tarefa, que são a ausência de **threds** em **PHP** e o processo de **IO**.



##Solução
Iremos desenvolver um serviço capaz de de criar as imagens que ainda não foram criadas, e melhor, paralelamente, e usando um recurso que não usa diretamente a thread do **php**

###Checando os arquivos a serem gerados
Vamos criar um processo capaz de criar thumbs de imagens que ainda não foram criadas, de modo que, antes de começarmos a processar nossa pilha de imagens saberemos o que devemos ou não criar, além disso, ao invés de checar a existência dessas imagens em um loop de file_exists, vamos primeiro listar todos os arquivos da pasta destino e guardar em um array (uma única requisição ao invés de varias requisições em um loop de file_exists), depois disso vamos fazer um diff_contains dos dos arrays de images a serem processadas com as imagens do servidor, ai o array resultante, é o array de imagens que precisamos criar (que não estão no servidor).

```php
// Origem
$origin = scandir('image/origin');
$origin = array_slice($origin,2,count($origin));

//destino
$destination = scandir('image/destination');
$destination = array_slice($destination,2,count($destination));

//Imagens a serem geradas
$terms_list = array_diff($origin, $destination);
```

###Criando os Thumbs
Já resolvemos o primeiro problema, que é checar quais imagens devemos criar os thumbs, agora temos dois problemas, na verdade um, só que dividido em duas partes, que é o processo de gerar e salvar as imagens. E como todos nós sabemos, todo processo de IO é custoso, isso sem contar com o custo de processamento que o PHP tem ao tratar essas imagens antes de salva-la. Como não temos threds no PHP, temos que gerar uma fila de execução, o que gera um grande buzy-waiting, de modo que pode até causar um timeout do servidor. Para resolver esse problema, precisamos dividir esse processo em duas partes, a primeira é processar as imagens em paralelo (usando o cURL mult_exec), depois usar a solução mais eficaz para processar as imagens que é usar o **Imagemagick** ao invés do GD.

> **Por que Imagemagick?**

> O Imagemagick ao contrario do GD executa seus processos fora do PHP, ou seja, ele não vai consumir o processo do PHP para fazer sua mágica, além disso, se compararmos a qualidade da imagem gerada entre um e outro, o Imagemagick consegue ter um nível bem superior ao GD.

Vamos criar uma classe capaz de usar ambas as bibliotecas, pois, desse modo, independente de haver ou não o Imagemagick no servidor, essas imagens serão geradas.

#####Interface Image
Vamos primeiramente criar um contrato, para que todas as classes de imagens apliquem os métodos usando o mesmo critério, embora de modo diferente.

***Image.php***:
```php
/**
 * Interface Image
 */
interface Image{

    /**
     * @param $file
     * @param $anchor
     */
    function open($file, $anchor);

    /**
     * @param $width
     * @param $height
     */
    function resize($width, $height);

    /**
     * @param $file
     * @param $quality
     */
    function save($file, $quality);
```

#####Classe imageGenerateGD
Feito Isso, vamos criar nossa Classe que usa o GD para criar nossos thumbs.

***imageGenerateGD.php***:
```php
include_once "Image.php";

/**
 * Class imageGenerateGD
 */
class imageGenerateGD implements  Image{
    private $image;

    public function __construct(){
        $options = func_get_args();
        if(count($options)){
            $this->open($options[0]);
        }

    }

    /**
     * @param $file
     * @param string $anchor
     * @return $this
     */
    function open($file, $anchor = ''){
        $imagePath = $anchor.$file;
        switch (pathinfo($imagePath)) {
            case 'gif':
                $this->image = imagecreatefromgif($imagePath);
                break;
            case 'png':
                $this->image = imagecreatefrompng($imagePath);
                break;
            case 'jpg':
            case 'jpeg':
            case 'pjpeg':
            default:
                $this->image = imagecreatefromjpeg($imagePath);
                break;
        }
        imagealphablending($this->image, true);
        imagesavealpha($this->image, true);
        imagecolorallocate($this->image, 0, 0, 0);

        return $this;
    }

    /**
     * @param $width
     * @param $height
     * @return $this
     */
    function resize($width = 160, $height = 160){

        // get the current image dimensions
        $geo = array(
            'width'     =>  imagesx($this->image),
            'height'    =>  imagesy($this->image)
        );

        $original_aspect = $geo['width'] / $geo['height'];
        $thumb_aspect = $width / $height;

        if ( $original_aspect >= $thumb_aspect )
        {
            // If image is wider than thumbnail (in aspect ratio sense)
            $new_height = $height;
            $new_width = $geo['width'] / ($geo['height'] / $height);
        }
        else
        {
            // If the thumbnail is wider than the image
            $new_width = $width;
            $new_height = $geo['height'] / ($geo['width'] / $width);
        }

        $thumb = imagecreatetruecolor( $width, $height );

        // Resize and crop
        imagecopyresampled($thumb,
            $this->image,
            0 - ($new_width - $width) / 2, // Center the image horizontally
            0 - ($new_height - $height) / 2, // Center the image vertically
            0, 0,
            $new_width, $new_height,
            $geo['width'], $geo['height']);
        $this->image = $thumb;
        return $this;
    }

    /**
     * @param $file
     * @param int $quality
     * @return $this
     */
    function save($file, $quality = 90){
        imagejpeg($this->image,$file, intval($quality));
        imagedestroy($this->image);
        return $this;
    }
}
```

#####Classe imageGenerateImagick
Agora vamos criar nossa classe que usa o Imagemagick para criar os thumbs.

***imageGenerateImagick.php***:
```php
include_once "Image.php";

/**
 * Class imageGenerateImagick
 */
class imageGenerateImagick implements Image{
    private $image;
    
    public function __construct(){
        $options = func_get_args();
        if(count($options)){
            $this->image = new Imagick($options[0]);
        } else {
            $this->image = new Imagick();
        }

    }

    /**
     * @param $file
     * @param $anchor
     * @return imageGenerateImagick
     */
    function open($file, $anchor = ''){
        $this->image->readImage($file);
        return $this;
    }

    /**
     * @param int $width
     * @param int $height
     * @return imageGenerateImagick
     */
    function resize($width = 160, $height = 90){

        $geo = $this->image->getImageGeometry();

        // crop the image
        if(($geo['width']/$width) < ($geo['height']/$height)){
            $this->image->cropImage($geo['width'], floor($height*$geo['width']/$width), 0, (($geo['height']-($height*$geo['width']/$width))/2));
        } else {
            $this->image->cropImage(ceil($width*$geo['height']/$height), $geo['height'], (($geo['width']-($width*$geo['height']/$height))/2), 0);
        }
        // thumbnail the image
        $this->image->ThumbnailImage($width,$height,true);
        return $this;

    }

    /**
     * @param $file
     * @param int $quality
     * @return imageGenerateImagick
     */
    function save($file, $quality = 80){
        $this->image->setImageCompression(Imagick::COMPRESSION_JPEG);
        $this->image->setImageCompressionQuality(intval($quality));
        $this->image->writeImage( $file );
        $this->image->clear();
        $this->image->destroy();
        return $this;
    }
}
```

#####Classe ImageGenerate

Vamos criar nossa classe que implementa ambas as classes, de GD e imagick, no caso, você irá definir qual recurso ele vai usar, e além disso, ele vai checar se tem e extensão do imagick, de modo que se não houver, o sistema irá utilizar o GD.

***ImageGenerate.php***:
```php
include_once "Image.php";
include_once "ImageGenerateImagick.php";
include_once "imageGenerateGD.php";
class ImageGenerate implements Image {

    private $image;
    private $mode;
    const MODE_IMAGICK = 1;
    const MODE_GD = 2;
    public function __construct(){
        $options = func_get_args();

        if(!isset($options[1])){
            $options[1] = self::MODE_IMAGICK;
        }

        if($options[1] == self::MODE_IMAGICK && extension_loaded('imagick')){
            if(count($options)){
                $this->image = new imageGenerateImagick($options[0]);
            } else {
                $this->image = new imageGenerateImagick();
            }
        } else {
            if(count($options)){
                $this->image = new imageGenerateGD($options[0]);
            } else {
                $this->image = new imageGenerateGD();
            }
        }

    }

    /**
     * @param mixed $mode
     */
    public function setMode($mode)
    {
        $this->mode = $mode;
    }

    /**
     * @return mixed
     */
    public function getMode()
    {
        return $this->mode;
    }

    /**
     * @param $file
     * @param string $anchor
     * @return imageGenerateGD/imageGenerateImagick
     */
    function open($file, $anchor = ''){
        $this->image->open($file, $anchor);
        return $this->image;
    }

    /**
     * @param int $width
     * @param int $height
     * @return imageGenerateGD/imageGenerateImagick
     */
    function resize($width = 160, $height = 160){
        $this->image->resize(160,160);
        return $this->image;

    }

    /**
     * @param $file
     * @param int $quality
     * @return imageGenerateGD/imageGenerateImagick
     */
    function save($file, $quality = 80){
        $this->image->save($file,$quality);
        return $this->image;
    }
}
```

###Criando o serviço de crop
Agora que já temos nossas classes de imagem, vamos criar um serviço ou melhor, uma API, que ao receber como parâmetro o nome e caminho da imagem de origem, ela se encarrega de gerar o thumb usando nossa classe.

***resize.php***:
```php
header('Content-Type: application/json');
if($_REQUEST['file']){

    include_once "../class/ImageGenerate.php";
    if(file_exists($_REQUEST['file'])){
        $teste = new imageGenerate($_REQUEST['file']);
        $width = (isset($_REQUEST['width']))? $_REQUEST['width'] : 160;
        $height = (isset($_REQUEST['height']))? $_REQUEST['height'] : 160;
        $teste->resize($width,$height)
              ->save('../image/destination/'.end(explode("/", $_REQUEST['file'])));
        echo json_encode(array(
            'status'    =>  'OK',
            'msg'       =>   utf8_encode('Image save on '.'image/destination/'.end(explode("/", $_REQUEST['file'])))
        ), true);

    } else {
        echo json_encode(array(
            'status'    =>  'ERROR',
            'msg'       =>  utf8_encode('picture does not exist')
        ), true);
    }


} else {
    echo json_encode(array(
        'status'    =>  'ERROR',
        'msg'       =>  utf8_encode('not sent parameter')
    ), true);
}
```

###Processando nossas imagens em paralelo
Agora que criamos nossas classes de imagens, e a API que gera as imagens, vamos criar uma página que utiliza um recurso do **cURL**, que é o **curl_multi_exec**, ele permite múltiplas requisições simultâneas a nossa API. No nosso caso, utilizamos uma Classe chamada **[RollingCurl](https://github.com/takinbo/rolling-curl)**, ela permite fazer essas chamadas paralelas, permitindo fazer os processos em lotes definidos, por exemplo, executar lotes de 5 em 5 imagens, após completar um lote, ele chama outro, até o fim do processamento. Fizemos apenas uma alteração, que foi criar um método que armazena todas as respostas em um array.

> **Cuidado**
> Requisições paralelas são recursos que devem ser usados com consciência, pois você irá forçar o seu servidor a processar **N** vezes em paralelo um determinado recurso, e se considerarmos que podemos ter **X** pessoas usando esse serviço ao mesmo tempo, podemos até derrubar nosso servidor (Não queremos fazer um auto ataque **DDos**).

***RollingCurl.php***:
```php
/*
Authored by Josh Fraser (www.joshfraser.com)
Released under Apache License 2.0

Maintained by Alexander Makarov, http://rmcreative.ru/

$Id$
*/

/**
 * Class that represent a single curl request
 */
class RollingCurlRequest {
  public $url = false;
  public $method = 'GET';
  public $post_data = null;
  public $headers = null;
  public $options = null;

    /**
     * @param string $url
     * @param string $method
     * @param  $post_data
     * @param  $headers
     * @param  $options
     * @return void
     */
    function __construct($url, $method = "GET", $post_data = null, $headers = null, $options = null) {
        $this->url = $url;
        $this->method = $method;
        $this->post_data = $post_data;
        $this->headers = $headers;
        $this->options = $options;
    }

    /**
     * @return void
     */
    public function __destruct() {
        unset($this->url, $this->method, $this->post_data, $this->headers, $this->options);
    }
}

/**
 * RollingCurl custom exception
 */
class RollingCurlException extends Exception {}

/**
 * Class that holds a rolling queue of curl requests.
 *
 * @throws RollingCurlException
 */
class RollingCurl {
    /**
     * @var int
     *
     * Window size is the max number of simultaneous connections allowed.
  * 
     * REMEMBER TO RESPECT THE SERVERS:
     * Sending too many requests at one time can easily be perceived
     * as a DOS attack. Increase this window_size if you are making requests
     * to multiple servers or have permission from the receving server admins.
     */
    private $window_size = 5;

    /**
     * @var float
     *
     * Timeout is the timeout used for curl_multi_select.
     */
    private $timeout = 10;

    /**
     * @var string|array
     *
     * Callback function to be applied to each result.
     */
    private $callback;

    /**
     * @var array
     *
     * Set your base options that you want to be used with EVERY request.
     */
    protected $options = array(
      CURLOPT_SSL_VERIFYPEER => 0,
        CURLOPT_RETURNTRANSFER => 1,
        CURLOPT_CONNECTTIMEOUT => 30,
        CURLOPT_TIMEOUT => 30
  );
  
    /**
     * @var array
     */
    private $headers = array();

    /**
     * @var Request[]
     *
     * The request queue
     */
    private $requests = array();

    /**
     * @var RequestMap[]
     *
     * Maps handles to request indexes
     */
    private $requestMap = array();

    /**
     * @var returns[]
     *
     * All returns of requests
     */
    private $returns = array();

    /**
     * @param  $callback
     * Callback function to be applied to each result.
     *
     * Can be specified as 'my_callback_function'
     * or array($object, 'my_callback_method').
     *
     * Function should take three parameters: $response, $info, $request.
     * $response is response body, $info is additional curl info.
     * $request is the original request
     *
     * @return void
     */
  function __construct($callback = null) {
        $this->callback = $callback;
    }

    /**
     * @param string $name
     * @return mixed
     */
    public function __get($name) {
        return (isset($this->{$name})) ? $this->{$name} : null;
    }

    /**
     * @param string $name
     * @param mixed $value
     * @return bool
     */
    public function __set($name, $value){
        // append the base options & headers
        if ($name == "options" || $name == "headers") {
            $this->{$name} = $value + $this->{$name};
        } else {
            $this->{$name} = $value;
        }
        return true;
    }

    /**
     * Add a request to the request queue
     *
     * @param Request $request
     * @return bool
     */
    public function add($request) {
         $this->requests[] = $request;
         return true;
    }

    /**
     * @param \returns[] $returns
     */
    public function setReturns($returns)
    {
        $this->returns = $returns;
    }

    /**
     * @return \returns[]
     */
    public function getReturns()
    {
        return $this->returns;
    }

    /**
     * Create new Request and add it to the request queue
     *
     * @param string $url
     * @param string $method
     * @param  $post_data
     * @param  $headers
     * @param  $options
     * @return bool
     */
    public function request($url, $method = "GET", $post_data = null, $headers = null, $options = null) {
         $this->requests[] = new RollingCurlRequest($url, $method, $post_data, $headers, $options);
         return true;
    }

    /**
     * Perform GET request
     *
     * @param string $url
     * @param  $headers
     * @param  $options
     * @return bool
     */
    public function get($url, $headers = null, $options = null) {
        return $this->request($url, "GET", null, $headers, $options);
    }

    /**
     * Perform POST request
     *
     * @param string $url
     * @param  $post_data
     * @param  $headers
     * @param  $options
     * @return bool
     */
    public function post($url, $post_data = null, $headers = null, $options = null) {
        return $this->request($url, "POST", $post_data, $headers, $options);
    }

    /**
     * Execute the curl
     *
     * @param int $window_size Max number of simultaneous connections
     * @return string|bool
     */
    public function execute($window_size = null) {
        // rolling curl window must always be greater than 1
        if (sizeof($this->requests) == 1) {
            return $this->single_curl();
        } else {
            // start the rolling curl. window_size is the max number of simultaneous connections
            return $this->rolling_curl($window_size);
        }
    }

    /**
     * Performs a single curl request
     *
     * @access private
     * @return string
     */
    private function single_curl() {
        $ch = curl_init();      
        $request = array_shift($this->requests);
        $options = $this->get_options($request);
        curl_setopt_array($ch,$options);
        $output = curl_exec($ch);
        $info = curl_getinfo($ch);

        // it's not neccesary to set a callback for one-off requests
        if ($this->callback) {
            $callback = $this->callback;
            if (is_callable($this->callback)){
                call_user_func($callback, $output, $info, $request);
            }
        }
      else
            return $output;
  return true;
    }

    /**
     * Performs multiple curl requests
     *
     * @access private
     * @throws RollingCurlException
     * @param int $window_size Max number of simultaneous connections
     * @return bool
     */
    private function rolling_curl($window_size = null) {
        if ($window_size)
            $this->window_size = $window_size;

        // make sure the rolling window isn't greater than the # of urls
        if (sizeof($this->requests) < $this->window_size)
            $this->window_size = sizeof($this->requests);

        if ($this->window_size < 2) {
            throw new RollingCurlException("Window size must be greater than 1");
        }

        $master = curl_multi_init();

        // start the first batch of requests
        for ($i = 0; $i < $this->window_size; $i++) {
            $ch = curl_init();

            $options = $this->get_options($this->requests[$i]);

            curl_setopt_array($ch,$options);
            curl_multi_add_handle($master, $ch);

            // Add to our request Maps
            $key = (string) $ch;
            $this->requestMap[$key] = $i;
        }

        do {
            while(($execrun = curl_multi_exec($master, $running)) == CURLM_CALL_MULTI_PERFORM);
            if($execrun != CURLM_OK) {
                break;
            }
            // a request was just completed -- find out which one
            while($done = curl_multi_info_read($master)) {

                // get the info and content returned on the request
                $info = curl_getinfo($done['handle']);
                $output = curl_multi_getcontent($done['handle']);

                array_push($this->returns, array(
                    'return'    =>  $output,
                    'info'      =>  $info,
                ));

                // send the return values to the callback function.
                $callback = $this->callback;
                if (is_callable($callback)){
              $key = (string)$done['handle'];
                    $request = $this->requests[$this->requestMap[$key]];
                    unset($this->requestMap[$key]);
                    call_user_func($callback, $output, $info, $request);
                }

                // start a new request (it's important to do this before removing the old one)
                if ($i < sizeof($this->requests) && isset($this->requests[$i]) && $i < count($this->requests)) {
                    $ch = curl_init();
                    $options = $this->get_options($this->requests[$i]);
                    curl_setopt_array($ch,$options);
                    curl_multi_add_handle($master, $ch);

                    // Add to our request Maps
                    $key = (string) $ch;
                    $this->requestMap[$key] = $i;
                    $i++;
                }

                // remove the curl handle that just completed
                curl_multi_remove_handle($master, $done['handle']);

            }

            // Block for data in / output; error handling is done by curl_multi_exec
            if ($running) {
                curl_multi_select($master, $this->timeout);
            }

        } while ($running);
        curl_multi_close($master);
        return true;
    }


    /**
     * Helper function to set up a new request by setting the appropriate options
     *
     * @access private
     * @param Request $request
     * @return array
     */
    private function get_options($request) {
        // options for this entire curl object
        $options = $this->__get('options');
        // NOTE: The PHP cURL library won't follow redirects if either safe_mode is on
        // or open_basedir is defined.
        // See: https://bugs.php.net/bug.php?id=30609
      if (( ini_get('safe_mode') == 'Off' || !ini_get('safe_mode') )
            && ini_get('open_basedir') == '') {
            $options[CURLOPT_FOLLOWLOCATION] = 1;
          $options[CURLOPT_MAXREDIRS] = 5;
        }
        $headers = $this->__get('headers');

      // append custom options for this specific request
      if ($request->options) {
            $options = $request->options + $options;
        }

      // set the request URL
        $options[CURLOPT_URL] = $request->url;

        // posting data w/ this request?
        if ($request->post_data) {
            $options[CURLOPT_POST] = 1;
            $options[CURLOPT_POSTFIELDS] = $request->post_data;
        }
        if ($headers) {
            $options[CURLOPT_HEADER] = 0;
            $options[CURLOPT_HTTPHEADER] = $headers;
        }

        // Due to a bug in cURL CURLOPT_WRITEFUNCTION must be defined as the last option
        // Otherwise it doesn't register. So let's unset and set it again
        // See http://stackoverflow.com/questions/15937055/curl-writefunction-not-being-called
        if( ! empty( $options[CURLOPT_WRITEFUNCTION]) ) {
            $writeCallback = $options[CURLOPT_WRITEFUNCTION];
            unset( $options[CURLOPT_WRITEFUNCTION] );
            $options[CURLOPT_WRITEFUNCTION] = $writeCallback;
        }

        return $options;
    }

    /**
     * @return void
     */
    public function __destruct() {
        unset($this->window_size, $this->callback, $this->options, $this->headers, $this->requests);
  }
}
```

***RollingCurl.php***:
```php
/*
Authored by Josh Fraser (www.joshfraser.com)
Released under Apache License 2.0

Maintained by Alexander Makarov, http://rmcreative.ru/

$Id$
*/

/**
 * Class that represent a single curl request
 */
class RollingCurlRequest {
  public $url = false;
  public $method = 'GET';
  public $post_data = null;
  public $headers = null;
  public $options = null;

    /**
     * @param string $url
     * @param string $method
     * @param  $post_data
     * @param  $headers
     * @param  $options
     * @return void
     */
    function __construct($url, $method = "GET", $post_data = null, $headers = null, $options = null) {
        $this->url = $url;
        $this->method = $method;
        $this->post_data = $post_data;
        $this->headers = $headers;
        $this->options = $options;
    }

    /**
     * @return void
     */
    public function __destruct() {
        unset($this->url, $this->method, $this->post_data, $this->headers, $this->options);
    }
}

/**
 * RollingCurl custom exception
 */
class RollingCurlException extends Exception {}

/**
 * Class that holds a rolling queue of curl requests.
 *
 * @throws RollingCurlException
 */
class RollingCurl {
    /**
     * @var int
     *
     * Window size is the max number of simultaneous connections allowed.
  * 
     * REMEMBER TO RESPECT THE SERVERS:
     * Sending too many requests at one time can easily be perceived
     * as a DOS attack. Increase this window_size if you are making requests
     * to multiple servers or have permission from the receving server admins.
     */
    private $window_size = 5;

    /**
     * @var float
     *
     * Timeout is the timeout used for curl_multi_select.
     */
    private $timeout = 10;

    /**
     * @var string|array
     *
     * Callback function to be applied to each result.
     */
    private $callback;

    /**
     * @var array
     *
     * Set your base options that you want to be used with EVERY request.
     */
    protected $options = array(
      CURLOPT_SSL_VERIFYPEER => 0,
        CURLOPT_RETURNTRANSFER => 1,
        CURLOPT_CONNECTTIMEOUT => 30,
        CURLOPT_TIMEOUT => 30
  );
  
    /**
     * @var array
     */
    private $headers = array();

    /**
     * @var Request[]
     *
     * The request queue
     */
    private $requests = array();

    /**
     * @var RequestMap[]
     *
     * Maps handles to request indexes
     */
    private $requestMap = array();

    /**
     * @var returns[]
     *
     * All returns of requests
     */
    private $returns = array();

    /**
     * @param  $callback
     * Callback function to be applied to each result.
     *
     * Can be specified as 'my_callback_function'
     * or array($object, 'my_callback_method').
     *
     * Function should take three parameters: $response, $info, $request.
     * $response is response body, $info is additional curl info.
     * $request is the original request
     *
     * @return void
     */
  function __construct($callback = null) {
        $this->callback = $callback;
    }

    /**
     * @param string $name
     * @return mixed
     */
    public function __get($name) {
        return (isset($this->{$name})) ? $this->{$name} : null;
    }

    /**
     * @param string $name
     * @param mixed $value
     * @return bool
     */
    public function __set($name, $value){
        // append the base options & headers
        if ($name == "options" || $name == "headers") {
            $this->{$name} = $value + $this->{$name};
        } else {
            $this->{$name} = $value;
        }
        return true;
    }

    /**
     * Add a request to the request queue
     *
     * @param Request $request
     * @return bool
     */
    public function add($request) {
         $this->requests[] = $request;
         return true;
    }

    /**
     * @param \returns[] $returns
     */
    public function setReturns($returns)
    {
        $this->returns = $returns;
    }

    /**
     * @return \returns[]
     */
    public function getReturns()
    {
        return $this->returns;
    }

    /**
     * Create new Request and add it to the request queue
     *
     * @param string $url
     * @param string $method
     * @param  $post_data
     * @param  $headers
     * @param  $options
     * @return bool
     */
    public function request($url, $method = "GET", $post_data = null, $headers = null, $options = null) {
         $this->requests[] = new RollingCurlRequest($url, $method, $post_data, $headers, $options);
         return true;
    }

    /**
     * Perform GET request
     *
     * @param string $url
     * @param  $headers
     * @param  $options
     * @return bool
     */
    public function get($url, $headers = null, $options = null) {
        return $this->request($url, "GET", null, $headers, $options);
    }

    /**
     * Perform POST request
     *
     * @param string $url
     * @param  $post_data
     * @param  $headers
     * @param  $options
     * @return bool
     */
    public function post($url, $post_data = null, $headers = null, $options = null) {
        return $this->request($url, "POST", $post_data, $headers, $options);
    }

    /**
     * Execute the curl
     *
     * @param int $window_size Max number of simultaneous connections
     * @return string|bool
     */
    public function execute($window_size = null) {
        // rolling curl window must always be greater than 1
        if (sizeof($this->requests) == 1) {
            return $this->single_curl();
        } else {
            // start the rolling curl. window_size is the max number of simultaneous connections
            return $this->rolling_curl($window_size);
        }
    }

    /**
     * Performs a single curl request
     *
     * @access private
     * @return string
     */
    private function single_curl() {
        $ch = curl_init();      
        $request = array_shift($this->requests);
        $options = $this->get_options($request);
        curl_setopt_array($ch,$options);
        $output = curl_exec($ch);
        $info = curl_getinfo($ch);

        // it's not neccesary to set a callback for one-off requests
        if ($this->callback) {
            $callback = $this->callback;
            if (is_callable($this->callback)){
                call_user_func($callback, $output, $info, $request);
            }
        }
      else
            return $output;
  return true;
    }

    /**
     * Performs multiple curl requests
     *
     * @access private
     * @throws RollingCurlException
     * @param int $window_size Max number of simultaneous connections
     * @return bool
     */
    private function rolling_curl($window_size = null) {
        if ($window_size)
            $this->window_size = $window_size;

        // make sure the rolling window isn't greater than the # of urls
        if (sizeof($this->requests) < $this->window_size)
            $this->window_size = sizeof($this->requests);

        if ($this->window_size < 2) {
            throw new RollingCurlException("Window size must be greater than 1");
        }

        $master = curl_multi_init();

        // start the first batch of requests
        for ($i = 0; $i < $this->window_size; $i++) {
            $ch = curl_init();

            $options = $this->get_options($this->requests[$i]);

            curl_setopt_array($ch,$options);
            curl_multi_add_handle($master, $ch);

            // Add to our request Maps
            $key = (string) $ch;
            $this->requestMap[$key] = $i;
        }

        do {
            while(($execrun = curl_multi_exec($master, $running)) == CURLM_CALL_MULTI_PERFORM);
            if($execrun != CURLM_OK) {
                break;
            }
            // a request was just completed -- find out which one
            while($done = curl_multi_info_read($master)) {

                // get the info and content returned on the request
                $info = curl_getinfo($done['handle']);
                $output = curl_multi_getcontent($done['handle']);

                array_push($this->returns, array(
                    'return'    =>  $output,
                    'info'      =>  $info,
                ));

                // send the return values to the callback function.
                $callback = $this->callback;
                if (is_callable($callback)){
              $key = (string)$done['handle'];
                    $request = $this->requests[$this->requestMap[$key]];
                    unset($this->requestMap[$key]);
                    call_user_func($callback, $output, $info, $request);
                }

                // start a new request (it's important to do this before removing the old one)
                if ($i < sizeof($this->requests) && isset($this->requests[$i]) && $i < count($this->requests)) {
                    $ch = curl_init();
                    $options = $this->get_options($this->requests[$i]);
                    curl_setopt_array($ch,$options);
                    curl_multi_add_handle($master, $ch);

                    // Add to our request Maps
                    $key = (string) $ch;
                    $this->requestMap[$key] = $i;
                    $i++;
                }

                // remove the curl handle that just completed
                curl_multi_remove_handle($master, $done['handle']);

            }

            // Block for data in / output; error handling is done by curl_multi_exec
            if ($running) {
                curl_multi_select($master, $this->timeout);
            }

        } while ($running);
        curl_multi_close($master);
        return true;
    }


    /**
     * Helper function to set up a new request by setting the appropriate options
     *
     * @access private
     * @param Request $request
     * @return array
     */
    private function get_options($request) {
        // options for this entire curl object
        $options = $this->__get('options');
        // NOTE: The PHP cURL library won't follow redirects if either safe_mode is on
        // or open_basedir is defined.
        // See: https://bugs.php.net/bug.php?id=30609
      if (( ini_get('safe_mode') == 'Off' || !ini_get('safe_mode') )
            && ini_get('open_basedir') == '') {
            $options[CURLOPT_FOLLOWLOCATION] = 1;
          $options[CURLOPT_MAXREDIRS] = 5;
        }
        $headers = $this->__get('headers');

      // append custom options for this specific request
      if ($request->options) {
            $options = $request->options + $options;
        }

      // set the request URL
        $options[CURLOPT_URL] = $request->url;

        // posting data w/ this request?
        if ($request->post_data) {
            $options[CURLOPT_POST] = 1;
            $options[CURLOPT_POSTFIELDS] = $request->post_data;
        }
        if ($headers) {
            $options[CURLOPT_HEADER] = 0;
            $options[CURLOPT_HTTPHEADER] = $headers;
        }

        // Due to a bug in cURL CURLOPT_WRITEFUNCTION must be defined as the last option
        // Otherwise it doesn't register. So let's unset and set it again
        // See http://stackoverflow.com/questions/15937055/curl-writefunction-not-being-called
        if( ! empty( $options[CURLOPT_WRITEFUNCTION]) ) {
            $writeCallback = $options[CURLOPT_WRITEFUNCTION];
            unset( $options[CURLOPT_WRITEFUNCTION] );
            $options[CURLOPT_WRITEFUNCTION] = $writeCallback;
        }

        return $options;
    }

    /**
     * @return void
     */
    public function __destruct() {
        unset($this->window_size, $this->callback, $this->options, $this->headers, $this->requests);
  }
}
```

***RollingCurlGroup.php***:
```php
/*

  Authored by Fabian Franz (www.lionsad.de)
  Released under Apache License 2.0

$Id$
*/

class RollingCurlGroupException extends Exception {}

abstract class RollingCurlGroupRequest extends RollingCurlRequest
{
        private $group = null;

  /**
  * Set group for this request
  *
  * @param group The group to be set
  */
        function setGroup($group)
        {
                if (!($group instanceof RollingCurlGroup))
                        throw new RollingCurlGroupException("setGroup: group needs to be of instance RollingCurlGroup");

                $this->group = $group;
        }

  /**
  * Process the request
  *
  *
  */
        function process($output, $info)
        {
                if ($this->group)
                        $this->group->process($output, $info, $this);
        }

  /**
  * @return void
  */
  public function __destruct() {
      unset($this->group);
      parent::__destruct();
  }

}

class RollingCurlGroup
{
        protected $name;
        protected $num_requests = 0;
        protected $finished_requests = 0;
        private $requests = array();

        function __construct($name)
        {
                $this->name = $name;
        }

  /**
  * @return void
  */
  public function __destruct() {
      unset($this->name, $this->num_requests, $this->finished_requests, $this->requests);
  }


        function add($request)
        {
                if ($request instanceof RollingCurlGroupRequest)
                {
                        $request->setGroup($this);
                        $this->num_requests++;
                        $this->requests[] = $request;
                }
      else if (is_array($request))
                {
          foreach ($request as $req)
              $this->add($req);
      }
                else
                        throw new RollingCurlGroupException("add: Request needs to be of instance RollingCurlGroupRequest");

      return true;
        }

        function addToRC($rc)
        {
      $ret = true;

                if (!($rc instanceof RollingCurl))
                        throw new RollingCurlGroupException("addToRC: RC needs to be of instance RollingCurl");

                while (count($this->requests) > 0)
      {
          $ret1 = $rc->add(array_shift($this->requests));
          if (!$ret1)
              $ret = false;
      }

      return $ret;
        }

        function process($output, $info, $request)
        {
                $this->finished_requests++;

                if ($this->finished_requests >= $this->num_requests)
                        $this->finished();
        }

        function finished()
        {
        }

}

class GroupRollingCurl extends RollingCurl {

  private $group_callback = null;

  protected function process($output, $info, $request)
  {
      if( $request instanceof RollingCurlGroupRequest)
          $request->process($output, $info);

      if (is_callable($this->group_callback))
          call_user_func($this->group_callback, $output, $info, $request);
  }

  function __construct($callback = null)
  {
      $this->group_callback = $callback;

      parent::__construct(array(&$this, "process"));
  }

  public function add($request)
  {
      if ($request instanceof RollingCurlGroup)
          return $request->addToRC($this);
      else
          return parent::add($request);
  }

  public function execute($window_size = null) {

      if (count($this->requests) == 0)
          return false;

      return parent::execute($window_size);
  }

}
```

###Fazendo a Mágica
Agora que temos tudo, vamos criar uma página que faz esse processo para nós, no nosso caso, vamos transferir imagens de uma pasta para outra, mas poderíamos simplesmente chamar isso de um banco de dados, ou outro lugar.

***index.php***:
```php
require_once "class/RollingCurl.php";
define ('URLPREFIX', 'http://127.0.0.1/processaImagem/services/resize.php');

$origin = scandir('image/origin');
$origin = array_slice($origin,2,count($origin));

$destination = scandir('image/destination');
$destination = array_slice($destination,2,count($destination));
$terms_list = array_diff($origin, $destination);

$rc = new RollingCurl();
$rc->window_size = 5; // limit requests per batch
foreach ($terms_list as $terms) {
    $search_url = URLPREFIX.'?file='.urlencode("../image/origin/".$terms);
    $request = new RollingCurlRequest($search_url);
    $rc->add($request);
}
if($rc->execute()){
    foreach($rc->getReturns() as $key => $value){
        echo $value['return']. '<br>';
    }
}
```

##Notas finais

As vezes nos deparamos com algumas limitações técnicas ou da propria linguagem, e pra resolvermos esses problemas, precisamos pensar fora da caxa. Existem diversas formas para realizarmos esse tipo de tarefa, mas dentro de uma limitação que tive, essa foi sem dúvida a melhor solução.
Solução no github **[serviceResize](https://github.com/gustavobeavis/serviceResize)**