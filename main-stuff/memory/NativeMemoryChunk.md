##NativeMemoryChunk
NativeMemoryChunkPool将调用NativeMemoryChunk的构造方法创建一块指定大小的Native内存：
```
  public NativeMemoryChunk(final int size) {
    Preconditions.checkArgument(size > 0);
    mSize = size;
    mNativePtr = nativeAllocate(mSize);
    mClosed = false;
  }
```
NativeMemoryChunk是调用native方法nativeAllocate()申请内存空间的，而close方法也是调用native方法nativeFree()释放内存空间的：
```
  public synchronized void close() {
    if (!mClosed) {
      mClosed = true;
      nativeFree(mNativePtr);
    }
  }
```
此外，还有以下几种对内存块进行操作的方法：   
1. 读取给定偏移处的内存值
```
  public synchronized byte read(int offset) {
    Preconditions.checkState(!isClosed());
    Preconditions.checkArgument(offset >= 0);
    Preconditions.checkArgument(offset < mSize);
    return nativeReadByte(mNativePtr + offset);
  }
```
2. 将内存中从指定偏移开始若干个字节读到byteArray中的指定偏移位置
```
  public synchronized int read(
      final int nativeMemoryOffset,
      final byte[] byteArray,
      final int byteArrayOffset,
      final int count) {
    Preconditions.checkNotNull(byteArray);
    Preconditions.checkState(!isClosed());
    final int actualCount = adjustByteCount(nativeMemoryOffset, count);
    checkBounds(nativeMemoryOffset, byteArray.length, byteArrayOffset, actualCount);
    nativeCopyToByteArray(mNativePtr + nativeMemoryOffset, byteArray, byteArrayOffset, actualCount);
    return actualCount;
  }
```
其中，adjustByteCount()将保证对Native内存的访问不会超出内存块边界，checkBounds()将检查Native内存块和byteArray的边界。
3. 将byteArray中的指定偏移位置若干个字节写到内存中
```
  public synchronized int write(
      int nativeMemoryOffset,
      final byte[] byteArray,
      int byteArrayOffset,
      int count) {
    Preconditions.checkNotNull(byteArray);
    Preconditions.checkState(!isClosed());
    final int actualCount = adjustByteCount(nativeMemoryOffset, count);
    checkBounds(nativeMemoryOffset, byteArray.length, byteArrayOffset, actualCount);
    nativeCopyFromByteArray(
        mNativePtr + nativeMemoryOffset,
        byteArray,
        byteArrayOffset,
        actualCount);
    return actualCount;
  }
```
4. NativeMemoryChunk之间内容的复制
```
public void copy(
      final int offset,
      final NativeMemoryChunk other,
      final int otherOffset,
      final int count) {
    Preconditions.checkNotNull(other);
//若目标内存块与源内存块指向同一块内存，那么不需要进行任何操作
    // Case 1: other memory chunk == this memory chunk
    if (other.mNativePtr == mNativePtr) {
      Preconditions.checkArgument(false);
    }
    //若目标内存块的地址小于源内存块的大小，那么先获取目标内存块锁，再获取源内存块锁
    // Case 2: other memory chunk < this memory chunk
    if (other.mNativePtr < mNativePtr) {
      synchronized (other) {
        synchronized (this) {
          doCopy(offset, other, otherOffset, count);
        }
      }
      return;
    }
    //若目标内存块的地址大于源内存块的大小，那么先获取源内存块锁，再获取目标内存块锁
    // Case 3: other memory chunk > this memory chunk
    synchronized (this) {
      synchronized (other) {
        doCopy(offset, other, otherOffset, count);
      }
    }
  }
```
其中，因为必须对Native内存块进行同步控制，以防止多线程下对内存块的访问冲突，而Copy操作需要同时同步源内存块和目标内存块，在等待源内存块和目标内存块锁的过程中有可能发生死锁，比如线程1发起Copy A to B，线程2发起Copy B to C，线程3发起Copy C to A，那么当线程1可能获取了A的锁，线程2可能获取了B的锁，线程3可能获取了C的锁，但各自无法获取另一个内存块的锁而将造成死锁。++**这里的解决方法是按低地址到高地址获取锁，并且必须一次性申请所有锁资源，即有序资源分配法，来破坏死锁产生的环路条件。**++

#####扩展阅读
native内存的操作并不复杂，如nativeAllocate()只是调用了malloc()尝试分配内存并返回jlong指针
```
static jlong NativeMemoryChunk_nativeAllocate(
    JNIEnv* env,
    jclass clzz,
    jint size) {
  UNUSED(clzz);
  void* pointer = malloc(size);
  if (!pointer) {
    (*env)->ThrowNew(env, jRuntimeException_class, "could not allocate memory");
    return 0;
  }
  return PTR_TO_JLONG(pointer);
}
```
此外，NativeMemoryChunk_nativeFree、NativeMemoryChunk_nativeMemcpy分别使用free()和memcpy()来进行内存的释放和拷贝，NativeMemoryChunk_nativeReadByte将返回内存块的jlong指针。
所不同的是与Java字节数组相关的读写操作需要借助JNIEnv的SetByteArrayRegion和GetByteArrayRegion调用来实现。
```
static void NativeMemoryChunk_nativeCopyToByteArray(
    JNIEnv* env,
    jclass clzz,
    jlong lpointer,
    jbyteArray byteArray,
    jint offset,
    jint count) {
  UNUSED(clzz);
  (*env)->SetByteArrayRegion(
      env,
      byteArray,
      offset,
      count,
      JLONG_TO_PTR(lpointer));
}
```
