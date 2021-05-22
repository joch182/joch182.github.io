---
layout: post
title: Propagation delay analysis tool for 3G/4G with Python in Huawei Equipment
date: 2020-07-25 13:00:00 -0500
description: This post shows how use python to create a propagation delay analysis tool to help RF engineers locating cells with high overshooting problems
img: posts_imgs/3G_TP_plot_ge.jpg
tags: [python, Google Earth, Huawei, RF, LTE, 3G]
---

Recently i have found an excel macro tool to generate propagation delay plots for each 3G cell. It was very interesting and also helpful for us, RF engineers who are constantly optimizing RF coverage. Unfortunetely the macro was protected by password so it was not possile to apply the same concep to create a 4G tool. Also the fact that it was made with excel was kind of dissapointing (i am not very fan of excel macros) so i decided to first replicate the same principle with Python and then apply it to create the LTE tool. 

## Basic operations to get the KMZ

Basically, there are 2 main operations to prepare the KMZ file.

* We need to use geometryic calculations to get each point (coordinates in degrees) to be graphed in Google earth. For this we use: location (coordinates) of the cell, the distance for each point (depends on the propagation delay counters used) and the Horizontal Beamwidth of the antenna (usually 60 degrees)

* A function to generate an XML file with some modifications (like saved as KMZ) to be opened in Google Earth

Next code block shows the 2 calculations required to get a latitude and longitude with only initial coordinate (cell location), distance from that initial coordinate to the new point, earth radius in Km (6378.1) and the angle (in radians) measure form the north (clockwise)between the initial point and the new point

```python
{
    lat2 = math.asin(math.sin(originLat)*math.cos(distance/R) + math.cos(originLat)*math.sin(distance/R)*math.cos(angleRad))
    lon2 = originLon + math.atan2(math.sin(angleRad)*math.sin(distance/R)*math.cos(originLat), math.cos(distance/R)-math.sin(originLat)*math.sin(lat2))
}
```

To generate the XML file we use a very popular library called: 

- xml.etree.ElementTree

We also use 'minidom' for making it pretty (indentation) before saving the file (as .kmz)

The file should start like this:

```python
    xml_doc = ET.Element('kml')
    xml_doc.set('xmlns', 'http://www.opengis.net/kml/2.2')
    xml_doc.set('xmlns:gx', 'http://www.google.com/kml/ext/2.2')
    xml_doc.set('xmlns:kml', 'http://www.opengis.net/kml/2.2')
    xml_doc.set('xmlns:atom', 'http://www.w3.org/2005/Atom')

    document = ET.SubElement(xml_doc, 'Document')
    ET.SubElement(document, 'name').text = '3G_TP_Analiysis.KMZ'
```

As you can see the parent element is called 'kml' and not 'xml'. For more details about how to structure the data for Google Earth purposes, cleck [here](https://developers.google.com/kml/documentation/kml_tut "KML Tutorial")

Since the principle is very similar to get the propagation delay analysis in both 3G and 4G, there are some minor changes to perform (mostly due to the LTE counters for propagation delay), so as you may realize both tools are very similar between each other.

## Use/Share/Fork/Improve it

Next, you can find the repo from GitHub whe you can download the tool for 3G scenario and start using on your daily activities. I just expect you can share, improve it as much as possible. If you find any bug please let me know or just open a pull request.

## [3G tool](https://github.com/joch182/3G-Huawei-Propagation-Delay-Tool-Google-Earth)

## NOTICE

Tool is created to be used only with Huawei counters,it should be possible to modify the source code to use it with other vendor data, nevertheless if you are trying to use it as it is please follow the instructions, specially when importing the data and EPT tables, those files must follow the exact format shown in the template