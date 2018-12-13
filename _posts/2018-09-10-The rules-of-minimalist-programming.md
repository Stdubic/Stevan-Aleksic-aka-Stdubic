---
layout: post
---




Less is often better. In mathematics, physics, and arts, simplifying and shedding every bit of complexity and redundancy have produced remarkable results. It leads to abstraction, elevates expressiveness, and reveals patterns that are otherwise buried in details.

Programming is no different. For a developer that looks for correctness (does my program behave as expected?), efficiency (does my program use CPU and memory resources appropriately?), and robustness (can I reuse my program or extend it easily for future applications?), minimalism is a sure guideline. The beauty of simplicity translates into better programming.

- The less code, the less bugs. Reduce the size of your code with factorization, dead code removal, and the use of standard libraries.
- Keep function size small. If it doesn’t fit on your screen, it is probably too complex.
- Keep the depth of control structure small. Too many imbricated conditional and loop statements are hard to understand.
- Minimize accessibility. Make your variables and functions private or protected whenever possible.
- Minimize the lifetime of variables. Use persistent data as the last resort.
- Minimize the scope of variables. This makes reading the code simpler. And it helps the compiler to better optimize the code.
- Make the API as simple as possible. It helps eliminating redundancy and strengthening semantics.
- Use meaningful, standardized names for classes, functions, and variables. Just reading the name of a function should be enough to understand what it does.
- Clearly differentiate which data is mutable and which is not in the scope of a function. Use const declaration whenever possible.

Now go and rake your Zen garden.
