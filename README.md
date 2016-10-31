# CCPowerLabel
基于响应式编程和va_list不定参原理，简化创建UILabel各种属性的代码！

函数名字起的不是太严谨 - - 

导入
---
```objective-c
    #import "UILabel+Category.h"
```

1.不定参va_list传递
---
```objective-c
//..case 1,参数传递个数是不确定的，也就是说你想传几个就传几个,像NSTextAlignmentCenter和@“qwe”并不依赖于函数传递，可以直接作为参数
    
    [label makeAttributes:@"qwe",
           MakeTextAlignment(NSTextAlignmentCenter),
           MakeBackgroundColor([UIColor redColor]),
           MakeRect(0,0,100,100),
           MakeTextColor([UIColor blueColor]),
           MakeTextFont([UIFont systemFontOfSize:20.f]),
           AddToSuperView(self.view),
           nil];

```

2.类似Masonry的响应式编程方式
---
```objective-c
//..case 2

    label.CBackgroundColor([UIColor redColor])
         .CText(@"asdad")
         .CTextColor([UIColor whiteColor])
         .CTextAlignment(1)
         .CFontSize(18.f)
         .CRect(CGRectMake(0, 0, 100, 100))
         .AddToSuperView(self.view);

```
其它控件原理类似，不赘述
===
