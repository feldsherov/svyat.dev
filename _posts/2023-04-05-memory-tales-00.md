---
layout: post
title:  "Tales of memory 0x00: Invisible leaks"
date:   2023-04-05 10:00:00 +0003
tags: C++ memory-leak profiling
---

Let's consider situational, which an average C++ developer will meet once in a while.

After a rollout of a new release of an application, instances started to catch OOM, not so often. Every instance OOMs once per several hours. What could it be, and how to deal with it?

## Trivial memory leak

The most boring one, we allocated memory and lost pointer to it. Nowadays it is difficult to meet such a thing in mature production, CI pipeline with address sanitizer will solve the problem. If you don’t have tests against sanitized builds, add them :)

But... You know, tests are tests. Maybe we do not cover this leak in the test.

## Production-only memory leak

Let’s build our component with address sanitizer and deploy one instance to production.

Here are a few things to pay attention to:

1. Trivial but still essential. Do not forget to build the same version as in production :)
2. The address sanitizer build is 3-5% slower than the usually optimized release build. So, if your application is latency sensitive it could be a problem. Another option is to build with leak sanitizer only, which promises to have merely zero overhead in runtime. 
3. Leak checks of address sanitizer sometimes triggered on shutdown only. Stop your application correctly, not

In most cases, it will find the bug.

# The first flavor of invisible leaks

What is a memory leak in terms of memory sanitizer? A leak is a memory that is not freed after shutdown.

But can we imagine OOM when we free all memory eventually? Sure, why not. Here is a "real-world example"

A service was designed to reload configs only during the start. And somewhere in the middle of the code base live class like this:

```cpp
class ConfigStorage() {
	std::vector<Config> storage_;
	
	int Store(Config&& config) {
		const int new_id = storage.size(); 
		storage_.emplace_back(config);
		return new_id;
	}

  // Here some application specific code.
};
```

After a project about improving development speed configs begin to be updated once in several minutes in runtime, without restart. And, nobody found that we never free old config objects until restart.

In this case sanitizers will find nothing because it is more a bug than a leak. The good news is that there is a tool to detect it, but it is a topic for a separate post :)

# The second flavor of invisible leaks

The example above is from a mature project, but still simple to find by reading the code. Let's look into the following snippet.

```cpp
class ConfigStorage() {
	std::vector<Config> storage_;
	
	int GetFreeId();
	void IdFreed(int32_t i);
	
	int Store(Config&& config) {
		const int new_id = GetFreeId(); 
		storage_.emplace_back(config);
		return new_id;
	}
	
	int Release(int id) {
		storage_[id].Clear();  // <- fun stuff could be hidden is here
		IdFreed(id);
	}

  // Here some application specific code.
};
```

Clear often has semantics not releasing the memory, just clearing the object. The is a case for protobuf, std::vector, std::string, easy to invent less popular examples.

If configs are not uniform by size, we can get behavior very similar to a memory leak, as every config object will consume memory for the largest stored object. And memory consumption will grow during the life of the application, exactly like with a leak, but surely will be limited by the max size of a config multiplied by the size of the configs pool.

All the hardest to debug and detect leaks I saw are based on this mechanic. 

Be careful with clear and good luck!

--
Svyat