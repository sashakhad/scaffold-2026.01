# Next.js + TypeScript + Tailwind + Prisma + shadcn-ui Scaffold

This project is a modern web app scaffold using:

- [Next.js 15 (App Router)](https://nextjs.org/docs/app)
- [TypeScript](https://www.typescriptlang.org/)
- [Tailwind CSS v4 (canary)](https://tailwindcss.com/)
- [shadcn/ui](https://ui.shadcn.com/)
- [Prisma ORM](https://www.prisma.io/)
- [React Hook Form](https://react-hook-form.com/)
- [Zod](https://zod.dev/)
- [Lucide icons](https://lucide.dev/)
- [pnpm](https://pnpm.io/)
- Deployed via [Vercel](https://vercel.com/)

---

## 🚀 Getting Started

### 1. Clone + Install

```bash
pnpm install
```

Make sure you're using Node 18+ and `pnpm`.

---

### 2. Environment Setup

Copy `.env.example` and fill in the actual values:

```bash
cp .env.example .env
```

You'll need:

- `DATABASE_URL` for Prisma
- Any other secrets depending on services (e.g. Vercel, Clerk, S3)

---

### 3. Prisma

```bash
npx prisma generate
npx prisma migrate dev --name init
```

Update your `schema.prisma` as needed.

---

### 4. shadcn/ui

```bash
pnpm dlx shadcn-ui@latest init
```

Follow the CLI prompts:

- Directory: `app`
- Components path: `components/ui`
- Tailwind: `yes`

---

## ✨ Dev Commands

```bash
pnpm dev            # start dev server
pnpm build          # build for prod
pnpm lint           # run eslint
pnpm format         # run prettier
pnpm prisma studio  # open Prisma UI
```

---

## 🧠 Folder Structure

```
/app
  /api        – server actions + route handlers
  /[page]     – app router pages
/components   – UI components (shadcn style)
/lib          – utils/helpers
/prisma       – Prisma schema + migrations
```

---

## 💅 Tailwind v4 (canary)

Using CSS variables instead of `theme()` in `.ts` files. If you need to use `@apply` with themes, define custom props in `tailwind.config.ts`.

---

## 🧠 Notes

- Uses `@/*` path aliases via `tsconfig.json`
- Uses `src/` directory layout
- Designed to work cleanly in Cursor IDE with AI-native refactors and UI generation
- Vercel + GitHub are set up for CI/CD

---

## 📦 Deployment (Vercel)

1. Push to GitHub  
2. Connect repo to Vercel  
3. Add your `.env` values  
4. Done ✅

---

## 🧪 Testing (optional add-on ideas)

Consider adding later:

- `vitest` for unit testing  
- `playwright` or `cypress` for E2E  
- `jest-dom` for UI snapshot tests# scaffold-2025.04
