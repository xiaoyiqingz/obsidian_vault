1、当你的数据量过大，而你的索引最初创建的分片数量不足，导致数据入库较慢的情况，此时需要扩大分片的数量，此时可以尝试使用Reindex。

2、当数据的mapping需要修改，但是大量的数据已经导入到索引中了，重新导入数据到新的索引太耗时；但是在ES中，一个字段的mapping在定义并且导入数据之后是不能再修改的，

```json
POST _reindex
{
  "source": {
    "index": "qa_funds_bak"
  },
  "dest": {
    "index": "qa_funds"
  }
}
```