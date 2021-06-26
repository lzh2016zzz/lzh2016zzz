---
tags: 
   - 技术
   - elasticsearch
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/cover4.jpg
---

Elasticsearch有一个重要特性: `动态映射`( Dynamic Mapping )

## 字段自动检测

当插入的文档中包含新的字段时,Elasticsearch会将该字段动态添加到文档或文档内的内嵌对象中.ES会自动检测它可能满足的类型，然后创建对应的映射:

| JSON数据                  | ES中的数据类型                                               |
| ------------------------- | ------------------------------------------------------------ |
| **`null`**                | **不会添加字段**                                             |
| **`true` or `false`**     | **boolean**                                                  |
| **floating point number** | **double**                                                   |
| **integer**               | **long**                                                     |
| **object**                | **object**                                                   |
| **array**                 | **依赖于第一个非null得值**                                   |
| **string**                | **如果通过了日期检测，则为date** **如果通过了数字检测，则为Number** |

- 在`mapping`中设置`dynamic=false`可以禁用自动新增字段.
- 配置`dynamic=strict`时,es会拒绝包含未知字段的文档.

<!--more-->

## 日期检测 

默认开启.会检查新字符串是否满足日期格式,然后以相应的格式添加一个新的日期字段

默认格式 : 

```
yyyy/MM/dd HH:mm:ss Z||yyyy/MM/dd Z
```

也可以通过配置修改日期的格式:
```json
"begintime": {
    "type": "date",
    "format": "yyyy-MM-dd'T'HH:mm:ssZ || yyyy-MM-dd HH:mm:ss || yyyy-MM-dd || yyyy-MM-dd'T'HH:mm:ss.SSSZ"
}
```

## 数字自动检测

默认是关闭的.需要手动打开:

```
PUT my_index
{"mappings":{"my_type":{"numeric_detection":true}}}
```

当执行自动新增字段时,如果符合float类型,则会创建为float映射.

