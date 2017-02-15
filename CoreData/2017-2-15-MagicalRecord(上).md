---  
layout: post  
title: "MagicalRecord（上）"  
description: ""  
category: ios  
tags: [CoreData]  
author: 饭小团  
---   


#MagicalRecord（上）  

安装：
	
	pod "MagicalRecord"  
	
默认情况下，所有的MagicalRecord框架都带有MR_前缀，这是apple要求的命名，以防命名冲突。  
不过在这里我就这样了，想要去掉他请访问[这里](https://github.com/magicalpanda/MagicalRecord/blob/master/Docs/Installing-MagicalRecord.md)  

#Getting-Started  

	// Objective-C
	#import <MagicalRecord/MagicalRecord.h>  
	
	
在- applicationDidFinishLaunching: withOptions:里使用其中一个函数：  

	+ (void)setupCoreDataStack;
	+ (void)setupAutoMigratingCoreDataStack;
	+ (void)setupCoreDataStackWithInMemoryStore;
	+ (void)setupCoreDataStackWithStoreNamed:(NSString *)storeName;
	+ (void)setupCoreDataStackWithAutoMigratingSqliteStoreNamed:(NSString *)storeName;
	+ (void)setupCoreDataStackWithStoreAtURL:(NSURL *)storeURL;
	+ (void)setupCoreDataStackWithAutoMigratingSqliteStoreAtURL:(NSURL *)storeURL;  

每次调用属于coredata stack的实例，都会提供getter和setter方法。  

当在DEBUG环境下，使用默认的SQLite数据库，改变你的model而不创建新的model版本会引起 MagicalRecord 自动删除旧的store并且创建新的。这将消耗大量的时间，只要你改变了data model，每次都会重装你的app。  **确保关闭DEBUG模式：在不告诉用户发生了什么的情况下删除数据是个糟糕的体验**  

在你app退出前，调用 +cleanUp  

	[MagicalRecord cleanUp];  
	
这是MagicalRecord的整理，删除我们自定义的错误处理并设置由MagicalRecord生成的CoreData堆栈为nil。  

#Working with Managed Object Contexts  
###Creating New Contexts  

* + [NSManagedObjectContext MR_newContext] 设置默认环境作为父环境，是一个并发类型 NSPrivateQueueConcurrencyType  
* + [NSManagedObjectContext MR_newMainQueueContext] 属于一个并发类型NSMainQueueConcurrencyType。主线程并发  
* + [NSManagedObjectContext MR_newPrivateQueueContext] 私有并发类型NSPrivateQueueConcurrencyType 
* + [NSManagedObjectContext MR_newContextWithParent:…] 允许你指定特殊的父context。	属于NSPrivateQueueConcurrencyType类型  
* + [NSManagedObjectContext MR_newContextWithStoreCoordinator:…]运行你指定持久话存储协调器（persistent store coordinator）为新的context。为NSPrivateQueueConcurrencyType类型  

#The Default Context  
在使用CoreData时，我们最长打交道的两个对象是：NSManagedObject 和NSManagedObjectContext。  
MagicalRecord 提供了一个简单的类函数用于检索默认的NSManagedObjectContext，他可以贯穿你的全局app里使用。这个context操作在主线程，是一个足够简单，单一线程的app。  

要访问默认context：  

	NSManagedObjectContext *defaultContext = [NSManagedObjectContext MR_defaultContext];  
	
这个context会被MagicalRecord的任何函数使用，但不会提供特殊的 managed object context 参数。  

如果你要创建一个新的 managed object context 用于非主线程：  

	NSManagedObjectContext *myNewContext = [NSManagedObjectContext MR_newContext];  
	
这会创建一个新的ManagedObjectContext，他的对象model和persistent store 是和默认的context是一样的，只不过会对其他线程安全。他会自动的设置默认context作为父context。  

你也可以设置自己的myNewContext： 

	[NSManagedObjectContext MR_setDefaultContext:myNewContext];  
	
**注意：这里强烈建议使用创建DefaultContext，并且设置在主线程上使用managed object context。并发类型为NSMainQueueConcurrencyType**


#Performing Work on Background Threads  
MagicalRecord 提供在后台线程设置和工作。后台存储操作使用block，灵感来源于UIView animation block，但有些不同点：  

* 在block里你对实体的改变将不会在主线程进行
* 只有一个NSManagedObjectContext在这个block提供给你  

		Person *person = ...;
		[MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext){

		  Person *localPerson = [person MR_inContext:localContext];
		  localPerson.firstName = @"John";
		  localPerson.lastName = @"Appleseed";
		}];   
		
这个函数的bock提供给你在适合的context下执行你自己的操作，所有的改变都执行在其他线程。  
	
	Person *person = ...;

	[MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext){

	  Person *localPerson = [person MR_inContext:localContext];
	  localPerson.firstName = @"John";
	  localPerson.lastName = @"Appleseed";

	} completion:^(BOOL success, NSError *error) {

	  self.everyoneInTheDepartment = [Person findAll];

	}];  
	
