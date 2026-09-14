# AI Chatbot Framework - LLM Configuration

## Supported LLMs

### OpenAI GPT-4 (Recommended)

**Advantages:**
- Best performance and conversation quality
- Extensive context understanding
- Function calling support
- Multimodal capabilities

**Setup:**

```python
import openai

openai.api_key = os.getenv("OPENAI_API_KEY")

async def get_ai_response_openai(message: str, context: list = None) -> str:
    """Get response using OpenAI GPT-4"""
    messages = [
        {"role": "system", "content": "You are a helpful customer service assistant."},
    ]
    
    if context:
        messages.extend(context)
    
    messages.append({"role": "user", "content": message})
    
    response = await openai.ChatCompletion.acreate(
        model="gpt-4",
        messages=messages,
        temperature=0.7,
        max_tokens=500,
        top_p=0.9
    )
    
    return response["choices"][0]["message"]["content"]
```

**Pricing:**
- Input: $0.03 per 1K tokens
- Output: $0.06 per 1K tokens

### Anthropic Claude

**Advantages:**
- Strong at reasoning and analysis
- Good context window (100K tokens)
- Constitutional AI (safer responses)

**Setup:**

```python
import anthropic

client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

async def get_ai_response_claude(message: str) -> str:
    """Get response using Anthropic Claude"""
    response = client.messages.create(
        model="claude-3-sonnet-20240229",
        max_tokens=1024,
        system="You are a helpful customer service assistant.",
        messages=[
            {"role": "user", "content": message}
        ]
    )
    
    return response.content[0].text
```

### Open Source: LLaMA 2

**Advantages:**
- Self-hosted option
- No API costs
- Full control over data

**Setup:**

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

model_name = "meta-llama/Llama-2-7b-chat-hf"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,
    device_map="auto"
)

async def get_ai_response_llama(message: str) -> str:
    """Get response using LLaMA 2"""
    inputs = tokenizer(message, return_tensors="pt")
    
    with torch.no_grad():
        outputs = model.generate(
            **inputs,
            max_new_tokens=500,
            temperature=0.7,
            top_p=0.9
        )
    
    return tokenizer.decode(outputs[0], skip_special_tokens=True)
```

## System Prompts

### Customer Support

```python
CUSTOMER_SUPPORT_PROMPT = """
You are a friendly and helpful customer service assistant for an e-commerce company.

Your responsibilities:
- Answer questions about products and services
- Help with order tracking
- Process returns and refunds
- Escalate complex issues to human support

Guidelines:
- Be empathetic and understanding
- Provide clear, concise answers
- Use the customer's name when appropriate
- Offer solutions, not just explanations
- Be honest about limitations

If you cannot help, politely ask for more information or escalate to a human agent.
"""
```

### Technical Support

```python
TECH_SUPPORT_PROMPT = """
You are a technical support specialist.

When helping customers:
- Ask clarifying questions to understand the issue
- Provide step-by-step solutions
- Use technical language appropriately
- Suggest troubleshooting steps
- Provide workarounds when applicable

Escalate to level 2 support if:
- Hardware replacement needed
- Account access issues
- Data loss or corruption
"""
```

## Context Management

### Conversation History

```python
class ConversationContext:
    def __init__(self, user_id: str, max_turns: int = 10):
        self.user_id = user_id
        self.max_turns = max_turns
        self.messages = []
    
    def add_user_message(self, content: str):
        self.messages.append({
            "role": "user",
            "content": content,
            "timestamp": datetime.now()
        })
    
    def add_assistant_message(self, content: str):
        self.messages.append({
            "role": "assistant",
            "content": content,
            "timestamp": datetime.now()
        })
    
    def get_context(self) -> list:
        """Return last N messages for context"""
        return [
            {"role": m["role"], "content": m["content"]}
            for m in self.messages[-self.max_turns:]
        ]
    
    def clear(self):
        """Clear conversation history"""
        self.messages = []
```

### Memory Store

```python
from typing import Dict
import json

class UserMemory:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.data = {}
    
    def set(self, key: str, value: any):
        """Store user information"""
        self.data[key] = value
        self._save_to_db()
    
    def get(self, key: str, default=None):
        """Retrieve user information"""
        return self.data.get(key, default)
    
    def _save_to_db(self):
        """Persist to database"""
        # Save implementation
        pass

# Usage
memory = UserMemory(user_id="123456")
memory.set("name", "John Doe")
memory.set("previous_orders", ["ORD-001", "ORD-002"])
```

## Function Calling (Agents)

```python
from typing import Callable, Dict, List

