---
title: 使用RecyclerView实现多type抽象需求
date: 2018-05-18 23:46:33
tags:
---

#### 多Type需求

   产品原型
   </br>
   
   ```
   最近在完成一个类型信息流产品，涉及到很多type类型
   ```
   
   技术支撑
   </br>
   
   ```
   在技术层面上，通过RecyclerView的ViewHolder，抽象业务逻辑，结合不同的type来注册不同的ViewHolder，然后各自ViewHolder处理对应样式的逻辑和业务
   ```
   
#### 抽象业务

主要有两部分，一部分是支持多type本身的Adapter的抽象，一部分是由数据驱动列表的数据解析，同时包含后端的数据构建。

{% asset_img img/mulitiAdapter.png %}

数据原型：
```
{
	"data": {
		"list": [
			{
				"cardType": "cardType_1",
				"description": "",
				"pic": "",
				"title": "",
				"id": "14499",
				"count": "1"
			},
			{
				"cardType": "cardType_2",
				"video": "",
				"videoUrl": "",
				"musicList": [
				],
				"price": "",
				"salePrice": "120"
			},
			{
				"cardType": "cardType_3",
				"advertising": "",
				"advertisingTitle": "",
				"advertisingUrl": [
				],
				"advertisingPrice": ""
			}
		]
	}
}
```

典型的信息流中包含的cardtype样式会有很多种，每种都会相应的对应一种数据，然后把不同的数据生成一个json的列表，每个item是不同的对象和字段，这样我们很难使用json工具来反射这个list，所以需要自己去解析list元素所属卡片的数据格式。


#### Adapter抽象

正常的写法是，我们需要自己去继承Adapter，然后编写一个对应的ViewHolder，然后写逻辑，这样代码耦合度高，往往需要写很多代码，包括处理业务逻辑。

抽象思想：

```
一个列表中包含了不同的类型的卡片，每个卡片对应自己的数据对象，每个卡片都需要绑定自己的数据对象和逻辑，也就是说ViewHolder + Data.class 组成一个个体

```

上图中已经标注出相应的模式。


#### 理想中的使用方式

```
        GridLayoutManager layoutManager = new GridLayoutManager(getContext(), SPAN_COUNT);
        layoutManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
            @Override
            public int getSpanSize(int position) {
                if (items.size() > 0) {
                    Object object = items.get(position);
                    return (object instanceof MusicData ? SPAN_COUNT : 1;
                }
                return 1;
            }
        });

        mRecyclerView.setLayoutManager(layoutManager);

        adapter = new MultiAdapter();
        adapter.register(MusicData.class, new MusicViewBinder());
        adapter.register(VideoData.class, new VideoViewBinder());
        adapter.register(MovVideoData.class, new MovViewBinder());
        adapter.setItems(items);
        mRecyclerView.setAdapter(adapter);
```

数据解析模块：

```

```

   