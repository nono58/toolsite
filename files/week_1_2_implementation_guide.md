# Week 1-2: 最初の10個ツール実装ガイド
## 詳細な技術仕様 + UI/UX + SEO戦略

---

## 📋 概要

**Week 1-2**: 最初の 10個ツール完成 → Google Search Console 登録 → 初期トラフィック獲得

**ツール一覧**:
1. PDF to JPG Converter
2. PDF to PNG Converter
3. Base64 Encode/Decode
4. URL Encode/Decode
5. JSON Formatter & Validator
6. HEIC to JPG Converter
7. Word Counter
8. EMI Loan Calculator
9. Percentage Calculator
10. Length Unit Converter

**目標**: 
- 各ツール: 2-4 時間で実装
- 総実装時間: 25-40 時間
- デプロイ: Vercel (自動)
- SEO: Google Search Console 登録 + インデックス

---

## 🏗️ 技術スタック

```
Frontend: Next.js 14 + React 18 + TypeScript
Styling: TailwindCSS + CSS Variables (dark mode対応)
State: React Hooks (useState, useCallback, useMemo)
File Handling: js-file-zip, pdfjs-dist, sharp (image), etc.
API: Server-side processing (Node.js built-in modules)
Deployment: Vercel
```

---

## 📁 ディレクトリ構造

```
/tools-site
├── app/
│   ├── layout.tsx (global)
│   ├── page.tsx (homepage)
│   ├── tools/
│   │   ├── [category]/
│   │   │   ├── [toolSlug]/
│   │   │   │   ├── page.tsx (tool page)
│   │   │   │   └── layout.tsx
│   │   │   └── page.tsx (category hub)
│   │   └── page.tsx (all tools)
│   ├── api/
│   │   └── tools/
│   │       ├── pdf-to-jpg/route.ts
│   │       ├── base64/route.ts
│   │       ├── json-formatter/route.ts
│   │       ├── word-counter/route.ts
│   │       └── ... (各ツール用API)
│   └── articles/
│       ├── [slug]/page.tsx
│       └── index.tsx
├── components/
│   ├── ToolWrapper.tsx (共通UI)
│   ├── FileUpload.tsx
│   ├── Output.tsx
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── Nav.tsx
├── lib/
│   ├── tools/ (各ツール実装)
│   ├── seo.ts (メタ生成)
│   └── utils.ts
├── public/
│   └── images/
├── styles/
│   └── globals.css
├── data/
│   ├── tools.json (全ツールメタデータ)
│   └── articles.json
└── next.config.js
```

---

## 🔧 ツール実装詳細

### **ツール1: PDF to JPG Converter**

#### 1-1. 技術仕様

```typescript
// lib/tools/pdfToJpg.ts

import { PDFDocument } from 'pdfjs-dist';
import sharp from 'sharp';

export async function convertPdfToJpg(
  pdfBuffer: Buffer,
  options: {
    quality?: number; // 80-100
    dpi?: number; // 150, 300
    scale?: number; // 1.0-3.0
  } = {}
): Promise<Buffer[]> {
  const { quality = 90, scale = 2 } = options;
  
  const pdf = await PDFDocument.getDocument(pdfBuffer).promise;
  const jpgImages: Buffer[] = [];
  
  for (let i = 0; i < pdf.numPages; i++) {
    const page = await pdf.getPage(i + 1);
    const viewport = page.getViewport({ scale });
    
    // Canvas rendering
    const canvas = createCanvas(viewport.width, viewport.height);
    const context = canvas.getContext('2d');
    
    await page.render({
      canvasContext: context,
      viewport: viewport
    }).promise;
    
    // JPG conversion
    const jpgBuffer = await sharp(canvas.toBuffer())
      .jpeg({ quality, progressive: true })
      .toBuffer();
    
    jpgImages.push(jpgBuffer);
  }
  
  return jpgImages;
}
```

#### 1-2. API Route

