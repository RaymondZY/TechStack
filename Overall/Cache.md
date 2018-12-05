# Android
## .AndroidStudio (Intellij IDEA)
AndroidStudio/bin/idea.properties :
```
idea.config.path=E:/Cache/dev/.AndroidStudio/config
idea.system.path=E:/Cache/dev/.AndroidStudio/system
```

## .android
环境变量:
```
ANDROID_SDK_HOME = E:/Cache/Dev
```

# Gradle
## .AndroidStudio (Intellij IDEA)
```
File -> Settings -> Build, Execution, Deployment -> Gradle
```

## Command Line
环境变量:
```
GRADLE_USER_HOME = E:/Cache/Dev/.gradle
```

## ShadowSocks proxy for Gradle
GRADLE_USER_HOME/gradle.properties :
```
systemProp.http.proxyHost=127.0.0.1
systemProp.http.proxyPort=1080
systemProp.https.proxyHost=127.0.0.1
systemProp.https.proxyPort=1080
```
