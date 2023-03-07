# Blog post - @Next-gen cache: De-experimentalize

Authors: Ksenia Anske ❄️
Modified: February 10, 2023 2:50 PM
Project stage: ✅ Done / launched / published,✅ Done / launched / published
Related Projects: https://www.notion.so/Blog-post-Introducing-two-new-caching-commands-to-replace-st-cache-1da52f8ead9641b29ae2c6506b6b21d7, https://www.notion.so/Next-gen-cache-De-experimentalize-3933a341daa542cb9798596aad7dbeab
Status: In progress / Draft
Type : 💄 Blog post

<aside>
📌 For reference, our previous blog post on memo/singleton: [https://blog.streamlit.io/new-experimental-primitives-for-caching/](https://blog.streamlit.io/new-experimental-primitives-for-caching/)

</aside>

![two-new-caching-commands.svg](Blog%20post%20-%20@Next-gen%20cache%20De-experimentalize%200cf3a930700248d5bfa39b46d6367635/two-new-caching-commands.svg)

# Introducing two new caching commands to replace `st.cache`!

## `st.cache_data` and `st.cache_resource` are here to make caching less complex and more performant

**Authors: Tim Conkling, Karen Javadyan, Johannes Rieke**

Caching is one of the most beloved *and* dreaded features of Streamlit. We understand why! `@st.cache` makes apps run faster—just slap it on top of a function, and its output will be cached for subsequent runs. But it comes with a lot of baggage: complicated exceptions, slow execution, and a [host of edge cases](https://github.com/streamlit/streamlit/issues?q=is%3Aopen+is%3Aissue+label%3Afeature%3Acache) that make it tricky to use. 😔 

Today, we’re excited to announce two new caching commands… 

`**st.cache_data` and `st.cache_resource` !**

They’re simpler and faster, and they will replace `st.cache` going forward! 👣

## What’s the problem with `st.cache`?

We spent a lot of time investigating it and talking to users. Our verdict: `st.cache` tries to solve too much at once! 

The two main use cases are: 

1. Caching data computations, e.g. when you transform a dataframe, compute NumPy arrays, or run an ML model.  
2. Caching the initialization of shared resources, such as database connections or ML models. 

These use cases require **very** different optimizations. One example:

- For (1), you want your cached object to be safe against mutations. Every time the function is used, the cache should return the same value, regardless of what your app does with it. That’s why `st.cache` constantly checks if the cached object has changed. Cool! But also…slow.
- For (2), you definitely don’t want these mutation checks. If you access a cached database connection many times throughout your app, checking for mutations will only slow it down without any benefit.

There are many more examples where `st.cache` tries to solve both scenarios but turns slow or throws exceptions. So in 2021, we decided to separate these use cases. We released [two experimental caching commands](https://blog.streamlit.io/new-experimental-primitives-for-caching/), `st.experimental_memo` and `st.experimental_singleton`. They work similarly to `st.cache` and are optimized for (1) and (2), respectively. Their behavior worked great, but the names confused the users (especially if they didn’t know the underlying CS concepts of [memoization](https://en.wikipedia.org/wiki/Memoization) and [singleton](https://en.wikipedia.org/wiki/Singleton_pattern)).

## The solution: `st.cache_data` and `st.cache_resource`

Today, we’re releasing our new solution for caching: `st.cache_data` and `st.cache_resource`. These commands have the same behavior as `st.experimental_memo` and `st.experimental_singleton` (with a few additions described below), but should be much easier to understand. 

Using these new commands is super easy. Just add them as a decorator on top of the function you want to cache:

```python
@st.cache_data
def long_running_function():
    return ...
```

Here’s how to use them: 

- `st.cache_data` is the recommended way to cache computations that return data: loading a DataFrame from CSV, transforming a NumPy array, querying an API, or any other function that returns a serializable data object (str, int, float, DataFrame, array, list, and so on). It creates a new copy of the return object at each function call, protecting it from [mutations and concurrency issues](https://docs.streamlit.io/library/advanced-features/caching#mutation-and-concurrency-issues). The behavior of `st.cache_data` is what you want in most cases—so if you're unsure, start with `st.cache_data` and see if it works!
- `st.cache_resource` is the recommended way to cache global resources such as ML models or database connections—unserializable objects that you don't want to load multiple times. By using it, you can share these resources across all reruns and sessions of an app without copying or duplication. Note that any mutations to the cached return value directly mutate the object in the cache.

## New documentation

Along with this release, we’re launching a [brand new docs page](https://docs.streamlit.io/library/advanced-features/caching) for caching. 🚀 It explains in detail how to use both commands, what their parameters are, and how they work under the hood. We’ve included many examples to make it as close to real life as possible. If anything is unclear, please let us know in the comments. ❤️

## Additional features ✨

The behavior of the new commands is mostly the same as `st.experimental_memo` and `st.experimental_singleton`, but we also implemented some highly requested features: 

- `st.cache_resource` now has a `ttl`—to expire cached objects.
- `st.cache_resource` got a `validate` parameter—to run a function that checks whether cached objects should be reused.
- `ttl` now accepts `timedelta` objects—instead of `ttl=604800` you can write `ttl=timedelta(days=7)`.
- Last but not least: cached functions can now contain most Streamlit commands! This powerful feature allows you to cache entire parts of your UI. See details [here](https://docs.streamlit.io/library/advanced-features/caching#using-streamlit-commands-in-cached-functions).

## What will happen to `st.cache`?

Starting with 1.18.0, you’ll get a deprecation warning if you use `st.cache`. We recommend you try the new commands the next time you build an app. They’ll make your life easier and your apps faster. In most cases, it’s as simple as changing a few words in your code. But we also know that many existing apps use `st.cache`, so we’ll keep `st.cache` around, for now, to preserve backward compatibility. Read more in [our migration guide in the docs](https://docs.streamlit.io/library/advanced-features/caching#migrating-from-stcache). 

We’ll also be deprecating `st.experimental_memo` and `st.experimental_singleton`—as they were experimental. The good news is: their behavior is the same as the new commands, so you’ll only need to replace *one name*. 

## The future of caching

We’ll continue to improve caching! The new commands will make it easier to use and understand. Plus, we have more features on our roadmap, such as limiting the amount of memory the cache can use or adding a cache visualizer right into your app.

Track our progress at [roadmap.streamlit.app](https://roadmap.streamlit.app/) and send us [feature requests on GitHub](https://github.com/streamlit/streamlit/issues). And if you have any questions or feedback, please leave them in the comments below.

Happy Streamlit-ing! 🎈

---

- Version 2
    
    ![Announcement (2).svg](Blog%20post%20-%20@Next-gen%20cache%20De-experimentalize%200cf3a930700248d5bfa39b46d6367635/Announcement_(2).svg)
    
    # Introducing two new caching commands to replace `st.cache`!
    
    ## `st.cache_data` and `st.cache_resource` are here to make caching less complex and more performant
    
    **Authors: Tim Conkling, Karen Javadyan, Johannes Rieke**
    
    [Caching](https://docs.streamlit.io/library/advanced-features/caching) is one of the most beloved *and* dreaded features of Streamlit. We understand why! `@st.cache` makes apps run faster—just slap it on top of a function, and its output will be cached for subsequent runs. But it comes with a lot of baggage: complicated exceptions, slow execution, and a [host of edge cases](https://github.com/streamlit/streamlit/issues?q=is%3Aopen+is%3Aissue+label%3Afeature%3Acache) that make it tricky to use. 😔 
    
    Today, we’re excited to announce two new caching commands… 
    
    `**st.cache_data` and `st.cache_resource` !**
    
    They’re simpler and faster, and they will replace `st.cache` going forward! 👣
    
    ## What’s the problem with `st.cache`?
    
    We spent a lot of time investigating it and talking to users. Our verdict: `st.cache` tries to solve too many use cases! 
    
    The two main use cases are: 
    
    1. Caching data computations, e.g., when you transform a dataframe, compute NumPy arrays, or run an ML model.  
    2. Caching the initialization of shared resources, such as database connections or ML models. 
    
    These use cases require **very** different optimizations:
    
    - For (1), you want your cached object to be safe against mutations. Every time the function is used, the cache should return the same value, regardless of what your app does to it. That’s why `st.cache` constantly checks if the cached object has changed. Cool! But also…slow!
    - For (2), you definitely don’t want these mutation checks! You might access a cached database connection many times throughout your app, so checking for mutations every time only slows down the app, without any upside.
    
    There are many similar examples where `st.cache` tries to solve both scenarios but ends up being slow or throwing exceptions.  
    
    In 2021, we decided that we needed to separate these two use cases. So we released [two experimental caching commands](https://blog.streamlit.io/new-experimental-primitives-for-caching/), `st.experimental_memo` and `st.experimental_singleton`. They work similarly to `st.cache` but are optimized for (1) and (2), respectively. After testing with users, we found that the behavior of these commands worked great. But the names caused confusion, especially for users who didn’t know the underlying CS concepts of [memoization](https://en.wikipedia.org/wiki/Memoization) and [singleton](https://en.wikipedia.org/wiki/Singleton_pattern). 
    
    ## The solution: `st.cache_data` and `st.cache_resource`
    
    Today, we’re releasing our new solution for caching: `st.cache_data` and `st.cache_resource`. These commands have the same behavior as `st.experimental_memo` and `st.experimental_singleton` (with a few additions described below) but should be much easier to understand. 
    
    Using these new commands is as simple as always. Just add them as a decorator on top of the function you want to cache:
    
    ```python
    @st.cache_data
    def long_running_function():
        return ...
    ```
    
    Here’s how to use each command: 
    
    - `st.cache_data` is the recommended way to cache data computations: loading a dataframe from CSV, transforming a NumPy array, running a database query, returning simple types like string/int/float, or any lists or dicts that contain data objects. It creates a new copy of the data at each function call, making it safe against mutations and race conditions. The behavior of `st.cache_data` is what you want in most cases – so if you're unsure, start with `st.cache_data` and see if it works!
    - `st.cache_resource` is the recommended way to cache global resources like ML models or database/API connections. These resources are shared across all reruns and sessions of your app, without any copying or duplication.
    
    ## New documentation
    
    Together with this launch, we are launching a [brand new docs page](https://docs.streamlit.io/library/advanced-features/caching) for caching. 🚀 It explains in depth how to use both commands, what their parameters are, and how they work under the hood. We included lots of examples to make it as close to real life as possible. And if anything is unclear, let us know in the comments. ❤️
    
    ## Additional features ✨
    
    The behavior of the new commands is mostly the same as `st.experimental_memo` and `st.experimental_singleton`, but we also implemented some highly requested features: 
    
    - `st.cache_resource` now has a `ttl`—to expire cached objects.
    - `st.cache_resource` got a `validate` parameter—to run a function that checks whether cached objects should be reused or not.
    - `ttl` can now accept `timedelta` objects—instead of `ttl=604800` you can now write `ttl=timedelta(days=7)`.
    - Last but definitely not least: cached functions can now contain most Streamlit commands! This is a powerful feature, as it allows you to cache entire parts of your UI. Read more about it in the docs.
    
    ## What will happen to `st.cache`?
    
    Starting in 1.16.0, you’ll see a deprecation warning if you use `st.cache`. We recommend that you try the new commands the next time you build an app. They’ll make your life easier and your apps faster! In most cases, it’s as simple as changing a few words in your code. But we also know that many existing apps use `st.cache`, and we don’t want to break them! That’s why we’re keeping `st.cache` around until we see its usage drop to very low levels or until we release a major version of Streamlit. 
    
    We’re also deprecating `st.experimental_memo` and `st.experimental_singleton`. Since they were marked as experimental, we plan to remove them more quickly. We’ll monitor their usage and plan to make a decision in ~6 months from now (July 2023). The good news is: since their behavior is the same as the new commands, you literally only need to replace *a single name*. 
    
    ## The future of caching
    
    Caching is an important topic in Streamlit, so we’ll continue to make it better. The new commands will make it easier to use and understand. But we have a few more features on our roadmap, such as limiting the amount of memory the cache can use or adding a cache visualizer right to your app. 
    
    You can always track our progress at [roadmap.streamlit.app](https://roadmap.streamlit.app/) or send us [feature requests on GitHub](https://github.com/streamlit/streamlit/issues). 
    
    Happy Streamlit-ing! 🎈
    
- Version 1
    
    ### Picture/GIF of the new app/feature 👇
    
    [https://www.notion.so](https://www.notion.so)
    
    ---
    
    Title: Two caching commands to rule them all 🌋
    
    Subtitle: `st.cache_data` and `st.cache_resource` are here to replace `st.cache`!
    
    Authors: Tim Conkling, Karen Javadyan, Johannes Rieke
    
    ---
    
    [Caching](https://docs.streamlit.io/library/advanced-features/caching) is one of Streamlit’s most loved but also most dreaded features. And we get why! `@st.cache` is an amazing tool to make apps run faster: just slap it on top of a function, and its output will be cached for subsequent runs. But it comes with a ton of baggage: complicated exceptions, slow execution, and a [host of edge cases](https://github.com/streamlit/streamlit/issues?q=is%3Aopen+is%3Aissue+label%3Afeature%3Acache) make it awful to use. 😔 
    
    Today, we are announcing two new caching commands, `st.cache_data` and `st.cache_resource`, that are much simpler to use and will replace `st.cache` going forward! 👣
    
    ## What’s the problem with `st.cache`?
    
    We spent a lot of time investigating and talking to users. Our verdict: `st.cache` tries to solve too many use cases! 
    
    The two major use cases are: 
    
    1. Caching data computations, e.g. when you transform a dataframe, compute numpy arrays, or run an ML model.  
    2. Caching the initialization of shared resources, e.g. database connections or ML models. 
    
    These use cases require **very** different optimizations! E.g., for (1), you want your cached object to be safe against mutations. Every time the function is used, the cache should return the same value, regardless of what your app does to it. That’s why `st.cache` constantly checks if the cached object changed. Cool! …but also: slow! Something you don’t want for (2), where you might access a cached database connection a few times throughout your app. There are lots of similar examples where `st.cache` tries to solve both scenarios but ends up being slow or throwing exceptions.  
    
    In 2021, we decided we need to separate these two use cases. That’s why we [released two experimental caching commands](https://blog.streamlit.io/new-experimental-primitives-for-caching/), `st.experimental_memo` and `st.experimental_singleton`. They work similar to `st.cache` but are optimized for (1) and (2), respectively. After testing with users, we realized that this behavior works great! But the names are way too complicated, especially for users who don’t know the underlying CS concepts of [memoization](https://en.wikipedia.org/wiki/Memoization) and [singleton](https://en.wikipedia.org/wiki/Singleton_pattern). 
    
    ## The solution: `st.cache_data` and `st.cache_resource`
    
    Today, we are releasing our (hopefully!) final solution for caching: `st.cache_data` and `st.cache_resource`. These commands have the same behavior as `st.experimental_memo` and `st.experimental_singleton` (with a few additions described below) but should be much easier to understand. 
    
    Here’s how to use them:
    
    **TODO. Waiting for docs to be finished, then we can use the same language here. I would only put down 3-4 sentences per command, and link to the docs for more.** 
    
    - cache_data to cache data transforms and computations. eg transforming df, calculating with np array, loading dataframe from csv, string manipulations. example. ...
    - cache_resource to cache resources shared across your entire app, eg db connections or models. example. ...
    
    Please make sure to read our brand-new docs that describe the two new commands in a lot more detail. And if anything is unclear, let us know in this forum post! ❤️
    
    ## Additional features ✨
    
    As mentioned above, the behavior of the new commands is mostly equal to `st.experimental_memo` and `st.experimental_singleton`. But while we were at it, we decided to implement a few highly requested features: 
    
    - `st.cache_resource` now has a `ttl`, so you can expire cached objects.
    - `st.cache_resource` got a `validate` parameter, that lets you run a function to check whether cached objects should be reused or not. Great to recreate expired database connections!
    - `ttl` can now accept `timedelta` objects. Instead of `ttl=604800` you can now write `ttl=timedelta(days=7)`.
    - TODO: did we do any work on hashing? need to ask Karen.
    
    ## What happens to `st.cache`?
    
    We will deprecate it slowly. Starting in 1.16.0, you’ll see a deprecation warning in your app if you use `st.cache`. We recommend to try the new commands the next time you build an app! They will make your life much easier and your apps faster. In most cases, it should be as simple as adding one word to your code. But don’t panic! We know that a lot of existing apps are using `st.cache` and we don’t want to break them. We’ll keep it around at least until 2.0. 
    
    We are also deprecating `st.experimental_memo` and `st.experimental_singleton`. Since they were marked as experimental, we plan to remove them earlier, in 6 months from now (i.e. July 2023). We will monitor how often they are used and might adapt our plans if we need to. And the good news: since their behavior is equal to the new commands, you literally just need to replace a single name. 
    
    ## The future of caching
    
    Caching is an important topic in Streamlit, so we’ll continue to make it better! We think that the new commands will already make it much easier to use and understand. But we have a few more features on our roadmap, e.g. limiting the amount of memory the cache can use or adding a cache visualizer right to your app. You can always follow our current progress on [roadmap.streamlit.app](https://roadmap.streamlit.app/) or send us [feature requests on GitHub](https://github.com/streamlit/streamlit/issues).