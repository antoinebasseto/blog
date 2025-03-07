---
title: "Connecting To A Database"
date: 2025-03-03
summary: "Need to store user credits securely? In this post, we’ll show you how to hook your Streamlit app up to a database—with row-level security, secret-handling best practices, and a quick table setup walkthrough."
weight: 20
---

{{<lead>}}
Need to store user credits securely? In this post, we’ll show you how to hook your Streamlit app up to a database—with row-level security, secret-handling best practices, and a quick table setup walkthrough.
{{</lead>}}

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

## Why Do We Need A Database?

Databases are the backbone of any dynamic application. In our case, we need to store and manage the credit balances of our users efficiently. Whether you’re updating credits after a transaction or querying the current balance, a database provides the reliability and performance necessary to keep everything in sync.

## Which Database Should I Choose?

There are a lot of options out there, and most would work just fine. For this project, I’ve chosen Supabase. It offers a generous free tier, is easy to set up, and integrates seamlessly with Streamlit. The decision ultimately comes down to deployment needs and the flexibility Supabase offers for modern apps.

Comparison + about supabase free tier (table)

## Requirements

add login

Before you start, install the necessary package with `pip install st-supabase-connection`.

## Connecting To A Database

### Configuring Your Supabase Project

- Sign up to [Supabase](https://supabase.com) and create a free account.
- Create a new project by filling out the required information:
  - project name
  - database password
  - region (closest to your users)
  - Click "Create new project"
- In the left sidebar, click "Table Editor", click "Create a new table" and fill out the required information:
  - Name: the name of the table, we will use `users`, if you decide to use something else just make sure to adpat the code examples accordingly
  - Description: optional
  - Keep "Enable Row Level Security (RLS)" checked.
  - Edit columns so you have the following:
    - id, int8, set as primary key
    - created_at, timestamptz, with default value `now()`
    - email, text, click the gear icon, uncheck "Is Nullable" and check "Is Unique"
    - credits, int4 (max value is 2147483647, i.e. more than $21 million because we will store cents), click the gear icon, uncheck "Is Nullable", set `0` as the default value
  - Click "Save"
- In the left sidebar, click "Project overview", scroll down a bit, this is where you can find the `Project URL` and `API Key`, you will need them soon.

RLS Policies

- In the left sidebar, click "Authentication", select "Policies" if it's not already selected, and click "Create policy".
- Create one policies from the templates "Enable insert for authenticated users only"
- Click on the policy template "Enable read access for all users", rename it "Enable read access for authenticated users only", select `authenticated` for target roles, click on "Save policy".
- Enable update policy
- Still in "Authentication", click on "Users" in the left sidebar. These are the Supabase users, not your app users, they are used to determine who has the right to which policy with RLS.
- Click on "Add user > Create a new user", put in an email address and a password of your choice, leave "auto confirm user" checked on and click "create user"
- Keep your email address and password available, you will need them soon.

Sign Up and Create a Project:
Visit Supabase and sign up for a free account.
Click “Create new project” and fill in:
Project Name: Pick something memorable.
Database Password: Choose a strong password.
Region: Select the region closest to your users.
Click "Create new project".
Create Your Table:
In the left sidebar, click Table Editor.
Click Create a new table and fill in the following:
Name: Use users (or any name you prefer—just update your code accordingly).
Description: Optional.
Enable Row Level Security (RLS): Keep this checked.
Edit Columns to include:
id: Type int8 and set as the primary key.
created_at: Type timestamptz with a default value of now().
email: Type text. Click the gear icon, uncheck "Is Nullable" and check "Is Unique".
credits: Type int4 (max value 2147483647, which is more than enough since we store cents). Click the gear icon, uncheck "Is Nullable", and set the default value to 0.
Click Save.
Get Your Project Credentials:
In the left sidebar, click Project overview.
Scroll down to find your Project URL and API Key. You’ll need these soon.
Set Up RLS Policies:
In the left sidebar, click Authentication, then select Policies.
Click Create policy and choose the template Enable insert for authenticated users only.
Next, select the template Enable read access for all users, rename it to Enable read access for authenticated users only, set the target role to authenticated, and click Save policy.
Still under Authentication, click Users in the left sidebar.
Click Add user > Create a new user, enter an email address and a password of your choice, ensure auto confirm user is checked, then click create user.
Keep these credentials handy for your connection.

### Connecting To Your Database

In your `secrets.toml` file, add the following section:

```toml
[connections.supabase]
SUPABASE_URL = "xxxx"
SUPABASE_KEY = "xxxx"
EMAIL_ADDRESS = "xxxx"
PASSWORD = "xxxx"
```

{{<alert>}} Make sure you never publish your secrets, do not commit them to GitHub! Always include `secrets.toml` in your `.gitignore` file. Read more about secrets management [here](https://docs.streamlit.io/develop/concepts/connections/secrets-management). {{</alert>}}

<!-- markdownlint-disable -->
{{<highlight python "linenos=table,hl_lines=2-3 5">}}
import streamlit as st
from st_supabase_connection import SupabaseConnection

# Initialize connection.
conn = st.connection("supabase",type=SupabaseConnection)
conn.auth.sign_in_with_password(
    {
        "email": st.secrets["connections"]["supabase"]["EMAIL_ADDRESS"],
        "password": st.secrets["connections"]["supabase"]["PASSWORD"],
    }
)
{{</highlight>}}
<!-- markdownlint-enable -->

Explain caching and ttl.

## Create users as they connect

## Testing Your Database Setup

## Next Steps
