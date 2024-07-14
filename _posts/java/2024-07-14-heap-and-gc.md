---
layout: post
title:  "JVM Heap과 GC를 다른 관점에서 바라보기"
date:   2024-07-14 17:00:00 +0900
author: leeyh0216
tags:
- java
- gc
- heap
---

# 개요

Garbage Collection을 검색해보면 대부분의 글이 특정 GC(CMS, G1, Z 등)의 배경이 되는 이론(ex. Generational Collection Theory)이나, 알고리즘(ex. Mark-Sweep), 튜닝 등에 대한 내용을 다루고 있다. 그리고 해당 이론에 근거하여 Heap 메모리의 구조를 설명하다보니, Heap 영역을 Eden, Old, Perm 등으로 나누어 생각하는 것이 일반화된 것 같다.

최근에 JVM 밑바닥까지 파헤치기 라는 책을 읽다가 다음과 같은 문장을 읽었다.

> 이 영역 구분(Eden, Survivor 등)은 가비지 컬렉터들의 일반적인 특성 또는 설계 방식일 뿐, 반드시 이 형태로 메모리를 구성해야 한다는 뜻은 아니라는 점이다. <<자바 가상 머신 명세>>의 자바 힙 절에는 세부 영역 구분에 관한 이야기 자체가 없다

실제로 [Oracle의 문서 중 Heap에 관한 설명](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.5.3)을 읽어보면, "단순히 클래스의 객체와 배열들이 위치한 메모리 공간"이라고만 설명되어 있는 것을 확인할 수 있다.

그리고 대부분의 글들이 사용되지 않는 객체(메모리)의 회수 방식에만 초점을 맞추고, TLAB(Thread Local Allocation Buffer)와 같은 기술을 별도로 설명하다보니 Heap 메모리에 대한 이해가 굉장히 파편화 되어 있는 경우가 있는데(나조차도), 이번 글이 파편화된 지식을 연결할 수 있는 발판이 되면 좋겠다는 생각을 한다.

# OpenJDK 코드 기반으로 살펴보기

