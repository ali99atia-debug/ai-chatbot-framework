# AI Chatbot Framework - WhatsApp Integration

## Prerequisites

- WhatsApp Business Account
- WhatsApp Business API access
- Business verification
- Phone number (dedicated for bot)
- HTTPS server (for webhooks)

## Setup WhatsApp Business Account

### Step 1: Create Business Account

1. Visit [business.facebook.com](https://business.facebook.com)
2. Sign in with Facebook account
3. Click "Create Account"
4. Fill in business details
5. Verify business information

### Step 2: Get API Credentials

1. Go to [developers.facebook.com](https://developers.facebook.com)
2. Create a new app (Business type)
3. Add "WhatsApp" product
4. Get credentials:
   - Account ID
   - Access Token
   - Phone Number ID
   - Business Account ID

Store in `.env`:
```
WHATSAPP_ACCOUNT_ID=1234567890
WHATSAPP_ACCESS_TOKEN=EAABs...token...
WHATSAPP_PHONE_NUMBER_ID=1234567890
WHATSAPP_BUSINESS_ACCOUNT_ID=1234567890
```

### Step 3: Configure Webhook

1. In App Settings → Webhooks
2. Set Callback URL: `https://your-domain.com/whatsapp/webhook`
3. Set Verify Token: `your-random-token`
4. Subscribe to messages webhook

## Webhook Setup

### FastAPI Implementation

```python
from fastapi import FastAPI, Request, Response
import hmac
import hashlib
import json

app = FastAPI()
WHATSAPP_VERIFY_TOKEN = "your-random-token"
WHATSAPP_PHONE_NUMBER_ID = "1234567890"

@app.get("/whatsapp/webhook")
async def verify_webhook(request: Request):
    """Verify webhook endpoint with Facebook"""
    mode = request.query_params.get("hub.mode")
    token = request.query_params.get("hub.verify_token")
    challenge = request.query_params.get("hub.challenge")
    
    if mode == "subscribe" and token == WHATSAPP_VERIFY_TOKEN:
        return Response(content=challenge, status_code=200)
    else:
        return Response(content="Unauthorized", status_code=403)

@app.post("/whatsapp/webhook")
async def handle_webhook(request: Request):
    """Handle incoming WhatsApp messages"""
    body = await request.json()
    
    # Verify request signature
    if not verify_signature(request, body):
        return Response(content="Unauthorized", status_code=403)
    
    # Process webhook
    await process_whatsapp_message(body)
    
    return Response(content="Received", status_code=200)

def verify_signature(request: Request, body: dict) -> bool:
    """Verify message signature from WhatsApp"""
    x_hub_signature = request.headers.get("X-Hub-Signature-256")
    if not x_hub_signature:
        return False
    
    body_str = json.dumps(body, separators=(',', ':'))
    expected_signature = hmac.new(
        WHATSAPP_APP_SECRET.encode(),
        body_str.encode(),
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(
        x_hub_signature,
        f"sha256={expected_signature}"
    )
```

## Handle Messages

### Parse Incoming Message

```python
import asyncio
from typing import Dict, Any

async def process_whatsapp_message(body: Dict[str, Any]):
    """Extract and process WhatsApp message"""
    try:
        entry = body["entry"][0]
        changes = entry["changes"][0]
        value = changes["value"]
        
        if "messages" not in value:
            return  # Not a message
        
        message = value["messages"][0]
        user_id = message["from"]
        phone_number = value["metadata"]["display_phone_number"]
        
        # Get message content
        message_type = message["type"]
        
        if message_type == "text":
            text_content = message["text"]["body"]
            await handle_text_message(user_id, text_content)
            
        elif message_type == "image":
            image_id = message["image"]["id"]
            await handle_image_message(user_id, image_id)
            
        elif message_type == "document":
            document_id = message["document"]["id"]
            await handle_document_message(user_id, document_id)
            
        elif message_type == "location":
            location = message["location"]
            latitude = location["latitude"]
            longitude = location["longitude"]
            await handle_location_message(user_id, latitude, longitude)
            
    except Exception as e:
        logger.error(f"Error processing message: {e}")

async def handle_text_message(user_id: str, text: str):
    """Process text message"""
    # Process with AI
    response = await get_ai_response(text)
    
    # Send reply
    await send_text_message(user_id, response)

async def handle_image_message(user_id: str, image_id: str):
    """Process image message"""
    # Download image
    image_url = await get_media_url(image_id)
    
    # Process with vision API
    response = await analyze_image(image_url)
    
    # Send reply
    await send_text_message(user_id, response)
```

## Send Messages

### Send Text Message

```python
import aiohttp

async def send_text_message(recipient_phone: str, message: str):
    """Send text message via WhatsApp API"""
    url = f"https://graph.instagram.com/v18.0/{WHATSAPP_PHONE_NUMBER_ID}/messages"
    
    headers = {
        "Authorization": f"Bearer {WHATSAPP_ACCESS_TOKEN}",
        "Content-Type": "application/json"
    }
    
    payload = {
        "messaging_product": "whatsapp",
        "recipient_type": "individual",
        "to": recipient_phone,
        "type": "text",
        "text": {
            "preview_url": False,
            "body": message
        }
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.post(url, json=payload, headers=headers) as resp:
            result = await resp.json()
            return result
```

### Send Template Message

```python
async def send_template_message(recipient_phone: str, template_name: str, parameters: list):
    """Send templated message"""
    url = f"https://graph.instagram.com/v18.0/{WHATSAPP_PHONE_NUMBER_ID}/messages"
    
    payload = {
        "messaging_product": "whatsapp",
        "to": recipient_phone,
        "type": "template",
        "template": {
            "name": template_name,
            "language": {
                "code": "en_US"
            },
            "components": [
                {
                    "type": "body",
                    "parameters": [
                        {"type": "text", "text": param}
                        for param in parameters
                    ]
                }
            ]
        }
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.post(url, json=payload, headers=headers) as resp:
            return await resp.json()
```

### Send Interactive Message (Buttons)

```python
async def send_button_message(
    recipient_phone: str,
    header: str,
    body: str,
    buttons: list
):
    """Send message with interactive buttons"""
    url = f"https://graph.instagram.com/v18.0/{WHATSAPP_PHONE_NUMBER_ID}/messages"
    
    payload = {
        "messaging_product": "whatsapp",
        "recipient_type": "individual",
        "to": recipient_phone,
        "type": "interactive",
        "interactive": {
            "type": "button",
            "header": {
                "type": "text",
                "text": header
            },
            "body": {
                "text": body
            },
            "action": {
                "buttons": [
                    {
                        "type": "reply",
                        "reply": {
                            "id": button["id"],
                            "title": button["title"]
                        }
                    }
                    for button in buttons
                ]
            }
        }
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.post(url, json=payload, headers=headers) as resp:
            return await resp.json()
```

## Message Templates

### Create Template

```python
async def create_message_template(
    name: str,
    category: str,
    body: str,
    footer: str = None
):
    """Create WhatsApp message template"""
    url = f"https://graph.instagram.com/v18.0/{WHATSAPP_BUSINESS_ACCOUNT_ID}/message_templates"
    
    components = [
        {
            "type": "BODY",
            "text": body
        }
    ]
    
    if footer:
        components.append({
            "type": "FOOTER",
            "text": footer
        })
    
    payload = {
        "name": name,
        "category": category,  # MARKETING, TRANSACTIONAL
        "components": components
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.post(url, json=payload, headers=headers) as resp:
            return await resp.json()
```

### Template Examples

```json
{
  "name": "order_confirmation",
  "category": "TRANSACTIONAL",
  "components": [
    {
      "type": "BODY",
      "text": "Your order {{1}} has been confirmed! Total: {{2}}"
    },
    {
      "type": "FOOTER",
      "text": "Thank you for your purchase"
    }
  ]
}
```

## Store Conversations

```python
from sqlalchemy import Column, String, DateTime, Text, Integer
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class WhatsAppConversation(Base):
    __tablename__ = "whatsapp_conversations"
    
    id = Column(Integer, primary_key=True)
    user_phone = Column(String(20), index=True)
    user_name = Column(String(100))
    message_type = Column(String(20))  # text, image, document
    message_content = Column(Text)
    response_content = Column(Text)
    timestamp = Column(DateTime, default=datetime.utcnow)
    
async def store_conversation(
    phone: str,
    name: str,
    message: str,
    response: str,
    message_type: str = "text"
):
    """Store conversation in database"""
    session = Session()
    conversation = WhatsAppConversation(
        user_phone=phone,
        user_name=name,
        message_type=message_type,
        message_content=message,
        response_content=response
    )
    session.add(conversation)
    session.commit()
    session.close()
```

## Error Handling

```python
async def handle_webhook_with_retry(request: Request):
    """Handle webhook with retry logic"""
    max_retries = 3
    retry_count = 0
    
    while retry_count < max_retries:
        try:
            body = await request.json()
            await process_whatsapp_message(body)
            return Response(content="Received", status_code=200)
        except Exception as e:
            retry_count += 1
            logger.warning(f"Error processing message (attempt {retry_count}): {e}")
            
            if retry_count < max_retries:
                await asyncio.sleep(2 ** retry_count)  # Exponential backoff
            else:
                logger.error(f"Failed to process message after {max_retries} attempts")
                return Response(content="Error", status_code=500)
```

## Testing

```python
import pytest
from unittest.mock import Mock, patch

@pytest.mark.asyncio
async def test_handle_text_message():
    """Test text message handling"""
    with patch('send_text_message') as mock_send:
        await handle_text_message('5551234567', 'Hello')
        mock_send.assert_called_once()

@pytest.mark.asyncio
async def test_webhook_verification():
    """Test webhook verification"""
    request = Mock()
    request.query_params = {
        "hub.mode": "subscribe",
        "hub.verify_token": WHATSAPP_VERIFY_TOKEN,
        "hub.challenge": "challenge_token"
    }
    
    response = await verify_webhook(request)
    assert response.status_code == 200
```

## Rate Limiting

WhatsApp API limits:
- 80 API calls per second
- 1000 messages per second per phone number

```python
from ratelimit import limits, sleep_and_retry
import time

@sleep_and_retry
@limits(calls=80, period=1)  # 80 calls per second
async def send_message_with_rate_limit(phone: str, message: str):
    """Send message with rate limiting"""
    return await send_text_message(phone, message)
```

## Troubleshooting

### Messages Not Delivering
- Verify phone number format (+1234567890)
- Check access token validity
- Ensure phone number is registered

### Webhook Not Receiving
- Verify HTTPS certificate
- Check firewall rules
- Verify webhook URL is accessible

### Template Issues
- Wait 24 hours after template creation
- Check template approval status
- Verify parameter count matches template

## Resources

- [WhatsApp Business API Docs](https://developers.facebook.com/docs/whatsapp/cloud-api/)
- [Message Types Reference](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/messages)
- [Webhook Reference](https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks/)
