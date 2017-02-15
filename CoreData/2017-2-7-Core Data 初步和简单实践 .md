---  
layout: post  
title: "Core Data 初步和简单实践"  
description: ""  
category: ios  
tags: [CoreData]  
author: 饭小团  
---   


##Core Data 初步和简单实践 

***首先：甭指望几篇文章，几篇教程就能让你掌握CoreData。这玩意东西不少***  
***其次：你该学习swift了，因为市面上的文章都在转向swift，OC的文章越来越少了***  

##简介
Core Data 是 Apple 为 iOS，OS X，watchOS 和 tvOS 设计的对象图管理 (object graph management) 和数据持久化框架。如果你的 app 需要存储结构化的数据，Core Data 是一个显而易见的方案：它是现成的，Apple 仍然在积极地维护它，而且它已经存在超过 10 年了。Core Data 是一个成熟，经过实践检验的代码库。

###CoreData是什么  
1. Core Data 是个框架，它使得开发者可以把**数据当做对象**来操作，而不必在乎数据在磁盘中的存储方式。  
2. Core Data所提供的**数据对象**叫托管对象（managed object），而CoreData本身则位于你的应用程序和持久化存储区（persistent store）之间。  
3. Core Data 使用**托管对象模型**来把数据从托管对象映射到持久化存储区中，开发者可以通过对象图（object graph）来配置应用程序的数据结构。  
4. 托管对象持有一份对持久化存储区里的相关数据的拷贝。如果用数据库，那么托管对象可以当做**数据表的一行**。如果用XML持久化，那么托管对象可认为是某个**数据元素**。
5. 所有托管对象都必须位于托管对象上下文里（managed object context）,而托管对象上下文位于RAM里。我的理解，托管对象上下文是个存储的操作环境，数据先存储在内存里。save之后才会写入磁盘。由于读内存比读磁盘快得多，所以读写数据大多会在内存中操作。只有当save之后数据才会进入磁盘。这样适时的操作内存和磁盘是提高性能的手段之一。  

####持久化存储协调器  
持久化存储协调器（persistent store coordinator）包含一份持久化存储区，存储区里又含有数据表里的若干行数据。  

####托管对象模型  
位于持久化存储协调器和托管对象上下文之间。顾名思义，托管对象模型是描述数据结构的模型或图示（graphical representation）。它与数据库模式相似，有时也叫对象

####托管对象上下文  
托管对象上下文负责管理对象的生命周期，并且还提供许多强大功能，如faulting，变更追踪，验证等。  
faulting就是用户从磁盘取数据时，只会把需要的一部分数据取出来。而userdefault每次都会把所有数据读到内存，操作完，再写回去。  
变更就是支持撤销和重做。  
验证机制用来确保托管对象的规则，比如某个实体的属性的限制的最大最小值。  

###实体  
托管对象模型由一系列实体描述对象构成，称为实体。我的理解是：实体类似与数据库的数据表。  
在设计数据表时，你需要：  

 * 数据表名称
 * 配置字段和字段的数据类型  
 
而设计实体时，你需要：  

 * 配置实体名称  
 * 配置属性和属性的数据类型  
 * 根据实体来配置NSManagedObject的子类（可选 ）  

 用数据库的术语来说，托管对象的实例类似于数据库中某张数据表里的一行（row）。  
 
###属性  
属性(attribute)是实体的特征(property),可以认为属性就是数据表里的列。  

#####可变类型  
单独说一下这个：可变（Transformable）数据类型适合把oc对象放在这个属性里。这个属性类型比较灵活，可以存放任意类的实例。如：UIColor

###属性的各种设置选项  
  * Transient  
  * Optional  
  * Indexed  
  * Validatoin  
  * Reg.Ex  
  * Default  
  * Allows External Storage  
  * Index in Spotlight  
  * Store in External Record File  
  * Name  
 

##MagicalRecord Tutorial for iOS

MagicalRecord 是针对CoreData的封装，并且提供了很多强大而且易用的API.本篇文章打算用MagicalRecord做一个简单的啤酒打分小程序，而前提是你已经有了对CoreData的基本认识。  

程序需求：  
1.增加一个你喜欢的啤酒  
2.对它打分  
3.对它评价  
4.拍个啤酒的照片  

