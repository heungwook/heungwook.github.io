---
title: Using Font in PostScript with GDI+ Graphics Path
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: usingfonts_postscript.html
folder: pdfpostscript
---


## Using Font in PostScript with GDI+ Graphics Path

- [PostScript Output](ABC.ps) (GhostView)

    ![PostScript Output](PS_Output.png)

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

- GDI+ Path Data contains below path commands
    >Path value == 0x00 : Move To<br/>
    Path value == 0x01 : Line To<br/>
    Path value == 0x03 : Point of Cubic Bezier Spline<br/>
    Path value == 0x80 : Close Path<br/>


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

### Drawing GDI+ Font Path in PostScript

- Set position and rotation
    >if (this.fRotation == 0F)<br/>
    {<br/>
        lPD.cPSBody.WriteUnicodeToASCII(String.Format("{0:0.###} {1:0.###} {2} ", <br/>
            lptfAPos.X + lptfAdjPos.X, lptfAPos.Y - lptfAdjPos.Y, OrionPostScript.csPStranslate));<br/>
    }<br/>
    else<br/>
    {<br/>
        lPD.cPSBody.PS_translate(lptfAPos.X, lptfAPos.Y);<br/>
        lPD.cPSBody.PS_rotate(-this.fRotation);<br/>
        lPD.cPSBody.PS_translate(lptfAdjPos.X, -lptfAdjPos.Y);<br/>
    }<br/>


- Font definitions of character 'A'(F_0) and '가'(F_1) in PostScript header
    >/F_0<br/>
    {<br/>
    3.257 -4.542 m<br/>
    3.221 -4.767 3.182 -4.925 3.14 -5.016 c<br/>
    2.002 -8.121 l<br/>
    4.536 -8.121 l<br/>
    3.389 -5.016 l<br/>
    3.346 -4.902 3.311 -4.744 3.281 -4.542 c<br/>
    3.257 -4.542 l<br/>
    h<br/>
    2.881 -3.698 m<br/>
    3.682 -3.698 l<br/>
    6.46 -10.885 l<br/>
    5.586 -10.885 l<br/>
    4.805 -8.844 l<br/>
    1.724 -8.844 l<br/>
    0.996 -10.885 l<br/>
    0.117 -10.885 l<br/>
    2.881 -3.698 l<br/>
    2.881 -3.698 l<br/>
    h<br/>
    } bind def<br/>

    >/F_1<br/>
    {<br/>
    1.24 -3.185 m<br/>
    5.64 -3.185 l<br/>
    5.568 -5.857 4.155 -7.975 1.401 -9.537 c<br/>
    0.84 -8.947 l<br/>
    1.826 -8.54 2.71 -7.88 3.491 -6.969 c<br/>
    4.272 -6.058 4.728 -5.013 4.858 -3.834 c<br/>
    1.24 -3.834 l<br/>
    1.24 -3.185 l<br/>
    h<br/>
    7.231 -2.345 m<br/>
    7.959 -2.345 l<br/>
    7.959 -6.227 l<br/>
    9.648 -6.227 l<br/>
    9.648 -6.876 l<br/>
    7.959 -6.876 l<br/>
    7.959 -11.764 l<br/>
    7.231 -11.764 l<br/>
    7.231 -2.345 l<br/>
    7.231 -2.345 l<br/>
    h<br/>
    } bind def<br/>

- Drawing character 'A', '가' in PostScript body at Page#1

    >%%Page: 1 1<br/>
    %%PageBoundingBox: 0 0 595 842<br/>
    %%BeginPageSetup<br/>
    /olddevice currentpagedevice def<br/>
    \<\<<br/>
    /PageSize [595 842]<br/>
    \>\> setpagedevice<br/>
    0 rotate 0 0 translate<br/>
    %%EndPageSetup<br/>
    q<br/>
    56.693 799.37 T 0 0 0 r<br/>
    q 0 0 T 1.9 1.9 s F_0 f Q       : CHAR 'A' = F_0<br/>
    Q<br/>
    q<br/>
    56.693 771.024 T 0 0 0 r<br/>
    q 0 0 T 1.9 1.9 s F_1 f Q       : CHAR '가' = F_1<br/>
    Q<br/>
    b<br/>

