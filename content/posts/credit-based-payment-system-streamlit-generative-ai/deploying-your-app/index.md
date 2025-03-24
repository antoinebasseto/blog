---
title: "Deploying Your App"
date: 2025-03-24
summary: "You’ve built something awesome — now it’s time to share it with the world! Learn the ins and outs of deploying your Streamlit app on Streamlit Cloud, it's (almost) as simple as a single click."
weight: 60
showComments: true
---

{{<lead>}}
You’ve built something awesome — now it’s time to share it with the world! Learn the ins and outs of deploying your Streamlit app on Streamlit Cloud, it's (almost) as simple as a single click.
{{</lead>}}

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

## Why Do We Need To Deploy Our App?

If your app isn’t online, no one else can use it. Deployment makes it accessible with a simple link, runs it on a reliable server, and lets multiple users access it without issues.

## Which Deployment Option Should I Choose?

There are many ways to deploy your app, from cloud platforms like AWS and Google Cloud to services like Heroku and Vercel. But if you're building with Streamlit, the easiest option is **Streamlit Cloud** — it's free, designed specifically for Streamlit apps, and takes just a few clicks to set up.

With Streamlit Cloud, you don’t need to worry about server management, scaling, or complicated configurations. Just connect your GitHub repo, set up dependencies, and deploy. Simple as that.

## Requirements

Before deploying, your app should be fully functional and ready for users. It needs a [login system]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/setting-up-user-login">}}), a [database]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/connecting-to-a-database">}}) to store user data and credits, and a way for users to [purchase credits]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/integrating-stripe-checkouts">}}). Your app should also be able to track and [deduct credits]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/calling-llm-apis">}}) as they are used. Once everything works locally, you’re ready to deploy.

## Deploying Your App

First, set up a Git repository for your project. You can keep it private, but make sure not to push any sensitive information like API keys.

Next, create a requirements.txt file by running:

```bash
pip freeze > requirements.txt
```

This tells Streamlit Cloud which dependencies your app needs. Push this file to your repository along with the rest of your code. Once that’s done, you're ready to deploy:

1. **Deploy to Streamlit Cloud**
   - Go to [Streamlit](https://streamlit.io) and log in.
   - Click **"Create app"** in the top right corner.
   - Select **"Deploy a public app from GitHub"**.
   - Enter your repository name, the branch you want to deploy (use `main` if you haven’t created any other branches), and the main entry point (the Python file you used with `streamlit run`).
   - Choose a valid and available app URL. Make sure to note it down, as you’ll need it soon.
2. **Configure Google Cloud OAuth**
   - Head to [Google Cloud Platform](https://console.cloud.google.com/) and ensure the correct project is selected.
   - In the left sidebar, go to **"APIs & Services > OAuth Consent Screen"**.
   - Under **Audience**, click **"Publish App"**.
   - Go to **Clients**, select your client, and add your app’s URL to **Authorised JavaScript origins**, and the same URL with `/oauth2callback` added at the end to the **Authorized URIs** (just as we had done for `localhost` when we set up [user login]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/setting-up-user-login">}})).
3. **Set Up Stripe**
   - In your Stripe dashboard, make sure you're in live mode and the correct project is selected.
   - Go to **"Developers > API Keys"** and copy your `secret key`.
   - Set up an endpoint for your webhooks as we did in [the previous article]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/integrating-stripe-checkouts">}}).
   - Make sure to save the webhook signing secret for later.
4. **Update Supabase**
   - Update the webhook signing secret in your Supabase `.env`.
   - Serve the new `.env` file to Supabase to reflect the latest changes, by running `supabase secrets set --env-file ./supabase/functions/stripe-webhook/.env`.
5. **Update Your `secrets.toml` File**
   - Update your auth `redirect_uri`.
   - Update your Stripe `api_key` and `success_url`.
6. **Upload Secrets to Streamlit Cloud**
   - Go back to [Streamlit](https://streamlit.io) and log in again.
   - Select your app, click **"Settings > Secrets"**, and paste your updated `secrets.toml` file. It’s safe there!

## Testing Your Deployment

And that's it, your app is ready to be used! Just click the link in Streamlit Cloud or enter the URL, share it around and let people have a go at what you made! You can make sure that when people sign up, your database gets updated correctly, make sure that credits are being consummed when your app is used, and try to buy some to test your Stripe integration.

{{<alert "circle-info">}}
If you deployed your app from the `main` branch, avoid pushing untested changes directly to production. Use branches for development and only merge changes when they are fully tested.
{{</alert>}}

## That's It

Now that you deployed your app, your next step would be to tell everyone about the great thing you made!

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
