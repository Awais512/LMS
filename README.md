# 📚 Online Learning Platform

A comprehensive e-learning platform built with Next.js, featuring course management, video processing, and seamless payment integration.

![Next JS](https://img.shields.io/badge/Next.js-black?style=flat&logo=next.js&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-2D3748?style=flat&logo=prisma&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-008CDD?style=flat&logo=stripe&logoColor=white)

## ✨ Key Features

### 📚 Learning Experience
- **Course Discovery**
  - Advanced course filtering
  - Search functionality
  - Category browsing
  - Progress tracking

- **Learning Tools**
  - Chapter completion tracking
  - Progress calculation
  - HLS video streaming
  - Interactive dashboard

### 👨‍🏫 Teacher Features
- **Course Management**
  - Course creation
  - Chapter organization
  - Drag-n-drop reordering
  - Content management

- **Content Upload**
  - Video processing with Mux
  - Thumbnail uploads
  - File attachments
  - Rich text editing

## 🗄️ Database Schema

```prisma
model Course {
  id          String  @id @default(uuid())
  userId      String
  title       String  @db.Text
  description String? @db.Text
  imageUrl    String? @db.Text
  price       Float?
  isPublished Boolean @default(false)

  categoryId String?
  category   Category? @relation(fields: [categoryId], references: [id])

  chapters    Chapter[]
  attachments Attachment[]
  purchases   Purchase[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([categoryId])
  @@fulltext([title])
}

model Category {
  id      String   @id @default(uuid())
  name    String   @unique
  courses Course[]
}

model Attachment {
  id   String @id @default(uuid())
  name String
  url  String @db.Text

  courseId  String
  course    Course   @relation(fields: [courseId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([courseId])
}

model Chapter {
  id          String  @id @default(uuid())
  title       String
  description String? @db.Text
  videoUrl    String? @db.Text
  position    Int
  isPublished Boolean @default(false)
  isFree      Boolean @default(false)

  muxData MuxData?

  courseId String
  course   Course @relation(fields: [courseId], references: [id], onDelete: Cascade)

  userProgress UserProgress[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([courseId])
}

model MuxData {
  id         String  @id @default(uuid())
  assetId    String
  playbackId String?

  chapterId String  @unique
  chapter   Chapter @relation(fields: [chapterId], references: [id], onDelete: Cascade)
}

model UserProgress {
  id     String @id @default(uuid())
  userId String

  chapterId String
  chapter   Chapter @relation(fields: [chapterId], references: [id], onDelete: Cascade)

  isCompleted Boolean @default(false)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([userId, chapterId])
  @@index([chapterId])
}

model Purchase {
  id     String @id @default(uuid())
  userId String

  courseId String
  course   Course @relation(fields: [courseId], references: [id], onDelete: Cascade)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt()

  @@unique([userId, courseId])
  @@index([courseId])
}

model StripeCustomer {
  id               String @id @default(uuid())
  userId           String @unique
  stripeCustomerId String @unique

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

## 💎 Core Features Implementation

### 🎥 Video Processing

```typescript
// lib/mux.ts
import Mux from '@mux/mux-node';

const { Video } = new Mux();

export async function createMuxUpload(file: File) {
  const upload = await Video.Uploads.create({
    new_asset_settings: {
      playback_policy: 'public',
      mp4_support: 'standard'
    }
  });

  // Create asset
  const asset = await Video.Assets.create({
    input: upload.url,
    playback_policy: 'public'
  });

  return {
    uploadUrl: upload.url,
    assetId: asset.id,
    playbackId: asset.playback_ids?.[0]?.id
  };
}

// components/VideoPlayer.tsx
import Hls from 'hls.js';

export function VideoPlayer({ playbackId }: { playbackId: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const videoUrl = `https://stream.mux.com/${playbackId}.m3u8`;

    if (video.canPlayType('application/vnd.apple.mpegurl')) {
      // Native HLS support
      video.src = videoUrl;
    } else if (Hls.isSupported()) {
      // HLS.js fallback
      const hls = new Hls();
      hls.loadSource(videoUrl);
      hls.attachMedia(video);
    }
  }, [playbackId]);

  return (
    <video
      ref={videoRef}
      controls
      className="w-full aspect-video"
    />
  );
}
```

### 📝 Course Management

```typescript
// app/api/courses/route.ts
export async function createCourse(
  userId: string,
  data: CreateCourseData
) {
  const course = await prisma.course.create({
    data: {
      userId,
      title: data.title,
      description: data.description,
      imageUrl: data.imageUrl,
      price: data.price,
      chapters: {
        create: data.chapters.map((chapter, index) => ({
          title: chapter.title,
          description: chapter.description,
          position: index
        }))
      }
    }
  });

  return course;
}

