---
date: '2025-07-24T10:30:00-04:00'
draft: false
title: 'Building Thread-Safe Event Systems in Go: A k6 waitForResponse Case Study'
description: "Walk through an example PR I merged into Grafana Lab's k6 repo and see how thread-safety works at scale"
---
# Building Thread-Safe Event Systems in Go: A k6 waitForResponse Case Study

Browser automation tools need to wait for specific network responses—but building this in a concurrent system is trickier than it looks. When testing web applications, you often need to wait for an API call to complete before proceeding with the next action. Without proper synchronization, tests become flaky and unreliable. Recently, I contributed the `waitForResponse` API to k6's browser module, architecting a thread-safe event system that handles hundreds of concurrent waiters without race conditions or resource leaks.

## The Problem: Reflection Hell

The initial approach used k6's generic event handler system with reflection-based cleanup. Here's what the problematic code looked like:

```go
func (p *Page) WaitForResponse(...) {
    // Manually add handler to generic event system
    p.eventHandlers[EventPageResponseCalled] = append(
        p.eventHandlers[EventPageResponseCalled], handler)
    
    // Complex reflection-based cleanup
    defer func() {
        for i, h := range handlers {
            if reflect.ValueOf(h).Pointer() == reflect.ValueOf(handler).Pointer() {
                handlers[i] = handlers[len(handlers)-1]
                p.eventHandlers[EventPageResponseCalled] = handlers[:len(handlers)-1]
                break
            }
        }
    }()
}
```

This approach had serious issues: race conditions when multiple goroutines modified the handler slice, resource leaks when cleanup failed, and reflection-based pointer comparisons that were fragile and hard to debug. The generic event system wasn't designed for the specific needs of blocking operations with automatic cleanup.

## Design Requirements: What Success Looks Like

The replacement system needed to handle several critical requirements. First, thread-safe concurrent access—multiple goroutines should be able to register waiters simultaneously without corruption. Second, automatic resource cleanup—no manual tracking of handler instances or complex reflection logic. Third, efficient pattern matching for URL filtering using both string matches and regex patterns. Finally, seamless integration with k6's existing context cancellation and timeout systems. The solution needed to handle edge cases like timeouts and context cancellation gracefully while maintaining high performance under load.

## The Core Architecture: Dedicated Event Handler

Instead of forcing blocking operations into a generic event system, I designed a dedicated `ResponseEventHandler`:

```go
type ResponseEventHandler struct {
    mu           sync.RWMutex
    activeWaiters map[string]*responseWaiter
    nextWaiterID  int64
}

type responseWaiter struct {
    id           string
    matcher      URLMatcher
    responseChan chan *Response
    ctx          context.Context
    cancel       context.CancelFunc
}
```

This separation of concerns eliminates the complexity of generic handlers while providing exactly what blocking operations need: unique waiter identification, context-based lifecycle management, and efficient lookup patterns. Each waiter gets its own dedicated channel and context, making cleanup deterministic and race-free. Furthermore, it built ontop of the existing process in `k6` to handle browser events without relying on a generic structure. 

{{< figure src="wait-for-response-architecture.png" width="100%" alt="low coupling high cohesion" >}}

As you can see, we create channels for each waiter that will be notified once a processed browser event is seen to match against it.

## Thread Safety: Lock-Free Notification Pattern

The key insight was separating read operations from write operations to minimize lock contention:

```go
func (reh *ResponseEventHandler) processResponse(response *Response) {
	if response == nil {
		return
	}

	reh.mu.RLock()
	defer reh.mu.RUnlock()

	for _, waiter := range reh.activeWaiters {
		select {
		case <-waiter.ctx.Done():
			continue
		default:
		}

		matched, err := waiter.matcher(response.URL())
		if err == nil && matched {
			select {
			case waiter.responseChan <- response:
			case <-waiter.ctx.Done():
			}
		}
	}
}
```

