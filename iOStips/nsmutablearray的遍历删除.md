# NSMutableArray的遍历删除

##### 原问题描述

	NSMutableArray *contacts = [NSMutableArray arrayWithArray:[incrementContactDict wbt_arrayForKey:@"origin"]];
    for (FindFriendsContact *contact in contacts) {
        if (!contact.isExisted) {
        	[contacts removeObject:contact];
        }
     }

实际运行时，**有时候正常，有时候又crash**。为什么呢，原来上面当数组被枚举时被修改了,因为forin遍历时规定不能修改或增加数组元素的。 **for...in loop**引入了 Mutation Guard的机制, 运行时抛出异常:`Collection was mutated while being enumerated`。但是，这里有一个特殊点，假如你想要删除的是数组中最后一个元素的话，程序就不会crash了。这是因为当到最后一个元素时，已经遍历结束了，forin的工作算是结束了，你再删除数组元素已经和它没有关系了，就不会发生冲突了。

##### 解决问题
* 既然forin遍历会crash,那么我们就采用for循环遍历  

		for (int i = 0; i < [contacts count]; i++) {
    		FindFriendsContact *contact = [contacts objectAtIndex:i];
        	if (!contact.isExisted) {
        		[contacts removeObject:contact];
        	}
    	}
    
* 定义一个副本,遍历副本找到想要删除的元素,然后在原数组中删除对应的元素
  
		NSMutableArray *copyArray = [NSMutableArray arrayWithArray:array];
        char name[20] = {0}; // 存储姓名
        NSLog(@"请输入想要删除的联系人的姓名:");
        scanf("%s", name);
        NSString *str1 = [NSString stringWithUTF8String:name];
        for (AddressPerson *perName in copyArray) {
            if ([[perName name] isEqualToString:str1]) {
                [array removeObject:perName];
            }
        }
* 对数组逆序遍历,查找对应元素后删除
  
		// 逆序遍历,然后查找删除
        NSEnumerator *enumerator = [array reverseObjectEnumerator];
        //forin遍历
        for (AddressPerson *groupName in enumerator) {
            if ([[groupName group] isEqualToString:@"Zhangsan"]) {
                [array removeObject:groupName];
            }
        }
          
	有些人会有疑问,你这也是遍历数组时对数组进行改变不了啊,为什么逆序就可以啊?  
具体情况是这样的,当我们正序遍历时,如果删除了一个,那么**没有遍历到的元素位置都会往前移动一位**,这样系统就无法确定接下来遍历**是从删除位置开始呢,还是从删除位置下一位开始呢**?这样就造成程序crash了.对于逆序遍历就不会,因为我们逆序遍历时,遇到匹配的元素删除后,位置改变的是遍历过得元素,而没有遍历到的元素位置却没有改变,所以遍历能够正常进行.  
  
####相关 : [如何避免程序崩溃-2：可变对象的异常 ](http://www.calios.gq/2015/11/13/%E3%80%90%E8%AF%91%E3%80%91%E5%A6%82%E4%BD%95%E9%81%BF%E5%85%8D%E7%A8%8B%E5%BA%8F%E5%B4%A9%E6%BA%83-2%EF%BC%9A%E5%8F%AF%E5%8F%98%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%BC%82%E5%B8%B8/)