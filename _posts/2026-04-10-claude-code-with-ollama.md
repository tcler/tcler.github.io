---
layout: post
title: "claude code with ollama"
---

## install ollama and claude
see: https://docs.ollama.com/integrations/claude-code  
It's simple, just install ollama and claude separately:  
```
curl -fsSL https://ollama.com/install.sh | sh
curl -fsSL https://claude.ai/install.sh | bash
```

## lanuch claude
1. use ollama launch sub-command  //recommend
```
ollama launch claude
ollama launch claude --model qwen3-coder:30b
```

2. manually setup  
```
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=""
export ANTHROPIC_BASE_URL=http://localhost:11434
claude --model qwen3-code:30b
```

## tips
1. depending on the differences in the system, you may also need to set up the firewall to allow Ollama's port 11434.  

2. sometimes it may also be necessary to delete the existing Claude configuration files.
