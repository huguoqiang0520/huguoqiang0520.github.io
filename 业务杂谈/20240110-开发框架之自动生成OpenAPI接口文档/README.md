本文记录并分享为开发框架实现接口文档自动生成功能（借校验绑定逻辑）的经历、思路和技术实现过程。

# 目录大纲

- **问题背景**：简述团队业务特点导致需要大量/快速创建项目和接口，对接口文档需求量大；
- **团队现状**：阐述团队内的技术规范和框架等，是自动同步生成接口文档的可利用条件；
- **优化思路**：记录思考过程；
- **分析实现**：简述程序实现；
- **成果未来**：陈列当前成果和未来计划。

# 问题背景

## 团队业务背景

团队会承接大量各游戏部门宣发、运营活动需求（也有其他类需求），这类需求呈现的基本特征：运行周期短、数量繁多、定制化程度高。
从后端项目和接口角度看，这意味着我们需要频繁创建/新增项目和接口。

## 接口文档

- **OpenAPI**：团队已经统一采用OpenAPI（原Swagger）标准作为接口文档规范标准。
- **业界方案**：大多数语言都提供了OpenAPI相关库包和工具，通过按照OpenAPI概念编写注释/注解等内容后手动执行命令生成接口文档。

```go
package demo

// GetStringByInt example
// @Summary Add a new pet to the store
// @Description get string by ID
// @ID get-string-by-int
// @Accept  json
// @Produce  json
// @Param   some_id      path   int     true  "Some ID"
// @Param   some_id      body web.Pet true  "Some ID"
// @Success 200 {string} string	"ok"
// @Failure 400 {object} web.APIError "We need ID!!"
// @Failure 404 {object} web.APIError "Can not find ID"
// @Router /testapi/get-string-by-int/{some_id} [get]
func GetStringByInt(c *gin.Context) {
}
```

## 主要问题

- **成本高**：主要体现在认知成本（需要理解和使用OpenAPI概念）和编写成本（每个项目大约会占据10～15%的时间进行接口文档撰写和发布）；
- **同步难**：接口变更、新增时，容易出现忘记同步调整接口文档、忘记重新生成接口文档等导致接口文档与服务本身不同步现象；
- **心情差**：印证那句最讨厌不写文档的人和让我写文档，接口开发者对于编写文档其实并不是那么开心。

# 团队现状

## 团队规范

后端团队有一套共识的接口规范和标准，比如：

- 对外入口域名按业务用途划分成几套固定域名；
- POST接口请求Body统一application/json；
- 接口响应Body统一application/json；
- 接口响应Body约定`{"code": 0, "message": "success", "data": ...}`，其中code代表业务处理结果，data放置业务数据

## 参数验绑要求

为了接口服务安全和质量，要求（同时集成在开发框架与工具中）：

- 接口请求参数必须校验：各语言都有校验框架，只需要集成至团队开发框架后在业务开发是编写校验规则即可；
- 请求参数和响应必须绑定模型：部分语言也提供模型绑定框架，同样集成至团队开发框架后在业务开发中进行使用即可。

```go
package demo

type Test struct {
	kernel.Api
	Request struct {
		Body body
	}
	Response body
}

// 业务开发人员定义参数、响应的具体结构属性以及验证规则
type body struct {
	Project    string `json:"project" binding:"required"`
	Path       string `json:"path" binding:"required"`
	ActivityId int    `json:"activity" binding:"required,min=1,max=10"`
}

func (t Test) Handle(ctx *gin.Context) {
}

```

# 优化思路

## 固化部分

从团队现状-团队规范中不难发现，其实一部分接口文档所需要编写的内容（OpenAPI的概念）在大部分场景下用不到或可以固化。比如：

- Server Object：可以按照项目用途当项目初始化时在脚手架中直接填充；
- Media Type Object：直接固化为application/json，如有需要留下可配口即可；
- Response Object：直接固化完整模板开放Body中的data映射业务响应具体数据即可。

## 重叠部分

结合问题背景-接口文档和团队现状-参数验绑要求来看，也不难发现很多接口文档所需要编写的内容（OpenAPI的概念）在参数验绑框架中也已经被声明定义过了。比如：

