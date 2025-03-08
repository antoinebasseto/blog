---
title: "Connecting To A Database"
date: 2025-03-03
summary: "Need to store user credits securely? In this post, we’ll show you how to hook your Streamlit app up to a database — with row-level security, secret-handling best practices, and a quick table setup walkthrough."
weight: 20
---

{{<lead>}}
Need to store user credits securely? In this post, we’ll show you how to hook your Streamlit app up to a database — with row-level security, secret-handling best practices, and a quick table setup walkthrough.
{{</lead>}}

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

## Why Do We Need A Database?

We need a database to keep track of user credit balances reliably. Whether adding or deducting credits, the data needs to stay consistent and accessible. Streamlit's `session_state` won’t work for this, as it resets every time the app reloads. Therefore, we need a proper database to store and manage credits.

## Which Database Should I Choose?

There are a lot of options out there, and there isn’t a one-size-fits-all solution. The ideal database depends on your specific needs—ease of setup, scalability and the available free tier are all key factors.

- If your application only needs to store **structured data** like user information and credits, a **relational (SQL) database** is typically the best choice due to its strong consistency and structured querying capabilities.
- However, if your app **requires flexibility** (e.g., dynamic or hierarchical data structures, unstructured metadata, or document-based storage), a **NoSQL database** might be a better fit.
- Also, if you need to store **large files**, consider pairing your database with a dedicated **file storage solution** (e.g., Supabase Storage, Firebase Storage, or AWS S3).

Below is a comparison of popular options:

