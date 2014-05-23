---
layout: post
title: "关于线程消息队列的设计"
date: 2014-04-16 19:35:20 +0800
comments: true
categories: [webkit,chrome]
tags: [thread,webkit,chrome,ruby]
---

###一. 实现一个有消息队列的线程

大概有以下接口：  

* 添加任务到消息队列: addTask
* 从消息队列中获取任务: popTask 
* 执行这个任务
* 要实现一个线程的循环体:threadRunloop.

上述操作是在多个线程中执行的，例如addTask可以由任何一个线程执行。
而popTask，threadRunloop只在当前线程执行。

MessageQueue的实现重要的一点：**要保证对消息队列操作的同步和互斥**.在linux中，就涉及到两个技术点取实现，一个是互斥变量Mute，一个是信号量Condition:
<!--more-->
下面是伪代码：
{% coderay lang:cplusplus linenos:true %}
class loopThread:public Thread{
	public:
		void addTask(Task * task,bool needSignal);
		Task * popTask();
	private:
		void threadRunloop();
		MessageQueue taskQueue;
		Mutex operationsMutex;
		Condition operationsCond;
	}
	
	void loopThread::addTask(Task * task,bool needSignal){
		if(!task)
			return;
			
		operationsMutex.lock();
		taskQueue.append(task);
		operationsMutex.unLock();
		
		if(needSignal){
			operationsCond.signal();
		}
	}
    
    Task * loopThread::popTask(){
    	Task * task = taskQueue.popTaskByPriority();
    	return task;
    }
    
    void loopThread::threadRunloop(){
    	
    	Task * currentTask = null; 
    	//run tasks 
    	bool stop = false;
    	while(!stop){
    	
    		operationsMutex.lock()
    		
    		//1.check taskQueue have pending task or not
    		while(!taskQueue.size()){
    			/*这里使用while而不是if，是因为有可能存在多个
    			loopThread线程都在等待，但唤醒后，
    			只能有一个线程进行下面的执行，导致条件发生改变，
    			所以其他线程在接着执行时，任务需要条件判断一次。
    			这就是所谓的“惊群效应”*/
    			operationsCond.wait(operationsMutex)
    		}
    		
    		//2.get a task from queue
    		
    		currentTask = popTask();
    		
    		operationsMutex.unLock();
    	
    		//3.excute this task;
    		if(currentTask){
    			currentTask->run();
    		}
    	}
    }{% endcoderay %}

###二. 互斥变量Mutex和条件变量Condition的使用场景
mutex和condition对象主要是pthread接口的封装，最终调用的POSIX api 如下：
	{% coderay lang:c %}
	//1.Mutex   
	pthread_mutex_init  
	pthread_mutex_lock  
	pthread_mutex_trylock  
	pthread_mutex_unlock
	ThreadCondition 
	//2.条件等待:  
	pthread_cond_init  
	pthread_cond_wait(&conidtion,&mutex);  
	phtread_cond_timewait	  
	phtread_cond_signal	
	{% endcoderay %}
	
####1. pthread_cond_wait()的使用过程说明
	{% coderay lang:c %}  
	pthread_mutex_lock(&mutex);
	//对临界资源的操作 
	pthread_cond_wait(&cond, &mutex);
	//对临界资源的操作 
	pthread_mutex_unlock(&mutex);
	{% endcoderay %}
上面的代码**锁**的变化流程如下 :
	
mutex.lock()->**mutext.unlock()-->mutex.lock()**-->mutext.unlock()
	
中间两步是在`pthread_cond_wait`中完成的。pthread_cond_wait把当前线程**放入阻塞队列后**，会进行一个**mutex unlock**释放锁，此后其他线程就可以访问临界资源了,例如这个临界资源可能是一个任务队列(也可是是其他条件变量)，其他线程就可以往任务队列里面添加新的任务。此时当前thread处于wait状态等待条件cond发生改变。当其他线程添加任务到队列后就会进行唤醒操作，`phtread_cond_signal(&cond)`就唤醒了当前线程，当前线程在**pthread_cond_wait返回前**，里面会进行一个mutex lock操作以重新获取锁。

	 

####2. pthread_cond_wait()存在的“惊群效应”  
当有多个线程处于pthread_cond_wait等同一个cond时，当cond条件满足时，这个多个线程都可能被唤醒，但只有其中一个线程得到了锁（得到下面临界资源的执行权限)，其他线程等待。当该线程执行完毕后，临界资源的执行条件可能发生了改变（例如上述任务队列为空了），此时其他线程要往下执行的话需要再次判断条件，这里就使用了`while`循环。
	{% coderay lang:c %}
	pthread_mutex_lock(&mutex);
	while(临界资源条件不满足){
	pthread_cond_wait(&cond,&mutex);
	}
	....
	..对临界资源的操作
	....
	phtread_mutext_unlock(&mutex);	
	{% endcoderay %}
