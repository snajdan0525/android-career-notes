```java
    @Override
    public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);//beginTransaction实则返回的是个BackStackRecord(FragmentTransaction的子类),
    }
```

