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

.NET Data Visualization library uses GDI+ for chart's components like X-Y axis lines, ellipses and label texts. So, I've implemented PDF and PostScript warappers to replace all GDI+ drawing methods that are used in chart drawing.

### PDF Chart
- ChartWinControl.cs

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


## PostScript Chart

- ChartWinControl.cs

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

