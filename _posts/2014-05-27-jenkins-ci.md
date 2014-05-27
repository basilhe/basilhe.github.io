---
layout: post
title: "Jenkins CI"
description: ""
category: devops
tags: [jenkins,ci]
---
{% include JB/setup %}

[Jenkins CI](http://jenkins-ci.org/) on [iMac Build Machine](http://192.168.1.118:8080)
=======================================================================================

Plugins
-------

-   [Environment Injector
    Plugin](https://wiki.jenkins-ci.org/display/JENKINS/EnvInject+Plugin)
-   [Jenkins GIT
    plugin](http://wiki.jenkins-ci.org/display/JENKINS/Git+Plugin)
-   [CocoaPods Jenkins
    Integration](http://wiki.jenkins-ci.org/display/JENKINS/CocoaPods+Integration)
-   [Xcode
    Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Xcode+Plugin)
-   [Credentials
    Plugin](http://wiki.jenkins-ci.org/display/JENKINS/Credentials+Plugin)
-   [OTA Builder
    Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Over-the-Air+Ad+Hoc+Deployment+Plugin+For+iOS)
-   [Gitlab Hookk
    Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Gitlab+Hook+Plugin)
-   [More Jenkins
    Plugins……](https://wiki.jenkins-ci.org/display/JENKINS/Plugins)

## GitLab CI 访问权限

CI 目前是通过用户 *builder* 访问GitLab项目
对GitLab上的项目，为了能使Jenkins CI能够访问代码进行打包，需要将用户 *builder* 加入到对应的项目中，角色只需 *Reporter* 即可，无需 Developer 或更高角色。

## iOS Projects

### Xcode-Select

Specify Xcode with Enrionment Injector Plugin: 
 
 * 5.1: *DEVELOPER_DIR=/Applications/Xcode5.1.app/Contents/Developer*
 * 5.0: *DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer*
 * 4.6: *DEVELOPER_DIR=/Applications/Xcode 4.6.app/Contents/Developer* 
 * 4.5: *DEVELOPER_DIR=/Applications/Xcode 4.5.app/Contents/Developer*

![envionment_inject.png](/assets/images/jenkins-ci/envionment_inject.png)

### Specify Target or Scheme

* Adjust dependencies to make command line build success, which you can test with command *"/usr/bin/xcodebuild -alltargets -configuration Release clean build"*
![build_dependency.png](/assets/images/jenkins-ci/build_dependency.png)
or
![scheme_dependency.png](/assets/images/jenkins-ci/scheme_dependency.png)
* Make build target scheme shared for projects using CocoaPods, which you can test with command *"/usr/bin/xcodebuild -scheme Lifesense -workspace Lifesense.xcworkspace -configuration Release clean build"*
![sheme_shared.png](/assets/images/jenkins-ci/sheme_shared.png)

### Xcode Plugin Parameters

* Specify IPA name pattern(VARS: BASE_NAME, VERSION, SHORT_VERSION, BUILD_DATE), code sign 
![xcode_ipa_parameters.png](/assets/images/jenkins-ci/xcode_ipa_parameters.png)
* Specify other Advanced Xcode build options
> * Xcode Schema File or Target
> * Xcode Workspace File or Xcode Project Directory if necessary
> * Build output directory -> "${WORKSPACE}/build"

## Android Projects

### Use "Android Tool":http://developer.android.com/tools/help/android.html to create Ant build file for command line build

<pre>
$ android update project -p $PATH -n $NAME
$ android update lib-project  -p weibo.sdk.android
</pre>

### Apply custom rules to rename destination APK with version and build time, named as custom_rules.xml

<pre>
&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;project&gt;

&lt;target name="rename-release-with-version-number"&gt;
&lt;xmlproperty file="AndroidManifest.xml" 
prefix="themanifest" 
collapseAttributes="true"/&gt;

&lt;!--  see ${sdk.dir}/tools/ant/build.xml -set-release-mode --&gt;
&lt;tstamp&gt;
&lt;format property="TODAY"
pattern="yy.MM.dd"
locale="en,zh"/&gt;
&lt;/tstamp&gt;
&lt;property name="out.packaged.file" 
location="${out.absolute.dir}/${ant.project.name}-${themanifest.manifest.android:versionName}-${TODAY}-release-unsigned.apk" /&gt;
&lt;property name="out.final.file" 
location="${out.absolute.dir}/${ant.project.name}-${themanifest.manifest.android:versionName}-${TODAY}-release.apk" /&gt;
&lt;/target&gt;
&lt;target name="-set-release-mode"
depends="rename-release-with-version-number,android_rules.-set-release-mode"&gt;
&lt;echo message="target: ${build.target}"&gt;&lt;/echo&gt;
&lt;/target&gt;

&lt;/project&gt; 
</pre>

### Inject SDK Home environment

<pre>
ANDROID_HOME=~/tool/android-sdks
</pre>

### "Enable ProGuard":http://developer.android.com/tools/help/proguard.html#enabling

### Invoke Ant to build target

![android_ant_build.png](/assets/images/jenkins-ci/android_ant_build.png)

## Trigger

###  Poll SCM with notifyCommit

Add Web Hooks to gitlab project with URL: *http://192.168.1.118:8080/gitlab/notify_commit*
And setup Build Triggers on Jenkins

![build_triggers.png](/assets/images/jenkins-ci/build_triggers.png)

### Invoke Build Now directly

Add Web Hooks to gitlab project with URL: *http://192.168.1.118:8080/gitlab/build_now*

![gitlab_web_hooks.png](/assets/images/jenkins-ci/gitlab_web_hooks.png)

## E-mail notification

![email_notification.png](/assets/images/jenkins-ci/email_notification.png)

## Unit Test, TODO

## Copy build result to specified directory

### iOS

<pre>
cp $WORKSPACE/build/archive/* ~/Buildresults/$TARGET/
</pre>

### Android

<pre>
cp bin/*release.apk ~/Buildresults/$TARGET
</pre>



 