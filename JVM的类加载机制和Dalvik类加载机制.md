
　　构建PathClassLoader时会调用这个去new 一个DexFile把他放入DexPathList中
```java 
/**
 * Opens a DEX file from a given filename. This will usually be a ZIP/JAR
 * file with a "classes.dex" inside.
 *
 * The VM will generate the name of the coresponding file in
 * /data/dalvik-cache and open it, possibly creating or updating
 * it first if system permissions allow.  Don't pass in the name of
 * a file in /data/dalvik-cache, as the named file is expected to be
 * in its original (pre-dexopt) state.
 * 
 * @param fileName
 *            the filename of the DEX file
 * 
 * @throws IOException
 *             if an I/O error occurs, such as the file not being found or
 *             access rights missing for opening it
 */
public DexFile(String fileName) throws IOException {
    String wantDex = System.getProperty("android.vm.dexfile", "false");
    if (!wantDex.equals("true"))
        throw new UnsupportedOperationException("No dex in this VM");

    mCookie = openDexFile(fileName, null, 0);
    mFileName = fileName;
    //System.out.println("DEX FILE cookie is " + mCookie);
}
```


　　注释：打开dex文件，并且指定dex优化文件的写入地址，这个一般是执行外部dex文件时候调用的，不直接调用，而是通过DexClassLoader调用的
```java
/**
 * Open a DEX file, specifying the file in which the optimized DEX
 * data should be written.  If the optimized form exists and appears
 * to be current, it will be used; if not, the VM will attempt to
 * regenerate it.
 *
 * This is intended for use by applications that wish to download
 * and execute DEX files outside the usual application installation
 * mechanism.  This function should not be called directly by an
 * application; instead, use a class loader such as
 * dalvik.system.DexClassLoader.
 *
 * @param sourcePathName
 *  Jar or APK file with "classes.dex".  (May expand this to include
 *  "raw DEX" in the future.)
 * @param outputPathName
 *  File that will hold the optimized form of the DEX data.
 * @param flags
 *  Enable optional features.  (Currently none defined.)
 * @return
 *  A new or previously-opened DexFile.
 * @throws IOException
 *  If unable to open the source or output file.
 */
static public DexFile loadDex(String sourcePathName, String outputPathName,
    int flags) throws IOException {

    /*
     * TODO: we may want to cache previously-opened DexFile objects.
     * The cache would be synchronized with close().  This would help
     * us avoid mapping the same DEX more than once when an app
     * decided to open it multiple times.  In practice this may not
     * be a real issue.
     */
    return new DexFile(sourcePathName, outputPathName, flags);
}
```


在openDexFile()中只是调用了openDexFileNative () 继续跟入在\ dalvik\v m\nat ive\dalvik _sys tem_DexFile.cpp文件中的openDexFileNative() 函数，接下重点就在这个函数：

复制代码
static void Dalvik_dalvik_system_DexFile_openDexFileNative(const u4* args,
    JValue* pResult)
{
//args[0]: sourceName java层传入的
//args[1]: outputName    
    StringObject* sourceNameObj = (StringObject*) args[0];
    StringObject* outputNameObj = (StringObject*) args[1];
    DexOrJar* pDexOrJar = NULL;
    JarFile* pJarFile;
RawDexFile* pRawDexFile;
//DexOrJar*  JarFile*   RawDexFile* 目录
    char* sourceName;
    char* outputName;
    //……
    sourceName = dvmCreateCstrFromString(sourceNameObj);
    if (outputNameObj != NULL)
        outputName = dvmCreateCstrFromString(outputNameObj);
    else
        outputName = NULL;
/*判断要加载的dex是否为系统中的dex文件
* gDvm ？？？
*/
    if (dvmClassPathContains(gDvm.bootClassPath, sourceName)) {
        ALOGW("Refusing to reopen boot DEX '%s'", sourceName);
        dvmThrowIOException(
            "Re-opening BOOTCLASSPATH DEX files is not allowed");
        free(sourceName);
        free(outputName);
        RETURN_VOID();
    }

    /*
     * Try to open it directly as a DEX if the name ends with ".dex".
     * If that fails (or isn't tried in the first place), try it as a
     * Zip with a "classes.dex" inside.
     */
    //判断sourcename扩展名是否是.dex
    if (hasDexExtension(sourceName)
            && dvmRawDexFileOpen(sourceName, outputName, &pRawDexFile, false) == 0) {
        ALOGV("Opening DEX file '%s' (DEX)", sourceName);
        pDexOrJar = (DexOrJar*) malloc(sizeof(DexOrJar));
        pDexOrJar->isDex = true;
        pDexOrJar->pRawDexFile = pRawDexFile;
        pDexOrJar->pDexMemory = NULL;
    //.jar文件
    } else if (dvmJarFileOpen(sourceName, outputName, &pJarFile, false) == 0) {
        ALOGV("Opening DEX file '%s' (Jar)", sourceName);
        pDexOrJar = (DexOrJar*) malloc(sizeof(DexOrJar));
        pDexOrJar->isDex = false;
        pDexOrJar->pJarFile = pJarFile;
        pDexOrJar->pDexMemory = NULL;
} else {
//都不满足，抛出异常
        ALOGV("Unable to open DEX file '%s'", sourceName);
        dvmThrowIOException("unable to open DEX file");
    }
if (pDexOrJar != NULL) {
        pDexOrJar->fileName = sourceName;
    //把pDexOr这个结构体中的内容加到gDvm中的userDexFile结构的hash表中，便于Dalvik以后的查找
        addToDexFileTable(pDexOrJar);
    } else {
        free(sourceName);
    }
    free(outputName);
    RETURN_PTR(pDexOrJar);
}
复制代码
接下来再看对.dex文件的处理函数dvmRawDexFileOpen 在dalvik\vm\RawDexFile.cpp文件中：

