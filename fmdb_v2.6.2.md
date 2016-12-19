###FMDB v2.6.2
####Usage
在FMDB内有三种主要的类  
1.FMDatabase  代表一个单一的SQLite 数据库。用于执行SQL  
2.FMResultSet 代表在FMDatabase上的查询结果  
3.FMDatabaseQueue 如果你希望在多线程上执行查询和更新，你应该会想用这个class  
在“线程安全”章节会有描述  

####Database Creation
一个 FMDatabase 由SQLite database 文件路径创建，这个路径可以包含以下三种之一  
1.一个文件系统路径。文件不会存在在硬盘上，如果不存在，他会自动为你创建  
2.一个空string（@“”），一个空数据库被创建在一个临时位置(内存中)。一旦FMDatabase连接关闭，数据库被删除  
3.NULL，数据库在内存里创建。数据库关闭连接时，立即销毁。  

	FMDatabase *db = [FMDatabase databaseWithPath:@"/tmp/tmp.db"];
	
####Opening  
在你和数据库交互之前，必须打开它。如果没有足够的资源和权限，打开和创建都会失败。
	
	if (![db open]) {
    	[db release];
	    return;
	}


####Executing Updates
任何种类的SQL语句不是SELECT语句就是 update。包括 CREATE, UPDATE, INSERT, ALTER, COMMIT, BEGIN, DETACH, DELETE, DROP, END, EXPLAIN, VACUUM, and REPLACE。基本上，你的SQL语句不是以SELECT开头，那就是个update陈述。  

执行一个更新会返回一个bool值。YES意味着更新成功，NO意味着遇见问题。你可以调用-lastErrorMessage 和 -lastErrorCode 检查更多的信息。


####Executing Queries
一个 SELECT 语句就是一个查询，并且通过 -executeQuery... 方法执行  
查询成功后会返回一个FMResultSet对象，失败为nil。使用-lastErrorMessage and -lastErrorCode 查看为什么会失败  

使用while() 来遍历你的查询结果，同样可以用“step” 从一个记录到另一个记录。对于FMDB，最容易的方式是这样的：
	
	FMResultSet *s = [db executeQuery:@"SELECT * FROM myTable"];
	while ([s next]) {
    	//retrieve values for each record
	}
	
在尝试访问返回值前，你必须调用-[FMResultSet next]，即使你只有一个请求：
	
	FMResultSet *s = [db executeQuery:@"SELECT COUNT(*) FROM myTable"];
	if ([s next]) {
    	int totalCount = [s intForColumnIndex:0];
	}
	
FMResultSet 有许多格式化的方法获取数据：

   * intForColumn:
   * longForColumn:
   * longLongIntForColumn:
   * boolForColumn:
   * doubleForColumn:
   * stringForColumn:
   * dateForColumn:
   * dataForColumn:
   * dataNoCopyForColumn:
   * UTF8StringForColumnName:
   * objectForColumnName:
   
典型的，这里你自己不需要 -close 一个 FMResultSet，这是因为ResultSet 被销毁或父数据库关闭

####Closing
查找更新结束后需关闭数据库，在操作期间SQLite会放弃所有资源：

	[db close];

####Multiple Statements and Batch Stuff
使用 executeStatements:withResultBlock: 执行多步操作
	
	NSString *sql = @"create table bulktest1 (id integer primary key autoincrement, x text);"
                 "create table bulktest2 (id integer primary key autoincrement, y text);"
                 "create table bulktest3 (id integer primary key autoincrement, z text);"
                 "insert into bulktest1 (x) values ('XXX');"
                 "insert into bulktest2 (y) values ('YYY');"
                 "insert into bulktest3 (z) values ('ZZZ');";

	success = [db executeStatements:sql];

	sql = @"select count(*) as count from bulktest1;"
    	   "select count(*) as count from bulktest2;"
	       "select count(*) as count from bulktest3;";

	success = [self.db executeStatements:sql withResultBlock:^int(NSDictionary *dictionary) {
    	NSInteger count = [dictionary[@"count"] integerValue];
	    XCTAssertEqual(count, 1, @"expected one record for dictionary %@", dictionary);
    	return 0;
	}];
	
####Data Sanitization
当提供SQL给FMDB，你不需要尝试处理任何值。代替的是，你应该使用标准SQLite语法：

	INSERT INTO myTable VALUES (?, ?, ?, ?)
	
