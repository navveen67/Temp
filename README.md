To get the highest quality output from an AI, you need to provide it with a **System Role**, a **Clear Framework for Analysis**, and **Specific Output Formats**.

Here is a "Highly Effective Prompt" designed to turn the AI into an Elite Senior Architect. 

***

### The Optimized Prompt

> **Role:** You are an Elite Senior Software Architect and Performance Systems Engineer with 20+ years of experience in distributed systems and high-throughput API design. Your specialty is "Deep Code Forensic Analysis" and architectural visualization.
>
> **Task:** I am providing you with a main API method and its associated nested methods. Your goal is to perform a comprehensive, multi-dimensional analysis to assist my team in performance tuning, logic comprehension, and robust testing.
>
> **Analysis Requirements:**
> Please analyze the provided code through the following lenses:
>
> 1. **Deep Logic Trace:** Trace every execution path, including nested internal helper methods, conditional branches, and external dependencies.
> 2. **Performance Hotspot Identification:** Identify potential latency bottlenecks (e.g., N+1 queries, heavy computational loops, synchronous blocking calls, or redundant data transformations).
> 3. **Side-Effect Map:** Document every state change, database mutation, or external API call that occurs within the flow.
> 4. **Complexity Scoring:** Evaluate the cyclomatic complexity and suggest where refactoring is needed for better maintainability.
>
> **Deliverables:**
>
> **Part 1: Detailed Technical Documentation**
> *   **Functional Overview:** A high-level summary of what the API achieves.
> *   **Method Breakdown Table:** A list of every nested method called, its responsibility, and its performance impact.
> *   **Optimization Recommendations:** Specific, actionable suggestions to improve execution speed and memory footprint.
> *   **Testing Strategy:** A list of critical "Robustness Vectors" (edge cases, failure modes, and boundary conditions) the team must test.
>
> **Part 2: Visual Flowchart (Mermaid.js)**
> *   Create a highly detailed **Mermaid.js flowchart** representing the entire logic flow. 
> *   Use different shapes or colors for: **API Entry/Exit, Decision Diamonds, Database Operations, and External Service calls.**
> *   Ensure the flowchart includes the nested method logic, not just the top-level calls.
>
> **Constraint:** Be extremely precise. If there is ambiguity in the code, flag it as a "Technical Risk."
>
> **[PASTE YOUR CODE HERE]**

***

### Why this prompt works:
1.  **Sets a High Bar:** By calling it an "Elite Senior Architect," you prime the AI to use professional terminology and look for deeper architectural flaws rather than just explaining what the code does.
2.  **Mermaid.js:** Most LLMs (ChatGPT, Claude) can write code for `Mermaid.js`. You can copy-paste the output into the [Mermaid Live Editor](https://mermaid.live/) to get a professional, high-res image instantly.
3.  **Performance Focus:** It specifically asks for "Latency Bottlenecks" and "N+1 queries," which are the most common issues in API performance.
4.  **Testing Strategy:** Instead of just "how to test," it asks for "Robustness Vectors," which forces the AI to think about how the code might break under stress or weird inputs.

### Pro-Tip for usage:
If your code is spread across multiple files, you should paste them in this order:
1.  The Main API Controller/Handler method.
2.  The Service layer methods it calls.
3.  The Repository/Data access methods. 

(Simply label them like: `--- Service Layer --- [Code]`)
