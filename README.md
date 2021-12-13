
调用python服务，创建一个线程，提交到线程池。等待返回。如果超时则终止，返回err，并记录日志。

方法二：Future.get(long million, TimeUnit unit)  配合Future.cancle(true)

Future系列（它的子类）的都可以实现，这里采用最简单的Future接口实现。

public class FutureTest {
static class Task implements Callable<Boolean>{
public String name;
private int time;
public Task(String s,int t) {
name=s;
time=t;
}
@Override
public Boolean call() throws Exception {
for(int i=0;i<time;++i){
System.out.println("task "+name+" round "+(i+1));
try {
Thread.sleep(1000);
} catch (InterruptedException e) {
System.out.println(name+" is interrupted when calculating, will stop...");
return false;  //注意这里如果不return的话，线程还会继续执行，所以任务超时后在这里处理结果然后返回
}
}
return true;
}
}

public static void main(String[] args) {
ExecutorService executor=Executors.newCachedThreadPool();
Task task1=new Task("one", 5);
Future<Boolean> f1=executor.submit(task1);
try {
if(f1.get(2, TimeUnit.SECONDS)){     //future将在2秒之后取结果
System.out.println("one complete successfully");
}
} catch (InterruptedException e) {
System.out.println("future在睡着时被打断");
executor.shutdownNow();
} catch (ExecutionException e) {
System.out.println("future在尝试取得任务结果时出错");
executor.shutdownNow();
} catch (TimeoutException e) {
System.out.println("future时间超时");

f1.cancel(true);
//executor.shutdownNow();
//executor.shutdown();
}finally{

executor.shutdownNow();

}
}
}
————————————————

	
	


ForkJoinPool 
package com.baeldung.controllers;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.request.async.DeferredResult;

import java.util.concurrent.ForkJoinPool;

@RestController
public class DeferredResultController {

	private final static Logger LOG = LoggerFactory.getLogger(DeferredResultController.class);

	@GetMapping("/async-deferredresult")
	public DeferredResult<ResponseEntity<?>> handleReqDefResult(Model model) {
	    LOG.info("Received async-deferredresult request");
	    DeferredResult<ResponseEntity<?>> output = new DeferredResult<>();
	    ForkJoinPool.commonPool().submit(() -> {
	        LOG.info("Processing in separate thread");
	        try {
			    Thread.sleep(6000);
	        } catch (InterruptedException e) {
	        }
	        output.setResult(ResponseEntity.ok("ok"));
	    });
	    LOG.info("servlet thread freed");
	    return output;
	}

	public DeferredResult<ResponseEntity<?>> handleReqWithTimeouts(Model model) {
		LOG.info("Received async request with a configured timeout");
		DeferredResult<ResponseEntity<?>> deferredResult = new DeferredResult<>(500l);
		deferredResult.onTimeout(new Runnable() {
			@Override
			public void run() {
				deferredResult.setErrorResult(
						ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT).body("Request timeout occurred."));
			}
		});
		ForkJoinPool.commonPool().submit(() -> {
			LOG.info("Processing in separate thread");
			try {
				Thread.sleep(600l);
				deferredResult.setResult(ResponseEntity.ok("ok"));
			} catch (InterruptedException e) {
				LOG.info("Request processing interrupted");
				deferredResult.setErrorResult(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
						.body("INTERNAL_SERVER_ERROR occurred."));
			}

		});
		LOG.info("servlet thread freed");
		return deferredResult;
	}

	public DeferredResult<ResponseEntity<?>> handleAsyncFailedRequest(Model model) {
		DeferredResult<ResponseEntity<?>> deferredResult = new DeferredResult<>();
		ForkJoinPool.commonPool().submit(() -> {
			try {
				// Exception occurred in processing
				throw new Exception();
			} catch (Exception e) {
				LOG.info("Request processing failed");
				deferredResult.setErrorResult(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
						.body("INTERNAL_SERVER_ERROR occurred."));
			}
		});
		return deferredResult;
	}

}
