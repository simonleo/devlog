来自[Get UIViewController from UIView](http://stackoverflow.com/questions/1340434/get-to-uiviewcontroller-from-uiview-on-iphone)  
  
a category of UIView	  
  
	@interface UIView (FindUIViewController)
	- (UIViewController *) firstAvailableUIViewController;
	- (id) traverseResponderChainForUIViewController;
	@end

	@implementation UIView (FindUIViewController)
	- (UIViewController *) firstAvailableUIViewController {
    	// convenience function for casting and to "mask" the recursive function
    	return (UIViewController *)[self traverseResponderChainForUIViewController];
	}

	- (id) traverseResponderChainForUIViewController {
    	id nextResponder = [self nextResponder];
    	if ([nextResponder isKindOfClass:[UIViewController class]]) {
        	return nextResponder;
    	} else if ([nextResponder isKindOfClass:[UIView class]]) {
        	return [nextResponder traverseResponderChainForUIViewController];
    	} else {
        	return nil;
    	}
	}
	@end  
  
使用:  
  
	// from a UIView subclass... returns nil if UIViewController not available
	UIViewController * myController = [self firstAvailableUIViewController];  
  
需要知道的是，这个解决办法是不得已之举，在合理的设计下view是不需要知道自己的controller。  

   + Your view should not need to access the view controller directly.
   + The view should instead be independent of the view controller, and be able to work in different contexts.
   + Should you need the view to interface in a way with the view controller, the recommended way, and what Apple does across Cocoa is to use the delegate pattern.  
  
+ but, one reason you need to allow the UIView to be aware of its UIViewController is when you have custom UIView subclasses that need to push a modal view/dialog.  
  
另外，通过响应链还可以这样优化:  
  
	- (UIViewController *)firstAvailableUIViewController
	{
    	id nextResponder = [self nextResponder];
    
    	while (nextResponder)
    	{
        	if ([nextResponder isKindOfClass:[UIViewController class]])
        	{
            return nextResponder;
        	}
        	else if (![nextResponder isKindOfClass:[UIViewController class]] &&
                 ![nextResponder isKindOfClass:[UIView class]])
        	{
            /*  根据文档：
                UIView          的nextResponder是它的controller或superview；
                UIViewController的nextResponder是它view的superview；
                UIWindow        的nextResponder是UIApplication对象；
                UIApplication   的nextResponder是nil。
    
                因此当responder不是UIView或UIViewController之一时，无需再进行剩下的操作
             */
            	break;
        	}
        
        	nextResponder = [nextResponder nextResponder];
    	}
    
    	return nil;
	}