| Database | Type | Free Tier | Connector? |
|----------|------|-----------|:----------:|
| **[Supabase](https://supabase.com/)** {{<badge>}}recommended{{</badge>}} | PostgreSQL | Unlimited API requests, up to 50,000 monthly active users, 500 MB database size, 5 GB bandwidth, and 1 GB file storage. [source](https://supabase.com/pricing) | {{<icon "check">}} |
| **[SQLite Cloud](https://sqlitecloud.io/)** | SQLite | 1 GB storage, up to 0.5 vCPUs, 0.5 GB RAM and 30 concurrent connections. [source](https://sqlitecloud.io/pricing) | {{<icon "xmark">}} |
| **[Firebase Firestore](https://firebase.google.com/products/firestore)** | NoSQL | Only `default` database, 1 GiB storage, 50000 reads / day, 20000 writes / day, 20000 deletes / day, 10 GiB data transfer / month. [source](https://firebase.google.com/docs/firestore/quotas) | {{<icon "check">}} |
| **[MongoDB Atlas](https://www.mongodb.com/atlas)** | NoSQL | 512 MB storage, shared RAM and vCPU. [source](https://www.mongodb.com/pricing) | {{<icon "check">}} |

Streamlit offers [step-by-step guides](https://docs.streamlit.io/develop/tutorials/databases) to show how to connect to most of these (and others).

For this project, we are going to use **Supabase**.  It offers a generous free tier, is easy to set up, and integrates seamlessly with Streamlit. We will also leverage their edge function features when [integrating Stripe]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/integrating-stripe-checkouts">}}).

## Requirements

Before we begin, make sure you have:

- **User Login Set Up:** If you haven't done this yet, follow the [first part of this series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/setting-up-user-login">}}).
- **Database Connector Installed:** Install it with `pip install st-supabase-connection`.

## Connecting To A Database

### Configure Your Supabase Project

- **Sign Up and Create a Project:**
  - Visit [Supabase](https://supabase.com) and sign up for a free account.
  - Click **“Create new project”** and fill in:
    - **Project Name.**
    - **Database Password:** Choose a strong password.
    - **Region:** Select the region closest to your users.
  - Click **"Create new project"**.
- **Create Your Table:**
  - In the left sidebar, click **Table Editor**.
  - Click **Create a new table** and fill in the following:
    - **Name:** Use users (or any name you prefer—just update your code accordingly).
    - **Description:** Optional.
    - **Enable Row Level Security (RLS):** Keep this checked.
    - **Edit Columns** to include:
      - **id:** Type int8 and set as the primary key.
      - **created_at:** Type timestamptz with a default value of now().
      - **email:** Type text. Click the gear icon, uncheck "Is Nullable" and check "Is Unique".
      - **credits:** Type int4 (we will be storing credits as cents, the max value of an int4 number is 2147483647 i.e. a lot of money). Click the gear icon, uncheck "Is Nullable", and set the default value to 0 (we will set a default value for new user in the code in case you do want to give new users some free credits).
  - Click **Save**.
- **Get Your Project Credentials:**
  - In the left sidebar, click **Project overview**.
  - Scroll down to find your **Project URL** and **API Key**. You’ll need these soon.
- **Set Up RLS Policies:**
  - In the left sidebar, click **Authentication**, then select **Policies**.
  - Click **Create policy** and choose the template **Enable insert for authenticated users only**, and click **Save policy**.
  - Next, select the template **Enable read access for all users**:
    - Rename it to **Enable read access for authenticated users only**.
    - Set the target role to **authenticated**.
    - Click **Save policy**.
  - Next, select again the template **Enable insert for authenticated users only**:
    - Rename it to **Enable update access for authenticated users only**.
    - Set the operation type to **Update**.
    - Click **Save policy**.
  - Still under **Authentication**, click **Users** in the left sidebar.
    - Click **Add user > Create a new user**, enter an email address and a password of your choice, ensure auto confirm user is checked, then click create user.
    - **Keep these credentials handy** for your connection.

### Configure Your Secrets

You know all this information I told you to keep handy? Well, your `.streamlit/secrets.toml` file is where you should put it!

{{<alert>}} Make sure you never publish your secrets, do not commit them to GitHub! Always include `secrets.toml` in your `.gitignore` file. Read more about secrets management [here](https://docs.streamlit.io/develop/concepts/connections/secrets-management). {{</alert>}}

Add the following section to your `secrets.toml` file:

```toml
[connections.supabase]
SUPABASE_URL = "xxxx"
SUPABASE_KEY = "xxxx"
EMAIL_ADDRESS = "xxxx"
PASSWORD = "xxxx"
```

### Connect With Your Streamlit App

Now, let's set up the connection to the Supabase database. The code below establishes and caches a connection, ensuring that you can interact with your database securely.

```python
@st.cache_resource(show_spinner=False)
def get_db_connection() -> SupabaseConnection:
    """
    Establish and cache a connection to the Supabase database.

    Returns:
        SupabaseConnection: Authenticated connection to Supabase
    """
    conn = st.connection("supabase", type=SupabaseConnection)
    conn.auth.sign_in_with_password(
        {
            "email": st.secrets["connections"]["supabase"]["EMAIL_ADDRESS"],
            "password": st.secrets["connections"]["supabase"]["PASSWORD"],
        }
    )
    return conn
```

### Create Users As They Connect

To ensure every user is accounted for, create a new database record as soon as they sign in—if one doesn’t already exist. This not only keeps your records clean but also prepares you for future updates like credit transactions. Here’s a simple function to achieve that:

```python
def ensure_user_in_database(conn: SupabaseConnection) -> None:
    """
    Ensure the current user exists in the database, creating a record if needed.

    Args:
        conn: Supabase connection
    """
    if not st.experimental_user.is_logged_in:
        return

    user_in_db = execute_query(
        conn.table("users").select("email").eq("email", st.experimental_user.email),
        ttl=0,
    )

    if len(user_in_db.data) == 0:
        execute_query(
            conn.table("users").insert(
                {"email": st.experimental_user.email, "credits": NEW_USER_CREDITS}
            ),
            ttl=0,
        )
```

Since logging in involves clicking a button, it triggers a rerun from Streamlit. So, as long as we place this at the start of our app, this function will be called as soon as the user has logged in and before any logic that requires the user to exist in the database.

Note that `NEW_USER_CREDITS` should be defined in your code with the value you wish to set as default credits for new users. This is where you can set how many (if any) free credits you want to give to your new users. Remember we store credits in cents (e.g. 50 credits is 50 cents).

## Caching and TTL

Caching is a handy way to boost performance by temporarily storing data so you don't have to query the database on every request. In our case, imagine frequently checking a user’s credit balance—caching avoids repetitive queries. The TTL (Time-to-Live) is the duration the cached data stays valid before a fresh query is made. This balances performance with accuracy. In this tutorial, we will just set every `ttl` to 0, which guarantees accuracy at the cost of unoptimized performance. A good optimization strategy would be to set long `ttl`s and clear the cache anytime we expect the database to have changed (a login, adding or removing credits).

## Next Steps

With a database in place, we can read and write credits for each of our users. Next, we’ll look at how to [integrate Stripe]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/integrating-stripe-checkouts">}}).

{{<alert "lightbulb">}}
Want to see an example? [SimplyPlot](https://simplyplot.streamlit.app/) uses a credit system for monetisation - it writes Matplotlib code for you, so you can stop troubleshooting your plots.
{{</alert>}}