可以参考[这里](http://hi.baidu.com/yangyangye2008/item/454628450f8f7890833ae1a8)


###三. 存在消息队列线程的使用案例

####1. thread线程类对外提供proxy接口

在thread线程类接口供外部使用时，建立一个ThreadProxy是一个良好的方式。


[CCScopedThreadProxy](http://opensource.apple.com/source/WebCore/WebCore-1640.1/platform/graphics/chromium/cc/CCScopedThreadProxy.h)的设计：线程API通过创建一个proxy给外部线程使用，但线程自己可以关闭这个使用。下面可以停止某个外部对象(就是个threadproxy)使用当前的thread。
`CCScopedThreadProxy::shutdown()`

{% coderay lang:cplusplus linenos:true CCScopedThreadProxy.h http://opensource.apple.com/source/WebCore/WebCore-1640.1/platform/graphics/chromium/cc/CCScopedThreadProxy.h  linek %} 
#ifndef CCScopedThreadProxy_h
#define CCScopedThreadProxy_h

#include "cc/CCProxy.h"
#include "cc/CCThreadTask.h"
#include <wtf/ThreadSafeRefCounted.h>

namespace WebCore {

// This class is a proxy used to post tasks to an target thread from any other thread. The proxy may be shut down at
// any point from the target thread after which no more tasks posted to the proxy will run. In other words, all
// tasks posted via a proxy are scoped to the lifecycle of the proxy. Use this when posting tasks to an object that
// might die with tasks in flight.
//
// The proxy must be created and shut down from the target thread, tasks may be posted from any thread.
//
// Implementation note: Unlike ScopedRunnableMethodFactory in Chromium, pending tasks are not cancelled by actually
// destroying the proxy. Instead each pending task holds a reference to the proxy to avoid maintaining an explicit
// list of outstanding tasks.
class CCScopedThreadProxy : public ThreadSafeRefCounted<CCScopedThreadProxy> {
public:
    static PassRefPtr<CCScopedThreadProxy> create(CCThread* targetThread)
    {
        ASSERT(currentThread() == targetThread->threadID());
        return adoptRef(new CCScopedThreadProxy(targetThread));
    }
    
    // Can be called from any thread. Posts a task to the target thread that runs unless
    // shutdown() is called before it runs.
    void postTask(PassOwnPtr<CCThread::Task> task)
    {
        ref();
        m_targetThread->postTask(createCCThreadTask(this, &CCScopedThreadProxy::runTaskIfNotShutdown, task));
    }
    
    void shutdown()
    {
        ASSERT(currentThread() == m_targetThread->threadID());
        ASSERT(!m_shutdown);
        m_shutdown = true;
    }

private:
    explicit CCScopedThreadProxy(CCThread* targetThread)
        : m_targetThread(targetThread)
        , m_shutdown(false) { }

    void runTaskIfNotShutdown(PassOwnPtr<CCThread::Task> popTask)
    {
        OwnPtr<CCThread::Task> task = popTask;
        // If our shutdown flag is set, it's possible that m_targetThread has already been destroyed so don't
        // touch it.
        if (m_shutdown) {
            deref();
            return;
        }
        ASSERT(currentThread() == m_targetThread->threadID());
        task->performTask();
        deref();
    }

    CCThread* m_targetThread;
    bool m_shutdown; // Only accessed on the target thread
};
}
#endif
{% endcoderay %}
	
 

####2. 简单的线程队列案例:  

* chromium [CCThread](http://www.opensource.apple.com/source/WebCore/WebCore-1298.39/platform/graphics/chromium/cc/CCThread.cpp)
* android42 [webkit TexturesGenerator.cpp的实现](https://android.googlesource.com/platform/external/webkit/+/42326004062d6b846c3050ad03a1e80fa9db425c/Source/WebCore/platform/graphics/android/rendering/TexturesGenerator.cpp)

####3. 参考
* chromium [CCThreadProxy](https://gitorious.org/webkit-clutter/webkit-clutter/source/804b17d000553df4f01db334f97c9bd01e416716:Source/WebCore/platform/graphics/chromium/cc/CCThreadProxy.cpp#L82-348)
* chromium [CCScopedThreadProxy](http://opensource.apple.com/source/WebCore/WebCore-1640.1/platform/graphics/chromium/cc/CCScopedThreadProxy.h)