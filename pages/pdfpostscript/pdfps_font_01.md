---
title: Fonts in PDF and PostScript 
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: pdfps_font_01.html
folder: pdfpostscript
---

## Overview

For the unified way of drawing texts in PDF, PostScript, HTML-5 Canvas, WPF and WinForms, WPF would be a good choice. WPF font can draw TTF(TrueType Font), OTF(OpenType Font, which is not supported by GDI+) and WPF geometry can be converted to GDI+ Graphics Path. GDI+ Graphics Path also converted to PostScript path and HTML-5 Canvas Path.  
All fonts have fixed size(10pt) with normal style and build its geometry then put the geometry into the caches.



## Getting WPF Font Geometry



```C#
public static Geometry BuildGeometry(string lsFntFamily, string lsCH, double positionXDiu, double positionYDiu)
{
    System.Windows.Media.Geometry lGeom = null;

    try
    {
        System.Windows.Media.FormattedText lFTXT = new System.Windows.Media.FormattedText(lsCH,
                                                            System.Threading.Thread.CurrentThread.CurrentUICulture,
                                                            FlowDirection.LeftToRight,
                                                            new Typeface(lsFntFamily),
                                                            UCNV.GetDIUFromPoint(_DEFFONTSIZE),
                                                            System.Windows.Media.Brushes.Black);

        lGeom = lFTXT.BuildGeometry(new Point(positionXDiu, positionYDiu));
    }
    catch (Exception lExe)
    {
        lGeom = null;
        ORIONDEBUG.LOG(LogInfo.EnumLogLevel.ERROR, "WpfFontCache::BuildGeometry()", lExe);
    }

    return lGeom;
}
```


### Entry Points to Create PDF, PostScript Charts

