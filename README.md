# 理解作用域存储

Android长久以来都支持外置存储空间这个功能，也就是我们常说的SD卡存储。这个功能使用得极其广泛，几乎所有的App都喜欢在SD卡的根目录下建立一个自己专属的目录，用来存放各类文件和数据。

那么这么做有什么好处吗？第一，存储在SD卡的文件不会计入到应用程序的占用空间当中，也就是说即使你在SD卡存放了1G的文件，你的应用程序在设置中显示的占用空间仍然可能只有几十K。第二，存储在SD卡的文件，即使应用程序被卸载了，这些文件仍然会被保留下来，这有助于实现一些需要数据被永久保留的功能。

然而，这些“好处”真的是好处吗？或 许对于开发者而言这算是好处吧，但对于用户而言，上述好处无异于一些流氓行为。因为这会将用户的SD卡空间搞得乱糟糟的，而且即使我卸载了一个完全不再使用的程序，它所产生的垃圾文件却可能会一直保留在我的手机上。

另外，存储在SD卡上的文件属于公有文件，所有的应用程序都有权随意访问，这也对数据的安全性带来了很大的挑战。

为了解决上述问题，Google在Android 10当中加入了作用域存储功能。

那么到底什么是作用域存储呢？简单来讲，就是Android系统对SD卡的使用做了很大的限制。从Android 10开始，每个应用程序只能有权在自己的外置存储空间关联目录下读取和创建文件，获取该关联目录的代码是：context.getExternalFilesDir()。关联目录对应的路径大致如下：

```java
/storage/emulated/0/Android/data/<包名>/files
```

将数据存放到这个目录下，你将可以完全使用之前的写法来对文件进行读写，不需要做任何变更和适配。但同时，刚才提到的那两个“好处”也就不存在了。这个目录中的文件会被计入到应用程序的占用空间当中，同时也会随着应用程序的卸载而被删除。

那么有些朋友可能会问了，我就是需要访问其他目录该怎么办呢？比如读取手机相册中的图片，或者向手机相册中添加一张图片。为此，Android系统针对文件类型进行了分类，图片、音频、视频这三类文件将可以通过MediaStore API来进行访问，而其他类型的文件则需要使用系统的文件选择器来进行访问。

另外，我们的应用程序向媒体库贡献的图片、音频或视频，将会自动拥有其读写权限，不需要额外申请READ_EXTERNAL_STORAGE和WRITE_EXTERNAL_STORAGE权限。而如果你要读取其他应用程序向媒体库贡献的图片、音频或视频，则必须要申请READ_EXTERNAL_STORAGE权限才行。WRITE_EXTERNAL_STORAGE权限将会在未来的Android版本中废弃。
# 升级兼容
一定会有很多朋友关心这个问题，因为每当适配升级面临着需要更改大量代码的时候，大多数人的第一想法都是能不升就不升，或者能晚升就晚升。而在作用域存储这个功能上面，恭喜大家，暂时确实是可以不用升级的。

目前Android 10系统对于作用域存储适配的要求还不是那么严格，毕竟之前传统外置存储空间的用法实在是太广泛了。如果你的项目指定的targetSdkVersion低于29，那么即使不做任何作用域存储方面的适配，你的项目也可以成功运行到Android 10手机上。

而如果你的targetSdkVersion已经指定成了29，也没有关系，假如你还不想进行作用域存储的适配，只需要在AndroidManifest.xml中加入如下配置即可：

```xml
<manifest ... >
  <application android:requestLegacyExternalStorage="true" ...>
    ...
  </application>
</manifest>
```
这段配置表示，即使在Android 10系统上，仍然允许使用之前遗留的外置存储空间的用法来运行程序，这样就不用对代码进行任何修改了。当然，这只是一种权宜之计，在未来的Android系统版本中，这段配置随时都可能会失效（Android 11中已强制启用作用域存储，这段配置在Android 11当中已不再有效）。
# 获取相册中的图片
首先来学习一下如何在作用域存储当中获取手机相册里的图片。注意，虽然本篇文章中我是以图片来举例的，但是获取音频、视频的用法也是基本相同的。

