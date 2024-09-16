# Backend Project Template with Next.js, React Server Components, and Modern Technologies

This project template leverages **Next.js** for routing and API calls, **React Server Components**, **TypeScript**, **Tailwind CSS**, **Shadcn** for styled components, **Zustand** for global state management, **Zod** and **react-hook-form** for form validation, **Prisma ORM** with **Supabase** for database interactions, **Stripe** for payments, **OAuth** for authentication, **Sanity** as a CMS, and **Cloudflare VPS** for hosting. This setup provides a robust foundation for developing scalable and maintainable web applications.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Setup and Installation](#setup-and-installation)
3. [Key Configuration Files](#key-configuration-files)
    - [package.json](#packagejson)
    - [tsconfig.json](#tsconfigjson)
    - [tailwind.config.js](#tailwindconfigjs)
    - [next.config.js](#nextconfigjs)
    - [Prisma Schema](#prisma-schema)
4. [Core Implementations](#core-implementations)
    - [Global State Management with Zustand](#global-state-management-with-zustand)
    - [Authentication with OAuth](#authentication-with-oauth)
    - [Stripe Integration](#stripe-integration)
    - [Form Validation with react-hook-form and Zod](#form-validation-with-react-hook-form-and-zod)
    - [Database Operations with Prisma and Supabase](#database-operations-with-prisma-and-supabase)
    - [CMS Integration with Sanity](#cms-integration-with-sanity)
    - [Styled Components with Shadcn](#styled-components-with-shadcn)
5. [Deployment with Cloudflare VPS](#deployment-with-cloudflare-vps)

---

## Project Structure

Here's an overview of the recommended project structure:

```
my-app/
├── prisma/
│   └── schema.prisma
├── public/
│   └── assets/
├── src/
│   ├── components/
│   │   └── ui/
│   ├── hooks/
│   │   └── useStore.ts
│   ├── pages/
│   │   ├── api/
│   │   │   └── auth/
│   │   │       └── [...oauth].ts
│   │   ├── index.tsx
│   │   └── _app.tsx
│   ├── styles/
│   │   └── globals.css
│   ├── utils/
│   │   ├── prisma.ts
│   │   └── supabaseClient.ts
│   └── validation/
│       └── schema.ts
├── .env
├── .gitignore
├── next.config.js
├── package.json
├── postcss.config.js
├── tailwind.config.js
└── tsconfig.json
```

---

## Setup and Installation

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/your-username/your-repo.git
   cd your-repo
   ```

2. **Install Dependencies:**

   ```bash
   npm install
   # or
   yarn install
   ```

3. **Configure Environment Variables:**

   Create a `.env` file in the root directory and populate it with the necessary variables:

   ```env
   # Prisma
   DATABASE_URL=your_supabase_database_url

   # OAuth
   OAUTH_CLIENT_ID=your_client_id
   OAUTH_CLIENT_SECRET=your_client_secret
   OAUTH_CALLBACK_URL=your_callback_url

   # Stripe
   STRIPE_SECRET_KEY=your_stripe_secret_key

   # Sanity
   SANITY_PROJECT_ID=your_sanity_project_id
   SANITY_DATASET=production

   # Cloudflare
   CLOUDFLARE_API_TOKEN=your_cloudflare_api_token
   ```

4. **Run Prisma Migrations:**

   ```bash
   npx prisma migrate dev --name init
   ```

5. **Start the Development Server:**

   ```bash
   npm run dev
   # or
   yarn dev
   ```

---

## Key Configuration Files

### `package.json`

```json:package.json
{
  "name": "my-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "prisma": "prisma"
  },
  "dependencies": {
    "@prisma/client": "^4.0.0",
    "@sanity/client": "^3.0.0",
    "next": "latest",
    "react": "latest",
    "react-dom": "latest",
    "react-hook-form": "^7.0.0",
    "zustand": "^4.0.0",
    "zod": "^3.0.0",
    "tailwindcss": "^3.0.0",
    "shadcn/ui": "^1.0.0",
    "stripe": "^10.0.0",
    "supabase": "^1.0.0"
    // Add other dependencies as needed
  },
  "devDependencies": {
    "typescript": "^4.0.0",
    "@types/react": "^17.0.0",
    "@types/node": "^16.0.0",
    "prisma": "^4.0.0",
    "autoprefixer": "^10.0.0",
    "postcss": "^8.0.0"
  }
}
```

### `tsconfig.json`

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

### `tailwind.config.js`

```javascript:tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./src/**/*.{js,ts,jsx,tsx}",
    "./components/**/*.{js,ts,jsx,tsx}",
    "./app/**/*.{js,ts,jsx,tsx}"
  ],
  theme: {
    extend: {},
  },
  plugins: [require("@shadcn/ui/plugin")],
};
```

### `next.config.js`

```javascript:next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  experimental: {
    serverComponents: true,
  },
  env: {
    DATABASE_URL: process.env.DATABASE_URL,
    OAUTH_CLIENT_ID: process.env.OAUTH_CLIENT_ID,
    OAUTH_CLIENT_SECRET: process.env.OAUTH_CLIENT_SECRET,
    OAUTH_CALLBACK_URL: process.env.OAUTH_CALLBACK_URL,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
    SANITY_PROJECT_ID: process.env.SANITY_PROJECT_ID,
    SANITY_DATASET: process.env.SANITY_DATASET,
  },
};

module.exports = nextConfig;
```

### `prisma/schema.prisma`

```prisma:prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  // Add other fields as necessary
}

model Product {
  id          Int      @id @default(autoincrement())
  name        String
  description String?
  price       Float
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  // Add other fields as necessary
}

```

---

## Core Implementations

### Global State Management with Zustand

```typescript:src/hooks/useStore.ts
import create from 'zustand';

interface AppState {
  user: User | null;
  setUser: (user: User | null) => void;
  // Add other state variables and setters as needed
}

export const useStore = create<AppState>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
}));
```

### Authentication with OAuth

```typescript:src/pages/api/auth/[...oauth].ts
import { NextApiRequest, NextApiResponse } from 'next';
import { handleOAuthCallback, handleOAuthLogin } from '../../../utils/auth';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { query, method } = req;

  if (method !== 'GET') {
    res.setHeader('Allow', ['GET']);
    return res.status(405).end(`Method ${method} Not Allowed`);
  }

  if (query.oauth === 'callback') {
    return handleOAuthCallback(req, res);
  } else {
    return handleOAuthLogin(req, res);
  }
}
```

```typescript:src/utils/auth.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { getSession } from 'next-auth/client';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export const handleOAuthLogin = (req: NextApiRequest, res: NextApiResponse) => {
  const redirectUrl = `https://your-oauth-provider.com/auth?client_id=${process.env.OAUTH_CLIENT_ID}&redirect_uri=${process.env.OAUTH_CALLBACK_URL}`;
  res.redirect(redirectUrl);
};

export const handleOAuthCallback = async (req: NextApiRequest, res: NextApiResponse) => {
  const { code } = req.query;

  // Exchange code for access token
  const tokenResponse = await fetch('https://your-oauth-provider.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      code,
      client_id: process.env.OAUTH_CLIENT_ID,
      client_secret: process.env.OAUTH_CLIENT_SECRET,
      redirect_uri: process.env.OAUTH_CALLBACK_URL,
      grant_type: 'authorization_code',
    }),
  });

  const tokenData = await tokenResponse.json();
  const accessToken = tokenData.access_token;

  // Fetch user info
  const userResponse = await fetch('https://your-oauth-provider.com/user', {
    headers: { Authorization: `Bearer ${accessToken}` },
  });

  const userData = await userResponse.json();

  // Upsert user in the database
  const user = await prisma.user.upsert({
    where: { email: userData.email },
    update: { name: userData.name },
    create: {
      email: userData.email,
      name: userData.name,
      // Add other fields as necessary
    },
  });

  // Set session or JWT as per your authentication strategy
  // For simplicity, redirect to homepage
  res.redirect('/');
};
```

### Stripe Integration

```typescript:src/pages/api/stripe/checkout.ts
import { NextApiRequest, NextApiResponse } from 'next';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2022-11-15',
});

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'POST') {
    try {
      const { items } = req.body;

      const session = await stripe.checkout.sessions.create({
        payment_method_types: ['card'],
        line_items: items.map((item: any) => ({
          price_data: {
            currency: 'usd',
            product_data: {
              name: item.name,
            },
            unit_amount: item.price * 100,
          },
          quantity: item.quantity,
        })),
        mode: 'payment',
        success_url: `${req.headers.origin}/success?session_id={CHECKOUT_SESSION_ID}`,
        cancel_url: `${req.headers.origin}/cancel`,
      });

      res.status(200).json({ sessionId: session.id });
    } catch (error: any) {
      res.status(500).json({ error: error.message });
    }
  } else {
    res.setHeader('Allow', 'POST');
    res.status(405).end('Method Not Allowed');
  }
}
```

### Form Validation with react-hook-form and Zod

```typescript:src/components/ui/LoginForm.tsx
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema } from '../../validation/schema';

