bb---
title: Using WPF Font Geomotry
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: using_wpf_font_geometry.html
folder: usingfonts
---

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





