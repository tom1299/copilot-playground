## Copilot session protocol
See also https://gist.github.com/tom1299/6de030515c438428832c8022bdd4f7cd
Agent pondering about when to update the todo list...
```
$ copilot 
╭──────────────────────────────────────────────────────────────────────────────────────────────────╮
│  ╭─╮╭─╮                                                                                          │
│  ╰─╯╰─╯  GitHub Copilot v1.0.4                                                                   │
│  █ ▘▝ █  Describe a task to get started.                                                         │
│   ▔▔▔▔                                                                                           │
│  Tip: /model Select AI model to use                                                              │
│  Copilot uses AI, so always check for mistakes.                                                  │
╰──────────────────────────────────────────────────────────────────────────────────────────────────╯

● 💡 No copilot instructions found. Run /init to generate a copilot-instructions.md file for this project.

● Environment loaded: 1 MCP server, 1 skill

❯ /hello-world

● skill(hello-world)

◐ I see that the next steps involve using the todos and todo_deps tables to keep track of work. I think it's important to always update todo status while working, even if the current task seems trivial. It might still be helpful to create a todo. I need to call the necessary tools after reporting 
  intent. I can run the "Running hello skill" in a parallel approach with SQL and view script. Let’s go for it!

● Todo added: Run hello-world skill

● Read .agents/skills/hello-world/scripts/hello-world.sh
  └ 7 lines read

◐ I need to run the script now and maybe update my to-do list afterward. I'm considering using parallel execution with bash and SQL, since I want the bash result to confirm success. But I could also run it synchronously with a short wait time. Should I use report_intent? It seems unnecessary if the 
  intent is the same. I think I might just run bash Sequentially since the SQL process depends on its success.

● Running the provided script now.

● Run hello-world skill script
  $ bash ~/git/github/copilot-playground/.agents/skills/hello-world/scripts/hello-world.sh
  └ 2 lines...

◐ I need to mark the todo as done. There’s a thought about using report_intent and maybe changing it to "Finishing hello task," but I'm not sure it’s necessary because I'm in a new phase. I think I can just do a simple SQL update instead. The developer mentioned I should report intent on the first 
  tool call after each user message, but I’ve already done that. So, it might not be needed to do it again.

● Todo completed: run-hello-world-skill

● Done — I invoked the hello-world skill, inspected its script, and ran it successfully.

  Output: Hello World from within the script located at ~/git/github/copilot-playground/.agents/skills/hello-world/scripts

 MCP Servers

 No user-configured servers.

 Built-in:
 ❯ ✓ github-mcp-server  http  https://api.individual.githubcopilot.com/mcp/readonly

 Config:
     ~/.copilot/mcp-config.json
```