type LoginFormInputs = {
  email: string;
  password: string;
};

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<LoginFormInputs>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = (data: LoginFormInputs) => {
    // Handle login
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label htmlFor="email" className="block text-sm font-medium text-gray-700">
          Email
        </label>
        <input
          id="email"
          type="email"
          {...register('email')}
          className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
        />
        {errors.email && <p className="text-red-500 text-sm">{errors.email.message}</p>}
      </div>
      
      <div>
        <label htmlFor="password" className="block text-sm font-medium text-gray-700">
          Password
        </label>
        <input
          id="password"
          type="password"
          {...register('password')}
          className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
        />
        {errors.password && <p className="text-red-500 text-sm">{errors.password.message}</p>}
      </div>

      <button type="submit" className="w-full bg-blue-600 text-white py-2 rounded-md">
        Login
      </button>
    </form>
  );
}

export default LoginForm;
```

```typescript:src/validation/schema.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string().email({ message: 'Invalid email address' }),
  password: z.string().min(6, { message: 'Password must be at least 6 characters' }),
});

// Add other schemas as needed
```

### Database Operations with Prisma and Supabase

```typescript:src/utils/prisma.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export default prisma;
```

```typescript:src/utils/supabaseClient.ts
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = 'https://your-supabase-url.supabase.co';
const supabaseKey = process.env.SUPABASE_KEY!;

