# Product spec v2 - @Next-gen cache: De-experimentalize

Authors: Johannes Rieke
Modified: January 12, 2023 11:34 AM
Project stage: ✅ Done / launched / published
Related Projects: https://www.notion.so/Next-gen-cache-De-experimentalize-3933a341daa542cb9798596aad7dbeab
Status: Done / Ready / Approved
Type : 👟 Draft product spec

# Reviewers

- Instructions
    
    If your name appears in the table below, your ✅ is required before we implement this spec. You are welcome to leave more feedback at the bottom of this page!
    
    ✅ = Approve (”I’m fine with the DRI making any further decisions based on this spec.”)
    
    ❌ = Request changes (”I want this spec to be improved and then review again. I added my feedback at the bottom.”)
    

| ✅ / ❌ | Name |
| --- | --- |
| ✅ | @Johannes Rieke  |
| ✅ | @Joshua Carroll  |
| ✅ | @Amanda Kelly ❄️  |
| ✅ | @Thiago Teixeira  |
| ✅ | @Adrien Treuille  |
| ✅ | @Tim Conkling ❄️  |
| ✅ | @Ken McGrady ❄️  |

<aside>
📌 December 20, 2022 
- Will show deprecation warnings in the app when developing (=app is accessed on localhost), and in the logs when deployed. 
- Removed some of warnings around reusing parameters of the old commands in the new commands. 
- Decided to not have a default `ttl`. This doesn’t actually help us with memory, since expired values only get cleared when the function is rerun. Instead, we’ll do [Cache invalidation by actual memory used](https://www.notion.so/Cache-invalidation-by-actual-memory-used-a991dc40f2b2442082cf767f3674b330) during the next few months. 
- Removed `suppress_st_warning`.

</aside>

# What

In this spec, I propose that we solve caching by having two decorators:

- `st.cache_data` (which has the `memo` behavior).
- `st.cache_resource` (which has the `singleton` behavior).

# Why

- See all the other docs.
- This solution should solve most requirements and wishes from everyone. See [Evaluation](https://www.notion.so/Product-spec-v2-62e6fe4ee96d4331a3d338b9b410b6b3) section below.

# How

## New commands

- Add `st.cache_data`, which has the same behavior and parameters as `st.experimental_memo`.
- Add `st.cache_resource`, which has the same behavior and parameters as `st.experimental_singleton`.

## Deprecations

- Deprecate `st.cache`, `st.experimental_memo`, and `st.experimental_singleton`.
- We’ll leave all three commands for a longer time in order to not break apps. The experimental commands can be removed once usage drops significantly – let’s revisit in ~6 months. `st.cache` will be removed in 2.0 at the earliest.
- Deprecation warnings:
    - For `st.cache` if we do **not** show a [recommendation on what to use](https://www.notion.so/Product-spec-v2-62e6fe4ee96d4331a3d338b9b410b6b3):
        
        > `st.cache` is deprecated. Please use one of Streamlit’s new caching commands, `st.cache_data` or `st.cache_resource`. More information [in our docs]().
        > 
    - For `st.cache` if we show a [recommendation on what to use](https://www.notion.so/Product-spec-v2-62e6fe4ee96d4331a3d338b9b410b6b3):
        
        > `st.cache` is deprecated. Please use one of Streamlit’s new caching commands, `st.cache_data` or `st.cache_resource`. Based on this function’s return value of type `foo.bar`, we recommend using `command`. More information [in our docs]().
        > 
    - For `st.experimental_memo`:
        
        > `st.experimental_memo` is deprecated. Please use the new command `st.cache_data` instead, which has the same behavior. More information [in our docs]().
        > 
    - For `st.experimental_singleton`:
        
        > `st.experimental_singleton` is deprecated. Please use the new command `st.cache_resource` instead, which has the same behavior. More information [in our docs]().
        > 
- The latter two should also show up when doing `st.experimental_memo.clear()` or `st.experimental_singleton.clear()`.
- Still need to insert the link to the docs above. Note that this needs to be a very durable URL so it doesn’t break for at least the next 2 years.
- The deprecation warnings should show up in the app when the `“client.showErrorDetails”` config option is `True`, and in the logs otherwise. This should happen for `st.cache`, `st.experimental_memo`, and `st.experimental_singleton`. We can set this up for ALL deprecation warnings, not only the ones introduced in this project.
    - ~~This can come as a fast follow if it pushes out the launch date.~~ (This is done)
    - Note that this is the same behavior as for the developer options in the hamburger menu, see also https://github.com/streamlit/streamlit/issues/3869.
- There’s no way to disable these deprecation warnings. Let’s see if people complain, and if they do, we can still add a config option to disable warnings for `st.cache`.
- If someone uses `st.cache`, add a recommendation to the deprecation warning for which new command to use.
    - We simply decide which command to recommend based on a hard-coded list of return types.
    - See [here](https://www.notion.so/Product-spec-v2-62e6fe4ee96d4331a3d338b9b410b6b3) for the text we’ll use for the recommendation.
    - List of types (using the most common return types tracked by page profiling + what we use in the connection guides in the docs + types from the most common ML libraries):
        - Raw data
            - Most common types for st.cache
                
                ![Untitled](Product%20spec%20v2%20-%20@Next-gen%20cache%20De-experimentali%2062e6fe4ee96d4331a3d338b9b410b6b3/Untitled.png)
                
            - Most common types for st.experimental_memo
                
                ![Untitled](Product%20spec%20v2%20-%20@Next-gen%20cache%20De-experimentali%2062e6fe4ee96d4331a3d338b9b410b6b3/Untitled%201.png)
                
            - Most common types for st.experimental_singleton
                
                ![Untitled](Product%20spec%20v2%20-%20@Next-gen%20cache%20De-experimentali%2062e6fe4ee96d4331a3d338b9b410b6b3/Untitled%202.png)
                
        - Return types where we recommend using `st.cache_data`:
            
            ```python
            - str
            - float
            - int
            - bytes
            - bool
            - datetime.datetime
            - pandas.DataFrame
            - pandas.Series
            - numpy.bool_
            - numpy.bool8
            - numpy.ndarray
            - numpy.float_
            - numpy.float16
            - numpy.float32
            - numpy.float64
            - numpy.float96
            - numpy.float128
            - numpy.int_
            - numpy.int8
            - numpy.int16
            - numpy.int32
            - numpy.int64
            - numpy.intp
            - numpy.uint8
            - numpy.uint16
            - numpy.uint32
            - numpy.uint64
            - numpy.uintp
            - PIL.Image.Image
            - plotly.graph_objects.Figure
            - matplotlib.figure.Figure
            - altair.Chart
            ```
            
        - Return types where we recommend using `st.cache_resource`:
            
            ```python
            - pyodbc.Connection
            - pymongo.mongo_client.MongoClient
            - mysql.connector.MySQLConnection
            - psycopg2.connection
            - psycopg2.extensions.connection
            - snowflake.connector.connection.SnowflakeConnection
            - snowflake.snowpark.sessions.Session
            - sqlalchemy.engine.base.Engine
            - sqlite3.Connection
            - torch.nn.Module
            - tensorflow.keras.Model
            - tensorflow.Module
            - tensorflow.compat.v1.Session
            - transformers.Pipeline
            - transformers.PreTrainedTokenizer
            - transformers.PreTrainedTokenizerFast
            - transformers.PreTrainedTokenizerBase
            - transformers.PreTrainedModel
            - transformers.TFPreTrainedModel
            - transformers.FlaxPreTrainedModel
            ```
            
        - If this takes longer, it can come as a fast follow.
- We should also remove these commands from all guides and explanations, to push people to the new commands. See [Docs](https://www.notion.so/Product-spec-v2-62e6fe4ee96d4331a3d338b9b410b6b3) section below.

## New behavior to make the transition easier

<aside>
🧘‍♀️ These changes are geared at making the transition from `st.cache` to the new commands easier and less frustrating! 15% of all pages use `st.cache` today, so we should make sure the transition is smooth.

</aside>

- Extend the `persist` parameter in `st.cache_data` to accept `True|False` in addition to `“disk”|None`.
    - I.e. don’t break if people use `persist=True` in their `st.cache`’d function and then switch to `st.cache_data`.
    - `True` should just be an alias for `“disk”` and `False` an alias for `None`.
    - Docstring:
        
        > **persist** *(”disk” or boolean or None):* Optional location to persist cached data to. If "disk" or True, data will be persisted to the local disk.
        > 
    - 3.8% of all `st.cache` calls use `persist=True|False` today. So wouldn’t be a major thing if we don’t do it. But I think the API is quite intuitive and Pythonic (`”disk”` is just the default caching location, that you also get with `True`), so I think we should do it.
- Old stuff (will not do)
    - If a user sets `allow_output_mutation` in `st.cache_data` or `st.cache_resource`, show a nice error message telling the user it’s not needed anymore. `allow_output_mutation` is used in 31.4% of all `st.cache` calls. So this should happen fairly often when people switch.
        - Today, calling `st.experimental_memo` with allow_output_mutation shows:
            
            > TypeError: __call__() got an unexpected keyword argument 'allow_output_mutation'
            > 
        - Maybe we turn that into something like:
            
            > TypeError: __call__() got an unexpected keyword argument 'allow_output_mutation' (allow_output_mutation is not needed for st.cache_data and can be safely removed)
            > 
    - If a user sets `persist` in `st.cache_resource`, show a nice error message, similar to above. Maybe something like:
        
        > TypeError: __call__() got an unexpected keyword argument 'persist' (st.cache_resource does not offer persistence since it is for non-serializable resources, only st.cache_data offers persistence)
        > 
    - If a user sets `hash_funcs` in `st.cache_data` or `st.cache_resource`, show a nice error message, similar to above. Maybe something like:
        
        > TypeError: __call__() got an unexpected keyword argument 'hash_funcs' (manual hashing is not supported anymore by st.cache_data but you can disable hashing for an input parameter by prepending an underscore to it)
        > 

## Actual new behavior

<aside>
🌱 There are a few tiny projects that have been lingering for a long time (due to uncertainty around caching). Let’s ship them as part of this project so they don’t get lost + we can announce them in the same blog post (”one more thing”).

</aside>

- Add `ttl` to `st.cache_resource`. Same behavior as for `st.cache_data`. This is something a lot of people asked for, it makes sense, and it’s a small change (eng estimate 1 day), so we should just do it.
- Add a `validate` parameter to `st.cache_resource`.
    - Docstring:
        
        > validate (callable): An optional validation function for cached data. `validate` is called each time the cached value is accessed. It receives the cached value as its only parameter and it must return a boolean. If `validate` returns False, the current cached value is discarded, and the decorated function is called to compute a new value. This is useful e.g. to check the health of database connections.
        > 
    - This is great e.g. for database connections. It’s a pretty obvious thing we should add, so again, let’s just do it. Eng estimate 2 days.
    - Brian Hess already [made a PR for this](https://github.com/streamlit/streamlit/pull/5342). If possible, let’s reuse it!
    - If the `validate` function changes, we do not invalidate the cache. I.e. we will not hash the bytecode of the validate function. We can still do this later on if we see that it’s annoying.
- Re-sort the parameters in `st.cache_data` and `st.cache_resource`.This is just to highlight the most important parameters first. This is a safe operation since they are keyword-only.
    - `cache_data`: `ttl`, `max_entries`, `persist`, `show_spinner`, `experimental_allow_widgets`
    - `cache_resource`: `ttl`, `max_entries`, `show_spinner`, `validate`, `experimental_allow_widgets` (same as cache_resource, but with validate swapped for persist)
- Note that we’re removing the `suppress_st_warning` parameter that we had in the old commands. Almost all st commands can now be properly replayed. There are a few exceptions (e.g. `st.file_uploader`, `st.camera_input`) for which we did not implement replaying, since it’s difficult + some of these widgets are not used a lot. For these commands, we just show the warning in the app, without an option for the developer to turn it off. If people complain, we can find a better solution.

## Out of scope

- [Add hash_funcs to st.cache_data/st.cache_resource](https://www.notion.so/Add-hash_funcs-to-st-cache_data-st-cache_resource-83c06c7efe8749eb946355a33ee29f21)
    - After doing a little research, the vast majority of cases where people use `hash_funcs` today are a) to hash a return object, b) disable hashing, c) actually hash an object but it could be easily handled by disabling hashing, d) hashing a global object. None of these cases is needed anymore with the new caching primitives (and the underscore syntax).
    - There aren’t that many use cases that need hash_funcs. There are some though, see [Add hash_funcs to st.cache_data/st.cache_resource](https://www.notion.so/Add-hash_funcs-to-st-cache_data-st-cache_resource-83c06c7efe8749eb946355a33ee29f21).
    - There have been one or two comments on the forum that we should add `hash_funcs` to memo/singleton. But it wasn’t a huge outcry. We have 0 Github issues on this. And the use cases in the forum were also really specific.
    - So I’d say let’s just wait and see if people complain! We can still add it later on if there’s a need. Created [Add hash_funcs to st.cache_data/st.cache_resource](https://www.notion.so/Add-hash_funcs-to-st-cache_data-st-cache_resource-83c06c7efe8749eb946355a33ee29f21) to keep track of that.
    - (To be clear, I don’t think brining back `hash_funcs` would be so dramatic. But I can see how it might be a tiny bit confusing to have both the underscore syntax and `hash_funcs`).
- [Per-session caching](https://www.notion.so/Per-session-caching-d4719ad0c7be46f59a18fc280796711f)
    
    Unrelated improvement. We might want to have this but it can come as a follow-up. 
    
- [Cache invalidation by actual memory used](https://www.notion.so/Cache-invalidation-by-actual-memory-used-a991dc40f2b2442082cf767f3674b330)
    
    Let’s do this in the next few months. 
    
- [Let ttl accept strings](https://www.notion.so/Let-ttl-accept-strings-a9000aa320ff40bc85ce3df25f857cde)

## How we’ll explain this

<aside>
✏️ I’ll refactor and expand on this section when we start working on docs. Leaving this in for a rough vision on how we’ll explain!

</aside>

I’ll write a bigger draft for how we’ll explain in the docs but TLDR is:

- Use `st.cache_data` to cache any data objects: dataframes, arrays, simple types like str/int/float, or any lists/tuples/dicts that contain such data objects. Under the hood, `st.cache_data` creates a new copy of the data at each rerun, making it safe against mutations and race conditions. Note that this means objects cached with `st.cache_data` must be serializable. This copying behavior is what you want in the majority of use cases – so if you're unsure, start with `st.cache_data` and see if it works!
- Use `st.cache_resource` to cache global resources like ML models or database/API connections. These resources are shared across all reruns and sessions of your app, without any copying or duplication. Note that this means any mutations after the cached function directly mutate the objects stored in the cache!
- Deciding which caching decorator to use is not always trivial. This table maps out some common use cases:
    
    
    | Cached return object | Command |
    | --- | --- |
    | Pandas dataframe | st.cache_data |
    | Numpy array | st.cache_data |
    | Simple types like str/int/float | st.cache_data |
    | Lists/tuples/dicts of the above | st.cache_data |
    | ML models, tokenizers, … | st.cache_resource |
    | Database/API connections | st.cache_resource |
    | Figure objects | Depends! Safer to use st.cache_data. But figure objects are not serializable for some charting libraries, then you’ll need to use st.cache_resource. |
    | Very large arrays/dataframes (> ca 100 million rows) | Can make sense to use st.cache_resource to avoid copying. But make sure the object isn’t mutated then! |
- Advanced section on how both decorators work under the hood, what limitations are, etc. So pro users can really dig in and feel like they 100% understand how everything works.
- We should also update all demo apps to use the new syntax.

## Platform parity

- No clarity yet on how caching and persistence will work on SiS. But according to @Tim Conkling ❄️, this isn’t blocking our work here. We can ship this here, and add a `CachePersistenceManager` interface afterward (similar to what we did for `MediaFileManager` to allow SiS to persist data caches to whatever’s appropriate).

## Risks

- There might be backlash from the community about deprecating `st.cache`. But I think we found a good solution here and we’re doing a bunch of things to make the transition easier, so I’m not too worried!
    - Mitigation: Added an update to the [forum post](https://discuss.streamlit.io/t/we-want-to-deprecate-st-cache-and-need-your-input/30382/27) + creators channel. Feedback was very positive so far, see #st-project-cache-v2

## Metrics

- The new commands need to be tracked in page profiling, which probably requires some updates.
- If that’s done, we should be able to check usage in the page profiling app without any additional work.

## Docs

- We definitely need a big change to the docs. Especially:
    - Rewrite the [docs page on caching](https://docs.streamlit.io/library/advanced-features/caching).
    - Mark the old commands as deprecated in the [API reference section on performance](https://docs.streamlit.io/library/api-reference/performance).
    - Update the docstrings for all caching commands.
    - Remove `st.cache` from all tutorials and guides that we own, and introduce the new commands instead.
- Will work on this with Snehan and share when we have a draft!

## Marketing

- We should definitely do a blog post to a) introduce the new commands and b) explain our reasoning behind deprecation and going away from memo/singleton. I think this was a bit of a confusing journey, so we should explain to users why we did it!
- Framing here will be important, so the community sees this as an actual improvement and not as something annoying.

## QA & release

- This is a pretty fundamental thing so we should take time to test everything properly.

## Eng estimate

- [Tim] 5 engineer weeks
    1. Rename memo/singleton everywhere - 1 week
    2. Deprecate `st.cache`, `st.memo`, `st.singleton` - 2 weeks
        - (This will require some new custom deprecation logic, so I’m being conservative)
    3. Special-case various “unsupported” flags into `st.cache_data`/`st.cache_resource` - 1 week
    4. `st.cache_resource`: `ttl` + `validation` - 1 week
    - 1-3 should be done by the same engineer. 4 can be parallelized if we care!

# Evaluation

## Why I like this solution

- Two separate decorators. Allows us to separate the concepts very clearly, and have different parameters. **[Tim is happy! 🎊]**
- We don't reuse `st.cache` in weird/breaking ways. **[Adrien is happy! 🍀]**
- The names are relatively intuitive. A lot better than memo and singleton in my eyes. **[Johannes is happy! 🐞]**
    - There are some edge cases where `data` and `resource` don’t map perfectly (e.g. figure objects). But I think it's fine having a bit less clarity for these small + advanced use cases if this helps us make the most common use cases easier. We can still explain in docs what to use in the more complicated situations.
    - `st.cache_resource` is not *THE* easiest name (or rather, understanding that resource = model/connection needs a few seconds of thinking). But IMO it’s still a lot better than `st.singleton` or `st.cache_reference`.
    - Then again, `st.cache_resource` is quite nice because it’s general. E.g. it maps to tokenizers, file handles, etc better than `st.cache_connection` / `st.cache_model`.
- The names map kinda nicely to how we usually name commands (after the use case, not after the underlying technical concept).
- The names still both contain `cache`, which is cool because:
    - It’s very obvious from the name that a) both commands are related (and just have different behavior under the hood) and b) are related to the old `st.cache`.
    - We can keep explaining the general concept as caching, which is a) what we always did and b) every other relevant Python library basically also explains this as caching. Don’t need to introduce new concepts to the wider community (memoization, singleton).
    - If people see `st.cache` in some old tutorial, and use it in their code, they will at least see the new commands in autocomplete.
    - On the other hand, there is some potential confusion if people want to use `st.cache_data` and see `st.cache` in autocomplete. But even if they do use `st.cache`, we’ll have a good deprecation and educational warning in place.
- The deprecation message for `st.cache` is not going to be super annoying. We can say "hey, we're keeping caching but we're splitting it up into two optimized decorators" vs. introducing the new concepts of memoization and singleton.
- We don’t have to do `st.cache_connection` or `st.cache_model` for now, i.e. there’s no potential confusion with `st.connection`.
    - …but we can do them as follow-up later on if we want to. And it doesn’t conflict with the naming scheme. So let’s ship `st.connection`, think about st.model (or whatever that is going to be), and then we can reconsider if we still need `st.cache_connection` or `st.cache_model` (or something else).
- We’re having the least amount of additional work. **[Ken is happy! 💐]**

## How this solution checks our requirements + goals

In a [recent meeting with Adrien & Thiago](https://www.notion.so/Johannes-Adrien-Thiago-caching-7039771d24524c9ca075b0f73b47e0df), we settled on these 3 rough constraints: 

1. Use two separate decorators → ✅
2. Don’t do st.cache_connection and st.cache_model for now → ✅ (but still option to do them as follow-up if we see a need after st.connection)
3. Don’t reuse st.cache, or at least don’t break its behavior in weird ways. → ✅ 

I [previously defined 8 goals](https://www.notion.so/Revamping-memo-singleton-7b5b22af645d4728a3a7f3ee0c0477ba) that we’re trying to fulfill with caching: 

1. Beginners can use it for basic use cases without full understanding.
    - ✅ The names are very straightforward. They directly map to the most common use cases (”cache a data computation”, “cache loading of a resource”). We can also push people towards starting with `st.cache_data` in most cases, which feels natural (since that’s the primary use case for caching).
2. Intermediate users understand everything after a few hours.
    - ✅ The commands do not explain the underlying behavior directly. But I think there’s not a huge mental step to explain them and to understand them very well. Also, they are not worse than any of the other names we were considering.
3. Experts are not annoyed during coding.
    - ✅ Two decorators, so very easy to switch back and forth.
4. Don’t fail silently.
    - ✅ If you’re using `st.cache_data` on an ML model (which is the main danger to silently accumulate memory and kill the app), then it’s pretty obvious you’re doing something wrong.
5. If something fails, make it easy to proceed.
    - ✅ Easy, just switch to the other decorator.
6. Don’t break existing apps.
    - ✅ We’re not reusing/changing any existing commands.
7. Most users shouldn’t even notice we changed something.
    - ⚠️ All apps that use `st.cache` need to change their code. But I think the new names make the transition seem relatively gentle, versus introducing completely new concepts like memo or singleton, that the user has to spend some time on to understand.
8. Code doesn’t smell. 
    - ✅ Names seem in line with the rest of Streamlit. No weird behavior or other tricks.

# Feedback

- name @date
- @Thiago Teixeira November 29, 2022
    - LGTM except:
        - **P0:** Set ttl=24h by default. Otherwise caching = memory leak. Can’t have that!
            - [Johannes]: Will not do this. Realized with Tim that it doesn’t actually help us with memory. Old values are only cleared by `ttl` when the function runs again. Instead, we’ll do [Cache invalidation by actual memory used](https://www.notion.so/Cache-invalidation-by-actual-memory-used-a991dc40f2b2442082cf767f3674b330) in the next few months.
        - **P0:** The copy for all warnings and docstrings need work. They’re confusing right now.
        - **P1:** We should deprecate experimental_foo more loudly. Show an st.warning in the app and provide a flag to hide the warning. (The st.cache deprecation message can appear in stdout, though)
            - [Johannes]: Decided to always show it in app when developing locally, and in logs when deployed. See above.
        - **P1:** Need a better solution for the `suppress_st_warning` and `experimental_allow_widgets` arguments. If we’re creating new commands, why start them off with all this baggage? For example, could replace it with `cache_widgets=True|False`
            - [Johannes]: Done. We removed `suppress_st_warning`.
            
            *— non-blockers —*
            
        - **P2:** TTL should support time deltas or even simple strings (”24h” or “1d” to mean 24*60*60). They’re too annoying to use today.
            - [Johannes] Timedelta already works. For string, maybe we should use [pytimeparse](https://github.com/wroberts/pytimeparse)? Will put on task bucket list.
- @Joshua Carroll November 23, 2022
    
    Looks good! ✅
    
    Highlighting from my comments above, key things to consider (non-blocking):
    
    - Can we put this out to the community ASAP once the spec is signed off, so we have more time to navigate any major feedback or backlash? Maybe at least post in Creators and Discord if not a full on thing? Might also give folks an opportunity to feel more engaged about it.
    - for `validate` parameter I really like it and don’t want to slow down momentum, I think it could be aligned with st.connection concept and we should consider “is it healthy?” vs “is it valid?” as the operative question (particularly if this is mostly for connection objects)
- @Brian Hess November 23, 2022
    - Have we considered using `[st.cache.data](http://st.cache.data)` and `st.cache.resource`? Basically, making `st.cache` a namespace that we can put these 2 in. It would also allow for a nice place to put new caching schemes (e.g., `st.cache.model` or `st.cache.connection`).
        - [Johannes]: Yes I’ve considered it briefly. But we don’t use namespacing anywhere else in Streamlit, so we should stay consistent here. *If* we want to change more generally towards namespacing commands, we should have that as a separate discussion (but I can already say that that’s only gonna happen over my dead body as long as I’m PM 😉 I think it will just end up making Streamlit a lot more complicated and less beginner-friendly if everything is namespaced).
    - One thing that will be good/important to highlight about the difference between `st.cache_data` and `st.cache_resource` is the isolation. In `st.cache_resource` if you mutate the returned object, you’re mutating it for all users (silently). In `st.cache_data` you are getting your own object, and making changes to it will not affect anyone else (including yourself if you call that function twice). Sometimes this is good; sometimes it’s not what you’re looking for.
        - [Johannes] 100%. We’ll work on docs in the next few weeks.
    - I made a comment above, but I’ll call it out down here, too. One place that I’ve seen issues with needing to modify the hash is when you create a function that takes a database connection as an argument. So, you have something like `get_data(session:Session, input1:Int)`. The reason is that you want to make sure that you use a particular Session object (e.g., if you are connected to 2 different Snowflake clusters). So, you don’t want to exclude the `session` from the hash, because you want a cache ****miss**** if the `session` is different. The hash you’d want to use is some kind of `session_id` that is gettable from the `session` object.
        - Another use case is to try to align the data access functions with Snowpark Python Stored Procedure format. Snowpark Python Stored Procedures need to have a Snowpark Python Session object as the first argument (this will be magically created by Snowflake). So the signature for the handler is `def handler(session, otherargs...)`. The use case is to use Streamlit to experiment with the functions you want, and then encapsulate that in a Stored Procedure in Snowflake. So this would ease migration (actually, in both directions…).
        - [Johannes]: Ooh that’s a great point!! Yeah that’s definitely a good use case to have hash_funcs.
    - Have we considered offering caching at the session level instead of globally? That is, to allow a user to cache the data simply for themselves, but not for other users. I know that the work around is to use session_state:
        - Such as
            
            ```jsx
            def do_a_thing(a, b, c):
            	if 'magic_string' not in st.session_state:
                st.session_state['magic_string'] = a + b + c
              return st.session_state['magic_string']
            ```
            
        - Versus something like:
            
            ```jsx
            @st.cache_data(global=False)
            def do_a_thing(a, b, c):
            	return a + b + c
            ```
            
            - `global=False` is just a placeholder/example of the syntax.
        - I have run into times where I have wanted the data cached per session. These are use cases like when I have the user provide their database credentials in the app and want to cache that connection locally.
        
        - [Johannes]: Logged in [Per-session caching](https://www.notion.so/Per-session-caching-d4719ad0c7be46f59a18fc280796711f). Not high priority in my eyes but I’ll at least get an eng estimate and will keep it in the back of my mind.

- @Tyler Simons November 17, 2022
- @Joshua Carroll November 17, 2022
    
    ✅ I like it!
    
    - As a follow-on / related, I want to look at our docs, docstrings and online resources and see how we can increase understanding / correct usage from that approach. This may be great to ship in roughly the same time as the release and include in announcement. cc @Snehan Kekre ❄️
        - [Johannes] 100% agree. Requirement for launching this will be that we overhaul a lot of the docs and our getting started guides.
    - Do we have an intention on doing some “generic command under the hood” or keeping these as fully separate code paths?
        - [Johannes] Separate. I don’t think there’s a huge need for a general command if the two specific commands are fairly generic (which I think they are now).
    - For the table of object → command, can we do “Database/API connections → *st.cache_resource (sometimes)”* 😅
        - [Johannes] 😀 Left a note for myself above that we specify this a bit more in the final docs.
- @❄️ Tyler Richards November 17, 2022
    
    I really like this! Happy to bring it down to 2 options. It seems easy to understand, way easier than singleton and memo. Let’s ship something!
    
- @Thiago Teixeira November 17, 2022
    - **About the “Database/API connections” use-case:**
        
        IMO Database/Connections should use st.connection, which will support reconnects, etc. st.cache_resource doesn’t work well for that since it’s immutable.
        
        - [Johannes]: Agree! But we’ll never cover *any* possible connection with st.connection, and it’s going to take considerable time to cover even 80%. IMO we still need this as a fallback. We should of course mention in the docs that st.connection is the preferred solution if you’re using a connection we support.
    - **Naming:** Not a blocker, but I’m not a big fan of the names. cache_data and cache_resource are a bit wordy (hard to type!) and don’t really help me understand what they’re for (”data” and “resource” are both pretty abstract terms. I don’t think people will remember)
        
        So I’m still in the memo/singleton camp.
        
        - But let’s see what the docs look like with these new terms. Maybe it will ******click****** for me then.
        - I’d like to see a table with *all* our primitives. session state, app state (maybe!), connection
- @Zachary Blackwood ❄️ November 17, 2022
    - I really like these names. I also feel like “_resource” is maybe not 100% the best name, but it’s definitely in the right ballpark in my opinion.
    - Love the bonus that people using autocomplete will automatically see the suggestion to use the new names if they’re used to using st.cache