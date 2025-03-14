---
title: "Integrating Stripe Checkouts"
date: 2025-03-14
summary: "Ready to start charging for your app? In this post we will learn how to set up a pay-per-use system — from creating Stripe Checkout sessions to listening to webhooks."
weight: 30
showComments: true
---

{{<lead>}}
Ready to start charging for your app? In this post we will learn how to set up a pay-per-use system — from creating Stripe Checkout sessions to listening to webhooks.
{{</lead>}}

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

## Why Do We Need Stripe?

If you want to charge users for your app, you need a payment processor — a service that handles credit card transactions, securely moves money, and ensures compliance with financial regulations. Stripe has become the default choice for many developers because it’s easy to integrate and widely supported. That’s why we’re using it here.

### What About LemonSqueezy, Polar...?

Some other payment processors, such as [LemonSqueezy](https://www.lemonsqueezy.com) and [Polar](https://polar.sh), work differently — they act as Merchants of Record (MoR). This means they don’t just process payments; they also handle taxes, chargebacks, and compliance for you (in exchange for a bigger cut of your profits).

With Stripe, you're responsible for these things yourself. If you want full control over your payments and customer relationships, Stripe is a good option. However, if dealing with VAT, sales tax, and regulations seems overwhelming, an MoR might be a better fit.

## Requirements

Before we get started, make sure you have:

- **User Authentication and a Database:** If you haven't done this yet, follow the first part of this blog for [authentication]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/setting-up-user-login">}}) and the second for [databases]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/connecting-to-a-database">}}).
- **Stripe’s Python SDK:** Install it with `pip install stripe`.

## Integrate Stripe Checkouts

Stripe operates in two modes:

- **Test Mode / Sandboxes:** Simulate transactions without real money, perfect for development and testing. Stripe recently transitioned from traditional test environments to "sandboxes," allowing you to start testing without entering business details until you're ready to go live.
- **Live Mode:** Handle real transactions with actual funds.

Each mode comes with its own API keys, keeping testing data separate from live transactions.

### Configuring Your Stripe Account

In the **"Settings"** menu, go to **"Business > Branding > Checkout & Payment Links"**. On the left, you can add your icon and logo, and adjust your brand colors.

### Calling The API

On the left sidebar, select "**Home**", which will display your API Keys on the right hand side (or search for "API keys" in the search bar). Click to reveal your secret key and copy it to your `.streamlit/secrets.toml` file.

{{<alert>}} Make sure you never publish your secrets, do not commit them to GitHub! Always include `secrets.toml` in your `.gitignore` file. Read more about secrets management [here](https://docs.streamlit.io/develop/concepts/connections/secrets-management). {{</alert>}}

Add the following section to your `secrets.toml` file:

```toml
[stripe]
API_KEY = "xxxx"
SUCCESS_URL = "http://localhost:8501"
```

## Testing Your Stripe Setup

To test your Stripe setup, you can use the following Python code to create a checkout session that allows users to purchase credits. This script integrates with Stripe's API to create a session where users can complete their payment.

```python
def create_checkout_session(amount: float, user_email: str) -> str:
    """
    Create a Stripe checkout session for purchasing credits.

    Args:
        amount: Dollar amount to charge (e.g., 5.00 means $5)
        user_email: Email of the current user

    Returns:
        str: URL for the checkout session
    """
    # Set the Stripe API key using the secrets stored in Streamlit
    stripe.api_key = st.secrets["stripe"]["API_KEY"]

    # Create a checkout session with the specified parameters
    checkout_session = stripe.checkout.Session.create(
        customer_email=user_email, # Associate the session with the user's email
        line_items=[ # Define the items being purchased
            {
                "price_data": {
                    "currency": "usd", # Currency for the payment
                    "product_data": {
                        "name": "Credits Purchase", # Name of the item being purchased
                    },
                    "unit_amount": int(amount * 100),  # Convert the amount to cents (Stripe expects cents)
                },
                "quantity": 1,
            }
        ],
        mode="payment", # This sets the checkout mode to a one-time payment
        success_url=st.secrets["stripe"]["SUCCESS_URL"], # URL to redirect to after a successful payment
    )

    # Return the URL where the user can complete their payment
    return checkout_session.url
```

This function takes an amount (in dollars) and the user's email to generate a Stripe checkout session. After creating the session, it returns a URL where the user can finalize the payment.

You can use this function in your app like this:

```python
# Allow users to choose the amount of credits they want to purchase
amount = st.number_input(
    "Amount to add", # Label for the input field
    min_value=1.00, # Minimum purchase amount
    max_value=10.00, # Maximum purchase amount
    value=3.00, # Default value
    step=0.5, # Step value for each increment (using the +/- buttons)
    format="%.2f", # Format the amount as currency
    help="Enter an amount between \\$1 and \\$10", # Help text for users
)

# Generate the Stripe checkout session URL using the amount and user's email
checkout_url = create_checkout_session(amount, st.experimental_user.email)

# Provide a link for the user to start the checkout process
st.link_button("Buy now", checkout_url)
```

This example allows users to enter the amount they wish to purchase (between $1 and $10). Once they select the amount, they can click the "Buy now" button, which directs them to the Stripe checkout page to complete their purchase.

You can test the checkout process using [Stripe's test cards](https://docs.stripe.com/testing#cards). These test cards simulate various payment scenarios, allowing you to ensure everything works correctly before going live.

## The Hard Part, Stripe Webhooks

### What Are Stripe Webhooks, Why Do They Exist?

Stripe webhooks are user-defined HTTP callbacks that allow your application to receive real-time notifications about events occurring in your Stripe account. When specific actions happen — such as a successful payment — Stripe sends an HTTP POST request to a URL you configure, containing details about the event.

But why? The reason webhooks exist is to make sure your app stays in sync with Stripe. For example, after a user completes a payment, you might want to update their account or grant credits. But if you only rely on the success page URL to handle this, there’s a chance the user could close the browser or lose internet connection before your app completes the action.

Webhooks let you handle payment events directly on your server, regardless of whether the user is still on the success page or not. This ensures your system processes payments accurately and consistently.

### Going Serverless

So, we need to listen to webhooks. That means we need something running on a server. But here’s the catch: if we’re deploying on community cloud ([yes we will]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/deploying-your-app">}})), we can’t just run server code.

Here are two options:

- **Deploy elsewhere** – Sure, you could set up a server with another provider to handle webhooks. But that brings its own set of headaches: managing servers, scaling, security, costs... you get it.
- **Use a serverless function** – This is where serverless comes in. With a serverless function, you don’t need to manage servers at all. You just write the function, deploy it, and it runs when triggered (like when Stripe sends a webhook). It scales automatically, and you only pay for when it’s running. Super efficient!

Okay, but what’s a serverless function exactly? Well, it's a piece of code you're giving to a cloud provider so you can run code without worrying about the infrastructure. You don’t have to set up or maintain a server. The cloud provider does all that work for you — they run your function, scale it when needed, and only charge you for the time your code runs.

{{<alert "circle-info">}}
Serverless functions are only serverless in the sense that **you don't have to deal with the server**, but it does exist!
{{</alert>}}

#### Choosing a Provider

Now, which provider to choose? There are a lot of options: **AWS Lambda**, **Google Cloud Functions**, and others. AWS Lambda is popular, but it can be a bit much for simple use cases.

But what if I told you that you already have a provider? **Supabase**! Supabase offers _edge functions_. Don't worry! It's just a fancy name for a special kind of serverless functions, that live on servers all over the world to be closer to the users (on the _edge_ of the network) and enable faster processing times. By using Supabase, we can easily integrate our code with our existing database. No need to jump between different services — it’s all in one place.

### Writing The Webhook Handler

So, we can run code on the cloud, but now we need the code! This part is an adapted version of [Supabase's official tutorial](https://supabase.com/docs/guides/functions/examples/stripe-webhooks), go check it out if you want. First, let's set up our supabase edge function:

- [Install the Supabase CLI](https://supabase.com/docs/guides/local-development/cli/getting-started?queryGroups=platform&platform=macos#installing-the-supabase-cli)
- Move to your project's directory and **create a new edge function** with `supabase functions new stripe-webhook`

This will create a `supabase` folder in your project, inside of which you will find `functions/stripe-webhook/index.ts`. This is where we will write the code we will put on the _edge_.

First of all, we need to talk to Stripe and Supabase, so we **import their libraries.**

```typescript
// Import Stripe and Supabase's createClient function.
import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
import { createClient } from 'jsr:@supabase/supabase-js@2'
```

Here, we’re pulling in the Stripe library (from a CDN) and Supabase’s client. These imports let us call Stripe’s API and interact with our Supabase database later on.

Then, we need to **initialize Stripe with our API key.**

<!-- markdownlint-disable MD034 MD046 -->
{{<highlight typescript "linenos=table,hl_lines= 5-7">}}
// Import Stripe and Supabase's createClient function.
import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
import { createClient } from 'jsr:@supabase/supabase-js@2'

const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
  apiVersion: '2024-11-20'
});
{{</highlight>}}

Here, we create a `stripe` instance using our API key (stored in an environment variable, which we will discuss later) and specify the API version.

Now, we want to **set up a server** that listens for incoming webhooks from Stripe.

{{<highlight typescript "linenos=table,hl_lines= 9-11">}}
// Import Stripe and Supabase's createClient function.
import Stripe from '<https://esm.sh/stripe@14.25.0?target=denonext>'
import { createClient } from 'jsr:@supabase/supabase-js@2'

const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
  apiVersion: '2024-11-20'
});

Deno.serve(async (request) => {
  // Our webhook handling code will go here.
});
{{</highlight>}}

Using Deno’s built-in serve function, we set up an HTTP server. Every time Stripe sends a webhook, this function gets triggered with the incoming request.

Before processing the webhook, we need to **confirm that the request is really from Stripe.**

{{<highlight typescript "linenos=table,hl_lines= 8 11-29">}}
// Import Stripe and Supabase's createClient function.
import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
import { createClient } from 'jsr:@supabase/supabase-js@2'

const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
  apiVersion: '2024-11-20'
});
const cryptoProvider = Stripe.createSubtleCryptoProvider();

