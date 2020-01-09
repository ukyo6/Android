每当发新版本时，美团团购Android客户端会被分发到各个应用市场，比如[豌豆荚](http://www.wandoujia.com/apps/com.sankuai.meituan)，[360手机助手](http://zhushou.360.cn/detail/index/soft_id/1068?recrefer=SE_D_美团)等。为了统计这些市场的效果（活跃数，下单数等），需要有一种方法来唯一标识它们。

团购客户端目前通过渠道号（`channel`）来区分不同的市场，代码中使用`Config.channel`变量记录该渠道号。比如，豌豆荚市场中美团应用的渠道号是`wandoujia`，360手机助手中美团应用的渠道号为`qihu360`。客户端访问API时会在请求参数中带上渠道号，以便后台接下来计算不同渠道的效果。

每次发版时，市场部会提供一个渠道列表，Android RD会根据这些渠道相应地生成等量的渠道包。随着渠道越来越多（截止本文写作时已有900多个渠道），客户端打渠道包的方式也一直在演进，本文接下来就详细介绍美团应用的打包之旅。

[Maven](http://maven.apache.org/)是一个软件项目管理和自动构建工具，配合使用[android-maven-plugin](https://code.google.com/p/maven-android-plugin/wiki/GettingStarted)插件，以及[maven-resources-plugin](http://maven.apache.org/plugins/maven-resources-plugin/)插件可以很方便的生成渠道包，下面简要介绍下打包过程，更多Maven以及插件的使用方法请参考相关文档。

首先，在`AndroidManifest.xml`的`节点中添加如下`元素，用来定义渠道的来源：

```xml
<!-- 使用Maven打包时会用具体的渠道号替换掉${channel} -->
<meta-data
        android:name="channel"
        android:value="${channel}" />
```

定义好渠道来源后，接下来就可以在程序启动时读取渠道号了：

```java
private String getChannel(Context context) {
        try {
            PackageManager pm = context.getPackageManager();
            ApplicationInfo appInfo = pm.getApplicationInfo(context.getPackageName(), PackageManager.GET_META_DATA);
            return appInfo.metaData.getString("channel");
        } catch (PackageManager.NameNotFoundException ignored) {
        }
        return "";

    }
```

要替换`AndroidManifest.xml`文件定义的渠道号，还需要在`pom.xml`文件中配置Resources插件：

```xml
<resources>           
    <resource>
        <directory>${project.basedir}</directory>
        <filtering>true</filtering>
        <targetPath>${project.build.directory}/filtered-manifest</targetPath>
        <includes>
            <include>AndroidManifest.xml</include>
        </includes>
    </resource>
</resources>
```

准备工作已经完成，现在需要的就是实际的渠道号了。下面的脚本会遍历渠道列表，逐个替换并打包：

```
#!/bin/bash

package(){
	while read line
	do
		mvn clean
		mvn  -Dchannel=$line package
	done < $1
}

package $1
```

在前期渠道很少时这种方法还可以接受，但只要渠道稍微增多该方法就不再适用了，原因是每打一个包都要执行一遍构建过程，效率太低。

[apktool](https://code.google.com/p/android-apktool/)是一个逆向工程工具，可以用它解码（decode）并修改apk中的资源。接下来详细介绍如何使用apktool生成渠道包。

前期工作和用Maven打包一样，也需要在`AndroidManifest.xml`文件中定义 `<meta-data>`元素，并在应用启动的时候读取清单文件中的渠道号。具体请参考上面的代码。

和Maven不一样的是，每次打包时不再需要重新构建项目。打包时，只需生成一个apk，然后在该apk的基础上生成其他渠道包即可。

首先，使用apktool decode应用程序，在终端中输入如下命令：

```
apktool d your_original_apk build 
```

上面的命令会在build目录中decode应用文件，decode完成后的目录如下：

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/3c880a77.jpg)

接下来，替换`AndroidManifest.xml`文件中定义的渠道号，下面是一段python脚本：

```python
import re

def replace_channel(channel, manifest):
	pattern = r'(<meta-data\s+android:name="channel"\s+android:value=")(\S+)("\s+/>)'
	replacement = r"\g<1>{channel}\g<3>".format(channel=channel)
	return re.sub(pattern, replacement, manifest)
```

然后，使用apktool构建未签名的apk：

```mariadb
apktool b build your_unsigned_apk
```

最后，使用jarsigner重新签名apk：

```
jarsigner -sigalg MD5withRSA -digestalg SHA1 -keystore your_keystore_path -storepass your_storepass -signedjar your_signed_apk, your_unsigned_apk, your_alias
```

上面就是使用apktool打包的方法，通过使用脚本可以批量地生成渠道包。不像Maven，每打一个包都需要执行一次构建过程，该方法只需构建一次，大大节省了时间。

但是好景不长，我们的渠道包越来越多，目前已有近900个渠道，打完所有的渠道包需要近3个小时。有没有更快的打包方式呢？且看下节。

如果能直接修改apk的渠道号，而不需要再重新签名能节省不少打包的时间。幸运的是我们找到了这种方法。直接解压apk，解压后的根目录会有一个`META-INF`目录，如下图所示：

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/b25b2615.jpg)

如果在`META-INF`目录内添加空文件，可以不用重新签名应用。因此，通过为不同渠道的应用添加不同的空文件，可以唯一标识一个渠道。

下面的python代码用来给apk添加空的渠道文件，渠道名的前缀为`mtchannel_`：

```python
import zipfile
zipped = zipfile.ZipFile(your_apk, 'a', zipfile.ZIP_DEFLATED) 
empty_channel_file = "META-INF/mtchannel_{channel}".format(channel=your_channel)
zipped.write(your_empty_file, empty_channel_file)
```

添加完空渠道文件后的目录，`META-INFO`目录多了一个名为`mtchannel_meituan`的空文件：

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/8eaa455e.jpg)

接下来就可以在Java代码中读取空渠道文件名了：

```java
public static String getChannel(Context context) {
        ApplicationInfo appinfo = context.getApplicationInfo();
        String sourceDir = appinfo.sourceDir;
        String ret = "";
        ZipFile zipfile = null;
        try {
            zipfile = new ZipFile(sourceDir);
            Enumeration<?> entries = zipfile.entries();
            while (entries.hasMoreElements()) {
                ZipEntry entry = ((ZipEntry) entries.nextElement());
                String entryName = entry.getName();
                if (entryName.startsWith("mtchannel")) {
                    ret = entryName;
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (zipfile != null) {
                try {
                    zipfile.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        String[] split = ret.split("_");
        if (split != null && split.length >= 2) {
            return ret.substring(split[0].length() + 1);

        } else {
            return "";
        }
    }
```

这样，每打一个渠道包只需复制一个apk，在`META-INF`中添加一个使用渠道号命名的空文件即可。这种打包方式速度非常快，900多个渠道不到一分钟就能打完。

上面总共介绍了三种打渠道包的方式。目前，Android团队打包基本使用第三种方式，完成了打包的自动化，解放了工程师的生产力，善哉善哉。

打包的问题解决了，但有时候还需要为不同的渠道定制不同的APK。下一讲会介绍Android构建利器Gradle，以及如何使用Gradle定制渠道包，敬请期待。