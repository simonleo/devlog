Notes From Effective OC (1)  
---  
---  
  
####tip3 . 多用字面量语法，少用与之等价的方法  
NSString, NSNumber, NSArray, NSDictionary  
  
	NSNumber *intNumber = @1; <-- NSNumber *intNumber = [NSNumber numberWithInt:1];  
	NSNumber *floatNumber = @2.5f  
	NSNumber *boolNumber = @YES;  
	NSString *someStr = @"effective oc";  
	NSArray *animals = @[@"cat", @"dog", @"mouse"];
	NSDictionary *personData = @{@"firstName" : @"Matt", @"lastName" : @"Galloway", @"age" : @28};  
  
1.用字面量语法创建数组或字典时，若值中有nil，则会抛出异常，这样更加安全。  
*用普通的方法创建 `NSArray *arrayA = [NSArray arrayWithObjects:object1, object2, object3, nil];`如果中间有nil，会由此截断～*  
  
2.使用字面量语法创建出来的字符串、数组、字典对象都是不可变的。若要可变版本对象，需复制一份:  
  
	NSMutableArray *mutable = [@[@1, @2, @3, @4] mutableCopy];  
  
  
####tip4 . 多用类型常量，少用#define预处理指令  
	#define ANIMATION_DURATION 0.3  -->  static const NSTimeInterval kAnimationDuration = 0.3  
1.预处理指令没有类型信息， 此外会把碰到的所有ANIMATION_DURATION都替换为0.3，这样的话，假设指令声明在某头文件中，那么引入此头文件的代码，ANIMATION_DURATION都会被替换。  
2.常量命名：若常量局限于某实现文件(.m)内，则在前面加字母k；若常量在类外可见，则通常以类名为前缀。  
3.变量一定要`同时用static与const`来声明。static修饰符意味着该变量仅在定义此变量的编译单元中可见(.m)。假如不加static，则编译器会为他创建一个“外部符号”。此时若另一个编译单元也声明了同名变量，编译器会抛出错误信息(duplicate symbol)。  
4.有时候需要对外公开常量。  
  
	//in the header file  
	extern NSString *const EOCStringConstant;  
  
	//in the impletation file  
	NSString *const EOCStringConstant = @"VALUE";  
  
	//.h  
	extern const NSTimeInterval EOCAnimationDuration;  
  
	//.m
	const NSTimeInterval EOCAnimationDuration = 0.3;  

  
####tip5 . 用枚举表示状态、选项、状态码  
`typedef NS_ENUM(NSUInteger, EOCCOonnectionState) {
};`  
  
`typedef NS_OPTIONS(NSUInteger, EOCPermittedDirection) {
};`  
  
1.凡是需要以按位或操作来组合的枚举都应使用 NS_OPTIONS 定义。若是枚举不需要相互结合，则应使用NS_ENUM 来定义。  
  
2.在处理枚举类型的switch语句中不要实现default分支。这样的话，加入新枚举之后，编译器就会提示开发者：switch语句并未处理所有枚举。  
  
####tip6 . 理解“属性”这一概念  
	@interface EOCPerson : NSObject
	@property NSString *firstName;
	@property NSString *lastName;
	@end  
1.使用了属性，编译器会自动编写访问这些属性所需的方法（“合成方法”），还会自动向类中添加适当类型的实例变量，并在属性名前加下划线，以此作为实例变量的名字。  
2.也可在类的实现代码里通过`@synthesize`语法来指定实例变量的名字  

	@implementation EOCPerson
	@synthesize firstName = _myFirstName;
	@synthesize lastName = _myLastName;
	@end
生成的实例变量命名为`_myFirstName,_lastName`  
3.也可以自己实现存取方法，如果只实现了其中一个存取方法，那么另外一个还是会由编译器合成。还有一种方法阻止编译器自动合成存取方法，就是使用`@dynamic`关键字。表示：不要自动创建实现属性所用的实例变量，也不要为其创建存取方法。  
  
	@implementation EOCPerson
	@dynamic firstName, lastName;
	@end  
  
####tip16 . 提供“全能初始化方法”  
  
1.在类中提供一个全能初始化方法（Designated initializer），并于文档里指明。其他初始化方法均应调用此方法。  
  
2.若子类全能初始化方法与超类不同，则需覆写超类中的对应方法。  
  
	@implementation EOCRectangle 
	//Designated initializer
	- (id)initWithWidth:(float)width andHeight:(float)height
	{
		if((self = [super init])) {
			_width = width;
			_height = height;
		}
		return self;
	}
	//Superclass's designated initializer
	- (id)init
	{
		return [self initWithWidth:5.0 andHeight:10.0f];
	}
	@end  
  
	@implementation EOCSquare : EOCRectangle
	//Designated initializer
	- (id)initWithDimension:(float)dimension
	{
		return [super initWithWidth:dimension andHeight:dimension];
	}
	//superclass's designated initializer
	-(id)initWithWidth:(float)width andHeight:(float)height
	{
		float dimension = MAX(width, height);
		return [self initWithDimension:dimension];
	}
	@end  
####tip18 . 尽量使用不可变对象  
应尽量把对外公布的属性设为只读，而且只在确有必要时才将属性对外公布。  
  
	@interface EOCPointOfInterest : NSObject
	@property (nonatomic, copy, readonly) NSString *identifier;
	@property (nonatomic, assign, readonly) float latitude;
	@property (nonatomic, assign, readonly) float longitude;
	@end
	
有时候可能想修改封装在对象内部的数据，但却不想令数据为外人所改动。这种情况通常是在对象内部(.m)`将readonly属性重新声明为readwrite`。  
对象里表示各种collection的那些属性，应该提供一个readonly属性供外界使用，该属性返回不可变的set，而此set则是`内部那个可变set的一份拷贝`。  
  
	//EOCPerson.h
	@interface EOCPerson : NSObject
	@property (nonatomic, copy, readonly) NSString *name;
	@property (nonatomic, strong, readonly) NSSet *friends;
	
	- (void)addFriend:(EOCPerson*)person;
	- (void)removeFriend:(EOCPerson*)person;
	@end
	
	//EOCPerson.m
	@interface EOCPerson()
	@property (nonatomic, copy, readwrite) NSString *name;
	@end
	
	@implementation EOCPerson {
		NSMutableSet *_internalFriends;
	}
	
	- (NSSet*)friends {
		return [_internalFriends copy];
	}
	
	- (void)addFriend:(EOCPerson*)person {
		[_internalFriends addObjcec:person];
	}
	@end