export const supabase = createClient(supabaseUrl, supabaseKey);
```

```typescript:src/pages/api/products/index.ts
import { NextApiRequest, NextApiResponse } from 'next';
import prisma from '../../../utils/prisma';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') {
    const products = await prisma.product.findMany();
    res.status(200).json(products);
  } else if (req.method === 'POST') {
    const { name, description, price } = req.body;
    const product = await prisma.product.create({
      data: {
        name,
        description,
        price,
      },
    });
    res.status(201).json(product);
  } else {
    res.setHeader('Allow', ['GET', 'POST']);
    res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}
```

### CMS Integration with Sanity

```typescript:src/utils/sanityClient.ts
import sanityClient from '@sanity/client';

export const client = sanityClient({
  projectId: process.env.SANITY_PROJECT_ID!,
  dataset: process.env.SANITY_DATASET!,
  useCdn: true,
});
```

```typescript:src/pages/api/cms/content.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { client } from '../../../utils/sanityClient';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { query } = req;

  if (req.method === 'GET') {
    const data = await client.fetch(`*[_type == "post"]{title, body}`);
    res.status(200).json(data);
  } else {
    res.setHeader('Allow', ['GET']);
    res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}
```

### Styled Components with Shadcn

```typescript:src/components/ui/Button.tsx
import React from 'react';
import { cn } from '../../utils/cn';

type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> & {
  variant?: 'primary' | 'secondary';
};

function Button({ variant = 'primary', className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        'px-4 py-2 rounded-md font-semibold',
        variant === 'primary' ? 'bg-blue-600 text-white' : 'bg-gray-200 text-gray-800',
        className
      )}
      {...props}
    />
  );
}

export default Button;
```

```typescript:src/utils/cn.ts
export function cn(...classes: string[]) {
  return classes.filter(Boolean).join(' ');
}
```

---

## Deployment with Cloudflare VPS

To deploy your Next.js application on a **Cloudflare VPS**, follow these general steps:

1. **Set Up Your VPS:**

   - Choose a plan that suits your application's requirements.
   - Set up your server environment (e.g., install Node.js, npm/yarn, PostgreSQL if not using Supabase).

2. **Configure DNS:**

   - Point your domain to your Cloudflare VPS by updating the DNS records.

3. **Clone Your Repository:**

   ```bash
   git clone https://github.com/your-username/your-repo.git
   cd your-repo
   ```

4. **Install Dependencies:**

   ```bash
   npm install
   # or
   yarn install
   ```

5. **Set Environment Variables:**

   Ensure all necessary environment variables are set on your VPS. You can use tools like **dotenv** or set them directly in your shell configuration.

6. **Build the Application:**

   ```bash
   npm run build
   # or
   yarn build
   ```

7. **Start the Application:**

   It's recommended to use a process manager like **PM2**:

   ```bash
   npm install -g pm2
   pm2 start npm --name "my-app" -- start
   pm2 save
   ```

8. **Set Up a Reverse Proxy (Optional):**

   Use a web server like **Nginx** to handle requests and proxy them to your Next.js application.

   Example `nginx` configuration:

   ```nginx
   server {
       listen 80;
       server_name yourdomain.com;

       location / {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

9. **Secure Your Application:**

   - Obtain and install SSL certificates (Cloudflare can handle SSL termination).
   - Ensure firewall rules are in place.

10. **Monitor and Maintain:**

    - Use monitoring tools to keep track of your application's performance and uptime.
    - Regularly update dependencies and apply security patches.

---

## Conclusion

This template integrates a comprehensive stack of modern technologies to streamline the development process, ensuring scalability, maintainability, and a seamless developer experience. Customize and extend the components and configurations as per your project's specific requirements.

Feel free to reach out or create issues if you encounter any challenges or have suggestions for improvements!