不同于过去可以直接获取到相册中图片的绝对路径，在作用域存储当中，我们只能借助MediaStore API获取到图片的Uri，示例代码如下：

```kotlin
val cursor = contentResolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null, null, null, "${MediaStore.MediaColumns.DATE_ADDED} desc")
if (cursor != null) {
    while (cursor.moveToNext()) {
        val id = cursor.getLong(cursor.getColumnIndexOrThrow(MediaStore.MediaColumns._ID))
        val uri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id)
        println("image uri is $uri")
    }
	cursor.close()
}
```
上述代码中，我们先是通过ContentResolver获取到了相册中所有图片的id，然后再借助ContentUris将id拼装成一个完整的Uri对象。一张图片的Uri格式大致如下所示：

```kotlin
content://media/external/images/media/321
```
那么有些朋友可能会问了，获取到了Uri之后，我又该怎样将这张图片显示出来呢？这就有很多种办法了，比如使用Glide来加载图片，它本身就支持传入Uri对象来作为图片路径：

```java
Glide.with(context).load(uri).into(imageView)
```
而如果你没有使用Glide或其他图片加载框架，想在不借助第三方库的情况下直接将一个Uri对象解析成图片，可以使用如下代码：

```kotlin
val fd = contentResolver.openFileDescriptor(uri, "r")
if (fd != null) {
    val bitmap = BitmapFactory.decodeFileDescriptor(fd.fileDescriptor)
	fd.close()
    imageView.setImageBitmap(bitmap)
}
```
上述代码中，我们调用了ContentResolver的openFileDescriptor()方法，并传入Uri对象来打开文件句柄，然后再调用BitmapFactory的decodeFileDescriptor()方法将文件句柄解析成Bitmap对象即可。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303224904366.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjgxNTgw,size_16,color_FFFFFF,t_70)

**这样我们就将获取相册中图片的方式掌握了，并且这种方式在所有的Android系统版本中都适用。**

# 将图片添加到相册
将一张图片添加到手机相册要相对稍微复杂一点，因为不同系统版本之间的处理方式是不太一样的。

我们还是通过一段代码示例来直观地学习一下，代码如下所示：

```kotlin
fun addBitmapToAlbum(bitmap: Bitmap, displayName: String, mimeType: String, compressFormat: Bitmap.CompressFormat) {
    val values = ContentValues()
    values.put(MediaStore.MediaColumns.DISPLAY_NAME, displayName)
    values.put(MediaStore.MediaColumns.MIME_TYPE, mimeType)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        values.put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DCIM)
    } else {
        values.put(MediaStore.MediaColumns.DATA, "${Environment.getExternalStorageDirectory().path}/${Environment.DIRECTORY_DCIM}/$displayName")
    }
    val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
    if (uri != null) {
        val outputStream = contentResolver.openOutputStream(uri)
        if (outputStream != null) {
            bitmap.compress(compressFormat, 100, outputStream)
			outputStream.close()
        }
    }
}
```
这段代码演示了如何将一个Bitmap对象添加到手机相册当中，我来简单解释一下。

想要将一张图片添加到手机相册，我们需要构建一个ContentValues对象，然后向这个对象中添加三个重要的数据。一个是DISPLAY_NAME，也就是图片显示的名称，一个是MIME_TYPE，也就是图片的mime类型。还有一个是图片存储的路径，不过这个值在Android 10和之前的系统版本中的处理方式不一样。Android 10中新增了一个RELATIVE_PATH常量，表示文件存储的相对路径，可选值有DIRECTORY_DCIM、DIRECTORY_PICTURES、DIRECTORY_MOVIES、DIRECTORY_MUSIC等，分别表示相册、图片、电影、音乐等目录。而在之前的系统版本中并没有RELATIVE_PATH，所以我们要使用DATA常量（已在Android 10中废弃），并拼装出一个文件存储的绝对路径才行。

