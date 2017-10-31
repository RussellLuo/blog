categories:
- 技术

tags:
- REST API
- Swagger
- Markdown

title: 基于 Swagger 描述语言为 REST API 生成 Markdown 文档
---

对于 REST API 的开发者而言，不管是对内作为团队的开发文档，还是对外作为给用户的说明文档，API 文档都是不可或缺的。

然而 “文档是死的、代码是活的”，在现实中，文档跟不上代码的更新节奏的情况比比皆是。如何编写 **实时更新的**、**易于阅读的** 文档成了一个普遍的难题。由此，API 描述语言应用而生。

[Swagger][1] 是一个简单但功能强大的 API 表达工具。它具有地球上最大的 API 工具生态系统。数以千计的开发人员，使用几乎所有的现代编程语言，都在支持和使用 Swagger。使用 Swagger 生成 API，我们可以得到交互式文档，自动生成代码的 SDK 以及 API 的发现特性等（参考 [使用Swagger生成RESTful API文档][2]）。

Swagger 的功能很丰富，但在这里我们只关心一点：如何基于简单的 Swagger 描述语言，为 REST API 生成易读的 Markdown 离线文档。


## 一、基于 Swagger Spec 编写 API 描述文档

这一步无需多说，打开你喜欢的编辑器，或者使用官方的 [Swagger Editor][3]，参考 [Spec 语法][4] 编写即可。

这里我们以 [petstore-minimal.yaml][5] 为例：

```yaml
---
  swagger: "2.0"
  info: 
    version: "1.0.0"
    title: "Swagger Petstore"
    description: "A sample API that uses a petstore as an example to demonstrate features in the swagger-2.0 specification"
    termsOfService: "http://swagger.io/terms/"
    contact: 
      name: "Swagger API Team"
    license: 
      name: "MIT"
  host: "petstore.swagger.io"
  basePath: "/api"
  schemes: 
    - "http"
  consumes: 
    - "application/json"
  produces: 
    - "application/json"
  paths: 
    /pets: 
      get: 
        description: "Returns all pets from the system that the user has access to"
        produces: 
          - "application/json"
        responses: 
          "200":
            description: "A list of pets."
            schema: 
              type: "array"
              items: 
                $ref: "#/definitions/Pet"
  definitions: 
    Pet: 
      type: "object"
      required: 
        - "id"
        - "name"
      properties: 
        id: 
          type: "integer"
          format: "int64"
        name: 
          type: "string"
        tag: 
          type: "string"
```


## 二、安装转换工具 Swagger2Markup

[Swagger2Markup][6] 是一个 Java 编写的工具，用于将 Swagger 文档转换为 AsciiDoc 或者 Markdown 文档。简直就是为我们这里的需求量身定做的 :-)

安装 Swagger2Markup 的步骤如下：

### 1. 安装 Java

以 Ubuntu 为例，参考 [How To Install Java on Ubuntu with Apt-Get][7] 和 [Ubuntu 安装 JDK 7 / JDK8 的两种方式][8]：

1. 安装默认的 JRE/JDK

    ```bash
    $ sudo apt-get update
    $ # 安装默认的 JRE
    $ sudo apt-get install default-jre
    $ # 安装默认的 JDK
    $ sudo apt-get install default-jdk
    ```

2. 安装 Oracle JDK 8

    ```bash
    $ # 添加 ppa
    $ sudo add-apt-repository ppa:webupd8team/java
    $ sudo apt-get update
    $ # 安装 oracle-java-installer（按提示依次选择 ok 和 yes 即可）
    $ sudo apt-get install oracle-java8-installer
    ```

### 2. 下载 Swagger2Markup 的命令行工具

参考 [Command Line Interface][9]，下载最新的 jar 包（当前为 [swagger2markup-cli-1.3.1.jar][10]）即可。


## 三、使用 Swagger2Markup 将 Swagger 转换为 Markdown

参考 [Command Line Interface][9] 中的步骤：

### 1. 创建一个 `config.properties` 配置文件

设置 markupLanguage 为 MARKDOWN

```
swagger2markup.markupLanguage=MARKDOWN
```

### 2. 将 Swagger 转换为 Markdown

```bash
$ java -jar swagger2markup-cli-1.3.1.jar convert -i /path/to/petstore-minimal.yaml -f /tmp/petstore-minimal -c /path/to/config.properties
```

