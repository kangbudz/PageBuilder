# PageBuilder - HTML assembly aid for ESP8266WebServer
*An arduino library to create html string in the sketch for ESP8266WebServer.* [![Build Status]()]()

PageBuilder is an Arduino library class dedicated to the _ESP8266WebServer_ for easily generating HTML pages and sending them to client.

## Features

* Ability to completely separate HTML structure and the web page generation logic in the sketch
* No need for inline coding of URI access handler of ESP8266WebServer class
* Fixed HTML statement parts like template can be allocated as PROGMEM
* Arbitrary token can be specified inline HTML statement
* Automatically sent to client for HTML page are generated


<table>
<th>Ordinary sketch</th><th>Sketch by PageBuilder</th>
<tr><td>
<pre><code class="language-c++">
#include "ESP8266WebServer.h"
ESP8266WebServer Server;

// This is HTML structure, but imcompleted.
static const char _ELEMENT_HELLO[] PROGMEM = {
  "<html><head><style type=\"text/css\">p.foo { font-size:larger; }</style>"
  "<body>hello, "
};

// This is complecated assemble process of HTML.
void hello() {
  String html = FPSTR(_ELEMENT_HELLO);
  if (Server.hasArg("foo")) {
    html += "<p class=\"foo\">";
    html += Server.arg("foo");
    html += "</p></body></html>";
  }
  else {
    html += "<p>everybody!</p></body></html>";
  }
  Server.send(200, "text/html", html);
}

void setup() {
  Server.on("/hello", hello);
  Server.begin();
}

void loop() {
  Server.handleClient();
}
</code></pre></td><td>
<pre><code class="language-c++">
#include "ESP8266WebServer.h"
#include "PageBuilder.h"

// This is HTML structure.
static const char _ELEMENT_HELLO[] PROGMEM = {
  "<html><head>"
  "<style type=\"text/css\">p.foo {font-size:larger;}</style>"
  "</head>"
  "<body>hello, <p class=\"{{STYLE}}\">{{FOO}}</p></body>"
  </html>"
};

// This is style section treatment.
String style(PageArgument& args) {
  return args.hasArg("foo") ?
    String("foo") : String("");
}

// This is body section treatement.
String echo(PageArgument& args) {
  return args.hasArg("foo") ?
    args.arg("foo") : String("everybody");
}

// A PageBuilder object can have multiple elements,
// and each element has an HTML structure.
PageElement HELLO_PAGE_CONTENT(_ELEMENT_HELLO, {
  {"STYLE", style}, 
  {"FOO", echo}
 });
PageBuilder HelloPage("/hello", {HELLO_PAGE_CONTENT});
ESP8266WebServer Server;

void setup() {
  HelloPage.insert(Server);
  Server.begin();
}

void loop() {
  Server.handleClient();
}
</code></pre></td></tr>
</table>  

## Works on

Generic esp8266 module and other representatives works fine. ESP8266 Arduino core 2.3.0 is necessary. 

## Installation