有了ContentValues对象之后，接下来调用ContentResolver的insert()方法即可获得插入图片的Uri。但仅仅获得Uri仍然是不够的，我们还需要向该Uri所对应的图片写入数据才行。调用ContentResolver的openOutputStream()方法获得文件的输出流，然后将Bitmap对象写入到该输出流当中即可。

以上代码即可实现将Bitmap对象存储到手机相册当中，那么有些朋友可能会问了，如果我要存储的图片并不是Bitmap对象，而是一张网络上的图片，或者是当前应用关联目录下的图片该怎么办呢？

其实方法都是相似的，因为不管是网络上的图片还是关联目录下的图片，我们都能获取到它的输入流，只要不断读取输入流中的数据，然后写入到相册图片所对应的输出流当中就可以了，示例代码如下：

```kotlin
fun writeInputStreamToAlbum(inputStream: InputStream, displayName: String, mimeType: String) {
    val values = ContentValues()
    values.put(MediaStore.MediaColumns.DISPLAY_NAME, displayName)
    values.put(MediaStore.MediaColumns.MIME_TYPE, mimeType)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        values.put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DCIM)
    } else {
        values.put(MediaStore.MediaColumns.DATA, "${Environment.getExternalStorageDirectory().path}/${Environment.DIRECTORY_DCIM}/$displayName")
    }
    val bis = BufferedInputStream(inputStream)
    val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
    if (uri != null) {
        val outputStream = contentResolver.openOutputStream(uri)
        if (outputStream != null) {
            val bos = BufferedOutputStream(outputStream)
            val buffer = ByteArray(1024)
            var bytes = bis.read(buffer)
            while (bytes >= 0) {
                bos.write(buffer, 0 , bytes)
                bos.flush()
                bytes = bis.read(buffer)
            }
            bos.close()
        }
    }
    bis.close()
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303224940318.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjgxNTgw,size_16,color_FFFFFF,t_70)

这段代码中只是将输入流和输出流的部分重新编写了一下，其他部分和之前存储Bitmap的代码是完全一致的，相信很好理解。
# 下载文件到Download目录
执行文件下载操作是一个很常见的场景，比如说下载pdf、doc文件，或者下载APK安装包等等。在过去，这些文件我们通常都会下载到Download目录，这是一个专门用于存放下载文件的目录。而从Android 10开始，我们已经不能以绝对路径的方式访问外置存储空间了，所以文件下载功能也会受到影响。

那么该如何解决呢？主要有以下两种方式。

第一种同时也是最简单的一种方式，就是更改文件的下载目录。将文件下载到应用程序的关联目录下，这样不用修改任何代码就可以让程序在Android 10系统上正常工作。但使用这种方式，你需要知道，下载的文件会被计入到应用程序的占用空间当中，同时如果应用程序被卸载了，该文件也会一同被删除。另外，存放在关联目录下的文件只能被当前的应用程序所访问，其他程序是没有读取权限的。

以上几个限制条件如果不能满足你的需求，那么就只能使用第二种方式，对Android 10系统进行代码适配，仍然将文件下载到Download目录下。

其实将文件下载到Download目录，和向相册中添加一张图片的过程是差不多的，Android 10在MediaStore中新增了一种Downloads集合，专门用于执行文件下载操作。但由于每个项目下载功能的实现都各不相同，有些项目的下载实现还十分复杂，因此怎么将以下的示例代码融合到你的项目当中是你自己需要思考的问题。

```kotlin
fun downloadFile(fileUrl: String, fileName: String) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
        Toast.makeText(this, "You must use device running Android 10 or higher", Toast.LENGTH_SHORT).show()
        return
    }
    thread {
		try {
			val url = URL(fileUrl)
			val connection = url.openConnection() as HttpURLConnection
			connection.requestMethod = "GET"
			connection.connectTimeout = 8000
			connection.readTimeout = 8000
			val inputStream = connection.inputStream
			val bis = BufferedInputStream(inputStream)
			val values = ContentValues()
			values.put(MediaStore.MediaColumns.DISPLAY_NAME, fileName)
			values.put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DOWNLOADS)
			val uri = contentResolver.insert(MediaStore.Downloads.EXTERNAL_CONTENT_URI, values)
			if (uri != null) {
				val outputStream = contentResolver.openOutputStream(uri)
				if (outputStream != null) {
					val bos = BufferedOutputStream(outputStream)
					val buffer = ByteArray(1024)
					var bytes = bis.read(buffer)
					while (bytes >= 0) {
						bos.write(buffer, 0 , bytes)
						bos.flush()
						bytes = bis.read(buffer)
					}
					bos.close()
				}
			}
			bis.close()
		} catch(e: Exception) {
			e.printStackTrace()
		}
    }
}
```
这段代码总体来讲还是比较好理解的，主要就是添加了一些Http请求的代码，并将MediaStore.Images.Media改成了MediaStore.Downloads，其他部分几乎是没有变化的，我就不再多加解释了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021030322495785.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjgxNTgw,size_16,color_FFFFFF,t_70)


注意，上述代码只能在Android 10或更高的系统版本上运行，因为MediaStore.Downloads是Android 10中新增的API。至于Android 9及以下的系统版本，请你仍然使用之前的代码来进行文件下载。
# 使用文件选择器
如果我们要读取SD卡上非图片、音频、视频类的文件，比如说打开一个PDF文件，这个时候就不能再使用MediaStore API了，而是要使用文件选择器。

但是，我们不能再像之前的写法那样，自己写一个文件浏览器，然后从中选取文件，而是必须要使用手机系统中内置的文件选择器。示例代码如下：

```kotlin
const val PICK_FILE = 1

