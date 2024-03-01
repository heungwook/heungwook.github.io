---
title: .NET DataVisualization for Creating PDF, PostScript Charts 
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: dotnet_datavisualization_01.html
folder: dotnetdev
---

## Overview

As you may know, .NET provides nice [Data Visualization features](https://github.com/dotnet/winforms-datavisualization/tree/main).

![Chart Sample Images](chart_samples.png)

But, .NET Data Visualization only supports RGB raster image charts(.png, .jpg, etc) and creating high-resolution images takes much time. Also, does not supoort PDF and PostScript outputs for volume printing.

- Why PDF and PostScript outputs
    - The size of PDF/PostScript charts in high-resolution output is normally 10kb ~ 50kb per chart, but the size of raster image charts(.png, .jpg, .tif formats) may have 1mb ~ 5mb size.
    - Much faster creation, comparing to bitmap charts.
    - Let's assume we create a chart with 2x2-inch (600 dpi resultion). While drawing raster chart image needs to handle 3 or 4 bytes for 1,200x1,200=1,440,000 pixels, this will cause slow and huge outputs. But, for vector outputs like PDF/PostScript, drawing lines/circles/stroke/fill commands just need few bytes and all these commands will be executed to create rasterization images by Adobe Acrobat PDF Readers or Printers.
    - PostScript and PDF support CMYK, printing color images requires CMYK color-space for precise colors.

- Source code for [.NET 8 Data Visualization](https://github.com/heungwook/NET8_DataVisualization)


## Implementation of PDF & PostScript Charts

.NET Data Visualization library uses GDI+ for drawing chart's components like X-Y axis lines, ellipses and label texts. So, I've implemented PDF and PostScript warappers to replace all GDI+ drawing methods that are used in chart drawing.


### Entry Points to Create PDF, PostScript Charts

- SavePDF(), SavePS() methods in ChartWinControls.cs are the entry points for PDF, PostScript charts creation. And, these methods initialize the rendering information of PDF and PostScript.

- SavePDF() Method

    ```C#
    // -ChartPDF

    public void SavePDF(string lsPdfFile, System.Windows.Point chartLocation, PdfGraphicsInfo.EnumColorSpace lColorSpace, OrionX2.OrionFont.FontManagerConfig fontMgr, bool lbUseKValueCMYKEqual)
    {
        using (FileStream lfsPdf = new FileStream(lsPdfFile, FileMode.Create, FileAccess.ReadWrite, FileShare.ReadWrite))
        {
            SavePDF(lfsPdf, chartLocation, lColorSpace, fontMgr, lbUseKValueCMYKEqual);
            lfsPdf.Close();
        }
        
    }

    public void SavePDF(Stream lStrm, System.Windows.Point chartLocation, PdfGraphicsInfo.EnumColorSpace colorSpace, OrionX2.OrionFont.FontManagerConfig fontMgr, bool lbUseKValueCMYKEqual)
    {
        try
        {
            iTextSharp.text.FontFactory.RegisterDirectory(@"C:\Windows\Fonts");
            iTextSharp.text.Rectangle pdfPageSize = new iTextSharp.text.Rectangle(0, 0, 
                (float)OrionX2.ConfigInfo.UCNV.GetPointFromPixel(this.Width, renderingDpiX),
                (float)OrionX2.ConfigInfo.UCNV.GetPointFromPixel(this.Height, renderingDpiY), 0);
            using (iTextSharp.text.Document pdfDoc = new iTextSharp.text.Document(pdfPageSize))
            {

                iTextSharp.text.pdf.PdfWriter pdfWrt = iTextSharp.text.pdf.PdfWriter.GetInstance(pdfDoc, lStrm);
                pdfDoc.Open();
                pdfDoc.NewPage();

                SavePDF(chartLocation, pdfDoc, pdfWrt, colorSpace, fontMgr, lbUseKValueCMYKEqual);
                pdfDoc.Close();
            }
        }
        finally
        {

        }
    }

    public void SavePDF(System.Windows.Point chartLocation, iTextSharp.text.Document pdfDoc, iTextSharp.text.pdf.PdfWriter pdfWrt, PdfGraphicsInfo.EnumColorSpace colorSpace, OrionX2.OrionFont.FontManagerConfig fontMgr, bool lbUseKValueCMYKEqual)
    {
        PdfGraphicsInfo lPdfInfo = new PdfGraphicsInfo(fontMgr,  new System.Windows.Rect(chartLocation.X, chartLocation.Y, this.Width, this.Height), 
            RenderingDpiX, RenderingDpiY, lbUseKValueCMYKEqual);
        PSGraphicsInfo lPSInfo = new PSGraphicsInfo(null, null, fontMgr,
            new System.Windows.Rect(chartLocation.X, chartLocation.Y, this.Width, this.Height), RenderingDpiX, RenderingDpiY);
        try
        {
            lPdfInfo.cPdfDoc = pdfDoc;
            lPdfInfo.cPdfWrt = pdfWrt;
            lPdfInfo.ceColorSpace = colorSpace;
            lPdfInfo.DpiX = RenderingDpiX;
            lPdfInfo.DpiY = RenderingDpiY;

            SavePDF(lPdfInfo);

        }
        finally
        {

        }
    }

    public void SavePDF(PdfGraphicsInfo lPdfInfo)
    {
        this.chartPicture.isSavingAsImage = true;
        this.chartPicture.GetPDF(lPdfInfo);
        this.chartPicture.isSavingAsImage = false;
    }
    // ChartPDF-
    ```


- SavePS() Method

    ```C#
    // -ChartPS

    public void SavePS(System.Windows.Point chartLocation, PdfGraphicsInfo.EnumColorSpace colorSpace, OrionX2.OrionFont.FontManagerConfig fontMgr, Dictionary<string, PSFontData> ldctPSFonts, Stream lStrmHeader, Stream lStrmBody)
    {
        PSGraphicsInfo lPSInfo = new PSGraphicsInfo(lStrmHeader, lStrmBody, fontMgr,
            new System.Windows.Rect(chartLocation.X, chartLocation.Y, this.Width, this.Height), RenderingDpiX, RenderingDpiY);
        PdfGraphicsInfo lPdfInfo = new PdfGraphicsInfo(fontMgr, new System.Windows.Rect(chartLocation.X, chartLocation.Y, this.Width, this.Height),
            RenderingDpiX, RenderingDpiY, true);//lbUseKValueCMYKEqual);
        lPSInfo.cdctFontData = ldctPSFonts;
        try
        {
            lPSInfo.ceColorSpace = colorSpace;
            lPSInfo.DpiX = RenderingDpiX;
            lPSInfo.DpiY = RenderingDpiY;

            SavePS(lPSInfo);
        }
        finally
        {

        }
    }

    public void SavePS(PSGraphicsInfo lPSInfo)
    {
        this.chartPicture.isSavingAsImage = true;
        this.chartPicture.GetPS(lPSInfo);
        this.chartPicture.isSavingAsImage = false;
    }
    // ChartPS-
    ```


### GDI+, PDF, PostScript Rendering Wrapper Code




- - -

<details>

<summary> ▶ RenderGDI.cs Source Code (Click To Expane)</summary> 

<pre>

<code>

//-------------------------------------------------------------
// <copyright company=�Microsoft Corporation?
//   Copyright ?Microsoft Corporation. All Rights Reserved.
// </copyright>
//-------------------------------------------------------------
// @owner=alexgor, deliant
//=================================================================
//  File:		RenderGDI.cs
//
//  Namespace:	DataVisualization.Charting
//
//	Classes:	RenderGDI
//
//  Purpose:	RenderGDI class is chart GDI+ rendering engine. It 
//              implements IChartRenderingEngine interface by mapping 
//              its methods to the drawing methods of GDI+. This 
//              rendering engine do not support animation.
//
//	Reviwed:	AG - Jul 15, 2003
//              AG - Microsoft 14, 2007
//
//===================================================================

#region Used namespaces

using System;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Drawing.Text;
using System.Drawing.Imaging;
using System.ComponentModel;
using System.Collections;
using System.Diagnostics.CodeAnalysis;

#if Microsoft_CONTROL

using Orion.DataVisualization.Charting.Utilities;
using Orion.DataVisualization.Charting.Borders3D;
#else
    //using System.Web.UI.DataVisualization.Charting.Utilities;
    //using System.Web.UI.DataVisualization.Charting.Borders3D;
#endif

#endregion

#if Microsoft_CONTROL
    namespace Orion.DataVisualization.Charting
#else
namespace System.Web.UI.DataVisualization.Charting

#endif
{
    /// <summary>
    /// GdiGraphics class is chart GDI+ rendering engine.
    /// </summary>
    [SuppressMessage("Microsoft.Naming", "CA1704:IdentifiersShouldBeSpelledCorrectly", MessageId = "Gdi")]
    internal class RenderGDI : IChartRenderingEngine
    {

        #region Constructors

        /// <summary>
        /// Default constructor
        /// </summary>
        public RenderGDI()
        {
        }

        #endregion // Constructor

        #region Drawing Methods

        /// <summary>
        /// Draws a line connecting two PointF structures.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the line.</param>
        /// <param name="pt1">PointF structure that represents the first point to connect.</param>
        /// <param name="pt2">PointF structure that represents the second point to connect.</param>
        public void DrawLine(
            Pen pen,
            PointF pt1,
            PointF pt2
            )
        {
            _graphics.DrawLine(pen.ToGdiPen(), pt1, pt2);
        }

        /// <summary>
        /// Draws a line connecting the two points specified by coordinate pairs.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the line.</param>
        /// <param name="x1">x-coordinate of the first point.</param>
        /// <param name="y1">y-coordinate of the first point.</param>
        /// <param name="x2">x-coordinate of the second point.</param>
        /// <param name="y2">y-coordinate of the second point.</param>
        public void DrawLine(
            Pen pen,
            float x1,
            float y1,
            float x2,
            float y2
            )
        {
            _graphics.DrawLine(pen.ToGdiPen(), x1, y1, x2, y2 );
        }

        /// <summary>
        /// Draws the specified portion of the specified Image object at the specified location and with the specified size.
        /// </summary>
        /// <param name="image">Image object to draw.</param>
        /// <param name="destRect">Rectangle structure that specifies the location and size of the drawn image. The image is scaled to fit the rectangle.</param>
        /// <param name="srcX">x-coordinate of the upper-left corner of the portion of the source image to draw.</param>
        /// <param name="srcY">y-coordinate of the upper-left corner of the portion of the source image to draw.</param>
        /// <param name="srcWidth">Width of the portion of the source image to draw.</param>
        /// <param name="srcHeight">Height of the portion of the source image to draw.</param>
        /// <param name="srcUnit">Member of the GraphicsUnit enumeration that specifies the units of measure used to determine the source rectangle.</param>
        /// <param name="imageAttr">ImageAttributes object that specifies recoloring and gamma information for the image object.</param>
        public void DrawImage(
            System.Drawing.Image image,
            Rectangle destRect,
            int srcX,
            int srcY,
            int srcWidth,
            int srcHeight,
            GraphicsUnit srcUnit,
            ImageAttributes imageAttr
            )
        {
            _graphics.DrawImage( 
                    image,
                    destRect,
                    srcX,
                    srcY,
                    srcWidth,
                    srcHeight,
                    srcUnit,
                    imageAttr
                );
        }

        /// <summary>
        /// Draws an ellipse defined by a bounding rectangle specified by 
        /// a pair of coordinates: a height, and a width.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the ellipse.</param>
        /// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse.</param>
        /// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse.</param>
        /// <param name="width">Width of the bounding rectangle that defines the ellipse.</param>
        /// <param name="height">Height of the bounding rectangle that defines the ellipse.</param>
        public void DrawEllipse(
            Pen pen,
            float x,
            float y,
            float width,
            float height
            )
        {
            _graphics.DrawEllipse(pen.ToGdiPen(), x, y, width, height );
        }

        /// <summary>
        /// Draws a cardinal spline through a specified array of PointF structures 
        /// using a specified tension. The drawing begins offset from 
        /// the beginning of the array.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and height of the curve.</param>
        /// <param name="points">Array of PointF structures that define the spline.</param>
        /// <param name="offset">Offset from the first element in the array of the points parameter to the starting point in the curve.</param>
        /// <param name="numberOfSegments">Number of segments after the starting point to include in the curve.</param>
        /// <param name="tension">Value greater than or equal to 0.0F that specifies the tension of the curve.</param>
        public void DrawCurve(
            Pen pen,
            PointF[] points,
            int offset,
            int numberOfSegments,
            float tension
            )
        {
            _graphics.DrawCurve(pen.ToGdiPen(), points, offset,  numberOfSegments, tension );
        }

        /// <summary>
        /// Draws a rectangle specified by a coordinate pair: a width, and a height.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the rectangle.</param>
        /// <param name="x">x-coordinate of the upper-left corner of the rectangle to draw.</param>
        /// <param name="y">y-coordinate of the upper-left corner of the rectangle to draw.</param>
        /// <param name="width">Width of the rectangle to draw.</param>
        /// <param name="height">Height of the rectangle to draw.</param>
        public void DrawRectangle(
            Pen pen,
            int x,
            int y,
            int width,
            int height
            )
        {
            _graphics.DrawRectangle(pen.ToGdiPen(), x, y, width, height );
        }

        /// <summary>
        /// Draws a polygon defined by an array of PointF structures.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the polygon.</param>
        /// <param name="points">Array of PointF structures that represent the vertices of the polygon.</param>
        public void DrawPolygon(
            Pen pen,
            PointF[] points
            )
        {
            _graphics.DrawPolygon(pen.ToGdiPen(), points );
        }

        /// <summary>
        /// Draws the specified text string in the specified rectangle with the specified Brush and Font objects using the formatting properties of the specified StringFormat object.
        /// </summary>
        /// <param name="s">String to draw.</param>
        /// <param name="font">Font object that defines the text format of the string.</param>
        /// <param name="brush">Brush object that determines the color and texture of the drawn text.</param>
        /// <param name="layoutRectangle">RectangleF structure that specifies the location of the drawn text.</param>
        /// <param name="format">StringFormat object that specifies formatting properties, such as line spacing and alignment, that are applied to the drawn text.</param>
        public void DrawString(
            string s,
            Font font,
            Brush brush,
            RectangleF layoutRectangle,
            StringFormat format
            )
        {
            _graphics.DrawString( s, font, brush.ToGdiBrush(), layoutRectangle, format );
        }

        /// <summary>
        /// Draws the specified text string at the specified location with the specified Brush and Font objects using the formatting properties of the specified StringFormat object.
        /// </summary>
        /// <param name="s">String to draw.</param>
        /// <param name="font">Font object that defines the text format of the string.</param>
        /// <param name="brush">Brush object that determines the color and texture of the drawn text.</param>
        /// <param name="point">PointF structure that specifies the upper-left corner of the drawn text.</param>
        /// <param name="format">StringFormat object that specifies formatting properties, such as line spacing and alignment, that are applied to the drawn text.</param>
        public void DrawString(
            string s,
            Font font,
            Brush brush,
            PointF point,
            StringFormat format
            )
        {
            _graphics.DrawString( s, font, brush.ToGdiBrush(), point, format );
        }

        /// <summary>
        /// Draws the specified portion of the specified Image object at the specified location and with the specified size.
        /// </summary>
        /// <param name="image">Image object to draw.</param>
        /// <param name="destRect">Rectangle structure that specifies the location and size of the drawn image. The image is scaled to fit the rectangle.</param>
        /// <param name="srcX">x-coordinate of the upper-left corner of the portion of the source image to draw.</param>
        /// <param name="srcY">y-coordinate of the upper-left corner of the portion of the source image to draw.</param>
        /// <param name="srcWidth">Width of the portion of the source image to draw.</param>
        /// <param name="srcHeight">Height of the portion of the source image to draw.</param>
        /// <param name="srcUnit">Member of the GraphicsUnit enumeration that specifies the units of measure used to determine the source rectangle.</param>
        /// <param name="imageAttrs">ImageAttributes object that specifies recoloring and gamma information for the image object.</param>
        public void DrawImage(
            System.Drawing.Image image,
            Rectangle destRect,
            float srcX,
            float srcY,
            float srcWidth,
            float srcHeight,
            GraphicsUnit srcUnit,
            ImageAttributes imageAttrs
            )
        {
            _graphics.DrawImage( image, destRect, srcX, srcY, srcWidth, srcHeight, srcUnit, imageAttrs );
        }

        /// <summary>
        /// Draws a rectangle specified by a coordinate pair: a width, and a height.
        /// </summary>
        /// <param name="pen">A Pen object that determines the color, width, and style of the rectangle.</param>
        /// <param name="x">The x-coordinate of the upper-left corner of the rectangle to draw.</param>
        /// <param name="y">The y-coordinate of the upper-left corner of the rectangle to draw.</param>
        /// <param name="width">The width of the rectangle to draw.</param>
        /// <param name="height">The height of the rectangle to draw.</param>
        public void DrawRectangle(
            Pen pen,
            float x,
            float y,
            float width,
            float height
            )
        {
            _graphics.DrawRectangle(pen.ToGdiPen(), x, y, width, height );
        }

        /// <summary>
        /// Draws a GraphicsPath object.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the path.</param>
        /// <param name="path">GraphicsPath object to draw.</param>
        public void DrawPath(
            Pen pen,
            GraphicsPath path
            )
        {
            _graphics.DrawPath(pen.ToGdiPen(), path );
        }

        /// <summary>
        /// Draws a pie shape defined by an ellipse specified by a coordinate pair: a width, a height and two radial lines.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the pie shape.</param>
        /// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
        /// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
        /// <param name="width">Width of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
        /// <param name="height">Height of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
        /// <param name="startAngle">Angle measured in degrees clockwise from the x-axis to the first side of the pie shape.</param>
        /// <param name="sweepAngle">Angle measured in degrees clockwise from the startAngle parameter to the second side of the pie shape.</param>
        public void DrawPie(
            Pen pen,
            float x,
            float y,
            float width,
            float height,
            float startAngle,
            float sweepAngle
            )
        {
            _graphics.DrawPie(pen.ToGdiPen(), x, y, width, height, startAngle, sweepAngle );
        }

        /// <summary>
        /// Draws an arc representing a portion of an ellipse specified by a pair of coordinates: a width, and a height.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the arc.</param>
        /// <param name="x">x-coordinate of the upper-left corner of the rectangle that defines the ellipse.</param>
        /// <param name="y">y-coordinate of the upper-left corner of the rectangle that defines the ellipse.</param>
        /// <param name="width">Width of the rectangle that defines the ellipse.</param>
        /// <param name="height">Height of the rectangle that defines the ellipse.</param>
        /// <param name="startAngle">Angle in degrees measured clockwise from the x-axis to the starting point of the arc.</param>
        /// <param name="sweepAngle">Angle in degrees measured clockwise from the startAngle parameter to ending point of the arc.</param>
        public void DrawArc(
            Pen pen,
            float x,
            float y,
            float width,
            float height,
            float startAngle,
            float sweepAngle
            )
        {
            _graphics.DrawArc(pen.ToGdiPen(), x, y, width, height, startAngle, sweepAngle );
        }

        /// <summary>
        /// Draws the specified Image object at the specified location and with the specified size.
        /// </summary>
        /// <param name="image">Image object to draw.</param>
        /// <param name="rect">RectangleF structure that specifies the location and size of the drawn image.</param>
        public void DrawImage(
            System.Drawing.Image image,
            RectangleF rect
            )
        {
            _graphics.DrawImage( image, rect );
        }

        /// <summary>
        /// Draws an ellipse defined by a bounding RectangleF.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the ellipse.</param>
        /// <param name="rect">RectangleF structure that defines the boundaries of the ellipse.</param>
        public void DrawEllipse(
            Pen pen,
            RectangleF rect
            )
        {
            _graphics.DrawEllipse(pen.ToGdiPen(), rect );
        }

        /// <summary>
        /// Draws a series of line segments that connect an array of PointF structures.
        /// </summary>
        /// <param name="pen">Pen object that determines the color, width, and style of the line segments.</param>
        /// <param name="points">Array of PointF structures that represent the points to connect.</param>
        public void DrawLines(
            Pen pen,
            PointF[] points
            )
        {
            _graphics.DrawLines(pen.ToGdiPen(), points );
        }

        #endregion // Drawing Methods

        #region Filling Methods

        /// <summary>
        /// Fills the interior of an ellipse defined by a bounding rectangle 
        /// specified by a RectangleF structure.
        /// </summary>
        /// <param name="brush">Brush object that determines the characteristics of the fill.</param>
        /// <param name="rect">RectangleF structure that represents the bounding rectangle that defines the ellipse.</param>
        public void FillEllipse(
            Brush brush,
            RectangleF rect
            )
        {
            _graphics.FillEllipse( brush.ToGdiBrush(), rect );
        }

        /// <summary>
        /// Fills the interior of a GraphicsPath object.
        /// </summary>
        /// <param name="brush">Brush object that determines the characteristics of the fill.</param>
        /// <param name="path">GraphicsPath object that represents the path to fill.</param>
        public void FillPath(
            Brush brush,
            GraphicsPath path
            )
        {
            _graphics.FillPath( brush.ToGdiBrush(), path );
        }

        /// <summary>
        /// Fills the interior of a Region object.
        /// </summary>
        /// <param name="brush">Brush object that determines the characteristics of the fill.</param>
        /// <param name="region">Region object that represents the area to fill.</param>
        public void FillRegion(
            Brush brush,
            Region region
            )
        {
            _graphics.FillRegion( brush.ToGdiBrush(), region );
        }

        /// <summary>
        /// Fills the interior of a rectangle specified by a RectangleF structure.
        /// </summary>
        /// <param name="brush">Brush object that determines the characteristics of the fill.</param>
        /// <param name="rect">RectangleF structure that represents the rectangle to fill.</param>
        public void FillRectangle(
            Brush brush,
            RectangleF rect
            )
        {
            _graphics.FillRectangle( brush.ToGdiBrush(), rect );
        }

        /// <summary>
        /// Fills the interior of a rectangle specified by a pair of coordinates, a width, and a height.
        /// </summary>
        /// <param name="brush">Brush object that determines the characteristics of the fill.</param>
        /// <param name="x">x-coordinate of the upper-left corner of the rectangle to fill.</param>
        /// <param name="y">y-coordinate of the upper-left corner of the rectangle to fill.</param>
        /// <param name="width">Width of the rectangle to fill.</param>
        /// <param name="height">Height of the rectangle to fill.</param>
        public void FillRectangle(
            Brush brush,
            float x,
            float y,
            float width,
            float height
            )
        {
            _graphics.FillRectangle( brush.ToGdiBrush(), x, y, width, height );
        }

        /// <summary>
        /// Fills the interior of a polygon defined by an array of points specified by PointF structures .
        /// </summary>
        /// <param name="brush">Brush object that determines the characteristics of the fill.</param>
        /// <param name="points">Array of PointF structures that represent the vertices of the polygon to fill.</param>
        public void FillPolygon(
            Brush brush,
            PointF[] points
            )
        {
            _graphics.FillPolygon( brush.ToGdiBrush(), points );
        }

        /// <summary>
        /// Fills the interior of a pie section defined by an ellipse 
        /// specified by a pair of coordinates, a width, and a height 
        /// and two radial lines.
        /// </summary>
        /// <param name="brush">Brush object that determines the characteristics of the fill.</param>
        /// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
        /// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
        /// <param name="width">Width of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
        /// <param name="height">Height of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
        /// <param name="startAngle">Angle in degrees measured clockwise from the x-axis to the first side of the pie section.</param>
        /// <param name="sweepAngle">Angle in degrees measured clockwise from the startAngle parameter to the second side of the pie section.</param>
        public void FillPie(
            Brush brush,
            float x,
            float y,
            float width,
            float height,
            float startAngle,
            float sweepAngle
            )
        {
            _graphics.FillPie( brush.ToGdiBrush(), x, y, width, height, startAngle, sweepAngle );
        }

        #endregion // Filling Methods

        #region Other Methods

        /// <summary>
        /// Measures the specified string when drawn with the specified 
        /// Font object and formatted with the specified StringFormat object.
        /// </summary>
        /// <param name="text">String to measure.</param>
        /// <param name="font">Font object defines the text format of the string.</param>
        /// <param name="layoutArea">SizeF structure that specifies the maximum layout area for the text.</param>
        /// <param name="stringFormat">StringFormat object that represents formatting information, such as line spacing, for the string.</param>
        /// <returns>This method returns a SizeF structure that represents the size, in pixels, of the string specified in the text parameter as drawn with the font parameter and the stringFormat parameter.</returns>
        public SizeF MeasureString(
            string text,
            Font font,
            SizeF layoutArea,
            StringFormat stringFormat
            )
        {
            return _graphics.MeasureString( text, font, layoutArea, stringFormat );
        }

        /// <summary>
        /// Measures the specified string when drawn with the specified 
        /// Font object and formatted with the specified StringFormat object.
        /// </summary>
        /// <param name="text">String to measure.</param>
        /// <param name="font">Font object defines the text format of the string.</param>
        /// <returns>This method returns a SizeF structure that represents the size, in pixels, of the string specified in the text parameter as drawn with the font parameter and the stringFormat parameter.</returns>
        public SizeF MeasureString(
            string text,
            Font font
            )
        {
            return _graphics.MeasureString( text, font );
        }

        /// <summary>
        /// Saves the current state of this Graphics object and identifies the saved state with a GraphicsState object.
        /// </summary>
        /// <returns>This method returns a GraphicsState object that represents the saved state of this Graphics object.</returns>
        public GraphicsState Save()
        {
            return _graphics.Save();
        }

        /// <summary>
        /// Restores the state of this Graphics object to the state represented by a GraphicsState object.
        /// </summary>
        /// <param name="gstate">GraphicsState object that represents the state to which to restore this Graphics object.</param>
        public void Restore(
            GraphicsState gstate
            )
        {
            _graphics.Restore( gstate );
        }

        /// <summary>
        /// Resets the clip region of this Graphics object to an infinite region.
        /// </summary>
        public void ResetClip()
        {
            _graphics.ResetClip();
        }

        /// <summary>
        /// Sets the clipping region of this Graphics object to the rectangle specified by a RectangleF structure.
        /// </summary>
        /// <param name="rect">RectangleF structure that represents the new clip region.</param>
        public void SetClip(
            RectangleF rect
            )
        {
            _graphics.SetClip( rect );
        }

        /// <summary>
        /// Sets the clipping region of this Graphics object to the result of the 
        /// specified operation combining the current clip region and the 
        /// specified GraphicsPath object.
        /// </summary>
        /// <param name="path">GraphicsPath object to combine.</param>
        /// <param name="combineMode">Member of the CombineMode enumeration that specifies the combining operation to use.</param>
        public void SetClip(
            GraphicsPath path,
            CombineMode combineMode
            )
        {
            _graphics.SetClip( path, combineMode );
        }

        /// <summary>
        /// Prepends the specified translation to the transformation matrix of this Graphics object.
        /// </summary>
        /// <param name="dx">x component of the translation.</param>
        /// <param name="dy">y component of the translation.</param>
        public void TranslateTransform(
            float dx,
            float dy
            )
        {
            _graphics.TranslateTransform( dx, dy );
        }

        /// <summary>
        /// This method starts Selection mode
        /// </summary>
        /// <param name="hRef">The location of the referenced object, expressed as a URI reference.</param>
        /// <param name="title">Title which could be used for tooltips.</param>
        public void BeginSelection( string hRef, string title )
        {
            // Not supported for GDI+
        }

        /// <summary>
        /// This method stops Selection mode
        /// </summary>
        public void EndSelection( )
        {
            // Not supported for GDI+
        }


        #endregion // Other Methods

        #region Properties

        /// <summary>
        /// Gets or sets the world transformation for this Graphics object.
        /// </summary>
        public Matrix Transform
        {
            get
            {
                return _graphics.Transform;
            }
            set
            {
                _graphics.Transform = value;
            }
        }

        /// <summary>
        /// Gets or sets the rendering quality for this Graphics object.
        /// </summary>
        public SmoothingMode SmoothingMode 
        {
            get
            {
                return _graphics.SmoothingMode;
            }
            set
            {
                _graphics.SmoothingMode = value;
            }
        }

        /// <summary>
        /// Gets or sets the rendering mode for text associated with this Graphics object.
        /// </summary>
        public TextRenderingHint TextRenderingHint 
        {
            get
            {
                return _graphics.TextRenderingHint;
            }
            set
            {
                _graphics.TextRenderingHint = value;
            }
        }

        public CompositingQuality CompositingQuality
        {
            get
            {
                return _graphics.CompositingQuality;
            }
            set
            {
                _graphics.CompositingQuality = value;
            }
        }
        public InterpolationMode InterpolationMode
        {
            get
            {
                return _graphics.InterpolationMode;
            }
            set
            {
                _graphics.InterpolationMode = value;
            }
        }
        /// <summary>
        /// Gets or sets a Region object that limits the drawing region of this Graphics object.
        /// </summary>
        public Region Clip 
        {
            get
            {
                return _graphics.Clip;
            }
            set
            {
                _graphics.Clip = value;
            }
        }

        /// <summary>
        /// Gets a value indicating whether the clipping region of this Graphics object is empty.
        /// </summary>
        public bool IsClipEmpty 
        {
            get
            {
                return _graphics.IsClipEmpty;
            }
        }

        /// <summary>
        /// Reference to the Graphics object
        /// </summary>
        public IRenderingGraphics Graphics
        {
            get
            {
                return _renderingGraphics;
            }
            set
            {
                _renderingGraphics = value;
            }
        }

        #endregion // Properties

        #region Fields


        /// <summary>
        /// Graphics object
        /// </summary>

        IRenderingGraphics _renderingGraphics = null; 

        Graphics _graphics { get => (_renderingGraphics as RenderingGraphicsGDI).GraphicsGdi; }

        #endregion // Fields
    }
}

</code>

</pre>

</details>


- - -

<details>

<summary> ▶ RenderPDF.cs Source Code (Click To Expane)</summary> 

<pre>

<code>

//=================================================================
//  File:		RenderPDF.cs
//
//  Namespace:	DataVisualization.Charting
//
//	Classes:	RenderPDF
//
//  Purpose:	RenderPDF class is chart PDF rendering engine. It 
//              implements IChartRenderingEngine interface by mapping 
//              its methods to the drawing methods of PDF. 
//
//	Reviwed:	
//===================================================================

#region Used namespaces

using System;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Drawing.Text;
using System.Drawing.Imaging;
using System.ComponentModel;
using System.Collections;
using System.Diagnostics.CodeAnalysis;

#if Microsoft_CONTROL

using Orion.DataVisualization.Charting.Utilities;
using Orion.DataVisualization.Charting.Borders3D;
#else
	//using System.Web.UI.DataVisualization.Charting.Utilities;
	//using System.Web.UI.DataVisualization.Charting.Borders3D;
#endif

#endregion

#if Microsoft_CONTROL
namespace Orion.DataVisualization.Charting
#else
namespace System.Web.UI.DataVisualization.Charting

#endif
{
	/// <summary>
	/// GdiGraphics class is chart GDI+ rendering engine.
	/// </summary>
	[SuppressMessage("Microsoft.Naming", "CA1704:IdentifiersShouldBeSpelledCorrectly", MessageId = "Gdi")]
    internal class RenderPDF : IChartRenderingEngine
	{
		public PdfGraphicsInfo cGraphics;
			
		#region Constructors

		/// <summary>
		/// Default constructor
		/// </summary>
		public RenderPDF(PdfGraphicsInfo lPdfInfo)
		{
			cGraphics = lPdfInfo;
		}

		#endregion // Constructor

		#region Drawing Methods

		/// <summary>
		/// Draws a line connecting two PointF structures.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the line.</param>
		/// <param name="pt1">PointF structure that represents the first point to connect.</param>
		/// <param name="pt2">PointF structure that represents the second point to connect.</param>
		public void DrawLine(
			Pen pen,
			PointF pt1,
			PointF pt2
			)
		{
			_graphicsGdi.DrawLine(pen.ToGdiPen(), pt1, pt2);
			_graphicsPdf.DrawLine(pen, pt1, pt2);

		}

		/// <summary>
		/// Draws a line connecting the two points specified by coordinate pairs.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the line.</param>
		/// <param name="x1">x-coordinate of the first point.</param>
		/// <param name="y1">y-coordinate of the first point.</param>
		/// <param name="x2">x-coordinate of the second point.</param>
		/// <param name="y2">y-coordinate of the second point.</param>
		public void DrawLine(
			Pen pen,
			float x1,
			float y1,
			float x2,
			float y2
			)
		{
			_graphicsGdi.DrawLine( pen.ToGdiPen(), x1, y1, x2, y2 );
			_graphicsPdf.DrawLine(pen, new PointF(x1, y1), new PointF(x2, y2));
		}

		/// <summary>
		/// Draws the specified portion of the specified Image object at the specified location and with the specified size.
		/// </summary>
		/// <param name="image">Image object to draw.</param>
		/// <param name="destRect">Rectangle structure that specifies the location and size of the drawn image. The image is scaled to fit the rectangle.</param>
		/// <param name="srcX">x-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcY">y-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcWidth">Width of the portion of the source image to draw.</param>
		/// <param name="srcHeight">Height of the portion of the source image to draw.</param>
		/// <param name="srcUnit">Member of the GraphicsUnit enumeration that specifies the units of measure used to determine the source rectangle.</param>
		/// <param name="imageAttr">ImageAttributes object that specifies recoloring and gamma information for the image object.</param>
		public void DrawImage(
			System.Drawing.Image image,
			Rectangle destRect,
			int srcX,
			int srcY,
			int srcWidth,
			int srcHeight,
			GraphicsUnit srcUnit,
			ImageAttributes imageAttr
			)
		{
			_graphicsGdi.DrawImage( 
					image,
					destRect,
					srcX,
					srcY,
					srcWidth,
					srcHeight,
					srcUnit,
					imageAttr
				);
		}

		/// <summary>
		/// Draws the specified portion of the specified Image object at the specified location and with the specified size.
		/// </summary>
		/// <param name="image">Image object to draw.</param>
		/// <param name="destRect">Rectangle structure that specifies the location and size of the drawn image. The image is scaled to fit the rectangle.</param>
		/// <param name="srcX">x-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcY">y-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcWidth">Width of the portion of the source image to draw.</param>
		/// <param name="srcHeight">Height of the portion of the source image to draw.</param>
		/// <param name="srcUnit">Member of the GraphicsUnit enumeration that specifies the units of measure used to determine the source rectangle.</param>
		/// <param name="imageAttrs">ImageAttributes object that specifies recoloring and gamma information for the image object.</param>
		public void DrawImage(
			System.Drawing.Image image,
			Rectangle destRect,
			float srcX,
			float srcY,
			float srcWidth,
			float srcHeight,
			GraphicsUnit srcUnit,
			ImageAttributes imageAttrs
			)
		{
			_graphicsGdi.DrawImage(image, destRect, srcX, srcY, srcWidth, srcHeight, srcUnit, imageAttrs);
		}


		/// <summary>
		/// Draws the specified Image object at the specified location and with the specified size.
		/// </summary>
		/// <param name="image">Image object to draw.</param>
		/// <param name="rect">RectangleF structure that specifies the location and size of the drawn image.</param>
		public void DrawImage(
			System.Drawing.Image image,
			RectangleF rect
			)
		{
			_graphicsGdi.DrawImage( image, rect );
		}


		/// <summary>
		/// Draws an ellipse defined by a bounding rectangle specified by 
		/// a pair of coordinates: a height, and a width.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the ellipse.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse.</param>
		/// <param name="width">Width of the bounding rectangle that defines the ellipse.</param>
		/// <param name="height">Height of the bounding rectangle that defines the ellipse.</param>
		public void DrawEllipse(
			Pen pen,
			float x,
			float y,
			float width,
			float height
			)
		{
			_graphicsGdi.DrawEllipse( pen.ToGdiPen(), x, y, width, height );
			_graphicsPdf.DrawEllipse(pen, x, y, width, height);
		}

		/// <summary>
		/// Draws an ellipse defined by a bounding RectangleF.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the ellipse.</param>
		/// <param name="rect">RectangleF structure that defines the boundaries of the ellipse.</param>
		public void DrawEllipse(
			Pen pen,
			RectangleF rect
			)
		{
			_graphicsGdi.DrawEllipse(pen.ToGdiPen(), rect);
			_graphicsPdf.DrawEllipse(pen, rect.X, rect.Y, rect.Width, rect.Height);
		}


		/// <summary>
		/// Draws a cardinal spline through a specified array of PointF structures 
		/// using a specified tension. The drawing begins offset from 
		/// the beginning of the array.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and height of the curve.</param>
		/// <param name="points">Array of PointF structures that define the spline.</param>
		/// <param name="offset">Offset from the first element in the array of the points parameter to the starting point in the curve.</param>
		/// <param name="numberOfSegments">Number of segments after the starting point to include in the curve.</param>
		/// <param name="tension">Value greater than or equal to 0.0F that specifies the tension of the curve.</param>
		public void DrawCurve(
			Pen pen,
			PointF[] points,
			int offset,
			int numberOfSegments,
			float tension
			)
		{
			_graphicsGdi.DrawCurve( pen.ToGdiPen(), points, offset,  numberOfSegments, tension );
			_graphicsPdf.DrawCurve(pen, points, offset, numberOfSegments, tension);
		}

		/// <summary>
		/// Draws a rectangle specified by a coordinate pair: a width, and a height.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the rectangle.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="width">Width of the rectangle to draw.</param>
		/// <param name="height">Height of the rectangle to draw.</param>
		public void DrawRectangle(
			Pen pen,
			int x,
			int y,
			int width,
			int height
			)
		{
			_graphicsGdi.DrawRectangle( pen.ToGdiPen(), x, y, width, height );
			_graphicsPdf.DrawRectangle(pen, x, y, width, height);
		}

		/// <summary>
		/// Draws a rectangle specified by a coordinate pair: a width, and a height.
		/// </summary>
		/// <param name="pen">A Pen object that determines the color, width, and style of the rectangle.</param>
		/// <param name="x">The x-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="y">The y-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="width">The width of the rectangle to draw.</param>
		/// <param name="height">The height of the rectangle to draw.</param>
		public void DrawRectangle(
			Pen pen,
			float x,
			float y,
			float width,
			float height
			)
		{
			_graphicsGdi.DrawRectangle(pen.ToGdiPen(), x, y, width, height);
			_graphicsPdf.DrawRectangle(pen, x, y, width, height);
		}

		/// <summary>
		/// Draws a polygon defined by an array of PointF structures.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the polygon.</param>
		/// <param name="points">Array of PointF structures that represent the vertices of the polygon.</param>
		public void DrawPolygon(
			Pen pen,
			PointF[] points
			)
		{
			_graphicsGdi.DrawPolygon( pen.ToGdiPen(), points );
			_graphicsPdf.DrawPolygon(pen, points);
		}


		/// <summary>
		/// Draws a series of line segments that connect an array of PointF structures.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the line segments.</param>
		/// <param name="points">Array of PointF structures that represent the points to connect.</param>
		public void DrawLines(
			Pen pen,
			PointF[] points
			)
		{
			_graphicsGdi.DrawLines(pen.ToGdiPen(), points);
			_graphicsPdf.DrawLines(pen, points);
		}

		/// <summary>
		/// Draws the specified text string in the specified rectangle with the specified Brush and Font objects using the formatting properties of the specified StringFormat object.
		/// </summary>
		/// <param name="s">String to draw.</param>
		/// <param name="font">Font object that defines the text format of the string.</param>
		/// <param name="brush">Brush object that determines the color and texture of the drawn text.</param>
		/// <param name="layoutRectangle">RectangleF structure that specifies the location of the drawn text.</param>
		/// <param name="format">StringFormat object that specifies formatting properties, such as line spacing and alignment, that are applied to the drawn text.</param>
		public void DrawString(
			string s,
			Font font,
			Brush brush,
			RectangleF layoutRectangle,
			StringFormat format
			)
		{
			_graphicsGdi.DrawString( s, font, brush.ToGdiBrush(), layoutRectangle, format );
			_graphicsPdf.DrawString(s, font, brush, layoutRectangle, format);
		}

		/// <summary>
		/// Draws the specified text string at the specified location with the specified Brush and Font objects using the formatting properties of the specified StringFormat object.
		/// </summary>
		/// <param name="s">String to draw.</param>
		/// <param name="font">Font object that defines the text format of the string.</param>
		/// <param name="brush">Brush object that determines the color and texture of the drawn text.</param>
		/// <param name="point">PointF structure that specifies the upper-left corner of the drawn text.</param>
		/// <param name="format">StringFormat object that specifies formatting properties, such as line spacing and alignment, that are applied to the drawn text.</param>
		public void DrawString(
			string s,
			Font font,
			Brush brush,
			PointF point,
			StringFormat format
			)
		{
			_graphicsGdi.DrawString( s, font, brush.ToGdiBrush(), point, format );
			_graphicsPdf.DrawString(s, font, brush, point, format);
		}


		/// <summary>
		/// Draws a GraphicsPath object.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the path.</param>
		/// <param name="path">GraphicsPath object to draw.</param>
		public void DrawPath(
			Pen pen,
			GraphicsPath path
			)
		{
			_graphicsGdi.DrawPath( pen.ToGdiPen(), path );
			_graphicsPdf.DrawPath(pen, path);
		}

		/// <summary>
		/// Draws a pie shape defined by an ellipse specified by a coordinate pair: a width, a height and two radial lines.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the pie shape.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="width">Width of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="height">Height of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="startAngle">Angle measured in degrees clockwise from the x-axis to the first side of the pie shape.</param>
		/// <param name="sweepAngle">Angle measured in degrees clockwise from the startAngle parameter to the second side of the pie shape.</param>
		public void DrawPie(
			Pen pen,
			float x,
			float y,
			float width,
			float height,
			float startAngle,
			float sweepAngle
			)
		{
			_graphicsGdi.DrawPie( pen.ToGdiPen(), x, y, width, height, startAngle, sweepAngle );
			_graphicsPdf.DrawPie(pen, x, y, width, height, startAngle, sweepAngle);
		}

		/// <summary>
		/// Draws an arc representing a portion of an ellipse specified by a pair of coordinates: a width, and a height.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the arc.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the rectangle that defines the ellipse.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the rectangle that defines the ellipse.</param>
		/// <param name="width">Width of the rectangle that defines the ellipse.</param>
		/// <param name="height">Height of the rectangle that defines the ellipse.</param>
		/// <param name="startAngle">Angle in degrees measured clockwise from the x-axis to the starting point of the arc.</param>
		/// <param name="sweepAngle">Angle in degrees measured clockwise from the startAngle parameter to ending point of the arc.</param>
		public void DrawArc(
			Pen pen,
			float x,
			float y,
			float width,
			float height,
			float startAngle,
			float sweepAngle
			)
		{
			_graphicsGdi.DrawArc( pen.ToGdiPen(), x, y, width, height, startAngle, sweepAngle );
			_graphicsPdf.DrawArc(pen, x, y, width, height, startAngle, sweepAngle);
		}



		#endregion // Drawing Methods

		#region Filling Methods

		/// <summary>
		/// Fills the interior of an ellipse defined by a bounding rectangle 
		/// specified by a RectangleF structure.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="rect">RectangleF structure that represents the bounding rectangle that defines the ellipse.</param>
		public void FillEllipse(
			Brush brush,
			RectangleF rect
			)
		{
			_graphicsGdi.FillEllipse( brush.ToGdiBrush(), rect );
			_graphicsPdf.FillEllipse(brush, rect);
		}

		/// <summary>
		/// Fills the interior of a GraphicsPath object.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="path">GraphicsPath object that represents the path to fill.</param>
		public void FillPath(
			Brush brush,
			GraphicsPath path
			)
		{
			_graphicsGdi.FillPath( brush.ToGdiBrush(), path );
			_graphicsPdf.FillPath(brush, path);
		}

		/// <summary>
		/// Fills the interior of a Region object.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="region">Region object that represents the area to fill.</param>
		public void FillRegion(
			Brush brush,
			Region region
			)
		{
			_graphicsGdi.FillRegion( brush.ToGdiBrush(), region );
			_graphicsPdf.FillRegion(brush, region);
		}

		/// <summary>
		/// Fills the interior of a rectangle specified by a RectangleF structure.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="rect">RectangleF structure that represents the rectangle to fill.</param>
		public void FillRectangle(
			Brush brush,
			RectangleF rect
			)
		{
			_graphicsGdi.FillRectangle( brush.ToGdiBrush(), rect );
			_graphicsPdf.FillRectangle(brush, rect);
		}

		/// <summary>
		/// Fills the interior of a rectangle specified by a pair of coordinates, a width, and a height.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the rectangle to fill.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the rectangle to fill.</param>
		/// <param name="width">Width of the rectangle to fill.</param>
		/// <param name="height">Height of the rectangle to fill.</param>
		public void FillRectangle(
			Brush brush,
			float x,
			float y,
			float width,
			float height
			)
		{
			_graphicsGdi.FillRectangle( brush.ToGdiBrush(), x, y, width, height );
			_graphicsPdf.FillRectangle(brush, x, y, width, height);
		}

		/// <summary>
		/// Fills the interior of a polygon defined by an array of points specified by PointF structures .
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="points">Array of PointF structures that represent the vertices of the polygon to fill.</param>
		public void FillPolygon(
			Brush brush,
			PointF[] points
			)
		{
			_graphicsGdi.FillPolygon( brush.ToGdiBrush(), points );
			_graphicsPdf.FillPolygon(brush, points);
		}

		/// <summary>
		/// Fills the interior of a pie section defined by an ellipse 
		/// specified by a pair of coordinates, a width, and a height 
		/// and two radial lines.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="width">Width of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="height">Height of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="startAngle">Angle in degrees measured clockwise from the x-axis to the first side of the pie section.</param>
		/// <param name="sweepAngle">Angle in degrees measured clockwise from the startAngle parameter to the second side of the pie section.</param>
		public void FillPie(
			Brush brush,
			float x,
			float y,
			float width,
			float height,
			float startAngle,
			float sweepAngle
			)
		{
			_graphicsGdi.FillPie( brush.ToGdiBrush(), x, y, width, height, startAngle, sweepAngle );
			_graphicsPdf.FillPie(brush, x, y, width, height, startAngle, sweepAngle);
		}

		#endregion // Filling Methods

		#region Other Methods

		/// <summary>
		/// Measures the specified string when drawn with the specified 
		/// Font object and formatted with the specified StringFormat object.
		/// </summary>
		/// <param name="text">String to measure.</param>
		/// <param name="font">Font object defines the text format of the string.</param>
		/// <param name="layoutArea">SizeF structure that specifies the maximum layout area for the text.</param>
		/// <param name="stringFormat">StringFormat object that represents formatting information, such as line spacing, for the string.</param>
		/// <returns>This method returns a SizeF structure that represents the size, in pixels, of the string specified in the text parameter as drawn with the font parameter and the stringFormat parameter.</returns>
		public SizeF MeasureString(
			string text,
			Font font,
			SizeF layoutArea,
			StringFormat stringFormat
			)
		{

			if (string.IsNullOrWhiteSpace(text))
				return _graphicsGdi.MeasureString(text, font, layoutArea, stringFormat);
			else
				return PdfGraphicsInfo.MeasureString(text, font, layoutArea, stringFormat, _graphicsPdf.DpiX, _graphicsPdf.DpiY);
		}

		/// <summary>
		/// Measures the specified string when drawn with the specified 
		/// Font object and formatted with the specified StringFormat object.
		/// </summary>
		/// <param name="text">String to measure.</param>
		/// <param name="font">Font object defines the text format of the string.</param>
		/// <returns>This method returns a SizeF structure that represents the size, in pixels, of the string specified in the text parameter as drawn with the font parameter and the stringFormat parameter.</returns>
		public SizeF MeasureString(
			string text,
			Font font
			)
		{
			if (string.IsNullOrWhiteSpace(text))
				return _graphicsGdi.MeasureString(text, font);
			else
				return PdfGraphicsInfo.MeasureString(text, font, _graphicsPdf.DpiX, _graphicsPdf.DpiY);
		}


		/// <summary>
		/// Saves the current state of this Graphics object and identifies the saved state with a GraphicsState object.
		/// </summary>
		/// <returns>This method returns a GraphicsState object that represents the saved state of this Graphics object.</returns>
		public GraphicsState Save()
		{
			_graphicsPdf.Save();
			return _graphicsGdi.Save();
		}

		/// <summary>
		/// Restores the state of this Graphics object to the state represented by a GraphicsState object.
		/// </summary>
		/// <param name="gstate">GraphicsState object that represents the state to which to restore this Graphics object.</param>
		public void Restore(
			GraphicsState gstate
			)
		{
			_graphicsPdf.Restore();
			_graphicsGdi.Restore( gstate );
		}

		/// <summary>
		/// Resets the clip region of this Graphics object to an infinite region.
		/// </summary>
		public void ResetClip()
		{
			_graphicsGdi.ResetClip();
		}

		/// <summary>
		/// Sets the clipping region of this Graphics object to the rectangle specified by a RectangleF structure.
		/// </summary>
		/// <param name="rect">RectangleF structure that represents the new clip region.</param>
		public void SetClip(
			RectangleF rect
			)
		{
			_graphicsGdi.SetClip( rect );
		}

		/// <summary>
		/// Sets the clipping region of this Graphics object to the result of the 
		/// specified operation combining the current clip region and the 
		/// specified GraphicsPath object.
		/// </summary>
		/// <param name="path">GraphicsPath object to combine.</param>
		/// <param name="combineMode">Member of the CombineMode enumeration that specifies the combining operation to use.</param>
		public void SetClip(
			GraphicsPath path,
			CombineMode combineMode
			)
		{
			_graphicsGdi.SetClip( path, combineMode );
		}

		/// <summary>
		/// Prepends the specified translation to the transformation matrix of this Graphics object.
		/// </summary>
		/// <param name="dx">x component of the translation.</param>
		/// <param name="dy">y component of the translation.</param>
		public void TranslateTransform(
			float dx,
			float dy
			)
		{
			_graphicsGdi.TranslateTransform( dx, dy );
			_graphicsPdf.TranslateTransform(dx, dy);
		}

		/// <summary>
		/// This method starts Selection mode
		/// </summary>
		/// <param name="hRef">The location of the referenced object, expressed as a URI reference.</param>
		/// <param name="title">Title which could be used for tooltips.</param>
		public void BeginSelection( string hRef, string title )
		{
			// Not supported for GDI+
		}

		/// <summary>
		/// This method stops Selection mode
		/// </summary>
		public void EndSelection( )
		{
			// Not supported for GDI+
		}


		#endregion // Other Methods

		#region Properties

		/// <summary>
		/// Gets or sets the world transformation for this Graphics object.
		/// </summary>
		public Matrix Transform
		{
			get
			{
				return _graphicsGdi.Transform;
			}
			set
			{
				_graphicsPdf.Transform = value;
				_graphicsGdi.Transform = value;
			}
		}

		/// <summary>
		/// Gets or sets the rendering quality for this Graphics object.
		/// </summary>
		public SmoothingMode SmoothingMode 
		{
			get
			{
				return _graphicsGdi.SmoothingMode;
			}
			set
			{
				_graphicsPdf.SmoothingMode = value;
				_graphicsGdi.SmoothingMode = value;
			}
		}

		/// <summary>
		/// Gets or sets the rendering mode for text associated with this Graphics object.
		/// </summary>
		public TextRenderingHint TextRenderingHint 
		{
			get
			{
				return _graphicsGdi.TextRenderingHint;
			}
			set
			{
				_graphicsPdf.TextRenderingHint = value;
				_graphicsGdi.TextRenderingHint = value;
			}
		}

		public CompositingQuality CompositingQuality
		{
			get
			{
				return _graphicsGdi.CompositingQuality;
			}
			set
			{
				_graphicsPdf.CompositingQuality = value;
				_graphicsGdi.CompositingQuality = value;
			}
		}
		public InterpolationMode InterpolationMode
		{
			get
			{
				return _graphicsGdi.InterpolationMode;
			}
			set
			{
				_graphicsPdf.InterpolationMode = value;
				_graphicsGdi.InterpolationMode = value;
			}
		}

		/// <summary>
		/// Gets or sets a Region object that limits the drawing region of this Graphics object.
		/// </summary>
		public Region Clip 
		{
			get
			{
				return _graphicsGdi.Clip;
			}
			set
			{
				_graphicsPdf.Clip = value;
				_graphicsGdi.Clip = value;
			}
		}

		/// <summary>
		/// Gets a value indicating whether the clipping region of this Graphics object is empty.
		/// </summary>
		public bool IsClipEmpty 
		{
			get
			{
				return _graphicsGdi.IsClipEmpty;
			}
		}

		/// <summary>
		/// Reference to the Graphics object
		/// </summary>
		public IRenderingGraphics Graphics
		{
			get
			{
				return _renderingGraphics;
			}
			set
			{
				_renderingGraphics = value;
			}
		}

		#endregion // Properties

		#region Fields


		/// <summary>
		/// Graphics object
		/// </summary>

		IRenderingGraphics _renderingGraphics = null;

		PdfGraphicsInfo _graphicsPdf { get => (_renderingGraphics as RenderingGraphicsPDF).GraphicsPdf; }
		Graphics _graphicsGdi { get => (_renderingGraphics as RenderingGraphicsPDF).GraphicsGdi; }

		#endregion // Fields
	}
}

</code>

</pre>

</details>


- - -

<details>

<summary> ▶ RenderPS.cs Source Code (Click To Expane)</summary> 

<pre>

<code>

//=================================================================
//  File:		RenderPS.cs
//
//  Namespace:	DataVisualization.Charting
//
//	Classes:	RenderPS
//
//  Purpose:	RenderPS class is chart PostScript rendering engine. It 
//              implements IChartRenderingEngine interface by mapping 
//              its methods to the drawing methods of PostScript.
//===================================================================

#region Used namespaces

using System;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Drawing.Text;
using System.Drawing.Imaging;
using System.ComponentModel;
using System.Collections;
using System.Diagnostics.CodeAnalysis;

#if Microsoft_CONTROL

using Orion.DataVisualization.Charting.Utilities;
using Orion.DataVisualization.Charting.Borders3D;
#else
	//using System.Web.UI.DataVisualization.Charting.Utilities;
	//using System.Web.UI.DataVisualization.Charting.Borders3D;
#endif

#endregion

#if Microsoft_CONTROL
namespace Orion.DataVisualization.Charting
#else
namespace System.Web.UI.DataVisualization.Charting

#endif
{
	/// <summary>
	/// GdiGraphics class is chart GDI+ rendering engine.
	/// </summary>
	[SuppressMessage("Microsoft.Naming", "CA1704:IdentifiersShouldBeSpelledCorrectly", MessageId = "Gdi")]
    internal class RenderPS : IChartRenderingEngine
	{
		public PSGraphicsInfo cGraphics;
			
		#region Constructors

		/// <summary>
		/// Default constructor
		/// </summary>
		public RenderPS(PSGraphicsInfo lPSInfo)
		{
			cGraphics = lPSInfo;
		}

		#endregion // Constructor

		#region Drawing Methods

		/// <summary>
		/// Draws a line connecting two PointF structures.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the line.</param>
		/// <param name="pt1">PointF structure that represents the first point to connect.</param>
		/// <param name="pt2">PointF structure that represents the second point to connect.</param>
		public void DrawLine(
			Pen pen,
			PointF pt1,
			PointF pt2
			)
		{
			_graphicsGdi.DrawLine(pen.ToGdiPen(), pt1, pt2);
			_graphicsPS.DrawLine(pen, pt1, pt2);

		}

		/// <summary>
		/// Draws a line connecting the two points specified by coordinate pairs.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the line.</param>
		/// <param name="x1">x-coordinate of the first point.</param>
		/// <param name="y1">y-coordinate of the first point.</param>
		/// <param name="x2">x-coordinate of the second point.</param>
		/// <param name="y2">y-coordinate of the second point.</param>
		public void DrawLine(
			Pen pen,
			float x1,
			float y1,
			float x2,
			float y2
			)
		{
			_graphicsGdi.DrawLine( pen.ToGdiPen(), x1, y1, x2, y2 );
			_graphicsPS.DrawLine(pen, new PointF(x1, y1), new PointF(x2, y2));
		}

		/// <summary>
		/// Draws the specified portion of the specified Image object at the specified location and with the specified size.
		/// </summary>
		/// <param name="image">Image object to draw.</param>
		/// <param name="destRect">Rectangle structure that specifies the location and size of the drawn image. The image is scaled to fit the rectangle.</param>
		/// <param name="srcX">x-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcY">y-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcWidth">Width of the portion of the source image to draw.</param>
		/// <param name="srcHeight">Height of the portion of the source image to draw.</param>
		/// <param name="srcUnit">Member of the GraphicsUnit enumeration that specifies the units of measure used to determine the source rectangle.</param>
		/// <param name="imageAttr">ImageAttributes object that specifies recoloring and gamma information for the image object.</param>
		public void DrawImage(
			System.Drawing.Image image,
			Rectangle destRect,
			int srcX,
			int srcY,
			int srcWidth,
			int srcHeight,
			GraphicsUnit srcUnit,
			ImageAttributes imageAttr
			)
		{
			_graphicsGdi.DrawImage( 
					image,
					destRect,
					srcX,
					srcY,
					srcWidth,
					srcHeight,
					srcUnit,
					imageAttr
				);
		}

		/// <summary>
		/// Draws the specified portion of the specified Image object at the specified location and with the specified size.
		/// </summary>
		/// <param name="image">Image object to draw.</param>
		/// <param name="destRect">Rectangle structure that specifies the location and size of the drawn image. The image is scaled to fit the rectangle.</param>
		/// <param name="srcX">x-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcY">y-coordinate of the upper-left corner of the portion of the source image to draw.</param>
		/// <param name="srcWidth">Width of the portion of the source image to draw.</param>
		/// <param name="srcHeight">Height of the portion of the source image to draw.</param>
		/// <param name="srcUnit">Member of the GraphicsUnit enumeration that specifies the units of measure used to determine the source rectangle.</param>
		/// <param name="imageAttrs">ImageAttributes object that specifies recoloring and gamma information for the image object.</param>
		public void DrawImage(
			System.Drawing.Image image,
			Rectangle destRect,
			float srcX,
			float srcY,
			float srcWidth,
			float srcHeight,
			GraphicsUnit srcUnit,
			ImageAttributes imageAttrs
			)
		{
			_graphicsGdi.DrawImage(image, destRect, srcX, srcY, srcWidth, srcHeight, srcUnit, imageAttrs);
		}


		/// <summary>
		/// Draws the specified Image object at the specified location and with the specified size.
		/// </summary>
		/// <param name="image">Image object to draw.</param>
		/// <param name="rect">RectangleF structure that specifies the location and size of the drawn image.</param>
		public void DrawImage(
			System.Drawing.Image image,
			RectangleF rect
			)
		{
			_graphicsGdi.DrawImage( image, rect );
		}


		/// <summary>
		/// Draws an ellipse defined by a bounding rectangle specified by 
		/// a pair of coordinates: a height, and a width.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the ellipse.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse.</param>
		/// <param name="width">Width of the bounding rectangle that defines the ellipse.</param>
		/// <param name="height">Height of the bounding rectangle that defines the ellipse.</param>
		public void DrawEllipse(
			Pen pen,
			float x,
			float y,
			float width,
			float height
			)
		{
			_graphicsGdi.DrawEllipse( pen.ToGdiPen(), x, y, width, height );
			_graphicsPS.DrawEllipse(pen, x, y, width, height);
		}

		/// <summary>
		/// Draws an ellipse defined by a bounding RectangleF.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the ellipse.</param>
		/// <param name="rect">RectangleF structure that defines the boundaries of the ellipse.</param>
		public void DrawEllipse(
			Pen pen,
			RectangleF rect
			)
		{
			_graphicsGdi.DrawEllipse(pen.ToGdiPen(), rect);
			_graphicsPS.DrawEllipse(pen, rect.X, rect.Y, rect.Width, rect.Height);
		}


		/// <summary>
		/// Draws a cardinal spline through a specified array of PointF structures 
		/// using a specified tension. The drawing begins offset from 
		/// the beginning of the array.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and height of the curve.</param>
		/// <param name="points">Array of PointF structures that define the spline.</param>
		/// <param name="offset">Offset from the first element in the array of the points parameter to the starting point in the curve.</param>
		/// <param name="numberOfSegments">Number of segments after the starting point to include in the curve.</param>
		/// <param name="tension">Value greater than or equal to 0.0F that specifies the tension of the curve.</param>
		public void DrawCurve(
			Pen pen,
			PointF[] points,
			int offset,
			int numberOfSegments,
			float tension
			)
		{
			_graphicsGdi.DrawCurve( pen.ToGdiPen(), points, offset,  numberOfSegments, tension );
			_graphicsPS.DrawCurve(pen, points, offset, numberOfSegments, tension);
		}

		/// <summary>
		/// Draws a rectangle specified by a coordinate pair: a width, and a height.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the rectangle.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="width">Width of the rectangle to draw.</param>
		/// <param name="height">Height of the rectangle to draw.</param>
		public void DrawRectangle(
			Pen pen,
			int x,
			int y,
			int width,
			int height
			)
		{
			_graphicsGdi.DrawRectangle( pen.ToGdiPen(), x, y, width, height );
			_graphicsPS.DrawRectangle(pen, x, y, width, height);
		}

		/// <summary>
		/// Draws a rectangle specified by a coordinate pair: a width, and a height.
		/// </summary>
		/// <param name="pen">A Pen object that determines the color, width, and style of the rectangle.</param>
		/// <param name="x">The x-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="y">The y-coordinate of the upper-left corner of the rectangle to draw.</param>
		/// <param name="width">The width of the rectangle to draw.</param>
		/// <param name="height">The height of the rectangle to draw.</param>
		public void DrawRectangle(
			Pen pen,
			float x,
			float y,
			float width,
			float height
			)
		{
			_graphicsGdi.DrawRectangle(pen.ToGdiPen(), x, y, width, height);
			_graphicsPS.DrawRectangle(pen, x, y, width, height);
		}

		/// <summary>
		/// Draws a polygon defined by an array of PointF structures.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the polygon.</param>
		/// <param name="points">Array of PointF structures that represent the vertices of the polygon.</param>
		public void DrawPolygon(
			Pen pen,
			PointF[] points
			)
		{
			_graphicsGdi.DrawPolygon( pen.ToGdiPen(), points );
			_graphicsPS.DrawPolygon(pen, points);
		}


		/// <summary>
		/// Draws a series of line segments that connect an array of PointF structures.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the line segments.</param>
		/// <param name="points">Array of PointF structures that represent the points to connect.</param>
		public void DrawLines(
			Pen pen,
			PointF[] points
			)
		{
			_graphicsGdi.DrawLines(pen.ToGdiPen(), points);
			_graphicsPS.DrawLines(pen, points);
		}

		/// <summary>
		/// Draws the specified text string in the specified rectangle with the specified Brush and Font objects using the formatting properties of the specified StringFormat object.
		/// </summary>
		/// <param name="s">String to draw.</param>
		/// <param name="font">Font object that defines the text format of the string.</param>
		/// <param name="brush">Brush object that determines the color and texture of the drawn text.</param>
		/// <param name="layoutRectangle">RectangleF structure that specifies the location of the drawn text.</param>
		/// <param name="format">StringFormat object that specifies formatting properties, such as line spacing and alignment, that are applied to the drawn text.</param>
		public void DrawString(
			string s,
			Font font,
			Brush brush,
			RectangleF layoutRectangle,
			StringFormat format
			)
		{
			_graphicsGdi.DrawString( s, font, brush.ToGdiBrush(), layoutRectangle, format );
			_graphicsPS.DrawString(s, font, brush, layoutRectangle, format);
		}

		/// <summary>
		/// Draws the specified text string at the specified location with the specified Brush and Font objects using the formatting properties of the specified StringFormat object.
		/// </summary>
		/// <param name="s">String to draw.</param>
		/// <param name="font">Font object that defines the text format of the string.</param>
		/// <param name="brush">Brush object that determines the color and texture of the drawn text.</param>
		/// <param name="point">PointF structure that specifies the upper-left corner of the drawn text.</param>
		/// <param name="format">StringFormat object that specifies formatting properties, such as line spacing and alignment, that are applied to the drawn text.</param>
		public void DrawString(
			string s,
			Font font,
			Brush brush,
			PointF point,
			StringFormat format
			)
		{
			_graphicsGdi.DrawString( s, font, brush.ToGdiBrush(), point, format );
			_graphicsPS.DrawString(s, font, brush, point, format);
		}


		/// <summary>
		/// Draws a GraphicsPath object.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the path.</param>
		/// <param name="path">GraphicsPath object to draw.</param>
		public void DrawPath(
			Pen pen,
			GraphicsPath path
			)
		{
			_graphicsGdi.DrawPath( pen.ToGdiPen(), path );
			_graphicsPS.DrawPath(pen, path);
		}

		/// <summary>
		/// Draws a pie shape defined by an ellipse specified by a coordinate pair: a width, a height and two radial lines.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the pie shape.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="width">Width of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="height">Height of the bounding rectangle that defines the ellipse from which the pie shape comes.</param>
		/// <param name="startAngle">Angle measured in degrees clockwise from the x-axis to the first side of the pie shape.</param>
		/// <param name="sweepAngle">Angle measured in degrees clockwise from the startAngle parameter to the second side of the pie shape.</param>
		public void DrawPie(
			Pen pen,
			float x,
			float y,
			float width,
			float height,
			float startAngle,
			float sweepAngle
			)
		{
			_graphicsGdi.DrawPie( pen.ToGdiPen(), x, y, width, height, startAngle, sweepAngle );
			_graphicsPS.DrawPie(pen, x, y, width, height, startAngle, sweepAngle);
		}

		/// <summary>
		/// Draws an arc representing a portion of an ellipse specified by a pair of coordinates: a width, and a height.
		/// </summary>
		/// <param name="pen">Pen object that determines the color, width, and style of the arc.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the rectangle that defines the ellipse.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the rectangle that defines the ellipse.</param>
		/// <param name="width">Width of the rectangle that defines the ellipse.</param>
		/// <param name="height">Height of the rectangle that defines the ellipse.</param>
		/// <param name="startAngle">Angle in degrees measured clockwise from the x-axis to the starting point of the arc.</param>
		/// <param name="sweepAngle">Angle in degrees measured clockwise from the startAngle parameter to ending point of the arc.</param>
		public void DrawArc(
			Pen pen,
			float x,
			float y,
			float width,
			float height,
			float startAngle,
			float sweepAngle
			)
		{
			_graphicsGdi.DrawArc( pen.ToGdiPen(), x, y, width, height, startAngle, sweepAngle );
			_graphicsPS.DrawArc(pen, x, y, width, height, startAngle, sweepAngle);
		}



		#endregion // Drawing Methods

		#region Filling Methods

		/// <summary>
		/// Fills the interior of an ellipse defined by a bounding rectangle 
		/// specified by a RectangleF structure.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="rect">RectangleF structure that represents the bounding rectangle that defines the ellipse.</param>
		public void FillEllipse(
			Brush brush,
			RectangleF rect
			)
		{
			_graphicsGdi.FillEllipse( brush.ToGdiBrush(), rect );
			_graphicsPS.FillEllipse(brush, rect);
		}

		/// <summary>
		/// Fills the interior of a GraphicsPath object.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="path">GraphicsPath object that represents the path to fill.</param>
		public void FillPath(
			Brush brush,
			GraphicsPath path
			)
		{
			_graphicsGdi.FillPath( brush.ToGdiBrush(), path );
			_graphicsPS.FillPath(brush, path);
		}

		/// <summary>
		/// Fills the interior of a Region object.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="region">Region object that represents the area to fill.</param>
		public void FillRegion(
			Brush brush,
			Region region
			)
		{
			_graphicsGdi.FillRegion( brush.ToGdiBrush(), region );
			_graphicsPS.FillRegion(brush, region);
		}

		/// <summary>
		/// Fills the interior of a rectangle specified by a RectangleF structure.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="rect">RectangleF structure that represents the rectangle to fill.</param>
		public void FillRectangle(
			Brush brush,
			RectangleF rect
			)
		{
			_graphicsGdi.FillRectangle( brush.ToGdiBrush(), rect );
			_graphicsPS.FillRectangle(brush, rect);
		}

		/// <summary>
		/// Fills the interior of a rectangle specified by a pair of coordinates, a width, and a height.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the rectangle to fill.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the rectangle to fill.</param>
		/// <param name="width">Width of the rectangle to fill.</param>
		/// <param name="height">Height of the rectangle to fill.</param>
		public void FillRectangle(
			Brush brush,
			float x,
			float y,
			float width,
			float height
			)
		{
			_graphicsGdi.FillRectangle( brush.ToGdiBrush(), x, y, width, height );
			_graphicsPS.FillRectangle(brush, x, y, width, height);
		}

		/// <summary>
		/// Fills the interior of a polygon defined by an array of points specified by PointF structures .
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="points">Array of PointF structures that represent the vertices of the polygon to fill.</param>
		public void FillPolygon(
			Brush brush,
			PointF[] points
			)
		{
			_graphicsGdi.FillPolygon( brush.ToGdiBrush(), points );
			_graphicsPS.FillPolygon(brush, points);
		}

		/// <summary>
		/// Fills the interior of a pie section defined by an ellipse 
		/// specified by a pair of coordinates, a width, and a height 
		/// and two radial lines.
		/// </summary>
		/// <param name="brush">Brush object that determines the characteristics of the fill.</param>
		/// <param name="x">x-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="y">y-coordinate of the upper-left corner of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="width">Width of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="height">Height of the bounding rectangle that defines the ellipse from which the pie section comes.</param>
		/// <param name="startAngle">Angle in degrees measured clockwise from the x-axis to the first side of the pie section.</param>
		/// <param name="sweepAngle">Angle in degrees measured clockwise from the startAngle parameter to the second side of the pie section.</param>
		public void FillPie(
			Brush brush,
			float x,
			float y,
			float width,
			float height,
			float startAngle,
			float sweepAngle
			)
		{
			_graphicsGdi.FillPie( brush.ToGdiBrush(), x, y, width, height, startAngle, sweepAngle );
			_graphicsPS.FillPie(brush, x, y, width, height, startAngle, sweepAngle);
		}

		#endregion // Filling Methods

		#region Other Methods

		/// <summary>
		/// Measures the specified string when drawn with the specified 
		/// Font object and formatted with the specified StringFormat object.
		/// </summary>
		/// <param name="text">String to measure.</param>
		/// <param name="font">Font object defines the text format of the string.</param>
		/// <param name="layoutArea">SizeF structure that specifies the maximum layout area for the text.</param>
		/// <param name="stringFormat">StringFormat object that represents formatting information, such as line spacing, for the string.</param>
		/// <returns>This method returns a SizeF structure that represents the size, in pixels, of the string specified in the text parameter as drawn with the font parameter and the stringFormat parameter.</returns>
		public SizeF MeasureString(
			string text,
			Font font,
			SizeF layoutArea,
			StringFormat stringFormat
			)
		{
		
			if (string.IsNullOrWhiteSpace(text))
				return _graphicsGdi.MeasureString( text, font, layoutArea, stringFormat );
			else
				return PdfGraphicsInfo.MeasureString(text, font, layoutArea, stringFormat, _graphicsPS.DpiX, _graphicsPS.DpiY);
		}

		/// <summary>
		/// Measures the specified string when drawn with the specified 
		/// Font object and formatted with the specified StringFormat object.
		/// </summary>
		/// <param name="text">String to measure.</param>
		/// <param name="font">Font object defines the text format of the string.</param>
		/// <returns>This method returns a SizeF structure that represents the size, in pixels, of the string specified in the text parameter as drawn with the font parameter and the stringFormat parameter.</returns>
		public SizeF MeasureString(
			string text,
			Font font
			)
		{
			if (string.IsNullOrWhiteSpace(text))
				return _graphicsGdi.MeasureString( text, font );
			else
				return PdfGraphicsInfo.MeasureString(text, font, _graphicsPS.DpiX, _graphicsPS.DpiY);
		}



		/// <summary>
		/// Saves the current state of this Graphics object and identifies the saved state with a GraphicsState object.
		/// </summary>
		/// <returns>This method returns a GraphicsState object that represents the saved state of this Graphics object.</returns>
		public GraphicsState Save()
		{
			_graphicsPS.Save();
			return _graphicsGdi.Save();
		}

		/// <summary>
		/// Restores the state of this Graphics object to the state represented by a GraphicsState object.
		/// </summary>
		/// <param name="gstate">GraphicsState object that represents the state to which to restore this Graphics object.</param>
		public void Restore(
			GraphicsState gstate
			)
		{
			_graphicsPS.Restore();
			_graphicsGdi.Restore( gstate );
		}

		/// <summary>
		/// Resets the clip region of this Graphics object to an infinite region.
		/// </summary>
		public void ResetClip()
		{
			_graphicsGdi.ResetClip();
		}

		/// <summary>
		/// Sets the clipping region of this Graphics object to the rectangle specified by a RectangleF structure.
		/// </summary>
		/// <param name="rect">RectangleF structure that represents the new clip region.</param>
		public void SetClip(
			RectangleF rect
			)
		{
			_graphicsGdi.SetClip( rect );
		}

		/// <summary>
		/// Sets the clipping region of this Graphics object to the result of the 
		/// specified operation combining the current clip region and the 
		/// specified GraphicsPath object.
		/// </summary>
		/// <param name="path">GraphicsPath object to combine.</param>
		/// <param name="combineMode">Member of the CombineMode enumeration that specifies the combining operation to use.</param>
		public void SetClip(
			GraphicsPath path,
			CombineMode combineMode
			)
		{
			_graphicsGdi.SetClip( path, combineMode );
		}

		/// <summary>
		/// Prepends the specified translation to the transformation matrix of this Graphics object.
		/// </summary>
		/// <param name="dx">x component of the translation.</param>
		/// <param name="dy">y component of the translation.</param>
		public void TranslateTransform(
			float dx,
			float dy
			)
		{
			_graphicsGdi.TranslateTransform( dx, dy );
			_graphicsPS.TranslateTransform(dx, dy);
		}

		/// <summary>
		/// This method starts Selection mode
		/// </summary>
		/// <param name="hRef">The location of the referenced object, expressed as a URI reference.</param>
		/// <param name="title">Title which could be used for tooltips.</param>
		public void BeginSelection( string hRef, string title )
		{
			// Not supported for GDI+
		}

		/// <summary>
		/// This method stops Selection mode
		/// </summary>
		public void EndSelection( )
		{
			// Not supported for GDI+
		}


		#endregion // Other Methods

		#region Properties

		/// <summary>
		/// Gets or sets the world transformation for this Graphics object.
		/// </summary>
		public Matrix Transform
		{
			get
			{
				return _graphicsGdi.Transform;
			}
			set
			{
				_graphicsPS.Transform = value;
				_graphicsGdi.Transform = value;
			}
		}

		/// <summary>
		/// Gets or sets the rendering quality for this Graphics object.
		/// </summary>
		public SmoothingMode SmoothingMode 
		{
			get
			{
				return _graphicsGdi.SmoothingMode;
			}
			set
			{
				_graphicsPS.SmoothingMode = value;
				_graphicsGdi.SmoothingMode = value;
			}
		}

		/// <summary>
		/// Gets or sets the rendering mode for text associated with this Graphics object.
		/// </summary>
		public TextRenderingHint TextRenderingHint 
		{
			get
			{
				return _graphicsGdi.TextRenderingHint;
			}
			set
			{
				_graphicsPS.TextRenderingHint = value;
				_graphicsGdi.TextRenderingHint = value;
			}
		}

		public CompositingQuality CompositingQuality
		{
			get
			{
				return _graphicsGdi.CompositingQuality;
			}
			set
			{
				_graphicsPS.CompositingQuality = value;
				_graphicsGdi.CompositingQuality = value;
			}
		}
		public InterpolationMode InterpolationMode
		{
			get
			{
				return _graphicsGdi.InterpolationMode;
			}
			set
			{
				_graphicsPS.InterpolationMode = value;
				_graphicsGdi.InterpolationMode = value;
			}
		}

		/// <summary>
		/// Gets or sets a Region object that limits the drawing region of this Graphics object.
		/// </summary>
		public Region Clip 
		{
			get
			{
				return _graphicsGdi.Clip;
			}
			set
			{
				_graphicsPS.Clip = value;
				_graphicsGdi.Clip = value;
			}
		}

		/// <summary>
		/// Gets a value indicating whether the clipping region of this Graphics object is empty.
		/// </summary>
		public bool IsClipEmpty 
		{
			get
			{
				return _graphicsGdi.IsClipEmpty;
			}
		}

		/// <summary>
		/// Reference to the Graphics object
		/// </summary>
		public IRenderingGraphics Graphics
		{
			get
			{
				return _renderingGraphics;
			}
			set
			{
				_renderingGraphics = value;
			}
		}

		#endregion // Properties

		#region Fields


		/// <summary>
		/// Graphics object
		/// </summary>

		IRenderingGraphics _renderingGraphics = null;

		PSGraphicsInfo _graphicsPS { get => (_renderingGraphics as RenderingGraphicsPS).GraphicsPS; }
		PdfGraphicsInfo _graphicsPdf { get => (_renderingGraphics as RenderingGraphicsPDF).GraphicsPdf; }
		Graphics _graphicsGdi { get => (_renderingGraphics as RenderingGraphicsPS).GraphicsGdi; }

		#endregion // Fields
	}
}

</code>

</pre>

</details>

- - -


### PDF, PostScript Rendering Code

- GDI+ Rendering

	For GDI+ Rendering, all the drawing warpper methods call the methods in .NET Forms Graphics class.
	> public Graphics GraphicsGdi { get; set; }

- - -

<details>

<summary>
▶ PdfGraphicsInfo.cs Source Code (Click To Expane)
</summary>

<pre>

<code>

#region Used namespaces

using System;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Drawing.Text;
using System.Drawing.Imaging;
using System.ComponentModel;
using System.Collections;
using System.Diagnostics.CodeAnalysis;

#if Microsoft_CONTROL

using Orion.DataVisualization.Charting.Utilities;
using Orion.DataVisualization.Charting.Borders3D;
#else
    //using System.Web.UI.DataVisualization.Charting.Utilities;
    //using System.Web.UI.DataVisualization.Charting.Borders3D;
#endif

using OrionX2.ConfigInfo;
using iTextSharp.text.pdf;
using System.Collections.Generic;

#endregion


#if Microsoft_CONTROL
namespace Orion.DataVisualization.Charting
#else
namespace System.Web.UI.DataVisualization.Charting

#endif
{

    public class PdfGraphicsInfo
    {
        public static OrionX2.Render.RenderInfo _RI;
        public enum EnumColorSpace
        {
            DEFAULT = 0,
            DeviceRGB = 1,
            DeviceCMYK = 2,
            DeviceGray = 3
        }

        public PdfGraphicsInfo(OrionX2.OrionFont.FontManagerConfig fontMgr, System.Windows.Rect rectChart, double renderDpiX, double renderDpiY, bool lbUseKValueForCMYKEqual)
		{
			OrionX2.Render.RenderInfo._FontMgr = fontMgr;
			if (_RI == null)
				_RI = new OrionX2.Render.RenderInfo();
			this.DpiX = renderDpiX;
			this.DpiY = renderDpiY;
			this.cptChartLocationPt = new System.Windows.Point(UCNV.GetPointFromPixel(rectChart.X, this.DpiX),
														  UCNV.GetPointFromPixel(rectChart.Y, this.DpiY));
			this.cszChartSizePt = new System.Windows.Size(UCNV.GetPointFromPixel(rectChart.Width, this.DpiX),
														  UCNV.GetPointFromPixel(rectChart.Height, this.DpiY));
			this.cbUseKValueForCMYKEqual = lbUseKValueForCMYKEqual;
		}

		public bool cbUseKValueForCMYKEqual;
        public iTextSharp.text.Document cPdfDoc;
        public iTextSharp.text.pdf.PdfWriter cPdfWrt;
        public iTextSharp.text.pdf.PdfContentByte cPdfCnts { get { return cPdfWrt.DirectContent; } }
        public EnumColorSpace ceColorSpace;
        public System.Windows.Point cptChartLocationPt;
        public System.Windows.Size cszChartSizePt;
        public float ChartHeightPt { get => (float)(cszChartSizePt.Height); }
        public double DpiX { get { return _RI.cdDpiX; } set { _RI.cdDpiX = value; } }
        public double DpiY { get { return _RI.cdDpiY; } set { _RI.cdDpiY = value; } }
        public bool IsClipEmpty;
        public Region Clip;
        public Matrix Transform { set { TransformSet(value); } }
        public SmoothingMode SmoothingMode;
        public TextRenderingHint TextRenderingHint;
        public CompositingQuality CompositingQuality;
        public InterpolationMode InterpolationMode;


        public void DrawLine(
            Pen pen,
            PointF pt1px,
            PointF pt2px
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            float x1Pt = (float)UCNV.GetPointFromPixel(pt1px.X, this.DpiX);
            float y1Pt = (float)(this.ChartHeightPt - UCNV.GetPointFromPixel(pt1px.Y, this.DpiY));
            float x2Pt = (float)UCNV.GetPointFromPixel(pt2px.X, this.DpiX);
            float y2Pt = (float)(this.ChartHeightPt - UCNV.GetPointFromPixel(pt2px.Y, this.DpiY));
            this.cPdfCnts.MoveTo(x1Pt, y1Pt);
            this.cPdfCnts.LineTo(x2Pt, y2Pt);
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }

        public void DrawLines(
            Pen pen,
            PointF[] points
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            if (points == null || points.Length <= 1)
                return;

            for (int i = 0; i < points.Length; i++)
            {
                float x = (float)UCNV.GetPointFromPixel(points[i].X, this.DpiX);
                float y = (float)(this.ChartHeightPt - UCNV.GetPointFromPixel(points[i].Y, this.DpiX));
                if (i == 0)
                    this.cPdfCnts.MoveTo(x, y);
                else
                    this.cPdfCnts.LineTo(x, y);
            }
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }

        public void DrawEllipse(
            Pen pen,
            float x,
            float y,
            float width,
            float height
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            float widthPt = (float)UCNV.GetPointFromPixel(width, this.DpiX);
            float heightPt = (float)UCNV.GetPointFromPixel(height, this.DpiY);
            float xPt = (float)UCNV.GetPointFromPixel(x, this.DpiX);
            float yPt = (float)(this.ChartHeightPt - UCNV.GetPointFromPixel(y, this.DpiY));
            this.cPdfCnts.Ellipse(xPt, yPt, xPt + widthPt, yPt + heightPt);
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }

        public void DrawCurve(
            Pen pen,
            PointF[] points,
            int offset,
            int numberOfSegments,
            float tension
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            using (GraphicsPath lGP = new GraphicsPath())
            {
                lGP.AddCurve(points, offset, numberOfSegments, tension);
                WriteGraphicsPath(lGP.PathData, true, true);
            }
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }

        public void DrawRectangle(
            Pen pen,
            float x,
            float y,
            float width,
            float height
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            float widthPt = (float)UCNV.GetPointFromPixel(width, this.DpiX);
            float heightPt = (float)UCNV.GetPointFromPixel(height, this.DpiY);
            float xPt = (float)UCNV.GetPointFromPixel(x, this.DpiX);
            float yPt = this.ChartHeightPt - heightPt - (float)UCNV.GetPointFromPixel(y, this.DpiY);
            this.cPdfCnts.Rectangle(xPt, yPt, widthPt, heightPt);
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }

        public void DrawPolygon(
            Pen pen,
            PointF[] points
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            if (points == null || points.Length <= 1)
                return;
            float x0 = 0f;
            float y0 = 0f;
            for(int i = 0; i < points.Length; i++)
			{
                float x = (float)UCNV.GetPointFromPixel(points[i].X, this.DpiX);
                float y = this.ChartHeightPt - (float)UCNV.GetPointFromPixel(points[i].Y, this.DpiX);
                if (i == 0)
                {
                    x0 = x;
                    y0 = y;
                    this.cPdfCnts.MoveTo(x, y);
                }
                else
                    this.cPdfCnts.LineTo(x, y);
            }
            this.cPdfCnts.LineTo(x0, y0);
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }



        public void DrawPath(
            Pen pen,
            GraphicsPath path
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            if (path == null || path.PathData == null)
                return;

            WriteGraphicsPath(path.PathData, true, true);
        
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }


        public void DrawPie(
            Pen pen,
            float x,
            float y,
            float width,
            float height,
            float startAngle,
            float sweepAngle
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            using (GraphicsPath lGP = new GraphicsPath())
            {
                lGP.AddPie(x, y, width, height, startAngle, sweepAngle);
                WriteGraphicsPath(lGP.PathData, true, true);
            }
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }

        public void DrawArc(
            Pen pen,
            float x,
            float y,
            float width,
            float height,
            float startAngle,
            float sweepAngle
            )
        {
            if (pen.Color.IsEmpty || pen.Color.A == 0)
                return;
            using (GraphicsPath lGP = new GraphicsPath())
            {
                lGP.AddArc(x, y, width, height, startAngle, sweepAngle);
                WriteGraphicsPath(lGP.PathData, true, true);
            }
            pen.ToPdfPen(this);
            this.cPdfCnts.Stroke();
        }

        public void FillEllipse(
            Brush brush,
            RectangleF rect
        )
        {
            if (brush.IsEmptyColor())
                return;
            float widthPt = (float)UCNV.GetPointFromPixel(rect.Width, this.DpiX);
            float heightPt = (float)UCNV.GetPointFromPixel(rect.Height, this.DpiY);
            float xPt = (float)UCNV.GetPointFromPixel(rect.X, this.DpiX);
            float yPt = this.ChartHeightPt - heightPt - (float)UCNV.GetPointFromPixel(rect.Y, this.DpiY);
            this.cPdfCnts.Ellipse(xPt, yPt, xPt + widthPt, yPt + heightPt);
            brush.ToPdfBrush(this);
            this.cPdfCnts.Fill();
        }

        public void FillPath(
            Brush brush,
            GraphicsPath path
            )
        {
            if (brush.IsEmptyColor())
                return;
            if (path == null || path.PathData == null)
                return;

            if (brush.GetBrushType().Equals(typeof(LinearGradientBrush)) 
                && ((LinearGradientBrush)brush).WrapMode == WrapMode.TileFlipX)
            {
                WriteGraphicsPath(path.PathData, true, true);
                ((LinearGradientBrush)brush).ToPdfBrushFlipXLeft(this);
                this.cPdfCnts.Fill();
                WriteGraphicsPath(path.PathData, true, true);
                ((LinearGradientBrush)brush).ToPdfBrushFlipXRight(this);
                this.cPdfCnts.Fill();
            }
            else
            {
                WriteGraphicsPath(path.PathData, true, true);
                brush.ToPdfBrush(this);
                this.cPdfCnts.Fill();
            }
        }

        public void FillRegion(
            Brush brush,
            Region region
            )
        {
            if (brush.IsEmptyColor())
                return;
            if (region == null)
                return;

            byte[] regionData = region.GetRegionData().Data;
            //RectangleF[] regionScans = region.

            //WriteGraphicsPath(path.PathData, true, true);

            //brush.ToPdfBrush(this);
            //this.cPdfCnts.Fill();
        }

        public void FillRectangle(
            Brush brush,
            RectangleF rect
            )
        {
            if (brush.IsEmptyColor())
                return;
            this.FillRectangle(brush, rect.X, rect.Y, rect.Width, rect.Height);

        }

        public void FillRectangle(
            Brush brush,
            float x,
            float y,
            float width,
            float height
            )
        {
            if (brush.IsEmptyColor())
                return;
            float widthPt = (float)UCNV.GetPointFromPixel(width, this.DpiX);
            float heightPt = (float)UCNV.GetPointFromPixel(height, this.DpiY);
            float xPt = (float)UCNV.GetPointFromPixel(x, this.DpiX);
            float yPt = this.ChartHeightPt - heightPt - (float)UCNV.GetPointFromPixel(y, this.DpiY);
            this.cPdfCnts.Rectangle(xPt, yPt, widthPt, heightPt);
            brush.ToPdfBrush(this);
            this.cPdfCnts.Fill();
        }

        public void FillPolygon(
            Brush brush,
            PointF[] points
            )
        {
            if (brush.IsEmptyColor())
                return;
            if (points == null || points.Length <= 1)
                return;
            float x0 = 0f;
            float y0 = 0f;
            for (int i = 0; i < points.Length; i++)
            {
                float x = (float)UCNV.GetPointFromPixel(points[i].X, this.DpiX);
                float y = this.ChartHeightPt - (float)UCNV.GetPointFromPixel(points[i].Y, this.DpiX);
                if (i == 0)
                {
                    x0 = x;
                    y0 = y;
                    this.cPdfCnts.MoveTo(x, y);
                }
                else
                    this.cPdfCnts.LineTo(x, y);
            }
            this.cPdfCnts.LineTo(x0, y0);
            brush.ToPdfBrush(this);
            this.cPdfCnts.Fill();
        }

        public void FillPie(
            Brush brush,
            float x,
            float y,
            float width,
            float height,
            float startAngle,
            float sweepAngle
            )
        {
            if (brush.IsEmptyColor())
                return;
            using (GraphicsPath lGP = new GraphicsPath())
            {
                lGP.AddPie(x, y, width, height, startAngle, sweepAngle);
                WriteGraphicsPath(lGP.PathData, true, true);
            }
            brush.ToPdfBrush(this);
            this.cPdfCnts.Fill();
        }



        //
        // Common Methods for PDF
        internal void SetColorStroke(Color lColor)
        {
            if (lColor.IsEmpty || lColor.A == 0)
                return;

            if (this.ceColorSpace == EnumColorSpace.DeviceGray)
            {
                this.cPdfCnts.SetColorStroke(new GrayColor((float)lColor.GetGray() / 255F));
            }
            else if (this.ceColorSpace == EnumColorSpace.DeviceRGB)
            {
                this.cPdfCnts.SetColorStroke(new iTextSharp.text.BaseColor(lColor.ToColor()));
            }
            else if (this.ceColorSpace == EnumColorSpace.DeviceCMYK)
            {
                if (lColor.IsSpotColor)
                {
                    PdfSpotColor lSpotColor = new PdfSpotColor(
                        lColor.SpotColorName,
                        new CMYKColor((float)lColor.CMYK.C, (float)lColor.CMYK.M,
                                      (float)lColor.CMYK.Y, (float)lColor.CMYK.K));
                    this.cPdfCnts.SetColorStroke(new SpotColor(lSpotColor, (float)lColor.SpotColorTint));
                }
                else
                {
                    if (this.cbUseKValueForCMYKEqual &&
                        (lColor.CMYK.C == lColor.CMYK.K && lColor.CMYK.M == lColor.CMYK.K && lColor.CMYK.Y == lColor.CMYK.K))
                    {
                        this.cPdfCnts.SetColorStroke(new CMYKColor(0F, 0F, 0F, (float)lColor.CMYK.K));
                    }
                    else
                    {
                        this.cPdfCnts.SetColorStroke(new CMYKColor(
                                (float)lColor.CMYK.C, (float)lColor.CMYK.M,
                                (float)lColor.CMYK.Y, (float)lColor.CMYK.K));
                    }
                }
            }
        }
        internal void SetColorFill(Color lColor)
        {
            if (lColor.IsEmpty || lColor.A == 0)
                return;

            if (this.ceColorSpace == EnumColorSpace.DeviceGray)
            {
                this.cPdfCnts.SetColorFill(new GrayColor(lColor.GetGray()));
            }
            else if (this.ceColorSpace == EnumColorSpace.DeviceRGB)
            {
                this.cPdfCnts.SetColorFill(new iTextSharp.text.BaseColor(lColor.ToColor()));
            }
            else if (this.ceColorSpace == EnumColorSpace.DeviceCMYK)
            {
                if (lColor.IsSpotColor)
                {
                    PdfSpotColor lSpotColor = new PdfSpotColor(
                        lColor.SpotColorName,
                        new CMYKColor((float)lColor.CMYK.C, (float)lColor.CMYK.M,
                                      (float)lColor.CMYK.Y, (float)lColor.CMYK.K) );
                    this.cPdfCnts.SetColorFill(new SpotColor(lSpotColor, (float)lColor.SpotColorTint));
                }
                else
                {
                    if (this.cbUseKValueForCMYKEqual && 
                        (lColor.CMYK.C == lColor.CMYK.K && lColor.CMYK.M == lColor.CMYK.K && lColor.CMYK.Y == lColor.CMYK.K))
					{
                        this.cPdfCnts.SetColorFill(new CMYKColor(0F, 0F, 0F, (float)lColor.CMYK.K));
                    }
                    else
					{
                        this.cPdfCnts.SetColorFill(new CMYKColor((float)lColor.CMYK.C, (float)lColor.CMYK.M,
                                                                 (float)lColor.CMYK.Y, (float)lColor.CMYK.K));

                    }
                    
                }

            }
        }
        //
        public void CurveTo(List<PointF> llptfCurveTo)
        {
            if (llptfCurveTo.Count == 2)
                this.cPdfCnts.CurveTo(llptfCurveTo[0].X, llptfCurveTo[0].Y, llptfCurveTo[1].X, llptfCurveTo[1].Y);
            else if (llptfCurveTo.Count == 3)
                this.cPdfCnts.CurveTo(llptfCurveTo[0].X, llptfCurveTo[0].Y, llptfCurveTo[1].X, llptfCurveTo[1].Y, llptfCurveTo[2].X, llptfCurveTo[2].Y);
        }
        public void SetLineDash(DashStyle ldsDashStyle, float lfLineWidth)
        {
            if (lfLineWidth == 0F)
                lfLineWidth = 0.1F;

            float lfDot = lfLineWidth;
            float lfDash = lfLineWidth * 3F;

            if (ldsDashStyle == DashStyle.Solid)
            {
                this.cPdfCnts.SetLineDash(lfDash);
            }
            else if (ldsDashStyle == DashStyle.Dot)
            {
                float[] lfaDash = { lfDot, lfDot };
                this.cPdfCnts.SetLineDash(lfaDash, 0);
            }
            else if (ldsDashStyle == DashStyle.Dash)
            {
                float[] lfaDash = { lfDash, lfDot };
                this.cPdfCnts.SetLineDash(lfaDash, 0);
            }
            else if (ldsDashStyle == DashStyle.DashDot)
            {
                float[] lfaDash = { lfDash, lfDot, lfDot, lfDot };
                this.cPdfCnts.SetLineDash(lfaDash, 0);
            }
            else if (ldsDashStyle == DashStyle.DashDotDot)
            {
                float[] lfaDash = { lfDash, lfDot, lfDot, lfDot, lfDot, lfDot };
                this.cPdfCnts.SetLineDash(lfaDash, 0);
            }
            else
            {
                this.cPdfCnts.SetLineDash(lfDash, 0);
            }

        }

        public void SetLineJoin(LineJoin lineJoin)
		{
            if (lineJoin == LineJoin.Bevel)
                cPdfCnts.SetLineJoin(PdfContentByte.LINE_JOIN_BEVEL);
            else if (lineJoin == LineJoin.Miter)
                cPdfCnts.SetLineJoin(PdfContentByte.LINE_JOIN_MITER);
            else if (lineJoin == LineJoin.Round)
                cPdfCnts.SetLineJoin(PdfContentByte.LINE_JOIN_ROUND);
        }

        public void SetLineCap(LineCap lineCap)
        {
            if (lineCap == LineCap.Round)
                cPdfCnts.SetLineCap(PdfContentByte.LINE_CAP_ROUND);
            else if (lineCap == LineCap.Flat)
                cPdfCnts.SetLineJoin(PdfContentByte.LINE_CAP_BUTT);
            else if (lineCap == LineCap.Square)
                cPdfCnts.SetLineJoin(PdfContentByte.LINE_CAP_PROJECTING_SQUARE);
        }

        //
        //
        public void WriteFontPath(PathData lPathD, float lfDpiX, float lfDpiY, bool lbNoClosePathAtEnd)
        {
            PointF[] lptfaPathPoint = lPathD.Points;
            List<PointF> llptfCurveTo = new List<PointF>();
            byte[] lbyaPathType = lPathD.Types;
            bool lbIsLastCommandCLOSEPATH = true;
            for (int idx = 0; idx < lptfaPathPoint.Length; idx++)
            {
                PointF lPP = new PointF(UCNV.GetPointFromPixel(lptfaPathPoint[idx].X, lfDpiX),
                                        -UCNV.GetPointFromPixel(lptfaPathPoint[idx].Y, lfDpiY));

                //lPP.Y = -lPP.Y;
                byte lPT = lbyaPathType[idx];

                if (llptfCurveTo.Count == 3)
                {
                    this.CurveTo(llptfCurveTo);
                    llptfCurveTo.Clear();
                }
                if (lPT >= 0x80)
                {
                    if (llptfCurveTo.Count == 0)
                    {
                        this.cPdfCnts.LineTo(lPP.X, lPP.Y);
                    }
                    else
                    {
                        llptfCurveTo.Add(lPP);
                        this.CurveTo(llptfCurveTo);
                        llptfCurveTo.Clear();
                    }
                }
                if (lPT == 0x00)    // Indicates that the point is the start of a figure.
                {
                    if (!lbIsLastCommandCLOSEPATH)
                        this.cPdfCnts.ClosePath();

                    this.cPdfCnts.MoveTo(lPP.X, lPP.Y);
                    lbIsLastCommandCLOSEPATH = false;
                }
                else if (lPT == 0x01)   // Indicates that the point is one of the two endpoints of a line.
                {
                    this.cPdfCnts.LineTo(lPP.X, lPP.Y);
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
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else if (lPT == 0x83)
                {
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else if (lPT == 0xa3)
                {
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else
                {
                }
            }
            if (llptfCurveTo.Count == 3)
            {
                this.CurveTo(llptfCurveTo);
                llptfCurveTo.Clear();

            }
            if (!lbIsLastCommandCLOSEPATH && !lbNoClosePathAtEnd)
                this.cPdfCnts.ClosePath();

        }
        public void WriteGraphicsPath(PathData lPathD, bool lbIsPixelData, bool lbNoClosePathAtEnd)
        {
            PointF[] lptfaPathPoint = lPathD.Points;
            List<PointF> llptfCurveTo = new List<PointF>();
            byte[] lbyaPathType = lPathD.Types;
            bool lbIsLastCommandCLOSEPATH = true;
            for (int idx = 0; idx < lptfaPathPoint.Length; idx++)
            {
                PointF lPP;
                if (lbIsPixelData)
                {
                    lPP = new PointF((float)UCNV.GetPointFromPixel(lptfaPathPoint[idx].X, this.DpiX),
                                     this.ChartHeightPt - (float)UCNV.GetPointFromPixel(lptfaPathPoint[idx].Y, this.DpiY));
                }
                else
                {
                    lPP = lptfaPathPoint[idx];
                }
                //lPP.Y = -lPP.Y;
                byte lPT = lbyaPathType[idx];

                if (llptfCurveTo.Count == 3)
                {
                    this.CurveTo(llptfCurveTo);
                    llptfCurveTo.Clear();
                }
                if (lPT >= 0x80)
                {
                    if (llptfCurveTo.Count == 0)
                    {
                        this.cPdfCnts.LineTo(lPP.X, lPP.Y);
                    }
                    else
                    {
                        llptfCurveTo.Add(lPP);
                        this.CurveTo(llptfCurveTo);
                        llptfCurveTo.Clear();
                    }
                }
                if (lPT == 0x00)    // Indicates that the point is the start of a figure.
                {
                    if (!lbIsLastCommandCLOSEPATH)
                        this.cPdfCnts.ClosePath();

                    this.cPdfCnts.MoveTo(lPP.X, lPP.Y);
                    lbIsLastCommandCLOSEPATH = false;
                }
                else if (lPT == 0x01)   // Indicates that the point is one of the two endpoints of a line.
                {
                    this.cPdfCnts.LineTo(lPP.X, lPP.Y);
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
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else if (lPT == 0x83)
                {
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else if (lPT == 0xa3)
                {
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else
                {
                }
            }
            if (llptfCurveTo.Count == 3)
            {
                this.CurveTo(llptfCurveTo);
                llptfCurveTo.Clear();

            }
            if (!lbIsLastCommandCLOSEPATH && !lbNoClosePathAtEnd)
                this.cPdfCnts.ClosePath();

        }
        public void WriteGraphicsPath(PathData lPathD, bool lbHeightInverse, float lfHeight, bool lbNoClosePathAtEnd)
        {
            PointF[] lptfaPathPoint = lPathD.Points;
            List<PointF> llptfCurveTo = new List<PointF>();
            byte[] lbyaPathType = lPathD.Types;
            bool lbIsLastCommandCLOSEPATH = true;
            for (int idx = 0; idx < lptfaPathPoint.Length; idx++)
            {
                PointF lPP = lptfaPathPoint[idx];
                if (lbHeightInverse)
                {
                    lPP.Y = lfHeight - lPP.Y;
                }
                byte lPT = lbyaPathType[idx];

                if (llptfCurveTo.Count == 3)
                {
                    this.CurveTo(llptfCurveTo);
                    llptfCurveTo.Clear();
                }
                if (lPT >= 0x80)
                {
                    if (llptfCurveTo.Count == 0)
                    {
                        this.cPdfCnts.LineTo(lPP.X, lPP.Y);
                    }
                    else
                    {
                        llptfCurveTo.Add(lPP);
                        this.CurveTo(llptfCurveTo);
                        llptfCurveTo.Clear();
                    }
                }
                if (lPT == 0x00)    // Indicates that the point is the start of a figure.
                {
                    if (!lbIsLastCommandCLOSEPATH)
                        this.cPdfCnts.ClosePath();

                    this.cPdfCnts.MoveTo(lPP.X, lPP.Y);
                    lbIsLastCommandCLOSEPATH = false;
                }
                else if (lPT == 0x01)   // Indicates that the point is one of the two endpoints of a line.
                {
                    this.cPdfCnts.LineTo(lPP.X, lPP.Y);
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
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else if (lPT == 0x83)
                {
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else if (lPT == 0xa3)
                {
                    this.cPdfCnts.ClosePath();
                    lbIsLastCommandCLOSEPATH = true;
                }
                else
                {
                }
            }
            if (llptfCurveTo.Count == 3)
            {
                this.CurveTo(llptfCurveTo);
                llptfCurveTo.Clear();

            }
            if (!lbIsLastCommandCLOSEPATH && !lbNoClosePathAtEnd)
                this.cPdfCnts.ClosePath();

        }

        public void DrawString(
            string s,
            Font font,
            Brush brush,
            PointF point,
            StringFormat format
            )
        {
            this.DrawString(s, font, brush, point.X, point.Y, 0F, 0F, format);
        }

        public void DrawString(
            string s,
            Font font,
            Brush brush,
            RectangleF layoutRectangle,
            StringFormat format)
		{
            this.DrawString(s, font, brush, layoutRectangle.X, layoutRectangle.Y, layoutRectangle.Width, layoutRectangle.Height, format);
        }

        public void DrawString(
            string s,
            Font font,
            Brush brush,
            float xPx, float yPx, float widthPx, float heightPx,
            StringFormat format)
        {

            if (string.IsNullOrWhiteSpace(s))
                return;

            OrionX2.ItemHandler.ItemText textItem = GetItemTextFromGdiFont(s, font, format, xPx, yPx, widthPx, heightPx, this.DpiX, this.DpiY);
            if (textItem == null)
                throw new Exception("GetItemTextFromGdiFont() returns NULL");


            this.cPdfCnts.SaveState();
            try
            {
                if (textItem.cFont.cFInfo == null)
                    textItem.cFont.cFInfo = OrionX2.Render.RenderInfo._FontMgr.FindFontInfo(font.FontFamily.Name);
                if (textItem.cFont.cFInfo == null)
                    textItem.cFont.cFInfo = new OrionX2.OrionFont.FontInfo();
                BaseFont pdfBFont = textItem.cFont.cFInfo.GetPDFBaseFont();
                if (pdfBFont == null)
                {
                    try
                    {
                        pdfBFont = iTextSharp.text.FontFactory.GetFont(font.FontFamily.Name, BaseFont.IDENTITY_H, BaseFont.EMBEDDED).BaseFont;
                    }
					catch
					{
                        pdfBFont = iTextSharp.text.FontFactory.GetFont("malgun.ttf", BaseFont.IDENTITY_H, BaseFont.EMBEDDED).BaseFont;
                    }
                }
                if (brush.GetBrushType().Equals(typeof(SolidBrush)))
				{
                    Color colorBrush = (((SolidBrush)brush).Color);
                    textItem.cForeground = OrionColor.FromACMYKBytes((byte)colorBrush.A,
                        (byte)colorBrush.C, (byte)colorBrush.M, (byte)colorBrush.Y, (byte)colorBrush.K);
                }
                else
                {
                    textItem.cForeground = OrionColor.FromCMYKBytes(0, 0, 0, 255);
                }
                
                System.Windows.Size lszTBoxPX;
                OrionX2.ItemHandler.TextAdjustment.SizeFChar[] lszfaChars = OrionX2.OrionFont.GlyphInfoCache.CharsSizeMeasure(textItem, _RI, null, out lszTBoxPX);
                OrionX2.ItemHandler.BoxInfo lBox = new OrionX2.ItemHandler.BoxInfo(textItem, _RI, lszTBoxPX);
                List<OrionX2.ItemHandler.TextAdjustment.LineChars> llLNChars = textItem.cTAdjustment.SetLineAlignment_Text(textItem, _RI, ref lBox, lszfaChars);
                //

                brush.ToPdfBrush(this);
                this.cPdfCnts.SetFontAndSize(pdfBFont, (float)textItem.cFont.cdSize);
                this.cPdfCnts.SetTextRise(0F);
                this.cPdfCnts.SetLeading(0F);
                if (textItem.cFont.cbBold)
                {
                    this.cPdfCnts.SetLineWidth((float)textItem.cFont.cdSize / 100F * 3F);
                    this.SetLineDash(DashStyle.Solid, (float)textItem.cFont.cdSize / 100F * 3F);
                    this.cPdfCnts.SetTextRenderingMode(PdfContentByte.TEXT_RENDER_MODE_FILL_STROKE);
                }
                else
                {
                    this.cPdfCnts.SetLineWidth((float)textItem.cFont.cdSize / 100F);
                    this.cPdfCnts.SetTextRenderingMode(PdfContentByte.TEXT_RENDER_MODE_FILL);
                }

                float boxWidthPt = (float)UCNV.GetPointFromPixel(lBox.cSizePX.Width, this.DpiX);
                float boxHeightPt = (float)UCNV.GetPointFromPixel(lBox.cSizePX.Height, this.DpiY);
                float boxLeftPt = (float)UCNV.GetPointFromPixel(lBox.cLocationPX.X + lBox.cLocationAdjPX.X, this.DpiX);
                float boxBottomPt = this.ChartHeightPt - boxHeightPt -
                    (float)UCNV.GetPointFromPixel(lBox.cLocationPX.Y + lBox.cLocationAdjPX.Y, this.DpiX);
                iTextSharp.awt.geom.AffineTransform transBox = new iTextSharp.awt.geom.AffineTransform();
                transBox.Translate(boxLeftPt, boxBottomPt);
                this.cPdfCnts.Transform(transBox);

                float lfItalicization = 0F;
                if (textItem.cFont.cbItalic)
                    lfItalicization = (float)textItem.cFont.cdSize / 20F;
                float lfAspectRatio = (float)textItem.cFont.cdAspectRatio / 100F;
                for (int iLine = 0; iLine < llLNChars.Count; iLine++)
                {
                    OrionX2.ItemHandler.TextAdjustment.LineChars lineChars = llLNChars[iLine];
                    if (lineChars.LSZFCH == null || lineChars.LSZFCH.Count == 0)
                        continue;
                    this.cPdfCnts.SaveState();
                    try
                    {
                        //float charBaseLinePt = (float)UCNV.GetPointFromPixel(lineChars.LSZFCH[0].cGlyph.cdBaseline, this.DpiY) * 2.0F;
                        float charHMargen = (float)textItem.cFont.cdSize * 0.275F;
                        float lineLeftPt = (float)UCNV.GetPointFromPixel(lineChars.PTF.X, this.DpiX);
                        float lineBottomPt = (float)UCNV.GetPointFromPixel(lineChars.PTF.Y, this.DpiY);

                        if (lfAspectRatio == 1.0F)
                        {
                            this.cPdfCnts.ConcatCTM(1F, 0F, lfItalicization, 1F, lineLeftPt, lineBottomPt + charHMargen);
                        }
                        else
						{
                            iTextSharp.awt.geom.AffineTransform transLine = new iTextSharp.awt.geom.AffineTransform();
                            transLine.Translate(lineLeftPt, lineBottomPt + charHMargen);
                            transLine.Scale(lfAspectRatio, 1);
                            this.cPdfCnts.ConcatCTM(transLine);
                            this.cPdfCnts.ConcatCTM(1F, 0F, lfItalicization, 1F, 0, 0);
                        }
                        foreach (OrionX2.ItemHandler.TextAdjustment.SizeFChar lSZFCH in lineChars.LSZFCH)
                        {
                            bool lbFontChanged = false;
                            try
                            {
                                if (!pdfBFont.CharExists(lSZFCH.CH)) 
                                {
                                    iTextSharp.text.pdf.BaseFont pdfDefaultFont = OrionX2.Render.RenderInfo._FontMgr.GetDefaultPdfBaseFont();
                                    if (pdfDefaultFont != null)
									{
                                        lbFontChanged = true;
                                        this.cPdfCnts.SetFontAndSize(pdfDefaultFont, (float)textItem.cFont.cdSize);
                                    }
                                }

                                SizeF charGapPt = new SizeF((float)UCNV.GetPointFromPixel((float)lSZFCH.GapHorz, this.DpiX),
                                                              (float)UCNV.GetPointFromPixel((float)lSZFCH.GapVert, this.DpiY));
                                this.cPdfCnts.BeginText();
                                this.cPdfCnts.ShowText(lSZFCH.CH.ToString());
                                this.cPdfCnts.EndText();
                                this.cPdfCnts.ConcatCTM(1, 0, 0, 1, (float)(charGapPt.Width / lfAspectRatio), 0);
                            }
                            catch { }
                            finally
							{
                                if (lbFontChanged)
                                    this.cPdfCnts.SetFontAndSize(pdfBFont, (float)textItem.cFont.cdSize);
                            }
                        }
                    }
                    finally
                    {
                        this.cPdfCnts.RestoreState();
                    }
                    /*
                    for (int iChar = 0;iChar < lineChars.LSZFCH.Count;iChar++)
					{
                        float charYPt = lineYPt - (float)UCNV.GetPointFromPixel(lineChars.LSZFCH[iChar].CharHeight, DpiY);
                        this.cPdfCnts.BeginText();
                        this.cPdfCnts.SetTextMatrix(charXPt, charYPt);
                        this.cPdfCnts.ShowText(lineChars.LSZFCH[iChar].CH.ToString());
                        this.cPdfCnts.EndText();
                        float charGapPt = (float)UCNV.GetPointFromPixel(lineChars.LSZFCH[iChar].GapHorz, DpiX);
                        charXPt += charGapPt;
                    }
                    */


                }

            }
            finally
            {
                this.cPdfCnts.RestoreState();
            }
        }

        public static OrionX2.ItemHandler.ItemText GetItemTextFromGdiFont(string textCnts, Font fontGdi, StringFormat formatText,
            double xPx, double yPx, double widthPx, double heightPx, double dpiX, double dpiY)
        {
            OrionX2.ItemHandler.ItemText textItem = new OrionX2.ItemHandler.ItemText();

            float fontSizePt = 10; //font.SizeInPoints; ;// * 96.0F / (float)this.DpiX;
            if (fontGdi.Unit == GraphicsUnit.Pixel)
            {
                fontSizePt = (float)UCNV.GetPointFromPixel(fontGdi.Size, dpiX);
            }
            else if (fontGdi.Unit == GraphicsUnit.Display)
            {
                fontSizePt = (float)UCNV.GetDIUFromPixel(fontGdi.Size, dpiX);
            }
            else
                fontSizePt = fontGdi.SizeInPoints * 96.0F / (float)dpiX;

            string lsFontNameTmp = fontGdi.FontFamily.Name;

            try
            {
                double xMM = UCNV.GetMMFromPixel(xPx, dpiX);
                double yMM = UCNV.GetMMFromPixel(yPx, dpiY);
                double widthMM = UCNV.GetMMFromPixel(widthPx, dpiX);
                double heightMM = UCNV.GetMMFromPixel(heightPx, dpiY);
                
                textItem.cLocation = new OrionX2.Structs.NLocation(xMM, yMM);
                textItem.cSize = new OrionX2.Structs.NSize(widthMM, heightMM);
                textItem.SetData(_RI, textCnts);
                textItem.cFont.cFInfo = OrionX2.Render.RenderInfo._FontMgr.FindFontInfo(lsFontNameTmp);
                if (textItem.cFont.cFInfo == null)
                {
                    textItem.cFont.cFInfo = new OrionX2.OrionFont.FontInfo();

                }
                textItem.cFont.cdRotation = 0;
                textItem.cFont.cdSize = fontSizePt;
                textItem.cFont.cbBold = fontGdi.Bold;
                textItem.cFont.cbItalic = fontGdi.Italic;
                textItem.cFont.cbUnderline = fontGdi.Underline;
                textItem.cFont.cbStrikethru = fontGdi.Strikeout;
                //textItem.cFont.cdAspectRatio = 100;
                if (formatText.Alignment == StringAlignment.Near)
                    textItem.ceHorzAlign = OrionX2.Enums.HorizontalAlignment.Left;
                else if (formatText.Alignment == StringAlignment.Center)
                    textItem.ceHorzAlign = OrionX2.Enums.HorizontalAlignment.Center;
                else if (formatText.Alignment == StringAlignment.Far)
                {
                    textItem.ceHorzAlign = OrionX2.Enums.HorizontalAlignment.Right;
                    textItem.cLocation.X -= UCNV.GetMMFromPoint(fontSizePt) * 0.25;
                }
                if (formatText.LineAlignment == StringAlignment.Near)
                    textItem.ceVertAlign = OrionX2.Enums.VerticalAlignment.Top;
                else if (formatText.LineAlignment == StringAlignment.Center)
                    textItem.ceVertAlign = OrionX2.Enums.VerticalAlignment.Center;
                else if (formatText.LineAlignment == StringAlignment.Far)
                    textItem.ceVertAlign = OrionX2.Enums.VerticalAlignment.Bottom;

                textItem.cForeground = new OrionColor(System.Drawing.Color.FromArgb(255, 0, 0, 0));
                // Prepare to draw string
                if (textItem.cTAdjustment.cbUseTextAutoSize && textItem.cSize.Width > 0)
                {
                    if (textItem.cTAdjustment.cbUseLineAutoWrap && textItem.cTAdjustment.cbUseLineWrapForce)
                    {
                        textItem.cTAdjustment.SetOptimalSize_ForceLineBreak(textItem, _RI, null);
                    }
                    else
                    {
                        textItem.cTAdjustment.SetOptimalSize(textItem, _RI, null);
                    }
                }
                else
                {
                    textItem.cTAdjustment.SetDefaultSize(textItem);
                }
            }
            catch
            {
                return null;
            }
            return textItem;
        }

        public static SizeF MeasureString(
            string text,
            Font font, double dpiX, double dpiY)
        {
            return MeasureString(text, font, new SizeF(0, 0), new StringFormat(), dpiX, dpiY);
        }
        public static SizeF MeasureString(
            string text,
            Font font,
            SizeF layoutArea,
            StringFormat stringFormat, double dpiX, double dpiY)
        {
            SizeF sizeText = new SizeF(0, 0);
            try
            {
                OrionX2.ItemHandler.ItemText textItem = GetItemTextFromGdiFont(text, font, stringFormat, 0, 0, layoutArea.Width, layoutArea.Height, dpiX, dpiY);
                System.Windows.Size lszTBoxPX;
                OrionX2.ItemHandler.TextAdjustment.SizeFChar[] lszfaChars = OrionX2.OrionFont.GlyphInfoCache.CharsSizeMeasure(textItem, _RI, null, out lszTBoxPX);
                OrionX2.ItemHandler.BoxInfo lBox = new OrionX2.ItemHandler.BoxInfo(textItem, _RI, lszTBoxPX);
                List<OrionX2.ItemHandler.TextAdjustment.LineChars> llLNChars = textItem.cTAdjustment.SetLineAlignment_Text(textItem, _RI, ref lBox, lszfaChars);
                RectangleF rtfAllLines = OrionX2.ItemHandler.TextAdjustment.GetAllLinesRTF(llLNChars);

                sizeText = new SizeF(rtfAllLines.Size.Width, rtfAllLines.Height);
            }
            catch
            {
                sizeText = new SizeF(0, 0);
            }
            return sizeText;

        }

        public void Save()
		{
            this.cPdfCnts.SaveState();
        }

        public void Restore()
		{
            this.cPdfCnts.RestoreState();
        }

        public void TranslateTransform(
            float dx,
            float dy
            )
        {
            iTextSharp.awt.geom.AffineTransform trnsfrm = new iTextSharp.awt.geom.AffineTransform();
            trnsfrm.Translate((float)UCNV.GetPointFromPixel(dx, this.DpiX), this.ChartHeightPt - (float)UCNV.GetPointFromPixel(dy, this.DpiY));
            this.cPdfCnts.Transform(trnsfrm);
        }


        public void TransformSet(Matrix mtrx)
		{
            iTextSharp.awt.geom.AffineTransform atrnsfrm = new iTextSharp.awt.geom.AffineTransform();
            double m00, m10, m01, m11, m02, m12;
            float[] mtrxElements = mtrx.Elements;
            m00 = mtrxElements[0];
            m01 = mtrxElements[1];
            m10 = mtrxElements[2];
            m11 = mtrxElements[3];
            m02 = UCNV.GetPointFromPixel(mtrxElements[4], this.DpiX);
            m12 = this.ChartHeightPt - UCNV.GetPointFromPixel(mtrxElements[5], this.DpiY);
            atrnsfrm.SetTransform(m00, m10, m01, m11, m02, m12);
        }

    }


}

</code>

</pre>

</details>

- - -



<details>

<summary>
▶ PSGraphicsInfo.cs Source Code (Click To Expane)
</summary>

<pre>

<code>

asdfsdafas

</code>

</pre>

</details>



- - -

