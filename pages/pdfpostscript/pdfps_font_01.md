---
title: Fonts in PDF, PostScript and HTML-5 Canvas
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
- To support extended font styles like reverse-order, flips(horizontally, vertically), width-height ratio, outlined, character gap, line gap all output formats should have commands for transforming the pathes 

## Using WPF Font Geometry

### Getting WPF Font Geometry

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

### Drawing WPF Font Geometry

- Set rotation
    >Matrix lMtrx = new Matrix();<br/>
    if (lCD.fRotation != 0F)<br/>
    {<br/>
        Point lptRotateCenterPercent = WPF.UC_Item.GetRotateCenterPercent(lCD);<br/>
        lMtrx.RotateAt(lCD.fRotation, lSzTBoxDIU.Width * lptRotateCenterPercent.X, lSzTBoxDIU.Height * lptRotateCenterPercent.Y);<br/>
    }<br/>
    lCD.cItemCanvas.RenderTransform = new MatrixTransform(lMtrx);<br>

- Get geometry from cache
    >System.Windows.Media.Geometry lGeom = GlyphInfoCache.GetWpfGeomCache(lGlyph, lPD.fWidthDPI, lPD.fHeightDPI, ldFontSize, ldFontScaleW, 1F);

- Set Italic, Horizontal/Vertical Flips
    >lMtrx = ((MatrixTransform)lGeom.Transform).Matrix;<br/>
    if (lCD.cbFontFlipVert)<br/>
        lMtrx.Append(new Matrix(1, 0, 0, -1, 0, UCNV.GetDIUFromPixel(lSZFCH.CharHeight, lPD.fHeightDPI)));<br/>
    if (lCD.cbFontFlipHorz)<br/>
        lMtrx.Append(new Matrix(-1, 0, 0, 1, UCNV.GetDIUFromPixel(lSZFCH.GapHorz, lPD.fWidthDPI), 0));<br/>
    if (lCD.bFontStyleItalic)<br/>
    {<br/>
        double ldItalicization = -0.35;// -lfFontSizePX / UCNV.GetPixelFromPoint(50F, lPD.fWidthDPI);<br/>
        lMtrx.Append(new Matrix(1, 0, ldItalicization, 1, UCNV.GetDIUFromPixel(lSZFCH.CharHeight, lPD.fWidthDPI) * 0.35, 0));<br/>
    }<br/>
    lGeom.Transform = new MatrixTransform(lMtrx);<br/>

- Set font size
    - WPF Font is extracted 10-point of size, and scale it to actual size when used
    >double ldFontSizeScale = ldFontSizePT / GlyphInfoCache._DEFFONTSIZE;<br/>
    ldFontScaleW *= ldFontSizeScale;<br/>
    ldFontScaleH *= ldFontSizeScale;<br/>
    lGeom = lGlyph.cGeom.Clone();<br/>
    System.Windows.Media.Matrix lMtrx = new System.Windows.Media.Matrix();<br/>
    lMtrx.Scale(ldFontScaleW, ldFontScaleH);<br/>
    lGeom.Transform = new MatrixTransform(lMtrx);<br/>

