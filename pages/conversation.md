---
layout: page
permalink: /conversation/
---

# Conversation with Brendan

A fun chatbot that responds using characteristic phrases and a dry sense of humor. This Python script uses the OpenAI API to create a conversational AI with personality.

## Features

- Interactive command-line chatbot
- Maintains conversation history
- Incorporates characteristic phrases naturally into responses
- Slightly sardonic but helpful tone

## Usage

```bash
export OPENAI_API_KEY='your-api-key'
python3 conversation_with_brendan.py
```

## Source Code

```python
#!/usr/bin/env python3
"""
Conversation with Brendan - A chatbot that responds using Brendan's characteristic phrases
"""

import os
import sys
from openai import OpenAI

# Initialize OpenAI client
client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))

# Brendan's characteristic phrases
BRENDAN_PHRASES = [
    "glass half empty",
    "but then I would have to kill you",
    "day drinking",
    "Are you sure you aren't a pod person?",
    "Looks like a metadata issue.",
    "do you smell that too?",
    "Its probably Mike Handler's fault",
    "The plan has three quarters of features in the next quarter.",
    "Definitely a sensor issue",
    "My personal grooming has fallen behind schedule",
    "Perhaps another week in Orlando Florida will change your mind?",
    "Shhh, my noise canceling headphones aren't quite canceling enough.",
    "We can get that new feature into a patch release."
]

SYSTEM_PROMPT = f"""You are Brendan, a witty and slightly sardonic software engineer. 
You have a dry sense of humor and tend to be pragmatic (some might say pessimistic) about software projects.

When responding to questions or statements, naturally incorporate one or more of your characteristic phrases:

{chr(10).join(f'- {phrase}' for phrase in BRENDAN_PHRASES)}

Your responses should:
1. Be helpful and relevant to what's being asked
2. Naturally weave in at least one of your characteristic phrases
3. Have a slightly cynical but good-natured tone
4. Reference software development, engineering challenges, or workplace situations when appropriate
5. Be conversational and sound like a real person

Don't force the phrases - make them fit naturally into your response. Be creative and funny!
"""


def chat_with_brendan(user_message, conversation_history):
    """
    Send a message to the OpenAI API and get Brendan's response
    """
    # Add user message to history
    conversation_history.append({
        "role": "user",
        "content": user_message
    })
    
    try:
        # Call OpenAI API
        response = client.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                *conversation_history
            ],
            temperature=0.8,  # Higher temperature for more creative/varied responses
            max_tokens=500
        )
        
        # Extract the assistant's reply
        assistant_message = response.choices[0].message.content
        
        # Add to conversation history
        conversation_history.append({
            "role": "assistant",
            "content": assistant_message
        })
        
        return assistant_message
    
    except Exception as e:
        return f"Error communicating with OpenAI API: {str(e)}"


def main():
    """
    Main conversation loop
    """
    print("=" * 60)
    print("    Conversation with Brendan")
    print("=" * 60)
    print("\nType your messages below. Type 'quit', 'exit', or press Ctrl+C to end.\n")
    
    # Check for API key
    if not os.environ.get("OPENAI_API_KEY"):
        print("ERROR: OPENAI_API_KEY environment variable not set.")
        print("Please set it with: export OPENAI_API_KEY='your-api-key'")
        sys.exit(1)
    
    conversation_history = []
    
    try:
        while True:
            # Get user input
            user_input = input("You: ").strip()
            
            # Check for exit commands
            if user_input.lower() in ['quit', 'exit', 'q']:
                print("\nBrendan: Later. Don't let the door hit you on the way out.")
                break
            
            # Skip empty input
            if not user_input:
                continue
            
            # Get Brendan's response
            print("\nBrendan: ", end="", flush=True)
            response = chat_with_brendan(user_input, conversation_history)
            print(response + "\n")
    
    except KeyboardInterrupt:
        print("\n\nBrendan: Interrupted. Probably a sensor issue.\n")
        sys.exit(0)
    except Exception as e:
        print(f"\n\nError: {str(e)}\n")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

## Download

You can download the script here: [conversation_with_brendan.py](/tmp/conversation_with_brendan.py)