？是个占位符。执行方法可以接受多个参数~

	NSInteger identifier = 42;
	NSString *name = @"Liam O'Flaherty (\"the famous Irish author\")";
	NSDate *date = [NSDate date];
	NSString *comment = nil;

	BOOL success = [db executeUpdate:@"INSERT INTO authors (identifier, name, date, comment) VALUES (?, ?, ?, ?)", @(identifier), name, 	date, comment ?: [NSNull null]];
	if (!success) {
    	NSLog(@"error = %@", [db lastErrorMessage]);
	}
	
此外，你还能使用下面的语法

	INSERT INTO authors (identifier, name, date, comment) VALUES (:identifier, :name, :date, :comment)
	
参数必须以冒号开头，SQLite 支持其他字符，但是内部的字典key是有冒号前缀的，不要在你的字典key里加冒号。

	NSDictionary *arguments = @{@"identifier": @(identifier), @"name": name, @"date": date, @"comment": comment ?: [NSNull null]};
	BOOL success = [db executeUpdate:@"INSERT INTO authors (identifier, name, date, comment) VALUES (:identifier, :name, :date, :comment)" withParameterDictionary:arguments];
	if (!success) {
    	NSLog(@"error = %@", [db lastErrorMessage]);
	}
关键点是不要用 NSString 的 stringWithFormat 管理插入值，也不要用swift的 string interpolation 。使用 ？ 占位符作为值插入进database

####Using FMDatabaseQueue and Thread Safety.
从多线程一次性的使用一个FMDatabase单例是个坏主意，在每个线程上使用一个FMDatabase 对象是OK的。只是不要通过线程共享一个单例，也不要通过多线程。
#####总之，不要初始化一个单独的FMDatabase对象和通过多线程使用它

替代的是，用FMDatabaseQueue，初始化一个单独的FMDatabaseQueue和通过多线程使用它。FMDatabaseQueue 将会通过多线程同步，协调的访问数据库。

	FMDatabaseQueue *queue = [FMDatabaseQueue databaseQueueWithPath:aPath];

	[queue inTransaction:^(FMDatabase *db, BOOL *rollback) {
    	[db executeUpdate:@"INSERT INTO myTable VALUES (?)", @1];
	    [db executeUpdate:@"INSERT INTO myTable VALUES (?)", @2];
    	[db executeUpdate:@"INSERT INTO myTable VALUES (?)", @3];

	    if (whoopsSomethingWrongHappened) {
    	    *rollback = YES;
        	return;
	    }
    	// etc…
	    [db executeUpdate:@"INSERT INTO myTable VALUES (?)", @4];
	}];
	
