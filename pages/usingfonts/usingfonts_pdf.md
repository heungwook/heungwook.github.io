---
title: Drawing Texts in PDF
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: usingfonts_pdf.html
folder: pdfpostscript
---


## Drawing Texts in PDF

### Using TTF/OTF fonts with embedding subset

- Emgedding fonts are required to view and print PDF documents without environtal restrictions. If the pc doesn't have fonts that are used in PDF, the viewer tries to find font resource in the PDF file, in case the fonts are not embedded in the PDF, the texts cannot be seen correctly.

- Embedding fonts makes the PDF file bigger, because the size of font file with CJK(Chinese - 30,000~150,000 characters in unicode, Japanese, Korean - 11,000 characters) fonts is few mega bytes or larger. So, we need subset embedding for embedding only used characters in the PDF document. 

    >lPdfBFont = FontFactory.GetFont(lsFontNameEng, BaseFont.IDENTITY_H, BaseFont.EMBEDDED).BaseFont;<br/>
    lPdfBFont.Subset = true;</br>

### Drawing Texts in PDF Page

- Set position and rotation with transformation

    >lPdfContent.SetFontAndSize(lPdfBFont, lfFontSize * lfFontAdjustScale);<br/>
    iTextSharp.awt.geom.AffineTransform lTransA = new iTextSharp.awt.geom.AffineTransform();<br/>

    if (lCD.fRotation == 0F)<br/>
    {<br/>
        lTransA.Translate(lptfAPos.X + lptfAdjPos.X, lptfAPos.Y - lptfAdjPos.Y);<br/>
    }<br/>
    else<br/>
    {<br/>
        lTransA.Translate(lptfAPos.X, lptfAPos.Y);<br/>
        lTransA.Rotate(-(lCD.fRotation * Math.PI / 180.0));<br/>
        lTransA.Translate(lptfAdjPos.X, -lptfAdjPos.Y);<br/>
    }<br/>
    lPdfContent.Transform(lTransA);<br/>

- Set Hoizontal/Vertical Flip

    >if (lCD.cbFontFlipHorz || lCD.cbFontFlipVert)<br/>
    {<br/>
        lPdfContent.SaveState();<br/>
        if (lCD.cbFontFlipVert)<br/>
            lPdfContent.ConcatCTM(1, 0, 0, -1, 0, OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.CharHeight, lPD.fHeightDPI) / 2F);<br/>
        if (lCD.cbFontFlipHorz)<br/>
            lPdfContent.ConcatCTM(-1, 0, 0, 1, OrionConfigInfo.UCNV.GetPointFromPixel((float)lSZFCH.CharWidth, lPD.fWidthDPI) * 1F, 0);<br/>
    }<br/>

 
- Place Text

    >lPdfContent.SetTextRise(0F);<br/>
    lPdfContent.SetLeading(0F);<br/>
    lPdfContent.BeginText();<br/>
    <br/>
    if (lCD.bFontStyleItalic)<br/>
    {<br/>
        float lfItalicization = lfFontSize / 50F;<br/>
        lPdfContent.SaveState();<br/>
        lPdfContent.ConcatCTM(1F, 0F, lfItalicization, 1F, 0F, 0F);<br/>
        lPdfContent.ShowText(lSZFCH.CH.ToString());<br/>
        lPdfContent.RestoreState();<br/>
    }<br/>
    else<br/>
    {<br/>
        lPdfContent.ShowText(lSZFCH.CH.ToString());<br/>
    }<br/>
    <br/>
    lPdfContent.EndText();<br/>