// Reorder chapters
export async function reorderChapters(
  courseId: string,
  updateData: { id: string; position: number }[]
) {
  const transaction = updateData.map((chapter) =>
    prisma.chapter.update({
      where: { id: chapter.id },
      data: { position: chapter.position }
    })
  );

  await prisma.$transaction(transaction);
}
```

### 📊 Progress Tracking

```typescript
// lib/progress.ts
export async function trackProgress(
  userId: string,
  chapterId: string,
  isCompleted: boolean
) {
  return await prisma.chapterProgress.upsert({
    where: {
      userId_chapterId: {
        userId,
        chapterId
      }
    },
    update: {
      isCompleted
    },
    create: {
      userId,
      chapterId,
      isCompleted
    }
  });
}

export async function calculateCourseProgress(
  userId: string,
  courseId: string
) {
  const chapters = await prisma.chapter.findMany({
    where: {
      courseId,
      isPublished: true
    },
    include: {
      userProgress: {
        where: {
          userId
        }
      }
    }
  });

  const completedChapters = chapters.filter((chapter) => 
    chapter.userProgress.some((progress) => progress.isCompleted)
  );

  return (completedChapters.length / chapters.length) * 100;
}
```

### 💳 Payment Integration

```typescript
// lib/stripe.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function createCheckoutSession(
  userId: string,
  courseId: string
) {
  const course = await prisma.course.findUnique({
    where: { id: courseId }
  });

  if (!course) {
    throw new Error('Course not found');
  }

  const session = await stripe.checkout.sessions.create({
    customer_email: user.email,
    payment_method_types: ['card'],
    line_items: [
      {
        price_data: {
          currency: 'usd',
          product_data: {
            name: course.title,
            description: course.description
          },
          unit_amount: Math.round(course.price * 100)
        },
        quantity: 1
      }
    ],
    mode: 'payment',
    success_url: `${process.env.NEXT_PUBLIC_APP_URL}/courses/${courseId}?success=1`,
    cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/courses/${courseId}?canceled=1`,
    metadata: {
      courseId,
      userId
    }
  });

  return session;
}
```

## 🚀 Getting Started

### Prerequisites
- Node.js 18+
- MySQL database (Planetscale)
- Clerk account
- Stripe account
- Mux account
- UploadThing account

### Installation

1. **Clone the repository**
```bash
git clone https://github.com/yourusername/learning-platform.git
cd learning-platform
```

2. **Install dependencies**
```bash
pnpm install
```

3. **Configure environment variables**
```bash
# .env
DATABASE_URL=
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
MUX_TOKEN_ID=
MUX_TOKEN_SECRET=
UPLOADTHING_SECRET=
UPLOADTHING_APP_ID=
```

4. **Initialize database**
```bash
npx prisma db push
```

5. **Start development server**
```bash
pnpm dev
```

## ⚡ Performance Optimizations

- Video streaming optimization
- Image optimization
- Lazy loading
- Caching strategies
- Database indexing

## 🔒 Security Features

- Authentication with Clerk
- Payment security with Stripe
- Content access control
- File upload validation
- Input sanitization

## 🚀 Deployment

1. **Database Setup**
```bash
npx prisma db push
```

2. **Configure Vercel**
```bash
vercel env pull
```

3. **Deploy**
```bash
vercel deploy
```

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## 📄 License

This project is licensed under the MIT License.

---

## 🙏 Acknowledgments

- [Next.js](https://nextjs.org/)
- [Prisma](https://www.prisma.io/)
- [Mux](https://mux.com/)
- [Stripe](https://stripe.com/)
- [Clerk](https://clerk.dev/)
- [UploadThing](https://uploadthing.com/)

---

Built with 📚 by Awais Raza
