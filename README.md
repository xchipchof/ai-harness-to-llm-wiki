# Research AI harness to feed project based LLM-wiki

The concept is based on the following concept:
Studiying the domain of a problem or an environment can be tricky, or time consuming for small personal projects.

In my case, i have a project idea with the goal of learning Rust and constraint based programming. The project is a calculator which employs Rust's
type system to enforce the domain restrictions. Thus i would need to have at least a deep understanding of the domain (definitions, relations, theorems, etc for the many math fields there are).

Hence, after seeing a work colleague using a similar system to gain deep knowledge on a subject i thought about building an LLM-wiki. Since the feeding of the wiki and the definition of the foundations is the most critical, yet strongly structured i decided to architecture an AI harness to deterministically reproduce the research of items of the domain, and the feeding of the LLM-wiki.
