# SQUARE

## 功能

计算参数的平方。

## 语法

```sql
SQUARE(arg)
```

### 参数说明

`arg`：需要计算平方的参数，支持的数据类型为数值类型，实现时会将该参数转换为 DOUBLE 类型计算。

## 返回值说明

返回值的数据类型为 DOUBLE。

## 注意事项

参数为非数值类型时返回值为NULL。

## 示例

示例一：计算数值类型的平方数值。

```Plain Text
mysql>  select square(11);
+------------+
| square(11) |
+------------+
|        121 |
+------------+
```

示例二：参数类型为非数值类型时返回NULL。

```Plain Text
mysql>  select square('2021-01-01');
+----------------------+
| square('2021-01-01') |
+----------------------+
|                 NULL |
+----------------------+
```
