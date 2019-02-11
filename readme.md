EROS是基于 weex 封装、面向前端的 vue 写法的一整套 APP 开源解决方案。https://github.com/bmfe/eros 

该项目对EROS框架源码进行了修改，实现了文件上传和下载功能。

#Tips:

1.	为什么不按照EROS官方文档 https://github.com/bmfe/android-eros-plugin-simple 新建插件项目？
    之前按照官方文档创建插件项目，项目中添加该插件项目依赖后出现了依赖文件重复，SDK依赖版本不同导致build fail的情况，因此迫不得已才在框架源码上进行修改。

2.文件上传功能
包括文件选择和文件上传两个部分，文件选择使用的github上的MultiType-FilePicker项目  https://github.com/fishwjy/MultiType-FilePicker
在此对@fishwjy提出感谢，支持根据后缀名进行筛选文件，支持选择多个文件。

文件上传使用的是框架中图片上传使用的AxiosManager类（基于okhttp3进行封装），支持自定义header,cookie部分使用的是系统自带android.webkit.CookieSyncManager，自动添加cookie到header中。

3.文件下载功能
支持自定义header,自动添加cookie，调用的是系统自带的DownloadManager。不足之处是需要传入文件名（不含后缀名）和后缀名，无法根据响应header中Content-Disposition判断文件名。

4.使用方法

文件下载
``` javascript
                          weex.requireModule("FileModule").downloadFile(
                  {
                    url:"xxxxxxx",//下载地址
                    name:"yyyyyyy"//文件名（不含后缀名）
                    suffix: "zzzzzzz",//后缀名（例如 "docx"）
                    params:{aaa:"bbb"},//自定义请求参数[可选]
                    headers: {//自定义header[可选]
                      Referer:'aaaaaa'
                    }
                  },
                  function(resData) {
                    weex.requireModule("bmModal").alert({
                      message:
                        resData.message == "success"
                          ? "正在下载，请稍后在通知中心查看"
                          : "下载失败"
                    });
                  }
                );
```


文件上传
``` javascript
                          weex.requireModule("FileModule").uploadFile(
                  {
                    url:"xxxxxxxx",//上传地址
                    maxCount:10,//最大可选择的文件数量
                    suffix:"rar,zip,7z",//支持的文件类型
                    params:{aaa:"bbb"},//自定义请求参数[可选]
                    headers: {
                      Referer:"yyyyyyy"//自定义请求头[可选]
                    }
                  },
                  function(resData) {
                    weex.requireModule("bmModal").alert({
                      message:
                        resData
                    });
                  }
                );
```



#具体改动的地方有以下几处：

修改多个build.gradle中的
compileSdkVersion 26
targetSdkVersion 26

\platforms\android\WeexFrameworkWrapper\wxframework\eros-framework\build.gradle
添加依赖
dependencies{
compile 'com.vincent.filepicker:MultiTypeFilePicker:1.0.8'
}



package com.eros.framework.extend.module;
创建
``` java
public class FileModule extends WXModule {
    @JSMethod
    public void uploadFile(String params, JSCallback jsCallback) {
        WeexEventBean eventBean = new WeexEventBean();
        eventBean.setContext(mWXSDKInstance.getContext());
        eventBean.setKey(WXEventCenter.EVENT_FILE_UPLOAD);
        eventBean.setJsParams(params);
        eventBean.setJscallback(jsCallback);
        ManagerFactory.getManagerService(DispatchEventManager.class).getBus().post(eventBean);
    }

    @JSMethod
    public void downloadFile(String params, JSCallback jsCallback) {
        WeexEventBean eventBean = new WeexEventBean();
        eventBean.setContext(mWXSDKInstance.getContext());
        eventBean.setKey(WXEventCenter.EVENT_FILE_DOWNLOAD);
        eventBean.setJsParams(params);
        eventBean.setJscallback(jsCallback);
        ManagerFactory.getManagerService(DispatchEventManager.class).getBus().post(eventBean);
    }
}
```



com.eros.framework.constant.WXEventCenter
添加3个事件字符串常量定义
``` java
public static final String EVENT_FILE_PICK = "EVENT_FILE_PICK";
public static final String EVENT_FILE_UPLOAD = "EVENT_FILE_UPLOAD";
public static final String EVENT_FILE_DOWNLOAD = "EVENT_FILE_DOWNLOAD";
```


