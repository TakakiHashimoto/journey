## **INTRODUCTION**
# REA (Reverse Engineer Anything) — A tool for understanding how software actually works

![GitHub - reaproject/rea: REA Core integration/staging repository · GitHub](https://images.openai.com/static-rsc-4/xAdhSUv8WmAUkvsRjAYwlmrTjeMzxo2I5k895mATF_BI62CkEgxhqnnFhcPZk4KFwuoaedBWwVHYYhcjtKGnxdgOENOzwYKvixaOIx0m7WrO_Ij6HGv8_8npq63pidKxBcZySQkQzNk30vSbag4eh9664AV--dUMwNAvTMJXhSs?purpose=inline)

REA (Reverse Engineer Anything)

Open-source · AI agents · Reverse engineering

[GitHub repository](https://github.com/morluto/rea?utm_source=chatgpt.com) · [Official website](https://rea.tools/?utm_source=chatgpt.com)

REA is a tool that allows AI coding agents to investigate how existing software works, even when you don't have its original source code.

Instead of simply asking an AI to guess how an application was built, you can give it an actual application, executable, or browser page and ask it to investigate the underlying implementation.

For example:

- How does a desktop application's search feature work?
    
- What happens internally when I click a button?
    
- How does an executable process inputs and calculate results?
    
- How does an Electron application communicate between its frontend and backend processes?
    
- Can I understand an existing feature well enough to implement my own version?
    

REA connects AI agents to analysis tools and provides evidence such as assembly instructions, decompiled pseudocode, JavaScript modules, function calls, and observed network requests.

![](https://www.google.com/s2/favicons?domain=https://github.com&sz=32)

GitHub

+1

Why I think it's particularly relevant to you: you're aiming to become an engineer who understands what happens underneath abstractions and eventually builds developer tools, frameworks, and infrastructure.

REA could help you move from reading how systems work to investigating how real systems are constructed.

But there's an important distinction: REA does not magically recover an entire application's original source code or reveal a remote server's private implementation. It helps reconstruct what can be inferred from accessible code, binaries, and observed behavior.

## 1. How REA works under the hood

REA is not itself an AI model. It is an interface between an AI agent and specialized reverse-engineering tools.

You

"Find how this application's export feature works."

AI Coding Agent

Claude Code · Codex · Cursor

Decides what evidence is needed

MCP tool calls

REA — Analysis Orchestrator

Selects tools, coordinates investigations, returns structured evidence

Ghidra / Hopper

Native binaries

JS analysis

Modules, IPC, imports

Browser tools

Requests, scripts, runtime

Target application or executable

Its code, binary data, and observable behavior

MCP means Model Context Protocol. It's a standardized way for an AI application to discover and invoke external tools.

When you ask a question, the agent can call an REA tool to identify executable functions, inspect assembly, follow references, or examine browser requests. REA returns observations, and the AI uses them to explain the system.

The important part is that the agent is working with actual evidence, not merely predicting a plausible implementation.

![](https://www.google.com/s2/favicons?domain=https://github.com&sz=32)

GitHub

+1

### Example: Reverse engineering a compiled program

Imagine you write a small C program:

```
int calculate(int x) {
    if (x > 100) {
        return x * 2;
    }
    return x + 10;
}
```

You compile it into a Linux executable.

The compilation process translates your C source code into machine instructions.

A disassembler might recover instructions conceptually similar to these:

```
cmp edi, 100
jle small_value
lea eax, [rdi + rdi]
ret

small_value:
lea eax, [rdi + 10]
ret
```

Notice what is missing: the original variable names, comments, and C source structure.

A decompiler attempts to reconstruct a higher-level representation of those machine instructions.

REA can then help the agent follow the relevant code, reason about the conditional branch, and explain why certain inputs produce particular results.

This illustrates the central technique:

Executable → machine instructions → reconstructed logic → understanding.

There is no guarantee that the reconstructed source matches what the original developer wrote, but it can still represent the same behavior.

## 2. What can you use REA for?

Here are the applications most relevant to your engineering goals.

|Application|What you could investigate|
|---|---|
|CLI and developer tools|How an executable parses arguments, performs validation, or handles errors|
|Backend and infrastructure|Inspect compiled Go services or trace an application's accessible HTTP behavior|
|Desktop applications|How Electron applications communicate between renderer and main processes|
|Websites|Browser-side JavaScript, API calls, loading behavior, and network activity|
|Security research|Analyze authorized binaries for suspicious behavior, unsafe input handling, or unexpected dependencies|
|Data storage|Examine local SQLite schemas, indexes, and triggers|

REA's current documented capabilities cover all these categories, although particular workflows require additional analysis tools.

![](https://www.google.com/s2/favicons?domain=https://github.com&sz=32)

GitHub

+1

## 3. Three projects I would recommend for you

![Intro to reverse engineering with Ghidra – Cyberbit](https://images.openai.com/static-rsc-4/ocbQRIQaHdGU8j_M5OLykU19IGEdaakIE_lL7i8M4NLtEoYi9gUkH9ZtfQb0bTofCKqqbu2iR5rB5IgjTWIpHnVGY-dFYryZA2dARz7h6IYtjuBR-e9lh_NEcR-ULR0ghKfbUK_GLjBhzSDl-nawv2wJQi4kmhi5GNSCT30lAdE?purpose=inline)

Project A — Reverse engineer your own Go executable

My recommended first systems project

Build a small Go CLI, compile it into a binary, and use REA to investigate its internal functions without opening your original Go source.

Then compare the reconstructed behavior with your actual implementation.

This helps you understand compilation, binary formats, symbols, assembly, and how high-level code becomes executable instructions.

![はじめに｜Electron入門 ~ Webの技術でつくるデスクトップアプリ](https://images.openai.com/static-rsc-4/DbebRxCoIb1FEvb4nMSEyyKncD4nAQqNGDdZeU8uBy5aPQ2JEkkUh5wYPyFUksriWcc1VkH2dHj6jzNu-fFKJMBbKKqIRierQM-moKwa9FJ2GkLXKDfHe1MmE5lHRMm17hr34DKUVhzRweAXdNbOzZJPjGuej_rgSrQYbG8Sg8k?purpose=inline)

Project B — Investigate a real desktop application

Take an Electron application you are permitted to analyze. Investigate how it stores preferences, exports data, or communicates across processes.

REA can help recover module and IPC relationships. You can then build a similar feature independently.

This develops your understanding of application architecture and process boundaries.

![How to Generate a Unique Filename for Uploaded Files | by Ashley | Medium](https://images.openai.com/static-rsc-4/9saslqloTIBp1MtVV-jxvNUTWcOyPyrE8TrWBWf1flxkRoYAd6Q_1hYoCIICuTvUoCTDJk9MenOVimPU32kxplT03WkKPUeYe9PuwebjICnLAJUVdUhd1r7fksZ5lwaim-JgB1_s5VWYS3uobxydO7jD0jiqqT7ooV7fHjpAYTQ?purpose=inline)

Project C — Investigate your own web application's requests

Use REA's browser tools to watch an action in your Explain It Back application.

For instance, trace what happens when a student requests a review question.

Inspect the visible request and response, identify frontend code responsible for the request, and compare your findings with the backend implementation.

An important distinction for Project C: browser analysis shows what happens on the client and what crosses the network. It cannot recover private server-side source code merely by observing the website.

## 4. How to install and start using REA

You can use REA on Ubuntu with a supported Node.js version and a compatible coding agent.

Step 1 — Check your environment

```
node -v
npm -v
```

REA requires Node.js 22.19+, 24.11+, or 26+ (not earlier unsupported versions).

Step 2 — Install REA integration

```
npx rea-agents@latest setup
```

This downloads the package, presents a setup plan, and lets you choose a coding agent such as Codex, Claude Code, or Cursor.

Review the proposed changes, approve them, then restart your agent.

Step 3 — Verify the integration

For example, with Codex:

```
npx -y rea-agents@latest doctor \
  --client codex --json
```

Step 4 — Run your first investigation

The official project provides an Electron example that is suitable for learning.

[Official REA JavaScript and Electron tutorial](https://rea.tools/guides/javascript/?utm_source=chatgpt.com)

You can run:

```
npx -y rea-agents@latest \
  analyze-javascript-application \
  /absolute/path/to/notes-example --json
```

This mode can analyze the example's files without Ghidra or Hopper. For deeper native binary analysis, you will need a supported disassembler or decompiler such as Ghidra.

![](https://www.google.com/s2/favicons?domain=https://rea.tools&sz=32)

REA

+1

Once connected, you can give your agent a prompt such as:

Use REA to investigate this application. Explain the architecture and trace the data flow of its export feature. Show the concrete evidence for each conclusion, including source locations, functions, and process boundaries. Clearly distinguish verified behavior from assumptions. Do not modify the application. After the investigation, explain how I could implement the same functionality independently.

Copy prompt

## 5. What are REA's limitations?

REA is powerful, but the results need careful interpretation.

Analysis is not perfect reconstruction. Optimized binaries can lose meaningful source-level information. A decompiler may recover equivalent control flow while missing the original variable names or data structures.

Static analysis and runtime analysis are different. Static analysis inspects code without executing it. Runtime analysis observes what actually happens during execution. You often need both to establish how a feature works.

Security and permissions matter. Start with your own programs or intentionally provided educational targets. Avoid running unknown executables on your primary machine. The project processes targets locally, but investigation results sent to an AI agent may still be transmitted to its model provider, so be careful with credentials, proprietary code, and sensitive data.

![](https://www.google.com/s2/favicons?domain=https://rea.tools&sz=32)

REA

+1

## 6. How I would incorporate REA into your learning

I would treat REA as a software investigation laboratory, rather than a shortcut that replaces understanding.

Your existing systems roadmap covers programming languages, memory, operating systems, networking, databases, and architecture.

REA gives you another way to investigate each layer. For example, once you understand how function calls and stacks work, inspect a compiled Go binary. Once you understand HTTP, investigate a browser's observable request flow. Once you understand OS processes, investigate Electron IPC.

My strongest recommendation is to start with a tiny program that you wrote yourself, then investigate its compiled output. Your source code becomes the ground truth against which you can verify the AI's explanations.

That makes REA useful not only for reverse engineering, but for learning to verify hypotheses about software using real evidence, which is a fundamental engineering skill.

## **ACTURAL USE**