```typescript
// app/api/tools/pdf-to-jpg/route.ts

import { convertPdfToJpg } from '@/lib/tools/pdfToJpg';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(req: NextRequest) {
  try {
    const formData = await req.formData();
    const file = formData.get('file') as File;
    const quality = parseInt(formData.get('quality') as string) || 90;
    
    if (!file) {
      return NextResponse.json(
        { error: 'No file provided' },
        { status: 400 }
      );
    }
    
    const pdfBuffer = Buffer.from(await file.arrayBuffer());
    const jpgImages = await convertPdfToJpg(pdfBuffer, { quality });
    
    // Return as zip or first image
    return new NextResponse(jpgImages[0], {
      headers: {
        'Content-Type': 'image/jpeg',
        'Content-Disposition': 'attachment; filename="page_1.jpg"'
      }
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Conversion failed' },
      { status: 500 }
    );
  }
}
```

#### 1-3. UI Component

```typescript
// app/tools/pdf/pdf-to-jpg/page.tsx

'use client';

import { useState, useCallback } from 'react';
import ToolWrapper from '@/components/ToolWrapper';
import FileUpload from '@/components/FileUpload';
import Output from '@/components/Output';

export default function PdfToJpgPage() {
  const [file, setFile] = useState<File | null>(null);
  const [quality, setQuality] = useState(90);
  const [output, setOutput] = useState<Blob | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleConvert = useCallback(async () => {
    if (!file) return;

    setLoading(true);
    setError('');
    
    try {
      const formData = new FormData();
      formData.append('file', file);
      formData.append('quality', quality.toString());

      const response = await fetch('/api/tools/pdf-to-jpg', {
        method: 'POST',
        body: formData
      });

      if (!response.ok) throw new Error('Conversion failed');
      
      const blob = await response.blob();
      setOutput(blob);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Error occurred');
    } finally {
      setLoading(false);
    }
  }, [file, quality]);

  return (
    <ToolWrapper
      title="PDF to JPG Converter"
      description="Convert PDF pages to JPG images online, free and secure"
    >
      <div className="space-y-6">
        {/* Input Section */}
        <div className="bg-white dark:bg-slate-800 rounded-lg p-6">
          <h2 className="text-lg font-semibold mb-4">Upload PDF File</h2>
          <FileUpload
            accept=".pdf"
            onChange={setFile}
            maxSize={50} // MB
          />
          {file && (
            <p className="mt-2 text-sm text-slate-600">
              Selected: {file.name}
            </p>
          )}
        </div>

        {/* Options Section */}
        <div className="bg-white dark:bg-slate-800 rounded-lg p-6">
          <h2 className="text-lg font-semibold mb-4">Conversion Options</h2>
          <div className="space-y-4">
            <div>
              <label className="block text-sm font-medium mb-2">
                Quality: {quality}%
              </label>
              <input
                type="range"
                min="50"
                max="100"
                step="10"
                value={quality}
                onChange={(e) => setQuality(parseInt(e.target.value))}
                className="w-full"
              />
            </div>
          </div>
        </div>

        {/* Convert Button */}
        <button
          onClick={handleConvert}
          disabled={!file || loading}
          className="w-full bg-blue-600 hover:bg-blue-700 disabled:bg-slate-400 text-white font-semibold py-3 rounded-lg transition"
        >
          {loading ? 'Converting...' : 'Convert to JPG'}
        </button>

        {error && (
          <div className="text-red-600 text-sm">{error}</div>
        )}

        {/* Output Section */}
        {output && (
          <Output
            blob={output}
            filename="converted.jpg"
            type="image/jpeg"
          />
        )}
      </div>

      {/* FAQ Section */}
      <div className="mt-10 space-y-4">
        <h2 className="text-xl font-semibold">FAQ</h2>
        <div>
          <h3 className="font-medium">Why use PDF to JPG converter?</h3>
          <p className="text-sm text-slate-600">
            JPG format is more compatible for sharing, viewing on all devices,
            and embedding in presentations or web pages.
          </p>
        </div>
        <div>
          <h3 className="font-medium">Is my file secure?</h3>
          <p className="text-sm text-slate-600">
            Yes, all conversions happen in your browser. Your files are never uploaded
            to our servers.
          </p>
        </div>
      </div>
    </ToolWrapper>
  );
}
```

---

### **ツール2-4: 簡易ツール（Base64, URL, Length Converter）**

これらは **実装時間: 1-2時間** の簡易ツール

