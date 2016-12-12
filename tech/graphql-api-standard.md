# IBBD GraphQL API 规范文档

- 官网：http://graphql.org/
- API IDE：https://github.com/graphql/graphiql
- PHP版实现：https://github.com/Folkloreatelier/laravel-graphql

## 与RESTful差异比较大的点

GraphQL是API的UI。

- 类型系统（服务器端）描述有什么数据，客户端决定获取什么数据。
- 不需要版本号的概念，接口可以方便的实现前后的版本兼容。对不需要的参数，可以标注为`deprecated`

## 权限

http头信息：

```
Authorization: token_value_of_successful_login
```

权限验证应该在中间件处理完成，尽量避免在业务逻辑中掺杂着权限校验的逻辑。

权限通常分为几个层面：

- 接口层面的权限: 有些接只有管理员才有权限访问。
- 记录层面的权限: 例如用户只能访问自己账户相关的记录，而管理员可以访问所有记录。

经过权限的中间件检验时，能否注入参数？例如统一注入`loginId`参数。

注入参数的方式如下（php实现）：

- 通过配置文件，指定controller为自己实现
- 在自己实现的controller里，就可以注入了
- 权限判断也要实现在自己实现的controller里
- 注入的参数需要先在每个类型的args参数里注入`loginId`等参数（注意在控制器需要避免该参数从接口传入）。 具体做法类似下面的`offset`, `limit`等参数。

具体实现参照：https://github.com/Folkloreatelier/laravel-graphql/blob/master/src/Folklore/GraphQL/GraphQLController.php

## 数据交换样式

样例如下

```
curl -g 'http://host/graphql?vars={"id":1}' -d '
{
    query getUserList($id:Int){
        users(id:$id){
            id,
            name,
            email,
            friends{
                id,
                name
            }
        }
    }
}
'
```

参数分为两部分：

### GET参数

参数名`vars`，json结构，其中公共的参数名如下：

参数    | 类型   | 是否允许不传 | 默认值   | 说明
---     | -----  | -------      | ------   | -------
offset  | uint   | 是           | 0        | 获取列表时
limit   | uint   | 是           | 0        | 获取列表时，0表示不限
sort    | string | 是           | 空字符串 | 获取列表时，可以同时按多个字段排序，格式如`field1,-field2`，`-`表示倒序，对应sql：`field1 ASC, field2 DESC`
groupby | string | 是           | 空字符串 | 获取列表时，可以同时按多个字段groupby，如`field1,field2`，对应sql：`GROUP BY field1, field2`

这些参数采取统一注入到args的定义里，避免每次都需要重新定义：

```php
// 原来的参数定义
public function args()
{
    return [
        'id' => ['name' => 'id', 'type' => Type::string()],
        'email' => ['name' => 'email', 'type' => Type::string()]
    ];
}

// 可以定义一个参数类
class ControllerArgs {
    public static function get(array $args, $default_offset=0, $default_limit=0, $use_sort=false, $use_groupby=false) 
    {
        $args["offset"] = ['name' => 'offset', 'type' => Type::int()];
        $args["limit"] = ['name' => 'offset', 'type' => Type::int()];

        // sort, groupby参数类似
        return $args;
    }
}

// 这里需要修改为
public function args()
{
    $resArgs = [
        'id' => ['name' => 'id', 'type' => Type::string()],
        'email' => ['name' => 'email', 'type' => Type::string()]
    ];
    return ControllerArgs::get($resArgs)
}
```

另外：`可以统一注入loginId`等参数，并且避免外部传入该值。

### POST参数

以文本的方式post，参数以变量的形式通过`vars`传递，应该避免在语句中写入了参数。这样语句就可以统一写在配置文件里。

## 避免N+1查询问题

javascript版本官方有`dataloader`的实现，但是没有其他语言，所以在使用的时候，还得注意。例如sql语句：

```sql
SELECT b.name, u.name AS userName FROM book b
LEFT JOIN user u ON(b.userId = u.id)
WHERE 1
LIMIT 10
```

这是非常常见的查询，但是在graphql中可能就会产生N+1次查询，如果我们定义类型系统如下：

```go
type User {
    id: uint
    name: string
}

type Book {
    id: uint
    name: string
    user: User
}
```

这是查询语句为：

```
books(limit:10){
    name
    user{ name }
}
```

这时，如果实现没有做特殊的处理，就会产生N+1次查询，因为对于每个Book都会对应的查询User一次！

在php中的解决方式见：https://github.com/Folkloreatelier/laravel-graphql/blob/master/docs/advanced.md#eager-loading-relationships （关于with的用法，具体看这里：http://laravelacademy.org/post/140.html#ipt_kb_toc_140_9 ）