```go
package demo

type body struct {
	Project string `json:"project" binding:"required"`
	Path    string `json:"path" binding:"required"`
	// int声明参数类型为number；
	// json tag声明参数序列化名为activity；
	// binding tag声明参数为必传、介于1和10之间
	ActivityId int `json:"activity" binding:"required,min=1,max=10"`
}

```

同理接口路由注册也声明了部分接口相关信息：

```go
package demo

// POST声明这是一个POST接口；/test声明该接口path为/test（当然需要结合完整group）
g.POST("/test", controllers.TestHandler())

```

## 欠缺部分

整理一下发现剩余接口文档需要但团队现有功能覆盖不到的内容基本只剩下了：

- 项目描述：项目仓库、部署平台等处都有补充，一般不关心；
- 接口描述：良好的接口路由一般已经具备了理解性，当然如果有就更好了；
- 参数描述：良好的参数命名一般已经具备了理解性，当然如果有就更好了。

## 消除哪个？

既然二者有很大的重叠部分，那么很显然方案就是消除一个，一般有两条路线：

- **消除接口大纲**：用接口文档手动生成接口大纲；
- **消除接口文档**：用接口大纲自动生成接口文档

---

思考过后决定采用**消除接口文档**的路线，主要考虑点如下：

- 消除接口大纲路线仍旧需要手动生成接口大纲，还是埋了一个人的因素导致接口文档和接口服务不同步的隐患;
- 消除接口文档路线则能保证完全同步，验绑框架本身就是服务功能，接口服务什么样，接口文档就这样；
- 接口验证中有更多详细验证规则是接口文档覆盖不到但却重要的，比如条件required、排除、JWT等，采用消除接口大纲路线仍旧还是要编写和细化这些验证规则；
- 参数和响应模型绑定中语言的数据类型一般更加丰富，比如go有(u)int/(u)int8/16/32/64，采用消除接口大纲路线在数据类型上会受限或仍旧需要修改；
- 接口文档需要但接口大纲中未覆盖的反而是相对不重要的内容，只需要留好填充口即可；
- 对于实际的接口服务开发者而言，写代码比写文档开心！

# 分析实现

## 概念对齐

OpenAPI接口文档所需的要素：

