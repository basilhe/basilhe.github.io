---
layout: post
title: "CocoaPods The Dependency Manager for Objective C"
description: ""
category: ios
tags: [ios,cocoapods]
---
{% include JB/setup %}

## References
 * [使用CocoaPods来做iOS程序的包依赖管理](http://blog.devtang.com/blog/2012/12/02/use-cocoapod-to-manage-ios-lib-dependency/)
 * [CocoaPods一个Objective-C第三方库的管理利器](http://blog.csdn.net/totogo2010/article/details/8198694#reply)
 * [CocoaPods 官方网站](http://www.cocoapods.org/)
 * [Create a CocoaPods of your library ](http://www.lafosca.cat/create-a-cocoapods-of-your-library/)

## [Private Pods](http://guides.cocoapods.org/making/private-cocoapods.html) & [Making a CocoaPod](http://guides.cocoapods.org/making/making-a-cocoapod.html)

### Add your Private Repo to your CocoaPods installation

<pre>
$ pod repo add $NAME $PRIVATE_GIT_URL
</pre>

### Create & Publish Private Pods

#### Create a new Pod from scratch

<pre>
$ pod lib create [NAME] 
</pre>

#### Port current project to a Pod

<pre>
$ pod spec create [ NAME | git@HOST:USER/REPO ]
</pre>

#### Release Pod

<pre>
$ cd ~/code/Pods/NAME
$ edit NAME.podspec
# set the new version to 0.0.1
# set the new tag to 0.0.1
$ pod lib lint

$ git add -A && git commit -m "Release 0.0.1."
$ git tag '0.0.1'
$ git push
$ git push --tags

# push to private pod repository
$ pod push $NAME
</pre>

h4. Vendored_frameworks

The paths of the framework bundles that come shipped with the Pod.

Examples:
<pre>
spec.ios.vendored_frameworks = 'Frameworks/MyFramework.framework'
spec.vendored_frameworks = 'MyFramework.framework', 'TheirFramework.framework'
</pre>