[这里](https://koenig-media.raywenderlich.com/uploads/2014/01/BeerTracker-Starter.zip)是这个项目的初始地址。下载下来先跑一下。
![](https://koenig-media.raywenderlich.com/uploads/2013/12/FirstRun.png)  


####Exploring MagicalRecord  
看一下NSManagedObjectModel+MagicalRecord.h里面的代码，发现有很多MR_大头的函数名。如果你不想在很多地方看见这两个令人疑惑的字母。可以在BeerTracker-Prefix.pch里添加这两行代码：  

	#define MR_SHORTHAND
	#import “CoreData+MagicalRecord.h” 
	
在MagicalRecord+ShorthandSupport.m里面可以看见他时如何动态替换这个函数的。    

####Beer Model  
command + N 后找见CoreData，选择Data Model。命名为BeerModel.xcdatamodeld。  
添加Entity,命名为Beer:
加个属性name: type: String  

再添加Entity,命名为BeerDetails:  
加个属性: image  Type: String
加个属性: note   Type: String
加个属性: rating Type: Integer 16  

创建relationship，由于Beer实体需要知道哪些BeerDetails属于他。在BeerDetails实体里，创建一个新的relationship，name是beer，destination是Beer。  
选择Beer 实体. 创建 relationship 命名为 beerDetails, destination 是 BeerDetails, inverse 是 Beer  

新的xcode和教程不一样，这里只需要编译一下即可生成Beer和BeerDetails类。不过这俩类不显示左侧的导航条里。  

现在已经创建好data对象类，可以初始化Core Data stack.   
Open AppDelegate.m, and in application:didFinishLaunchingWithOptions 里 

	// Setup CoreData with MagicalRecord
	// Step 1. Setup Core Data Stack with Magical Record
	// Step 2. Relax. Why not have a beer? Surely all this talk of beer is making you thirsty…
	[MagicalRecord setupCoreDataStackWithStoreNamed:@"BeerModel"];

####Brewing a Beer (entity)  
在BeerViewController.h里  

	@class Beer;
	@property (nonatomic, strong) Beer *beer;

在BeerViewController.m里  
	
	#import "Beer.h"
	#import "BeerDetails.h"
	
	- (void)viewDidLoad {
    // 1. If there is no beer, create new Beer
    //假如你没有在model里创建它，那就用代码创建
    if (!self.beer) {
        self.beer = [Beer createEntity];
    }
    // 2. If there are no beer details, create new BeerDetails
    if (!self.beer.beerDetails) {
        self.beer.beerDetails = [BeerDetails createEntity];
    }
    // View setup
    // 3. Set the title, name, note field and rating of the beer
    self.title = self.beer.name ? self.beer.name : @"New Beer";
    self.beerNameField.text = self.beer.name;
    self.beerNotesView.text = self.beer.beerDetails.note;
    self.ratingControl.rating = [self.beer.beerDetails.rating integerValue];
    [self.cellOne addSubview:self.ratingControl];
 
    // 4. If there is an image path in the details, show it.
    if ([self.beer.beerDetails.image length] > 0) {
        // Image setup
        NSData *imgData = [NSData dataWithContentsOfFile:[NSHomeDirectory() stringByAppendingPathComponent:self.beer.beerDetails.image]];
         [self setImageForBeer:[UIImage imageWithData:imgData]];
     }
	}  
	
在BeerViewController里，你需要给Beer赋值，把输入的数据保存起来  
	
	- (void)textFieldDidEndEditing:(UITextField *)textField {
		if ([textField.text length] > 0) {
	     	self.title = textField.text;
          self.beer.name = textField.text;//存入实体
       }
	}
	
	- (void)textViewDidEndEditing:(UITextView *)textView {
    	[textView resignFirstResponder];
	    if ([textView.text length] > 0) {
      	 	 self.beer.beerDetails.note = textView.text;//存入实体
	   	 }
	}  
	
	- (void)updateRating {
		self.beer.beerDetails.rating = @(self.ratingControl.rating); //存入实体
	}
	
还要把获得的图片存起来  

	- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary *)info {
    // 1. Grab image and save to disk
    UIImage *image = info[UIImagePickerControllerOriginalImage];	
    // 2. Remove old image if present
    if (self.beer.beerDetails.image) {
        [ImageSaver deleteImageAtPath:self.beer.beerDetails.image];
    }
    // 3. Save the image
    if ([ImageSaver saveImageToDisk:image andToBeer:self.beer]) {
        [self setImageForBeer:image];
    }
    
在ImageSaver.m里把这行注释关掉  
	
	if ([imgData writeToFile:jpgPath atomically:YES]) {
        beer.beerDetails.image = path;
	}

并且  
	
	#import "Beer.h"
	#import "BeerDetails.h"  
	
这时候，用户可能有两种选择。一个是取消，一个是确定  
如果取消： 

	- (void)cancelAdd {
    [self.beer deleteEntity];
    [self.navigationController popViewControllerAnimated:YES];
	}

取消后，数据从缓存里删除。就像数据表里的这行数据删除了。  
如果确定：  

	- (void)saveContext {
    [[NSManagedObjectContext defaultContext] saveToPersistentStoreWithCompletion:^(BOOL success, NSError *error) {
        if (success) {
            NSLog(@"You successfully saved your context.");
        } else if (error) {
            NSLog(@"Error saving context: %@", error.description);
        }
    }];
	}
确定后，数据被保存进磁盘数据库。 

![](https://koenig-media.raywenderlich.com/uploads/2013/11/Working-Details-Page.png)

PS:

	#define MR_ENABLE_ACTIVE_RECORD_LOGGING  
	可以打开/关闭 debug模式。默认DEBUG时打开了。  
	

###Noch ein Bier, bitte  
在MasterViewController里取出数据  
	
	
	- (void)viewWillAppear:(BOOL)animated {
    	[super viewWillAppear:animated];
	    // Check if the user's sort preference has been saved.
   		...
	    [self fetchAllBeers];//这行是新加的
   	 	[self.tableView reloadData];
	}
	
 	- (void)fetchAllBeers {
   	 	// 1. Get the sort key
	    NSString *sortKey = [[NSUserDefaults standardUserDefaults] objectForKey:WB_SORT_KEY];
   		// 2. Determine if it is ascending
	    BOOL ascending = [sortKey isEqualToString:SORT_KEY_RATING] ? NO : YES;
   		// 3. Fetch entities with MagicalRecord
	    self.beers = [[Beer findAllSortedBy:sortKey ascending:ascending] mutableCopy];
	}
	
findAllSortedBy:ascending 抓取实体数据，根据键值，是否顺序  
在NSManagedObject+MagicalFinders里有很多方法可以拿出这些对象  

* findAllInContext: – 找见所有某种类型的所有实体
* findAll – will find all entities on the current thread’s context
* findAllSortedBy:ascending:inContext: – 类似于第一个，对Context有限制
* findAllWithPredicate: – 使用 NSPredicate 搜索对象
* findAllSortedBy:ascending:withPredicate:inContext: 

在configureCell:atIndex:里把beer的name和rating 赋值上去：  
	
	- (void)configureCell:(UITableViewCell*)cell atIndex:(NSIndexPath*)indexPath {
    // Get current Beer
    Beer *beer = self.beers[indexPath.row];
    cell.textLabel.text = beer.name;	
    // Setup AMRatingControl
    AMRatingControl *ratingControl;
    if (![cell viewWithTag:20]) {
        ratingControl = [[AMRatingControl alloc] initWithLocation:CGPointMake(190, 10) emptyImage:[UIImage imageNamed:@"beermug-empty"] solidImage:[UIImage imageNamed:@"beermug-full"] andMaxRating:5];
        ratingControl.tag = 20;
        ratingControl.autoresizingMask = UIViewAutoresizingFlexibleLeftMargin;
        ratingControl.userInteractionEnabled = NO;
        [cell addSubview:ratingControl];
    } else {
        ratingControl = (AMRatingControl*)[cell viewWithTag:20];
    }
    // Put beer rating in cell
    ratingControl.rating = [beer.beerDetails.rating integerValue];
	
	}

prepareForSegue:sender里传值  
	
	if ([[segue identifier] isEqualToString:@"editBeer"]) {
    NSIndexPath *indexPath = [self.tableView indexPathForSelectedRow];
    Beer *beer = self.beers[indexPath.row];
    upcoming.beer = beer;
	}  

###Finishing Touches  
还有一些事要做，比如删除，预览，搜索  

####Delete  

In MasterViewController.m, find tableView:commitEditingStyle:forRowAtIndexPath:, and add the following code:  

	- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        Beer *beerToRemove = self.beers[indexPath.row];
        // Remove Image from local documents
        if (beerToRemove.beerDetails.image) {
            [ImageSaver deleteImageAtPath:beerToRemove.beerDetails.image];
        }
        // Deleting an Entity with MagicalRecord
        [beerToRemove deleteEntity];
        [self saveContext];
        [self.beers removeObjectAtIndex:indexPath.row];
        [tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];
    }
	}
	
	- (void)saveContext {
	// Save ManagedObjectContext using MagicalRecord
	[[NSManagedObjectContext defaultContext] saveToPersistentStoreAndWait];
	}
	
	至此，你就不必知道什么时候完成这个操作，而且还有MagicalRecord的其他选项来保存managedObjectContext。  
	
####Searching  
先前的数据的取出都是全部取出，这次的检索可以使用符合特定的术语。  
使用NSPredicate。  

	- (void)doSearch {
	// 1. Get the text from the search bar.
	NSString *searchText = self.searchBar.text;
	// 2. Do a fetch on the beers that match Predicate criteria.
	// In this case, if the name contains the string
	self.beers = [[Beer findAllSortedBy:SORT_KEY_NAME ascending:YES withPredicate:[NSPredicate predicateWithFormat:@"name contains[c] %@", searchText] inContext:[NSManagedObjectContext defaultContext]] mutableCopy];
	// 3. Reload the table to show the query results.
	[self.tableView reloadData];
	}  
	
![](https://koenig-media.raywenderlich.com/uploads/2013/12/Search.png)

完整项目在[这里](https://koenig-media.raywenderlich.com/uploads/2014/01/BeerTracker-Finished.zip)