####示例：

	-(FMDatabase *)openDataBaseWithpath:(NSString *)DBpath{
    	FMDatabase *db = [FMDatabase databaseWithPath:DBpath];
	    if (![db open]) {
    	    [db close];
        	return nil;
	    }
    	return db;
	}

       
	-(BOOL)createTable:(FMDatabase *)db tableName:(NSString *)tableName SQL:(NSString *)sql{
    
		if (![sql hasPrefix:@"CREATE TABLE IF NOT EXISTS"]) {
        	NSLog(@"TABLE is not CREATE");
	        return NO;
    	}
    
	    if (!db.goodConnection) {
    	    NSLog(@"DataBase is not goodConnection");
        	return NO;
	    }
    
    	BOOL validate = [self validateSQLWithDB:db SQL:sql];
	    if (!validate) {
    	    return NO;
	    }
    
    	BOOL success = [db executeUpdate:sql];
	    if (!success) {
    	    NSLog(@"error = %@", [db lastErrorMessage]);
	    }
    	return success;
	}

	-(BOOL)executeDeleteRow:(FMDatabase *)db tableName:(NSString *)tableName ColumnName:(NSString *)columnName Value:(NSString *)value{
    
    	NSString *sql = [NSString stringWithFormat:@"DELETE FROM %@ WHERE %@ = '%@'",tableName,columnName,value];
    
	    if (!db.goodConnection) {
    	    NSLog(@"DataBase is not goodConnection");
        	return NO;
	    }
    
    	BOOL validate = [self validateSQLWithDB:db SQL:sql];
	    if (!validate) {
    	    return NO;
	    }
    
    	BOOL success = [db executeUpdate:sql];
	    if (!success) {
    	    NSLog(@"error = %@", [db lastErrorMessage]);
	    }
    	return success;
	}

	-(BOOL)executeDeleteAll:(FMDatabase *)db tableName:(NSString *)tableName{
    	
	    if (!db.goodConnection) {
    	    NSLog(@"DataBase is not goodConnection");
        	return NO;
	    }
    
    	NSString *sql = [NSString stringWithFormat: @"DELETE FROM %@",tableName];
    
	    BOOL validate = [self validateSQLWithDB:db SQL:sql];
    	if (!validate) {
        	return NO;
	    }
    
    	BOOL success = [db executeUpdate:sql];
	    if (!success) {
    	    NSLog(@"error = %@", [db lastErrorMessage]);
	    }
    	return success;
	}


	-(BOOL)executeInsert:(FMDatabase *)db tableName:(NSString *)tableName SQL:(NSString *)sql{
    	
	    if (!db.goodConnection) {
    	    NSLog(@"DataBase is not goodConnection");
        	return NO;
    	}
    
	    BOOL tableExist = [self tableExistWithDB:db tableName:tableName];
    	if (!tableExist) {
        	return NO;
	    }
    
    	BOOL validate = [self validateSQLWithDB:db SQL:sql];
	    if (!validate) {
    	    return NO;
	    }
    
    	BOOL success = [db executeUpdate:sql];
	    if (!success) {
    	    NSLog(@"error = %@", [db lastErrorMessage]);
    	    return NO;
	    }
    
    	return success;
	}

	-(BOOL)executeUpdate:(FMDatabase *)db tableName:(NSString *)tableName
    	          Column:(NSString *)ColumnName
        	   oldString:(NSString *)oldStr
           	newString:(NSString *)newStr{
    
	    NSString *sql = [NSString stringWithFormat:@"UPDATE %@ SET %@ = '%@' WHERE %@ = '%@' ",tableName,ColumnName,newStr,ColumnName,oldStr];
    
    	if (!db.goodConnection) {
        	NSLog(@"DataBase is not goodConnection");
	        return NO;
    	}
    
	    BOOL validate = [self validateSQLWithDB:db SQL:sql];
    	if (!validate) {
        	return NO;
	    }
    
    	BOOL success = [db executeUpdate:sql];
	    if (!success) {
    	    NSLog(@"error = %@", [db lastErrorMessage]);
        	return NO;
    	}
    
	    return success;
	}


	-(NSMutableArray *)executeSelect:(FMDatabase *)db tableName:(NSString *)tableName{
    
    	NSString *sql = [NSString stringWithFormat: @"SELECT * FROM %@",tableName];
    
	    if (!db.goodConnection) {
    	    NSLog(@"DataBase is not goodConnection");
        	return NO;
	    }
    
    	BOOL validate = [self validateSQLWithDB:db SQL:sql];
	    if (!validate) {
    	    return nil;
	    }
    
    	FMResultSet *set = [db executeQuery:sql];
	    int count = [set columnCount];
    	NSMutableArray *mutArr = [NSMutableArray new];
    
	    while ([set next]) {
        
    	    for (int i = 0 ; i < count +1; i ++) {
        	    id obj = [set objectForColumnIndex:i];
            	[mutArr addObject:obj];
	        }
    	}
    
	    return mutArr;
	}



	-(FMDatabaseQueue *)openDatabaseQueueWithpath:(NSString *)DBpath sqlArray:(NSArray <NSString *> *)sqlArr{
    
    	FMDatabaseQueue *queue = [FMDatabaseQueue databaseQueueWithPath:DBpath];
    
	    [queue inTransaction:^(FMDatabase *db, BOOL *rollback) {
        
    	    for (int i = 0; [sqlArr count]; i++) {
        	    NSString *sql = [sqlArr objectAtIndex:i];
            	BOOL success = [db executeUpdate:sql];
	            if (!success) {
    	            NSLog(@"error = %@", [db lastErrorMessage]);
        	        *rollback = YES;
            	    return;
	            }
    	    }

	    }];
    
    	return queue;
	}
	
	
	-(int)totalCount:(FMDatabase *)db tableName:(NSString *)tableName{
    
    	NSString *sql = [NSString stringWithFormat: @"SELECT COUNT(*) FROM %@",tableName];
    
	    if (!db.goodConnection) {
    	    NSLog(@"DataBase is not goodConnection");
        	return NO;
	    }
    
    	BOOL validate = [self validateSQLWithDB:db SQL:sql];
	    if (!validate) {
    	    return 0;
	    }
    
    	FMResultSet *s = [db executeQuery:sql];
		if ([s next]) {
    	    int total = [s intForColumnIndex:0];
        	return total;
	    }else{
    	    return 0;
	    }
    	return 0;
	}


	-(BOOL)closeDB:(FMDatabase *)db{
    
    	return [db close];
	}


	-(BOOL)tableExistWithDB:(FMDatabase *)db tableName:(NSString *)tableName{
    
    	BOOL isTable = [db tableExists:tableName];
	    if (!isTable) {
    	    NSLog(@"error =  %@ table is not Exists",tableName);
        	return NO;
	    }else{
    	    return YES;
	    }
	}


	-(BOOL)validateSQLWithDB:(FMDatabase *)db SQL:(NSString *)sql{
    
    	NSError * error;
	    BOOL validateSQL = [db validateSQL:sql error:&error];
    	if (!validateSQL) {
        	NSLog(@"sql error = %@", error);
	        return NO;
    	}else{
        	return YES;
	    }
	}