private fun pickFile() {
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT)
    intent.addCategory(Intent.CATEGORY_OPENABLE)
    intent.type = "*/*"
    startActivityForResult(intent, PICK_FILE)
}

override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    when (requestCode) {
        PICK_FILE -> {
            if (resultCode == Activity.RESULT_OK && data != null) {
                val uri = data.data
                if (uri != null) {
                    val inputStream = contentResolver.openInputStream(uri)
					// 执行文件读取操作
                }
            }
        }
    }
}
```
这里在pickFile()方法当中通过Intent去启动系统的文件选择器，注意Intent的action和category都是固定不变的。而type属性可以用于对文件类型进行过滤，比如指定成image/就可以只显示图片类型的文件，这里写成/*表示显示所有类型的文件。注意type属性必须要指定，否则会产生崩溃。

然后在onActivityResult()方法当中，我们就可以获取到用户选中文件的Uri，之后通过ContentResolver打开文件输入流来进行读取就可以了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303225009811.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjgxNTgw,size_16,color_FFFFFF,t_70)

# 第三方SDK不支持作用域存储怎么办？
阅读完了本篇文章之后，相信你对Android 10作用域存储的用法和适配基本上都已经掌握了。然而我们在实际的开发工作当中还可能会面临一个非常头疼的问题，就是我自己的代码当然可以进行适配，但是项目中使用的第三方SDK还不支持作用域存储该怎么办呢？

这个情况确实是存在的，比如我之前使用的七牛云SDK，它的文件上传功能要求你传入的就是一个文件的绝对路径，而不支持传入Uri对象，大家应该也会碰到类似的问题。

由于我们是没有权限修改第三方SDK的，因此最简单直接的办法就是等待第三方SDK的提供者对这部分功能进行更新，在那之前我们先不要将targetSdkVersion指定到29，或者先在AndroidManifest文件中配置一下requestLegacyExternalStorage属性。

然而如果你不想使用这种权宜之计，其实还有一个非常好的办法来解决此问题，就是我们自己编写一个文件复制功能，将Uri对象所对应的文件复制到应用程序的关联目录下，然后再将关联目录下这个文件的绝对路径传递给第三方SDK，这样就可以完美进行适配了。这个功能的示例代码如下：

```kotlin
fun copyUriToExternalFilesDir(uri: Uri, fileName: String) {
    val inputStream = contentResolver.openInputStream(uri)
    val tempDir = getExternalFilesDir("temp")
    if (inputStream != null && tempDir != null) {
        val file = File("$tempDir/$fileName")
        val fos = FileOutputStream(file)
        val bis = BufferedInputStream(inputStream)
        val bos = BufferedOutputStream(fos)
        val byteArray = ByteArray(1024)
        var bytes = bis.read(byteArray)
        while (bytes > 0) {
            bos.write(byteArray, 0, bytes)
            bos.flush()
            bytes = bis.read(byteArray)
        }
        bos.close()
        fos.close()
    }
}
```

**以上所有内容便是Android10的作用域存储，接下来讲解Android11中作用域存储新加的功能。**
# 强制启用Scoped Storage
首先，在Android 11中，Scoped Storage被强制启用了。

那么强制启用是什么意思呢？

在Android 10中虽然也有Scoped Storage功能，但是Google考虑到广大应用程序适配也是需要时间的，因此并没有强制启用这个功能。

只要应用程序指定的targetSdkVersion低于29，或targetSdkVersion等于29，但在AndroidManifest.xml中加入了如下配置：

```xml
<manifest ... >
  <application android:requestLegacyExternalStorage="true" ...>
    ...
  </application>
