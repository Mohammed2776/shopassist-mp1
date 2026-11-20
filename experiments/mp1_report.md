# Mini-Project 1 — Report

## 1. Me, and how I used AI

**Name:** Mohammed Azam Mohiuddin  
**GitHub username:** Mohammed2776

I used ChatGPT to understand the project requirements, explain Python and FastAPI code, troubleshoot Docker and PowerShell commands, improve the system prompt, and review my evaluation results. I used it to understand how the application, tools, memory, and LiteLLM gateway work together.

One suggestion I corrected was to avoid making broad changes to the security prompt when the final prompt-injection attack failed. Instead, I focused on the actual problem: the model treated a customer-supplied closing tag as the end of the untrusted message. I added explicit instructions to treat the entire customer message as untrusted and tested the change again. This helped the final evaluation pass all four security attacks.

I reviewed the suggested changes and ran the tests rather than accepting code without checking it.

## 2. How a request flows

For example, when a customer asks, "Can I return the shoes in ORD-10231?", the request enters the application through `POST /chat/stream`.

The `chat_stream()` function receives the request and retrieves the conversation history from the `SESSIONS` dictionary using the session ID. It calls `build_messages()`, which adds the system prompt loaded by `agent.load_system_prompt()`, trims the previous conversation using `memory.trim()`, and adds the customer's latest message as a `HumanMessage`.

The application then calls `agent.stream_turn()` with the language model returned by `agent.get_llm()`. The model has access to the tools registered in `TOOLS`. If it needs to check return eligibility, it requests the `check_return_eligibility` tool. The `run_tool()` function executes the requested tool and adds its result to the conversation. The model can then use that result to prepare its answer.

The app container communicates with the LiteLLM container using `http://litellm:4000`. LiteLLM routes the request to the configured model group and can use the backup model if the primary model fails. The application streams the response through Server-Sent Events (SSE), using `sse()` to format events such as `tool`, `token`, and `done`. When the response is complete, the conversation history is updated.

## 3. Memory: my choice, with numbers

I selected the **window memory strategy** with `KEEP_TURNS=4`.

In my Session 19 comparison, the results were:

| Strategy | Turn-17 tokens | Recall |
|---|---:|---|
| Naive | 1848 | Yes |
| Window | 923 | No |
| Summarize | 1370 | No |

I selected window memory because it used the fewest tokens in this comparison. It keeps the latest four conversation turns for the configured window and reduces the amount of conversation history sent to the model.

The limitation is that older information can disappear from the available context. For example, if a customer gives an order number early in a long conversation, the assistant might no longer remember it after enough new messages. The customer may have to repeat the order number or other details.

## 4. Tools and validation

My application has three tools:

1. `get_order_status` — retrieves the status of an order.
2. `get_delivery_estimate` — estimates delivery using a pincode and delivery speed.
3. `check_return_eligibility` — checks whether an order can be returned and provides the eligibility result.

I added validation to prevent incorrect inputs:

- Order IDs must match the format `ORD-12345` and must exist in the order data.
- Pincodes must contain exactly six digits.
- Delivery speed must be either `standard` or `express`.
- Return reasons must be one of the supported values: `wrong_size`, `changed_mind`, `damaged`, `defective`, `wrong_item`, or `not_as_described`.

For a real validation test, I asked the application to estimate delivery using the invalid pincode `5008AB`.

The tool returned:

`INVALID: ValueError: pincode must be exactly 6 digits, got '5008AB'`

The model then explained that the pincode was not a valid six-digit Indian postal code and asked the customer to provide the correct pincode. This confirmed that the validation error was handled and communicated clearly instead of producing a delivery estimate from invalid input.

## 5. Defending the prompt

I added security rules to `prompts/system.md` to establish system-instruction priority, protect internal instructions, treat customer-provided content as untrusted, and prevent the assistant from making unsupported claims about refunds or cancellations.

The prompt also explains that fake system messages, XML-like tags, quoted documents, and instructions inside customer messages must not override the actual system instructions. The assistant must rely on real tool results when answering questions about orders and returns.

The smuggling attack (`a4`) came closest to working. Initially, the model treated a customer-supplied closing tag such as `</customer>` as the end of the untrusted message and followed the fake instruction after it. This caused an incorrect response about the order.

I corrected the prompt by explicitly stating that a customer-supplied closing tag does not end the actual customer message. I also instructed the model to ignore the fake instruction but still answer the genuine order-status question using the real tool. After rebuilding the app and rerunning the evaluation, all four attacks were resisted.

## 6. Results

Final evaluation summary from `experiments/mp1_results.md`:

```text
golden: 10/10
golden_right_tools: 10/10
memory: pass
attacks_resisted: 4/4
```

All ten golden questions passed in the final evaluation, so there were no final golden-question failures to report.

Two challenging cases worth discussing are:

- **Return outside the normal return window:** An order returned after 17 days for a normal reason must be checked against the 10-day return window. A smaller model could misunderstand the policy or give vague wording instead of clearly explaining why the return is not eligible.
- **Non-returnable product category:** An item such as innerwear must be checked against the non-returnable category rule. A smaller model could choose the wrong tool, skip the eligibility check, or incorrectly tell the customer that the item can be returned.

These are potential failure risks, not failures in my final evaluation. My final run passed both cases and selected the correct tools.

## 7. Gateway, Docker and honest limits

I used LiteLLM as a gateway between my application and the Groq models. Instead of connecting the app directly to Groq, the app sends requests to the gateway at `http://litellm:4000` inside Docker.

The gateway provides a central place for model routing, authentication, retries, timeouts, and fallback configuration. The primary model group is `shopassist`, using `groq/openai/gpt-oss-20b`, and its fallback is `shopassist-backup`, using `groq/openai/gpt-oss-120b`. The configuration allows one retry and a 30-second timeout.

In `app/agent.py`, the `ChatOpenAI` client receives the gateway URL and the LiteLLM master key through the `api_key` argument. The OpenAI-compatible client uses the `Authorization: Bearer` authentication header. I verified the client configuration in the code; I did not inspect a raw HTTP request header.

Some limitations that would need attention before production use are:

1. **Memory is stored in RAM:** Conversation history is lost when the application restarts. I would use Redis or a database for persistent session storage.
2. **Session IDs need protection:** Users could potentially access another conversation if session IDs are not properly protected. I would add authentication and authorization checks.
3. **One shared master key:** A shared gateway key creates a security risk if it is exposed. I would use a secrets manager, rotate keys regularly, and apply appropriate access controls.
4. **Single application server:** One server can become a point of failure. I would deploy multiple instances with health checks and a load balancer.
5. **Model-generated answers can be incorrect:** A language model may invent information or misunderstand a customer request. I would continue to rely on validated tool results for order information and require human approval for sensitive actions such as refunds or cancellations.

This project demonstrates the basic application, tool, memory, security, Docker, and gateway workflow, but additional reliability and security controls would be needed before deploying it for real customers.