复制代码
/* See documentation comment in header. */
int dvmRawDexFileOpen(const char* fileName, const char* odexOutputName,
    RawDexFile** ppRawDexFile, bool isBootstrap)
{
    DvmDex* pDvmDex = NULL;
    char* cachedName = NULL;
    int result = -1;
    int dexFd = -1;
    int optFd = -1;
    u4 modTime = 0;
    u4 adler32 = 0;
    size_t fileSize = 0;
    bool newFile = false;
    bool locked = false;
    dexFd = open(fileName, O_RDONLY);  //打开dex文件
    if (dexFd < 0) goto bail;
    /* If we fork/exec into dexopt, don't let it inherit the open fd. */
dvmSetCloseOnExec(dexFd);//dexfd不继承
//校验dex文件的标志，将第8字节开始的4个字节赋值给adler32。
    if (verifyMagicAndGetAdler32(dexFd, &adler32) < 0) {
        ALOGE("Error with header for %s", fileName);
        goto bail;
    }
    //得到dex文件的大小和修改时间，保存在modTime和filesize中
    if (getModTimeAndSize(dexFd, &modTime, &fileSize) < 0) {
        ALOGE("Error with stat for %s", fileName);
        goto bail;
    }

    //odexOutputName就是odex文件名，如果odexOutputName为空，则自动生成一个。
    if (odexOutputName == NULL) {
        cachedName = dexOptGenerateCacheFileName(fileName, NULL);
        if (cachedName == NULL)
            goto bail;
    } else {
        cachedName = strdup(odexOutputName);   
    }
    //主要是验证缓存文件名的正确性，之后将dexOptHeader结构写入fd中
    optFd = dvmOpenCachedDexFile(fileName, cachedName, modTime,
        adler32, isBootstrap, &newFile, /*createIfMissing=*/true);
    locked = true;
  
    if (newFile) {
        u8 startWhen, copyWhen, endWhen;
        bool result;
        off_t dexOffset;
        dexOffset = lseek(optFd, 0, SEEK_CUR);  //文件指针的位置
        result = (dexOffset > 0);
        if (result) {
            startWhen = dvmGetRelativeTimeUsec();
    //将dex文件中的内容拷贝到当前odex文件，也就是dexOffset开始
            result = copyFileToFile(optFd, dexFd, fileSize) == 0;
            copyWhen = dvmGetRelativeTimeUsec();
        }
        if (result) {
    //优化odex文件
            result = dvmOptimizeDexFile(optFd, dexOffset, fileSize,
                fileName, modTime, adler32, isBootstrap);
        }
    }
    /*
     * Map the cached version.  This immediately rewinds the fd, so it
     * doesn't have to be seeked anywhere in particular.
     */
//将odex文件映射到内存空间(mmap)，并用mprotect将属性置为只读属性，并将映射的dex结构放在pDvmDex数据结构中，具体代码在下面。
    if (dvmDexFileOpenFromFd(optFd, &pDvmDex) != 0) {
        ALOGI("Unable to map cached %s", fileName);
        goto bail;
    }
……
}
复制代码
 

 

