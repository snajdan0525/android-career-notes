
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
