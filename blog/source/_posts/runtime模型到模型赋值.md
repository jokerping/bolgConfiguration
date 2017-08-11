---
title: runtime模型到模型赋值
categories: iOS
tags: runtime

---


需求：两个同样的界面，同样的功能，只因为是不一个入口，接口不一样。接口不一样就算了，接口下来的JSON的key都不一样。我又懒的手动赋值。后来发现这样一个方法

<!--more--> 

```objectivec
-(talkuser*)changeModle:(id)model

{

unsigned int count;

objc_property_t *properties=class_copyPropertyList([model class], &count);//获取传进来模型的Property列表

talkuser *backModel=[[talkuser alloc]init];//创建目标模型

for(int i =0; i < count; i++) {

objc_property_t property = properties[i];//从Property列表中取Property属性

//获取属性的名称C语言字符串

const char *cName =property_getName(property);

//转换为Objective C字符串

NSString *name = [NSString stringWithCString:cName encoding:NSUTF8StringEncoding];

id value= [model valueForKey:name];

[backModel setValue:value forKey:name];//赋值

}
free(propertys);//用完释放掉列表,按住option+左键看class_copyPropertyList方法会提示
returnbackModel;

}
```