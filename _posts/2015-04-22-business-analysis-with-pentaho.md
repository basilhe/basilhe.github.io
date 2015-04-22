---
layout: post
title: "Business Analysis with Pentaho"
description: "Business Analysis with Pentaho， Mondrian"
category: bi
tags: [pentaho,mondrian,bi]
---
{% include JB/setup %}

Pentaho Business Analysis Platform

# Installation

* http://community.pentaho.com/
* http://community.pentaho.com/projects/mondrian/
* 本地安装位置： s60:~/download/biserver-ce

# Configuration

* [Setting the Active Hadoop Configuration](http://infocenter.pentaho.com/help/index.jsp?topic=%2Fbigdata_guide%2Freference_active_hadoop_configuration.html)

		# Edit pentaho-solutions/system/kettle/plugins/pentaho-big-data-plugin/plugin.properties.
		# The Hadoop Configuration to use when communicating with a Hadoop cluster. 
		# This is used for all Hadoop client tools including HDFS, Hive, HBase, and Sqoop.
		active.hadoop.configuration=cdh52
		
# Login

http://s60:10080/pentaho

default account: admin/password

# CDE 绘制统计报表

* [Pentaho CDE 教程（三）走进CDE 之 联动](http://alenzhai.iteye.com/blog/2059879)
* [Creating Dashboards with CDE](http://type-exit.org/adventures-with-open-source-bi/2011/06/creating-dashboards-with-cde/)

# Mondrian OLAP

[Introduction to Mondrian OLAP schema](http://codeissue.com/articles/a04edc3096d9c7d/introduction-to-mondrian-olap-schema)
