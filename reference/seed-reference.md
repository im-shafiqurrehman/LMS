# Database Seed — Reference

The seed file populates your local (and optionally staging) database with enough data to develop against. It lives at `prisma/seed.js`.

Run it with: `npx prisma db seed`

The seed is safe to run multiple times — it uses `upsert` to avoid duplicate errors.

---

## What the seed creates

- 3 categories (Frontend, Backend, DevOps)
- 1 admin user
- 2 instructor users (both verified and approved)
- 5 student users (all verified)
- 10 courses spread across categories (all in PUBLISHED status)
- 3 lessons per course (no video — videoUrl is null)
- 5 enrollments from students in various courses

---

## `prisma/seed.js`

```js
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  console.log('🌱  Seeding database...');

  // ── Categories ──────────────────────────────────────────────────────────
  const [frontend, backend, devops] = await Promise.all([
    prisma.category.upsert({
      where: { slug: 'frontend' },
      update: {},
      create: { name: 'Frontend', slug: 'frontend' }
    }),
    prisma.category.upsert({
      where: { slug: 'backend' },
      update: {},
      create: { name: 'Backend', slug: 'backend' }
    }),
    prisma.category.upsert({
      where: { slug: 'devops' },
      update: {},
      create: { name: 'DevOps', slug: 'devops' }
    }),
  ]);
  console.log('  ✅  Categories seeded');

  // ── Users ───────────────────────────────────────────────────────────────
  const password = await bcrypt.hash('Password123!', 12);
  const now = new Date();

  const admin = await prisma.user.upsert({
    where: { email: 'admin@eduflow.dev' },
    update: {},
    create: {
      email: 'admin@eduflow.dev',
      name: 'Platform Admin',
      passwordHash: password,
      role: 'ADMIN',
      emailVerifiedAt: now,
    }
  });

  const instructor1 = await prisma.user.upsert({
    where: { email: 'instructor1@eduflow.dev' },
    update: {},
    create: {
      email: 'instructor1@eduflow.dev',
      name: 'Shafiq Rehman',
      passwordHash: password,
      role: 'INSTRUCTOR',
      emailVerifiedAt: now,
    }
  });

  const instructor2 = await prisma.user.upsert({
    where: { email: 'instructor2@eduflow.dev' },
    update: {},
    create: {
      email: 'instructor2@eduflow.dev',
      name: 'Ayesha Khan',
      passwordHash: password,
      role: 'INSTRUCTOR',
      emailVerifiedAt: now,
    }
  });

  const students = await Promise.all([
    'student1@eduflow.dev',
    'student2@eduflow.dev',
    'student3@eduflow.dev',
    'student4@eduflow.dev',
    'student5@eduflow.dev',
  ].map((email, i) =>
    prisma.user.upsert({
      where: { email },
      update: {},
      create: {
        email,
        name: `Student ${i + 1}`,
        passwordHash: password,
        role: 'STUDENT',
        emailVerifiedAt: now,
      }
    })
  ));

  console.log('  ✅  Users seeded');

  // ── Courses ─────────────────────────────────────────────────────────────
  const coursesData = [
    { title: 'Mastering React', slug: 'mastering-react', categoryId: frontend.id, instructorId: instructor1.id, price: 49.99, description: 'A comprehensive deep dive into React — hooks, performance, patterns, and real-world architecture.' },
    { title: 'Node.js in Production', slug: 'nodejs-in-production', categoryId: backend.id, instructorId: instructor1.id, price: 59.99, description: 'Build, secure, and deploy Node.js APIs the right way — with real patterns from production systems.' },
    { title: 'Next.js Full Stack', slug: 'nextjs-full-stack', categoryId: frontend.id, instructorId: instructor1.id, price: 69.99, description: 'Build full-stack applications with Next.js 14 App Router, server components, and Prisma.' },
    { title: 'PostgreSQL Mastery', slug: 'postgresql-mastery', categoryId: backend.id, instructorId: instructor2.id, price: 44.99, description: 'Go beyond basic queries — indexes, query planning, transactions, and performance tuning.' },
    { title: 'Docker and Kubernetes', slug: 'docker-and-kubernetes', categoryId: devops.id, instructorId: instructor2.id, price: 79.99, description: 'Containerise your applications and orchestrate them at scale with Kubernetes.' },
    { title: 'Redis for Developers', slug: 'redis-for-developers', categoryId: backend.id, instructorId: instructor2.id, price: 39.99, description: 'Caching, pub/sub, rate limiting, and session storage with Redis.' },
    { title: 'TypeScript Fundamentals', slug: 'typescript-fundamentals', categoryId: frontend.id, instructorId: instructor1.id, price: 34.99, description: 'Stop writing JavaScript with no type safety — learn TypeScript from the ground up.' },
    { title: 'AWS for Backend Engineers', slug: 'aws-for-backend-engineers', categoryId: devops.id, instructorId: instructor2.id, price: 89.99, description: 'EC2, S3, Lambda, RDS, VPC — everything a backend engineer needs to ship on AWS.' },
    { title: 'API Design Principles', slug: 'api-design-principles', categoryId: backend.id, instructorId: instructor1.id, price: 29.99, description: 'Design REST APIs that developers love — versioning, pagination, error formats, documentation.' },
    { title: 'CI/CD with GitHub Actions', slug: 'cicd-with-github-actions', categoryId: devops.id, instructorId: instructor2.id, price: 49.99, description: 'Automate your test, build, and deploy pipeline with GitHub Actions.' },
  ];

  const courses = await Promise.all(
    coursesData.map(data =>
      prisma.course.upsert({
        where: { slug: data.slug },
        update: {},
        create: {
          ...data,
          price: data.price,
          status: 'PUBLISHED',
          totalLessons: 3,
        }
      })
    )
  );

  console.log('  ✅  Courses seeded');

  // ── Lessons ─────────────────────────────────────────────────────────────
  for (const course of courses) {
    await Promise.all([
      prisma.lesson.upsert({
        where: { courseId_position: { courseId: course.id, position: 1 } },
        update: {},
        create: { courseId: course.id, title: 'Introduction and Setup', position: 1, durationSeconds: 600, isFree: true }
      }),
      prisma.lesson.upsert({
        where: { courseId_position: { courseId: course.id, position: 2 } },
        update: {},
        create: { courseId: course.id, title: 'Core Concepts', position: 2, durationSeconds: 1800, isFree: false }
      }),
      prisma.lesson.upsert({
        where: { courseId_position: { courseId: course.id, position: 3 } },
        update: {},
        create: { courseId: course.id, title: 'Building the Project', position: 3, durationSeconds: 3600, isFree: false }
      }),
    ]);
  }

  console.log('  ✅  Lessons seeded');

  // ── Enrollments ─────────────────────────────────────────────────────────
  // Note: real enrollments go through Stripe — these are test-only direct inserts
  const enrollmentData = [
    { studentId: students[0].id, courseId: courses[0].id, amountPaid: 49.99 },
    { studentId: students[0].id, courseId: courses[1].id, amountPaid: 59.99 },
    { studentId: students[1].id, courseId: courses[0].id, amountPaid: 49.99 },
    { studentId: students[2].id, courseId: courses[4].id, amountPaid: 79.99 },
    { studentId: students[3].id, courseId: courses[2].id, amountPaid: 69.99 },
  ];

  for (const data of enrollmentData) {
    await prisma.enrollment.upsert({
      where: { studentId_courseId: { studentId: data.studentId, courseId: data.courseId } },
      update: {},
      create: {
        ...data,
        amountPaid: data.amountPaid,
        stripePaymentIntentId: `seed_pi_${data.studentId.slice(0, 8)}_${data.courseId.slice(0, 8)}`,
      }
    });
  }

  console.log('  ✅  Enrollments seeded');
  console.log('\n🎉  Seed complete!\n');
  console.log('Test credentials (all use password: Password123!):');
  console.log('  Admin      : admin@eduflow.dev');
  console.log('  Instructor : instructor1@eduflow.dev');
  console.log('  Instructor : instructor2@eduflow.dev');
  console.log('  Student    : student1@eduflow.dev');
}

main()
  .catch((e) => { console.error(e); process.exit(1); })
  .finally(async () => { await prisma.$disconnect(); });
```