### 3. 查看生成的文档

```
# Swagger Petstore


<a name="overview"></a>
## Overview
A sample API that uses a petstore as an example to demonstrate features in the swagger-2.0 specification


### Version information
*Version* : 1.0.0


### Contact information
*Contact* : Swagger API Team


### License information
*License* : MIT
*Terms of service* : http://swagger.io/terms/


### URI scheme
*Host* : petstore.swagger.io
*BasePath* : /api
*Schemes* : HTTP


### Consumes

* `application/json`


### Produces

* `application/json`




<a name="paths"></a>
## Paths

<a name="pets-get"></a>
### GET /pets

#### Description
Returns all pets from the system that the user has access to


#### Responses

|HTTP Code|Description|Schema|
|---|---|---|
|**200**|A list of pets.|< [Pet](#pet) > array|


#### Produces

* `application/json`




<a name="definitions"></a>
## Definitions

<a name="pet"></a>
### Pet

|Name|Schema|
|---|---|
|**id**  <br>*required*|integer (int64)|
|**name**  <br>*required*|string|
|**tag**  <br>*optional*|string|


```


## 四、CLI as a service

如果团队内部人员都会用到这个工具，但是又不想在每个人的电脑上都安装 Java 和 Swagger2Markup，这时可以基于命令行工具 Swagger2Markup 提供一个 “文档转换服务”。

作为示例，以下是使用 Python 语言并且借助 [RESTArt][11] 库实现的一个 “文档转换服务”：

```python
# swagger2markdown.py

import os
import tempfile

from restart import status
from restart.api import RESTArt
from restart.parsers import Parser
from restart.renderers import Renderer
from restart.resource import Resource

api = RESTArt()


class SwaggerParser(Parser):

    content_type = 'text/plain'

    def parse(self, stream, content_type, content_length, context=None):
        return stream.read()


class MarkdownRenderer(Renderer):

    content_type = 'text/plain'
    format_suffix = 'md'

    def render(self, data, context=None):
        return data


@api.register
class SwaggerMarkdownDocs(Resource):

    name = 'swagger_markdown_docs'

    parser_classes = (SwaggerParser,)
    renderer_classes = (MarkdownRenderer,)

    def create(self, request):
        with tempfile.NamedTemporaryFile(suffix='.yml', delete=False) as yml:
            yml_filename = yml.name
            yml.write(request.data)

        with tempfile.NamedTemporaryFile(suffix='.md', delete=False) as md:
            md_filename = md.name

        jar = '/path/to/swagger2markup-cli-1.3.1.jar'
        conf = '/path/to/config.properties'
        os.system('java -jar {jar} convert -i {yml} -f {md} -c {conf}'.format(
            jar=jar, yml=yml_filename, md=md_filename[:-len('.md')], conf=conf,
        ))

        with open(md_filename) as md:
            content = md.read()

        os.unlink(yml_filename)
        os.unlink(md_filename)

        return content, status.HTTP_201_CREATED
```

启动 “文档转换服务”：

```bash
$ restart swagger2markdown:api
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

使用 “文档转换服务” 生成 Markdown 文档：

```bash
$ curl -H 'Content-Type: text/plain' -XPOST http://localhost:5000/swagger_markdown_docs --data-binary @/path/to/petstore-minimal.yaml > /tmp/petstore-minimal.md
```


[1]: https://swagger.io/
[2]: https://www.xncoding.com/2017/06/09/restful/swagger.html
[3]: https://swagger.io/swagger-editor/
[4]: https://swagger.io/specification/
[5]: https://github.com/OAI/OpenAPI-Specification/blob/master/examples/v2.0/yaml/petstore-minimal.yaml
[6]: https://github.com/Swagger2Markup/swagger2markup
[7]: https://www.digitalocean.com/community/tutorials/how-to-install-java-on-ubuntu-with-apt-get
[8]: http://www.cnblogs.com/a2211009/p/4265225.html
[9]: http://swagger2markup.github.io/swagger2markup/1.3.1/#_command_line_interface
[10]: https://jcenter.bintray.com/io/github/swagger2markup/swagger2markup-cli/1.3.1/
[11]: https://restart.readthedocs.io/en/latest/