</manifest>
```
那么Scoped Storage功能就不会被启用。

在Android 11中以上配置依然有效，但仅限于targetSdkVersion小于或等于29的情况。如果你的targetSdkVersion等于30，Scoped Storage就会被强制启用，requestLegacyExternalStorage标记将会被忽略。

那么强制启用了Scoped Storage之后对开发者而言有什么影响吗？

其实如果你的应用程序已经按照 Android 10适配要点，作用域存储 这篇文章中讲解的方式对Scoped Storage进行了适配，那么恭喜你，现在你什么都不需要做，就已经能够适配Android 11系统了。

也就是说，对于绝大部分开发者而言，强制启用Scoped Storage其实并没有什么影响，只要你的应用程序在之前已经适配了Android 10的Scoped Storage。

但是有一类应用程序非常特殊，就是文件浏览器，如Root Explorer、ES Explorer等。这类程序本身提供的功能就是对SD上的文件进行浏览与管理，而强制启用了Scoped Storage之后，本质上就没有文件浏览的概念了，我们也无法以文件的真实路径来对文件进行管理。

从这个角度上看，Scoped Storage对于文件浏览器类的程序造成了毁灭性的打击。不过不用担心，Google仍然还是给这类程序提供了另外一种解决方案，下面我们就来学习一下。
# 管理设备上所有的文件
首先明确一点，Android 11中强制启用Scoped Storage是为了更好地保护用户的隐私，以及提供更加安全的数据保护。对于绝大部分应用程序来说，使用MediaStore提供的API就已经可以满足大家的开发需求了。如果你没有类似于开发文件浏览器这种需求，请尽可能不要使用接下来即将介绍的技术。

拥有对整个SD卡的读写权限，在Android 11上被认为是一种非常危险的权限，同时也可能会对用户的数据安全造成比较大的影响。

但文件浏览器就是要对设备的整个SD卡进行管理的，这怎么办呢？对于这类危险程度比较高的权限，Google通常采用的做法是，使用Intent跳转到一个专门的授权页面，引导用户手动授权，比如悬浮窗，无障碍服务等。

没错，在Android 11中，如果你想要管理整个设备上的文件，也需要使用类似的技术。

首先，你必须在AndroidManifest.xml中声明MANAGE_EXTERNAL_STORAGE权限，如下所示：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.scopedstoragedemo">

    <uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE"
        tools:ignore="ScopedStorage" />

</manifest>
```
注意相比于传统声明一个权限，这里增加了tools:ignore="ScopedStorage"这样一个属性。因为如果不加上这个属性，Android Studio会用一个警告提醒我们，绝大部分的应用程序都不应该申请这个权限，正如我前面介绍的一样。