```typescript
// lib/tools/converters.ts

// Base64
export const base64Encode = (text: string): string => 
  Buffer.from(text).toString('base64');

export const base64Decode = (text: string): string =>
  Buffer.from(text, 'base64').toString('utf-8');

// URL Encode/Decode
export const urlEncode = (text: string): string =>
  encodeURIComponent(text);

export const urlDecode = (text: string): string =>
  decodeURIComponent(text);

// Length Unit Converter
const conversions: Record<string, Record<string, number>> = {
  'meter': { 'cm': 100, 'mm': 1000, 'km': 0.001, 'inch': 39.3701, 'foot': 3.28084, 'yard': 1.09361, 'mile': 0.000621371 },
  // ... more units
};

export const convertLength = (value: number, from: string, to: string): number => {
  const inMeters = value / conversions['meter'][from];
  return inMeters * conversions['meter'][to];
};
```

UI は統一テンプレート使用 → 開発時間 2-3 時間/ツール

---

### **ツール5: JSON Formatter（最高優先度）**

```typescript
// lib/tools/jsonFormatter.ts

export interface FormatResult {
  formatted: string;
  valid: boolean;
  error?: string;
  stats: {
    lines: number;
    size: number;
    keys: number;
  };
}

export function formatJson(input: string, indent: number = 2): FormatResult {
  try {
    const parsed = JSON.parse(input);
    const formatted = JSON.stringify(parsed, null, indent);
    
    return {
      formatted,
      valid: true,
      stats: {
        lines: formatted.split('\n').length,
        size: formatted.length,
        keys: Object.keys(parsed).length
      }
    };
  } catch (error) {
    return {
      formatted: '',
      valid: false,
      error: error instanceof Error ? error.message : 'Invalid JSON',
      stats: { lines: 0, size: 0, keys: 0 }
    };
  }
}

export function minifyJson(input: string): string {
  const parsed = JSON.parse(input);
  return JSON.stringify(parsed);
}
```

---

### **ツール6: Word Counter**

```typescript
// lib/tools/wordCounter.ts

export interface CountResult {
  words: number;
  characters: number;
  charactersNoSpace: number;
  sentences: number;
  paragraphs: number;
  readingTime: string;
}

export function countText(text: string): CountResult {
  const words = text
    .trim()
    .split(/\s+/)
    .filter(w => w.length > 0).length;

  const characters = text.length;
  const charactersNoSpace = text.replace(/\s/g, '').length;
  const sentences = (text.match(/[.!?]+/g) || []).length || 1;
  const paragraphs = text.split(/\n\n+/).filter(p => p.trim()).length;
  
  // Average 200 words per minute
  const readingMinutes = Math.ceil(words / 200);
  const readingTime = readingMinutes === 1 ? '1 min' : `${readingMinutes} mins`;

  return {
    words,
    characters,
    charactersNoSpace,
    sentences,
    paragraphs,
    readingTime
  };
}
```

---

## 📝 SEO メタデータ自動生成

```typescript
// lib/seo.ts

export interface ToolMetadata {
  title: string;
  description: string;
  keywords: string[];
  openGraph: {
    title: string;
    description: string;
    image: string;
  };
  schema: {
    '@context': string;
    '@type': string;
    name: string;
    description: string;
    url: string;
  };
}

const toolConfigs: Record<string, ToolMetadata> = {
  'pdf-to-jpg': {
    title: 'PDF to JPG Converter Online - Free, Fast, Secure | YourSite',
    description: 'Convert PDF to JPG in seconds. Free online tool, no signup, no file size limits. Secure & fast conversion.',
    keywords: ['pdf to jpg', 'pdf converter', 'convert pdf online'],
    openGraph: {
      title: 'Free PDF to JPG Converter',
      description: 'Convert your PDF files to JPG images instantly',
      image: '/images/pdf-to-jpg-og.png'
    },
    schema: {
      '@context': 'https://schema.org',
      '@type': 'SoftwareApplication',
      name: 'PDF to JPG Converter',
      description: 'Convert PDF documents to JPG images online',
      url: 'https://yoursite.com/tools/pdf/pdf-to-jpg'
    }
  },
  // ... more tools
};

export function getToolMetadata(toolId: string): ToolMetadata {
  return toolConfigs[toolId];
}
```

---

## 📄 支援記事テンプレート

### **Article 1: "How to Convert PDF to JPG - Step by Step Guide"**

