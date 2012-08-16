---
layout: post
title: "Circuit Breaker Pattern Revisited"
description: ""
category: 
tags: []
---

The circuit breaker pattern can be really useful when building a system based on distributed services. It helps to keep the whole system healthy even when some of its sub-services have troubles. The first proposal of the circuit breaker for this kind on problem I found in Michael Nygard's great book, [Release It!](http://www.pragprog.com/titles/mnee/release-it). The pattern doesn't require sophisticated implementation. But you don't even have to do it because there are some ready solutions like the Spring-based [kite-lib](http://code.google.com/p/kite-lib/). Let's check what's inside.

The core logic of the circuit breaker extracted from the library:

    public class CircuitBreakerTemplate {
      public enum State { CLOSED, OPEN, HALF_OPEN };
      private int exceptionThreshold = 5; 
      private long timeout = 30000L;
      private volatile State state = State.CLOSED;
      private final AtomicInteger exceptionCount = new AtomicInteger(); 
      private volatile long attemptResetAfter = Long.MAX_VALUE;

      public State getState() { 
        if (state == State.OPEN) {
          if (System.currentTimeMillis() >= attemptResetAfter) { 
            this.state = State.HALF_OPEN;
          }
        } 
        return state;
      }

      public void reset() {
        this.state = State.CLOSED; 
        this.exceptionCount.set(0);
      }

      public void trip() {
        this.state = State.OPEN; 
        this.attemptResetAfter = System.currentTimeMillis() + timeout;
      }

      public <T> T execute(CircuitBreakerCallback&lt;T&gt; action) throws Exception { 
        final State currState = getState(); 
        switch (currState) {
          case CLOSED:
            try {
              T value = action.doInCircuitBreaker(); 
              this.exceptionCount.set(0); 
              return value;
            } catch (Exception e) {
              if (exceptionCount.incrementAndGet() >= exceptionThreshold) { trip(); } 
              throw e;
            }
          case OPEN: throw new RuntimeException("CircuitOpenException");
          case HALF_OPEN:
            try {
              T value = action.doInCircuitBreaker(); 
              reset();
              return value; 
            } catch (Exception e) {
              trip(); 
              throw e;
            }
          default: throw new IllegalStateException("Unknown state: " + currState);
        }
      }
    }

    interface CircuitBreakerCallback&lt;T&gt; {
      T doInCircuitBreaker() throws Exception;
    }

The code is copied from the library - just simplified a little, without Spring and monitoring (JMX) stuff.

It just forwards requests to the target service as long as the service is up which means the number of allowed failures isn't reached. When the configured number of allowed failures is reached the circuit is switched to OPEN state for some defined time (timeout). After the timeout the circuit breaker tries access the service once again and in case of success the state is switched to CLOSED. Otherwise it's OPEN again. It's really basic logic and implementation useful in 95% of cases. But we are free to think about different scenarios as well. Maybe our service sometimes is really slow for quite long time e.g. a few days. In such case the above implementation will access the service every defined period of time making that overall system response slow. Of course we can tune all the parameters through the JMX but it's manual work. Maybe the solution which instead of periodical checking notifies us when the service is fast again would be better?

### Threads to rescue

Instead of using timeout for implementing scenario when the circuit is OPEN we can use a background thread. The thread can be started when the number of allowed failures is reached (like in previous implementation) to monitor the service. In the meantime the attempts to access the service can be rejected with the original exception (the last handled exception from the service). When the thread decides that the service is healthy again the circuit can be closed.

    public class CircuitBreaker implements InvocationHandler {
      private int maxFailures = 3;
      private AtomicInteger failures = new AtomicInteger(0);
      private volatile Throwable lastThrowable;
      private volatile Thread monitor;
      private RemoteServiceClient remoteServiceClient;
      private long timeoutBetweenChecks = 2000;

      public Object invoke(Object proxy, final Method method, final Object[] args) throws Throwable {
        if (lastThrowable != null) {
          throw lastThrowable;
        }
        try {
          return method.invoke(remoteServiceClient, args);
        } catch (Throwable t) {
          if (failures.incrementAndGet() == maxFailures) {
            monitor = new Thread(new Runnable() {
              boolean fails = true;
              public void run() {
                while (fails) {
                  lastThrowable = t.getCause();
                  try {
                    Thread.sleep(timeoutBetweenChecks);
                    method.invoke(remoteServiceClient, args);
                    failures.set(0);
                    lastThrowable = null;
                    fails = false;
                  } catch (Throwable t) {}
                }
              }
            });
            monitor.start();
          }
          throw t.getCause();
        }
      }
    }

### Safety vs Performance

The environment for the solution is multithreaded by its nature. In most cases it's a web application aggregating content from external services. This kind of application has thread pool used to process request handlers. So the circuit breaker needs to be prepared to be executed by many threads at the same time. Also, it must be transparent for the whole system especially in terms of performance and liveness. The biggest risk with the solution (in both implementations) is the situation that the circuit is OPEN even in the case when the service is healthy again. That's something we have to prevent for sure. At the same time we can not make too much locking especially for the case when everything is fine.

Both solutions are based on similar pattern - global mutable variable holding the state of the circuit. What we must be sure here is the fact that modifications to this state must be propagated through the CPU caches to main memory so that other threads can see it. Because the change of this variable isn't based on the previous value (read-modify-write) we can do it with quite lightweight <code>volatile</code>. We have also additional global variable - failures counter. In this case we couldn't use <code>volatile</code> for it because <code>failures++</code> isn't atomic (read-modify-write). So, <code>AtomicInteger.incrementAndGet</code> is used which is a little more expensive (from the performance point of view) then <code>volatile</code> but much less then any explicit locking.

It must be said that the second implementation is more risky because of this additional monitor thread which is the only one place where once opened circuit can be closed. So in case the monitor thread dies we must have some fallback solution in place. Also it might happen that the monitor thread failed at startup, so we should have here some logic to reset the failures counter after some time/value.

### Final notes

- The solution proposed by me isn't the standard way of implementing the pattern, so use it only if you really need it. The original one should be sufficient in most cases.
- The circuit breaker can be really useful but should not be used to hides problems in your system/architecture. You must be sure that the case when the sub-service is down or slow is really normal case. It shouldn't be a workaround for some problems of unstable services.