接下来的工作也相当简单，我们可以使用ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION这个action来跳转到指定的授权页面，可以通过Environment.isExternalStorageManager()这个函数来判断用户是否已授权，下面我写了一段比较简单的代码来演示这个功能：

```kotlin
if (Build.VERSION.SDK_INT < Build.VERSION_CODES.R ||
        Environment.isExternalStorageManager()) {
    Toast.makeText(this, "已获得访问所有文件权限", Toast.LENGTH_SHORT).show()
} else {
    val builder = AlertDialog.Builder(this)
        .setMessage("本程序需要您同意允许访问所有文件权限")
        .setPositiveButton("确定") { _, _ ->
            val intent = Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION)
            startActivity(intent)
        }
    builder.show()
}
```
可以看到，这里首先判断如果系统版本低于Android 11，或者Environment.isExternalStorageManager()返回true，那么就说明我们已经拥有管理整个SD卡的权限了。现在你可以直接使用传统的写法，以文件真实路径的形式对文件进行操作。

而如果还没有管理SD卡的权限，则会弹出一个对话框，告知用户申请权限的原因，然后使用Intent跳转到指定的授权页面，让用户手动进行授权。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303224325768.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjgxNTgw,size_16,color_FFFFFF,t_70)


有了这个权限之后，你就可以用过去熟知的方式去开发文件浏览器了。

不过还有一点需要注意，即使我们获得了管理SD卡的权限，对于Android这个目录下的很多资源仍然是访问受限的，比如说Android/data这个目录在Android 11中使用任何手段都无法访问。因为很多应用程序的数据信息都会存放在这个目录下，做这个限制的目的主要还是考虑到用户的数据安全吧。不然的话，允许微信去读取淘宝中的数据，怎么想好像都是不合适的。
# Batch operations
下面我们再来看Android 11中关于Scoped Storage的另外一个新特性。

Scoped Storage规定，每个应用程序都有权限向MediaStore贡献数据，比如说插入一张图片到手机相册当中。也有权限读取其他应用程序所贡献的数据，比如说获取手机相册中的所有图片。这些功能我在 Android 10适配要点，作用域存储 这篇文章中都进行了演示。

但是，假如你要修改其他应用程序所贡献的数据，那不好意思，Scoped Storage是不允许你这样做的。

原因也很简单，如果一张图片是你插入到手机相册的，你当然有权限对它进行任意修改。但是如果这张图片是其他应用程序插入到手机相册的，你还能对它进行任意修改，这在Google看来就又是一个安全隐患，所以Scoped Storage限制了这个功能。

不过，如果有些应用程序就是需要修改别的应用所贡献的数据呢？这种例子也不难找，比如Photoshop、美图秀秀等，它们的目的就是为了修改手机相册中的图片，不管这个图片是不是它们自己所创建的。

针对这个问题，Android 10中提供了一种解决方案：

```kotlin
try {
    contentResolver.openFileDescriptor(imageContentUri, "w")?.use {
        Toast.makeText(this, "现在可以修改图片的灰度了", Toast.LENGTH_SHORT).show()
    }
} catch (securityException: SecurityException) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        val recoverableSecurityException = securityException as?
            RecoverableSecurityException ?:
            throw RuntimeException(securityException.message, securityException)

        val intentSender = recoverableSecurityException.userAction.actionIntent.intentSender
        intentSender?.let {
            startIntentSenderForResult(intentSender, IMAGE_REQUEST_CODE,
                    null, 0, 0, 0, null)
        }
    } else {
        throw RuntimeException(securityException.message, securityException)
    }
}
```
下面我来简单解释一下这段代码。

首先这段代码的目的是为了修改一张图片的灰度，但由于这张图片并不是由当前应用程序所贡献的，所以理论上当前应用程序并没有权限去修改这张图片的灰度。