##优化Sqlite  

1.开启SQLite的WAL模式（Write-Ahead-Log）

###Write-Ahead Logging   

#####1. Overview  
默认情况下，SQLite实现原子提交和回滚操作，称为rollback journal。在3.7.0之后(2010-07-21)，一个新的"Write-Ahead Log" 选项 (hereafter referred to as "WAL")可以使用了：  

有几个优点来代替rollback journal模式：  
1.WAL在大多场景下是非常快的。  
2.WAL提供，在读的时候不会阻碍写，写的时候不会阻碍读。读写可以并发进行。
3.Disk I/O 操作倾向于更有次序  
4.使用少很多的fsync() 操作，因此系统调用的fsync() 失败时，会减少损失。  

####2.How WAL Works  
传统的rollback journal工作是通过生成一份原始未修改的数据库内容拷贝到一个单独的rollback journal文件，再把改变直接写入数据库文件内。当crash或rollback，在rollback journal内的原始内容恢复到原来的状态。commit后rollback journal删除。  

WAL是相反的，原始内容被保存在数据库文件内，变化被加载到WAL文件里。当一个特殊的标记标出一个commit被添加到WAL时，才会COMMIT到数据库里。因此一个COMMIT可以发生在任何写原始数据库的时候。允许读写同时进行，多个事务可以追加到一个单独WAL文件后面。  

此时写操作会先append到wal文件末尾，而不是直接覆盖旧数据。而读操作开始时，会记下当前的WAL文件状态，并且只访问在此之前的数据。这就确保了多线程读与读、读与写之间可以并发地进行。  

####I/O 性能优化
开启WAL模式后，写入的数据会先append到WAL文件的末尾。待文件增长到一定长度后，SQLite会进行checkpoint。当执行checkpoint操作时，WAL文件的内容会被写回数据库文件，这个长度默认为1000个页大小，在iOS上约为3.9MB。

同样的，在数据库关闭时，SQLite也会进行checkpoint。不同的是，checkpoint成功之后，会将WAL文件长度删除或truncate到0。下次打开数据库，并写入数据时，WAL文件需要重新增长。而对于文件系统来说，这就意味着需要消耗时间重新寻找合适的文件块。

显然SQLite的设计是针对容量较小的设备，尤其是在十几年前的那个年代，这样的设备并不在少数。而随着硬盘价格日益降低，对于像iPhone这样的设备，几MB的空间已经不再是需要斤斤计较的了。

因此我们可以修改为：

    数据库关闭并checkpoint成功时，不再truncate或删除WAL文件只修改WAL的文件头的Magic Number。下次数据库打开时，SQLite会识别到WAL文件不可用，重新从头开始写入。

保留WAL文件大小后，每个数据库都会有这约3.9MB的额外空间占用。如果数据库较多，这些空间还是不可忽略的。因此，微信中目前只对读写频繁且检测到卡顿的数据库开启，如聊天记录数据库。  
