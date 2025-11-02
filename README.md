# Paystack Cloudflare SaaS Kit

A production-ready SaaS starter kit built on Cloudflare's edge infrastructure, combining TanStack, Better Auth, and Paystack for a complete full-stack application.

## Overview

This starter kit provides a powerful foundation for building scalable SaaS applications with modern tools and best practices. It leverages Cloudflare Workers/Pages for deployment, TanStack for routing and data management, Better Auth for authentication, and Paystack for payment processing.

**Stack includes:**
- **Frontend:** React + Vite, TailwindCSS, ShadcnUI
- **Routing/Data:** TanStack Router/Query/Form
- **Auth:** Better Auth (Google OAuth, Email/Password, plugin extensibility)
- **Database:** Drizzle ORM + Cloudflare D1 (SQL)
- **Deployment:** Cloudflare Workers/Pages, Wrangler CLI
- **Payments:** Paystack for payment and subscription management

## Features

- **Cloudflare-first:** Auto deployment scripts and resource provisioning, optimized for Workers/Pages
- **Database:** Type-safe Drizzle ORM with support for migrations; instant D1 setup
- **Authentication:** Secure multi-provider auth (OAuth, email/password); session, geolocation, and IP tracking by default
- **Payments:** Paystack integration for managing payments, subscriptions, and billing
- **Developer Experience:** Vite hot reload, modern package managers, and linting
- **Modern UI:** Component libraries with TailwindCSS and ShadcnUI
- **Extensible:** Plugins for advanced auth, legal consent, email normalization, and more

## Getting Started

### 1. Clone and Install

```sh
git clone https://github.com/alexasomba/paystack-cloudflare-saas-kit
cd paystack-cloudflare-saas-kit
npm install    # or pnpm install / bun install
```

### 2. Set Up Environment

Create a `.dev.vars` or `.env` file for your secrets:

```env
BETTER_AUTH_URL=your_auth_url
BETTER_AUTH_SECRET=your_auth_secret
DATABASE_URL=your_database_url
PAYSTACK_SECRET_KEY=your_paystack_secret_key
PAYSTACK_PUBLIC_KEY=your_paystack_public_key
```

Upload environment variables to Cloudflare:

```sh
npx wrangler pages secret bulk .env
```

### 3. Configure Database and Auth

**Drizzle ORM Setup:**

Define your schema in `src/db/schema.ts` and run migrations:

```sh
npm run db:migrate
```

**Better Auth Configuration:**

```ts
import { betterAuth } from 'better-auth';

export const auth = betterAuth({
  database: {
    // your database config
  },
  socialProviders: {
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    },
  },
  // additional auth options
});
```

### 4. Payment Integration (Paystack)

Configure Paystack in your application:

```ts
// Using the Paystack API directly with fetch
const PAYSTACK_SECRET_KEY = process.env.PAYSTACK_SECRET_KEY;

// Initialize payment
const initializePayment = async (email, amount) => {
  const response = await fetch('https://api.paystack.co/transaction/initialize', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${PAYSTACK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      email,
      amount: amount * 100, // amount in kobo
      callback_url: 'https://yourapp.com/payment/callback',
    }),
  });
  return response.json();
};

// Verify payment
const verifyPayment = async (reference) => {
  const response = await fetch(`https://api.paystack.co/transaction/verify/${reference}`, {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${PAYSTACK_SECRET_KEY}`,
    },
  });
  return response.json();
};
```

For subscriptions:

```ts
// Create subscription plan
const createPlan = async (name, amount, interval) => {
  const response = await fetch('https://api.paystack.co/plan', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${PAYSTACK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      name,
      amount: amount * 100,
      interval, // 'daily', 'weekly', 'monthly', 'yearly'
    }),
  });
  return response.json();
};

// Subscribe customer
const subscribeCustomer = async (customer, plan) => {
  const response = await fetch('https://api.paystack.co/subscription', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${PAYSTACK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      customer,
      plan,
    }),
  });
  return response.json();
};
```

### 5. Development and Deployment

Start local dev server:

```sh
npm run dev
```

Deploy to Cloudflare:

```sh
npx wrangler pages deploy
```

Preview deployment:

```sh
npx wrangler pages preview
```

## Usage Tips

- **Session Management:** Better Auth handles session management with secure cookies and tokens
- **Cloudflare Resources:** Ensure your Cloudflare resources (D1, KV, R2) are properly provisioned
- **Paystack Webhooks:** Configure webhook endpoints to handle payment events:
  - `/api/webhooks/paystack` for payment notifications
  - Verify webhook signatures using your Paystack secret key
- **Environment Variables:** Never commit secrets to version control; use Cloudflare secrets management

## Paystack Integration Guide

### Payment Flow

1. **Initialize Payment:** Create a payment transaction with Paystack
2. **Redirect User:** Send user to Paystack's payment page
3. **Handle Callback:** Process the payment callback from Paystack
4. **Verify Transaction:** Always verify transactions on your backend
5. **Update Database:** Update user's payment status in your database

### Webhook Handling

```ts
import crypto from 'crypto';

const verifyPaystackWebhook = (payload, signature) => {
  const hash = crypto
    .createHmac('sha512', process.env.PAYSTACK_SECRET_KEY)
    .update(JSON.stringify(payload))
    .digest('hex');
  return hash === signature;
};

// Webhook endpoint (Cloudflare Workers example)
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === 'POST' && new URL(request.url).pathname === '/api/webhooks/paystack') {
      const signature = request.headers.get('x-paystack-signature');
      const body = await request.json();
      
      if (!verifyPaystackWebhook(body, signature)) {
        return new Response('Invalid signature', { status: 400 });
      }
      
      const event = body;
      
      switch (event.event) {
        case 'charge.success':
          // Handle successful payment
          break;
        case 'subscription.create':
          // Handle new subscription
          break;
        case 'subscription.disable':
          // Handle cancelled subscription
          break;
      }
      
      return new Response('OK', { status: 200 });
    }
    
    return new Response('Not Found', { status: 404 });
  }
};
```

## Troubleshooting

- **Auth Issues:** Check environment variables and ensure callback URLs are properly configured
- **Database Migrations:** All schemas, including Better Auth's, must be merged for seamless Drizzle migrations
- **Paystack Webhooks:** Ensure webhook URLs are publicly accessible and verify signatures
- **Cloudflare Routing:** Check that API endpoints are reachable and not blocked by Cloudflare routing rules

## Resources

- [Paystack Documentation](https://paystack.com/docs)
- [Better Auth Documentation](https://www.better-auth.com)
- [TanStack Documentation](https://tanstack.com)
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers)
- [Drizzle ORM Documentation](https://orm.drizzle.team)

## License

MIT

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.