```C#
...

Matrix lMtrx = new Matrix();
if (lCD.fRotation != 0F)
{
    Point lptRotateCenterPercent = WPF.UC_Item.GetRotateCenterPercent(lCD);
    lMtrx.RotateAt(lCD.fRotation, lSzTBoxDIU.Width * lptRotateCenterPercent.X, lSzTBoxDIU.Height * lptRotateCenterPercent.Y);
}
lCD.cItemCanvas.RenderTransform = new MatrixTransform(lMtrx);

double ldWpfStringPosAdjustX = 0; //WpfFontInfo._DEFFONTSIZE / 5;
double ldWpfStringPosAdjustY = 0; // WpfFontInfo._DEFFONTSIZE / 10;

foreach (OrionTextBoxInfo.LineChars lLine in llLNChars)
{
    if (lLine.LSZFCH == null || lLine.LSZFCH.Count <= 0)
        continue;

    Point lPtRPos = new Point(UCNV.GetDIUFromPixel(lLine.PTF.X, lPD.fWidthDPI) + ldWpfStringPosAdjustX,
                                UCNV.GetDIUFromPixel(lLine.PTF.Y, lPD.fHeightDPI) - ldWpfStringPosAdjustY);
    double ldInitPosX = lPtRPos.X;
    double ldInitPosY = lPtRPos.Y;
    double ldLineWidth = 0;
    //
    List<OrionConfigInfo.OrionColor> llColors = new List<OrionConfigInfo.OrionColor>();
    int liColorIX = 0;
    if (lCD.cColorList.cbUseColorList)
        llColors = lCD.cColorList.GetColors(lLine.LSZFCH.Count);
    //
    foreach (OrionTextBoxInfo.SizeFChar lSZFCH in lLine.LSZFCH)
    {
        if (lCD.cColorList.cbUseColorList)
        {
            lForeColor = new NewColor(llColors[liColorIX]);
            liColorIX++;
        }
        GlyphInfoCache lGlyph = lSZFCH.cGlyph;
        if (lGlyph == null || lGlyph.cGeom == null)
            throw new Exception("OrinWpfDesigner::DrawData_WPF() GlyphInfoCache:lGlyph is NULL");

        System.Windows.Media.Geometry lGeom = GlyphInfoCache.GetWpfGeomCache(lGlyph, lPD.fWidthDPI, lPD.fHeightDPI, ldFontSize, ldFontScaleW, 1F);
        if (lGeom == null)
            throw new Exception("OrinWpfDesigner::DrawData_WPF() GlyphInfoCache.GetWpfGeomCache() Return value is NULL");

        if (lCD.bFontStyleItalic || lCD.cbFontFlipVert || lCD.cbFontFlipHorz)
        {
            lMtrx = ((MatrixTransform)lGeom.Transform).Matrix;
            if (lCD.cbFontFlipVert)
                lMtrx.Append(new Matrix(1, 0, 0, -1, 0, UCNV.GetDIUFromPixel(lSZFCH.CharHeight, lPD.fHeightDPI)));
            if (lCD.cbFontFlipHorz)
                lMtrx.Append(new Matrix(-1, 0, 0, 1, UCNV.GetDIUFromPixel(lSZFCH.GapHorz, lPD.fWidthDPI), 0));
            if (lCD.bFontStyleItalic)
            {
                double ldItalicization = -0.35;// -lfFontSizePX / UCNV.GetPixelFromPoint(50F, lPD.fWidthDPI);
                lMtrx.Append(new Matrix(1, 0, ldItalicization, 1, UCNV.GetDIUFromPixel(lSZFCH.CharHeight, lPD.fWidthDPI) * 0.35, 0));
            }
            lGeom.Transform = new MatrixTransform(lMtrx);
        }

        GlyphInfoCache.DrawPathData_WPF(lCD.cItemCanvas, lPD, lCD, lSZFCH, ldFontSize, lForeColor, lGeom, lPtRPos);

        ldLineWidth += UCNV.GetDIUFromPixel(lSZFCH.GapHorz, lPD.fWidthDPI);
        lPtRPos.X += UCNV.GetDIUFromPixel(lSZFCH.GapHorz, lPD.fWidthDPI);
    }
    if (lCD.bFontStyleStrikeout || lCD.bFontStyleUnderline)
    {
        this.DrawSThru_ULine_WPF(lCD.cItemCanvas, lCD, lPD, lLine, ldInitPosX, ldInitPosY, ldLineWidth);
    }
    
}


... GlyphInfoCache.cs

public static bool DrawPathData_WPF(Canvas lCNVS, OD_PageData lPD, OD_ColumnData lCD,
                    OrionTextBoxInfo.SizeFChar lSZFCH, double ldFontSizePT, NewColor lncrFore,
                    System.Windows.Media.Geometry lGeom, System.Windows.Point lptPos)
{
    bool lbSuccess = false;
    try
    {
        //
        if (lSZFCH.cGlyph != null && lSZFCH.cGlyph.isSubstituted && lPD.fontSubstitutionPositionAdjust)
        {
            double fontScale = ldFontSizePT / GlyphInfoCache._DEFFONTSIZE;
            double adjustSubstitutedFontPositionY = UCNV.GetDIUFromPoint(lSZFCH.cGlyph.baselineDifference) * fontScale * 0.7;
            lptPos.Y += adjustSubstitutedFontPositionY;
        }
        //
        System.Windows.Shapes.Path lFntPath = new System.Windows.Shapes.Path();
        lFntPath.Data = lGeom;
        lFntPath.Margin = new Thickness(lptPos.X, lptPos.Y, 0, 0);

        if (lCD.cbFontOutline)
        {
            lFntPath.Stroke = new SolidColorBrush(System.Windows.Media.Color.FromRgb(lncrFore.RGB.R, lncrFore.RGB.G, lncrFore.RGB.B));
            lFntPath.StrokeThickness = UCNV.GetDIUFromMM(lCD.cfPenWidth);
            if (lFntPath.StrokeThickness < 0.2)
                lFntPath.StrokeThickness = 0.2;
            lFntPath.StrokeDashArray = OrionWpfDesigner.SetLineDash(lCD.cPenDashStyle);
            lCNVS.Children.Add(lFntPath);
        }
        else
        {
            lFntPath.Fill = new SolidColorBrush(System.Windows.Media.Color.FromRgb(lncrFore.RGB.R, lncrFore.RGB.G, lncrFore.RGB.B));
            if (lCD.bFontStyleBold)
            {
                lFntPath.Stroke = new SolidColorBrush(System.Windows.Media.Color.FromRgb(lncrFore.RGB.R, lncrFore.RGB.G, lncrFore.RGB.B));
                lFntPath.StrokeThickness = UCNV.GetDIUFromPoint(ldFontSizePT / 100.0 * 4.0);
                if (lFntPath.StrokeThickness < 0.2)
                    lFntPath.StrokeThickness = 0.2;
            }
            lCNVS.Children.Add(lFntPath);
        }

        lbSuccess = true;
    }
    catch (Exception lExe)
    {
        lbSuccess = false;
        ORIONDEBUG.LOG(LogInfo.EnumLogLevel.ERROR, "GlyphInfoCache::DrawPathData_WPF()", lExe);
    }
    finally
    {
    }

    return lbSuccess;
}

...
```
## Using Font in PostScript with GDI+ Graphics Path