```markdown
# How to Convert PDF to JPG - Step by Step Guide

## Introduction
PDFs are essential for document sharing, but JPG images are more versatile...

## Why Convert PDF to JPG?

### Use Cases:
1. **Social Media**: Share documents on Instagram, Facebook
2. **Presentations**: Embed images in PowerPoint
3. **Web**: Display documents on websites
4. **Compression**: Reduce file size for email sharing

## Step-by-Step Guide

### Using Our Tool:
1. Upload your PDF file
2. Adjust quality (50-100%)
3. Click "Convert"
4. Download JPG images

### Alternative Methods:
- Online tools
- Desktop software
- Cloud services

## Tips for Best Results
- Higher quality = larger file size
- Multi-page PDFs: each page becomes separate JPG
- Choose quality based on use case

## FAQ

**Q: How large can my PDF be?**
A: Up to 50 MB

**Q: Is it secure?**
A: Yes, conversion happens in your browser.

## Related Tools
- [JPG to PDF Converter](/tools/pdf/jpg-to-pdf)
- [PDF Compressor](/tools/pdf/compress)
- [Image Resizer](/tools/image/resize)

## Conclusion
Converting PDF to JPG is simple with our online tool. No software needed!
```

---

## 🔗 内部リンク戦略

**Week 1-2で構築**:

```
Home (/tools/)
├── PDF Hub (/tools/pdf/)
│   ├── PDF to JPG (ツール1)
│   ├── PDF to PNG (ツール2)
│   └── Articles:
│       └── "Best PDF Tools 2026"
│
├── Dev Hub (/tools/developer/)
│   ├── Base64 Encoder (ツール3)
│   ├── URL Encoder (ツール4)
│   ├── JSON Formatter (ツール5)
│   └── Articles:
│       └── "Developer Tools Guide"
│
├── Image Hub (/tools/image/)
│   ├── HEIC to JPG (ツール6)
│   └── Articles:
│       └── "Image Format Comparison"
│
├── Text Hub (/tools/text/)
│   ├── Word Counter (ツール7)
│   └── Articles:
│       └── "Text Analysis Tools"
│
└── Calculator Hub (/tools/calculator/)
    ├── EMI Calculator (ツール8)
    ├── Percentage Calculator (ツール9)
    └── Articles:
        └── "Financial Calculators Explained"
```

**各ツールページ下部**:
```
Related Tools:
- [Similar Tool 1]
- [Similar Tool 2]
- [Similar Tool 3]

Learn More:
- [Support Article 1]
- [Support Article 2]
```

---

## 📊 デプロイメント & SEO チェックリスト

### **Vercel へのデプロイ**

```bash
# 1. Initialize Next.js project
npx create-next-app@latest tools-site

# 2. Install dependencies
npm install sharp pdfjs-dist js-file-zip

# 3. Develop locally
npm run dev

# 4. Push to GitHub
git add .
git commit -m "Initial: 10 tools setup"
git push origin main

# 5. Vercel へデプロイ（自動）
# https://vercel.com/new - GitHub repo 接続
```

### **Google Search Console**

```
1. Property add: https://yourdomain.com
2. Verify ownership (HTML file or DNS)
3. Submit sitemap: /sitemap.xml
4. Request indexing for:
   - /tools/
   - /tools/pdf/pdf-to-jpg
   - /tools/developer/base64-encoder
   - ... (全10ツール)
```

### **Google AdSense**

```
1. 最初の 3-5 ツール デプロイ完了後に申請
2. AdSense コード挿入:
   - /tools/[tool]/page.tsx の上下
   - サイドバー (desktop)
3. Approval 待機（5-7日）
```

---

## 📈 Week 1-2 目標

| マイルストーン | 目標日 | 成功指標 |
|--------------|------|--------|
| ツール完成 | Day 5 | 10個すべて機能確認 |
| Vercel デプロイ | Day 7 | 全て https://yourdomain.com で動作 |
| GSC 登録 | Day 8 | Sitemap indexed |
| AdSense 申請 | Day 10 | Application submitted |
| Blog 記事 | Day 10 | 3-5 本公開 |
| Internal Links | Day 12 | Hub pages + cross-links |
| AdSense Approval | Day 15-20 | 広告表示開始 |

**Week 1-2 終了時の目標状態**:
✅ 10個ツール運用中
✅ Google でインデックス開始
✅ AdSense 承認待機中
✅ 初期トラフィック: 100-500 visits/日
✅ 初期収益: $0-50/日（AdSense 承認後）

