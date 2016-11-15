
```java
private void ensurePinnedHeaderLayout(View header) {
    if (header.isLayoutRequested()) {
        int widthSpec = MeasureSpec.makeMeasureSpec(getMeasuredWidth(), mWidthMode);
        
        int heightSpec;
        ViewGroup.LayoutParams layoutParams = header.getLayoutParams();
        if (layoutParams != null && layoutParams.height > 0) {
            heightSpec = MeasureSpec.makeMeasureSpec(layoutParams.height, MeasureSpec.EXACTLY);
        } else {
            heightSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
        }
        header.measure(widthSpec, heightSpec);
        header.layout(0, 0, header.getMeasuredWidth(), header.getMeasuredHeight());
    }
}
```