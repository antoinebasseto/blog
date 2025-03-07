---
title: "Integrating Stripe Checkouts"
date: 2025-03-06
summary: ""
weight: 30
---

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

## Why Do We Need Stripe?

### What About LemonSqueezy, Polar...?

## Requirements

add login + database

`pip install stripe`

test mode vs. production mode as an alert

## Integrate Stripe Checkouts

### Configuring Your Stripe Account

- "Settings > Payment methods", deactivate buy now pay later
- "Setting > Business > Branding > Checkout & Payment links", on the left add icon and logo, change your brand colors

### Calling The API

secrets.toml

```toml
API_KEY = "xxxx"
SUCCESS_URL = "http://localhost:8501"
```

## Testing Your Stripe Setup

test mode vs. production mode

## The Hard Part, Stripe Webhooks

### What Are Stripe Webhooks, Why Do They Exist?

### Going Serverless - Choosing A Provider

### Writing The Webhook Handler

- Install the supabase CLI
- Create a new function with `supabase functions new stripe-webhook`
- Use the following code for the function

    ```typescript
    // Import via bare specifier thanks to the import_map.json file.
    import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
    import { createClient } from 'jsr:@supabase/supabase-js@2'

    const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
        // This is needed to use the Fetch API rather than relying on the Node http
        // package.
        apiVersion: '2024-11-20'
    })
    // This is needed in order to use the Web Crypto API in Deno.
    const cryptoProvider = Stripe.createSubtleCryptoProvider()

    Deno.serve(async (request) => {
        const signature = request.headers.get('Stripe-Signature')

        // First step is to verify the event. The .text() method must be used as the
        // verification relies on the raw request body rather than the parsed JSON.
        const body = await request.text()
        let receivedEvent
        try {
            receivedEvent = await stripe.webhooks.constructEventAsync(
            body,
            signature!,
            Deno.env.get('STRIPE_WEBHOOK_SIGNING_SECRET')!,
            undefined,
            cryptoProvider
            )
        } catch (err) {
            if (err instanceof Error) {
            return new Response(err.message, { status: 400 })
            }
            return new Response('Unknown error occurred', { status: 400 })
        }

        // Once the event has been verified, we can handle it based on its type.
        if (receivedEvent.type === 'checkout.session.completed') {
            const session = receivedEvent.data.object;
            const userEmail = session.customer_email;
            const amount = session.amount_total;

            const supabase = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_ANON_KEY')!)

            try {
            // First, sign in with the correct credentials to see the data
            const { error: loginError } = await supabase.auth.signInWithPassword({
                email: Deno.env.get('EMAIL')!,
                password: Deno.env.get('PASSWORD')!,
            });
            
            if (loginError) {
                throw loginError;
            }

            // Then, get the current credits of the user
            const { data: userData, error: fetchError } = await supabase
                .from('users')
                .select('credits')
                .eq('email', userEmail)
                .single();

            if (fetchError) {
                throw fetchError;
            }

            const currentCredits = userData?.credits || 0;
            const newCredits = currentCredits + amount;

            // Finally, update with the new total
            const { error: updateError } = await supabase
                .from('users')
                .update({ credits: newCredits })
                .eq('email', userEmail);

            if (updateError) {
                throw updateError;
            }

            console.log(`Updated credits for user ${userEmail} from ${currentCredits} to ${newCredits}`);
            } catch (error) {
            console.error(`Error updating user credits for user ${userEmail}:`, error);
            return new Response('Failed to update user credits', { status: 500 });
            }
        }
        
        return new Response(JSON.stringify({ ok: true }), { status: 200 })
    });
    ```

Explain code

- Install Docker
- Run `supabase functions deploy --no-verify-jwt stripe-webhook`, choose the correct project
- Go to Supabase, copy the endpoint URL

- Go to Stripe
- In the left sidebar, click "Developpers > Webhooks" at the bottom.
  - Click "Add destination"
  - Keep as is, for event select `checkout.session.completed`
  - Select `webhook endpoint`
  - Paste the endpoint URL
  - Click "Create destination"
- Copy the "Signing secret" on the right-hand side
- Create a `.env` file in `functions/stripe-webhook`, MAKE SURE TO ADD IT TO GITIGNORE
- Add your stripe api key and webhook signing secret:

```.env
STRIPE_API_KEY="xxxx"
STRIPE_WEBHOOK_SIGNING_SECRET="xxxx"
EMAIL="xxxx"
PASSWORD="xxxx"
```

- Give supabase your secrets `supabase secrets set --env-file ./supabase/functions/stripe-webhook/.env`

### Connecting To The Database

## Testing Your Payment Workflow

## Next Steps
