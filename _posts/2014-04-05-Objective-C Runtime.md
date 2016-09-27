SEL和IMP

SEL是一个结构体，用来指定一个OC方法，基本上可以看成一个C字符串

typedef struct objc\_selector \*SEL; 

IMP是一个函数指针

typedef id (\*IMP)(id self,SEL \_cmd,...); 

Method是一个不透明的类型，用来表示一个方法在类里面的定义

typedef struct objc\_method \*Method; 

oc发消息-\>objc\_msgSend()-\>class cache-\>class dispatch table-\>ResolveInstanceMethod:-\>forwardingTargetForSelector:-\>forwardInvocation: 

+load vs. +initialize
---------------------

**Swizzling should always be done in `+load`.**

There are two methods that are automatically invoked by the Objective-C runtime for each class.`+load` is sent when the class is initially loaded, while `+initialize` is called just before the application calls its first method on that class or an instance of that class. Both are optional, and are executed only if the method is implemented.

Because method swizzling affects global state, it is important to minimize the possibility of race conditions. `+load` is guaranteed to be loaded during class initialization, which provides a modicum of consistency for changing system-wide behavior. By contrast, `+initialize` provides no such guarantee of when it will be executed—in fact, it may *never* be called, if that class is never messaged directly by the app.