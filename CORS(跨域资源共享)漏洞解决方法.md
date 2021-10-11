# 一、低危漏洞：CORS漏洞问题

测试人员访问某个url，将请求头中的Origin字段修改为任意值，结果仍然能获得正确的响应报文，就说明有CORS漏洞。
当CORS的设置不正确时，就会带来安全问题；当响应头中的Access-Control-Allow-Origin设置为null或*时，表示信任任何域，这时候就可能引入安全问题。
修复方法是合理配置CORS，判断Origin是否合法；具体说就是不让在nginx或tomcat中配置【Access-Control-Allow-Origin *】或【Access-Control-Allow-Origin null】。

# 二、解决方法

1. 在nginx配置文件中配置：

(1)一种写法，用*；
```
location / {
  add_header Access-Control-Allow-Origin *.xxx.com;
  add_header Access-Control-Allow-Headers "Origin， X-Requested-With, Content-Type, Accept";
  add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
}
```
(2)另一种写法，指定域名；
```
location / {
  add_header Access-Control-Allow-Origin http://www.hao123.com;
  add_header Access-Control-Allow-Headers "Origin， X-Requested-With, Content-Type, Accept";
  add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
}
```
(3)另一种写法，指定ip与端口，也可以逗号拼接；
```
location / {
  add_header Access-Control-Allow-Origin http://10.130.222.222:6500,http://10.130.222.223:6500;
  add_header Access-Control-Allow-Headers "Origin， X-Requested-With, Content-Type, Accept";
  add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
}
```
(4)另一种写法，使用正则表达式；
```
location ~ /myurl(.*) {
  if ( $http_origin ~ '^http(s)?://(localhost|10\.130\.222\.222):6500$' ){
  add_header Access-Control-Allow-Origin $http_origin;
  }
  if ( $http_origin ~ '^http(s)?://(localhost|10\.130\.222\.223):6500$' ){
  add_header Access-Control-Allow-Origin $http_origin;
  }
  
  add_header Access-Control-Allow-Headers "Origin， X-Requested-With, Content-Type, Accept";
  add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
}
```
说明：
- $ http_origin可以获取到请求头中的Origin字段；但是如果请求头没有，就获取不到了；
- ^是正则表达式，表示开头位置；$是正则表达式，表示结尾位置
- ?是正则表达式，表示s可能有，也可能没有，这两种情况都可以匹配
- .是把.转义成普通字符的意思
- nginx中，if后必须加空格，然后才能写(，否则会报错
- nginx中，没有else if

2. 如果有必要，在tomcat配置(有nginx时，配nginx即可，tomcat就不需要配置了)：
- 把cors-filter-1.7.jar与java-property-utils-1.9.jar这两个文件放到tomcat的lib目录下
- 在tomcat的web.xml中配置：
```
<filter>
  <filter-name>CORS</filter-name>
  <filter-class>com.thetransactioncompany.cors.CORSFilter</filter-class>
  <init-param>
    <param-name>cors.allowOrigin</param-name>
    <!-- <param-value>*</param-value> -->
    <!-- 允许访问的网站，多个时用逗号分隔，*代表允许所有 -->
    <param-value>*.xxx.com,http://10.130.222.222:6500</param-value> 
  </init-param>
    <init-param>
    <param-name>cors.exposedHeaders</param-name>
    <param-value>Set-Cookie</param-value> 
  </init-param>
      <init-param>
    <param-name>cors.supportsCredentials</param-name>
    <param-value>true</param-value> 
  </init-param>
</filter>
<filter-mapping>
  <filter-name>CORS</filter-name>
  <url-pattern>/*</urlpattern>
</filter-mapping>
```
# 三、其它相关笔记
相关网址
http://www.ruanyifeng.com/blog/2016/04/cors.html
https://www.cnblogs.com/ytssjbhy616/p/12779506.html

nginx配置中，可以使用log_format指定日志格式，需要写在http域中，如下：
```
http {
  log_format myerror '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     'Origin: $http_origin '
}
```
上方代码自定义了一个日志格式，myerror，配置了打印时要打印出请求头中的Origin来($http_origin)。

然后可以指定access_log使用自己配置的那种日志格式；access_log可以配置在http域中，也可以配置在server域中，可以配置多个，如下:
```
server {
  access_log /usr/local/nginx/logs/mylog/myserver.access_$year-$month-$day.log myerror;
}
```
上方代码指定了日志输出的路径，日志名随时间变化，日志格式为myerror;
nginx的这个server每收到一个请求，就会在日志中打印一行，就能发现请求头中是否含有Origin了。

nginx配置中，不能给error_log指定自定义的日志格式，否则会报错；
首先应该明白，error_log中打印的错误信息一般不是我们需要的；我们需要的日志一般都在access_log中(不管请求的响应码$status是多少，一般都打印在access_log中，不是说404、500的请求就打印到error_log了)

nginx的一个location配置：
```
location ~ \/my-server\/[^\/]*\/env$ {
  return 404;
}
```
如果请求的url是类似/my-server/*/env的，就返回404；
[ ^ \ / ]的意思是，不匹配/my-server/env的。

原文链接：https://blog.csdn.net/BHSZZY/article/details/119024992
