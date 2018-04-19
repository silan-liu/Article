@(ios)

### iOS单元测试问题小结

在写unit test的过程中遇到一些问题，记录一下。

#### xx.h file not found

* 如果是给pod lib create创建的库写单元测试，检查Podfile中`tests     target`有无如下配置。这样cocoapods会给test target设置search path。

 ```
	target 'xxTests' do
	    inherit! :search_paths
	 end
 ```

 pod install之后，会发现`Pods/Targets Support Files`下面多了个`Pods-xxx_Tests的文件夹`，.xcconfig文件中配置了搜索路径。 

* 如果不是，需要在tests target的build setting中设置`header search path`。

#### symbols not found
	
 1. 切到tests target的build settings，设置`BUNDLE_LOADER=$(TEST_HOST)`，设置`TEST_HOST= $(BUILT_PRODUCTS_DIR)/xx.app/xx` (xx为host target名称)
				
 2. 切到host target的build settings，选择`Symbols hidden by default`为NO。

还有一种情况，如果`podfile`中是以`use_framework`的方式(即动态库)，那么在引入头文件时需要使用`<xx/xx.h>`，不然也会报symbol not found。	 

#### Library not loaded

在9.x的系统上发现会报错`Library not loaded`。

```
Library not loaded: 
/System/Library/Frameworks/CallKit.framework/CallKit
```

 在`test target`的`build phase -> link binary with library`， 添加callkit，选择optional即可。

#### 跑真机

 真机跑单元测试需要选择证书，确保host target和test target的证书一致。  

#### Unable to connect to test manager

 重启手机后解决。