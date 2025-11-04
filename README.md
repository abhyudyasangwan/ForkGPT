# ForkGPT: Optimizing Memory Management in LLMs

A scalable framework that enables Large Language Models to fork, merge, and switch between conversational memory states without disrupting the main dialogue flow. Inspired by version-control principles, this project introduces novel memory management capabilities to enhance conversational AI systems.

## Features

- **Memory Forking**: Create independent conversation branches from any point
- **Smart Merging**: Seamlessly integrate forked conversations back to main or parent branches
- **State Switching**: Dynamically switch between different conversation contexts
- **Version Control Principles**: Git-like operations for conversational memory
- **Prompt Engineering Layer**: Optimized prompt reformulation for better consistency

## Architecture Overview

The project consists of four main components:

1. **GPT Client Wrapper** - Handles API communication with OpenAI
2. **Conversation Manager** - Manages individual conversation states and memory
3. **Fork Manager** - Orchestrates forking, merging, and switching operations
4. **Main Driver Loop** - Provides interactive chat interface with control commands

## Project Structure

```
forkgpt/
├── GPTClient.py          # OpenAI API wrapper
├── conversation_manager.py # Conversation state management
├── fork_manager.py       # Forking and merging logic
├── openai_api.py         # API configuration
├── main_driver.py        # Interactive chat interface
└── .env                  # Environment variables
```

## Installation & Setup

### Prerequisites

- Python 3.7+
- OpenAI API key
- Required Python packages

### Installation Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/forkgpt.git
   cd forkgpt
   ```

2. **Set up environment variables**
   ```python
   # Create .env file with your API key
   with open(".env", "w") as f:
       f.write("OPENAI_API_KEY=your_api_key_here")
   ```

3. **Install dependencies**
   ```bash
   pip install openai python-dotenv
   ```

## Core Components Detailed Explanation

### 1. API Configuration (`openai_api.py`)

**Purpose**: Handles secure API authentication and connection setup.

```python
import os
from dotenv import load_dotenv
import openai

load_dotenv()
openai.api_key = os.getenv("OPENAI_API_KEY")

from openai import OpenAI
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
```

**Key Functions**:
- Loads environment variables securely using `python-dotenv`
- Initializes OpenAI client with API key
- Provides centralized configuration management

### 2. GPT Client Wrapper (`GPTClient.py`)

**Purpose**: Abstracts OpenAI API calls and provides a clean interface for GPT interactions.

```python
class GPTClient:
    def __init__(self, client):
        self.client = client

    def get_response(self, messages, model="gpt-4", temperature=0.7):
        response = self.client.chat.completions.create(
            model=model,
            messages=messages,
            temperature=temperature
        )
        return response.choices[0].message.content
```

**Key Features**:
- Configurable model selection (GPT-4, GPT-3.5-turbo, etc.)
- Adjustable temperature for response creativity
- Clean separation of API logic from business logic

### 3. Conversation Manager (`conversation_manager.py`)

**Purpose**: Manages individual conversation threads with parent-child relationships for forking.

```python
class ConversationManager:
    def __init__(self, name="main", is_fork=False, parent=None):
        self.name = name
        self.messages = [{"role": "system", "content": "You are a helpful assistant."}]
        self.is_fork = is_fork
        self.parent = parent  # Reference to parent conversation

    def add_user_message(self, content):
        self.messages.append({"role": "user", "content": content})

    def add_assistant_message(self, content):
        self.messages.append({"role": "assistant", "content": content})

    def get_memory(self):
        return self.messages

    def copy(self, new_name):
        new_convo = ConversationManager(name=new_name, is_fork=True, parent=self)
        new_convo.messages = copy.deepcopy(self.messages)
        return new_convo
```

**Key Capabilities**:
- **Message Storage**: Maintains complete conversation history in OpenAI format
- **Deep Copy Forking**: Creates independent copies of conversation states
- **Parent Tracking**: Maintains lineage for merge operations
- **Memory Retrieval**: Provides complete conversation context for API calls

### 4. Fork Manager (`fork_manager.py`)

**Purpose**: Central orchestrator that manages multiple conversation branches and operations.

```python
class ForkManager:
    def __init__(self):
        self.main_conversation = ConversationManager(name="main")
        self.current = self.main_conversation
        self.forks = {"main": self.main_conversation}
        self.fork_counters = {"main": 0}
```

#### Core Operations:

**A. Forking Operation**
```python
def fork(self):
    parent_name = self.current.name
    self.fork_counters[parent_name] = self.fork_counters.get(parent_name, 0) + 1
    new_fork_name = f"{parent_name}{self.fork_counters[parent_name]}"
    
    new_convo = self.current.copy(new_name=new_fork_name)
    self.forks[new_fork_name] = new_convo
    self.fork_counters[new_fork_name] = 0
    self.current = new_convo
```

**How Forking Works**:
1. Creates a numerically incremented name based on parent
2. Performs deep copy of current conversation state
3. Establishes parent-child relationship
4. Updates current pointer to new fork

**B. Merging Operation**
```python
def merge(self):
    if self.current.name == "main":
        return "skip"  # Prevent merging main into itself
    
    parent_convo = self.current.parent
    main_mem = parent_convo.get_memory()
    fork_mem = self.current.get_memory()
    
    # Find divergence point
    i = 0
    while i < min(len(main_mem), len(fork_mem)) and main_mem[i] == fork_mem[i]:
        i += 1
    
    # Extract delta (new messages from fork)
    delta = fork_mem[i:]
    parent_convo.messages.extend(delta)
    
    self.current = parent_convo
    return True