Deno.serve(async (request) => {
  // First step is to verify the event. The .text() method must be used as the
  // verification relies on the raw request body rather than the parsed JSON.
  const signature = request.headers.get('Stripe-Signature')
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
});
{{</highlight>}}

In this code:

- We extract the Stripe-Signature from the request headers.
- We read the raw request body (using .text()) because verification needs the unaltered data.
- We call constructEventAsync with our body, signature, webhook secret (from env variables), and crypto provider.
- If verification fails, we catch the error and return a 400 response.

Now that we’ve verified the webhook, we only want to **act on events indicating a successful checkout.**

{{<highlight typescript "linenos=table,hl_lines= 31-37">}}
// Import Stripe and Supabase's createClient function.
import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
import { createClient } from 'jsr:@supabase/supabase-js@2'

const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
  apiVersion: '2024-11-20'
});
const cryptoProvider = Stripe.createSubtleCryptoProvider();

Deno.serve(async (request) => {
  // First step is to verify the event. The .text() method must be used as the
  // verification relies on the raw request body rather than the parsed JSON.
  const signature = request.headers.get('Stripe-Signature')
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
    /// Continue processing the payment
  }
});
{{</highlight>}}

Here, we check if the event type is `checkout.session.completed`. If it is, we extract important details: the customer's email and the payment amount. We’ll use these details to update the user's credits.

