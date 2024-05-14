### clickhouse学习笔记

---

- query相关
```sql
-- 1. 展示正在处理的请求列表
show processlist

-- 2. 杀掉正在处理的查询
KILL QUERY WHERE query_id='2-857d-4a57-9ee0-327da5d60a90'
```

- 修改名称
```sql
-- 1. 重命名
RENAME DATABASE|TABLE|DICTIONARY name TO new_name
       
-- 2. 交换2个表的名称
EXCHANGE TABLES [db0.]table_A AND [db1.]table_B
```

- 数组函数
```sql
-- 1.检测输入的数组是否空
empty([x])
-- 2.检测输入的数组是否非空
notEmpty([x])
-- 3.获取数组长度
length([x])
-- 4.返回一个以step作为增量步长的从start到end - 1的整形数字数组
range(start, end, step)
-- 5.合并参数中传递的所有数组
SELECT arrayConcat([1, 2], [3, 4], [5, 6]) AS res
-- 6.根据索引查找元素
arrayElement(arr,n),运算符arr[n]
-- 7.检查’arr’数组是否具有’elem’元素
has(arr,elem)
-- 8.检查一个数组是否是另一个数组的子集
hasAll(set, subset)
-- 9.检查两个数组是否存在交集
hasAny(array1, array2)
-- 10.检查 array2 的所有元素是否以相同的顺序出现在 array1 中。当且仅当 array1 = prefix + array2 + suffix时，该函数将返回 1。
hasSubstr(array1, array2)
-- 11.返回数组中第一个’x’元素的索引（从1开始），如果’x’元素不存在在数组中，则返回0
indexOf(arr,x)
-- 12.返回结果为非零值的数量
arrayCount(func, arr1)
-- 13.返回数组中等于x的元素的个数
countEqual(arr,x)
-- 14.从数组中删除最后一项
arrayPopBack(array)
-- 15.从数组中删除第一项
arrayPopFront(array)
-- 16.添加一个元素到数组的末尾
arrayPushBack(array, single_value)
-- 17.将一个元素添加到数组的开头
arrayPushFront(array, single_value)
--  18.返回一个子数组，包含从指定位置的指定长度的元素
SELECT arraySlice([1, 2, NULL, 4, 5], 2, 3) AS res
-- 19.以升序对arr数组的元素进行排序。如果指定了func函数，则排序顺序由func函数的调用结果决定
SELECT arraySort([1, 3, 3, 0]);
SELECT arraySort((x) -> -x, [1, 2, 3]) as res;
SELECT arraySort((x, y) -> y, ['hello', 'world'], [2, 1]) as res;
-- 20.以降序对arr数组的元素进行排序。如果指定了func函数，则排序顺序由func函数的调用结果决定
SELECT arrayReverseSort([1, 3, 3, 0]);
-- 21.计算数组中不同元素的数量
arrayUniq(arr, …)
-- 22.计算相邻数组元素之间的差异
SELECT arrayDifference([1, 2, 3, 4]);
-- 23.对数组去重
SELECT arrayDistinct([1, 2, 2, 3, 1]);
-- 24.返回所有数组元素的交集
SELECT arrayIntersect([1, 2], [1, 3], [2, 3]);
-- 25.将聚合函数应用于数组元素并返回其结果
SELECT arrayReduce('max', [1, 2, 3]);
-- 26.将嵌套的数组展平
SELECT flatten([[[1]], [[2], [3]]]);
-- 27.从数组中删除连续的重复元素。结果值的顺序由源数组中的顺序决定。
SELECT arrayCompact([1, 1, nan, nan, 2, 3, 3, 3]);
-- 28.将多个数组组合成一个数组。结果数组包含按列出的参数顺序分组为元组的源数组的相应元素
SELECT arrayZip(['a', 'b', 'c'], [5, 2, 1]);
-- 29.将从 func 函数的原始应用中获得的数组返回给 arr 数组中的每个元素
SELECT arrayMap(x -> (x + 2), [1, 2, 3]) as res;
-- 30.返回一个仅包含 arr1 中的元素的数组，其中 func 返回的值不是 0
SELECT arrayFilter(x -> x LIKE '%World%', ['Hello', 'abc World']) AS res
-- 31.从第一个元素到最后一个元素扫描arr1，如果func返回0，则用arr1[i - 1]替换arr1[i]
SELECT arrayFill(x -> not isNull(x), [1, null, 3, 11, 12, null, null, 5, 6, 14, null, null]) AS res
-- 32.将 arr1 拆分为多个数组。当 func 返回 0 以外的值时，数组将在元素的左侧拆分
SELECT arraySplit((x, y) -> y, [1, 2, 3, 4, 5], [1, 0, 0, 1, 0]) AS res
-- 33.如果 arr 中至少有一个元素 func 返回 0 以外的值，则返回1
arrayExists([func, arr1)
-- 34.如果 func 为 arr 中的所有元素返回 0 以外的值，则返回 1
arrayAll([func, arr1)
-- 35.’arrayJoin’函数获取每一行并将他们展开到多行
```

- 条件函数
```sql
-- 1. if
SELECT if(1, plus(2, 2), plus(2, 6))
-- 2. multiIf
multiIf(cond_1, then_1, cond_2, then_2, ..., else)
-- 3. 三元运算
cond ? then : else
```

- 聚合函数
```sql
-- 1.返回指定列中近似最常见值的数组
SELECT topK(3)(AirlineID) AS res
FROM ontime
```

- 查看占用存储空间大小
```sql
-- 1.查看partition
SELECT partition, formatReadableSize(sum(bytes)) AS size
FROM system.parts
WHERE database = 'your_database' AND table = 'access_local'
GROUP BY partition
ORDER BY partition;
 
-- 2.查看列的大小
SELECT name, sum(data_compressed_bytes) AS size
FROM system.columns
WHERE database = 'your_database' AND table = 'access'
GROUP BY name
ORDER BY size DESC;<br>

-- 3.查看表的大小
SELECT table, formatReadableSize(sum(data_compressed_bytes)) AS size
FROM system.parts
WHERE active
GROUP BY table
ORDER BY size DESC;
```

- 删除数据
```sql
-- 1.删除partition
alter table access_local drop partation '20230901'
-- 2.删除列
alter table access_local clear column req_body on partation '20230901'
```