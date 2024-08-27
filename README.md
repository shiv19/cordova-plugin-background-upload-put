
# cordova-plugin-background-upload

This plugin provides a file upload functionality that can continue to run even while the app is in background. It includes progress updates suitable for long-term transfer operations of large files.

[![npm version](https://badge.fury.io/js/@spoonconsulting%2Fcordova-plugin-background-upload.svg)](https://badge.fury.io/js/@spoonconsulting%2Fcordova-plugin-background-upload)
[![Build Status](https://travis-ci.com/spoonconsulting/cordova-plugin-background-upload.svg?branch=master)](https://travis-ci.org/spoonconsulting/cordova-plugin-background-upload)

**Supported Platforms**

- iOS
- Android

**Installation**

To install the plugin:

```
cordova plugin add @spoonconsulting/cordova-plugin-background-upload --save
```

To uninstall this plugin:

```
cordova plugin rm @spoonconsulting/cordova-plugin-background-upload
```

**Sample usage**

The plugin needs to be initialised before any upload. Ideally this should be called on application start. The uploader will provide global events which can be used to check the progress of the uploads. By default, the maximum number of parallel uploads allowed is set to 1. You can override it by changing the configuration on init.

```javascript
declare var FileTransferManager: any;
var config = {};
var uploader = FileTransferManager.init(config, callback);
```

**Methods**

### uploader.init(config, callback)

Initialises the uploader with provided configuration. To control the number of parallel uploads, pass `parallelUploadsLimit` in config.
The callback is used to track progress of the uploads
`var uploader = FileTransferManager.init({parallelUploadsLimit: 2}, event => {});`

### uploader.startUpload(payload)

Adds an upload. In case the plugin was not able to enqueue the upload, an error will be emitted in the global event listener.

```javascript
var payload = {
    id: "c3a4b4c7-4f1e-4c69-a951-773602e269fb",
    // eg: Android: 'file:///storage/emulated/0/Android/data/com.simpro.mobile/files/somefile.jpeg',
    // also supports content:// paths.
    // iOS: /var/mobile/Containers/Data/Application/396C27F0-1E40-4003-A605-DDFFFC716747/Library/NoCloud/photo-5.jpg
    filePath: "file:///storage/emulated/0/Android/data/com.simpro.mobile/files/somefile.jpeg",
    fileKey: "file",
    serverUrl: "<S3 Presigned URL to make PUT request to>",
    notificationTitle: "Uploading images",
};
uploader.startUpload(payload);
```

Param | Description
-------- | -------
id | a unique id of the file (UUID string)
filePath | the absolute path for the file to upload
fileKey | the name of the key to use for the file
serverUrl | remote server url
headers | custom http headers
parameters | custom parameters for multipart data
notificationTitle | Notification title when file is being uploaded (Android only)

### uploader.removeUpload(uploadId, successCallback, errorCallback)

Cancels and removes an upload

```javascript
uploader.removeUpload(uploadId, function () {
    //upload aborted
}, function (err) {
    //could not abort the upload
});
```

### uploader.acknowledgeEvent(eventId)

Confirms event received and remove it from plugin cache

```javascript
uploader.acknowledgeEvent(eventId);
```

The uploader will provide global events which can be used to check the status of the uploads.

```javascript
FileTransferManager.init({}, function (event) {
    if (event.state == 'UPLOADED') {
        console.log("upload: " + event.id + " has been completed successfully");
        console.log(event.statusCode, event.serverResponse);
    } else if (event.state == 'FAILED') {
        if (event.id) {
            console.log("upload: " + event.id + " has failed");
        } else {
            console.error("uploader caught an error: " + event.error);
        }
    } else if (event.state == 'UPLOADING') {
        console.log("uploading: " + event.id + " progress: " + event.progress + "%");
    }
});

```

To prevent any event loss while transitioning between native and Javascript side, the plugin stores success/failure events on disk. Once you have received the event, you will need to acknowledge it else it will be broadcast again when the plugin is initialised. Progress events do not have eventId and are not persisted.

```javascript
if (event.eventId) {
    uploader.acknowledgeEvent(event.eventId, function(){
        //success
    }, function (error){
        //error
    });
}
```

An event has the following attributes:

Property | Comment
-------- | -------
id | id of the upload
state | state of the upload (either `UPLOADING`, `UPLOADED` or `FAILED`)
statusCode | response code returned by server after upload is completed
serverResponse | server response received after upload is completed
error | error message in case of failure
errorCode | error code for any exception encountered
progress | progress for ongoing upload
eventId | id of the event

## iOS

The plugin runs on ios 10.0 and above and internally uses [AFNetworking](https://github.com/AFNetworking/AFNetworking). AFNetworking uses [NSURLSession](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/URLLoadingSystem/Articles/UsingNSURLSession.html#//apple_ref/doc/uid/TP40013509-SW44) under the hood to perform the upload in a background session. When an upload is initiated, it will continue until it has been completed successfully or until the user kills the application. If the application is terminated by the OS, the uploads will still continue. When the user relaunches the application, after calling the init method, events will be emitted with the ids of these uploads. If the user kills the application by swiping it up from the multitasking pane, the uploads will not be continued. Upload tasks in background sessions are automatically retried by the URL loading system after network errors as decided by the OS.

## Android

The minimum API level required is 21(Android 5) and the background file upload is handled by the WorkMananger library. If you have configured a notification to appear in the notifications area, the uploads will continue even if the user kills the app manually. If an upload is added when there is no network connection, it will be retried as soon as the network becomes reachable unless the app has already been killed.

On Android 12 and above, there are strict limitations on background services that does not allow to start a Foreground Service when the app is processing in background(<https://developer.android.com/guide/components/foreground-services>). Hence to prevent this on Android 12 and above, we have a classic notification. On Android 11 and below, we are still using foreground service along with WorkManager to start the notification.

> Note: Please add the following permission to your android manifest:

```xml
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
    <uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />
    <uses-permission android:maxSdkVersion="32" android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:maxSdkVersion="29" android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

## License

cordova-plugin-background-upload is licensed under the Apache v2 License.

## Credits

cordova-plugin-background-upload is brought to you by [Spoon Consulting Ltd] (<http://www.spoonconsulting.com/>).
