---
title: A Lesson from AI Code Review - Exploring Human Psychology
published: 2025-08-08
description: A code review session sparked my new approach toward working with different types of people
image: cover.png
tags: [Technology, Psychology]
category: English
draft: false
---

How the Lesson Started, By Accident
-----------------------------------

It was a regular AI review session in
[a pull request](https://github.com/QubitPi/gearbox/pull/23#discussion_r2260539188) on one of my projects. Little
background - I was implementing a general BFS (Breadth-First Search) algorithm which can be whipped as a JAR library
into one of my developing Java webservices to solve various business problems. To boost its generality, I decided to
implement it using the [Visitor pattern](https://refactoring.guru/design-patterns/visitor), which by design is able to
separate the fixed algorithm and variable business logic and, therefore, make the algorithm applicable to general use
cases. The code below shows the _simplified_ version of my first draft sent to
[gemini-code-assist](https://developers.google.com/gemini-code-assist/docs/review-github-code) for review:

:::tip

- The irrelevant codes, such as Javadoc, `import`s, null-checks, etc., have been removed to highlight the essentials
- The complete conversation between AI and me can be found in the
  [original pull request](https://github.com/QubitPi/gearbox/pull/23)

:::

```java title="NodeVisitor" "void"
public interface NodeVisitor {

    void visit(AbstractNode node);
}
```

```java title="AbstractNode" "void"
public abstract class AbstractNode {
    
    void accept(final NodeVisitor visitor) {
        visitor.visit(this);
    }
}
```

```java title="General BFS implementation" {"AI Review: ⚠️ No early return for large graph dataset":4-13}
public static void traverse(final AbstractNode startNode, final NodeVisitor visitor) {
    final Deque<AbstractNode> queue = new ArrayDeque<>(Collections.singleton(startNode));
    final Set<String> visited = new HashSet<>(Collections.singleton(startNode.getLabel()));

    while (!queue.isEmpty()) {
        final AbstractNode currentNode = queue.removeFirst();
        currentNode.accept(visitor);
        for (final AbstractNode neighbor : currentNode.getNeighbors()) {
            if (!visited.contains(neighbor.getLabel())) {
                visited.add(neighbor.getLabel());
                queue.addLast(neighbor);
            }
        }
    }
}
```

Having analyzed the code above, the Gemini AI review found a performance issue and argued that if `traverse` method is
applied to a large dataset, the `while` loop above could take a lot of time. So the AI figured out a way for
the early termination of the loop with the following review comments:

![](./img/conversation-with-ai.png)

Basically, the AI is asking me to __break the strict Visitor pattern for a performance gain__ by changing the return
type of `NodeVisitor.visit(AbstractNode node)` from `void` to `boolean`. This almost immediately triggered my opposition
and I left my response as follows:

![](./img/conversation-with-ai-2.png)

My response focused on telling AI returning `boolean` in this case was __not a standard__ Visitor practice. The AI,
however, is resisting with the following followup:

![](./img/conversation-with-ai-3.png)

Man what a thoughtful challenge... What's interesting though is I started to feel a common tension in software design:
balancing __strict adherence to a pattern's principles__ with __real-world performance needs__. I keep defending the
`void` return in `visit`

![](./img/conversation-with-ai-4.png)

The story went on...The [original pull request](https://github.com/QubitPi/gearbox/pull/23) had all the followup
details and it's enough for us to transcendentalize the topic:

- The AI really masters communicating _professionally_ in the tech debate, and more importantly
- I started to ask: __what are the two conflicting philosophical or psychological ideologies behind this BFS design
  debate between human and machine?__

Turning Point - From Code Review to Ideologies
----------------------------------------------

The debate over the `void` versus `boolean` return type for a `visit` method in a BFS Visitor pattern implementation,
particularly concerning early termination, actually stems from two conflicting philosophical or psychological ideologies
in software design: Purism (or Idealism) vs. Pragmatism

__Idealism__ champions strict adherence to design patterns and principles (like the _Single Responsibility Principle_
and _Command-Query Separation_). From a purist perspective, the _Visitor_'s sole responsibility is to perform an
operation on the visited element, not to dictate the flow of the traversal. Returning a _boolean_ for control flow
introduces a coupling that compromises the pattern's intended separation of concerns. The traversal algorithm (_BFS_)
should be generic and complete its traversal of reachable nodes, while the _Visitor_ focuses purely on the
element-specific action. This approach values the elegance, maintainability, and long-term extensibility that comes from
clear, decoupled responsibilities.

Idealism favors the perspective of _Generality_, which emphasizes creating reusable, broadly applicable algorithms. The
[`traverse`](#how-the-lesson-started-by-accident) method designed for generality aims to visit all reachable nodes,
regardless of the visitor's internal goal. It's a foundational traversal primitive. Making it stop early based on a
visitor's signal would specialize it into a "search-and-terminate" algorithm, potentially limiting its utility for other
scenarios that genuinely require a full traversal (e.g., calculating graph properties, building a complete map of
connections). This aligns with the idea of building robust, foundational components.

__Pragmatism__, on the other hand, prioritizes practical outcomes and efficiency over strict theoretical purity. A
pragmatic designer would argue that if a slight deviation from a pattern's rigid definition (like returning a `boolean`
from `visit`) leads to significant performance improvements (e.g., avoiding unnecessary computation on large graphs),
it's a worthwhile trade-off. The goal is to build effective software that meets real-world demands, and sometimes that
means adapting patterns to fit specific performance requirements. This approach values utility, performance, and solving
immediate problems efficiently.

Pragmatism possesses the viewpoint of _specific optimization_ that focuses on optimizing for a particular use case, such as finding a
target node as quickly as possible. For this specific scenario, continuing the traversal after the target is found is
clearly inefficient. The argument is that if a common and critical use case (like searching) can be made significantly
faster by a minor adjustment to the API, that optimization should be considered. This prioritizes the performance of a
frequently executed operation over the absolute theoretical purity of the general traversal.

The Two Conflicting Personalities behind the Two Software Design
----------------------------------------------------------------

Different personality types can drive these software design debates as well. In the [context of the BFS visitor pattern
discussion](#how-the-lesson-started-by-accident), two conflicting personalities often emerge: the __Architect__ and the
__Hacker__.

### The Architect 🏛️

The Architect embodies the __Purism/Idealism__ and _Generality_ ideologies. This personality is deeply concerned with
the long-term health, scalability, and maintainability of the codebase.

For the Architect, the Visitor pattern's strength lies in its clear _separation of concerns_. The `visit` method should
only perform an operation on the element, not control the traversal. Deviating from this feels like a compromise of
the pattern's integrity. Architects practices __strict adherence to patterns__. They design the BFS algorithm to be a
truly __general-purpose__ traversal mechanism, capable of being used for any scenario where we need to visit every
reachable node. Optimizing for a single "search" use case by baking termination logic into the `visit` method feels like
specializing a general tool.

Performance optimizations are important, but not at the expense of muddying the design, making it harder to understand,
or limiting future extensibility. Architects argue that for most graphs, the overhead of continuing a few extra
iterations after a target is found is negligible, and for truly massive graphs, a __long-term view__ of more fundamental
architectural changes might be needed anyway. They prioritize __correctness over convenience__

:::tip[Core motivations of Architect]

- elegance
- correctness
- consistency, and
- future-proofing

Architects believe in
[doing things "the right way"](https://ai.qubitpi.org/posts/software-is-about-making-it-right/) according to established
patterns and principles.

:::

### The Hacker 💻

The Hacker (in the positive sense of a clever problem-solver, not a malicious one) embodies the __Pragmatism__ and
_specific optimization_ ideologies. They are driven by getting things done efficiently and solving immediate,
high-impact problems

While they appreciate patterns, they honor __"Get it done" mentality__ and are willing to bend or adapt them if it leads
to a clear and measurable performance gain, especially for common use cases. If a search operation is a critical
bottleneck for large graphs, they put __performance first__ and see the continued traversal after finding the target as
a waste of resources. A `boolean` return that allows the traversal to immediately stop is a direct and effective
solution to this problem. Hackers are more focused on the __short-term impact__ of the immediate, measurable benefits of
an optimization. The "slight deviation" from pattern purity is a small price to pay for a "significant performance
improvement." They prioritize __convenience over correctness__

:::tip[Core motivations of Hacker]

- speed
- efficiency
- tangible results, and
- practical solutions

Hackers are focused on making the current system perform optimally.

:::

What Different Past Personal Experiences Could Lead to Such a Diverted World-View like Architect v.s. Hacker?
-------------------------------------------------------------------------------------------------------------

### Technical

Different past personal experiences often shape the divergent worldviews of the "Architect" and "Hacker" in software
design. These experiences cultivate distinct priorities, risk tolerances, and problem-solving approaches.

:::important

Most people possess traits from both, and their experiences can lead to a more balanced approach over time.

:::

#### The Architect's Formative Experiences 🏛️

The "Architect" mindset is typically forged in environments where __stability__, __scalability__, __long-term
maintainability__, and risk mitigation are paramount.

- __Working on Large, Complex, and Long-Lived Systems__: Think enterprise software, financial systems, aerospace, or
  critical infrastructure. In these domains, a single bug can have catastrophic consequences (financial loss, safety
  hazards). Architects learn the hard way that quick fixes often lead to compounding
  [technical debt](https://neontri.com/blog/enterprise-software-development/#technical-debt) and instability over
  years.
- __Experiencing the Pain of Technical Debt__: They've likely inherited or worked on systems that were initially
  "hacked" together for speed. They've spent countless hours debugging, refactoring, and trying to extend brittle
  codebases, leading to a deep appreciation for upfront design and robust architecture.
- __Involvement in System Outages or Critical Bugs__: Being on call for major incidents caused by poor design, lack of
  testing, or insufficient planning teaches them the severe consequences of prioritizing speed over stability.
- __Formal Education and Mentorship__: Often, Architects have a strong academic background in computer science,
  emphasizing theoretical correctness, algorithms, data structures, and software engineering principles. They may also
  have been mentored by senior engineers who instilled a disciplined approach to design.
- __Working in Regulated Industries__: Industries with strict compliance requirements (e.g., healthcare, finance,
  government) necessitate meticulous design, extensive documentation, and rigorous testing, reinforcing a methodical and
  risk-averse approach.

These experiences instill a deep-seated value for __structure__, __predictability__, and __preventing future problems__,
even if it means a slower initial pace. They see the entire lifecycle of software and understand the compounding cost of
shortcuts.

#### The Hacker's Formative Experiences 💻

The "Hacker" mindset, in its positive sense, often develops in environments that prioritize __speed__, __rapid
iteration__, __immediate impact__, and __direct problem-solving__.

- __Startup or Fast-Paced Environments__: In startups, the primary goal is often to find product-market fit quickly,
  ship features rapidly, and iterate based on user feedback. Survival depends on speed, and "perfect" architecture can
  be seen as a luxury that delays crucial market entry.
- __Being a "Hero" Problem Solver__: They might have gained recognition for quickly fixing urgent production issues,
  implementing features under tight deadlines, or building prototypes that demonstrate immediate value. This reinforces
  the idea that direct action and quick results are highly valued.
- __Self-Taught or Project-Based Learning__: Many "Hackers" are highly skilled self-starters who learned by doing,
  experimenting, and building. Their knowledge is often gained through practical application and overcoming immediate
  technical hurdles, rather than through abstract theoretical study.
- __Projects with Short Lifespans or Disposable Prototypes__: If a project is meant to be a proof-of-concept or has a
  limited expected lifespan, the long-term architectural concerns become less relevant than getting it working quickly.
- __Focus on Direct Performance Gains__: They are often acutely aware of performance bottlenecks and driven to optimize
  code directly for speed, seeing any "unnecessary" abstraction or indirection as a barrier to efficiency.

These experiences cultivate a strong bias towards __action__, __efficiency__, and __delivering tangible results now__.
They are less concerned with hypothetical future problems and more focused on solving the current challenge in the most
direct way possible.

### Psychological

Certain early life experiences and inherent temperaments can also predispose individuals to lean towards an "Architect"
or "Hacker" worldview.

:::important

Early experiences often lay the groundwork for how individuals approach challenges, deal with uncertainty, and
prioritize different aspects of a solution. It is, therefore, important to remember that these are tendencies, not
deterministic paths. Many individuals develop a blend of these traits, and [professional experience](#technical) will
reinforce or modify these predispositions.

:::

#### For the "Architect" Mindset 🏛️

The Architect's preference for structure, planning, and long-term stability often stems from experiences that emphasize
__order__, __consequences__, and __thoroughness__.

There are positive forces that drives one into this direction. Children raised in environments with clear, consistently
enforced rules and predictable consequences may develop a strong internal sense of order and a belief in the importance
of adherence to established guidelines. They learn that following rules prevents negative outcomes and leads to
stability. If parents or caregivers encouraged planning, thinking ahead, and considering the long-term implications of
actions (e.g., "What happens if we don't save money now?"), this can foster a systematic, preventative mindset. Growing
up in a household where tasks were expected to be completed meticulously, with attention to detail and a focus on
durability (e.g., "Do it right the first time, so you don't have to do it again"), can instill a deep appreciation for
robust solutions.

Negative forces can come into play as well. This could be anything from a chaotic household where things frequently went
wrong due to lack of planning, to observing poorly organized projects in school or community activities. Experiencing
the negative consequences of disorder can create a strong desire to build robust, resilient systems. If they frequently
encountered systems or processes that were unreliable, broke easily, or were difficult to understand, it could foster a
drive to design for stability and clarity.

From a personality traits, potentially innate or early-developed, perspective, individuals naturally high in conscientiousness (a Big Five personality trait) tend to be organized, disciplined,
dutiful, and prefer planned behavior. This aligns well with the Architect's need for structure and foresight. A natural
aversion to risk or uncertainty might lead someone to prefer designs that minimize potential failure points and
prioritize predictability. A predisposition for breaking down problems into components, understanding interdependencies, and seeing the "big picture" from an early age (e.g., enjoying complex puzzles, building intricate models) can translate into architectural thinking.

#### For the "Hacker" Mindset 💻

The "Hacker's" drive for rapid iteration, immediate solutions, and direct impact often stems from experiences that reward
__adaptability__, __quick problem-solving__, and __tangible results__.

This mind set usually result from the early experiences in Resource-Constrained or Fast-Paced Environments. Growing up in situations where resources were scarce, and creative, immediate solutions were necessary to overcome challenges, can foster a "hack it together" mentality. They learn to be resourceful and prioritize getting something working over perfect execution. Hobbies like building with LEGOs or playing video games where immediate actions lead to immediate, visible results reinforce a preference for quick iterations, rapid feedback loops and tangible progress. A childhood spent tinkering, disassembling things to see how they work, and experimenting with solutions (even if they were "messy") can lead to a comfort with trial-and-error and a focus on functional outcomes.

Individuals high in openness tend to be curious, inventive, and prefer novelty. This can manifest as a desire to explore new solutions quickly and a willingness to deviate from established norms. A comfort with uncertainty and a willingness to take calculated risks (e.g., trying unconventional solutions, not needing a complete plan before starting) can drive a more experimental, pragmatic approach. A personality that prefers to jump in and start building rather than spending extensive time planning. They learn by doing and iterating. The intense satisfaction of quickly solving a problem or seeing a piece of code immediately work can be a powerful motivator, leading them to prioritize rapid implementation.
