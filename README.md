# agent-dev
## API provider abstraction：
###  context handoff： 在conversage session中，甚至是一个agent loop 执行过程中，切换LLM provider

## Harness（Agent runtime）：
###  split tool results： 一份给LLM，一份给UI显示
###  message queue：在执行agent loop的过程中，允许用户提前发送的message需要queue起来，在合适时机加入一次性全插入，还是一次插入一个，在什么位置插入（tool result 之后，assistant message 之前）

### Context Engineering： 
####  context compression
####  note-writing

# vibe-coding setup
