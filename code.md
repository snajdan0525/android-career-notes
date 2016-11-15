```java
public final int getSectionForPosition(int position) {
    // first try to retrieve values from cache
    Integer cachedSection = mSectionCache.get(position);
    if (cachedSection != null) {
        return cachedSection;
    }
    int sectionStart = 0;
    for (int i = 0; i < internalGetSectionCount(); i++) {
        int sectionCount = internalGetCountForSection(i);
        int sectionEnd = sectionStart + sectionCount + 1;
        if (position >= sectionStart && position < sectionEnd) {
            mSectionCache.put(position, i);
            return i;
        }
        sectionStart = sectionEnd;
    }
    return 0;
}
```