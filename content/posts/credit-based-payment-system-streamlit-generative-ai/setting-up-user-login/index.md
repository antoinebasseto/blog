---
title: "Setting Up User Login"
date: 2025-03-07
summary: "User authentication is the first step in building any app that needs to track who's who — like one with user credits 👀 — and with Streamlit's new native authentication features, it's easier than ever!"
weight: 10
showComments: true
---

{{<lead>}}
User authentication is the first step in building any app that needs to track who's who — like one with user credits 👀 — and with Streamlit's new native authentication features, it's easier than ever!
{{</lead>}}

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

## Why User Authentication?

So, why bother with user authentication when all we want is a credit system? Well, every user needs their own balance. We need to be able to track who paid for what, and who's using how much, and for that we first need to know who's who. Luckily, Streamlit has made this process super easy with its native authentication features introduced in version 1.42. This means setting up secure logins will be a breeze!

## Prerequisites

Before we jump in, make sure you have the following ready:

- **Streamlit version >= 1.42.0:** The native authentication features are available from this version onward. Check your version with `streamlit --version`.
- **Authlib >= 1.3.2:** This library is required to use Streamlit's native authentication features. Install it with `pip install Authlib`.

Once you’ve got these sorted, you’re ready to set up authentication!

## Setting Up Native Authentication in Streamlit

Streamlit now offers a straightforward way to handle user logins. You can read more from the official docs [here](https://docs.streamlit.io/develop/concepts/connections/authentication), but we'll break down the process step by step.

### Configure Your Identity Provider

Identity providers handle the heavy lifting of user authentication and security — so you don't have to manage passwords or other sensitive details yourself. Streamlit supports any provider using OpenID Connect (OIDC), but in this tutorial we will use Google Identity. Why? Well, it's straightforward to set up and widely used. However, this does mean users will need a Google account to sign up for our app. If that’s a problem, you can explore [other providers](https://docs.streamlit.io/develop/concepts/connections/authentication#openid-connect), such as [Auth0](https://auth0.com/docs/get-started/auth0-overview/create-applications/regular-web-apps).

Let's walk through the process of setting up Google as your identity provider:

1. **Sign up for Google Cloud**

    First, you'll need a Google Cloud account. If you don't already have one, go to the [Google Cloud Console](https://console.cloud.google.com/) and sign up with your Google account.

2. **Create a New Project**
      - Look for the **project dropdown** near the top of the page
      - Click **"New Project"**
      - **Give your project a name** that's relevant to your Streamlit app
      - Click **"Create"**
      - **Wait for Google** to set up your new project (it might take a minute)
      - Make sure **your new project is selected** in the project dropdown

3. **Set Up OAuth Credentials**

    Now we need to create OAuth credentials that your Streamlit app will use to authenticate users:
      - In the left sidebar, navigate to **"APIs & Services > Credentials"**
      - Click the **"Create Credentials"** button at the top of the page
      - Select **"OAuth client ID"** from the dropdown menu
      - When prompted to configure the consent screen first, click **"Configure Consent Screen"** and fill in the required information:
        - **App name:** The name users will see when signing in
        - **User support email:** Your email address
        - Select **"External"** as the user type (unless you're building an internal app for a Google Workspace organization)
        - **Developer contact information:** Your email address
      - Under **"Google Auth Platform / Audience"**, go to "Test Users" and add the Google account you will use for testing.
      - Under **"Google Auth Platform / Data access"**, click "Add or remove scopes" and select the scope `.../auth/userinfo.email`, which means we will have access to our users' email address, then click "Update".
      - Now in the left sidebar, return to **"APIs & Services > Credentials"** and click "Create Credentials > OAuth client ID" again
        - For "Application type," select **"Web application"**
        - Fill in the required information:
          - **Name of your OAuth client**, I named mine `web_client_projectname`.
          - Under **"Authorized JavaScript origins"**, add `http://localhost:8501`
          - Under **"Authorized redirect URIs"**, add `http://localhost:8501/oauth2callback`
        {{<alert>}}Once we [deploy our app]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/deploying-your-app">}}), we will have to add our live domain to the authorized origins and redirect URIs.{{</alert>}}
        - Click **"Create Client"**
        - Copy paste your **client secret** and **client ID**, we will need them soon! You can find them again under "Google Auth Platform / Clients" and then clicking on the client you just created.

The consent screen is initially in "Testing" mode with restrictions on the number of users. For a production app, you would need to go through Google's verification process to publish it, but testing mode is sufficient for development or internal apps. We'll worry about that when we try to [deploy our app]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/deploying-your-app">}}).

### Configure Your Secrets

Now that we're all set up on Google's side, we need to let our Streamlit app know! To use native authentication, Streamlit expects some information in the `.streamlit/secrets.toml` file. Remember those client secret and client id? This is where we use them.

{{<alert>}} Make sure never to publish your secrets (e.g., avoid committing them to GitHub). Always include `secrets.toml` in your `.gitignore` file. Read more about secrets management [here](https://docs.streamlit.io/develop/concepts/connections/secrets-management). {{</alert>}}

Your `secrets.toml` file should look like this:

```toml
[auth]
redirect_uri = "http://localhost:8501/oauth2callback"
cookie_secret = "a-long-string-of-random-characters-you-generated" # You can use https://www.random.org/strings/

[auth.google]
client_id = "xxxx"
client_secret = "xxxx"
server_metadata_url = "https://accounts.google.com/.well-known/openid-configuration"
```

### Implement the Login Flow

Next, let’s implement the login flow in your Streamlit app. We’ll use Streamlit’s native auth functions: `st.login`, `st.logout`, and `st.experimental_user` to manage user sessions.

A simple example would be:

```python
import streamlit as st

with st.sidebar:
  if not st.experimental_user.is_logged_in:
      st.button("Log in", on_click=st.login, args=("google"))
      st.stop() # Stop further execution if user is not logged in.

  st.button("Log out", on_click=st.logout, args=("google"))

# Paid features come here (after st.stop())
```

## Testing Your Authentication Setup

After implementing the login functionality, it’s time to test it out. Run your app with `streamlit run your-app.py` and make sure you can log in, log out and `st.experimental_user` contains the expected information — you can try `st.write(st.experimental_user)`. Remember to log in with the Google account you set as a test account during set up.

## Next Steps

With user authentication in place, we can associate an email address with each user's credit balance. Next, we’ll look at how to [connect to a database]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/connecting-to-a-database">}}).

{{<alert "lightbulb">}}
Want to see an example? [SimplyPlot](https://simplyplot.streamlit.app/) uses a credit system for monetisation - it writes Matplotlib code for you, so you can stop troubleshooting your plots.
{{</alert>}}
