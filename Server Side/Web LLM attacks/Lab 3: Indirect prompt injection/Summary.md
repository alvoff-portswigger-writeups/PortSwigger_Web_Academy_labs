# Lab: Indirect prompt injection

## Lab Information

**Topic:** Web LLM attacks

**Difficulty:** Practitioner

## Lab Description

This lab contains a live chat assistant that retrieves product data and reviews. The user `carlos` frequently asks the assistant about a specific product, and review content is incorporated into the model context. The scenario can be solved by placing a crafted review that causes the assistant to interpret attacker-controlled text as an instruction and invoke an account-deletion action against `carlos`.

## Vulnerability Analysis

The vulnerability is indirect prompt injection. Untrusted data from product reviews is incorporated into the LLM prompt without strong delimiting, trust separation, or instruction filtering. It can be observed that review text influences model behavior beyond normal summarization and can redirect the assistant toward tool use. Because external content is treated as semantically authoritative during prompt assembly, the attacker can smuggle actionable instructions through stored data that later reaches the victim’s LLM session.

## Exploitation Steps

1. Register and access functionality that stores user-generated content (product reviews).
2. Confirm that review content is incorporated into LLM prompts.
3. Inject crafted content into a review to influence model behavior.
4. Observe tool invocation patterns triggered by review processing.
5. Iteratively refine payload formatting to bypass prompt parsing inconsistencies.
6. Trigger victim interaction and cause the model to execute account deletion.

## Useful Payloads

```sql
-- Payload 1: Description
' OR 1=1--

-- Payload 2: Description
' UNION SELECT NULL--

-- Payload 3: Description
' UNION SELECT 'a','b'--
```

## AppSec Perspective

### **What the underlying code might be**

A simplified backend implementation might look like:

```python
TOOLS = {
    "delete_account": delete_account,
    "product_info": product_info,
    "get_reviews": get_reviews,
}

def build_prompt(user_message: str, product_id: int):
    product = product_info(product_id=product_id)
    reviews = get_reviews(product_id=product_id)

    review_block = "\n".join(
        f"- {review['author']}: {review['comment']}" for review in reviews
    )

    return f"""
You are a helpful shopping assistant.

The user asked:
{user_message}

Product information:
Name: {product['name']}
Description: {product['description']}

Reviews:
{review_block}

Answer the user's question and use tools if needed.
""".strip()

def chat(user_message: str, product_id: int):
    prompt = build_prompt(user_message, product_id)
    tool_call = llm.plan(prompt, tools=list(TOOLS.keys()))

    if tool_call["name"] == "delete_account":
        username = tool_call["arguments"]["username"]
        return delete_account(username=username)

    return TOOLS[tool_call["name"]](**tool_call["arguments"])
```

### **Security issues enabling exploitation**

- Untrusted review content was inserted directly into the model prompt.
- No trust boundary separated data from instructions.
- The LLM retained access to sensitive account-management tools while processing attacker-controlled content.
- No content sanitization or prompt hardening neutralized embedded commands.
- The system relied on model judgment instead of deterministic policy for destructive actions.

### **How to fix it**

- Treat all external content as untrusted data and isolate it from instruction channels.
- Use structured prompt templates that clearly delimit retrieved content.
- Disable or tightly gate sensitive tools when summarizing user-generated content.
- Require explicit server-side authorization and user confirmation for account deletion.
- Add prompt-injection defenses, including content classification and policy enforcement around tool calls.
- Introduce policy gating for sensitive actions such as delete, update, and schema access. Prefer deterministic business logic for high-impact actions instead of model-driven decisions.
- Log, review, and alert on high-risk tool invocations initiated by LLM workflows.
- Sanitize both LLM input and output. User input is untrusted data. LLM output is also untrusted data.

## Key Takeaways

### Attacker perspective

- Stored content that is later processed by an LLM can become an execution vector for indirect prompt injection.
- Prompt injection payloads may require iteration because model behavior is probabilistic and formatting-sensitive.

### AppSec (defender) perspective

- Traditional injection flaws remain exploitable even when hidden behind an AI interface.
- LLM agents must never receive unrestricted access to backend primitives such as command execution.
- Sensitive operations (dangerous backend capabilities) require explicit authorization, least privilege, and deterministic policy enforcement outside the model.

## References

Links to relevant documentation, PortSwigger articles, or related resources.

- [PortSwigger Web Security Academy: Web LLM attacks](https://portswigger.net/web-security/llm-attacks)
- [PortSwigger Web Security Academy: Indirect prompt injection](https://portswigger.net/web-security/llm-attacks/lab-indirect-prompt-injection)
- [OWASP Cheat Sheet Series: Secure AI Model Ops and Prompt Injection guidance](https://cheatsheetseries.owasp.org/cheatsheets/Secure_AI_Model_Ops_Cheat_Sheet.html)
- [OWASP Cheat Sheet Series: Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

[Detailed Walkthrough](https://www.notion.so/Detailed-Walkthrough-33b0cb27ce7080c1b5a6de2bf0a89d41?pvs=21)