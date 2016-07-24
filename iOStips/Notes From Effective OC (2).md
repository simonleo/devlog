Notes From Effective OC (2)  
---  
---  
  
####tip22 . 理解NSCopying协议  
1.若想令自己所写的对象具有拷贝功能，则需实现`NSCopying`协议。  
并实现`copyWithZone:` 方法。  
  
	@interface EOCPerson : NSObject<NSCopying>
	@property (nonatomic, copy, readonly) NSString name;
	- (id)initWithName:(NSString *)name;
	@end
	
	@implementation EOCPerson {
		NSMutableSet *_friends;
	}
	- (id)initWithName:(NSString *)name {
		if ((self = [super init])) {
			_name = [name copy];
			_friends = [NSMutableSet new];
		}
		return self;
	}
	- (id)copyWithZone:(NSZone *)zone {
		EOCPerson *copy = [[[self class] allocWithZone:zone] initWithName:_name];
		copy->_friends = [_friends mutableCopy];
		return copy;
	}
	@end    
注：这里使用了->语法，因为_friends并非属性，只是在内部使用的实例变量。  
2.如果自定义的对象分为可变版本与不可变版本，那么就要同时实现`NSCopying`与`NSMutableCopying`协议。实现 `- (id)mutableCopyWithZone:(NSZone *)zone;`  
若需获取可变版本的拷贝，均需调用mutableCopy方法。同理，若需要不可变的拷贝，则总应通过copy方法来获取。下列关系总是成立的:  
  
	-[NSMutableArray copy] => NSArray
	-[NSArray mutableCopy] => NSMutableArray  
3.复制对象时需决定采用浅拷贝还是深拷贝，一般情况应尽量执行浅拷贝。如果需要深拷贝，那么可考虑新增一个专门执行深拷贝的方法。如NSSet,该类提供了下面这个初始化方法以执行深拷贝:  
`-(id)initWithSet:(NSArray *)array copyItems:(BOOL)copyItems;`  
再比如:  
  
	- (id)deepCopy {
		EOCPerson *copy = [[[self class] alloc] initWithName:_name];
		copy->_friends = [[NSMutableSet alloc] initWithSet:_friends copyItems:YES];
		return copy;
	}  
  
####tip26 . 勿在分类中声明属性  
1.除了"class-continuation分类"外，其他分类都无法向类中新增实例变量，因此，它们无法把实现属性所需的实例变量合成出来。  
有时候只读属性还是可以在分类中定义的。  
  
	@interface NSCalender (EOC_Additions)
	@property (nonatomic, strong, readonly) NSArray *eoc_allMonths;
	@end
	
	@implementation NSCalender (EOC_Additions)
	- (NSArray *)eoc_allMonths {
		...
		return ..
	}
	@end  
由于实现属性所需的全部方法（本例中，属性是只读的，所以只需实现一个方法）都已实现，所以不会再为该属性自动合成实例变量了。  
但即便这样，也最好不要用属性。属性是用来封装数据的。本例中应该直接声明一个方法，用以获取月份名称列表：
  
	@interface NSCalender (EOC_Additions)
	- (NSArray *)eoc_allMonths;
	@end  
总结：分类中可以定义存取方法，但尽量不要定义属性。把封装数据所用的全部属性都应该定义在主接口里。  
2.关联对象  
  
	static const char *kFriendsPropertyKey = "kFriendsPropertyKey";
	@implementation EOCPerson (Friendship)
	- (NSArray *)friends {
		return objc_getAssociateObject(self, kFriendsPropertyKey);
	}
	- (void)setFriends:(NSArray *)friends {
		objc_setAssociateObject(self, kFriendsPropertyKey, friends, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
	}
	@end  
这样可行，但不太理想。在内存管理问题上容易出错，因为在为属性实现存取方法时，经常会忘记遵从其内存管理语义。  
  
####tip