Download this file as a zip, and extract the resulting folder into your Arduino Libraries folder. See ![Installing Additional Arduino Libraries](https://www.arduino.cc/en/Guide/Libraries).  
Required [Arduino IDE](http://www.arduino.cc/en/main/software) is current upstream at **the 1.8 level or later**, and also [ESP8266 Arduino core 2.3.0](https://github.com/esp8266/Arduino).

## Example

- A most simple example. - No uri handler is needed, only the uri path and html coded.

```c++
#include "PageBuilder.h"

// root page
PageElement ROOT_PAGE_ELEMENT("<a href=\"/hello\">hello</a>");
PageBuilder ROOT_PAGE("/", {ROOT_PAGE_ELEMENT});
// /hello page
PageElement HELLO_PAG_ELEMENT("Hello, world!<p><a href=\"/bye\">bye</a></p>");
PageBuilder HELLO_PAGE("/hello", {HELLO_PAG_ELEMENT});
// /bye page
PageElement BYE_PAGE_ELEMENT("Good bye!");
PageBuilder BYE_PAGE("/bye", {BYE_PAGE_ELEMENT});

ROOT_PAGE.insert(Server);     // Add root page
HELLO_PAGE.insert(Server);    // Add /hello page
BYE_PAGE.insert(Server);      // add /bye page
```
- Share multiple elements on different pages.

```c++
#include "PageBuilder.h"

static const char _HEAD[] PROGMEM = "<html>" ;
static const char _BODY[] PROGMEM = "<body>This is {{PAGE}}.</body>";
static const char _FOOT[] PROGMEM = "</html>";

String setBody1(PageArgument& args) {
	return String("Page1");
}

String setBody2(PageArgument& args) {
	return String("Page2");
}

PageElement header( _HEAD );
PageElement body1( _BODY, { {"PAGE", setBody1} });
PageElement body2( _BODY, { {"PAGE", setBody2} });
PageElement footer( _FOOT );

PageBuilder Page1( "/page1", {header, body1, footer} );
PageBuilder Page2( "/page2", {header, body2, footer} );
```

## Usage

### Data structure of PageBuilder

In order to successfully generate an HTML page using PageBuilder please understand the data structure of PageBuilder.  
PageBuilder library consists of three objects that are related to each other as the below. `PageBuilder` inherits `RequestHandler` provided from ESP8266WebServer library and is invoked from `ESP8266WebServer` in response to http requests. PageBuilder owns its uri string and multiple PageElement objects.  
Source strings of HTML are owned by `PageElement` (`mold` in the figure). Its string contains an identifier called a **token**. The **token** appears as `{{ }}` in the middle of the source HTML string (`_token` in the figure). The tokens are paired with functions to replace them with actual HTML sentences. When uri access has occurred server from the client, its paired function is invoked by extension of `handleClient()` method then the **token** will replace to actual statement to complete the HTML and sends it. `PageElement` can have multiple tokens (ie, it can define several tokens in one HTML source element).  
![default_data_structure](https://user-images.githubusercontent.com/12591771/33360699-293dc5ac-d518-11e7-8d31-728d500f02bf.png)  
To properly generate a web page, you need to code its function that replaces the token with HTML, and its function must return a String.  
```c++
String AsName(PageArgument& args) {      // User coded function
...    ~~~~~~
  return String("My name");
}
String AsDaytime(PageArgument& args) {   // User coded function
...    ~~~~~~~~~
  return String("afternoon");
}

// Source HTML string
const char html[] = "hello <b>{{NAME}}</b>, <br>Good {{DAYTIME}}.";
...                             ^^^^                   ^^^^^^^
...                             token                  token

PageElement header_elem("<html><body>");
PageElement footer_elem("</body></html>");
PageElement body_elem(html, { {"NAME", AsName}, {"DAYTIME", AsDayTime} });
...                             ^^^^   ~~~~~~     ^^^^^^^   ~~~~~~~~~
...                             token  User coded function to replace the token

PageBuilder page("/hello", { header_elem, body_elem, footer_elem });
...
ESP8266WebServer  webServer;
page.insert(webServer);
webServer.begin();
...  // 'on' method is no needed.
webServer.handleClient();
```
http\://your.webserver.address/hello will respond as follows.

```html
<html><body>hello <b>My name</b>, <br>Good afternoon.</body></html>
```

### Invoke the HTML assemble

No need in the sketch. It would be invoked from ESP8266WebServer. It is registration necessary to the ESP8266WebServer like as `on` method.
```c++
String func(PageArgument& args);
PageElement element("mold", {{"token", func}})
PageBuilder page("/uri", { element });

ESP8266WebServer server;
page.insert(server);    // This is needed.

server.handleClient();  // Invoke from this.
```

### Arguments of invked user function

Arguments are passed to the **function** that should be implemented corresponding to tokens. It is the parameter value as GET or POST at the http request occurred like as `url?param=value`, and its parameters are stored in `PageArgument` object and passed to the function as below.  
- HTTP GET with `http://xxx.xxx.xxx/?param=value`  

```c++
String func(PageArgument& args) {
  if (args.hasArg("param"))
    return args.arg("param");
}

PageElement element("hello {{NAME}}.", {{"NAME", func}});
```
An argument can be accessed with the following method of `PageArgument` class.

#### `String arg(String name)`  
Returns the value of the parameter specified by `name`.

#### `String arg(int i)`  
Returns the value of the parameter indexed `i`.

#### `String argName(int i)`  
Retuens parameter name of indexed i.

#### `int args()`  
Get parameters count of the current http request.

#### `size_t size()`  
Same as `args()`.

#### `bool hasArg(String name)`  
Returns whether the `name` parameter is specified in the current http request.

## Declare PageBuilder object and PageElement object

### Include directive

```c++
#include "PageBuilder.h"
```

#### PageBuilder constructor
```c++
PageBuilder::PageBuilder();
PageBuilder::PageBuilder(PageElementVT element, HTTPMethod method = HTTP_ANY);
PageBuilder::PageBuilder(const char* uri, PageElementVT element, HTTPMethod method = HTTP_ANY);
```
- `element` : **PageElement** container wrapper.
- `method` : Enum value of HTTP method as `HTTP_ANY`, `HTTP_GET`, `HTTP_POST` that page should respond.
- `uri` : uri string of the page.

#### PageElement constructor
```c++
PageElement::PageElement();
PageElement::PageElement(const char* mold);
PageElement::PageElement(const char* mold, TokenVT source);
```
- `mold` : A pointer to HTML model string(const char array, PROGMEM available).
- `source` : Container of processable token and handler function.

## Methods

### PageBuilder Methods

#### `void addElement(PageElement& element)`  
Add a new **PageElement object** to the container of **PageBuilder**. 
- `element` : PageElement object.

#### `String build(void)`
Returns the built html string from `const char* mold` that processed *token* by the user *function* of **TokenVT** which code as `{"token",function_name}`. The `build` method handles all *PageElement* objects that a *PageBuilder* contained.

#### `void insert(ESP8266WebServer& server)`  
Register the page and starts handling. It has the same effect as `on` method of `ESP8266WebServer`.
- `server` : A reference of ESP8266WebServer object to register the page.

#### `void setUri(const char* uri)`  
Set uri of this page.
- `uri` : A pointer of uri string.

#### `const char* uri()`  
Get uri of this page.

### PageElement methods

#### `const char* mold()`  
Get mold string in the PageElement.

#### `String build()`  
Returns the HTML element string from `const char* mold` that processed *token* by the user *function* of **TokenVT**.

License
-----------
The PseudoPWM class is licensed under the [MIT License](LICENSE.md).  
Copyright &copy; 2017 hieromon@gmail.com
