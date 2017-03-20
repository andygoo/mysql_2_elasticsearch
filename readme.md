# mysql_2_elasticsearch


可定制的 elasticsearch 数据导入工具

##版本更新说明：

|releases|logs|
|--------|----|
|v1.0.8|修复 BUG : YYYY.MM.DD 格式日期的兼容问题|
|v1.0.7|对不规范的时间日期字符串的宽容性优化；代码稳健性优化；导库任务结束后自动断开 Mysql 连接|
|v1.0.6|新增功能：支持使用 SQL 语句，将查询结果集导入 ES|
|v1.0.5|新增功能：配置项 ```exception_handler[field_name].writeAs``` 支持传递回调函数|

##主要功能
1. 完全使用 JS 实现数据从 MySQL 到 elasticsearch 的迁移；
2. 可一次性导入多张 MySQL 数据表和数据集合；
2. 可自定义的数据迁移的规则（数据表/字段关系、字段过滤、使用正则进行异常处理）；
3. 可自定义的异步分片导入方式，数据导入效率更高。

##一键安装
```
npm install mysql_2_elasticsearch
```

##快速开始（简单用例）
```
var esMysqlRiver = require('mysql_2_elasticsearch');

var river_config = {
  mysql: {
    host: '127.0.0.1',
    user: 'root',
    password: 'root',
    database: 'users',
    port: 3306
  },
  elasticsearch: {
    host_config: {               // es客户端的配置参数
      host: 'localhost:9200',
      // log: 'trace'
    },
    index: 'myIndex'
  },
  riverMap: {
    'users => users': {}         // 将数据表 users 导入到 es 类型: /myIndex/users
  }
};


/*
** 以下代码内容：
** 通过 esMysqlRiver 方法进行数据传输，方法的回调参数(一个JSON对象) obj 包含此次数据传输的结果
** 其中：
** 1. obj.total    => 需要传输的数据表数量
** 2. obj.success  => 传输成功的数据表数量
** 3. obj.failed   => 传输失败的数据表数量
** 4. obj.result   => 本次数据传输的结论
*/

esMysqlRiver(river_config, function(obj) {
  /* 将传输结果打印到终端 */
  console.log('\n---------------------------------');
  console.log('总传送：' + obj.total + '项');
  console.log('成功：' + obj.success + '项');
  console.log('失败：' + obj.failed + '项');
  if (obj.result == 'success') {
    console.log('\n结论：全部数据传送完成！');
  } else {
    console.log('\n结论：传送未成功...');
  }
  console.log('---------------------------------');
  /* 将传输结果打印到终端 */
});
```

##最佳实现（完整用例）
```
var esMysqlRiver = require('mysql_2_elasticsearch');

/*
** mysql_2_elasticsearch 的相关参数配置(详情见注释)
*/

var river_config = {

  /* [必需] MySQL数据库的相关参数(根据实际情况进行修改) */
  mysql: {
    host: '127.0.0.1',
    user: 'root',
    password: 'root',
    database: 'users',
    port: 3306
  },

  /* [必需] es 相关参数(根据实际情况进行修改) */
  elasticsearch: {
    host_config: {               // [必需] host_config 即 es客户端的配置参数，详细配置参考 es官方文档
      host: 'localhost:9200',
      log: 'trace',
      // Other options...
    },
    index: 'myIndex',            // [必需] es 索引名
    chunkSize: 8000,             // [非必需] 单分片最大数据量，默认为 5000 (条数据)
    timeout: '2m'                // [非必需] 单次分片请求的超时时间，默认为 1m
    //(注意：此 timeout 并非es客户端请求的timeout，后者请在 host_config 中设置)
  },

  /* [必需] 数据传送的规则 */
  riverMap: {
    'users => users': {            // [必需] 'a => b' 表示将 mysql数据库中名为 'a' 的 table 的所有数据 输送到 es中名为 'b' 的 type 中去
      filter_out: [                // [非必需] 需要过滤的字段名，即 filter_out 中的设置的所有字段将不会被导入 elasticsearch 的数据中
        'password',
        'age'
      ],
      exception_handler: {               // [非必需] 异常处理器，使用JS正则表达式处理异常数据，避免 es 入库时由于类型不合法造成数据缺失
        'birthday': [                    // [示例] 对 users 表的 birthday 字段的异常数据进行处理
          {
            match: /NaN/gi,              // [示例] 正则条件(此例匹配字段值为 "NaN" 的情况)
            writeAs: null                // [示例] 将 "NaN" 重写为 null
          },
          {
            match: /(\d{4})年/gi,        // [示例] 正则表达式(此例匹配字段值为形如 "2016年" 的情况)
            writeAs: '$1.1'              // [示例] 将 "2016年" 样式的数据重写为 "2016.1" 样式的数据
          },
          {
            match: /(\d{4})年/gi,        // [示例] 正则表达式(此例匹配字段值为形如 "2016年" 的情况)
            writeAs: function (word){    // [示例] 用回调函数处理数据
              return parseInt(word) + '.1';
            }
          }
        ]
      }
    },
    'favors => favors': {
      // [非必需] MySQL 查询语句，将查询结果导入名为 favors 的es类型中，若不配置此项，默认将数据表 favors 的数据导入 es 中
      SQL: 'SELECT favors.id,users.name,favors.favor FROM favors,users WHERE favors.user_id = users.id',
      filter_out: [ ... ],
      exception_handler: {
        ...
      }
    },
    // Other fields' options...
  }

};


/*
** 将传输结果打印到终端
*/

esMysqlRiver(river_config, function(obj) {
  console.log('\n---------------------------------');
  console.log('总传送：' + obj.total + '项');
  console.log('成功：' + obj.success + '项');
  console.log('失败：' + obj.failed + '项');
  if (obj.result == 'success') {
    console.log('\n结论：全部数据传送完成！');
  } else {
    console.log('\n结论：传送未成功...');
  }
  console.log('---------------------------------');
});
```

##注意事项及参考
1. elasticsearch 数据导入前请事先导入或配置好 index/type 的数据结构；
2. ```host_config``` 参数设置详见 [es官方文档](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/configuration.html)；
3. mysql 表的自增 id 自动替换为 ```表名+_id``` 的格式，如：```users_id```；
4. 如因数据格式或内容问题导致的元数据导入缺失或失败，可通过设置 exception_handler 参数进行包容。

##github 项目地址
https://github.com/parksben/mysql_2_elasticsearch
