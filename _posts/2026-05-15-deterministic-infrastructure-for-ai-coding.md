---
layout: post
title: "Deterministic Infrastructure for AI-Assisted Coding"
date: 2026-05-15
categories: [coding-practices, ai]
---

Following is an excerpt from Martin Fowler's recent [Fragments post](https://martinfowler.com/fragments/2026-05-14.html) that references also remarks made by James Pritchard [here](https://james-pritchard.com/blog/llms-are-functions):

> Most "agent" use cases are actually workflows, a known sequence of steps where one or two of those steps happen to involve an LLM. You don't need autonomy for that. You need a function call.

> He points out that functions compose predictably, so if you know the workflow, then composing in a program text is better than agents figuring out how to coordinate themselves. It's faster, and needs less tokens. It's usually easier to deal with failures, since the scope of the interaction is smaller.

---

> Pritchard also thinks that people use skills far more than they should. He thinks people accumulate folders of markdown skill files but LLMs use them inconsistently, often missing them when they're needed, or bloating context when they are not. Many things that should go in skills should be other parts of a harness, preferably computational. Skills should only be used with deliberate, infrequent workflows.

> The skills obsession is a symptom of a deeper pattern: people reaching for configuration when they should be reaching for architecture.

Consider that ArchUnit can let you build tests like the following:

```java
@AnalyzeClasses(packages="com.example")
public class SinglePublicMethodRuleTest {

    @ArchTest
    static final var usecases_should_have_only_one_public_method =
        classes()
            .that()
            .resideInAPackage("..usecases..")
            .should(haveOnlyOnePublicMethod())
}
```

The actual rules should of course be specific to your project and modular to the modules within. This kind of stuff often gets leaked to skills files when it can be done perfectly well with deterministic tools.

Keeping code maintainable largely means keeping it comprehensible. Comprehensibility comes with architecture that enforces relationships across "entities" that are easier to reason about. When the entities are code, the architecture includes abstractions, adherence to using terminology from the domain of the system, and style heuristics.

Style heuristics include things like immutability, method lengths, line lengths against nesting, condition lengths before they're refactored into explainer variables, all that makes cyclomatic complexity balloon up. With good style heuristics, you require less cognitive load to build abstractions, and you build better abstractions as a result. With poor style heuristics, your abstractions need to abstract such large surface areas that namespaces tend to collapse, and code reviews become a headache.

When I started agent-assisted coding, I struggled between setting the agent instructions, and wanting it to do things my way in a manner that "skills" was too unreliable for. Your "feedback loop" is not just the sum of tools you have fitted into your agentic workflow, you can also further categorize your prompts broadly into domain-feature/bug resolution/migration instructions (the meat of the chore), and refactoring instructions like how functionality is grouped, how you want variables to be named, etc.

When these style heuristics were violated during my earlier exploration with Claude Code, I thrashed with finding the balance between using the agent for feature work vs refactoring work. Refactoring work felt like it was bogging down the feedback loop, it was pedantic and it felt non-idiomatic.

It felt non-idiomatic because agents should essentially be seen as injections of non-determinism in our systems. Trying to reimagine those computing systems as inherently non-deterministic now because of AI and the need for reinvention is not just terribly misguided, but it also reflects the lack of an engineering intuition and bone in one's perspective.

When we know agents are making our code quality worse, as engineers, our job is break problems down into smaller problems and solve them, not wish for Anthropic to fix it in the next Opus model.

Code style heuristics are like a developer's handwriting, or an organization's culture. They cannot be passed by a singular organization like Anthropic anyway. Abandoning them is not just not necessary, but these heuristics are in fact shortcuts that save extra work from being done. That "non-determinism" (AI generated code) does not need to be corrected in the form of a human feedback loop and those decisions can be automatically enforced by some tests and rules on code "styles" instead, instead of just the code's behavior and execution path.

If these heuristics can be encoded as deterministic tests and rules using ArchUnit and SonarQube, it frees us up into breaking down "coding" itself as a problem and solve those individually with AI. I categorize code, or work that I do in writing features, as dealing with:

- **Architecture** (what goes where and how modularly)
- **Abstractions**
- **Naming** (classes, variables, methods)
- **Performance and security** considerations

For architecture, one can study books, learn from good codebases, study different styles like functional programming, and gain practice and experience at their job for improvement. For better abstractions, one must understand the business domain better. For naming skills, it is simply your skills in the English language, and for performance and security, it is what we learn in academia with respect to algorithms and maths.

In agent assisted coding the same things map to the use of artifacts and tools. The interesting part is not skills, but how well you can inject the context and make use of deterministic tooling to restrain the agent from making mistakes. And it remains work that requires human interaction, comprehension and decision making, that LLMs fundamentally cannot do, and which is what keeps these tasks interesting and within the realm of "engineering".

For architecture, explore ArchUnit. For the business domain, inter-departmental interaction requires human communication for identification and resolution of misunderstandings and scopes of work. The artifact is documentation. How is this going to be injected? Naming is done well enough automatically by the LLM if the architecture is good enough to keep namespaces well spaced. Maybe add some rules about explainer variables and condition complexity in conditional clauses using ArchUnit and SonarQube. For performance and security, again SonarQube can be used, but also other SaaS tools that are in use today by large organizations.

Again, that all these things are yet to become commonplace makes me still feel relevant as a software engineer.
