# Claude Code Agent Instructions - Akıcı Website Development

## 🎯 Project Overview

You are building the frontend website for **Akıcı**, an AI text humanization service. This is a minimalistic, conversion-focused web application where users can paste AI-generated text, humanize it through our n8n backend, and receive natural-sounding output.

**Core Purpose**: Create a clean, fast, and intuitive interface that makes text humanization effortless.

---

## 🎨 Design Philosophy

### Minimalism First
- **Less is more**: Every element must serve a purpose
- **Generous whitespace**: Let content breathe
- **Clear hierarchy**: Guide users naturally through the flow
- **No clutter**: Remove anything that doesn't add value

### Color Scheme

**Light Mode (Default):**
- **Background**: Pure white (#FFFFFF)
- **Text Primary**: Near black (#0A0A0A)
- **Text Secondary**: Dark gray (#525252)
- **Accent**: Subtle gray for borders (#E5E5E5)
- **Interactive**: Charcoal (#171717) with subtle hover states

**Dark Mode:**
- **Background**: Pure black (#000000)
- **Text Primary**: Near white (#FAFAFA)
- **Text Secondary**: Light gray (#A3A3A3)
- **Accent**: Subtle gray for borders (#262626)
- **Interactive**: White (#FAFAFA) with subtle hover states

**Action Colors:**
- **Primary Button**: Black (light mode) / White (dark mode)
- **Success**: #10B981 (emerald green)
- **Error**: #EF4444 (red)
- **Warning**: #F59E0B (amber)
- **Info**: #3B82F6 (blue)

### Typography

**Font Stack:**
```css
font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Inter', 'Helvetica Neue', Arial, sans-serif;
```

**Font Sizes:**
- Heading 1: 48px (3rem) - Hero
- Heading 2: 36px (2.25rem) - Section titles
- Heading 3: 24px (1.5rem) - Subsections
- Body Large: 18px (1.125rem) - Important text
- Body: 16px (1rem) - Default
- Body Small: 14px (0.875rem) - Secondary info
- Caption: 12px (0.75rem) - Metadata

**Font Weights:**
- Regular: 400
- Medium: 500
- Semibold: 600
- Bold: 700

**Line Heights:**
- Headings: 1.2
- Body: 1.6
- Compact: 1.4

---

## 🏗️ Technical Stack

### Required Technologies
- **Framework**: Next.js 14+ (App Router)
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **State Management**: React hooks (useState, useReducer)
- **HTTP Client**: fetch API
- **Icons**: Lucide React
- **Animations**: Framer Motion (subtle, purposeful only)
- **Forms**: React Hook Form + Zod validation

### Project Structure
```
akici-web/
├── app/
│   ├── layout.tsx          # Root layout with theme provider
│   ├── page.tsx            # Main humanizer page
│   ├── globals.css         # Global styles
│   └── api/
│       └── humanize/
│           └── route.ts    # API route to n8n webhook
├── components/
│   ├── ui/                 # Reusable UI components
│   │   ├── Button.tsx
│   │   ├── Textarea.tsx
│   │   ├── Card.tsx
│   │   ├── Badge.tsx
│   │   └── LoadingSpinner.tsx
│   ├── TextInput.tsx       # Main text input area
│   ├── TextOutput.tsx      # Results display area
│   ├── StatsDisplay.tsx    # AI score visualization
│   ├── HistoryTimeline.tsx # Iteration history
│   └── ThemeToggle.tsx     # Dark mode switch
├── lib/
│   ├── api.ts             # API client functions
│   ├── utils.ts           # Helper functions
│   └── types.ts           # TypeScript types
└── public/
    └── logo.svg           # Akıcı logo
```

---

## 📐 Layout & Components

### 1. Main Page Layout

**Structure:**
```
┌─────────────────────────────────────────────────┐
│  Header (Logo + Theme Toggle)                   │
├─────────────────────────────────────────────────┤
│                                                  │
│  Hero Section                                    │
│  - Headline: "Metinlerini doğallaştır"         │
│  - Subheading: Brief value prop                 │
│  - CTA: "Hemen dene" scroll to input           │
│                                                  │
├─────────────────────────────────────────────────┤
│                                                  │
│  Main Editor Section (Two Column)               │
│  ┌──────────────────┬──────────────────┐       │
│  │   Input Panel    │   Output Panel   │       │
│  │                  │                  │       │
│  │  [Text Input]    │  [Loading/       │       │
│  │                  │   Results]       │       │
│  │  [HumanizeBtn]   │                  │       │
│  │                  │  [Stats Display] │       │
│  │  [Clear]         │  [Copy Button]   │       │
│  └──────────────────┴──────────────────┘       │
│                                                  │
│  Iteration History (if applicable)              │
│  ┌───────────────────────────────────┐         │
│  │  Timeline showing iterations       │         │
│  └───────────────────────────────────┘         │
│                                                  │
├─────────────────────────────────────────────────┤
│  Footer (Minimal info + Links)                  │
└─────────────────────────────────────────────────┘
```

**Responsive Breakpoints:**
- Mobile: < 768px (Stack vertically)
- Tablet: 768px - 1024px (Adjust padding)
- Desktop: > 1024px (Full two-column layout)

### 2. Header Component

**Design:**
```tsx
// Sticky header with blur backdrop
<header className="sticky top-0 z-50 backdrop-blur-lg bg-white/80 dark:bg-black/80 border-b border-gray-200 dark:border-gray-800">
  <div className="max-w-7xl mx-auto px-6 py-4 flex justify-between items-center">
    {/* Logo */}
    <div className="flex items-center gap-3">
      <Logo className="h-8" />
      <span className="text-xl font-semibold">Akıcı</span>
    </div>
    
    {/* Right side actions */}
    <div className="flex items-center gap-4">
      <ThemeToggle />
      {/* Future: User menu, pricing link, etc */}
    </div>
  </div>
</header>
```

**Requirements:**
- Logo should be SVG, monochrome (black/white based on theme)
- Smooth transitions on theme change
- Minimal height (64px)
- Fixed on scroll with backdrop blur

### 3. Hero Section

**Design:**
```tsx
<section className="max-w-4xl mx-auto px-6 py-20 text-center">
  <h1 className="text-5xl md:text-6xl font-bold mb-6 tracking-tight">
    Metinlerini 
    <span className="bg-gradient-to-r from-gray-900 to-gray-600 dark:from-gray-100 dark:to-gray-400 bg-clip-text text-transparent">
      {" "}doğallaştır
    </span>
  </h1>
  
  <p className="text-xl text-gray-600 dark:text-gray-400 mb-8 max-w-2xl mx-auto">
    AI dedektörlerini atla. Yapay zeka tarafından üretilen metinleri 
    doğal ve insan gibi yazılmış metinlere dönüştür.
  </p>
  
  <div className="flex gap-4 justify-center">
    <Button size="lg" variant="primary">
      Hemen Dene
      <ChevronDown className="ml-2 h-5 w-5" />
    </Button>
  </div>
  
  {/* Stats row */}
  <div className="flex gap-8 justify-center mt-12 text-sm text-gray-500 dark:text-gray-500">
    <div>✓ Ücretsiz deneme</div>
    <div>✓ Anında sonuç</div>
    <div>✓ Güvenli</div>
  </div>
</section>
```

**Requirements:**
- Centered, generous spacing
- Smooth scroll to editor on CTA click
- Gradient text effect subtle and tasteful
- Stats row optional, can be removed for more minimalism

### 4. Text Input Component

**Design:**
```tsx
<div className="flex flex-col h-full">
  {/* Header */}
  <div className="flex justify-between items-center mb-4">
    <h3 className="text-lg font-semibold">Metin Gir</h3>
    <div className="text-sm text-gray-500">
      {characterCount} / 12,000
    </div>
  </div>
  
  {/* Textarea */}
  <textarea
    placeholder="AI tarafından üretilmiş metninizi buraya yapıştırın..."
    className="flex-1 w-full p-4 rounded-lg border border-gray-200 dark:border-gray-800 
               bg-white dark:bg-black resize-none focus:outline-none focus:ring-2 
               focus:ring-gray-900 dark:focus:ring-gray-100 transition-all"
    value={text}
    onChange={handleChange}
    maxLength={12000}
  />
  
  {/* Actions */}
  <div className="flex gap-3 mt-4">
    <Button 
      variant="primary" 
      size="lg" 
      onClick={handleHumanize}
      disabled={!isValid || isLoading}
      className="flex-1"
    >
      {isLoading ? (
        <>
          <LoadingSpinner className="mr-2" />
          İşleniyor...
        </>
      ) : (
        <>
          <Sparkles className="mr-2 h-5 w-5" />
          Doğallaştır
        </>
      )}
    </Button>
    
    <Button 
      variant="ghost" 
      size="lg"
      onClick={handleClear}
    >
      Temizle
    </Button>
  </div>
  
  {/* Validation messages */}
  {error && (
    <div className="mt-3 text-sm text-red-600 dark:text-red-400">
      {error}
    </div>
  )}
</div>
```

**Requirements:**
- Auto-resize based on content (within limits)
- Real-time character count
- Clear validation states
- Smooth loading states
- Disable button when loading or invalid

### 5. Text Output Component

**States:**

**A. Empty State:**
```tsx
<div className="flex flex-col items-center justify-center h-full text-center p-8">
  <FileText className="h-16 w-16 text-gray-300 dark:text-gray-700 mb-4" />
  <h3 className="text-lg font-medium text-gray-600 dark:text-gray-400 mb-2">
    Sonuçlar burada görünecek
  </h3>
  <p className="text-sm text-gray-500 dark:text-gray-500">
    Metninizi girin ve doğallaştır butonuna tıklayın
  </p>
</div>
```

**B. Loading State:**
```tsx
<div className="flex flex-col items-center justify-center h-full p-8">
  <div className="relative">
    <LoadingSpinner className="h-12 w-12" />
  </div>
  <p className="mt-4 text-lg font-medium">Metniniz işleniyor...</p>
  <p className="mt-2 text-sm text-gray-500">
    Iterasyon {currentIteration} / {maxIterations}
  </p>
  
  {/* Progress bar */}
  <div className="w-full max-w-xs mt-6">
    <div className="h-1 bg-gray-200 dark:bg-gray-800 rounded-full overflow-hidden">
      <div 
        className="h-full bg-gray-900 dark:bg-gray-100 transition-all duration-300"
        style={{ width: `${progress}%` }}
      />
    </div>
  </div>
</div>
```

**C. Results State:**
```tsx
<div className="flex flex-col h-full">
  {/* Header with success badge */}
  <div className="flex justify-between items-center mb-4">
    <h3 className="text-lg font-semibold">Sonuç</h3>
    <Badge variant={success ? "success" : "warning"}>
      {success ? "✓ Başarılı" : "⚠ Manuel kontrol öneriliyor"}
    </Badge>
  </div>
  
  {/* Output text */}
  <div className="flex-1 p-4 rounded-lg border border-gray-200 dark:border-gray-800 
                  bg-gray-50 dark:bg-gray-900/50 overflow-y-auto">
    <p className="whitespace-pre-wrap">{outputText}</p>
  </div>
  
  {/* Stats mini display */}
  <div className="flex gap-4 mt-4 text-sm">
    <div className="flex items-center gap-2">
      <div className="h-2 w-2 rounded-full bg-red-500" />
      <span className="text-gray-600 dark:text-gray-400">
        İlk skor: {initialScore}%
      </span>
    </div>
    <div className="flex items-center gap-2">
      <div className="h-2 w-2 rounded-full bg-green-500" />
      <span className="text-gray-600 dark:text-gray-400">
        Son skor: {finalScore}%
      </span>
    </div>
  </div>
  
  {/* Actions */}
  <div className="flex gap-3 mt-4">
    <Button 
      variant="primary" 
      size="lg"
      onClick={handleCopy}
      className="flex-1"
    >
      <Copy className="mr-2 h-5 w-5" />
      Kopyala
    </Button>
    
    <Button 
      variant="ghost" 
      size="lg"
      onClick={handleDownload}
    >
      <Download className="h-5 w-5" />
    </Button>
  </div>
</div>
```

**Requirements:**
- Read-only output display
- Clear success/warning indicators
- One-click copy functionality
- Optional download as .txt
- Smooth transitions between states

### 6. Stats Display Component

**Design:**
```tsx
<div className="bg-gray-50 dark:bg-gray-900/50 rounded-xl p-6 border border-gray-200 dark:border-gray-800">
  <h4 className="text-sm font-semibold uppercase tracking-wide text-gray-500 mb-4">
    İstatistikler
  </h4>
  
  <div className="grid grid-cols-2 gap-6">
    {/* AI Score Improvement */}
    <div>
      <div className="text-2xl font-bold mb-1">
        {improvement}%
      </div>
      <div className="text-sm text-gray-600 dark:text-gray-400">
        İyileşme
      </div>
    </div>
    
    {/* Iterations Used */}
    <div>
      <div className="text-2xl font-bold mb-1">
        {iterationsUsed}
      </div>
      <div className="text-sm text-gray-600 dark:text-gray-400">
        Iterasyon
      </div>
    </div>
    
    {/* Processing Time */}
    <div>
      <div className="text-2xl font-bold mb-1">
        {processingTime}s
      </div>
      <div className="text-sm text-gray-600 dark:text-gray-400">
        İşlem süresi
      </div>
    </div>
    
    {/* Word Count Change */}
    <div>
      <div className="text-2xl font-bold mb-1">
        {wordCountChange > 0 ? "+" : ""}{wordCountChange}
      </div>
      <div className="text-sm text-gray-600 dark:text-gray-400">
        Kelime değişimi
      </div>
    </div>
  </div>
  
  {/* Score visualization */}
  <div className="mt-6">
    <div className="flex justify-between text-xs text-gray-500 mb-2">
      <span>AI Skoru</span>
      <span>{finalScore}%</span>
    </div>
    <div className="h-2 bg-gray-200 dark:bg-gray-800 rounded-full overflow-hidden">
      <div 
        className={`h-full transition-all duration-500 ${
          finalScore < 20 ? 'bg-green-500' : 
          finalScore < 50 ? 'bg-yellow-500' : 
          'bg-red-500'
        }`}
        style={{ width: `${finalScore}%` }}
      />
    </div>
  </div>
</div>
```

**Requirements:**
- Clear data hierarchy
- Visual score indicator with color coding
- Responsive grid layout
- Smooth animations when data updates

### 7. History Timeline Component

**Design:**
```tsx
<div className="mt-8">
  <h4 className="text-sm font-semibold uppercase tracking-wide text-gray-500 mb-4">
    İterasyon Geçmişi
  </h4>
  
  <div className="relative">
    {/* Timeline line */}
    <div className="absolute left-4 top-0 bottom-0 w-px bg-gray-200 dark:bg-gray-800" />
    
    {/* Iteration items */}
    {history.map((iteration, index) => (
      <div key={index} className="relative pl-12 pb-8 last:pb-0">
        {/* Dot */}
        <div className={`absolute left-2.5 w-3 h-3 rounded-full border-2 
                        ${iteration.aiScore < 20 
                          ? 'bg-green-500 border-green-500' 
                          : 'bg-gray-200 dark:bg-gray-800 border-gray-300 dark:border-gray-700'
                        }`} 
        />
        
        {/* Content */}
        <div className="bg-white dark:bg-black rounded-lg border border-gray-200 
                        dark:border-gray-800 p-4">
          <div className="flex justify-between items-start mb-2">
            <div className="text-sm font-medium">
              İterasyon {iteration.iteration + 1}
            </div>
            <Badge variant={iteration.aiScore < 20 ? "success" : "default"}>
              {iteration.aiScore}% AI
            </Badge>
          </div>
          
          <div className="text-xs text-gray-500">
            {iteration.flaggedSentences} bayraklı cümle
          </div>
        </div>
      </div>
    ))}
  </div>
</div>
```

**Requirements:**
- Only show when results exist
- Collapsible on mobile
- Clear visual progression
- Highlight successful iteration

---

## 🎨 Reusable UI Components

### Button Component

```tsx
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  loading?: boolean;
  children: React.ReactNode;
  onClick?: () => void;
  className?: string;
}

export function Button({
  variant = 'primary',
  size = 'md',
  disabled,
  loading,
  children,
  onClick,
  className,
}: ButtonProps) {
  const baseStyles = 'inline-flex items-center justify-center font-medium rounded-lg transition-all focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed';
  
  const variants = {
    primary: 'bg-black dark:bg-white text-white dark:text-black hover:bg-gray-800 dark:hover:bg-gray-200 focus:ring-gray-900 dark:focus:ring-gray-100',
    secondary: 'bg-gray-100 dark:bg-gray-900 text-gray-900 dark:text-gray-100 hover:bg-gray-200 dark:hover:bg-gray-800 focus:ring-gray-500',
    ghost: 'text-gray-700 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-900 focus:ring-gray-500',
    danger: 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500',
  };
  
  const sizes = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-4 py-2 text-base',
    lg: 'px-6 py-3 text-lg',
  };
  
  return (
    <button
      onClick={onClick}
      disabled={disabled || loading}
      className={cn(baseStyles, variants[variant], sizes[size], className)}
    >
      {loading ? <LoadingSpinner className="mr-2" /> : null}
      {children}
    </button>
  );
}
```

### Badge Component

```tsx
interface BadgeProps {
  variant?: 'default' | 'success' | 'warning' | 'error';
  children: React.ReactNode;
}

export function Badge({ variant = 'default', children }: BadgeProps) {
  const variants = {
    default: 'bg-gray-100 dark:bg-gray-900 text-gray-700 dark:text-gray-300',
    success: 'bg-green-100 dark:bg-green-900/30 text-green-700 dark:text-green-400',
    warning: 'bg-yellow-100 dark:bg-yellow-900/30 text-yellow-700 dark:text-yellow-400',
    error: 'bg-red-100 dark:bg-red-900/30 text-red-700 dark:text-red-400',
  };
  
  return (
    <span className={cn(
      'inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium',
      variants[variant]
    )}>
      {children}
    </span>
  );
}
```

### Loading Spinner Component

```tsx
export function LoadingSpinner({ className }: { className?: string }) {
  return (
    <svg
      className={cn('animate-spin', className)}
      xmlns="http://www.w3.org/2000/svg"
      fill="none"
      viewBox="0 0 24 24"
    >
      <circle
        className="opacity-25"
        cx="12"
        cy="12"
        r="10"
        stroke="currentColor"
        strokeWidth="4"
      />
      <path
        className="opacity-75"
        fill="currentColor"
        d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
      />
    </svg>
  );
}
```

---

## 🔌 API Integration

### API Client

```typescript
// lib/api.ts

interface HumanizeRequest {
  text: string;
}

interface HumanizeResponse {
  success: boolean;
  finalText: string;
  statistics: {
    initialAiScore: number;
    finalAiScore: number;
    improvement: number;
    improvementPercentage: string;
    iterationsUsed: number;
    exitReason: string;
    processingTime: string;
  };
  iterationHistory: Array<{
    iteration: number;
    aiScore: number;
    flaggedSentences: number;
  }>;
  textMetrics: {
    originalLength: number;
    finalLength: number;
    lengthChange: string;
    originalWords: number;
    finalWords: number;
  };
  recommendation: string;
  metadata: {
    workflowVersion: string;
    timestamp: string;
    model: string;
  };
}

export async function humanizeText(text: string): Promise<HumanizeResponse> {
  const response = await fetch('/api/humanize', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ text }),
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message || 'İşlem başarısız oldu');
  }
  
  return response.json();
}
```

### Next.js API Route

```typescript
// app/api/humanize/route.ts

import { NextRequest, NextResponse } from 'next/server';

const N8N_WEBHOOK_URL = process.env.N8N_WEBHOOK_URL!;

export async function POST(request: NextRequest) {
  try {
    const { text } = await request.json();
    
    // Validation
    if (!text || text.trim().length === 0) {
      return NextResponse.json(
        { success: false, error: 'Metin gerekli' },
        { status: 400 }
      );
    }
    
    if (text.length > 12000) {
      return NextResponse.json(
        { success: false, error: 'Metin çok uzun (max 12,000 karakter)' },
        { status: 400 }
      );
    }
    
    // Forward to n8n webhook
    const response = await fetch(N8N_WEBHOOK_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ text }),
    });
    
    if (!response.ok) {
      throw new Error('n8n workflow failed');
    }
    
    const data = await response.json();
    return NextResponse.json(data);
    
  } catch (error) {
    console.error('Humanize API error:', error);
    return NextResponse.json(
      { 
        success: false, 
        error: 'Bir hata oluştu. Lütfen tekrar deneyin.' 
      },
      { status: 500 }
    );
  }
}
```

---

## 🎭 Interaction & Animation Guidelines

### Micro-interactions

**Hover States:**
- Buttons: Slight darkening (10% opacity change)
- Links: Underline appearance
- Cards: Subtle lift (2px transform + shadow)
- Inputs: Border color change

**Focus States:**
- All interactive elements must have visible focus rings
- Use 2px offset focus ring in theme color
- Keyboard navigation should be obvious

**Loading States:**
- Spinner for async operations
- Skeleton loaders for content
- Progress indicators for long operations
- Disable buttons during loading

**Success States:**
- Brief green flash on copy success
- Checkmark animation
- Toast notification (bottom right)

### Animation Principles

**Use animations for:**
- State changes (loading → success)
- User feedback (button clicks)
- Page transitions (smooth, not jarring)
- Progressive disclosure (expand/collapse)

**Don't animate:**
- Initial page load (keep it fast)
- Constantly (avoid distraction)
- If user prefers reduced motion

**Timing:**
- Quick interactions: 150-200ms
- State changes: 300ms
- Page transitions: 400ms
- Max duration: 500ms

```tsx
// Framer Motion example for smooth state changes
<motion.div
  initial={{ opacity: 0, y: 10 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.3 }}
>
  {content}
</motion.div>
```

---

## 📱 Responsive Design

### Mobile First Approach

**Mobile (<768px):**
- Stack input/output vertically
- Full width buttons
- Reduced padding (4-6)
- Collapsible sections
- Bottom sheet for stats

**Tablet (768px - 1024px):**
- Two column layout possible
- Maintain good spacing
- Touch-friendly targets (44px min)

**Desktop (>1024px):**
- Side-by-side panels
- Generous whitespace
- Max width container (1280px)
- Hover states active

```tsx
// Example responsive layout
<div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
  <TextInput />
  <TextOutput />
</div>
```

---

## ⚡ Performance Requirements

### Core Web Vitals Targets
- **LCP (Largest Contentful Paint)**: < 2.5s
- **FID (First Input Delay)**: < 100ms
- **CLS (Cumulative Layout Shift)**: < 0.1

### Optimization Strategies

**Code Splitting:**
```tsx
// Lazy load heavy components
const StatsDisplay = dynamic(() => import('@/components/StatsDisplay'), {
  loading: () => <LoadingSpinner />,
});
```

**Image Optimization:**
- Use Next.js Image component
- SVG for icons and logo
- WebP format with fallbacks

**Bundle Size:**
- Keep initial bundle < 200KB
- Lazy load non-critical components
- Tree-shake unused code

**Caching:**
- Static assets: Cache-Control headers
- API responses: Consider short cache for identical requests
- Service worker for offline capability (optional)

---

## 🔒 Security Considerations

### Input Sanitization
```typescript
// Sanitize user input before sending
function sanitizeInput(text: string): string {
  // Remove potential XSS vectors
  return text
    .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
    .trim();
}
```

### Rate Limiting
- Implement client-side debouncing (500ms)
- Server-side rate limiting (10 req/min per IP)
- Show clear error messages

### Data Privacy
- Don't store user text in browser storage
- Clear textarea on success (optional)
- HTTPS only
- No analytics on text content

---

## 🧪 Testing Requirements

### Unit Tests
- All UI components
- API client functions
- Validation logic
- Utility functions

### Integration Tests
- Full user flow (input → submit → display results)
- Error handling scenarios
- Theme switching
- Responsive behavior

### E2E Tests (Playwright/Cypress)
```typescript
test('User can humanize text successfully', async ({ page }) => {
  await page.goto('/');
  
  // Input text
  await page.fill('textarea', 'Sample AI-generated text...');
  
  // Click humanize button
  await page.click('button:has-text("Doğallaştır")');
  
  // Wait for results
  await page.waitForSelector('[data-testid="output-text"]');
  
  // Verify success
  await expect(page.locator('[data-testid="success-badge"]')).toBeVisible();
});
```

---

## 📋 Development Checklist

### Phase 1: Setup
- [ ] Initialize Next.js project with TypeScript
- [ ] Configure Tailwind CSS
- [ ] Set up folder structure
- [ ] Install dependencies (lucide-react, framer-motion, etc.)
- [ ] Configure environment variables

### Phase 2: Core UI
- [ ] Create layout component with theme provider
- [ ] Build reusable UI components (Button, Badge, etc.)
- [ ] Implement header with logo and theme toggle
- [ ] Create hero section
- [ ] Build text input component
- [ ] Build text output component

### Phase 3: Integration
- [ ] Set up API route to n8n webhook
- [ ] Implement API client functions
- [ ] Connect UI to API
- [ ] Add loading states
- [ ] Add error handling

### Phase 4: Advanced Features
- [ ] Stats display component
- [ ] Iteration history timeline
- [ ] Copy to clipboard functionality
- [ ] Download as text file
- [ ] Toast notifications

### Phase 5: Polish
- [ ] Add animations and transitions
- [ ] Implement dark mode
- [ ] Responsive design testing
- [ ] Performance optimization

### Phase 6: Testing & Deployment
- [ ] Write unit tests

---


