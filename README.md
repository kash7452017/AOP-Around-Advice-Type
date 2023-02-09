## AOP：@Around Advice Type
### 基本用法演示
>參考網址：https://www.cnblogs.com/csniper/p/5499248.html
>
>@Around註解可以用來在調用一個具體方法前和調用後來完成一些具體的任務。
>
>Around advice可以通過一個在joinpoint執行前後做一些事情的機會，可以決定什麼時候，怎麼樣去執行joinpoint，甚至可以決定是否真的執行joinpoint的方法調用。Around advice通常是用在下面這樣的情況：
>
>在多線程環境下，在joinpoint方法調用前後的處理中需要共享一些數據。如果使用Before advice和After advice也可以達到目的，但是就需要在aspect裡面創建一個存儲共享信息的field，而且這種做法並不是線程安全的

**創建TrafficFortuneService.java類別，添加`getFortune()`方法，透過延時5秒模擬一個方法執行**
```
@Component
public class TrafficFortuneService {

	public String getFortune() {
		
		// simulate a delay
		try {
			
			TimeUnit.SECONDS.sleep(5);
			
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		// return a fortune
		return "Expect heavy traffic this morning";
	}
}
```
>ProceedingJoinPoint是JoinPoint的擴展，它公開了額外的proceed()方法。調用時，代碼執行跳轉到下一個建議或目標方法。它使我們能夠控制代碼流並決定是否繼續進行進一步的調用。
>
>它可以與@Around通知一起使用，它圍繞著整個方法調用

**在MyDemoLoggingAspect.java中添加@Around建議方法，`long begin`可以看作@Before，在方法執行前啟動計時；而`long end`看作@After，在方法執行完畢後停止計時並計算運行時間，而方法是否執行、在甚麼時間點執行，則透過`proceed()`方法來控制**
```
@Aspect
@Component
@Order(2)
public class MyDemoLoggingAspect {
	
	@Around("execution(* com.luv2code.aopdemo.service.*.getFortune(..))")
	public Object aroundGetFortune(ProceedingJoinPoint theProceedingJoinPoint) throws Throwable {
		
		
		// print out method we are advising on
		String method = theProceedingJoinPoint.getSignature().toShortString();
		System.out.println("\n=====>>> Excuting @Around on method: " + method);		
		
		// get begin timestamp
		long begin = System.currentTimeMillis();
		
		// now, let's execute the method
		Object result = theProceedingJoinPoint.proceed();
		
		// get end timestamp
		long end = System.currentTimeMillis();
		
		// compute duration and display it
		long duration = end - begin;
		System.out.println("\n=====> Duration: " + duration / 1000.0 + " seconds");
		
		return result;		
	}
}
```
**創建主程序AroundDemoApp.java，打印一些訊息以便查看整體運作順序，調用getFortune()方法以了解@Around建議的作用**
```
public class AroundDemoApp {

	public static void main(String[] args) {
		
		// read spring config java class
		AnnotationConfigApplicationContext context = 
				new AnnotationConfigApplicationContext(DemoConfig.class);
		
		// get the bean from spring container
		TrafficFortuneService theFortuneService = context.getBean("trafficFortuneService", TrafficFortuneService.class);
		
		System.out.println("\nMain Program: AroundDemoApp");
		
		System.out.println("Calling getFortune");
		
		String data = theFortuneService.getFortune();
		
		System.out.println("\nMy fortune is: " + data);
		
		System.out.println("Finished");
		
		// close the context
		context.close();

	}
}
```
![image](https://user-images.githubusercontent.com/101872264/217838101-56cf3ef4-ddef-48b7-9999-b60f0fe9bf62.png)

-------------------------------------------------------------------------
### 異常處理情況
**在MyDemoLoggingAspect.java中稍作修改，@Around同時包含攔截例外錯誤的功能，並且可以依需求改寫、處理錯誤，再次拋出修改後的錯誤訊息或是不顯示錯誤訊息**
```
@Aspect
@Component
@Order(2)
public class MyDemoLoggingAspect {
	
	private Logger myLogger = Logger.getLogger(getClass().getName());
	
	@Around("execution(* com.luv2code.aopdemo.service.*.getFortune(..))")
	public Object aroundGetFortune(ProceedingJoinPoint theProceedingJoinPoint) throws Throwable {
		
		
		// print out method we are advising on
		String method = theProceedingJoinPoint.getSignature().toShortString();
		myLogger.info("\n=====>>> Excuting @Around on method: " + method);		
		
		// get begin timestamp
		long begin = System.currentTimeMillis();
		
		// now, let's execute the method
		Object result = null;
		
		try {
			
			result = theProceedingJoinPoint.proceed();
		
		} catch (Exception e) {
			
			// log the exception
			myLogger.warning(e.getMessage());
			
			// give user a customer message
			result = "Major accident! But no worries, " 
					+ "your private AOP helicopter is on the way!";
			
			// rethrow exception
			// throw e;
		}
		
		// get end timestamp
		long end = System.currentTimeMillis();
		
		// compute duration and display it
		long duration = end - begin;
		myLogger.info("\n=====> Duration: " + duration / 1000.0 + " seconds");
		
		return result;		
	}
```
**在TrafficFortuneService.java中添加`getFortune(boolean tripWire)`方法，此處依照`true`or `false`來決定是否強制拋出例外，若tripWire值為`false`則正常運作繼續執行`getFortune()`**
```
@Component
public class TrafficFortuneService {

	public String getFortune() {
		
		// simulate a delay
		try {
			
			TimeUnit.SECONDS.sleep(5);
			
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		// return a fortune
		return "Expect heavy traffic this morning";
	}

	public String getFortune(boolean tripWire) {
		
		if(tripWire) {
			throw new RuntimeException("Major accident! Highway is closed!");
		}
		
		return getFortune();
	}
}
```
**創建AroundHandleExceptionDemoApp.java主程序，調用`theFortuneService.getFortune(tripWire)`以查看@Around運作過程**
```
public class AroundHandleExceptionDemoApp {

	private static Logger myLogger = 
					Logger.getLogger(AroundHandleExceptionDemoApp.class.getName());
	
	public static void main(String[] args) {
		
		// read spring config java class
		AnnotationConfigApplicationContext context = 
				new AnnotationConfigApplicationContext(DemoConfig.class);
		
		// get the bean from spring container
		TrafficFortuneService theFortuneService = context.getBean("trafficFortuneService", TrafficFortuneService.class);
		
		myLogger.info("\nMain Program: AroundDemoApp");
		
		myLogger.info("Calling getFortune");
		
		boolean tripWire = true;
		String data = theFortuneService.getFortune(tripWire);
		
		myLogger.info("\nMy fortune is: " + data);
		
		myLogger.info("Finished");
		
		// close the context
		context.close();
	}
}
```
![image](https://user-images.githubusercontent.com/101872264/217845419-3002b4bc-df94-4d5b-bcd2-9b75153fe612.png)
