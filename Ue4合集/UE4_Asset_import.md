导入文件说明。
当点下导入按钮之后，将会触发windows的消息循环，最终达到
void SContentBrowser::ImportAsset( const FString& InPath )
这里，这个Inpath将是要导入到的路径地址。
然后从FModuleManager里边获取AssetTools，得到一个FAssetToolsModule的引用，然后调用 FAssetToolsModule.get().ImportAssets(InPath);
在这里面，将会查找所有的UFactory的子类，如果这个子类不是纯虚的类型，并且 bool UFactory::bEditorImport 为帧的话，那么加入到一个列表中。
然后调用 ObjectTools::GenerateFactoryFileExtensions ，访问TArray<FString> UFactory::Formats ，产生所有支持的后缀。接着打开windows下的资源窗口，此时选中资源，并且点打开。
此时进入 FAssetTools::ImportAssets，然后进入 FAssetTools::ImportAssetsInternal
最终将会将后缀与对应的UFactory进行绑定，最后调用 UFactory::FactoryVCanImport(Filename) 进行询问。
然后调用 TSubclassOf<UObject> UFactory::SupportedClass 获取到支持的 class。
然后生成一个.uassert的包
设置SetAutomatedAssetImportData
调用 UFactory::ImportObject
询问 UFactory::CanCreateNew()
调用 UFactory::FactoryCreateFile()
最终得到了 对应的Object。
然后 调用 FAssetRegistryModule::AssetCreated 创建asset



