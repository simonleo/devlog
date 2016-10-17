# ARC下dealloc方法
---

&emsp;&emsp;设置属性值，一般在初始化方法(init, initWithCoder:, 等…)，dealloc 方法和自定义的setters和getters中，应该直接访问实例变量，因为当init dealloc方法被执行时，类的运行时环境不是处于正常状态的，使用存取方法访问变量可能会导致不可预料的结果。  
&emsp;&emsp;但这是ARC下，而在MRC下dealloc方法有些不同。MRC下需要手动release，而 `_XXX = nil` 没有release，因此需要 `[_XXX release], _XXX = nil;`  
&emsp;&emsp;`self.XXX = nil;` 倒是也可以（不推荐），因为在赋值nil前已经release了。  
  
	- (void)setObj:(id)obj {  
     	//判断是否是自我赋值
     	if (_obj != obj)
     	{
         	//对原对象作一次release,释放对旧对象的所有权
         	[_obj release];
         	//对新对象作一次retain，并赋值给当前对象
         	_obj = [obj retain];
     	}
	 }
Effective OC中对不能使用存取方法的解释：p(29)  
>在初始化方法中应该如何设置属性值。这种情况下总是应该直接访问实例变量，因为子类可能会“覆写”（override）设置方法。假设EOCPerson有一个子类叫做EOCSmithPerson，这个子类专门表示那些姓“Smith”的人。该子类可能会覆写lastName属性所对应的设置方法：  
  
	-(void)setLastName:(NSString *)lastName {
		if (![lastName isEqualToString:@"Smith"]) {
			[NSException raise:NSInvalidArgumentException  
						format:@"Last name must be Smith"];
		}
		self.lastName = lastname;
	}
>在基类EOCPerson的默认初始化方法中，可能会将姓氏设为空字符串。此时若是通过“设置方法”来做，那么调用的将会是子类的设置方法，从而抛出异常。但是，某些情况下却又必须在初始化方法中调用设置方法：如果待初始化的实例变量声明在超类中，而我们又无法在子类中直接访问此实例变量的话，那么就需要调用“设置方法”了。