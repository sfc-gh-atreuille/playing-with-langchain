# Dostrings for st.cache_resource and st.cache_data

Authors: Johannes Rieke
Modified: January 24, 2023 1:55 AM
Project stage: ✅ Done / launched / published
Related Projects: https://www.notion.so/Next-gen-cache-De-experimentalize-3933a341daa542cb9798596aad7dbeab
Status: Done / Ready / Approved
Type : 📜 Documentation spec

# st.cache_data

### Current

Function decorator to cache function executions.

Cached data is stored in "pickled" form, which means that the return value of a cached function must be pickleable.

Each caller of the cached function gets its own copy of the cached data.

You can clear a cached function's cache with f.clear().

### Proposed

Decorator to cache functions that return data (e.g. dataframe transforms, database queries, ML inference). 

Cached objects are stored in "pickled" form, which means that the return value of a cached function must be pickleable. Each caller of the cached function gets its own copy of the cached data. 

You can clear a function's cache with `func.clear()` or clear the entire cache with `st.cache_data.clear()`. 

To cache global resources, use `st.cache_resource` instead. Learn more about caching at [https://docs.streamlit.io/library/advanced-features/caching](https://docs.streamlit.io/library/advanced-features/caching)

# st.cache_resource

### Current

Function decorator to store cached resources.

Each cache_resource object is shared across all users connected to the app. Cached resources *must* be thread-safe, because they can be accessed from multiple threads concurrently.

(If thread-safety is an issue, consider using `st.session_state` to store per-session cached resources instead.)

You can clear a cache_resource function's cache with f.clear().

### Proposed

Decorator to cache functions that return global resources (e.g. database connections, ML models).

Cached objects are shared across all users, sessions, and reruns. They must be thread-safe because they can be accessed from multiple threads concurrently. If thread safety is an issue, consider using `st.session_state` to store resources per session instead.

You can clear a function's cache with `func.clear()` or clear the entire cache with `st.cache_resource.clear()`. 

To cache data, use `st.cache_data` instead. Learn more about caching at [https://docs.streamlit.io/library/advanced-features/caching](https://docs.streamlit.io/library/advanced-features/caching)