##(翻译)Let's Build NSObject  

###Components of a Root Class  
就Objective-C 而言，有一个精确的需求：root class的第一个实例变量必须是isa，他指向object的class。这个isa用于，当分发消息时，一个object是什么class。从一个严格的语言来看，这是必须的~  

NSObject 提供了以下三个功能类别：  
1.Memory management:  标准的内存管理，如retain，release会在NSObject实现。alloc也会在这实现~  

2.Introspection:  NSObject封装了大量的Objective-C runtime 功能，比如:class, respondsToSelector: isKindOfClass:.

3.Default implementations of miscellaneous methods:   

####Instance Variables:  
MAObject有两个实例变量，一个是isa指针，一个是对象引用计数：
	  
	  @implementation MAObject {
        Class isa;
        volatile int32_t retainCount;
    	}  
    	
引用计数来自于OSAtomic.h，确保线程安全。

实际上NSObject 在外部把握引用计数。有个外部全局table用来映射一个对象地址到他的引用计数。这里保存在内存，because the table represents the common reference count of 1 by not having an entry in the table at all.  

####Memory Management：
MAObject第一件事是通过+alloc创建一个实例。  
Subclasses很少复写alloc函数，都依赖于root class 来分配内存。这意味着MAObject除了为自己分配内存，还要为子类分配。可以利用self做到此功能，如果[Subclass alloc]，则self可hold一个指针来指向Subclass。

	+ (id)alloc
	{
		MAObject *obj = calloc(1, class_getInstanceSize(self));
		obj->isa = self;
		obj->retainCount = 1;
		return obj;
	}  


retain 使用OSAtomicIncrement32 增加retain count，返回self  
	
	- (id)retain
	{
        OSAtomicIncrement32(&retainCount);
        return self;
    }  

release会减retain count ，为0时会deallac  

	- (oneway void)release
    {
        uint32_t newCount = OSAtomicDecrement32(&retainCount);
        if(newCount == 0)
            [self dealloc];
    }  
   
autorelease的实现是调用NSAutoreleasePool，把self加进到当前autorelease pool，Autorelease pools 是runtime的一部分，是私有的~，所以我们最好这么做：   

	- (id)autorelease
    {
        [NSAutoreleasePool addObject: self];
        return self;
    }  
retainCount：

	- (NSUInteger)retainCount
    {
        return retainCount;
    }   

dealloc
    
	 - (void)dealloc
    {
        free(self);
    }  
    
NSObject provides a do-nothing init  

	- (id)init
    {
        return self;
    }
    
     + (id)new
    {
        return [[self alloc] init];
    }
    
###Introspection  
反射机制都封装了runtime functions。  

	- (Class)class
    {
        return isa;
    }
    
Technically, this method will fail on tagged pointers. A proper implementation should call object_getClass, which behaves correctly for tagged pointers, and extracts the isa from a normal pointer. 

    - (Class)superclass
    {
        return [[self class] superclass];
    }

	 + (Class)class
    {
        return self;
    }
    
    + (Class)superclass
    {
        return class_getSuperclass(self);
    }

	 - (BOOL)isMemberOfClass: (Class)aClass
    {
        return isa == aClass;
    }
    
    - (BOOL)isKindOfClass: (Class)aClass
    {
        return [isa isSubclassOfClass: aClass];
    }
    
由最后两个函数可知：isMemberOfClass 比 isKindOfClass更严格些，isMemberOfClass只对比当前对象是否为其子类。而isKindOfClass会不断向上遍历，在每个级别和目标class对比。一直到class顶部。isSubclassOfClass 实现原理如下： 
    
    + (BOOL)isSubclassOfClass: (Class)aClass
    {
        for(Class candidate = self; candidate != nil; candidate = [candidate superclass])
            if (candidate == aClass)
                return YES;

        return NO;
    }

isKindOfClass的缺点在于，如果你的类特别深。这回导致有很多循环，而且会有些性能瓶颈。仅这一条原因便尽可能的避免使用它。  

respondsToSelector 是依靠runtime的class_respondsToSelector调用，会在class的函数列表里循环的查找有没有此条目： 

	- (BOOL)respondsToSelector: (SEL)aSelector
    {
        return class_respondsToSelector(isa, aSelector);
    }

instancesRespondToSelector 和上面差不多，只不过把isa改成self： 

	 + (BOOL)instancesRespondToSelector: (SEL)aSelector
    {
        return class_respondsToSelector(self, aSelector);
    }

conformsToProtocol 也是封装runtime，咨询protocol列表。

	  - (BOOL)conformsToProtocol: (Protocol *)aProtocol
    {
        return class_conformsToProtocol(isa, aProtocol);
    }

    + (BOOL)conformsToProtocol: (Protocol *)protocol
    {
        return class_conformsToProtocol(self, protocol);
    }
    
methodForSelector，instanceMethodForSelector:  

	 - (IMP)methodForSelector: (SEL)aSelector
    {
        return class_getMethodImplementation(isa, aSelector);
    }

    + (IMP)instanceMethodForSelector: (SEL)aSelector
    {
        return class_getMethodImplementation(self, aSelector);
    }


	 - (NSMethodSignature *)methodSignatureForSelector: (SEL)aSelector
    {
        return [isa instanceMethodSignatureForSelector: aSelector];
    }

    + (NSMethodSignature *)instanceMethodSignatureForSelector: (SEL)aSelector=
    {
        Method method = class_getInstanceMethod(self, aSelector);
        if(!method)
            return nil;

        const char *types = method_getTypeEncoding(method);
        return [NSMethodSignature signatureWithObjCTypes: types];
    }

	  - (id)performSelector: (SEL)aSelector
    {
        IMP imp = [self methodForSelector: aSelector];
        return ((id (*)(id, SEL))imp)(self, aSelector);
    }

    - (id)performSelector: (SEL)aSelector withObject: (id)object
    {
        IMP imp = [self methodForSelector: aSelector];
        return ((id (*)(id, SEL, id))imp)(self, aSelector, object);
    }

    - (id)performSelector: (SEL)aSelector withObject: (id)object1 withObject: (id)object2
    {
        IMP imp = [self methodForSelector: aSelector];
        return ((id (*)(id, SEL, id, id))imp)(self, aSelector, object1, object2);
    }
    
###Default Implementations  

	   - (BOOL)isEqual: (id)object
    {
        return self == object;
    }

    - (NSUInteger)hash
    {
        return (NSUInteger)self;
    } 
    
     + (NSString *)description
    {
        return [NSString stringWithUTF8String: class_getName(self)];
    }