completion的block在主线程，对于UI更新是安全的。  


#Creating Entities  
在默认context下创建和插入一个新的Entity实例  
	
	Person *myPerson = [Person MR_createEntity];  
	
在指定context下：  

	Person *myPerson = [Person MR_createEntityInContext:otherContext];  
	
#Deleting Entities  
在默认环境下删除单一的实体 

	[myPerson MR_deleteEntity];  
	
在指定环境下删除实体： 
	
	[myPerson MR_deleteEntityInContext:otherContext]; 
	
删除所有实体 
	
	[Person MR_truncateAll]; 
	
在指定环境下：  

	[Person MR_truncateAllInContext:otherContext];

#Fetching Entities  

####Basic Finding  

返回一个person类的数组，包含所有Person 

	NSArray *people = [Person MR_findAll];  
	
返回的数组是根据LastName排序的：

	NSArray *peopleSorted = [Person MR_findAllSortedBy:@"LastName"
                                         ascending:YES];  

返回的实体是多种排序的：
	
	NSArray *peopleSorted = [Person MR_findAllSortedBy:@"LastName,FirstName"
                                         ascending:YES];  
                                         

还可以混合：

	NSArray *peopleSorted = [Person MR_findAllSortedBy:@"LastName:NO,FirstName"
                                         ascending:YES]; 
	
	整体是升序，LastName是降序
                                         

	// OR

	NSArray *peopleSorted = [Person MR_findAllSortedBy:@"LastName,FirstName:YES"
                                         ascending:NO];
	整体是升序，FirstName是升序  
	
从数据库里找到一个单一的对象：  
	
	Person *person = [Person MR_findFirstByAttribute:@"FirstName"
                                       withValue:@"Forrest"];  
	在FirstName里找Forrest这个对象实体   
	
####Advanced Finding  

使用谓语predicate，更精确一些  
	
	NSPredicate *peopleFilter = [NSPredicate predicateWithFormat:@"Department IN %@", @[dept1, dept2]];
	NSArray *people = [Person MR_findAllWithPredicate:peopleFilter];
	
####Returning an NSFetchRequest  
	
	NSPredicate *peopleFilter = [NSPredicate predicateWithFormat:@"Department IN %@", departments];
	NSFetchRequest *people = [Person MR_requestAllWithPredicate:peopleFilter];  
	
####Customizing the Request  

	NSPredicate *peopleFilter = [NSPredicate predicateWithFormat:@"Department IN %@", departments];

	NSFetchRequest *peopleRequest = [Person MR_requestAllWithPredicate:peopleFilter];
	[peopleRequest setReturnsDistinctResults:NO];
	[peopleRequest setReturnPropertiesNamed:@[@"FirstName", @"LastName"]];

	NSArray *people = [Person MR_executeFetchRequest:peopleRequest];  
	
####Find the number of entities  

在你的persistent store区域里，找见指定类型的所有实体数量  
	
	NSNumber *count = [Person MR_numberOfEntities];  
	
或者指定谓词或其他过滤条件：  
	
	NSNumber *count = [Person MR_numberOfEntitiesWithPredicate:...];  
	
还有其他可以直接返回int值而不是NSNumber: 
	
	+ (NSUInteger) MR_countOfEntities;
	+ (NSUInteger) MR_countOfEntitiesWithContext:(NSManagedObjectContext *)context;
	+ (NSUInteger) MR_countOfEntitiesWithPredicate:(NSPredicate *)searchFilter;
	+ (NSUInteger) MR_countOfEntitiesWithPredicate:(NSPredicate *)searchFilter
                                     inContext:(NSManagedObjectContext *)context;
	

####Aggregate Operations  
统计操作 

	NSNumber *totalCalories = [CTFoodDiaryEntry MR_aggregateOperation:@"sum:"
                                                      onAttribute:@"calories"
                                                    withPredicate:predicate];

	NSNumber *mostCalories  = [CTFoodDiaryEntry MR_aggregateOperation:@"max:"
                                                      onAttribute:@"calories"
                                                    withPredicate:predicate];

	NSArray *caloriesByMonth = [CTFoodDiaryEntry MR_aggregateOperation:@"sum:"
                                                       onAttribute:@"calories"
                                                     withPredicate:predicate
                                                           groupBy:@"month"];


####Finding entities in a specific context

所有操作都要加上 inContext: 作为参数。
	
	NSArray *peopleFromAnotherContext = [Person MR_findAllInContext:someOtherContext];

	Person *personFromContext = [Person MR_findFirstByAttribute:@"lastName"
   		                                               withValue:@"Gump"
       	                                           inContext:someOtherContext];

	NSUInteger count = [Person MR_numberOfEntitiesWithContext:someOtherContext];  
	






