> 아래 내용은 [OpenJDK](https://github.com/openjdk/jdk)의 `jdk-24+6` Tag를 기반으로 작성하였습니다.
>
> 실제 메모리 관리나 구현에 초점을 맞추기보다, 코드와 클래스의 구조를 기반으로 Heap 메모리와 GC에 대해 이해하는 것을 목표로 합니다.

OpenJDK 코드 중 메모리 관리에 대한 전체적인 레이아웃을 명시한 코드는 [`src/hotspot/share/memory/allocation.hpp`](https://github.com/openjdk/jdk/blob/jdk-24%2B6/src/hotspot/share/memory/allocation.hpp)이다. JVM 메모리 구조에 관련된 클래스들을 설명되어 있으며, 우리가 주목해야할 부분은 `CHeapObj` 이다.

```
// For objects allocated in the resource area (see resourceArea.hpp).
// - ResourceObj
//
// For objects allocated in the C-heap (managed by: free & malloc and tracked with NMT)
// - CHeapObj
//
// For objects allocated on the stack.
// - StackObj
//
// For classes used as name spaces.
// - AllStatic
//
// For classes in Metaspace (class data)
// - MetaspaceObj
```

이 `CHeapObj`의 세부 인터페이스와 기본적인 뼈대를 구현한 클래스가 [`src/hotspot/share/gc/shared/collectedHeap.hpp`](https://github.com/openjdk/jdk/blob/jdk-24%2B6/src/hotspot/share/gc/shared/collectedHeap.hpp)에 위치한 `CollectedHeap` 클래스이다.

## Heap 메모리의 추상 클래스 `CollectedHeap`

이 클래스의 주석을 보면 아래와 같이 기술되어 있는 것을 확인할 수 있다.

```
// A "CollectedHeap" is an implementation of a java heap for HotSpot.  This
// is an abstract class: there may be many different kinds of heaps.  This
// class defines the functions that a heap must implement, and contains
// infrastructure common to all heaps.
```

`CollectedHeap`은 HotSpot을 위한 Java Heap의 구현체(추상 클래스)이며, 세부 구현은 Heap의 종류마다 다르다. 이 클래스는 Heap이 반드시 구현해야 하는 함수를 정의하고, 모든 Heap이 사용할 수 있는 공통 인프라(코드)를 구현하고 있다.

```
// CollectedHeap
//   SerialHeap
//   G1CollectedHeap
//   ParallelScavengeHeap
//   ShenandoahHeap
//   ZCollectedHeap
class CollectedHeap : public CHeapObj<mtGC> {
  // Create a new tlab. All TLAB allocations must go through this.
  // To allow more flexible TLAB allocations min_size specifies
  // the minimum size needed, while requested_size is the requested
  // size based on ergonomics. The actually allocated size will be
  // returned in actual_size.
  virtual HeapWord* allocate_new_tlab(size_t min_size,
                                      size_t requested_size,
                                      size_t* actual_size) = 0;

  // Raw memory allocation facilities
  // The obj and array allocate methods are covers for these methods.
  // mem_allocate() should never be
  // called to allocate TLABs, only individual objects.
  virtual HeapWord* mem_allocate(size_t size,
                                 bool* gc_overhead_limit_was_exceeded) = 0;

  // Perform a collection of the heap; intended for use in implementing
  // "System.gc".  This probably implies as full a collection as the
  // "CollectedHeap" supports.
  virtual void collect(GCCause::Cause cause) = 0;

  void print_heap_before_gc();
  void print_heap_after_gc();
}
```

`CollectedHeap`의 선언부와 주요 함수들을 뽑아낸 간단한 버전은 위와 같다.

```
// CollectedHeap
//   SerialHeap
//   G1CollectedHeap
//   ParallelScavengeHeap
//   ShenandoahHeap
//   ZCollectedHeap
```

우선 선언부에는 `CollectedHeap`을 상속하는 클래스(Heap)들의 목록이 위치하고 있다. 우리가 알고 있는 GC의 이름들이 접두사로 붙은 형태의 Heap 클래스들이 `CollectedHeap`을 상속하는 것을 확인할 수 있다.

```
  // Create a new tlab. All TLAB allocations must go through this.
  // To allow more flexible TLAB allocations min_size specifies
  // the minimum size needed, while requested_size is the requested
  // size based on ergonomics. The actually allocated size will be
  // returned in actual_size.
  virtual HeapWord* allocate_new_tlab(size_t min_size,
                                      size_t requested_size,
                                      size_t* actual_size) = 0;

  // Raw memory allocation facilities
  // The obj and array allocate methods are covers for these methods.
  // mem_allocate() should never be
  // called to allocate TLABs, only individual objects.
  virtual HeapWord* mem_allocate(size_t size,
                                 bool* gc_overhead_limit_was_exceeded) = 0;
```

그리고 위와 같이 `allocate_new_tlab`, `mem_allocate` 함수의 선언이 위치한 것을 볼 수 있는데, 이는 결국 GC 종류에 따라 메모리의 할당 방식도 달라진다는 것을 암시한다.

사실 GC를 회수의 영역에서만 바라보면 나올 수 밖에 없는 질문이 "그럼 Garbage Collector는 어떻게 Heap 메모리 레이아웃과 객체의 메모리 위치, 관계 등을 파악할 수 있는거지?" 이다. 이에 대한 답은 위의 코드와 같이 GC 알고리즘 내에 객체에 대한 메모리 할당이 포함되어 있기 때문이라고 할 수 있다. 또한 TLAB도 메모리의 할당에 연관된 기술이기 때문에, 메모리 할당을 담당하는 GC에 연관되어 있다고 말할 수 있다.

```
  // Perform a collection of the heap; intended for use in implementing
  // "System.gc".  This probably implies as full a collection as the
  // "CollectedHeap" supports.
  virtual void collect(GCCause::Cause cause) = 0;
```

그리고 위와 같이 메모리 회수를 수행하는 `collect` 함수가 선언되어 있는 것을 확인할 수 있다.

```
  void print_heap_before_gc();
  void print_heap_after_gc();
```

그리고 위와 같은 유틸리티 성 함수들 또한 선언되어 있는 것을 확인할 수 있다.

`CollectedHeap` 클래스만 보아도 Heap 메모리의 구조는 결국 GC 알고리즘에 직접적으로 연관되어 있으며, GC는 단순히 메모리 회수만이 아닌 할당에도 관여한다는 것을 알 수 있다.

추가로 [`src/hotspot/share/gc/shared/collectedHeap.inline.hpp`](https://github.com/openjdk/jdk/blob/jdk-24%2B6/src/hotspot/share/gc/shared/collectedHeap.inline.hpp)에 다음과 같이 Inline 함수들이 정의되어 있는 것을 볼 수 있는데, 아래에서 객체에 대한 메모리를 할당하는 과정에서 사용되니, 간단히 봐두면 좋을 것 같다.

```
inline oop CollectedHeap::obj_allocate(Klass* klass, size_t size, TRAPS) {
  ObjAllocator allocator(klass, size, THREAD);
  return allocator.allocate();
}

inline oop CollectedHeap::array_allocate(Klass* klass, size_t size, int length, bool do_zero, TRAPS) {
  ObjArrayAllocator allocator(klass, size, length, do_zero, THREAD);
  return allocator.allocate();
}

inline oop CollectedHeap::class_allocate(Klass* klass, size_t size, TRAPS) {
  ClassAllocator allocator(klass, size, THREAD);
  return allocator.allocate();
}
```

### 예시 살펴보기: `G1CollectedHeap`

그럼 `CollectedHeap`을 구현한 클래스 중 하나인 [`G1CollectedHeap`](https://github.com/openjdk/jdk/blob/jdk-24%2B6/src/hotspot/share/gc/g1/g1CollectedHeap.cpp)을 살펴보도록 하자. 어차피 세부 구현에 대해 하나하나 뜯어볼 생각은 없기 때문에 실제로 구현이 되어 있는지에 대해 살펴보기 위한 목적이다.

```
HeapWord* G1CollectedHeap::allocate_new_tlab(size_t min_size,
                                             size_t requested_size,
                                             size_t* actual_size) {
  assert_heap_not_locked_and_not_at_safepoint();
  assert(!is_humongous(requested_size), "we do not allow humongous TLABs");

  return attempt_allocation(min_size, requested_size, actual_size);
}

HeapWord*
G1CollectedHeap::mem_allocate(size_t word_size,
                              bool*  gc_overhead_limit_was_exceeded) {
  assert_heap_not_locked_and_not_at_safepoint();

  if (is_humongous(word_size)) {
    return attempt_allocation_humongous(word_size);
  }
  size_t dummy = 0;
  return attempt_allocation(word_size, word_size, &dummy);
}

inline HeapWord* G1CollectedHeap::attempt_allocation(size_t min_word_size,
                                                     size_t desired_word_size,
                                                     size_t* actual_word_size) {
  assert_heap_not_locked_and_not_at_safepoint();
  assert(!is_humongous(desired_word_size), "attempt_allocation() should not "
         "be called for humongous allocation requests");

  HeapWord* result = _allocator->attempt_allocation(min_word_size, desired_word_size, actual_word_size);

  if (result == nullptr) {
    *actual_word_size = desired_word_size;
    result = attempt_allocation_slow(desired_word_size);
  }

  assert_heap_not_locked();
  if (result != nullptr) {
    assert(*actual_word_size != 0, "Actual size must have been set here");
    dirty_young_block(result, *actual_word_size);
  } else {
    *actual_word_size = 0;
  }

  return result;
}

void G1CollectedHeap::collect(GCCause::Cause cause) {
  try_collect(cause, collection_counters(this));
}

bool G1CollectedHeap::try_collect(GCCause::Cause cause,
                                  const G1GCCounters& counters_before) {
  if (should_do_concurrent_full_gc(cause)) {
    return try_collect_concurrently(cause,
                                    counters_before.total_collections(),
                                    counters_before.old_marking_cycles_started());
  } else if (cause == GCCause::_gc_locker || cause == GCCause::_wb_young_gc
             DEBUG_ONLY(|| cause == GCCause::_scavenge_alot)) {

    // Schedule a standard evacuation pause. We're setting word_size
    // to 0 which means that we are not requesting a post-GC allocation.
    VM_G1CollectForAllocation op(0,     /* word_size */
                                 counters_before.total_collections(),
                                 cause);
    VMThread::execute(&op);
    return op.gc_succeeded();
  } else {
    // Schedule a Full GC.
    return try_collect_fullgc(cause, counters_before);
  }
}
```

위와 같이 `CollectedHeap`의 가상 함수인 `allocate_new_tlab`, `mem_allocate`, `collect` 등을 구현하고 있는 것을 볼 수 있다.

추가로 메모리의 할당과 회수(Garbage Collection)는 각각 `G1Allocator`와 `G1YoungCollector`, `G1FullCollector` 등의 클래스가 담당하고, `G1CollectedHeap`은 이러한 클래스들을 통해 위의 기능들을 수행하는 클래스이다.

## 객체 생성 시 메모리 할당 살펴보기

[`src/hotspot/share/oops/instanceKlass`](https://github.com/openjdk/jdk/blob/jdk-24%2B6/src/hotspot/share/oops/instanceKlass.cpp)의 클래스 파일을 기반으로 객체를 생성하는 역할을 수행한다. 이 중 `allocate_instance` 함수는 클래스의 객체를 생성하는데, 여기서 Heap의 `obj_allocate` 함수를 호출하는 것을 확인할 수 있다.

```
instanceOop InstanceKlass::allocate_instance(TRAPS) {
  assert(!is_abstract() && !is_interface(), "Should not create this object");
  size_t size = size_helper();  // Query before forming handle.
  return (instanceOop)Universe::heap()->obj_allocate(this, size, CHECK_NULL);
}
```

> 참고로 `Universe`의 `heap()`은 전역 함수로써 현 JVM에서 사용 중인 Heap(`CollectedHeap` 인스턴스)를 반환한다.

위의 `CollectedHeap` 설명 중 Inline 함수 중 `obj_allocate`가 호출되는 것이며, 다시 한번 보자면 아래와 같다.

```
inline oop CollectedHeap::obj_allocate(Klass* klass, size_t size, TRAPS) {
  ObjAllocator allocator(klass, size, THREAD);
  return allocator.allocate();
}
```

`ObjAllocator`는 `MemAllocator`를 상속 받은 클래스이며, 위의 `allocate` 함수의 구현을 보면 아래와 같다.

```
HeapWord* MemAllocator::mem_allocate_inside_tlab_fast() const {
  return _thread->tlab().allocate(_word_size);
}

HeapWord* MemAllocator::mem_allocate_inside_tlab_slow(Allocation& allocation) const {
  HeapWord* mem = nullptr;
  ThreadLocalAllocBuffer& tlab = _thread->tlab();

  if (JvmtiExport::should_post_sampled_object_alloc()) {
    tlab.set_back_allocation_end();
    mem = tlab.allocate(_word_size);
    ...
    if (mem != nullptr) {
      return mem;
    }
  }
  ...

  // Allocate a new TLAB requesting new_tlab_size. Any size
  // between minimal and new_tlab_size is accepted.
  size_t min_tlab_size = ThreadLocalAllocBuffer::compute_min_size(_word_size);
  mem = Universe::heap()->allocate_new_tlab(min_tlab_size, new_tlab_size, &allocation._allocated_tlab_size);
  if (mem == nullptr) {
    assert(allocation._allocated_tlab_size == 0,
           "Allocation failed, but actual size was updated. min: " SIZE_FORMAT
           ", desired: " SIZE_FORMAT ", actual: " SIZE_FORMAT,
           min_tlab_size, new_tlab_size, allocation._allocated_tlab_size);
    return nullptr;
  }
  ...
  tlab.fill(mem, mem + _word_size, allocation._allocated_tlab_size);
  return mem;
}

HeapWord* MemAllocator::mem_allocate(Allocation& allocation) const {
  if (UseTLAB) {
    // Try allocating from an existing TLAB.
    HeapWord* mem = mem_allocate_inside_tlab_fast();
    if (mem != nullptr) {
      return mem;
    }
  }
  ...
  if (UseTLAB) {
    // Try refilling the TLAB and allocating the object in it.
    HeapWord* mem = mem_allocate_inside_tlab_slow(allocation);
    if (mem != nullptr) {
      return mem;
    }
  }

  return mem_allocate_outside_tlab(allocation);
}

oop MemAllocator::allocate() const {
  oop obj = nullptr;
  {
    Allocation allocation(*this, &obj);
    HeapWord* mem = mem_allocate(allocation);
    if (mem != nullptr) {
      obj = initialize(mem);
    } else {
      // The unhandled oop detector will poison local variable obj,
      // so reset it to null if mem is null.
      obj = nullptr;
    }
  }
  return obj;
}
```

관련 있는 호출 시퀀스가 많아 모두 포함시켰는데, 결과적으로 `allocate`를 통해 TLAB에 메모리 할당을 요청(TLAB은 스레드 시작 시 초기화되며, 해당 초기화 코드에서 Heap에 TLAB을 위한 공간을 할당)하며, 결국 내부적으로 `CollectedHeap` 구현체들(GC 별)의 함수들이 호출되는 것을 알 수 있다.

# 결론

더 자세히 작성하고 싶었지만, 연관된 클래스들과 코드가 워낙 많아서 오히려 복잡도가 증가할 것 같아 여기까지만 작성하게 되었다.

사실 현업에서는 이러한 이론적인 내용보다 애플리케이션에 적합한 GC는 무엇이고, 그 GC가 어떻게 동작하는지, 어떻게 튜닝하면 Stop The World를 줄일 수 있는지가 더 중요하긴 하다. 다만 단순히 GC를 회수의 목적에서만 바라보면, 할당 부분과 적절히 이어지지 않기 때문에, 한번 쯤 읽어보면 좋을만한 주제라 생각한다.