We need to connect to our Supabase database so we can update the user's credits, but first **we must sign in.**

{{<highlight typescript "linenos=table,hl_lines= 37-54">}}
// Import Stripe and Supabase's createClient function.
import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
import { createClient } from 'jsr:@supabase/supabase-js@2'

const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
  apiVersion: '2024-11-20'
});
const cryptoProvider = Stripe.createSubtleCryptoProvider();

Deno.serve(async (request) => {
  // First step is to verify the event. The .text() method must be used as the
  // verification relies on the raw request body rather than the parsed JSON.
  const signature = request.headers.get('Stripe-Signature')
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

      // Continue with fetching and updating user data...
    } catch (error) {
      console.error(`Error updating user credits for user ${userEmail}:`, error);
      return new Response('Failed to update user credits', { status: 500 });
    }
  }
});
{{</highlight>}}

- We create a Supabase client using our URL and anon key.
- Then, we sign in using our service account’s credentials (stored in our environment).
- If the sign-in fails, we log the error and return a 500 response, halting further processing.

Now, we need to **fetch the user’s current credits, add the amount from the Stripe session, and update the record.**

{{<highlight typescript "linenos=table,hl_lines= 50-73">}}
// Import Stripe and Supabase's createClient function.
import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
import { createClient } from 'jsr:@supabase/supabase-js@2'

const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
  apiVersion: '2024-11-20'
});
const cryptoProvider = Stripe.createSubtleCryptoProvider();

Deno.serve(async (request) => {
  // First step is to verify the event. The .text() method must be used as the
  // verification relies on the raw request body rather than the parsed JSON.
  const signature = request.headers.get('Stripe-Signature')
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
});
{{</highlight>}}

- We query the users table to fetch the current credit balance for the user.
- We calculate the new total by adding the amount from the payment.
- We update the user’s record with the new credit balance.
- A log message confirms the update, which is useful for tracking and debugging.

After processing the webhook, we need to **let Stripe know everything went well.**

{{<highlight typescript "linenos=table,hl_lines= 80">}}
// Import Stripe and Supabase's createClient function.
import Stripe from 'https://esm.sh/stripe@14.25.0?target=denonext'
import { createClient } from 'jsr:@supabase/supabase-js@2'

const stripe = new Stripe(Deno.env.get('STRIPE_API_KEY') as string, {
  apiVersion: '2024-11-20'
});
const cryptoProvider = Stripe.createSubtleCryptoProvider();

