---
title: Generate non comma delimiter CSV file
date: 2020-12-07 10:21:13
categories: [IT]
tags: [file system, csv]
toc: true
cover: /img/waytoexcel.jpg
thumbnail: /img/csvicon.png
---

This is some tip but sometime makes you life easier when you work with business team and deal with data from business input. Excel spreadsheet is kind of standard file format to communicate with business team, but in data manipulation side, it's not a ideal input data format, technically, we usually convert spreadsheet to csv file. But default comma delimiter might cause some trouble because business data might contains quite a lot of `,` in attributes such as comments, suggestions, reasons etc. In order to better identify the column, non-comma delimiter should be used like `|` pipe.

How do we generate the `|` delimiter csv file. There are two ways

<!-- more -->

## Change format setting system-wise

In windows, open the control panel, find the `Region` setting, on the `Formats` tab click `Additional setting`, a pop-up window will show up, on that window, find the option `list separator` then type whatever delimiter you want to setup like `|`. 

![region_settting.png](/img/screenshots/region_settting.png))

save the setting, when you `Save as` csv file in excel, it will generate `|` separated csv file afterwards.

## Use database client tool to save query result to csv file with customized delimiter

Some sql database client applications provide the functionality to save query result to csv file, like WinSQL.

Before running your query, select execute to save to a text file, then select `|` from drop-down list

![export_file.png](/img/screenshots/export_file.png)

![pipe_delimiter.png](/img/screenshots/pipe_delimiter.png)

