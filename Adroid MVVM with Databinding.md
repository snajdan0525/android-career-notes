```java
public class MainViewModel implements ViewModel {
    .
    .
    .
//true的时候弹出Dialog，false的时候关掉dialog
public final ObservableBoolean isShowDialog = new ObservableBoolean();
    .
    .
    .
}


// 在View层做一个对isShowDialog改变的监听
public class MainActivity extends RxBasePmsActivity {
private MainViewModel mainViewModel;

@Override
protected void onCreate(Bundle savedInstanceState) {
..... 
mainViewModel.isShowDialog.addOnPropertyChangedCallback(new android.databinding.Observable.OnPropertyChangedCallback() {
      @Override
      public void onPropertyChanged(android.databinding.Observable sender, int propertyId) {
          if (mainViewModel.isShowDialog.get()) {
               dialog.show();
          } else {
               dialog.dismiss();
          }
       }
    });
 }
 ...
}
```
这段代码的总结：
　
　简单的说你可以对任意的ObservableField做监听，然后根据数据的变化做相应UI的改变，业务层ViewModel 只要根据业务处理数据就行，以数据来驱动UI。