This pattern allows high-concurrency reads while ensuring notifications happen outside the critical section. The non-blocking send prevents deadlocks when contexts are cancelled mid-notification, and the RWMutex optimizes for the common case of many concurrent response processors with fewer waiter registrations.

## Resource Management: Context-Driven Cleanup

Rather than manual cleanup tracking, the system leverages Go's context cancellation for automatic resource management:

```go
func (p *Page) WaitForResponse(urlPattern string, opts *WaitForResponseOptions) (*Response, error) {
    ctx, cancel := context.WithTimeout(p.ctx, opts.Timeout)
    defer cancel()
    
    matcher, err := urlMatcher(urlPattern)
    if err != nil {
        return nil, fmt.Errorf("parsing URL pattern: %w", err)
    }
    
    return p.responseEventHandler.waitForMatch(ctx, matcher)
}
```

When the context expires or gets cancelled, cleanup happens automatically through the defer chain. No need to hunt for specific handler instances in slices or complex reflection logic. The waiter removes itself from the active map when its context completes, ensuring zero resource leaks even under high load or unexpected cancellations. We can find this explicitly in the `waitForMatch` function that recieves that page context and builds out the waiter:

```go
func (reh *ResponseEventHandler) waitForMatch(ctx context.Context, matcher URLMatcher) (*Response, error) {
	waiterContext, waiterCancel := context.WithCancel(ctx)

	waiter := &responseWaiter{
		matcher:      matcher,
		responseChan: make(chan *Response),
		ctx:          waiterContext,
		cancel:       waiterCancel,
	}

	// Add waiter
	id := atomic.AddInt64(&reh.nextWaiterID, 1)
	reh.mu.Lock()
	reh.activeWaiters[id] = waiter
	reh.mu.Unlock()

	defer func() {
		waiterCancel()
		// Remove waiter
		reh.mu.Lock()
		delete(reh.activeWaiters, id)
		reh.mu.Unlock()
	}()

	select {
	case response := <-waiter.responseChan:
		return response, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}
```
Here, we use the higher level page context and listen for the context to cancel in a `select`. As soon as it's canceled, we lock the `activeWaiter` and safely remove it, or if a match is found through the Event Processing System, the waiter is also removed. 

## Real-World Testing: Edge Cases and Performance

Integration testing revealed several edge cases that the architecture handled gracefully. Concurrent timeouts with overlapping response patterns worked without interference. Context cancellation during high-volume response processing didn't cause goroutine leaks. The system maintained consistent performance with 100+ concurrent waiters, and memory usage remained stable over extended test runs. Code review feedback led to switching from buffered to unbuffered channels, eliminating potential synchronization smells and making the concurrency model clearer. The reviewer's attention to detail helped identify potential panic scenarios that were addressed through proper channel lifecycle management.

## Key Takeaways: Patterns That Scale

Three main lessons emerged from this implementation. 

1. Dedicated data structures often outperform generic solutions in concurrent systems—the complexity savings and performance benefits justify the additional code. 
2. Proper Go concurrency patterns (RWMutex for read-heavy workloads, context-based cancellation, non-blocking sends) prevent entire classes of bugs before they happen. 
3. Resource lifecycle management becomes much simpler when you design for it upfront rather than retrofitting cleanup logic. Taking advantage of existing tools, such as Go's native `context`, is how you build reliable code.

This approach improved maintainability significantly—the new code is easier to understand, debug, and extend. The pattern is now setup to be used across multiple k6 browser APIs where blocking operations are needed.

## Impact and Future: Building on Solid Foundations

The PR was successfully merged into k6's, adding a critical piece of browser automation functionality that developers had been requesting. The thread-safe architecture enables reliable testing of applications with complex network timing requirements, particularly important for load testing scenarios where network conditions vary. This foundation also sets up future enhancements like predicate function support (`await page.waitForResponse(response => response.status() === 200)`) and more sophisticated filtering options. Most importantly, the architectural patterns demonstrated here—dedicated concurrent data structures, context-driven resource management, and lock-free notification systems—are applicable far beyond browser automation to any Go system that needs reliable event handling under concurrent load.