- [OpenAPI官网](https://spec.openapis.org/oas/latest.html)
- [Github项目](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.1.0.md)

---

开发框架中的要素：

- 接口路由注册：Method、Api Url、接口实际处理器
- 接口参数/响应模型定义：参数（序列化）名称、参数数据类型
- 接口参数验证规则：比如go的gin采用的[validator](https://github.com/go-playground/validator)

## 数据类型映射

其他语言基本类似，此处拿团队前语言PHP举例：

| OpenAPI | PHP            | Go                                      |
|---------|----------------|-----------------------------------------|
| boolean | bool           | bool                                    |
| number  | int、float      | int、uint、intx系列、uintx系列、float32、float64 |
| string  | string         | string                                  |
| array   | array<subtype> | array、slice                             |
| object  | array<hashmap> | struct                                  |

## 定义与实现

首先定义好OpenAPI的概念结构(Json Schema和OpenAPI Object)：

```go
package documease

type Property struct {
	Items       *Property            `json:"items,omitempty" yaml:"items,omitempty"`
	Properties  map[string]*Property `json:"properties,omitempty" yaml:"properties,omitempty"`
	Description string               `json:"description,omitempty" yaml:"description,omitempty"`

	// null/boolean/number/string/array/object
	Type string `json:"type,omitempty" yaml:"type,omitempty"`
	Enum []any  `json:"enum,omitempty" yaml:"enum,omitempty"`

	// string type: date-time/date/time/duration
	// string type: email/idn-email
	// string type: hostname/idn-hostname
	// string type: ipv4/ipv6
	// string type: uri/uri-reference/iri/iri-reference/uuid
	// string type: uri-template
	// string type: json-pointer/relative-json-pointer
	// string type: regex
	Format string `json:"format,omitempty" yaml:"format,omitempty"`

	// for numbers instances
	MultipleOf       float64 `json:"multipleOf,omitempty" yaml:"multipleOf,omitempty"`
	Maximum          float64 `json:"maximum,omitempty" yaml:"maximum,omitempty"`
	Minimum          float64 `json:"minimum,omitempty" yaml:"minimum,omitempty"`
	ExclusiveMaximum float64 `json:"exclusiveMaximum,omitempty" yaml:"exclusiveMaximum,omitempty"`
	ExclusiveMinimum float64 `json:"exclusiveMinimum,omitempty" yaml:"exclusiveMinimum,omitempty"`

	// for string instances
	MaxLength int64  `json:"maxLength,omitempty" yaml:"maxLength,omitempty"`
	MinLength int64  `json:"minLength,omitempty" yaml:"minLength,omitempty"`
	Pattern   string `json:"pattern,omitempty" yaml:"pattern,omitempty"`

	// for array instances
	MaxItems    int64 `json:"maxItems,omitempty" yaml:"maxItems,omitempty"`
	MinItems    int64 `json:"minItems,omitempty" yaml:"minItems,omitempty"`
	UniqueItems bool  `json:"uniqueItems,omitempty" yaml:"uniqueItems,omitempty"`
	MaxContains int64 `json:"maxContains,omitempty" yaml:"maxContains,omitempty"`
	MinContains int64 `json:"minContains,omitempty" yaml:"minContains,omitempty"`

	// for object instances
	MaxProperties int64    `json:"maxProperties,omitempty" yaml:"maxProperties,omitempty"`
	MinProperties int64    `json:"minProperties,omitempty" yaml:"minProperties,omitempty"`
	Required      []string `json:"required,omitempty" yaml:"required,omitempty"`

	AdditionalProperties bool `json:"additionalProperties,omitempty" yaml:"additionalProperties,omitempty"`
}
```

```go
package documease

type OpenAPI struct {
	Openapi string              `json:"openapi,omitempty" yaml:"openapi,omitempty"`
	Info    Info                `json:"info,omitempty" yaml:"info,omitempty"`
	Servers []Server            `json:"servers,omitempty" yaml:"servers,omitempty"`
	Paths   map[string]PathItem `json:"paths" yaml:"paths"`
}

type Info struct {
	Title       string `json:"title,omitempty" yaml:"title,omitempty"`
	Summary     string `json:"summary,omitempty" yaml:"summary,omitempty"`
	Description string `json:"description,omitempty" yaml:"description,omitempty"`
	Version     string `json:"version,omitempty" yaml:"version,omitempty"`
}

type Server struct {
	Url         string `json:"url" yaml:"url"`
	Description string `json:"description" yaml:"description"`
}

type PathItem struct {
	Summary     string      `json:"summary,omitempty" yaml:"summary,omitempty"`
	Description string      `json:"description,omitempty" yaml:"description,omitempty"`
	Get         *Operation  `json:"get,omitempty" yaml:"get,omitempty"`
	Put         *Operation  `json:"put,omitempty" yaml:"put,omitempty"`
	Post        *Operation  `json:"post,omitempty" yaml:"post,omitempty"`
	Delete      *Operation  `json:"delete,omitempty" yaml:"delete,omitempty"`
	Options     *Operation  `json:"options,omitempty" yaml:"options,omitempty"`
	Head        *Operation  `json:"head,omitempty" yaml:"head,omitempty"`
	Patch       *Operation  `json:"patch,omitempty" yaml:"patch,omitempty"`
	Trace       *Operation  `json:"trace,omitempty" yaml:"trace,omitempty"`
	Servers     []Server    `json:"servers,omitempty" yaml:"servers,omitempty"`
	Parameters  []Parameter `json:"parameters,omitempty" yaml:"parameters,omitempty"`
}

type Operation struct {
	Tags        []string            `json:"tags,omitempty" yaml:"tags,omitempty"`
	Summary     string              `json:"summary,omitempty" yaml:"summary,omitempty"`
	Description string              `json:"description,omitempty" yaml:"description,omitempty"`
	Parameters  []Parameter         `json:"parameters,omitempty" yaml:"parameters,omitempty"`
	RequestBody *RequestBody        `json:"requestBody,omitempty" yaml:"requestBody,omitempty"`
	Responses   map[string]Response `json:"responses,omitempty" yaml:"responses,omitempty"`
	Servers     []Server            `json:"servers,omitempty" yaml:"servers,omitempty"`
}

type Parameter struct {
	Name        string   `json:"name,omitempty" yaml:"name,omitempty"`
	In          string   `json:"in,omitempty" yaml:"in,omitempty"`
	Description string   `json:"description,omitempty" yaml:"description,omitempty"`
	Required    bool     `json:"required,omitempty" yaml:"required,omitempty"`
	Schema      Property `json:"schema,omitempty" yaml:"schema,omitempty"`
}

type RequestBody struct {
	Description string               `json:"description,omitempty" yaml:"description,omitempty"`
	Content     map[string]MediaType `json:"content,omitempty" yaml:"content,omitempty"`
	Required    bool                 `json:"required,omitempty" yaml:"required,omitempty"`
}

type Response struct {
	Description string               `json:"description,omitempty" yaml:"description,omitempty"`
	Content     map[string]MediaType `json:"content,omitempty" yaml:"content,omitempty"`
}

type MediaType struct {
	Schema Property `json:"schema,omitempty" yaml:"schema,omitempty"`
}
```

---

整理并编写从验证规则到OpenAPI属性的解析功能块，这里放一个MAX校验规则的例子（其他规则和数据信息同理）：

```go
package documease

func MAX(binding string, property *Property, kind reflect.Kind) bool {
	if len(binding) < 4 || "max=" != binding[:4] {
		return false
	}
	if reflect.String == kind {
		vi, err := strconv.ParseInt(binding[4:], 10, 64)
		if nil == err {
			property.MaxLength = vi
			return true
		}
	} else if reflect.Slice == kind || reflect.Array == kind {
		vi, err := strconv.ParseInt(binding[4:], 10, 64)
		if nil == err {
			property.MaxItems = vi
			return true
		}
	} else {
		vf, err := strconv.ParseFloat(binding[4:], 64)
		if nil == err {
			property.Maximum = vf
			property.ExclusiveMaximum = 0
			return true
		}
	}
	return false
}
```

---

编写接口文档生成接口逻辑，获取全量接口，逐个通过反射（不需要过分在意性能）获取接口参数/响应验绑信息，并映射称OpenAPI概念对象，最后序列化为json/yaml格式输出：

```go
package documease

func DocumentHandler(engine *gin.Engine) gin.HandlerFunc {
	return func(ctx *gin.Context) {
		// 确定output
		output = configurease.MustBool("config", "documease", "output")

		// openapi版本
		// 项目基本信息
		// 项目服务实例信息
		projectKey := fmt.Sprintf("%s|%s API Document",
			configurease.MustString("config", "base", "group"),
			configurease.MustString("config", "base", "name"))
		serversConfig, err := configurease.GetSection("config", "documease_servers")
		var servers []Server
		if nil == err {
			for key, addr := range serversConfig {
				servers = append(servers, Server{
					Url:         addr,
					Description: key,
				})
			}
		}
		openapi := OpenAPI{
			Openapi: "3.1.0",
			Info: Info{
				Title:       configurease.MustString("config", "documease", "title", projectKey),
				Summary:     configurease.MustString("config", "documease", "summary", projectKey+" Summary"),
				Description: configurease.MustString("config", "documease", "description", projectKey+" Description"),
				Version:     configurease.MustString("config", "documease", "version", "1.0.0"),
			},
			Servers: servers,
			Paths:   map[string]PathItem{},
		}

		// 确定group规则
		groupKeyStart := configurease.MustInt("config", "documease", "group_key_start", 3)
		groupKeyEnd := configurease.MustInt("config", "documease", "group_key_end", 4)

		// 取出所有接口
		routes := engine.Routes()
		for _, route := range routes {
			api, exist := CM[route.Handler]
			if !exist {
				Output("发现未注册CM的接口[%s]\n", route.Path)
				continue
			}

			// 获取接口group、path、method
			group := strings.Join(strings.Split(route.Path, "/")[groupKeyStart:groupKeyEnd], "/")
			pathSlice := strings.Split(route.Path, "/")
			for i, s := range pathSlice {
				if "" != s && (':' == s[0] || '*' == s[0]) {
					pathSlice[i] = "{" + s[1:] + "}"
				}
			}
			path := strings.Join(pathSlice, "/")
			method := strings.ToUpper(route.Method)

			// 获取api的type、value反射
			t := reflect.TypeOf(api)
			v := reflect.ValueOf(api)

			// 获取接口name、description、summary
			info := ParseApiInfo(v)

			// 获取接口request query、header、uri、cookie
			parameters := ParseApiStandRequestParameters(t, "Query", "form", "query")
			parameters = append(parameters, ParseApiStandRequestParameters(t, "Header", "header", "header")...)
			parameters = append(parameters, ParseApiStandRequestParameters(t, "Uri", "uri", "path")...)

			// 获取接口request body
			request := ParseApiStandRequestBody(t)

			// 按标准模式获取接口response
			standResponse := ParseApiStandResponse(t, ParseAllCodeDescriptionList())

			// 组装operation
			operation := &Operation{
				Tags:        []string{group},
				Summary:     info.Name + " " + info.Summary,
				Description: info.Name + " " + info.Description,
				Parameters:  parameters,
				RequestBody: request,
				Responses:   map[string]Response{"200": standResponse},
				Servers:     nil,
			}

			// 放入pathItem
			pathItem, exist := openapi.Paths[path]
			if !exist {
				pathItem = PathItem{}
			}
			switch method {
			case "GET":
				pathItem.Get = operation
			case "PUT":
				pathItem.Put = operation
			case "POST":
				pathItem.Post = operation
			case "DELETE":
				pathItem.Delete = operation
			case "OPTIONS":
				pathItem.Options = operation
			case "HEAD":
				pathItem.Head = operation
			case "PATCH":
				pathItem.Patch = operation
			case "TRACE":
				pathItem.Trace = operation
			}
			openapi.Paths[path] = pathItem
		}

		format := ctx.Query("format")
		if "yaml" == format {
			ctx.YAML(http.StatusOK, openapi)
		} else {
			ctx.JSON(http.StatusOK, openapi)
		}
	}
}
```

# 成果未来

## 当前成果

- 在PHP开发框架和Go开发框架中均实现该套方案；
- 截止目前已有数百项目、上千接口使用并享受了无需编写接口文档，自动且同步生成接口文档的便利性：
    - PHP项目目前共计**150**+项目（且上线）；
    - Go项目目前也已有10个左右项目（**已上线3个**）。

## 未来计划

除了不断维护升级接口文档方案之外，未来规划主要是基于OpenAPI接口文档做功能延伸，以下是目前已在研究的两个方向：

---

### 对接Mock Server

**问题**：目前部门中前后端基于接口文档实现了部分分离与开发并行，但遇到前端需要联调后端接口的各种分支情况时，就需要依赖后端先提测。

**蓝图**：如果将OpenAPI产出的接口文档对接至Mock Server平台后，前端同学就可以自行编辑接口mock规则模拟接口响应分支，从而进一步推动前后端开发的分离，提升开发效率。

---

市面上有一些开源且兼容OpenAPI接口文档导入的Mock Server平台，比如：

- [rap2](https://github.com/thx/rap2-delos)
- [easy-mock](https://github.com/easy-mock/easy-mock)
- [yapi](https://github.com/ymfe/yapi)
- [apifox](https://apifox.com/)

### 接口自动化测试

**问题**：目前QA团队是需要根据接口文档自己构建测试用例进行手动测试的、另外QA团队也有用例平台用于存放和标记用例信息；

**蓝图**：如果奖OpenAPI产出的接口文档对接至自动化测试平台后，QA同学就可以通过配置相关参数和断言语句，从而实现自动化测试，提升接口质量的同时大量降低QA重复工作成本。

---

市面上有一些兼容OpenAPI接口文档导入的自动化测试平台/工具，比如：

- [yapi](https://github.com/ymfe/yapi)
- [apifox](https://apifox.com/)
- [postman](https://www.postman.com/)

### 参考实践案例

- [利用Swagger UI接口文档同步本地Mock数据](https://blog.csdn.net/setsunadoudou/article/details/101604663)
- [在项目中接入YApi的mock服务](https://www.cnblogs.com/Jingge/p/14500043.html)
- [YApi-mocK、自动化测试（简易版）](https://blog.csdn.net/weixin_69681418/article/details/129377705)
- [基于YAPI的API接口单元测试与自动化](https://zhuanlan.zhihu.com/p/595242918)