package com.eros.framework.event.file; （需创建file包）
创建
``` java
public class EventFile extends EventGate {
    private JSCallback mPickCallback,mUploadAvatar;
    private Context mUploadContext;

    @Override
    public void perform(Context context, WeexEventBean weexEventBean, String type) {

        String params = weexEventBean.getJsParams();

        if (WXEventCenter.EVENT_FILE_DOWNLOAD.equals(type)) {
            download(params, context, weexEventBean.getJscallback());
        }
        else if (WXEventCenter.EVENT_FILE_UPLOAD.equals(type)) {
            upload(params, context, weexEventBean.getJscallback());
        }
    }

    public void upload(String json, Context context, JSCallback jsCallback) {
        //Manifest.permission.READ_EXTERNAL_STORAGE 权限申请
        if (!PermissionUtils.checkPermission(context, Manifest.permission.READ_EXTERNAL_STORAGE)) {
            return;
        }
        mUploadAvatar = jsCallback;
        mUploadContext = context;
        UploadFileBean bean = ManagerFactory.getManagerService(ParseManager.class).parseObject
            (json, UploadFileBean.class);
        ManagerFactory.getManagerService(DispatchEventManager.class).getBus().register(this);
        FileUpdownloaderManager fileUpdownloadManager= ManagerFactory.getManagerService(FileUpdownloaderManager.class);
        fileUpdownloadManager.pick(context,bean,Constant.FileConstants.FILE_PICKER);
    }

    public void download(String json, Context context, JSCallback jsCallback) {

        DownloadFileBean bean = ManagerFactory.getManagerService(ParseManager.class).parseObject
            (json, DownloadFileBean.class);

        String fileName = bean.name;
        String suffix = bean.suffix;
        DownloadManager.Request request = new DownloadManager.Request(Uri.parse(bean.url));
        HashMap<String,String> headers= ManagerFactory.getManagerService(ParseManager.class).parseObject
            (bean.headers, HashMap.class);
        for(Map.Entry<String,String> e:headers.entrySet()){
            request.addRequestHeader(e.getKey(),e.getValue());
        }
        CookieSyncManager.createInstance(context);
        CookieManager cookieManager = CookieManager.getInstance();
        cookieManager.setAcceptCookie(true);
        cookieManager.removeSessionCookie();// 移除旧的[可以省略]
        List<Cookie> cookies = new PersistentCookieStore(context).getCookies();
        String cookieString= "";
        for (int i = 0; i < cookies.size(); i++) {
            Cookie cookie = cookies.get(i);
            if (cookie.matches(HttpUrl.parse(bean.url))) {
                String value = cookie.name() + "=" + cookie.value();
                cookieString += (value + "; ");
                cookieManager.setCookie(bean.url, value);
            }
        }
        if(cookieString.length()>0){request.addRequestHeader("Cookie",cookieString);}
        // 允许在计费流量下下载
        request.setAllowedOverMetered(true);
        // 允许媒体扫描，根据下载的文件类型被加入相册、音乐等媒体库
        request.allowScanningByMediaScanner();
        // 设置通知的显示类型，下载进行时和完成后显示通知
        request.setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED);
        // 设置通知栏的标题，如果不设置，默认使用文件名
        request.setTitle(fileName+"."+suffix);
        // 设置通知栏的描述
        request.setAllowedOverMetered(true);
        // 允许该记录在下载管理界面可见
        request.setVisibleInDownloadsUi(true);
        // 允许漫游时下载
        request.setAllowedOverRoaming(true);
        // 允许下载的网路类型
        //request.setAllowedNetworkTypes(DownloadManager.Request.NETWORK_WIFI);
        // 设置下载文件保存的路径和文件名

        request.setMimeType("application/octet-stream");
        switch(suffix){
            case "rar":
                request.setMimeType("application/x-rar-compressed");
                break;
            case "zip":
                request.setMimeType("application/zip");
                break;
            case "ppt":
                request.setMimeType("application/vnd.ms-powerpoint");
                break;
            case "xls":
                request.setMimeType("application/vnd.ms-excel");
                break;
            case "xlsx":
                request.setMimeType("application/vnd.ms-excel");
                break;
            case "doc":
                request.setMimeType("application/msword");
                break;
            case "docx":
                request.setMimeType("application/msword");
                break;
            case "jpg":
                request.setMimeType("image/jpeg");
                break;
            case "png":
                request.setMimeType("image/png");
                break;
            case "gif":
                request.setMimeType("image/gif");
                break;
            case "apk":
                request.setMimeType("application/vnd.android.package-archive");
                break;
        }
        request.setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, fileName+"."+suffix);
//                //另外可选一下方法，自定义下载路径
//                request.setDestinationUri();
//                request.setDestinationInExternalFilesDir();
        try{
            final DownloadManager downloadManager = (DownloadManager) context.getSystemService(DOWNLOAD_SERVICE);
            // 添加一个下载任务
            long downloadId = downloadManager.enqueue(request);
            //Toast.makeText(context, "正在下载，请稍后在通知中心查看", Toast.LENGTH_LONG).show();
            JsPoster.postSuccess("success", jsCallback);
            return;
        }
        catch (Exception e){
            e.printStackTrace();
        }
        JsPoster.postFailed("fail",jsCallback);
        //Toast.makeText(context, "下载失败", Toast.LENGTH_LONG).show();
    }

    @Subscribe
    public void OnUploadResult(UploadResultBean uploadResultBean) {
        if (uploadResultBean != null && mPickCallback != null) {
            JsPoster.postSuccess(uploadResultBean.data, mPickCallback);
        }

        ManagerFactory.getManagerService(DispatchEventManager.class).getBus().unregister(this);
        mPickCallback = null;
        ManagerFactory.getManagerService(PersistentManager.class).deleteCacheData(Constant
            .FileConstants.UPLOAD_FILE_BEAN);
    }
}
```