那么明明没有权限去修改，但是我们还是执意去修改会发生什么情况呢？这个很好理解，当然是抛异常了。于是这里用try catch的方式包裹了修改图片灰度的操作，然后在catch的代码块中判断，如果当前系统版本大于等于Android 10，并且异常的类型是RecoverableSecurityException，那么就说明这是一个由于Scoped Storage限制导致操作没有权限的异常。

接下来会从RecoverableSecurityException对象中获取一个intentSender，再借助这个intentSender进行页面跳转，引导用户手动授予我们修改这张图片的权限。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303224352717.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjgxNTgw,size_16,color_FFFFFF,t_70)
这种方式虽然可行，但却有一个非常明显的缺点：每次我们只能操作一张图片。如果一个程序需要修改很多张图片，没有什么好办法，只能每张图片都用上述方式去申请权限。

相信Google也是意识到了这个问题，于是在Android 11中引入了一个新的功能，叫作Batch operations，从而允许我们可以一次性对多个文件的操作权限进行申请。

关于Batch operations的用法也很好理解，Google一共提供了4种类型的权限申请API，如下所示：

 - createWriteRequest() 用于请求对多个文件的写入权限。
 - createFavoriteRequest() 用于请求将多个文件加入到Favorite（收藏）的权限。
 - createTrashRequest() 用于请求将多个文件移至回收站的权限。
 - createDeleteRequest() 用于请求将多个文件删除的权限。

其中最常用的主要是createWriteRequest()和createDeleteRequest()这两个接口，这里我们以createWriteRequest()举例。

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
    val urisToModify = listOf(uri1, uri2, uri3, uri4)
    val editPendingIntent = MediaStore.createWriteRequest(contentResolver, urisToModify)
    startIntentSenderForResult(editPendingIntent.intentSender, EDIT_REQUEST_CODE,
            null, 0, 0, 0)
}
```
代码非常简单，首先我们创建了一个集合，用于存放所有要批量申请权限的文件Uri，然后调用createWriteRequest()函数去创建一个PendingIntent，接下来再调用startIntentSenderForResult进行权限申请即可。

关于权限申请的结果，我们可以在onActivityResult()中进行监听：

```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    when (requestCode) {
        EDIT_REQUEST_CODE -> {
            if (resultCode == Activity.RESULT_OK) {
                Toast.makeText(this, "用户已授权", Toast.LENGTH_SHORT).show()
            } else {
                Toast.makeText(this, "用户没有授权", Toast.LENGTH_SHORT).show()
            }
        }
    }
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303224634991.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjgxNTgw,size_16,color_FFFFFF,t_70)
其它几个API的用法都是完全相同的，这里就不再重复举例了。

看到这里，有的朋友可能会说，Android 10和Android 11提供的API完全不同，Android 10是要依赖于异常捕获机制，从RecoverableSecurityException中解析出intentSender，而Android 11可以借助Batch operations提供的API直接创建intentSender。我该不会需要在一个项目中针对Android 10和Android 11分别写两套代码去进行适配吧？

这确实是个头疼的问题，而且我觉得主要是由于Google一开始在Android 10中API设计不合理所导致的。依赖于异常捕获机制的方案，无论如何都不能说是一种出色的API设计。

不过随着后来更多的思考，我发现这并不是一个无法解决的问题，并且解决方案还非常简单。

为什么呢？别忘了，Android 10中的Scoped Storage并不是强制启用的，我们可以在AndroidManifest.xml中配置requestLegacyExternalStorage标记来禁用Scoped Storage。这样的话，Android 10就是不需要适配的，我们只需要在Android 11中使用更加科学规范的API来进行Scoped Storage适配就可以了。





[Demo传送门](https://github.com/ruanyandong/Android10-Scoped-Storage)

参考文档

[Android 10适配要点，作用域存储](https://guolin.blog.csdn.net/article/details/105419420)

[Android 11新特性，Scoped Storage又有了新花样](https://guolin.blog.csdn.net/article/details/113954552)








