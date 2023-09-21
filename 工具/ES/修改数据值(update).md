```json
POST /index/_update/{id}  //POST方式修改的话，可以针对对应field来修改，比PUT要轻量
{ 
	"doc": { 
		"field": "new_value" 
	} 
}
```

```json
  
PUT /product/_doc/{id} //PUT方式进行修改，这种是把原来对应文档覆盖掉 
{ 
	"id": 10001,
	"productName": "牛肉片(潮汕集锦)",
	"brandName": "潮汕集锦",
	"categoryName": "零食",
	"utime": 1614666414
}
```