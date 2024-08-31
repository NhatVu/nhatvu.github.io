---
layout: post
title:  "79.  Core Design Principles for Software Developers by Venkat Subramaniam"
date:   2024-08-24 03:03:00 +0000
category: technical
---
Keynote for the talk: Core Design Principles for Software Developers by Venkat Subramaniam

### What is a good design? 
- A good design is a design that easy to change
- Almost impossible to get it right at the first time, we need many iterations
- Better software quality: need time and revisit/evolve over time
- Software is always evolve 

### How to evaluate quality of design?

### How to create good design? 
- The first step, let go of the ego, the dinasour people, feeling like my design, ...
- The second step, be unemotional. People can challenge each other, and come with better idea 
There are two kinds of people that are dangerous to work with: 
1. Who can't follow instructions 
2. Who can only follow instruction 

- 3rd, take time to review design and code. It's extremely important
two benefits:
- learn from other design and code
- help them to better at design and code 

### KISS principle: Keep it simple, stupid
- The thing we should fear the most is: complexity 
- Complexity accumulates and increases so quickly, until we can't manage it, like Compounding Interest
- The more complexity, the more difficult to modify/change code 
- Simple keep you focused 
- Simple solves only real problem we know about. Should have a clear requirement before writing code 
- Simple is easier to understand

### Complexity 
#### Inherent complexity 
(Phức tạp từ bản chất)
- The complxity comes from problem domain. 
- Nothing you can do about it 

#### Accidential complexity 
- Most complexity is coming from accidential complexity
- Accidential complexity is often come from the solution that we use to solve the problem
- To solve 1 solution, we bring some complexities. To solve those complexities, we bring some other solutions. And those solutions, bring some other complexities, and so on.
- Example: if you introduce concurrency, we need to to deal with threads. And make sure threads don't collide each other, data sharing is correct, no race condition, synchronization, locking, ... 
- Need frequently ask this question: what is the really problem that I want to solve? 
- **Simple is not neccessarily familiar**. Simple is something that easier to understand. Familiar is something you know too well and you don't want to know anything more above it. 

Example:
- Simple: for-loop with 1 iteration variable 
- Familiar: for-loop with 4 iteration variables

A good design is the one that hides inherent complexity and eliminate accidential complexity 

### Think YAGNI - You aren't gonna need it
- You aren't gonna need it" (YAGNI) is a principle which arose from extreme programming (XP) that states a programmer should not add functionality until deemed necessary.
- Tomorrow, we will have more data, more information than today. Then, decision made can be more data centric ==> better decision 

https://martinfowler.com/bliki/Yagni.html

### High cohension and Loose coupling
**Cohesion**
- Cohesion refers to the degree to which the elements inside a module belong together.
=> Meaning the code in 1 module is now focus, does 1 thing and 1 thing well => module independent with others => changing module doesn't effect other modules 

**Coupling**
- Coupling refers to the defree of connection between things. Ex: if your code calling database or external api, that's coupling. Coupling is what you depend on. The best thing you can do is to reduce dependencies
- Worst form of coupling: inheritance. 
- Tight coupling: depends on class. Loose coupling: depends on interface

A good design is a design with high cohesion and loose coupling.

### DRY : Don't repeat yourself
- **Don't duplicate code and effort**

Duplicate effort: example: 



Ex: change database schema 
- Every piece of knowledge in a system should have a single unambiguous authorative representation 



### References 
1. [Core Design Principles for Software Developers by Venkat Subramaniam](https://www.youtube.com/watch?v=llGgO74uXMI)