```C#
lPD.cPSBody.PS_gsave();

try
{

    if (this.fRotation == 0F)
    {
        lPD.cPSBody.WriteUnicodeToASCII(String.Format("{0:0.###} {1:0.###} {2} ", 
                    lptfAPos.X + lptfAdjPos.X, lptfAPos.Y - lptfAdjPos.Y, OrionPostScript.csPStranslate));
    }
    else
    {
        lPD.cPSBody.PS_translate(lptfAPos.X, lptfAPos.Y);
        lPD.cPSBody.PS_rotate(-this.fRotation);
        lPD.cPSBody.PS_translate(lptfAdjPos.X, -lptfAdjPos.Y);
    }


    float lfFontSize = this.GetVariableFontSize();
    float lfFontScaleW = lfFontSize / PostScriptFontData._DEFFONTSIZE;
    float lfFontScaleH = lfFontSize / PostScriptFontData._DEFFONTSIZE;
    //if (this.cfFontHWRatio != 100F)
    lfFontScaleW *= lfScaleWidth;// / 100F;

    lPD.cPSBody.PS_setrgbcolor(lForeColor);
    if (this.bFontStyleBold)
    {
        lPD.cPSBody.PS_setlinewidth(PostScriptFontData._DEFFONTSIZE * 0.030F);
    }

    foreach (OrionTextBoxInfo.LineChars lLine in llLNChars)
    {
        PointF lptfRPos = new PointF(OrionConfigInfo.UCNV.GetPointFromPixel(lLine.PTF.X, lPD.fWidthDPI),
                                        -OrionConfigInfo.UCNV.GetPointFromPixel(lLine.PTF.Y, lPD.fHeightDPI));

        if (this.bFontStyleItalic || this.cbFontOutline)
        {
            foreach (OrionTextBoxInfo.SizeFChar lSZFCH in lLine.LSZFCH)
            {
                SizeF lszfCharGap = new SizeF(OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.GapHorz, lPD.fWidthDPI),
                                            OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.GapVert, lPD.fHeightDPI));
                if (lSZFCH.CH != '\u200B')
                    lPD.cPSBody.PS_WriteChar(lPD, this, lptfRPos,
                                            lfFontScaleW, lfFontScaleH,
                                            lfFontSize, lSZFCH);

                lptfRPos.X += lszfCharGap.Width;
            }
        }
        else
        {
            StringBuilder lSB = new StringBuilder();
            lSB.AppendFormat(OrionPostScript.csPSgsave + " {0:0.##} {1:0.##} {2} ", lptfRPos.X, lptfRPos.Y, OrionPostScript.csPStranslate);
            if (lfFontScaleW != 1.0F || lfFontScaleH != 1.0F)
                lSB.AppendFormat("{0:0.##} {1:0.##} {2} ", lfFontScaleW, lfFontScaleH, OrionPostScript.csPSscale);

            bool lbIsFirstChar = true;
            float lfGapHPrevChar = 0F;
            float lfXPosTransform = 0F;
            //
            List<OrionConfigInfo.OrionColor> llColors = new List<OrionConfigInfo.OrionColor>();
            int liColorIX = 0;
            if (cColorList.cbUseColorList)
                llColors = cColorList.GetColors(lLine.LSZFCH.Count);
            //
            if (this.cbFontFlipHorz || this.cbFontFlipVert)
            {
                foreach (OrionTextBoxInfo.SizeFChar lSZFCH in lLine.LSZFCH)
                {
                    if (lSZFCH.CH == '\u200B')
                    {
                        lfGapHPrevChar = OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.GapHorz, lPD.fWidthDPI);
                        lptfRPos.X += lfGapHPrevChar;
                        continue;
                    }
                    if (cColorList.cbUseColorList)
                    {
                        NewColor lNCR = new NewColor(llColors[liColorIX]);
                        liColorIX++;
                        lPD.cPSBody.PS_setrgbcolor(lSB, lNCR.RGB);
                    }
                    //
                    lSB.AppendFormat("{0}\n", OrionPostScript.csPSgsave);
                    //
                    if (lSZFCH.cGlyph != null && lSZFCH.cGlyph.isSubstituted && lPD.fontSubstitutionPositionAdjust)
                    {
                        double adjustSubstitutedFontPositionY = lSZFCH.cGlyph.baselineDifference * 0.7;
                        lSB.AppendFormat("[1 0 0 1 {0:0.##} {1:0.##}] {2}\n",
                            0,
                            -adjustSubstitutedFontPositionY,
                            OrionPostScript.csPSconcat);
                    }
                    //
                    lfXPosTransform += lfGapHPrevChar;
                    lSB.AppendFormat("[1 0 0 1 {0:0.##} 0] {1}\n", lfXPosTransform / lfFontScaleW, OrionPostScript.csPSconcat);
                    if (this.cbFontFlipVert)
                    {
                        lSB.AppendFormat("[1 0 0 -1 {0:0.##} {1:0.##}] {2}\n",
                            0,
                            -OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.CharHeight, lPD.fHeightDPI) / lfFontScaleH,
                            OrionPostScript.csPSconcat);
                    }
                    if (this.cbFontFlipHorz)
                    {
                        lSB.AppendFormat("[-1 0 0 1 {0:0.##} {1:0.##}] {2}\n",
                            OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.CharWidth, lPD.fWidthDPI) / lfFontScaleW * 1.35F,
                            0,
                            OrionPostScript.csPSconcat);
                    }
                    lSB.Append(lSZFCH.FontCache + "\n");

                    if (this.bFontStyleBold)
                    {
                        lSB.Append(OrionPostScript.csPSgsave + " " + OrionPostScript.csPSfill + " " + OrionPostScript.csPSgrestore +
                            " [] 0 " + OrionPostScript.csPSsetdash + " " + OrionPostScript.csPSstroke + " ");
                    }
                    else
                    {
                        lSB.Append(OrionPostScript.csPSfill + " ");
                    }

                    lSB.AppendFormat("{0}\n", OrionPostScript.csPSgrestore);


                    lfGapHPrevChar = OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.GapHorz, lPD.fWidthDPI);
                    lptfRPos.X += lfGapHPrevChar;

                }
            }
            else
            {
                foreach (OrionTextBoxInfo.SizeFChar lSZFCH in lLine.LSZFCH)
                {
                    if (lSZFCH.CH == '\u200B')
                    {
                        lfGapHPrevChar = OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.GapHorz, lPD.fWidthDPI);
                        lptfRPos.X += lfGapHPrevChar;
                        continue;
                    }

                    if (lSZFCH.cGlyph != null && lSZFCH.cGlyph.isSubstituted && lPD.fontSubstitutionPositionAdjust)
                    {
                        double adjustSubstitutedFontPositionY = lSZFCH.cGlyph.baselineDifference * 0.7;
                        lSB.AppendFormat("[1 0 0 1 {0:0.##} {1:0.##}] {2}\n",
                            0,
                            -adjustSubstitutedFontPositionY,
                            OrionPostScript.csPSconcat);

                        if (lbIsFirstChar)
                        {
                            lSB.Append(lSZFCH.FontCache + " ");
                            lbIsFirstChar = false;
                        }
                        else
                        {
                            lSB.AppendFormat("{0:0.##} X {1} ", lfGapHPrevChar / lfFontScaleW, lSZFCH.FontCache);
                        }
                        lSB.AppendFormat("[1 0 0 1 {0:0.##} {1:0.##}] {2}\n",
                            0,
                            adjustSubstitutedFontPositionY,
                            OrionPostScript.csPSconcat);
                    }
                    else
                    {
                        if (lbIsFirstChar)
                        {
                            lSB.Append(lSZFCH.FontCache + " ");
                            lbIsFirstChar = false;
                        }
                        else
                        {
                            lSB.AppendFormat("{0:0.##} X {1} ", lfGapHPrevChar / lfFontScaleW, lSZFCH.FontCache);
                        }
                    }
                    //


                    lfGapHPrevChar = OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.GapHorz, lPD.fWidthDPI);
                    lptfRPos.X += lfGapHPrevChar;
                    if (cColorList.cbUseColorList)
                    {
                        NewColor lNCR = new NewColor(llColors[liColorIX]);
                        liColorIX++;
                        lPD.cPSBody.PS_setrgbcolor(lSB, lNCR.RGB);
                        if (this.bFontStyleBold)
                        {
                            lSB.Append(OrionPostScript.csPSgsave + " " + OrionPostScript.csPSfill + " " + OrionPostScript.csPSgrestore +
                                " [] 0 " + OrionPostScript.csPSsetdash + " " + OrionPostScript.csPSstroke + " ");
                        }
                        else
                        {
                            lSB.Append(OrionPostScript.csPSfill + " ");
                        }

                    }
                }
                if (!cColorList.cbUseColorList)
                {
                    if (this.bFontStyleBold)
                    {
                        lSB.Append(OrionPostScript.csPSgsave + " " + OrionPostScript.csPSfill + " " + OrionPostScript.csPSgrestore +
                            " [] 0 " + OrionPostScript.csPSsetdash + " " + OrionPostScript.csPSstroke + " ");
                    }
                    else
                    {
                        lSB.Append(OrionPostScript.csPSfill + " ");
                    }
                }
            }
            lSB.AppendLine(OrionPostScript.csPSgrestore);
            lPD.cPSBody.WriteUnicodeToASCII(lSB.ToString());
        }

        if (this.bFontStyleStrikeout || this.bFontStyleUnderline)
            this.DrawSThru_ULine_PS(lPD, lPD.cPSBody, lLine);

    }
}
catch (Exception exe)
{
    ORIONDEBUG.LOG(LogInfo.EnumLogLevel.ERROR, "Error in OD_ColumnData::DrawData_PS()", exe);
}
finally
{
    lPD.cPSBody.PS_grestore();
    lStrFmt.Dispose();
}
```