Deno.serve(async (request) => {
  // First step is to verify the event. The .text() method must be used as the
  // verification relies on the raw request body rather than the parsed JSON.
  const signature = request.headers.get('Stripe-Signature')
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
{{</highlight>}}

If everything is processed without errors, we return a JSON response with `{ ok: true }` and a `200` status code. This tells Stripe that we’ve handled the webhook successfully.

### Deploying The Webhook Handler

1. **Install Docker**
   - Supabase Edge Functions run in a containerized environment, so you’ll need **Docker** installed on your machine.
   - Download and follow the instructions from their [website](https://www.docker.com).
2. **Deploy the Webhook Function**
   - Do this by running `supabase functions deploy --no-verify-jwt stripe-webhook`.
   - When prompted, select the correct Supabase project.
   - The `--no-verify-jwt` flag ensures that the function runs without authentication, since Stripe doesn’t send a JWT token with its webhooks.
3. **Get the Endpoint URL**
   - Go to your Supabase dashbaord and select your project.
   - In the left sidebar click "Edge Functions".
   - Here you should see the `stripe-webhook` function we just deployed, and an accompanying `URL` field, copy its contents.
4. **Register the Webhook in Stripe**
   - Go to your Stripe Dashboard.
   - In the left sidebar, down at the bottom, click "Developers > Webhooks".
   - Click "Add destination".
   - Under "Events," select `checkout.session.completed`, click "Continue".
   - For "Destination types," select "Webhook endpoint", click "Continue".
   - Under "Endpoint URL," paste the previously copied URL from Supabase and click "Create destination".
5. **Setup the Environment Variables**
   - Still in Stripe, get the "Signing secret" visible on the right-hand side when you select the event destination we just set up.
   - Create a `.env` file in `functions/stripe-webhook` that should look like this:

    ```.toml
    STRIPE_API_KEY="xxxx"
    STRIPE_WEBHOOK_SIGNING_SECRET="xxxx"
    # The email and password combo of the Supabase user you set up previously.
    # If you followed the previous guide, they should be present in your .streamlit/secrets.toml file
    EMAIL="xxxx"
    PASSWORD="xxxx"
    ```

    {{<alert>}}Add this file to `.gitignore` to prevent exposing secrets in your repo!{{</alert>}}

    - Send this file to Supabase by running `supabase secrets set --env-file ./supabase/functions/stripe-webhook/.env`

## Testing Your Payment Workflow

Alright, time to make sure everything is working! We’ll check if the user’s credit balance updates after a successful payment.

First, let’s get the user’s current balance from our database:

```python
def get_user_credits(conn: SupabaseConnection) -> int:
    """
    Get the current user's credit balance.

    Args:
        conn: Supabase connection

    Returns:
        int: User's credit balance in cents
    """
    return execute_query(
        conn.table("users").select("credits").eq("email", st.experimental_user.email),
        ttl=None,
    ).data[0]["credits"]
```

Now, let's show the user their current balance in a nice, simple UI:

```python
st.metric(
    "Credit Balance",
    f"${user_credits / 100:.2f}",  # Convert cents to dollars
    help="Your credit balance will be consumed as you use the app.",
)
```

This displays the balance in dollars instead of cents, making it more readable.

Finally, run a test payment using one of Stripe's [test cards](https://docs.stripe.com/testing#cards), complete the checkout session and you should be automatically redirected to your Streamlit app, with the updated balance showing up!

If it doesn’t update, check your Supabase logs and Stripe dashboard for any webhook errors.

## Next Steps

Now that payments are working and credits update correctly, it's time to put them to use! Next, we'll [connect to an LLM API]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/calling-llm-apis">}}) and track usage, making sure credits are deducted as the user interacts with the model.

{{<alert "lightbulb">}}
Want to see an example? [FastPrompt](https://fastprompt.streamlit.app/) uses a credit system for monetisation - it generates high quality LLM system prompts from a simple description of inputs outputs. You can refine it with AI suggestions and iterate quickly.
{{</alert>}}

**Get updates & insights!**
{{<rawhtml>}}
<form method="post" action="https://ymlp.com/subscribe.php?id=gybbemegmgj" class="subscribe-form">
  <span>
    <input type="email" id="email" name="YMP0" placeholder="Enter your email" required style="display: inline-block;
             border-radius: 0.375rem;
             background-color:rgb(255, 255, 255);
             padding: .25rem .25rem .25rem .5rem;
             box-shadow: inset 0 0 0 3px rgb(200,200,200);">
  </span>
  <span>
    <button type="submit" class="submit-btn">{{<button type="submit">}}Subscribe{{</button>}}</button>
  </span>
</form>
{{</rawhtml>}}