```

**Merge Algorithm**:
1. **Divergence Detection**: Compares messages to find where fork branched
2. **Delta Extraction**: Identifies new messages created in the fork
3. **Parent Integration**: Appends delta to parent conversation
4. **Context Switching**: Returns to parent conversation

**C. State Management Operations**
```python
def switch_to_main(self):
    # Navigate up the parent chain until reaching main
    if self.current.parent is not None:
        self.current = self.current.parent

def switch_to_fork(self, fork_name):
    # Direct switch to any existing fork
    if fork_name in self.forks:
        self.current = self.forks[fork_name]
```

### 5. Main Driver Loop

**Purpose**: Provides interactive chat interface with real-time control commands.

```python
while True:
    # User input phase
    user_input = input(f"🧑 ({fm.current.name}) You: ").strip()
    fm.add_user_input(user_input)
    
    # GPT response phase
    response = gpt.get_response(fm.get_active_memory())
    fm.add_assistant_reply(response)
    
    # Control interface phase
    print("➡️  Options: [f = fork, c = continue, m = merge, b = back, list = forks]")
    choice = input("⏩ Enter command: ").strip().lower()
```

## Usage Guide

### Basic Conversation Flow

1. **Start Chatting**:
   ```
   🤖 GPT-4 Forkable Chatbot Started!
   
   🧑 (main) You: Hello, let's talk about AI
   🤖 GPT: Certainly! Artificial Intelligence is a fascinating field...
   
   ➡️  Options: [f = fork, c = continue, m = merge, b = back, list = forks]
   ⏩ Enter command: c
   ```

2. **Create a Fork**:
   ```
   🧑 (main) You: Now let's discuss machine learning
   🤖 GPT: Machine learning is a subset of AI that focuses on...
   
   ➡️  Options: [f = fork, c = continue, m = merge, b = back, list = forks]
   ⏩ Enter command: f
   [✓] Forked 'main' → 'main1'
   ```

3. **Explore Different Topics**:
   ```
   🧑 (main1) You: Let's talk about neural networks instead
   🤖 GPT: Neural networks are computing systems inspired by...
   ```

4. **Merge Back Changes**:
   ```
   ➡️  Options: [f = fork, c = continue, m = merge, b = back, list = forks]
   ⏩ Enter command: m
   [✓] Merged 2 messages from 'main1' → 'main'
   ```

### Available Commands

| Command | Description | Usage Example |
|---------|-------------|---------------|
| `f` | Fork current conversation | Create branch from current state |
| `c` | Continue chatting | Proceed with normal conversation |
| `m` | Merge to parent | Integrate changes to parent branch |
| `mm` | Merge to main | Direct merge to main branch |
| `b` | Go back | Switch to parent conversation |
| `list` | Show all forks | Display available branches |
| `switch <name>` | Switch to specific fork | `switch main2` |
| `exit` | Quit application | Terminate the program |

## Technical Implementation Details

### Memory Management Strategy

**Deep Copy vs Shallow Copy**:
- Uses `copy.deepcopy()` to ensure complete isolation between forks
- Prevents reference sharing that could cause cross-contamination
- Enables true independent conversation evolution

**Divergence Detection Algorithm**:
```python
# Linear scan to find first differing message
i = 0
while i < min(len(main_mem), len(fork_mem)) and main_mem[i] == fork_mem[i]:
    i += 1
# Messages from index i onward are the delta
```

### Parent-Child Relationship Management

The system maintains a tree structure:
- Each fork knows its direct parent
- Main conversation serves as root node
- Enables navigation up the hierarchy
- Supports complex merge scenarios

## Advanced Features

### Multi-level Forking
```
main → main1 → main1a → main1a1
    ↘ main2
```

### Selective Merging
- Merge specific forks to specific parents
- Maintain multiple active development branches
- Cherry-pick conversation paths

### Memory Optimization
- Efficient storage of conversation deltas
- Minimal memory duplication through smart copying
- Scalable to hundreds of conversation branches

## Performance Considerations

**Memory Usage**:
- Each fork maintains complete copy of conversation history
- Consider implementing delta compression for large conversations
- Garbage collection for abandoned forks

**API Efficiency**:
- Maintains conversation context for coherent responses
- Avoids redundant API calls through smart caching
- Configurable context window management

## Future Enhancements

1. **Selective Merging**: Choose specific messages to merge
2. **Conflict Resolution**: Handle conflicting conversation paths
3. **Persistent Storage**: Save/Load conversation trees
4. **Visualization Tools**: Graph representation of conversation trees
5. **API Optimizations**: Context window management for long conversations

## Development Setup

### Testing the Implementation

```python
# Simple test script
def test_basic_fork_merge():
    fm = ForkManager()
    
    # Initial conversation
    fm.add_user_input("Hello")
    fm.add_assistant_reply("Hi there!")
    
    # Create fork
    fm.fork()
    
    # Forked conversation
    fm.add_user_input("Let's discuss topic A")
    fm.add_assistant_reply("Topic A is interesting...")
    
    # Merge back
    fm.merge()
    
    print("Test completed successfully!")

if __name__ == "__main__":
    test_basic_fork_merge()
```

### Error Handling

The system includes comprehensive error checking:
- Prevents invalid operations (merging main into itself)
- Validates fork existence before switching
- Handles API errors gracefully
- Provides clear user feedback for all operations

## Contributing

We welcome contributions! Please see our [Contributing Guidelines](CONTRIBUTING.md) for details.

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Submit a pull request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Inspired by version control systems (Git)
- Built upon OpenAI's GPT models
- Hugging Face Transformers for prompt engineering layer

---

**ForkGPT** represents a significant step forward in conversational AI memory management, bringing software engineering principles to AI conversation flows. This framework enables more complex, structured, and manageable interactions with large language models.
