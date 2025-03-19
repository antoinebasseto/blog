---
title: "Calling LLM APIs"
date: 2025-03-18
summary: "Ready for your app to do something? In this post learn how to query an LLM API — including measuring costs and removing credits from users."
weight: 50
showComments: true
---

{{<lead>}}
Ready for your app to do something? In this post learn how to query an LLM API — including measuring costs and removing credits from users.
{{</lead>}}

<!-- Series blurb -->
{{<alert "circle-info">}}
This post is [part of a series]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai">}}) to learn how to build and deploy a Streamlit app with a credit system, covering authentication, payments with Stripe, databases, and more.
{{</alert>}}

## Why Are We Talking About This?

LLM APIs charge based on token usage, which means every request has a cost. If you're letting users interact with these models, you need to understand how much you're being charged and ensure you're pricing your credits accordingly. Without a solid system, you risk losing money on every API call, as users might consume more than what they're paying for.

### Choosing The Right API

There are _a lot_ of ways to call LLMs in a python application. If you've been keeping an eye on the space, you might feel like there's a new API every week. Let's break down the main options:

- **Direct Integration:** Providers like [OpenAI](https://platform.openai.com/docs/overview), [Google](https://ai.google.dev/gemini-api/docs), and [Anthropic](https://docs.anthropic.com/en/api/getting-started) offer direct access to their models. This approach allows you to tap into each model's unique features but can make switching between providers a bit of a headache due to differing integration methods.
- **Unified Interface Libraries:** Tools such as [LiteLLM](https://docs.litellm.ai/docs/providers/openai) and [PydanticAI](https://ai.pydantic.dev) provide a standardized interface for interacting with various LLMs. This abstraction simplifies the process of switching providers by minimizing code changes.
- **High-Level Frameworks:** Frameworks like [LangChain](https://python.langchain.com/docs/introduction/) offer advanced functionalities for building applications with LLMs, including prompt management and chaining model calls.
- **Specialized Tools for Prompt Optimization:** Libraries like [AdalFlow](https://adalflow.sylph.ai) and [DSPy](https://dspy.ai) focus on optimizing prompts to enhance model performance.

In this tutorial, we'll use PydanticAI because it strikes a balance between simplicity and functionality. It handles essential tasks like structured responses and tool declarations without bogging you down with unnecessary abstractions. Plus, it allows for easy switching between providers, giving your application the flexibility it needs.

### Choosing The Right LLM

No matter how you call an LLM, you still need to pick the right model. I recommend using [Artificial Analysis](https://artificialanalysis.ai) to compare models based on performance and cost, but ultimately, the best choice depends on your specific application. For this tutorial, we'll use `gemini-2.0-flash` — an affordable yet powerful option that balances cost and capability.

## Requirements

Before we dive in, make sure you have:

- **User Authentication, a Database, and Stripe integration:** If you haven't done this yet, follow the first part of this blog for [authentication]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/setting-up-user-login">}}), the second for [databases]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/connecting-to-a-database">}}) and the third for [Stripe integration]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/integrating-stripe-checkouts">}}).
- **PydanticAI's Python SDK:** Install it with `pip install pydantic-ai`.
- **An API Key from a Model Provider:** Either one from [OpenAI](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key), [Google](https://ai.google.dev/gemini-api/docs/api-key), [Anthropic](https://console.anthropic.com/settings/keys) or any of the other model providers supported by [PydanticAI](https://ai.pydantic.dev/models/).

## Calling an LLM API

Let's set up a simple agent to answer questions using PydanticAI.

### Setting Up Your API Key

First, grab an API key from your chosen provider and store it securely in your `.streamlit/secrets.toml` file.

{{<alert>}} Make sure you never publish your secrets, do not commit them to GitHub! Always include `secrets.toml` in your `.gitignore` file. Read more about secrets management [here](https://docs.streamlit.io/develop/concepts/connections/secrets-management). {{</alert>}}

Add the following section to your `secrets.toml` file:

```toml
GEMINI_API_KEY = "xxxx"
```

### Making API Requests

Now that we have our API key set up, let's make a request to the LLM using PydanticAI. We'll define a helper function to initialize the agent and another function to send queries.

We’ll use Streamlit's caching mechanism to store the agent instance:

```python
@st.cache_resource
def get_agent() -> (Agent[str, str]):
    os.environ["GEMINI_API_KEY"] = st.secrets["GEMINI_API_KEY"]

    agent = Agent(
        "google-gla:gemini-2.0-flash",
        system_prompt="You are a helpful assistant.",
    )

    return agent
```

This function retrieves the API key from secrets.toml, initializes an Agent with the gemini-2.0-flash model, and caches it for efficient reuse.

To send a user query and retrieve a response, use the following function:

```python
def get_answer(query: str) -> str:
    agent = get_agent()

    run_result = agent.run_sync(query)

    return run_result.data
```

This function fetches the cached agent, sends the query synchronously (but not really, more on that [later]({{<relref "#a-word-on-asynchronous-operations-in-streamlit">}})), and returns the response.

To make this interactive in a Streamlit app, add a simple input field and display the response:

```python
query = st.chat_input()

if query:
    response = get_answer(query)
    st.write(response)
```

With this setup, users can type a message, send it to the model, and see the response displayed in the app.

### Consuming Credits Properly

Since LLM APIs charge based on token usage, we need to track consumption and deduct credits accordingly. Let’s break this down step by step.

To determine how many credits to subtract, we calculate the cost of input and output tokens based on the model's pricing.

```python
from pydantic_ai.agent import AgentRunResult

INPUT_PRICE_PER_MILLION_TOKENS = 0.1
OUTPUT_PRICE_PER_MILLION_TOKENS = 0.4

def get_credits_to_remove(run_result: AgentRunResult) -> int:
    """
    Calculate the number of credits to remove from the user's balance based on the API usage of the provided run result.

    Args:
        run_result: The run result
    """
    request_tokens = run_result.usage().request_tokens
    response_tokens = run_result.usage().response_tokens

    request_price = request_tokens / 1000000 * INPUT_PRICE_PER_MILLION_TOKENS
    response_price = response_tokens / 1000000 * OUTPUT_PRICE_PER_MILLION_TOKENS

    api_price = request_price + response_price
    credits_to_remove = calculate_credits_for_api_usage(api_price)

    return credits_to_remove
```

This function extracts token usage from the API response, calculates the cost, and converts it into a credit deduction.

You will notice we use a function called `calculate_credits_for_api_usage`, now this function is really simple, it just applies a pricing formula. Here's the function:

To convert API cost into credits, we use a simple formula:

```python
def calculate_credits_for_api_usage(api_price: float | None) -> int:
    """
    Calculate the number of credits to remove for an API call.

    Args:
        api_price: API usage price in dollars

    Returns:
        int: Number of credits to remove (in cents)
    """
    # If API was free, return 1 cent
    if api_price is None:
        return 1

    # Apply pricing formula and convert to cents
    return ceil((api_price * 4) * 100)
```

The formula allows flexibility in adjusting pricing, ensuring users are charged fairly based on API costs.

We charge users 4 times what the API costs us, but there’s one big thing to keep in mind—Stripe’s fees. In the worst case, they take 3.5% + $0.30 per transaction.

Let’s break it down: if a user buys the minimum $1 in credits and spends it all, we actually get $0.75 (since we’re marking up 4x). After Stripe takes their cut:

{{<katex>}}
$$0.965 * 0.75 - 0.3 = 0.42$$

That leaves us with $0.42 after fees. Not great, but this is the worst case. Things look better when users top up with larger amounts (since the $0.30 fee matters less) or if they don’t use all their credits.

Once we calculate the credits to remove, we update the user’s balance in the database:

```python
def remove_credits_from_user(conn: SupabaseConnection, credits_to_remove: int) -> None:
    """
    Removes credits from user balance after API usage.

    Args:
        conn: Supabase connection instance
        credits_to_remove: Number of credits to subtract from user balance
    """
    # Get current user credits
    user_credits = get_user_credits(conn)

    # Calculate and update new credit balance
    new_credits = user_credits - credits_to_remove

    execute_query(
        conn.table("users")
        .update({"credits": new_credits})
        .eq("email", st.experimental_user.email),
        ttl=0,
    )
```

This function retrieves the user’s current balance, deducts the required credits, and updates the database.

Finally, we need to check the user has enough credits before making anb API call, and remove the credits after.

<!-- markdownlint-disable MD034 MD046 -->
{{<highlight python "linenos=table,hl_lines= 4 7-8 12-14">}}
def get_answer(query: str, conn: SupabaseConnection) -> str:
    agent = get_agent()

    if get_user_credits(conn) > 0:
      run_result = agent.run_sync(query)

      credits_to_remove = get_credits_to_remove(run_result)
      remove_credits_from_user(conn, credits_to_remove)

      return run_result.data
    
    else:
      # Handle not enough credits
      st.error("Insufficient credits")
      return ""
{{</highlight>}}

If the user has enough credits, we proceed with the query, calculate usage, and deduct credits. Otherwise, we display an error message.

With this system in place, users are charged accurately for their API usage, ensuring your app remains financially sustainable.

### A Word on Asynchronous Operations in Streamlit

Streamlit runs everything synchronously by default, which can be tricky when working with LLM APIs that support async operations. If you’re calling an API that provides an async interface, you might be tempted to use asyncio.run(), but Streamlit doesn’t play well with that. If you end up having to go down that rabbit hole though, I recommend [this article](https://sehmi-conscious.medium.com/got-that-asyncio-feeling-f1a7c37cab8b).

## Testing Your Consumption Workflow

Now that everything is set up, it’s time to test. Make sure the API is responding correctly and verify that credits are being deducted accurately after each request.

## Next Steps

Now that payments are working and credits update correctly, it's time to put them to use! Next, we'll learn how to [deploy our app]({{<ref "/posts/credit-based-payment-system-streamlit-generative-ai/deploying-your-app">}})!

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
