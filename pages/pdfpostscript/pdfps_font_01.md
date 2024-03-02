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

Orion Rport Designer supports various output formats including PDF, PostScript, HTML-5, Images(png, jpg, tiff, bmp) and also supports WPF, WinForms, Web design user interfaces.
Each output handles text fonts in different ways, 

- WPF supports TTF(TreuType Font) and OTF(OpenType Font)
- WinForms only supports TTF
- PostScript supports PS specific fonts which are not compatible with other output formats
- HTML supports TTF/OTF, WOFF, WOFF2

Drawing the exactly same text in different output formats is a key issue. 

So, for the unified way of drawing texts in PDF, PostScript, HTML-5 Canvas, WPF and WinForms, WPF font geometry would be a good choice. WPF can extract font geometry from TTF/OTF font files and WPF geometry can be converted to GDI+ Graphics Path. GDI+ Graphics Path also converted to PostScript path and HTML-5 Canvas Path. 

Using path based text presentation requires some tricks for better speed and smaller outputs.

- WPF Geometries/GDI+ graphics pathes should be cached
    - Extracting geometry from font file and converion takes time
- PostScript and HTML-5 Canvas font pathes should be reused
    - In PostScript, once a character is used, put its path into the header of PS file then call the definition when reuse the character
    - In HTML, the path of a character is defined as a function at the top of JavaScript, then the function is called where the character appears
- PostScript header and body should be compressed
- To support extended font styles like reverse-order, flips(horizontally, vertically), width-height ratio, outlined, all output formats should have commands for transforming the pathes 

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


## WPF Geometry To GDI+ Graphics Path

```C#
public static System.Drawing.Drawing2D.PathData GeometryToGraphicsPath(System.Windows.Media.Geometry lGeom)
{
    System.Drawing.Drawing2D.PathData lPData = null;
    try
    {
        if (lGeom == null)
            return lPData;

        if (lGeom.IsEmpty())  // for SPACE
        {
            lPData = new System.Drawing.Drawing2D.PathData();
            if (lPData.Points == null)
                lPData.Points = new System.Drawing.PointF[1] { new System.Drawing.PointF() };
            if (lPData.Types == null)
                lPData.Types = new byte[1] { 0 };
            return lPData;
        }
        using (System.Drawing.Drawing2D.GraphicsPath lGPath = new System.Drawing.Drawing2D.GraphicsPath())
        {
            PathGeometry lPathGeom = lGeom.GetOutlinedPathGeometry();
            string lsGeom = lPathGeom.ToString();
            lsGeom = lsGeom.Replace('E', 'e');
            Svg2.Pathing.SvgPathSegmentList llSvgPathSegments = Svg2.SvgPathBuilder.Parse(lsGeom);
            foreach (Svg2.Pathing.SvgPathSegment lSvgPath in llSvgPathSegments)
            {
                lSvgPath.AddToPath(lGPath);
            }
            lPData = lGPath.PathData;
            if (lPData.Points == null)
                lPData.Points = new System.Drawing.PointF[1] { new System.Drawing.PointF() };
            if (lPData.Types == null)
                lPData.Types = new byte[1] { 0 };
        }

        for (int IX = 0;IX < lPData.Points.Length;IX++)
        {
            lPData.Points[IX] = new System.Drawing.PointF((float)UCNV.GetPointFromDIU(lPData.Points[IX].X), (float)UCNV.GetPointFromDIU(lPData.Points[IX].Y));
        }                
    }
    catch (Exception lExe)
    {
        lPData = null;
        ORIONDEBUG.LOG(LogInfo.EnumLogLevel.ERROR, "WpfFontCache::ToGraphicsPath()", lExe);
    }

    return lPData;
}
```

