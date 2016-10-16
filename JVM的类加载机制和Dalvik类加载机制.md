
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