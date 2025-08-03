---
layout: post
title: "95. A quick note about the performance of Java Mapping Framework (Baeldung)"
date: 2025-08-03 08:02:00 +0000
category: technical
---

According to [1], Creating large Java applications composed of multiple layers require using multiple models such as persistence model, domain model or so-called DTOs. Using multiple models for different application layers will require us to provide a way of mapping between beans.

Doing this manually can quickly create much boilerplate code and consume a lot of time. Luckily for us, there are multiple object mapping frameworks for Java.[1]

This article just take a quick note on the performance of Java Mapping frameworks, include MapStruct, ModelMapper, JMapper.

### Benmarks

The average running time per operation (in ms), the less is better:

| Framework Name | Average Running Time (ms per operation) |
| -------------- | --------------------------------------- |
| MapStruct      | 10⁻⁴                                    |
| JMapper        | 10⁻⁴                                    |
| Orika          | 0.007                                   |
| ModelMapper    | 0.137                                   |
| Dozer          | 0.145                                   |

Throughput benmarks: the benchmark returns the number of operations per second (the more is better)

| Framework Name | Throughput (operations per ms) |
| -------------- | ------------------------------ |
| JMapper        | 3205                           |
| MapStruct      | 3467                           |
| Orika          | 121                            |
| ModelMapper    | 7                              |
| Dozer          | 6.342                          |

Personally, I only had a chance to work with MapStruct and ModelMapper. Not working with other mapping framework.

### Conclusion:

- MapStruct is far faster than ModelMapper
- MapStruct use compile-time code generation. The generation code is nearly the same way we write the convertion code manually.
- ModelMapper use reflection, less performance.
- MapStruct is more explicit when define mapping. ModelMapping provide more "intelligent" mapping option, but it can be hard to debug and understand. I'm not prefer it
- Both support nested object mapping

### References

1. [Performance of Java Mapping frameworks](https://www.baeldung.com/java-performance-mapping-frameworks)