复制代码
//Dalvik/vm/RewDexFile.cpp
static int verifyMagicAndGetAdler32(int fd, u4 *adler32)
{
    u1 headerStart[12];
    ssize_t amt = read(fd, headerStart, sizeof(headerStart));
    if (amt < 0) {
        ALOGE("Unable to read header: %s", strerror(errno));
        return -1;
    }
    if (amt != sizeof(headerStart)) {
        ALOGE("Unable to read full header (only got %d bytes)", (int) amt);
        return -1;
    }
    if (!dexHasValidMagic((DexHeader*) (void*) headerStart)) {
        return -1;
    }
    *adler32 = (u4) headerStart[8]
        | (((u4) headerStart[9]) << 8)
        | (((u4) headerStart[10]) << 16)
        | (((u4) headerStart[11]) << 24);

    return 0;
}
复制代码
 

 

复制代码
//dalvik\vm\DvmDex.cpp
/*
 * Given an open optimized DEX file, map it into read-only shared memory and
 * parse the contents.
 *
 * Returns nonzero on error.
 */
int dvmDexFileOpenFromFd(int fd, DvmDex** ppDvmDex)
{
    DvmDex* pDvmDex;
    DexFile* pDexFile;
    MemMapping memMap;
    int parseFlags = kDexParseDefault;
    int result = -1;

    if (gDvm.verifyDexChecksum)
        parseFlags |= kDexParseVerifyChecksum;
    if (lseek(fd, 0, SEEK_SET) < 0) {
        ALOGE("lseek rewind failed");
        goto bail;
    }
    //mmap映射fd文件,就是我们之前的odex文件
   if (sysMapFileInShmemWritableReadOnly(fd, &memMap) != 0) {
        ALOGE("Unable to map file");
        goto bail;
    }
    pDexFile = dexFileParse((u1*)memMap.addr, memMap.length, parseFlags);
    if (pDexFile == NULL) {
        ALOGE("DEX parse failed");
        sysReleaseShmem(&memMap);
        goto bail;
    }
    pDvmDex = allocateAuxStructures(pDexFile);
    if (pDvmDex == NULL) {
        dexFileFree(pDexFile);
        sysReleaseShmem(&memMap);
        goto bail;
    }
/* tuck this into the DexFile so it gets released later */
//将映射odex文件的内存拷贝到DvmDex的结构中
    sysCopyMap(&pDvmDex->memMap, &memMap);
    pDvmDex->isMappedReadOnly = true;
    *ppDvmDex = pDvmDex;
    result = 0;

bail:
    return result;
}


/*dalvik\libdex\SysUtil.cpp
*/
int sysMapFileInShmemWritableReadOnly(int fd, MemMapping* pMap)
{
    off_t start;
    size_t length;
    void* memPtr;
assert(pMap != NULL);
//获得文件长度和文件开始地址
    if (getFileStartAndLength(fd, &start, &length) < 0)
        return -1;
//映射文件
    memPtr = mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_FILE | MAP_PRIVATE,
            fd, start);
    //……
//将保护属性置为只读属性
    if (mprotect(memPtr, length, PROT_READ) < 0) {
      //…….
    }
    pMap->baseAddr = pMap->addr = memPtr;
    pMap->baseLength = pMap->length = length;
return 0;
//……
}
复制代码
下面在分析文件后缀不是.dex的情况：

