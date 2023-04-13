---
layout: post
title:  "Tales of memory 0x00: Invisible leaks"
date:   2023-04-05 10:00:00 +0003
tags: C++ memory-leak profiling
---

Let's consider a situation that an average C++ developer will encounter once in a while.

After rolling out a new release of an application, instances started to catch OOM, but not so often. Every instance OOMs
once per several hours. What could it be, and how to deal with it?

## Trivial memory leak

The most boring issue is a memory leak. We allocate memory and lose the pointer to it. Nowadays, it is difficult to
encounter such a thing in mature production, as the CI pipeline with address sanitizer will find the problem. If you
don’t have tests against sanitized builds, add them.

But tests are tests, and maybe we do not cover this leak.


## Production-only memory leak

Let's build our component with address sanitizer and deploy one instance to production. Here are a few things to pay
attention to:

1. Don't forget to build the same version as in production.
2. The address sanitizer build is 3-5% slower than the usually optimized release build. So, if your application is
   latency-sensitive, it could be a problem. Another option is to build with leak sanitizer only, which promises to have
   nearly zero overhead in runtime.
3. Leak checks of address sanitizer sometimes triggered on shutdown only. Stop your application correctly, not kill it.

In most cases, it will find the bug.

## The first flavor of invisible leaks

What is a memory leak in terms of memory sanitizer? A leak is a memory that is not freed after shutdown. But can we
imagine OOM when we free all memory eventually? Sure, why not. Here is a "real-world example".

A service was designed to reload configs only during the start. And somewhere in the middle of the code base live class
like this:


```cpp
class ConfigStorage() {
	std::vector<Config> storage_;

	int Store(Config&& config) {
		const int new_id = storage.size();
		storage_.emplace_back(config); // <- code never decrease capacity of storage_
		return new_id;
	}

  // Here some application specific code.
};
```

After a project about improving development speed, configs begin to be updated once in several minutes in runtime,
without restart. And nobody found that we never free old config objects until the restart.

In this case, sanitizers will find nothing because it is more of a bug than a leak. The good news is that there is a
tool to detect it, but it is a topic for a separate post.

## The second flavor of invisible leaks

The example above is from a mature project, but still simple to find by reading the code. Let's look at the following
snippet.

```cpp
class ConfigStorage() {
	std::vector<Config> storage_;

	// Returns id of unused element in vector.
	int GetFreeId();
	// Marks id as unused. After this call, this id can be
	// returned by a consecutive call of GetFreeId.
	void IdFreed(int32_t i);

	int Store(Config&& config) {
		const int new_id = GetFreeId();
		storage_.emplace_back(config);
		return new_id;
	}

	int Release(int id) {
		storage_[id].Clear();  // <- fun stuff could be hidden here
		IdFreed(id);
	}

  // Here some application specific code.
};
```

Clear often has semantics not releasing the memory but just clearing the object. This is a case for protobuf messages,
`std::vector`, `std::string`, easy to invent less popular examples.

If configs are not uniform by size, we can get behavior very similar to a memory leak, as every config object will
consume memory for the largest stored object. And memory consumption will grow during the life of the application,
exactly like with a leak, but surely will be limited by the max size of a config multiplied by the size of the configs
pool.

All the hardest-to-debug leaks I saw are based on this mechanic. Be careful with clear and good luck!


--\\
Svyat