# paginate - Gorm Pagination

Simple way to paginate [Gorm](https://github.com/go-gorm/gorm) result. **paginate** is compatible with [net/http](https://golang.org/pkg/net/http/) and [fasthttp](https://github.com/valyala/fasthttp). This library also supports many net/http or fasthttp based frameworks.

## Table Of Contents
- [Installation](#installation)
- [Configuration](#configuration)
- [Pagination Result](#pagination-result)
- [Paginate using http request](#paginate-using-http-request)
- [Example usage](#example-usage)
  - [net/http](#nethttp-example)
  - [Fasthttp](#fasthttp-example)
  - [Mux Router](#mux-router-example)
  - [Fiber](#fiber-example)
  - [Echo](#echo-example)
  - [Gin](#gin-example)
  - [Martini](#martini-example)
  - [Beego](#beego-example)
  - [jQuery DataTable Integration](#jquery-datatable-integration)
  - [jQuery Select2 Integration](#jquery-select2-integration)
  - [Programmatically Pagination](#programmatically-pagination)
- [Filter format](#filter-format)
- [Customize default configuration](#customize-default-configuration)
- [Override results](#override-results)
- [Field Selector](#field-selector)
- [Dynamic Field Selector](#dynamic-field-selector)
- [Speed up response with cache](#speed-up-response-with-cache)
  - [In Memory Cache](#in-memory-cache)
  - [Disk Cache](#disk-cache)
  - [Redis Cache](#redis-cache)
  - [Elasticsearch Cache](#elasticsearch-cache)
  - [Custom cache](#custom-cache)
  - [Clean up cache](#clean-up-cache)
- [Limitations](#limitations)
- [License](#license)

## Installation

```bash
go get -u github.com/mahmut/paginate
```

## Configuration

```go
var db *gorm.DB = ...
var req *http.Request = ...
// or
// var req *fasthttp.Request

stmt := db.Where("id > ?", 1).Model(&Article{})
pg := paginate.New()
page := pg.With(stmt).Request(req).CountField("id").Response(&[]Article{})

log.Println(page.Total)
log.Println(page.Items)
log.Println(page.First)
log.Println(page.Last)

```
you can customize config with `paginate.Config` struct.  
```go
pg := paginate.New(&paginate.Config{
    DefaultSize: 50,
})
```
see more about [customize default configuration](#customize-default-configuration).

## Pagination Result

```js
{
    // the result items
    "items": *[]any, 
    
    // total results
    // including next pages
    "total": number,   

    // Current page
    // (provided by request parameter, eg: ?page=1)
    // note: page is always start from 0
    "page": number,
    
    // Current size
    // (provided by request parameter, eg: ?size=10)
    // note: negative value means unlimited
    "size": number,    

    // Total Pages
    "totalPages": number,

    // Max Page
    // start from 0 until last index
    // example: 
    //   if you have 3 pages (page0, page1, page2)
    //   maxPage is 2 not 3
    "maxPage": number,

    // Last Page is true if the page 
    // has reached the end of the page
    "last": bool,

    // First Page is true if the page is 0
    "first": bool,

    // Visible
    // total visible items
    "visible": number,

    // Error
    // true if an error has occurred and
    // paginate.Config.ErrorEnabled is true
    "error": bool,

    // Error Message
    // current error if available and
    // paginate.Config.ErrorEnabled is true
    "errorMessage": string,
}
```
## Paginate using http request
example paging, sorting and filtering:  
1. `http://localhost:3000/?size=10&page=0&sort=-name`  
    produces:
    ```sql
    SELECT * FROM user ORDER BY name DESC LIMIT 10 OFFSET 0
    ```
    `JSON` response:  
    ```js
    {
        // result items
        "items": [
            {
                "id": 1,
                "name": "john",
                "age": 20
            }
        ],
        "page": 0, // current selected page
        "size": 10, // current limit or size per page
        "maxPage": 0, // maximum page
        "totalPages": 1, // total pages
        "total": 1, // total matches including next page
        "visible": 1, // total visible on current page
        "last": true, // if response is first page
        "first": true // if response is last page
    }
    ```
2. `http://localhost:3000/?size=10&page=1&sort=-name,id`  
    produces:
    ```sql
    SELECT * FROM user ORDER BY name DESC, id ASC LIMIT 10 OFFSET 10
    ```
3. `http://localhost:3000/?filters=["name","john"]`  
    produces:
    ```sql
    SELECT * FROM user WHERE name = 'john' LIMIT 10 OFFSET 0
    ```
4. `http://localhost:3000/?filters=["name","like","john"]`  
    produces:
    ```sql
    SELECT * FROM user WHERE name LIKE '%john%' LIMIT 10 OFFSET 0
    ```
5. `http://localhost:3000/?filters=["age","between",[20, 25]]`  
    produces:
     ```sql
    SELECT * FROM user WHERE ( age BETWEEN 20 AND 25 ) LIMIT 10 OFFSET 0
    ```
6. `http://localhost:3000/?filters=[["name","like","john%25"],["OR"],["age","between",[20, 25]]]`  
    produces:
     ```sql
    SELECT * FROM user WHERE (
        (name LIKE '%john\%%' ESCAPE '\') OR (age BETWEEN (20 AND 25))
    ) LIMIT 10 OFFSET 0
    ```
7. `http://localhost:3000/?filters=[[["name","like","john"],["AND"],["name","not like","doe"]],["OR"],["age","between",[20, 25]]]`  
    produces:
     ```sql
    SELECT * FROM user WHERE (
        (
            (name LIKE '%john%')
                    AND
            (name NOT LIKE '%doe%')
        ) 
        OR 
        (age BETWEEN (20 AND 25))
    ) LIMIT 10 OFFSET 0
    ```
8. `http://localhost:3000/?filters=["name","IS NOT",null]`  
    produces:
    ```sql
    SELECT * FROM user WHERE name IS NOT NULL LIMIT 10 OFFSET 0
    ```
9. Using `POST` method:  
   ```bash
   curl -X POST \
   -H 'Content-type: application/json' \
   -d '{"page":1,"size":20,"sort":"-name","filters":["name","john"]}' \
   http://localhost:3000/
   ```  
10. You can bypass HTTP Request with [Custom Request](#programmatically-pagination).

## Example usage

### NetHTTP Example

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(r).Response(&[]Article{})
        j, _ := json.Marshal(page)
        w.Header().Set("Content-type", "application/json")
        w.Write(j)
    })

    log.Fatal(http.ListenAndServe(":3000", nil))
}
```

### Fasthttp Example

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()

    fasthttp.ListenAndServe(":3000", func(ctx *fasthttp.RequestCtx) {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(&ctx.Request).Response(&[]Article{})
        j, _ := json.Marshal(page)
        ctx.SetContentType("application/json")
        ctx.SetBody(j)
    })
}
```

### Mux Router Example
```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()
    app := mux.NewRouter()
    app.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(req).Response(&[]Article{})
        j, _ := json.Marshal(page)
        w.Header().Set("Content-type", "application/json")
        w.Write(j)
    }).Methods("GET")
    http.Handle("/", app)
    http.ListenAndServe(":3000", nil)
}
```

### Fiber example

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()
    app := fiber.New()
    app.Get("/", func(c *fiber.Ctx) error {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(c.Request()).Response(&[]Article{})
        return c.JSON(page)
    })

    app.Listen(":3000")
}
```

### Echo example

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()
    app := echo.New()
    app.GET("/", func(c echo.Context) error {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(c.Request()).Response(&[]Article{})
        return c.JSON(200, page)
    })

    app.Logger.Fatal(app.Start(":3000"))
}
```

### Gin Example

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()
    app := gin.Default()
    app.GET("/", func(c *gin.Context) {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(c.Request).Response(&[]Article{})
        c.JSON(200, page)
    })
    app.Run(":3000")
}

```

### Martini Example

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()
    app := martini.Classic()
    app.Use(render.Renderer())
    app.Get("/", func(req *http.Request, r render.Render) {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(req).Response(&[]Article{})
        r.JSON(200, page)
    })
    app.Run()
}
```
### Beego Example

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()
    web.Get("/", func(c *context.Context) {
        stmt := db.Joins("User").Model(&Article{})
        page := pg.With(stmt).Request(c.Request).Response(&[]Article{})
        c.Output.JSON(page, false, false)
    })
    web.Run(":3000")
}
```

### jQuery DataTable Integration

```js
var logicalOperator = "OR"

$('#myTable').DataTable({

    columns: [
        {
            title: "Author",
            data: "user.name"
        }, {
            title: "Title",
            data: "title"
        }
    ],

    processing: true,
    
    serverSide: true,

    ajax: {
        cache: true,
        url: "http://localhost:3000/articles",
        dataSrc: function(json) {
            json.recordsTotal = json.visible
            json.recordsFiltered = json.total
            return json.items
        },
        data: function(params) {
            var custom = {
                page: !params.start ? 0 : Math.round(params.start / params.length),
                size: params.length
            }

            if (params.order.length > 0) {
                var sorts = []
                for (var o in params.order) {
                    var order = params.order[o]
                    if (params.columns[order.column].orderable != false) {
                        var sort = order.dir != 'desc' ? '' : '-'
                        sort += params.columns[order.column].data
                        sorts.push(sort)
                    }
                }
                custom.sort = sorts.join()
            }

            if (params.search.value) {
                var columns = []
                for (var c in params.columns) {
                    var col = params.columns[c]
                    if (col.searchable == false) {
                        continue
                    }
                    columns.push(JSON.stringify([col.data, "like", encodeURIComponent(params.search.value.toLowerCase())]))
                }
                custom.filters = '[' + columns.join(',["' + logicalOperator + '"],') + ']'
            }

            return custom
        }
    },
})
```

### jQuery Select2 Integration

```js
$('#mySelect').select2({
    ajax: {
        url: "http://localhost:3000/users",
        processResults: function(json) {
            json.items.forEach(function(item) {
                item.text = item.name
            })
            // optional
            if (json.first) json.items.unshift({id: 0, text: 'All'})

            return {
                results: json.items,
                pagination: {
                    more: json.last == false
                }
            }
        },
        data: function(params) {
            var filters = [
                ["name", "like", params.term]
            ]

            return {
                filters: params.term ? JSON.stringify(filters) : "",
                sort: "name",
                page: params.page && params.page - 1 ? params.page - 1 : 0
            }
        },
    }
})
```

### Programmatically Pagination

```go
package main

import (
    "github.com/mahmut/paginate"
    ...
)

func main() {
    // var db *gorm.DB
    pg := paginate.New()
    req := &paginate.Request{
        Page: 2,
        Size: 20,
        Sort: "-publish_date",
        filters: []interface{}{
            []interface{}{"user.name", "like", "john"},
            []interface{}{"and"},
            []interface{}{"publish_date", ">=", "2025-12-31"},
            []interface{}{"and"},
            []interface{}{"user.active", "=", true},
            []interface{}{"and"},
            []interface{}{"user.last_login", "is not", nil},
        }
    }

    stmt := db.Joins("User").Model(&Article{})
    page := pg.With(stmt).
        Request(req).
        Response(&[]Article{})

}
```


## Filter format

Paginate Filters was inspired by [Frappe Framework API](https://docs.frappe.io/framework/user/en/api/rest#listing-documents). This feature is very powerful to support deep search and keep it safe. The format of filter param is a json encoded of multidimensional array.  
Maximum array members is three, first index is `column_name`, second index is `operator` and third index is `values`, you can also pass array to values.  

```js
// Format:
["column_name", "operator", "values"]

// Example:
["age", "=", 20]
// Shortcut:
["age", 20]

// Produces:
// WHERE age = 20
```

Single array member is known as **Logical Operator**.
```js
// Example
[["age", "=", 20],["or"],["age", "=", 25]]

// Produces:
// WHERE age = 20 OR age = 25
```


You are allowed to send array inside a value.  
```js
["age", "between", [20, 30] ]
// Produces:
// WHERE age BETWEEN 20 AND 30

["age", "not in", [20, 21, 22, 23, 24, 25, 26, 26] ]
// Produces:
// WHERE age NOT IN(20, 21, 22, 23, 24, 25, 26, 26)
```

Define chain columns with same value separated by comma.
```js
// Example 1
["price,discount", ">", 10]
// Produces:
// WHERE price > 10 OR discount > 10

// Example 2
["deleted_at,expiration_date", null]
// Produces:
// WHERE deleted_at IS NULL OR expiration_date IS NULL
```

You can filter nested condition with deep array.  
```js
[
    [
        ["age", ">", 20],
        ["and"]
        ["age", "<", 30]
    ],
    ["and"],
    ["name", "like", "john"],
    ["and"],
    ["name", "like", "doe"]
]
// Produces:
// WHERE ( (age > 20 AND age < 20) and name like '%john%' and name like '%doe%' )
```

For `null` value, you can send string `"null"` or `null` value, *(lower)*
```js
// Wrong request
[ "age", "is", NULL ]
[ "age", "is", Null ]
[ "age", "is not", NULL ]
[ "age", "is not", Null ]

// Right request
[ "age", "is", "NULL" ]
[ "age", "is", "Null" ]
[ "age", "is", "null" ]
[ "age", "is", null ]
[ "age", null ]
[ "age", "is not", "NULL" ]
[ "age", "is not", "Null" ]
[ "age", "is not", "null" ]
[ "age", "is not", null ]
```

## Customize default configuration

You can customize the default configuration with `paginate.Config` struct. 

```go
pg := paginate.New(&paginate.Config{
    DefaultSize: 50,
})
```

Config             | Type       | Default               | Description
------------------ | ---------- | --------------------- | -------------
Operator           | `string`   | `OR`                  | Default conditional operator if no operator specified.<br>For example<br>`GET /user?filters=[["name","like","jo"],["age",">",20]]`,<br>produces<br>`SELECT * FROM user where name like '%jo' OR age > 20`
FieldWrapper       | `string`   | `LOWER(%s)`           | FieldWrapper for `LIKE` operator *(for postgres default is: `LOWER((%s)::text)`)*
DefaultSize        | `int64`    | `10`                  | Default size or limit per page
PageStart          | `int64`    | `0`                   | Set start page, default `0` if not set. `totalPages` , `maxPage` and `page` variable will be affected if you set `PageStart` greater than `0` 
LikeAsIlikeDisabled | `bool`    | `false`               | By default, paginate using Case Insensitive on `LIKE` operator. Instead of using `ILIKE`, you can use `LIKE` operator to find what you want. You can set `LikeAsIlikeDisabled` to `true` if you need this feature to be disabled.
SmartSearchEnabled | `bool`     | `false`               | Enable smart search *(Experimental feature)*
CustomParamEnabled | `bool`     | `false`               | Enable custom request parameter
FieldSelectorEnabled | `bool`   | `false`               | Enable partial response with specific fields. Comma separated per field. eg: `?fields=title,user.name`
SortParams         | `[]string` | `[]string{"sort"}`    | if `CustomParamEnabled` is `true`,<br>you can set the `SortParams` with custom parameter names.<br>For example: `[]string{"sorting", "ordering", "other_alternative_param"}`.<br>The following requests will capture same result<br>`?sorting=-name`<br>or `?ordering=-name`<br>or `?other_alternative_param=-name`<br>or `?sort=-name`
PageParams         | `[]string` | `[]string{"page"}`    | if `CustomParamEnabled` is `true`,<br>you can set the `PageParams` with custom parameter names.<br>For example:<br>`[]string{"number", "num", "other_alternative_param"}`.<br>The following requests will capture same result `?number=0`<br>or `?num=0`<br>or `?other_alternative_param=0`<br>or `?page=0`
SizeParams         | `[]string` | `[]string{"size"}`    | if `CustomParamEnabled` is `true`,<br>you can set the `SizeParams` with custom parameter names.<br>For example:<br>`[]string{"limit", "max", "other_alternative_param"}`.<br>The following requests will capture same result `?limit=50`<br>or `?limit=50`<br>or `?other_alternative_param=50`<br>or `?max=50`
OrderParams         | `[]string` | `[]string{"order"}`    | if `CustomParamEnabled` is `true`,<br>you can set the `OrderParams` with custom parameter names.<br>For example:<br>`[]string{"order", "direction", "other_alternative_param"}`.<br>The following requests will capture same result `?order=desc`<br>or `?direction=desc`<br>or `?other_alternative_param=desc`
FilterParams       | `[]string` | `[]string{"filters"}` | if `CustomParamEnabled` is `true`,<br>you can set the `FilterParams` with custom parameter names.<br>For example:<br>`[]string{"search", "find", "other_alternative_param"}`.<br>The following requests will capture same result<br>`?search=["name","john"]`<br>or `?find=["name","john"]`<br>or `?other_alternative_param=["name","john"]`<br>or `?filters=["name","john"]`
FieldsParams       | `[]string` | `[]string{"fields"}`  | if `FieldSelectorEnabled` and `CustomParamEnabled` is `true`,<br>you can set the `FieldsParams` with custom parameter names.<br>For example:<br>`[]string{"fields", "columns", "other_alternative_param"}`.<br>The following requests will capture same result `?fields=title,user.name`<br>or `?columns=title,user.name`<br>or `?other_alternative_param=title,user.name`
CacheAdapter       | `*gocache.AdapterInterface` | `nil` | the cache adapter, see more about [cache config](#speed-up-response-with-cache).
ErrorEnabled       | `bool` | `false` | Show error message in pagination result.

## Override results

You can override result with custom function.  

```go
// var db = *gorm.DB
// var httpRequest ... net/http or fasthttp instance
// Example override function
override := func(article *Article) {
    if article.UserID > 0 {
        article.Title = fmt.Sprintf(
            "%s written by %s", article.Title, article.User.Name)
    }
}

var articles []Article
stmt := db.Joins("User").Model(&Article{})

pg := paginate.New()
page := pg.With(stmt).Request(httpRequest).Response(&articles)
for index := range articles {
    override(&articles[index])
}

log.Println(page.Items)

```

## Field selector
To implement a custom field selector, struct properties must have a json tag with omitempty.

```go
// real gorm model
type User {
    gorm.Model
    Name string `json:"name"`
    Age  int64  `json:"age"`
}

// fake gorm model
type UserNullable {
    ID        *string    `json:"id,omitempty"`
    CreatedAt *time.Time `json:"created_at,omitempty"`
    UpdatedAt *time.Time `json:"updated_at,omitempty"`
    Name      *string    `json:"name,omitempty"`
    Age       *int64     `json:"age,omitempty"`
}
```

```go
// usage
nameAndIDOnly := []string{"name","id"}
stmt := db.Model(&User{})

page := pg.With(stmt).
   Request(req).
   Fields(nameAndIDOnly).
   Response([]&UserNullable{})
```

```javascript
// response
{
    "items": [
        {
            "id": 1,
            "name": "John"
        }
    ],
    ...
}
```
## Dynamic field selector
If the request contains query parameter `fields` (eg: `?fieilds=name,id`), then the response will show only `name` and `id`. To activate this feature, please set `FieldSelectorEnabled` to `true`.
```go
config := paginate.Config{
    FieldSelectorEnabled: true,
}

pg := paginate.New(config)
```

## Speed up response with cache
You can speed up results without looking database directly with cache adapter. See more about [cache adapter](https://github.com/morkid/gocache).

### In memory cache
in memory cache is not recommended for production environment:
```go
import (
    "github.com/morkid/gocache"
    ...
)

func main() {
    ...
    adapterConfig := gocache.InMemoryCacheConfig{
        ExpiresIn: 1 * time.Hour,
    }
    pg := paginate.New(&paginate.Config{
        CacheAdapter: gocache.NewInMemoryCache(adapterConfig),
    })

    page := pg.With(stmt).
               Request(req).
               Cache("article"). // set cache name
               Response(&[]Article{})
    ...
}
```

### Disk cache
Disk cache will create a file for every single request. You can use disk cache if you don't care about inode.
```go
import (
    "github.com/morkid/gocache"
    ...
)

func main() {
    adapterConfig := gocache.DiskCacheConfig{
        Directory: "/writable/path/to/my-cache-dir",
        ExpiresIn: 1 * time.Hour,
    }
    pg := paginate.New(&paginate.Config{
        CacheAdapter: gocache.NewDiskCache(adapterConfig),
    })

    page := pg.With(stmt).
               Request(req).
               Cache("article"). // set cache name
               Response(&[]Article{})
    ...
}
```

### Redis cache
Redis cache require [redis client](https://github.com/go-redis/redis) for golang.
```go
import (
    cache "github.com/morkid/gocache-redis/v8"
    "github.com/go-redis/redis/v8"
    ...
)

func main() {
    client := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,
    })

    adapterConfig := cache.RedisCacheConfig{
        Client:    client,
        ExpiresIn: 1 * time.Hour,
    }
    pg := paginate.New(&paginate.Config{
        CacheAdapter: cache.NewRedisCache(adapterConfig),
    })

    page := pg.With(stmt).
               Request(req).
               Cache("article").
               Response(&[]Article{})
    ...
}
```
> if your code already adopts another redis client, you can implement the [redis adapter](https://github.com/morkid/gocache-redis) according to its version. See more about [redis adapter](https://github.com/morkid/gocache-redis).

### Elasticsearch cache
Elasticsearch cache require official [elasticsearch client](https://github.com/elastic/go-elasticsearch) for golang.
```go
import (
    cache "github.com/morkid/gocache-elasticsearch/v7"
    "github.com/elastic/go-elasticsearch/v7"
    ...
)

func main() {
    config := elasticsearch.Config{
        Addresses: []string{
            "http://localhost:9200",
        },
    }
    es, err := elasticsearch.NewClient(config)
    if nil != err {
        panic(err)
    }

    adapterConfig := cache.ElasticCacheConfig{
        Client:    es,
        Index:     "exampleproject",
        ExpiresIn: 1 * time.Hour,
    }
    pg := paginate.New(&paginate.Config{
        CacheAdapter: cache.NewElasticCache(adapterConfig),
    })

    page := pg.With(stmt).
               Request(req).
               Cache("article").
               Response(&[]Article{})
    ...
}
```
> if your code already adopts another elasticsearch client, you can implement the [elasticsearch adapter](https://github.com/morkid/gocache-elasticsearch) according to its version. See more about [elasticsearch adapter](https://github.com/morkid/gocache-elasticsearch).

### Custom cache
Create your own cache adapter by implementing [gocache AdapterInterface](https://github.com/morkid/gocache/blob/master/gocache.go). See more about [cache adapter](https://github.com/morkid/gocache).
```go
// AdapterInterface interface
type AdapterInterface interface {
    // Set cache with key
    Set(key string, value string) error
    // Get cache by key
    Get(key string) (string, error)
    // IsValid check if cache is valid
    IsValid(key string) bool
    // Clear clear cache by key
    Clear(key string) error
    // ClearPrefix clear cache by key prefix
    ClearPrefix(keyPrefix string) error
    // Clear all cache
    ClearAll() error
}
```

### Clean up cache
Clear cache by cache name
```go
pg.ClearCache("article")
```
Clear multiple cache
```go
pg.ClearCache("cache1", "cache2", "cache3")
```

Clear all cache
```go
pg.ClearAllCache()
```


## Limitations

Paginate doesn't support has many relationship. You can make API with separated endpoints for parent and child:
```javascript
GET /users

{
    "items": [
        {
            "id": 1,
            "name": "john",
            "age": 20,
            "addresses": [...] // doesn't support
        }
    ],
    ...
}
```

Best practice:

```javascript
GET /users
{
    "items": [
        {
            "id": 1,
            "name": "john",
            "age": 20
        }
    ],
    ...
}

GET /users/1/addresses
{
    "items": [
        {
            "id": 1,
            "name": "home",
            "street": "home street"
            "user": {
                "id": 1,
                "name": "john",
                "age": 20
            }
        }
    ],
    ...
}
```

Paginate doesn't support for customized json or table field name.  
Make sure your struct properties have same name with gorm column and json property before you expose them.  

Example bad configuration:  

```go

type User struct {
    gorm.Model
    UserName       string `gorm:"column:nickname" json:"name"`
    UserAddress    string `gorm:"column:user_address" json:"address"`
}

// request: GET /path/to/endpoint?sort=-name,address
// response: "items": [] with sql error (column name not found)
```

Best practice:
```go
type User struct {
    gorm.Model
    Name       string `gorm:"column:name" json:"name"`
    Address    string `gorm:"column:address" json:"address"`
}

```

## License

Published under the [MIT License](https://github.com/mahmut/paginate/blob/master/LICENSE).