package com.eros.framework.model;
创建
``` java
public class UploadFileBean implements Serializable {
    public String url;
    public int maxCount; //最大可选择个数
    public String params; //上传附带参数 json
    public String headers;//请求头
    public List<String> files; // 上传文件本地地址列表
    public String suffix;//支持的文件后缀名（形如"pptx,docx,xls"的字符串）
}
```



package com.eros.framework.manager.impl;
创建
``` java
public class FileUpdownloaderManager extends Manager  {


    /**
     * 选择多个文件上传
     *
     * @param context 上下文对象
     * @param bean    Js 返回的数据集合
     */
    public void pick(final Context context, UploadFileBean bean, int requestCode) {
        DefaultFileAdapter.getInstance().pickFile(context, bean,requestCode);
    }


    /**
     * 上传多个文件
     *
     * @param items 选择文件集合
     * @param bean  uplaod 参数对象
     */
    public void UpMultipleFileData(Context context, ArrayList<NormalFile> items, UploadFileBean
            bean) {
        DefaultFileAdapter.getInstance().UpMultipleFileData(context, items, bean);

    }
}
```


package com.eros.framework.adapter;
创建
``` java
public class DefaultFileAdapter {
    private static DefaultFileAdapter mInstance = new DefaultFileAdapter();


    private DefaultFileAdapter() {
    }

    public static DefaultFileAdapter getInstance() {
        return mInstance;
    }


    public void pickFile(final Context context, UploadFileBean bean, int requestCode) {
        if (!checkPermission(context)) return;
        Intent intent4 = new Intent(context, NormalFilePickActivity.class);
        intent4.putExtra(Constant.MAX_NUMBER, 9);
        intent4.putExtra(NormalFilePickActivity.SUFFIX, bean.suffix.split(","));
        PersistentManager persistentManager = ManagerFactory.getManagerService(PersistentManager
            .class);
        persistentManager.setCacheData(com.eros.framework.constant.Constant.FileConstants.UPLOAD_FILE_BEAN, bean);
        ((Activity) context).startActivityForResult(intent4, requestCode);

    }


    public void UpMultipleFileData(Context context, ArrayList<NormalFile> items, UploadFileBean
            bean) {
        ModalManager.BmLoading.showLoading(context, null, false);
        ArrayList imagesFilrUrl = new ArrayList();
        if (items != null && items.size() > 0) {
            for (NormalFile item : items) {
                imagesFilrUrl.add(item.getPath());
            }
        }
        HashMap<String, String> uploadParams = null;
        HashMap<String, String> heads = null;
        if (bean != null) {
            String params = bean.params;
            ParseManager parseManager = ManagerFactory.getManagerService(ParseManager.class);
            uploadParams = parseManager.parseObject(params, HashMap.class);
            heads = parseManager.parseObject(bean.headers, HashMap.class);
        }
        CookieSyncManager.createInstance(context);
        CookieManager cookieManager = CookieManager.getInstance();
        cookieManager.setAcceptCookie(true);
        cookieManager.removeSessionCookie();// 移除旧的[可以省略]
        List<Cookie> cookies = new PersistentCookieStore(context).getCookies();
        String cookieString= "";
        for (int i = 0; i < cookies.size(); i++) {
            Cookie cookie = cookies.get(i);
            if (cookie.matches(HttpUrl.parse(bean.url))) {
                String value = cookie.name() + "=" + cookie.value();
                cookieString += (value + "; ");
                cookieManager.setCookie(bean.url, value);
            }
        }
        if(cookieString.length()>0){heads.put("Cookie",cookieString);}
        AxiosManager axiosManager = ManagerFactory.getManagerService(AxiosManager.class);
        String url = TextUtils.isEmpty(bean.url) ? Api.UPLOAD_URL : bean.url;
        axiosManager.upload(url, imagesFilrUrl, uploadParams, heads);
    }


    /**
     * 判断Sd卡是否挂载，是否有Sd卡权限
     */
    private boolean checkPermission(Context context) {
        PermissionManager permissionManager = ManagerFactory.getManagerService(PermissionManager
                .class);
        boolean hasPermisson = permissionManager.hasPermissions(context, Manifest.permission
                .READ_EXTERNAL_STORAGE);
        if (!hasPermisson) {
            ModalManager.BmToast.toast(context, "读取sd卡存储权限未授予，请到应用设置页面开启权限!", Toast.LENGTH_SHORT);
        }
        return hasPermisson;
    }
}
```




