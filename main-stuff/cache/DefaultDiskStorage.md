##DefaultDiskStorage
&#8195;DefaultDiskStorage是以Supplier模式构造的，**++DefaultDiskStorageSupplier根据磁盘缓冲配置来创建和维护一个DefaultDiskStorage，保存在Supplier的State成员中，供FileCache使用++，以实现对象构造与使用的分离，使得FileCache不用关心缓存根目录的创建和冲突问题**。
```
  public synchronized DiskStorage get() throws IOException {
    if (shouldCreateNewStorage()) {
      // discard anything we created
      deleteOldStorageIfNecessary();
      createStorage();
    }
    return Preconditions.checkNotNull(mCurrentState.storage);
  }
  
    private void createStorage() throws IOException {
    File rootDirectory = new File(mBaseDirectoryPathSupplier.get(), mBaseDirectoryName);
    createRootDirectoryIfNecessary(rootDirectory);
    DiskStorage storage = new DefaultDiskStorage(rootDirectory, mVersion, mCacheErrorLogger);
    mCurrentState = new State(rootDirectory, storage);
  }
```
&#8195;DefaultDiskStorage的构造方法如下：
```
  public DefaultDiskStorage(
      File rootDirectory,
      int version,
      CacheErrorLogger cacheErrorLogger) {
    Preconditions.checkNotNull(rootDirectory);

    mRootDirectory = rootDirectory;
    mVersionDirectory = new File(mRootDirectory, getVersionSubdirectoryName(version));
    mCacheErrorLogger = cacheErrorLogger;
    recreateDirectoryIfVersionChanges();
    mClock = SystemClock.get();
  }
```
&#8195;可以由DiskCacheConfig配置磁盘缓存文件的根目录，并且在根目录下为该缓存文件系统的版本目录。此外，需要注意的是，由于三星的根目录系统在13次创建失败后会产生bug，所以当版本变化而导致目录不匹配时，将销毁原版本目录下的所有内容，并创建新的版本目录。   
&#8195;接下来看DefaultDiskStorage是如何创建和操作资源文件的：   
1. createTemporary()：创建临时资源文件   
&#8195;++**临时资源文件将以ResourceId的编码值为基础散列到各个文件分区中(这里设置了100个文件分区)，这样可以通过资源ID编码均匀散列图片资源文件到各个子目录，以加快查找速度和解决碰撞问题，**++注意这里的临时文件暂时是没有任何内容的。
```
  public FileBinaryResource createTemporary(
      String resourceId,
      Object debugInfo)
      throws IOException {
    // ensure that the parent directory exists
    FileInfo info = new FileInfo(FileType.TEMP, resourceId);
    File parent = getSubdirectory(info.resourceId);
    if (!parent.exists()) {
      mkdirs(parent, "createTemporary");
    }
    try {
      File file = info.createTempFile(parent);
      return FileBinaryResource.createOrNull(file);
    } catch (IOException ioe) {
      //...
      throw ioe;
    }
  }
```
2. updateResource()：图像写入文件   
(1).打开临时文件的文件输出流   
```
    FileBinaryResource fileBinaryResource = (FileBinaryResource)resource;
    File file = fileBinaryResource.getFile();
    FileOutputStream fileStream = null;
    try {
      fileStream = new FileOutputStream(file);
    } catch (FileNotFoundException fne) {
      //...
      throw fne;
    }
```   
(2).调用写回调将图像资源写到文件中
```
    long length = -1;
    try {
      CountingOutputStream countingStream = new CountingOutputStream(fileStream);
      callback.write(countingStream);
      countingStream.flush();
      length = countingStream.getCount();
    } finally {
      fileStream.close();
    }
    if (file.length() != length) {
      throw new IncompleteFileException(length, file.length());
    }
```   
3. commit()：保存资源文件   
(1).创建一个Content类型的资源文件(.cnt)，并将临时文件重命名为该资源文件。Fresco对图像资源的缓存使用了临时文件，这样可以更好地将临时文件均匀散列到各个子目录下。  
```
    FileBinaryResource tempFileResource = (FileBinaryResource) temporary;
    File tempFile = tempFileResource.getFile();
    File targetFile = getContentFileFor(resourceId);
    try {
      FileUtils.rename(tempFile, targetFile);
    }
```   
其中getContentFileFor()是返回对应的散列目录下对应文件的File对象。
(2).出错处理，最后更新资源文件的最近修改时间。

4. 查询资源文件：   
&#8195;对于触及性查询，在查询到结果后，会更新资源文件的最近被修改时间   
```
  @Override
  public boolean contains(String resourceId, Object debugInfo) {
    return query(resourceId, false);
  }
  @Override
  public boolean touch(String resourceId, Object debugInfo) {
    return query(resourceId, true);
  }
  private boolean query(String resourceId, boolean touch) {
    File contentFile = getContentFileFor(resourceId);
    boolean exists = contentFile.exists();
    if (touch && exists) {
      contentFile.setLastModified(mClock.now());
    }
    return exists;
  }
```   
5. 获取资源内容：   
&#8195;在查询到结果后，会更新资源文件的最近被修改时间
```
  @Override
  public FileBinaryResource getResource(String resourceId, Object debugInfo) {
    final File file = getContentFileFor(resourceId);
    if (file.exists()) {
      file.setLastModified(mClock.now());
      return FileBinaryResource.createOrNull(file);
    }
    return null;
  }
  ```