class FunctionRegistry:
    def __init__(self):
        self.functions: Dict[str, Callable] = {}
        self.schemas: Dict[str, Dict] = {}
    
    def register(
        self,
        name: str,
        func: Callable,
        schema: Dict
    ):
        """Register a function for AI to call"""
        self.functions[name] = func
        self.schemas[name] = schema
    
    async def call(self, name: str, **kwargs) -> any:
        """Execute registered function"""
        if name not in self.functions:
            raise ValueError(f"Function {name} not found")
        
        func = self.functions[name]
        if asyncio.iscoroutinefunction(func):
            return await func(**kwargs)
        else:
            return func(**kwargs)
    
    def get_schemas(self) -> List[Dict]:
        """Get function schemas for AI model"""
        return list(self.schemas.values())

# Register functions
registry = FunctionRegistry()

# Order lookup function
registry.register(
    "lookup_order",
    lookup_order_func,
    {
        "type": "function",
        "function": {
            "name": "lookup_order",
            "description": "Look up order details by order ID",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string", "description": "Order ID"}
                },
                "required": ["order_id"]
            }
        }
    }
)

# Process AI response with function calls
async def process_ai_response(response: Dict):
    """Handle AI response that includes function calls"""
    if response["stop_reason"] == "tool_use":
        for tool_use in response["content"]:
            if tool_use["type"] == "tool_use":
                result = await registry.call(
                    tool_use["name"],
                    **tool_use["input"]
                )
                # Continue conversation with function result
```

## Prompt Engineering

### Few-Shot Learning

```python
def create_few_shot_prompt(user_input: str) -> str:
    """Create prompt with examples"""
    examples = [
        {
            "input": "How do I track my order?",
            "output": "You can track your order using your order ID on our tracking page."
        },
        {
            "input": "What's your return policy?",
            "output": "We offer 30-day returns on all items in original condition."
        }
    ]
    
    prompt = "Based on these examples, answer the following question:\n\n"
    for ex in examples:
        prompt += f"Q: {ex['input']}\nA: {ex['output']}\n\n"
    
    prompt += f"Q: {user_input}\nA:"
    return prompt
```

### Chain-of-Thought Prompting

```python
def create_chain_of_thought_prompt(question: str) -> str:
    """Use chain-of-thought for complex reasoning"""
    return f"""
Let's think through this step by step.

Question: {question}

Thinking:
1. First, I'll identify what's being asked
2. Then, I'll gather relevant information
3. Finally, I'll provide a clear answer

Answer:
"""
```

## Error Handling

```python
class LLMError(Exception):
    pass

class RateLimitError(LLMError):
    pass

class ContextLengthError(LLMError):
    pass

async def get_ai_response_with_fallback(
    message: str,
    primary_provider: str = "openai",
    fallback_provider: str = "claude"
) -> str:
    """Try primary provider, fall back to alternative"""
    try:
        if primary_provider == "openai":
            return await get_ai_response_openai(message)
        elif primary_provider == "claude":
            return await get_ai_response_claude(message)
    except RateLimitError:
        logger.warning(f"Rate limited on {primary_provider}, using fallback")
        if fallback_provider == "claude":
            return await get_ai_response_claude(message)
    except ContextLengthError:
        logger.warning("Context too long, using shorter window")
        # Truncate context and retry
        return await get_ai_response_openai(message[-1000:])
    except Exception as e:
        logger.error(f"Error getting AI response: {e}")
        return "I apologize, I'm having trouble understanding that right now."
```

## Performance Optimization

### Response Caching

```python
from functools import lru_cache
import hashlib

class ResponseCache:
    def __init__(self, ttl: int = 3600):
        self.cache = {}
        self.ttl = ttl
        self.timestamps = {}
    
    def _get_key(self, message: str, context: list) -> str:
        content = f"{message}:{json.dumps(context)}"
        return hashlib.md5(content.encode()).hexdigest()
    
    async def get_or_fetch(
        self,
        message: str,
        fetch_func: Callable,
        context: list = None
    ) -> str:
        key = self._get_key(message, context or [])
        
        # Check cache
        if key in self.cache:
            if time.time() - self.timestamps[key] < self.ttl:
                return self.cache[key]
        
        # Fetch fresh response
        response = await fetch_func(message)
        self.cache[key] = response
        self.timestamps[key] = time.time()
        
        return response
```

## Monitoring & Logging

```python
import logging

logger = logging.getLogger(__name__)

async def log_llm_usage(
    user_id: str,
    model: str,
    prompt_tokens: int,
    completion_tokens: int,
    cost: float
):
    """Log LLM API usage for cost tracking"""
    logger.info(
        f"LLM Usage - User: {user_id}, Model: {model}, "
        f"Prompt: {prompt_tokens}, Completion: {completion_tokens}, "
        f"Cost: ${cost:.4f}"
    )
```

## Testing LLM Responses

```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_ai_response():
    """Test AI response generation"""
    with patch('openai.ChatCompletion.acreate') as mock_api:
        mock_api.return_value = {
            "choices": [{
                "message": {"content": "Your order is being shipped."}
            }]
        }
        
        response = await get_ai_response_openai("Where is my order?")
        assert "shipped" in response.lower()
```