### Converting WPF Geometry To GDI+ Graphics Path

- Since, there is no way to convert WPF PathGeometry to GDI+ Graphics Path directly, I used [SVG Path](https://github.com/svg-net/SVG/tree/master/Source/Paths) converstion. 
    - Fitst, WPF PathGeometry is converted to SVG Path
    >PathGeometry lPathGeom = lGeom.GetOutlinedPathGeometry();<br/>
    string lsGeom = lPathGeom.ToString();<br/>
    
    - Then, convert [SVG](https://github.com/svg-net/SVG/tree/master/Source/Paths) to GDI+ Path
    >lsGeom = lsGeom.Replace('E', 'e');<br/>
    Svg2.Pathing.SvgPathSegmentList llSvgPathSegments = Svg2.SvgPathBuilder.Parse(lsGeom);<br/>
    foreach (Svg2.Pathing.SvgPathSegment lSvgPath in llSvgPathSegments)<br/>
    {<br/>
        lSvgPath.AddToPath(lGPath);<br/>
    }<br/>
    lPData = lGPath.PathData;<br/>


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


### GDI+ Graphics Path To PostScript Path



```C#
internal void WriteUPath(string lsPathName, PathData lPathD)
{
    UPathHeader(lsPathName);

    PointF[] lptfaPathPoint = lPathD.Points;
    List<PointF> llptfCurveTo = new List<PointF>();
    byte[] lbyaPathType = lPathD.Types;
    bool lbIsLastCommandCLOSEPATH = true;
    for (int idx = 0; idx < lptfaPathPoint.Length; idx++)
    {
        PointF lPP = lptfaPathPoint[idx];
        lPP.Y = -lPP.Y;
        byte lPT = lbyaPathType[idx];

        if (llptfCurveTo.Count == 3)
        {
            PS_curveto(llptfCurveTo);
            llptfCurveTo.Clear();
        }
        if (lPT >= 0x80)
        {
            if (llptfCurveTo.Count == 0)
            {
                PS_lineto(lPP.X, lPP.Y);
            }
            else
            {
                llptfCurveTo.Add(lPP);
                PS_curveto(llptfCurveTo);
            }
            llptfCurveTo.Clear();
        }
        if (lPT == 0x00)    // Indicates that the point is the start of a figure.
        {
            if (!lbIsLastCommandCLOSEPATH)
                PS_closepath();
            PS_moveto(lPP.X, lPP.Y);
            lbIsLastCommandCLOSEPATH = false;
        }
        else if (lPT == 0x01)   // Indicates that the point is one of the two endpoints of a line.
        {
            PS_lineto(lPP.X, lPP.Y);
            lbIsLastCommandCLOSEPATH = false;
        }
        else if (lPT == 0x03)   // Indicates that the point is an endpoint or control point of a cubic Bezier spline.
        {
            llptfCurveTo.Add(lPP);
            lbIsLastCommandCLOSEPATH = false;
            //PS_LineTo(lFStream, lPP.X, lPP.Y);
        }
        else if (lPT == 0x07)   // Masks all bits except for the three low-order bits, which indicate the point type.
        {
            lbIsLastCommandCLOSEPATH = false;
        }
        else if (lPT == 0x20)   // Specifies that the point is a marker.
        {
            lbIsLastCommandCLOSEPATH = false;
        }
        else if (lPT == 0x80)   // Specifies that the point is the last point in a closed subpath (figure).
        {
            PS_closepath();
            lbIsLastCommandCLOSEPATH = true;
        }
        else if (lPT == 0x83)
        {
            PS_closepath();
            lbIsLastCommandCLOSEPATH = true;
        }
        else if (lPT == 0xa3)
        {
            PS_closepath();
            lbIsLastCommandCLOSEPATH = true;
        }
        else
        {
        }
    }
    if (!lbIsLastCommandCLOSEPATH)
        PS_closepath();
    UPathTail();
}

private void UPathHeader(string lstrProcedureName)
{
    WriteLF();
    WriteUnicodeToASCII_LF("/" + lstrProcedureName);
    WriteUnicodeToASCII_LF("{");
}
private void UPathTail()
{
    WriteUnicodeToASCII_LF("} bind def");
}
internal void PS_curveto(List<PointF> llptfCurveTo)
{
    string lstrPostScript = string.Empty;
    for (int idx = 0; idx < llptfCurveTo.Count; idx++)
    {
        lstrPostScript += llptfCurveTo[idx].X.ToString("0.###") + " " + llptfCurveTo[idx].Y.ToString("0.###") + " ";
    }
    lstrPostScript += csPScurveto;
    WriteUnicodeToASCII_LF(lstrPostScript);
}
internal void PS_moveto(float lfX, float lfY)
{
    WriteUnicodeToASCII_LF(String.Format("{0:0.###} {1:0.###} {2}", lfX, lfY, csPSmoveto));
}
internal void PS_lineto(float lfX, float lfY)
{
    WriteUnicodeToASCII_LF(String.Format("{0:0.###} {1:0.###} {2}", lfX, lfY, csPSlineto));
}
internal void PS_closepath()
{
    WriteUnicodeToASCII_LF(csPSclosepath);
}

```

## Using 


## GDI+ Graphics Path to HTML-5 Canvas Path

```C#
public static void WriteGraphicsPath(StringBuilder lSB, string Ctx, PathData lPathD)
{
    try
    {
        HtmlPrint.beginPath(lSB, Ctx);
        PointF[] lptfaPathPoint = lPathD.Points;
        List<Point> llptCurveTo = new List<Point>();
        byte[] lbyaPathType = lPathD.Types;
        bool lbIsLastCommandCLOSEPATH = true;
        for (int idx = 0; idx < lptfaPathPoint.Length; idx++)
        {
            Point lPP = Point.Round(lptfaPathPoint[idx]);                    
            byte lPT = lbyaPathType[idx];
            if (llptCurveTo.Count == 3)
            {
                CurveTo(lSB, Ctx, llptCurveTo);
                llptCurveTo.Clear();
            }
            if (lPT >= 0x80)
            {
                if (llptCurveTo.Count == 0)
                {
                    HtmlPrint.lineTo(lSB, Ctx, lPP.X, lPP.Y);
                }
                else
                {
                    llptCurveTo.Add(lPP);
                    CurveTo(lSB, Ctx, llptCurveTo);
                    llptCurveTo.Clear();
                }
            }
            if (lPT == 0x00)    // Indicates that the point is the start of a figure.
            {
                if (!lbIsLastCommandCLOSEPATH)
                    HtmlPrint.closePath(lSB, Ctx);

                HtmlPrint.moveTo(lSB, Ctx, lPP.X, lPP.Y);
                lbIsLastCommandCLOSEPATH = false;
            }
            else if (lPT == 0x01)   // Indicates that the point is one of the two endpoints of a line.
            {
                HtmlPrint.lineTo(lSB, Ctx, lPP.X, lPP.Y);
                lbIsLastCommandCLOSEPATH = false;
            }
            else if (lPT == 0x03)   // Indicates that the point is an endpoint or control point of a cubic Bezier spline.
            {
                llptCurveTo.Add(lPP);
                lbIsLastCommandCLOSEPATH = false;
                //PS_LineTo(lFStream, lPP.X, lPP.Y);
            }
            else if (lPT == 0x07)   // Masks all bits except for the three low-order bits, which indicate the point type.
            {
                lbIsLastCommandCLOSEPATH = false;
            }
            else if (lPT == 0x20)   // Specifies that the point is a marker.
            {
                lbIsLastCommandCLOSEPATH = false;
            }
            else if (lPT == 0x80)   // Specifies that the point is the last point in a closed subpath (figure).
            {
                HtmlPrint.closePath(lSB, Ctx);
                lbIsLastCommandCLOSEPATH = true;
            }
            else if (lPT == 0x83)
            {
                HtmlPrint.closePath(lSB, Ctx);
                lbIsLastCommandCLOSEPATH = true;
            }
            else if (lPT == 0xa3)
            {
                HtmlPrint.closePath(lSB, Ctx);
                lbIsLastCommandCLOSEPATH = true;
            }
            else
            {
            }
        }
        if (llptCurveTo.Count == 3)
        {
            CurveTo(lSB, Ctx, llptCurveTo);
            llptCurveTo.Clear();

        }
        if (!lbIsLastCommandCLOSEPATH)
            HtmlPrint.closePath(lSB, Ctx);
    }
    catch (Exception lExe)
    {
        ORIONDEBUG.LOG(LogInfo.EnumLogLevel.ERROR, "HtmlPrint::WriteGraphicsPath()", lExe);
    }
}

public static void CurveTo(StringBuilder lSB, string Ctx, List<Point> llptCurveTo)
{
    if (llptCurveTo.Count == 2)
        HtmlPrint.quadraticCurveTo(lSB, Ctx, llptCurveTo[0].X, llptCurveTo[0].Y, llptCurveTo[1].X, llptCurveTo[1].Y);
    else if (llptCurveTo.Count == 3)
        HtmlPrint.bezierCurveTo(lSB, Ctx, llptCurveTo[0].X, llptCurveTo[0].Y, llptCurveTo[1].X, llptCurveTo[1].Y, llptCurveTo[2].X, llptCurveTo[2].Y);
}

...C#

public static void beginPath(StringBuilder lSB, string Ctx)
{
    lSB.Append(Ctx + ".beginPath();\n");
}
public static void closePath(StringBuilder lSB, string Ctx)
{
    lSB.Append(Ctx + ".closePath();\n");
}
//quadraticCurveTo(cp1x, cp1y, x, y)
public static void quadraticCurveTo(StringBuilder lSB, string Ctx, int cp1x, int cp1y, int x, int y)
{
    lSB.Append(Ctx + ".quadraticCurveTo(" + cp1x + "," + cp1y + "," + x + "," + y + ");\n");
}
//bezierCurveTo(cp1x, cp1y, cp2x, cp2y, x, y)
public static void bezierCurveTo(StringBuilder lSB, string Ctx, int cp1x, int cp1y, int cp2x, int cp2y, int x, int y)
{
    lSB.Append(Ctx + ".bezierCurveTo(" + cp1x + "," + cp1y + "," + cp2x + "," + cp2y + "," + x + "," + y + ");\n");
}
//moveTo(x, y)
public static void moveTo(StringBuilder lSB, string Ctx, int x, int y)
{
    lSB.Append(Ctx + ".moveTo(" + x + "," + y + ");\n");
}
//lineTo(x, y)
public static void lineTo(StringBuilder lSB, string Ctx, int x, int y)
{
    lSB.Append(Ctx + ".lineTo(" + x + "," + y + ");\n");
}
```