复制代码
/*如果不是.dex文件*/
int dvmJarFileOpen(const char* fileName, const char* odexOutputName,
    JarFile** ppJarFile, bool isBootstrap)
{
    ZipArchive archive;
    DvmDex* pDvmDex = NULL;
    char* cachedName = NULL;
    bool archiveOpen = false;
    bool locked = false;
    int fd = -1;
   int result = -1;

//打开.jar文件并映射，内存结构放在ZipArchive中，之后将具体分析的代码
    if (dexZipOpenArchive(fileName, &archive) != 0)
        goto bail;
    archiveOpen = true;
dvmSetCloseOnExec(dexZipGetArchiveFd(&archive));  //不继承

    // openAlternateSuffix函数将fileName的后缀名改为”.odex”，例如
    //”Hello.jar”--”Hello.odex”，然后调用open()”打开”Hello.odex文件
    //如果成功返回”Hello.odex”的文件描述符
    fd = openAlternateSuffix(fileName, "odex", O_RDONLY, &cachedName);
    if (fd >= 0) {
        ALOGV("Using alternate file (odex) for %s ...", fileName);
        //…检验optHeader 
        if (!dvmCheckOptHeaderAndDependencies(fd, false, 0, 0, true, true)) {
            //……
            goto tryArchive;
        } 
    } else {  
        ZipEntry entry;
tryArchive:
        /*
         * Pre-created .odex absent or stale.  Look inside the jar for a
         * "classes.dex".
         */
// static const char* kDexInJarName = "classes.dex";
        /*
            在dexZipFindEntry函数中，对kDexInJarName也就是”class.dex”进行hash运算，找到”class.dex”在archive结构中的表项
        */
entry = dexZipFindEntry(&archive, kDexInJarName);
        if (entry != NULL) {
            bool newFile = false;
           
        //如果odex缓存路径为空，则自动生成一个路径
            if (odexOutputName == NULL) {
                cachedName = dexOptGenerateCacheFileName(fileName,
                                kDexInJarName);
                if (cachedName == NULL)
                    goto bail;
            } else {
                cachedName = strdup(odexOutputName);
            }
             //创建cachedName对应的文件  (.odex)
            fd = dvmOpenCachedDexFile(fileName, cachedName,
                    dexGetZipEntryModTime(&archive, entry),
                    dexGetZipEntryCrc32(&archive, entry),
            //……
            locked = true;
            //……
            if (newFile) {   //成功创建.odex文件
                u8 startWhen, extractWhen, endWhen;
                bool result;
                off_t dexOffset;
                dexOffset = lseek(fd, 0, SEEK_CUR);
                result = (dexOffset > 0);
                if (result) {
                    startWhen = dvmGetRelativeTimeUsec();
                    result = dexZipExtractEntryToFile(&archive, entry, fd) == 0;
                    extractWhen = dvmGetRelativeTimeUsec();
                }
                if (result) {
                    //优化dex文件-.odex
                    result = dvmOptimizeDexFile(fd, dexOffset,
                                dexGetZipEntryUncompLen(&archive, entry),
                                fileName,
                                dexGetZipEntryModTime(&archive, entry),
                                dexGetZipEntryCrc32(&archive, entry),
                                isBootstrap);
                }
//已经得到了.odex文件，下面的流程就和.dex文件一样了。
    //映射.odex文件，
    if (dvmDexFileOpenFromFd(fd, &pDvmDex) != 0)   
    //…………
    return result;
}
复制代码
 

 

复制代码
//\dalvik\libdex\SysUtil.cpp
int dexZipOpenArchive(const char* fileName, ZipArchive* pArchive)
{
    int fd, err;
    …….
    memset(pArchive, 0, sizeof(ZipArchive));
    //打开文件
    fd = open(fileName, O_RDONLY | O_BINARY, 0);
      ……
    return dexZipPrepArchive(fd, fileName, pArchive);
}


int dexZipPrepArchive(int fd, const char* debugFileName, ZipArchive* pArchive)
{
    int result = -1;
    memset(pArchive, 0, sizeof(*pArchive));
    pArchive->mFd = fd;   //Zip的文件描述符
    if (mapCentralDirectory(fd, debugFileName, pArchive) != 0)
        goto bail;
    if (parseZipArchive(pArchive) != 0) { 
        goto bail;
    }
    /* success */
    result = 0;
bail:
    if (result != 0)
        dexZipCloseArchive(pArchive); //失败释放pArchive结构
    return result;
}


static int mapCentralDirectory(int fd, const char* debugFileName,
    ZipArchive* pArchive)
{
    /*
     * Get and test file length.
     */
//检验文件长度的有效性
    off64_t fileLength = lseek64(fd, 0, SEEK_END);
    if (fileLength < kEOCDLen) {
             return -1;
    }  
    size_t readAmount = kMaxEOCDSearch;
    if (fileLength < off_t(readAmount))
        readAmount = fileLength;

    u1* scanBuf = (u1*) malloc(readAmount);
    if (scanBuf == NULL) {
        return -1;
    }
    int result = mapCentralDirectory0(fd, debugFileName, pArchive,
            fileLength, readAmount, scanBuf);
    free(scanBuf);
    return result;
}


tatic int mapCentralDirectory0(int fd, const char* debugFileName,
        ZipArchive* pArchive, off64_t fileLength, size_t readAmount, u1* scanBuf)
{
    /*
     * Make sure this is a Zip archive.
     */
//校验文件是否合法的Zip文件
    //……                                //偏移16的地方  //偏移12
    if (sysMapFileSegmentInShmem(fd, centralDirOffset, centralDirSize,
            &pArchive->mDirectoryMap) != 0)
    {
        ALOGW("Zip: cd map failed");
        return -1;
    }
    pArchive->mNumEntries = numEntries;
    pArchive->mDirectoryOffset = centralDirOffset;
    return 0;
}


int sysMapFileSegmentInShmem(int fd, off_t start, size_t length,
    MemMapping* pMap)
{
    size_t actualLength;
    off_t actualStart;
    int adjust;
    void* memPtr;
    assert(pMap != NULL);
    /* adjust to be page-aligned */
    adjust = start % SYSTEM_PAGE_SIZE;
    actualStart = start - adjust;
    actualLength = length + adjust;
    //映射
    memPtr = mmap(NULL, actualLength, PROT_READ, MAP_FILE | MAP_SHARED,
                fd, actualStart);
       // …….
    pMap->baseAddr = memPtr;
    pMap->baseLength = actualLength;
    pMap->addr = (char*)memPtr + adjust;
    pMap->length = length;
    return 0;
}
复制代码
ZipArchive的结构体如下：

复制代码
struct ZipArchive {
    /* open Zip archive */
    int         mFd;   //打开的zip文件
    /* mapped central directory area */
    off_t       mDirectoryOffset;   
    MemMapping  mDirectoryMap;   //映射内存的结构
    /* number of entries in the Zip archive */
    int         mNumEntries;      //
    int         mHashTableSize;   //名字hash表的大小
    ZipHashEntry* mHashTable;     //hash表的表项，
};


struct ZipHashEntry {
    const char*     name;
    unsigned short   nameLen;
};
复制代码
 

 

我们可以简要总结下整个的加载流程，首先是对文件名的修正，后缀名置为”.dex”作为输出文件，然后生个一个DexPathList对象函数直接返回一个DexPathList对象，

在DexPathList的构造函数中调用makeDexElements()函数，在makeDexElement()函数中调用loadDexFile()开始对.dex或者是.jar .zip .apk文件进行处理，

跟入loadDexFile()函数中，会发现里面做的工作很简单，调用optimizedPathFor()函数对optimizedDiretcory路径进行修正。

之后才真正通过DexFile.loadDex()开始加载文件中的数据，其中的加载也只是返回一个DexFile对象。

在DexFile类的构造函数中，重点便放在了其调用的openDexFile()函数，在openDexFile()中调用了openDexFileNative()真正进入native层，

在openDexFileNative()的真正实现中，对于后缀名为.dex的文件或者其他文件(.jar .apk .zip)分开进行处理：

.dex文件调用dvmRawDexFileOpen()；
其他文件调用dvmJarFileOpen()。

在dvmRawDexFileOpen()函数中，检验dex文件的标志，检验odex文件的缓存名称，之后将dex文件拷贝到odex文件中，并对odex进行优化

调用dvmDexFileOpenFromFd()对优化后的odex文件进行映射，通过mprotect置为"只读"属性并将映射的内存结构保存在DvmDex*结构中。

dvmJarFileOpen()先对文件进行映射，结构保存在ZipArchive中，然后再尝试以文件名作为dex文件名来“打开”文件，
如果失败，则调用dexZipFindEntry在ZipArchive的名称hash表中找名为"class.dex"的文件,然后创建odex文件，下面就和
dvmRawDexFileOpen()一样了，就是对dex文件进行优化和映射。

也只是分析了一个大概流程，还有很多有待之后进行深入。而这里对于阅读Android源码，有了新的体会，首先是工具上，我之前一直是用Source InSight 但是对于一些函数的实现，找起来却是不太方便，因为必须要将函数实现的文件导入到工程中，而用VS来阅读源码，利用Ctrl+Shift+F的功能，在Android源码目录下搜索更为方便，然后可以在Source InSight中进行导入，阅读。其次不得不说阅读源码真的是一个比较痛苦的过程，但真的学习下来，收获还是很大的。