com.eros.framework.constant.Constant
添加
``` java
public static final class FileConstants {
    public static final String UPLOAD_FILE_BEAN = "upload_file_bean";
    public static final int FILE_PICKER = 102;
}
```


package com.eros.framework.model;
创建
``` java
public class DownloadFileBean implements Serializable {
    public String url;//下载地址

    public String headers;//请求头

    public String name;//文件名（不包含后缀名）

    public String suffix;//后缀名
}
```



com.eros.framework.activity.AbstractWeexActivity#onActivityResult
添加
``` java
/**
*选择文件 上传
*/
if(requestCode==REQUEST_CODE_PICK_FILE&&resultCode == RESULT_OK){
    ArrayList<NormalFile> list = data.getParcelableArrayListExtra(RESULT_PICK_FILE);
    UpMultipleFileData(list);
}
```



com.eros.framework.activity.AbstractWeexActivity
添加
``` java
/**
 * 上传文件
 */
private void UpMultipleFileData(ArrayList<NormalFile> items) {
    FileUpdownloaderManager fileUpdownloadManager = ManagerFactory.getManagerService(FileUpdownloaderManager
        .class);
    UploadFileBean bean = ManagerFactory.getManagerService
        (PersistentManager.class).getCacheData
        (Constant.FileConstants.UPLOAD_FILE_BEAN, UploadFileBean.class);
    fileUpdownloadManager.UpMultipleFileData(this, items, bean);
}
```



com.eros.framework.manager.impl.AxiosManager#upload(java.lang.String, java.lang.String, java.util.Map<java.lang.String,java.lang.String>, java.util.Map<java.lang.String,java.lang.String>, com.eros.framework.http.okhttp.callback.StringCallback)
修改为
``` java
builder.addFile("file", TextUtils.isEmpty(ext) ? "file.jpg" : AppUtils.getFileName(filePath), new File
        (filePath));
```        
    


\platforms\android\WeexFrameworkWrapper\wxframework\eros-framework\src\main\res\xml\app_config.xml
添加
``` xml
<module name="FileModule">com.eros.framework.extend.module.FileModule</module>
```



com.eros.framework.event.DispatchEventCenter#onWeexEvent
添加
``` java
            case WXEventCenter.EVENT_FILE_UPLOAD:
            case WXEventCenter.EVENT_FILE_DOWNLOAD:
                reflectionClazzPerform("com.eros.framework.event.file.EventFile", context
                    , weexEventBean
                    , "", weexEventBean.getKey());
                break;
```
