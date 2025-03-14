---
title: "Deploying Your App"
date: 
summary: "You’ve built something awesome — now it’s time to share it with the world! Learn the ins and outs of deploying your Streamlit app on Streamlit Cloud, it's (almost) as simple as a single click."
weight: 60
draft: true
---

{{<badge>}}
Draft
{{</badge>}}

{{<lead>}}
You’ve built something awesome — now it’s time to share it with the world! Learn the ins and outs of deploying your Streamlit app on Streamlit Cloud, it's (almost) as simple as a single click.
{{</lead>}}

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

<!-- Series blurb -->
<!-- 
## Why Do We Need To Deploy Our App?

## Which Deployment Option Should I Choose?

streamlit cloud

## Requirements

`pip freeze > requirements.txt`

## Preparing Your App For Deployment

Setup git repo
`pip freeze > requirements.txt`

### Configuring Your Deployment Environment

Need to change:

- Google auth
  - status
  - authorized uris

- Deploy to streamlit, check domain is available, copy URL

- Go to google cloud platform
- Left sidebar, click "APIs and Service > OAuth Consent Screen"
- Click "Audience"
- Click "Publish App"
- Go to "Clients", click on your client
- Add your app's URL to the authorized thingies

- Stripe
  - api keys
  - redirect url
  - create restricted key
  - create webhook
- Supabase nothing!
- `secrets.toml`
  - auth `redirect_uri`
  - stripe `api_key` and `success_url`

update webhook signing secret
serve new .env file to supabase

### Deploying Your App

- build `requirements.txt`
- git push everything
- click deploy
- copy paste `secrets.toml`

## Testing Your Deployment

## Next Steps -->
