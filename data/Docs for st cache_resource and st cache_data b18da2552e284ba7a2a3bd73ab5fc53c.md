# Docs for st.cache_resource and st.cache_data

Authors: Snehan Kekre ❄️
Modified: February 5, 2023 11:17 PM
Project stage: ✅ Done / launched / published
Related Projects: https://www.notion.so/Next-gen-cache-De-experimentalize-3933a341daa542cb9798596aad7dbeab
Reviewers: Johannes Rieke, Tim Conkling ❄️
Status: Done / Ready / Approved
Type : 📜 Documentation spec

### V2

<aside>
📝 Note: You can find the documentation for the deprecated `st.cache` command in [Optimize performance with st.cache](http://localhost:3000/library/advanced-features/st.cache).

</aside>

# Caching

Streamlit runs your script from top to bottom at every user interaction or code change. This execution model makes development super easy. But it comes with two major challenges:

1. Long-running functions run again and again, which slows down your app. 
2. Objects get recreated again and again, which makes it hard to persist them across reruns or sessions. 

But don’t worry! Streamlit lets you tackle both issues with its built-in caching mechanism. Caching stores the results of slow function calls, so they only need to run once. This makes your app much faster and helps with persisting objects across reruns. 

- Medium old text
    
    But don’t worry! Streamlit has a built-in caching mechanism that lets you speed up long-running functions and retain data persistently. Caching stores the results of expensive function calls and returns the cached result when the same inputs occur again, instead of rerunning the function. This significantly improves the performance of apps. 
    
- Old text
    
    But don’t despair! Streamlit comes with dedicated commands to help in both situations. The Streamlit cache allows your app to stay performant even when loading data from the web, manipulating large datasets, or performing expensive computations. 
    
    The basic idea behind caching is to store the results of expensive function calls and return the cached result when the same inputs occur again rather than calling the function on subsequent runs. Caching can significantly improve the performance of the function, especially if it is called repeatedly with the same inputs.
    

# Table of contents

1. [Minimal example](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
2. [Basic usage](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
3. [Advanced usage](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
4. [Migrating from st.cache](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)

# Minimal example

To cache a function in Streamlit, you must decorate it with one of two decorators (`st.cache_data` or `st.cache_resource`): 

```python
@st.cache_data
def long_running_function(param1, param2):
    return ...
```

In this example, decorating `long_running_function` with `@st.cache_data` tells Streamlit that whenever the function is called, it checks two things:

1. The values of the input parameters (in this case, `param1` and `param2`).
2. The code inside the function.

If this is the first time Streamlit sees these parameter values and function code, it runs the function and stores the return value in a cache. The next time the function is called with the same parameters and code (e.g., when a user interacts with the app), Streamlit will skip executing the function altogether and return the cached value instead. During development, the cache updates automatically as the function code changes, ensuring that the latest changes are reflected in the cache.

As mentioned, there are two caching decorators:

- `st.cache_data` is the recommended way to cache computations that return data: loading a DataFrame from CSV, transforming a NumPy array, querying an API, or any other function that returns a serializable data object (str, int, float, DataFrame, array, list, …). It creates a new copy of the data at each function call, making it safe against [mutations and race conditions](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e). The behavior of `st.cache_data` is what you want in most cases – so if you're unsure, start with `st.cache_data` and see if it works!
- `st.cache_resource` is the recommended way to cache global resources like ML models or database connections – unserializable objects that you don’t want to load multiple times. Using it, you can share these resources across all reruns and sessions of an app without copying or duplication. Note that any mutations to the cached return value directly mutate the object in the cache (more details below).

![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled.png)

# Basic usage

## **st.cache_data**

`st.cache_data` is your go-to command for all functions that return data – whether DataFrames, NumPy arrays, str, int, float, or other serializable types. It’s the right command for almost all use cases!

### Usage

Let's look at an example of using `st.cache_data`. Suppose your app loads the [Uber ride-sharing dataset](https://github.com/plotly/datasets/blob/master/uber-rides-data1.csv) – a CSV file of 50 MB – from the internet into a DataFrame: 

```python
def load_data(url):
    df = pd.read_csv(url)  # 👈 Download the data
    return df

df = load_data("https://github.com/plotly/datasets/raw/master/uber-rides-data1.csv")
st.dataframe(df)

st.button("Rerun")
```

Running the `load_data` function takes 2 to 30 seconds, depending on your internet connection. (Tip: if you are on a slow connection, use [this 5 MB dataset instead](https://github.com/plotly/datasets/blob/master/26k-consumer-complaints.csv)). Without caching, the download is rerun each time the app is loaded or with user interaction. Try it yourself by clicking the button we added! Not a great experience… 😕

Now let’s add the `@st.cache_data` decorator on `load_data`:

```python
@st.cache_data  # 👈 Add the caching decorator
def load_data(url):
    df = pd.read_csv(url)
    return df

df = load_data("https://github.com/plotly/datasets/raw/master/uber-rides-data1.csv")
st.dataframe(df)

st.button("Rerun")
```

Run the app again. You'll notice that the slow download only happens on the first run. Every subsequent rerun should be almost instant! 💨

### Behavior

How does this work? Let’s go through the behavior of `st.cache_data` step by step:

- On the first run, Streamlit recognizes that it has never called the `load_data` function with the specified parameter value (the URL of the CSV file) So it runs the function and downloads the data.
- Now our caching mechanism becomes active: the returned DataFrame is serialized (converted to bytes) via [pickle](https://docs.python.org/3/library/pickle.html) and stored in the cache (together with the value of the `url` parameter).
- On the next run, Streamlit checks the cache for an entry of `load_data` with the specific `url`. There is one! So it retrieves the cached object, deserializes it to a DataFrame, and returns it instead of re-running the function and downloading the data again.

This process of serializing and deserializing the cached object creates a copy of our original DataFrame. While this copying behavior may seem unnecessary, it’s what we want when caching data objects since it effectively prevents mutation and concurrency issues. Read the section “[Mutation and concurrency issues](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)” below to understand this in more detail. 

### Examples

**DataFrame transformations**

In the example above, we already showed how to cache loading a DataFrame. It can also be useful to cache DataFrame transformations such as `df.filter`, `df.apply`, or `df.sort_values`. Especially with large DataFrames, these operations can be slow. 

```python
@st.cache_data
def transform(df):
    df = df.filter(items=['one', 'three'])
		df = df.apply(np.sum, axis=0)
		return df
```

**Array computations**

Similarly, it can make sense to cache computations on NumPy arrays:

```python
@st.cache_data
def add(arr1, arr2):
		return arr1 + arr2
```

**Database queries**

You usually make SQL queries to load data into your app when working with databases. Repeatedly running these queries can be slow, cost money, and degrade the performance of your database. We strongly recommend caching any database queries in your app. See also [our guides on connecting Streamlit to different databases](https://docs.streamlit.io/streamlit-community-cloud/get-started/deploy-an-app/connect-to-data-sources) for in-depth examples. 

```python
connection = database.connect()

@st.cache_data
def query():
    return pd.read_sql_query("SELECT * from table", connection)
```

<aside>
💡 You should set a `ttl` (time to live) to get new results from your database. If you set `st.cache_data(ttl=3600)`, Streamlit invalidates any cached values after 1 hour (3600 seconds) and runs the cached function again. See details in [Controlling cache size and duration](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e).

</aside>

**API calls**

Similarly, it makes sense to cache API calls. Doing so also avoids rate limits. 

```python
@st.cache_data
def api_call():
    response = requests.get('https://jsonplaceholder.typicode.com/posts/1')
    return response.json()
```

**Running ML models (inference)**

Running complex machine learning models can use significant time and memory. To avoid rerunning the same computations over and over, use caching. 

```python
@st.cache_data
def run_model(inputs):
    return model.predict(inputs)
```

## **st.cache_resource**

`st.cache_resource` is the right command to cache “resources” that should be available globally across all users, sessions, and reruns. It has more limited use cases than `st.cache_data`, especially for caching database connections and ML models. 

### Usage

As an example for `st.cache_resource`, let’s look at a typical machine learning app. As a first step, we need to load an ML model. We do this with [Hugging Face’s transformers library](https://huggingface.co/docs/transformers/index):

```python
from transformers import pipeline
model = pipeline("sentiment-analysis")  # 👈 Load the model
```

If we put this code into a Streamlit app directly, the app will load the model at each rerun or user interaction. Repeatedly loading the model poses two problems:

- Loading the model takes time and slows down the app.
- Each session loads the model from scratch, which takes up a huge amount of memory.

Instead, it would make much more sense to load the model once and use that same object across all users and sessions. That’s exactly the use case for `st.cache_resource`! Let’s add it to our app and process some text the user entered:

```python
from transformers import pipeline

@st.cache_resource  # 👈 Add the caching decorator
def load_model():
    return pipeline("sentiment-analysis")

query = st.text_input("Your query", value="I love Streamlit! 🎈")
if query:
    result = model(query)[0]  # 👈 Classify the query text
    st.write(result)
```

If you run this app, you’ll see that the app calls `load_model` only once – right when the app starts. Subsequent runs will reuse that same model stored in the cache, saving time and memory! 

### Behavior

Using `st.cache_resource` is very similar to using `st.cache_data`. But there are a few important differences in behavior: 

- `st.cache_resource` does **not** create a copy of the cached return value but instead stores the object itself in the cache. All mutations on the function’s return value directly affect the object in the cache, so you must ensure that mutations from multiple sessions do not cause problems. In short, the return value must be thread-safe.
    
    <aside>
    ⚠️ Using `st.cache_resource` on objects that are not thread-safe might lead to crashes or corrupted data. Learn more below under [Mutation and concurrency issues](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e).
    
    </aside>
    
- Not creating a copy means there’s just one global instance of the cached return object, which saves memory, e.g. when using a large ML model. In computer science terms, we create a [singleton](https://en.wikipedia.org/wiki/Singleton_pattern).
- Return values of functions do not need to be serializable. This behavior is great for types not serializable by nature, e.g., database connections, file handles, or threads. Caching these objects with `st.cache_data` is not possible.

### Examples

**Database connections**

`st.cache_resource` is useful for connecting to databases. Usually, you’re creating a connection object that you want to reuse globally for every query. Creating a new connection object at each run would be inefficient and might lead to connection errors. That’s exactly what `st.cache_resource` can do, e.g., for a Postgres database:

```python
@st.cache_resource
def init_connection():
    host = "hh-pgsql-public.ebi.ac.uk"
    database = "pfmegrnargs"
    user = "reader"
    password = "NWDMCE5xdipIjRrp"
    return psycopg2.connect(host=host, database=database, user=user, password=password)

conn = init_connection()
```

Of course, you can do the same for any other database. Have a look at [our guides on how to connect Streamlit to databases](https://docs.streamlit.io/streamlit-community-cloud/get-started/deploy-an-app/connect-to-data-sources) for in-depth examples. 

**Loading ML models**

Your app should always cache ML models, so they are not loaded into memory again for every new session. See the [example](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e) above for how this works with 🤗 Hugging Face models. You can do the same thing for PyTorch, TensorFlow, etc. Here’s an example for PyTorch:

```python
@st.cache_resource
def load_model():
		model = torchvision.models.resnet50(weights=ResNet50_Weights.DEFAULT)
		model.eval()
		return model

model = load_model()
```

- Old file handle example
    
    **~~File handles~~**
    
    ~~A more unusual use case is caching file handles. When working with files, it's common to open and close the file frequently. But if the file is very large, it might make more sense to open the file once and reuse the handle throughout the app. You can do this with `st.cache_resource`:~~ 
    
    ```python
    ~~@st.cache_resource
    def open_file():
        return open('large_file.txt', 'r')
    
    f = open_file()
    content = f.read()~~
    ```
    
    ~~In the example above, the `init_file` function opens a file named 'large_file.txt' and returns the file handle. The `@st.cache_resource` decorator is applied to the function, so the first time the function is called, the file handle will be created and the results will be cached. The next time the function is called, the cached file handle will be returned instead of opening the file again.~~
    
- Old callout
    
    <aside>
    ⚠️ ~~It is worth noting that while caching global resources with `st.cache_resource` will improve performance and reduce the resources required by your app, it should be used with care, as it may lead to unexpected behavior if the object is not thread-safe or if it has mutable state. Learn more in [Mutation and concurrency issues](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e).~~
    
    </aside>
    

## **Deciding which caching decorator to use**

The sections above showed many common examples for each caching decorator. But there are edge cases for which it’s less trivial to decide which caching decorator to use. Eventually, it all comes down to the difference between “data” and “resource”: 

- Data are serializable objects (objects that can be converted to bytes via [pickle](https://docs.python.org/3/library/pickle.html)) that you could easily save to disk. Imagine all the types you would usually store in a database or on a file system – basic types like str, int, and float, but also arrays, DataFrames, images, or combinations of these types (lists, tuples, dicts, and so on).
- Resources are unserializable objects that you usually would not save to disk or a database. They are often more complex, non-permanent objects like database connections, ML models, file handles, threads, etc.

From the types listed above, it should be obvious that most objects in Python are “data.” That’s also why `st.cache_data` is the correct command for almost all use cases. `st.cache_resource` is a more exotic command that you should only use in specific situations. 

Or if you’re lazy and don’t want to think too much, just look up your use case or return type in the table below 😉:

| Use case | Typical return types | Caching decorator |
| --- | --- | --- |
| Reading a CSV file with pd.read_csv | pandas.DataFrame | st.cache_data |
| Reading a text file | str, list of str | st.cache_data |
| Transforming pandas dataframes | pandas.DataFrame, pandas.Series | st.cache_data |
| Computing with numpy arrays | numpy.ndarray | st.cache_data |
| Simple computations with basic types | str, int, float, … | st.cache_data |
| Querying a database | pandas.DataFrame | st.cache_data |
| Querying an API | pandas.DataFrame, str, dict | st.cache_data |
| Running an ML model (inference) | pandas.DataFrame, str, int, dict, list | st.cache_data |
| Creating or processing images | PIL.Image.Image, numpy.ndarray  | st.cache_data |
| Creating charts | matplotlib.figure.Figure, plotly.graph_objects.Figure, altair.Chart | st.cache_data (but some libraries require st.cache_resource, since the chart object is not serializable – make sure not to mutate the chart after creation!) |
| Loading ML models | transformers.Pipeline, torch.nn.Module, tensorflow.keras.Model | st.cache_resource |
| Initializing database connections | pyodbc.Connection, sqlalchemy.engine.base.Engine, psycopg2.connection, mysql.connector.MySQLConnection, sqlite3.Connection | st.cache_resource |
| Opening persistent file handles | _io.TextIOWrapper | st.cache_resource |
| Opening persistent threads | threading.thread | st.cache_resource |

- Slides Joshua
    
    Streamlit apps create an interactive app experience from script-like code by running the script top to bottom with the latest inputs on every interaction.
    
    ![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled%201.png)
    
    This works great when your script runs fast. But introduce a long-running operation like a big data load, transformation or slow remote API call, and your app slows to a crawl.
    
    ![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled%202.png)
    
    `@st.cache_data` solves this problem by computing and caching the data output once and just returning it (while skipping the slow part) on future re-runs.
    
    ![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled%203.png)
    
    `@st.cache_resource` solves a different problem, where your app depends on some shared global resource (like a database connection or ML model) that is expensive to initialize and/or maintain over time.
    
    ![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled%204.png)
    
    This problem gets worse when you have multiple users and app sessions each frequently initializing these objects.
    
    ![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled%205.png)
    
    With `@st.cache_resource`, the object—usually a data connection, ML model or similar—is initialized once and then shared as a global object for every re-run and session.
    
    ![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled%206.png)
    
- Old text
    
    Deciding which caching decorator to use is not always trivial. It comes down to understanding the difference between “data” and “resource”. 
    
    One way to conceptualize the difference is to consider a case where your app connects to a database, pulls data from it and writes data back. Pulling and writing stored *serializable* data to and from the database is what you use `@st.cache_data` for, while the *non-serializable* database connection itself should be cached with `@st.cache_resource`.
    
    `@st.cache_resource` is used in a small number of common cases (database connections and ML models are the foremost use-cases; things like file handles, and threads are less common use-cases). `@st.cache_data` is *everything else*.
    
    Please note that the above table is a guideline, and the use case and return type may vary depending on the specific application or scenario. It is worth reiterating that:
    
    - `st.cache_data` is used for serializable objects
    - `st.cache_resource` is used for non-serializable objects such as database connections, ML models, file handles, and threads
    - For very large serializable objects such as arrays/dataframes (> ca 100 million rows) it can make sense to use `st.cache_resource` to avoid copying. But make sure the object isn’t mutated!
    - For other types of return objects that are not listed in the above table, you will have to evaluate them and decide which decorator to use based on the return object's serializability.
- Old table
    
    Here is a table of the most common use cases we’ve observed, their most common return types, and the corresponding caching decorator we recommend using:
    
    | Use case | Return type | Decorator |
    | --- | --- | --- |
    | String literals, user input | str | st.cache_data |
    | Floating-point numbers, numerical computations | float | st.cache_data |
    | Integer values, counters | int | st.cache_data |
    | Binary data, image data | bytes | st.cache_data |
    | Boolean values, flags | bool | st.cache_data |
    | Timestamps, dates | datetime.datetime | st.cache_data |
    | Data manipulation, analysis | pandas.DataFrame | st.cache_data |
    | Data manipulation, analysis | pandas.Series | st.cache_data |
    | Numerical computations, image processing | numpy.ndarray | st.cache_data |
    | Images, image processing | PIL.Image.Image | st.cache_data |
    | Plotly figures | plotly.graph_objects.Figure | st.cache_data |
    | Matplotlib figures | matplotlib.figure.Figure | st.cache_data |
    | Altair figures | altair.Chart | st.cache_data |
    | Database connections | pyodbc.Connection | st.cache_resource |
    | MongoDB connections | pymongo.mongo_client.MongoClient | st.cache_resource |
    | MySQL connections | mysql.connector.MySQLConnection | st.cache_resource |
    | PostgreSQL connections | psycopg2.connection | st.cache_resource |
    | SQLAlchemy connections | sqlalchemy.engine.base.Engine | st.cache_resource |
    | SQLite connections | sqlite3.Connection | st.cache_resource |
    | PyTorch models | torch.nn.Module | st.cache_resource |
    | TensorFlow models | tensorflow.keras.Model | st.cache_resource |
    | HuggingFace models | transformers.Pipeline | st.cache_resource |

# Advanced usage

## Controlling cache size and duration

If your app runs for a long time and constantly caches functions, you might run into two problems:

1. The app runs out of memory because the cache is too large. 
2. Objects in the cache become stale, e.g. because you cached old data from a database. 

You can combat these problems with the `ttl` and `max_entries` parameters, which are available for both caching decorators. 

**The `ttl` (time-to-live) parameter**

`ttl` sets a **t**ime **t**o **l**ive on a cached function. If that time is up and you call the function again, the app will discard any old, cached values, and the function will be rerun. The newly computed value will then be stored in the cache. This behavior is useful for preventing stale data (problem 2) and the cache from growing too large (problem 1). Especially when pulling data from a database or API, you should always set a `ttl` so you are not using old data. Here’s an example:

```python
@st.cache_data(ttl=3600)  # 👈 Cache data for 1 hour (=3600 seconds)
def get_api_data():
    data = api.get(...)
    return data
```

<aside>
💡 Tip: You can also set `ttl` values using `timedelta`, e.g., `ttl=datetime.timedelta(hours=1)`.

</aside>

- Old text
    
    The `ttl` parameter is used to set the maximum number of seconds that a cached resource should be kept in the cache. This is useful for resources that may become stale after a certain amount of time, such as data from a remote API that is only updated every hour. If a resource is requested and it has been in the cache for longer than the specified `ttl`, it will be recomputed and the new value will be stored in the cache. 
    
     ~~It is important to note that the `ttl` parameter is incompatible with `persist="disk"`. If `persist` is specified, the `ttl` parameter will be ignored.~~
    

**The `max_entries` parameter**

`max_entries` sets the maximum number of entries in the cache. An upper bound on the number of cache entries is useful for limiting memory (problem 1), especially when caching large objects. The oldest entry will be removed when a new entry is added to a full cache. Here’s an example:

```python
@st.cache_data(max_entries=1000)  # 👈 Maximum 1000 entries in the cache
def get_large_array(seed):
		np.random.seed(seed)
    arr = np.random.rand(100000)
    return arr
```

~~The default value for `max_entries` is `None`, meaning the cache is unbounded.~~ 

## Customizing the spinner

By default, Streamlit shows a small loading spinner in the app when a cached function is running. You can modify it easily with the `show_spinner` parameter, which is available for both caching decorators: 

```python
@st.cache_data(show_spinner=False)  # 👈 Disable the spinner
def get_api_data():
		data = api.get(...)
    return data

@st.cache_data(show_spinner="Fetching data from API...")  # 👈 Use custom text for spinner
def get_api_data():
    data = api.get(...)
    return data
```

- Old text
    
    The `show_spinner` parameter is used to control the display of a spinner when a "cache miss" occurs and the cached resource is being created. A "cache miss" occurs when the requested resource is not found in the cache and needs to be recomputed. By default, the spinner is enabled (`show_spinner=True`) to indicate to the user that something is happening. If set to `False`, the spinner will not be shown. The parameter also accepts a string, which will be used as the text for the spinner.
    
    Example usage:
    

## Excluding input parameters

In a cached function, all input parameters must be hashable. Let’s quickly explain why and what it means. When the function is called, Streamlit looks at its parameter values to determine if it was cached before. Therefore, it needs a reliable way to compare the parameter values across function calls. Trivial for a string or int – but complex for arbitrary objects! Streamlit uses [hashing](https://en.wikipedia.org/wiki/Hash_function) to solve that. It converts the parameter to a stable key and stores that key. At the next function call, it hashes the parameter again and compares it with the stored hash key. 

Unfortunately, not all parameters are hashable! E.g., you might pass an unhashable database connection or ML model to your cached function. In this case, you can exclude input parameters from caching. Simply prepend the parameter name with an underscore (e.g., `_param1`), and it will not be used for caching. Even if it changes, Streamlit will return a cached result if all the other parameters match up. 

Here’s an example: 

```python
@st.cache_data
def fetch_data(_db_connection, num_rows):  # 👈 Don't hash _db_connection
    data = _db_connection.fetch(num_rows)
    return data

connection = init_connection()
fetch_data(connection, 10)
```

## Using Streamlit commands in cached functions

### **Static elements**

Since version 1.16.0, cached functions can contain Streamlit commands! For example, you can do this:

```python
@st.cache_data  
def get_api_data():
		data = api.get(...)
    st.success("Fetched data from API!")  # 👈 Show a success message
    return data
```

As we know, Streamlit only runs this function if it hasn’t been cached before. On this first run, the `st.success` message will appear in the app. But what happens on subsequent runs? It still shows up! Streamlit realizes that there is an `st.` command inside the cached function, saves it during the first run, and replays it on subsequent runs. Replaying static elements works for both caching decorators. 

You can also use this functionality to cache entire parts of your UI:

```python
@st.cache_data  
def show_data():
    st.header("Data analysis")
		data = api.get(...)
    st.success("Fetched data from API!")
    st.write("Here is a plot of the data:")
    st.line_chart(data)
    st.write("And here is the raw data:")
    st.dataframe(data)
```

### **Input widgets**

You can also use [interactive input widgets](https://docs.streamlit.io/library/api-reference/widgets) like `st.slider` or `st.text_input` in cached functions.  Widget replay is an experimental feature at the moment. To enable it, you need to set the `experimental_allow_widgets` parameter:

```python
@st.cache_data(experimental_allow_widgets=True)  # 👈 Set the parameter
def get_data():
    num_rows = st.slider("Number of rows to get")  # 👈 Add a slider
		data = api.get(..., num_rows)
    return data
```

Streamlit treats the slider like an additional input parameter to the cached function. If you change the slider position, Streamlit will see if it has already cached the function for this slider value. If yes, it will return the cached value. If not, it will rerun the function using the new slider value. 

Using widgets in cached functions is extremely powerful because it lets you cache entire parts of your app. But it can be dangerous! Since Streamlit treats the widget value as an additional input parameter, it can easily lead to excessive memory usage. Imagine your cached function has five sliders and returns a 100 MB DataFrame. Then we’ll add 100 MB to the cache for *every permutation* of these five slider values – even if the sliders do not influence the returned data! These additions can make your cache explode very quickly. Please be aware of this limitation if you use widgets in cached functions. We recommend using this feature only for isolated parts of your UI where the widgets directly influence the cached return value.

<aside>
⚠️ Support for widgets in cached functions is experimental. We may change or remove it anytime without warning. Please use it with care!

</aside>

<aside>
💡 Two widgets are currently not supported in cached functions: `st.file_uploader` and `st.camera_input`. We may support them in the future. Feel free to [open a GitHub issue](https://github.com/streamlit/streamlit/issues) if you need them!

</aside>

- Old text
    
    **The `experimental_allow_widgets` parameter**
    
    Functions decorated with either decorator can also contain [input widgets](http://localhost:3000/library/api-reference/widgets)! Replaying input widgets is disabled by default. To enable it, you can set the `experimental_allow_widgets` parameter for to `True`.
    
    Example usage:
    
    Learn more about widget replay in the [st.cache_data](http://localhost:3000/library/api-reference/performance/st.cache_data) and [st.cache_resource](http://localhost:3000/library/api-reference/performance/st.cache_resource) API.
    
    <aside>
    ⚠️ Setting this parameter to `True` may lead to excessive memory use since the widget value is treated as an additional input parameter to the cache. Every cache function will store a different cache value for every permutation of input parameters (just like any other cache function), AND *every permutation of widget values*. This can blow up your cache size in surprising ways.
    
    </aside>
    

## Dealing with large data

As we explained, you should cache data objects with `st.cache_data`. But this can be slow for extremely large data, e.g., DataFrames or arrays with >100 million rows. That’s because of the [copying behavior](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e) of `st.cache_data`: on the first run, it serializes the return value to bytes and deserializes it on subsequent runs. Both operations take time. 

If you’re dealing with extremely large data, it can make sense to use `st.cache_resource` instead. It does not create a copy of the return value via serialization/deserialization and is almost instant. But watch out: any mutation to the function’s return value (such as dropping a column from a DataFrame or setting a value in an array) directly manipulates the object in the cache. You must ensure this doesn’t corrupt your data or lead to crashes. See the section on [Mutation and concurrency issues](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e) below. 

When benchmarking `st.cache_data` on pandas DataFrames with four columns, we found that it becomes slow when going beyond 100 million rows. The table shows runtimes for both caching decorators at different numbers of rows (all with four columns):

|  |  | 10M rows | 50M rows | 100M rows | 200M rows |
| --- | --- | --- | --- | --- | --- |
| st.cache_data | First run* | 0.4 s | 3 s | 14 s | 28 s |
|  | Subsequent runs | 0.2 s | 1 s | 2 s | 7 s |
| st.cache_resource | First run* | 0.01 s | 0.1 s | 0.2 s | 1 s |
|  | Subsequent runs | 0 s | 0 s | 0 s  | 0 s |

* For the first run, the table only shows the overhead time of using the caching decorator. It does not include the runtime of the cached function itself. 

## Mutation and concurrency issues

in the sections above, we talked a lot about issues when mutating return objects of cached functions. This topic is complicated! But it’s central to understanding the behavior differences between `st.cache_data` and `st.cache_resource`. So let’s dive in a bit deeper. 

First, we should clearly define what we mean by mutations and concurrency: 

- By **mutations**, we mean any changes made to a cached function’s return value *after* that function has been called. I.e. something like this:
    
    ```python
    @st.cache_data
    def create_list():
        l = [1, 2, 3]
    
    l = create_list()  # 👈 Call the function
    l[0] = 2  # 👈 Mutate its return value
    ```
    
- By **concurrency**, we mean that multiple sessions can cause these mutations at the same time. Streamlit is a web framework that needs to handle many users and sessions connecting to an app. If two people view an app at the same time, they will both cause the Python script to rerun, which may manipulate cached return objects at the same time – concurrently.

Mutating cached return objects can be dangerous. It can lead to exceptions in your app and even corrupt your data (which can be worse than a crashed app!). Below, we’ll first explain the copying behavior of `st.cache_data` and show how it can avoid mutation issues. Then, we’ll show how concurrent mutations can lead to data corruption and how to prevent it.

- Old text
    
    It's important to be aware of mutation and concurrency issues when using `@st.cache_resource`, since it creates global resources that are shared across all users, reruns, and sessions. Note that since `@st.cache_data` returns a new copy of its data on each call, using it does not come with the same concerns.
    
    **Mutations** refer to changes made to an object after it has been created. In the context of caching, it means modifying a cached object after it has been retrieved and returned by a function.
    
    **Concurrency** refers to the ability of multiple users or processes to access and make changes to a shared resource simultaneously. When it comes to caching, concurrency issues can arise when multiple users are accessing the same cached object simultaneously and making changes to it.
    
    Mutating a cached resource can lead to unexpected and inconsistent results for different users, as the app will reflect changes made by one user in the resource for all other usersTo understand how mutations may arise, we first need to understand `st.cache_data`'s copying behavior.
    

### ********************************Copying behavior********************************

`st.cache_data` creates a copy of the cached return value each time the function is called. This avoids most mutations and concurrency issues. To understand it in detail, let’s go back to the [Uber ridesharing example](https://www.notion.so/Restructured-based-on-Johannes-TOC-d4f2b2aff3b24f89904f6b9bbde0fd89) from the section on `st.cache_data` above. We are making two modifications to it: 

1. We are using `st.cache_resource` instead of `st.cache_data`. `st.cache_resource` does **not** create a copy of the cached object, so we can see what happens without the copying behavior. 
2. After loading the data, we manipulate the returned DataFrame (in place!) by dropping the column `"Lat"`. 

Here’s the code:

```python
@st.cache_resource   # 👈 Turn off copying behavior
def load_data(url):
    df = pd.read_csv(url)
    return df

df = load_data("https://raw.githubusercontent.com/plotly/datasets/master/uber-rides-data1.csv")
st.dataframe(df)

df.drop(columns=['Lat'], inplace=True)  # 👈 Mutate the dataframe inplace

st.button("Rerun")
```

Let’s run it and see what happens! The first run should work fine. But in the second run, you see an exception: `KeyError: "['Lat'] not found in axis"`. Why is that happening? Let’s go step by step:

- On the first run, Streamlit runs `load_data` and stores the resulting DataFrame in the cache. Since we’re using `st.cache_resource`, it does **not** create a copy but stores the original DataFrame.
- Then we drop the column `"Lat"` from the DataFrame. Note that this is dropping the column from the *original* DataFrame stored in the cache. We are manipulating it!
- On the second run, Streamlit returns that exact same manipulated DataFrame from the cache. It does not have the column `"Lat"` anymore! So our call to `df.drop` results in an exception. Pandas cannot drop a column that doesn’t exist.

The copying behavior of `st.cache_data` prevents this kind of mutation error. Mutations can only affect a specific copy and not the underlying object in the cache. The next rerun will get its own, unmutated copy of the DataFrame. You can try it yourself, just replace `st.cache_resource` with `st.cache_data` above, and you’ll see that everything works. 

Because of this copying behavior, `st.cache_data` is the recommended way to cache data transforms and computations – anything that returns a serializable object. 

### **Concurrency issues**

Now let’s look at what can happen when multiple users concurrently mutate an object in the cache. Let's say you have a function that returns a list. Again, we are using `st.cache_resource` to cache it so that we are not creating a copy: 

```python
@st.cache_resource
def create_list():
    l = [1, 2, 3]
    return l

l = create_list()
first_list_value = l[0]
l[0] = first_list_value + 1

st.write("l[0] is:", l[0])
```

Let's say user A runs the app. They will see the following output:

```python
l[0] is: 2
```

Let's say another user, B, visits the app right after. In contrast to user A, they will see the following output:

```python
l[0] is: 3
```

Now, user A reruns the app immediately after user B. They will see the following output: 

```python
l[0] is: 4
```

What is happening here? Why are all outputs different?

- When user A visits the app, `create_list()` is called, and the list `[1, 2, 3]` is stored in the cache. This list is then returned to user A. The first value of the list, `1`, is assigned to `first_list_value` , and `l[0]` is changed to `2`.
- When user B visits the app, `create_list()` returns the mutated list from the cache: `[2, 2, 3]`. The first value of the list, `2`, is assigned to `first_list_value` and `l[0]` is changed to `3`.
- When user A reruns the app, `create_list()` returns the mutated list again: `[3, 2, 3]`. The first value of the list, `3`, is assigned to `first_list_value,` and `l[0]` is changed to 4.

If you think about it, this makes sense. Users A and B use the same list object (the one stored in the cache). And since the list object is mutated, user A's change to the list object is also reflected in user B's app. 

This is why you must be careful about mutating objects cached with `st.cache_resource`, especially when multiple users access the app concurrently. If we had used `st.cache_data` instead of `st.cache_resource`, the app would have copied the list object for each user, and the above example would have worked as expected – users A and B would have both seen:

```python
l[0] is: 2
```

<aside>
🔥 This toy example might seem benign. But data corruption can be extremely dangerous! Imagine we had worked with the financial records of a large bank here. You surely don’t want to wake up with less money on your account just because someone used the wrong caching decorator 😉

</aside>

# Migrating from st.cache

We introduced the caching commands described above in Streamlit 1.18.0. Before that, we had one catch-all command `st.cache`. Using it was often confusing, resulted in weird exceptions, and was slow. That’s why we replaced `st.cache` with the new commands in 1.18.0 (read more in this blog post). The new commands provide a more intuitive and efficient way to cache your data and resources and are intended to replace `st.cache` in all new development.

If your app is still using `st.cache`, don’t despair! Here are a few notes on migrating:

- `st.cache` is deprecated. • New versions of Streamlit will show a deprecation warning if your app uses it.
- We will not remove `st.cache` soon, so you don’t need to worry about your 2-year-old app breaking. But we encourage you to try the new commands going forward – they will be way less annoying!
- Switching code to the new commands should be easy in most cases. To decide whether to use `st.cache_data` or `st.cache_resource`, read [Deciding which caching decorator to use](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e). Streamlit will also recognize common use cases and show hints right in the deprecation warnings.
- Most parameters from `st.cache` are also present in the new commands, with a few exceptions:
    - `allow_output_mutation` does not exist anymore. You can safely delete it. Just make sure you use the right caching command for your use case.
    - `suppress_st_warning` does not exist anymore. You can safely delete it. Cached functions can now contain Streamlit commands and will replay them. If you want to use widgets inside cached functions, set `experimental_allow_widgets=True`. See [here](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e).
    - `hash_funcs` does not exist anymore. You can exclude parameters from caching (and being hashed) by prepending them with an underscore: `_excluded_param`. See [here](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e).

If you have any questions or issues during the migration process, please contact us on the [forum](https://discuss.streamlit.io/), and we will be happy to assist you. 🎈

- Old FAQ
    
    # Frequently asked questions
    
    1. What is the difference between `st.cache_data` and `st.cache_resource`?
        - `st.cache_data` is used to cache serializable objects such as Pandas DataFrames, NumPy arrays, and simple types like strings, integers, and floats. These objects can be [pickled and unpickled](https://docs.python.org/3/library/pickle.html), so `st.cache_data` creates a new copy of the object from the serialized data and returns it with every run.
        - `st.cache_resource` *must* be used to cache non-serializable objects such as database connections, ML models, file handles, and threads. These objects cannot be serialized, so `st.cache_resource` returns the same object with every run. But in some cases, you *may* want to use `st.cache_resource` for caching very large serializable objects so that your app doesn’t exhaust its memory by creating a copy of it every time the app is rerun.
    2. Can I use `st.cache_data` for non-serializable objects?
        
        No, `st.cache_data` should only be used for serializable objects. If you are working with non-serializable objects, you must should use `st.cache_resource`.
        
    3. Can I use `st.cache_resource` for simple types like strings, integers, and floats?
        
        No, you should use `st.cache_data` for simple types like strings, integers, and floats. `st.cache_resource` is meant for non-serializable objects only. However, in some few cases, it may make sense to use `st.cache_resource` to cache very large serializable objects. But be aware that caching the serializable object as a resource means that it is now a global resource that is shared across all users, reruns, and sessions. You need to be careful about mutating the object, as this will affect all users.
        
    4. What happens if I mutate a cached resource?
        
        If you mutate a cached resource, the change will be reflected in the resource for all users, reruns, and sessions. This can lead to unexpected behavior if multiple users are accessing the app concurrently. To avoid this, you should make sure that the resource is not mutated after it is cached, or make sure that your cached resources are safe to mutate from multiple threads simultaneously.
        

### V1 (deprecated)

<aside>
📝 Note: documentation for the deprecated `st.cache` command can be found in [Optimize performance with st.cache](http://localhost:3000/library/advanced-features/st.cache).

</aside>

# Caching in Streamlit apps

Streamlit runs your script from top to bottom at every user interaction or code change. This is great to make your code simple. But it comes with two major challenges:

1. Long-running functions will run again and again, which slows down your app to a crawl… 😞
2. Objects get reset at each run, making it hard to store data persistently across reruns. 🤯

But don’t despair! Streamlit comes with dedicated commands to help in both situations. The Streamlit cache allows your app to stay performant even when loading data from the web, manipulating large datasets, or performing expensive computations. 

The basic idea behind caching is to store the results of expensive function calls and return the cached result when the same inputs occur again, rather than calling the function on subsequent runs. This can significantly improve the performance of the function, especially if it is called repeatedly with the same inputs.

## Table of contents

1. [Minimal example](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
2. [st.cache_data](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
    1. [Basic usage](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
    2. [Behavior](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
    3. [More examples](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
3. [st.cache_resource](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
    1. [Basic usage](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
    2. [Behavior](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
    3. [More examples](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
4. [Deciding which caching decorator to use](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
5. [Cache size and validation](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
6. [Mutation and concurrency issues](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
7. [Frequently asked questions](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)
8. [Migrating from st.cache](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)

- Johannes idea for TOC
    - Minimal example → same as below
    - Basic usage → this is what a first time reader should read, they should understand how to use it but don’t need to know everything under the hood or everything they can tweak
        - st.cache_data → same content as we have in “Usage”, “Behavior”, “Examples” right now; can be in subchapters, but might also just be one text; might want to exclude most details from “Copying behavior” here, so that we keep it really simple for first-time readers
        - st.cache_resource → again, “Usage”, “Behavior”, “Examples”; might be subchapters or one text
        - Deciding which caching decorator to use → I think this overview is critical, even for beginners
    - Advanced usage → this is what people should read if they run into the limits of caching, want to customize more stuff, or want to understand how everything works under the hood
        - Excluding input parameters → explain adding underscore to parameters to exclude them from hashing
        - Controlling cache size → explain ttl and max_entries
        - Validating cached objects → explain validate
        - Customizing the spinner → explain that cached functions always show a spinner, how to turn it off, how to change the text
        - Using st commands in cached functions → explain that cached commands replay st commands, explain how experimental_allow_widgets works, why we have it, and the potential memory problems
        - Mutation and concurrency issues → show the example from “Copying behavior”, then explain concurrency issues in more detail (i.e. what’s at the end of this doc today)
    - Migrating from st.cache (or should this be inside of Advanced usage?)
    - FAQ

## Minimal example

Caching in Streamlit is done by applying one of two decorators on top of a function:

```python
@st.cache_data
def long_function(param1, param2):
    return …
```

In this example, marking the `long_function` function with the `@st.cache_data` decorator tells Streamlit that whenever the function is called it checks two things:

1. The input parameters that you called the function with.
2. The body of the function.

If this is the first time Streamlit has seen these components with these exact values and in this exact combination and order, it runs the function and stores the result in a local cache. The next time the cached function is called (e.g. when a user interacts with the app), if none of these components changed, Streamlit will just skip executing the function altogether and, instead, return the output data previously stored in the cache.

Caching comes in two flavors:

- `@st.cache_data` is the recommended way to cache data transforms and computations. E.g. loading dataframes from CSVs, transforming NumPy arrays, running database queries, returning simple types like string/int/float, or any lists or dicts that contain such data objects. It creates a new copy of the data each time the function is called, making it safe against [mutations and race conditions](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e). Note that this means objects cached with `@st.cache_data` must be serializable. This copying behavior is what you want in the majority of use cases – so if you're unsure, start with `@st.cache_data` and see if it works!
- `@st.cache_resource` is the recommended way to cache global resources like ML models or database/API connections. These resources are shared across all reruns and sessions of your app, without any copying or duplication. ~~Note that this means any mutations after the cached function directly mutate the objects stored in the cache!~~

![Untitled](Snowflake%20633ea7e4d8434fa0b28d36b2805d572d/Teams%20c18c64df7f62472c8b9963e47cd073e4/Streamlit%20eb94cf6fb5c643c9a67044ed990c10e2/Sprint%20tasks%208f6dafda43644ef18501741cb62b88e9/Feature%20draft%20docs%20for%20st%20cache_resource%20and%20st%20ca%2029a3e77c43a24bb7bba413dc7cbc097e/Untitled%207.png)

## **st.cache_data**

`st.cache_data` is your go-to command for all functions that cache data – may it be dataframes, numpy arrays, str, int, float, or other serializable types. It’s the right command for almost all use cases!

### Usage

Let's take a look at an example for using `st.cache_data`. Suppose your app is loading the [Uber ride-sharing dataset](https://github.com/plotly/datasets/blob/master/uber-rides-data1.csv) – a CSV file of 50 MB – from the internet into a dataframe: 

```python
import streamlit as st
import pandas as pd

def load_data(url):
    df = pd.read_csv(url)  # 👈 Download the data
    return df

df = load_data("https://raw.githubusercontent.com/plotly/datasets/master/uber-rides-data1.csv")
st.dataframe(df)

st.button("Rerun")
```

Running the `load_data` function takes between two and 30 seconds depending on your internet connection. (Tip: if you are on a slow connection, use [this 5 MB dataset instead](https://github.com/plotly/datasets/blob/master/26k-consumer-complaints.csv)). Without caching, the download is rerun each time the app is loaded or a user interacts with it. Try it yourself by clicking on the button we added! Not a great experience… 😕

Now let’s add the `@st.cache_data` decorator on `load_data`:

```python
import streamlit as st
import pandas as pd

@st.cache_data
def load_data(url):
    df = pd.read_csv(url)
    return df

df = load_data("https://raw.githubusercontent.com/plotly/datasets/master/uber-rides-data1.csv")
st.dataframe(df)

st.button("Rerun")
```

Run the app again. You'll notice that the slow download is only happening on the first run. Every subsequent rerun should be almost instant! 💨

- Old example
    
    Suppose you’re doing data exploration and have a function that loads data from a CSV into a dataframe. Let’s start without any caching: 
    
    ```python
    import streamlit as st
    import pandas as pd
    import time
    
    def load_data(t):
        time.sleep(t)  # 👈 This makes the function take 2s to run
        df = pd.read_csv('https://raw.githubusercontent.com/plotly/datasets/master/solar.csv')
        return df
    
    df = load_data(2)
    
    rows = st.slider("Number of rows to display", 1, 8, 4)
    
    st.write(df.head(rows))
    ```
    
    This is because `load_data(t)` is being re-executed every time the app runs. This isn't a great experience. Now let’s add the `@st.cache_data` decorator:
    
    ```python
    import streamlit as st
    import pandas as pd
    import time
    
    @st.cache_data # 👈 Added this
    def load_data(t):
        time.sleep(t)# This makes the function take 2s to run
        df = pd.read_csv('https://raw.githubusercontent.com/plotly/datasets/master/solar.csv')
        return df
    
    df = load_data(2)
    
    rows = st.slider("Number of rows to display", 1, 8, 4)
    
    st.write(df.head(rows))
    ```
    
    Now run the app again and you'll notice that it only takes a few seconds to run the function initially. Every subsequent rerun is almost instant!
    

### Behavior

How does this work? Let’s go through the behavior of `st.cache_data` step by step:

- On the first run, Streamlit realizes that `load_data` was not called with the given parameter value yet (here: the URL of the CSV file). So it runs the function and downloads the data.
- Now our caching mechanism becomes active: the returned dataframe is serialized (converted to bytes) via [pickle](https://docs.python.org/3/library/pickle.html) and stored in the cache (together with the value of the `url` parameter).
- On the next run, Streamlit looks in the cache whether there’s an entry for `load_data` with the specific value of `url`. There is! So instead of running the function (and downloading all of that data again), it deserializes the cached object to a dataframe and returns it.

This process of serializing and deserializing the cached object effectively creates a copy of our original dataframe. This might seem like unnecessary overhead but it’s exactly what we want when caching data objects. Let's understand why! 

### Copying behavior

To illustrate why the copying behavior of `st.cache_data` is useful, we’re reusing our example from above but making two modifications:

1. We are using `st.cache_resource` instead of `st.cache_data`. We’ll explain this command in detail later on. For now, you only need to know that `st.cache_resource` does **not** create a copy of the cached object. 
2. After loading the data, we are manipulating the returned dataframe (in place!) by dropping the column `"Lat"`. 

Here’s the code:

```python
import streamlit as st
import pandas as pd

@st.cache_resource   # 👈 Turn off copying behavior
def load_data(url):
    df = pd.read_csv(url)
    return df

df = load_data("https://raw.githubusercontent.com/plotly/datasets/master/uber-rides-data1.csv")
st.dataframe(df)

df.drop(columns=['Lat'], inplace=True)  # 👈 Mutate the dataframe

st.button("Rerun")
```

Let’s run it again and see what happens! The first run should work fine. But in the second run, you’re seeing an exception: `KeyError: "['Lat'] not found in axis"`. Why is that happening? Let’s go step by step again:

- On the first run, Streamlit runs `load_data` and stores the resulting dataframe in the cache. It does **not** create a copy but stores the original dataframe.
- Then we are dropping the column `"Lat"` from the dataframe. Note that this is dropping the column from the original dataframe stored in the cache. We are manipulating it!
- On the second run, Streamlit returns that exact same, manipulated dataframe. It does not have the column `"Lat"` anymore. So our call to `df.drop` results in an error! Pandas cannot drop a column that doesn’t exist.

The copying behavior of `st.cache_data` prevents this kind of mutation error. Mutations can only affect a specific copy, and not the underlying object in the cache. The next rerun will get its own, unmutated copy of the dataframe. You can try it yourself, just replace `st.cache_resource` with `st.cache_data` above and you’ll see that everything works. 

Because of this copying behavior, `st.cache_data` is the recommended way to cache data transforms and computations – basically anything that returns a serializable object. 

- Old text on copy behavior
    
    ~~In the example below, we’ll use `st.cache_resource` instead of `st.cache_data`. We’ll explain this command in detail below but for now, you only need to know that `st.cache_resource` does **not** create a copy of the cached object. Let’s load our dataframe just like above (but with `st.cache_resource`) and then directly manipulate the dataframe by dropping a column:~~ 
    
    Instead, it stores the original dataframe in the cache and returns that exact same dataframe object each time `load_data` is called.  ~~, which does **not** create a copy but stores the original dataframe object in the cache, and returns this exact same object at each run. This means that any mutations after the cached function directly mutate the objects stored in the cache!~~
    
    ~~Suppose you load the CSV file as above. Then, you mutate the dataframe by dropping a column:~~  
    
    ```python
    import streamlit as st
    import pandas as pd
    import time
    
    @st.cache_resource
    def load_data(t):
        time.sleep(t)# This makes the function take 2s to run
        df = pd.read_csv('https://raw.githubusercontent.com/plotly/datasets/master/solar.csv')
        return df
    
    df = load_data(2)
    
    rows = st.slider("Number of rows to display", 1, 8, 4)
    
    df.drop(columns=['State'], inplace=True)# 👈 Mutate the dataframe
    
    st.write(df.head(rows))
    ```
    
    ~~On the first run, the data is loaded from the CSV and this dataframe is stored in the Streamlit cache. This original object is then returned by the function, as opposed to a copy. That means on the first run, the column "State" exists in the dataframe and is dropped. This transformation or mutation is performed on the original dataframe stored in the cache. On the second run, since the mutation already happened in the previous run, the column "State" no longer exists in the dataframe, resulting in an error: `KeyError: "['State'] not found in axis"`.~~ 
    
    ~~This is generally not what you want, and is why `st.cache_data` is the recommended way to cache data transforms and computations. By creating a **copy** of the return value at each run, this means that if you mutate the dataframe, you only mutate the copy that you got, and at the next rerun you’ll get a new (unmutated) copy.~~
    

### Examples

**Dataframe transformations**

In the example above, we already showed how to cache loading a dataframe. It can also be useful to cache dataframe transformations such as `df.filter`, `df.apply`, or `df.sort_values`. Especially with large dataframes, these operations can significantly slow down your app. 

```python
@st.cache_data
def transform(df):
    df = df.filter(items=['one', 'three'])
		df = df.apply(np.sum, axis=0)
		return df
```

**Array computations**

Similarly, it can make sense to cache computations on numpy arrays:

```python
@st.cache_data
def add(arr1, arr2):
		return arr1 + arr2
```

**Database queries**

When working with databases, you usually make SQL queries to load data into your app. Running these queries repeatedly can be slow, cost money, and degrade the performance of your database. We strongly recommend to cache any database queries in your app. See also [our guides on connecting Streamlit to different databases](https://docs.streamlit.io/streamlit-community-cloud/get-started/deploy-an-app/connect-to-data-sources). 

```python
@st.cache_data
def query(_connection):
    return pd.read_sql_query("SELECT * from table", _connection)
```

<aside>
💡 To get new results from your database, you should set a `ttl` (time to live). If you set `st.cache_data(ttl=3600)`, Streamlit will invalidate any cached values after 1 hour (3600 seconds) and run the cached function again.

</aside>

**API calls**

Similarly, it makes sense to cache API call. This also avoids rate limits. 

```python
@st.cache_data
def api_call():
    response = requests.get('https://jsonplaceholder.typicode.com/posts/1')
    return response.json()
```

**ML inference**

Running complex machine learning models can use significant time and memory. To avoid rerunning the same computations over and over, use caching. 

```python
@st.cache_data
def run_model(inputs):
    return model(inputs)
```

- Old text
    
    When working with data, it's common to perform transformations on that data. However, performing these transformations can slow down the performance of your app, especially if they are complex or if the data is large. One way to improve performance is to cache the results of the data transformations so that they can be reused without having to perform the transformations again.
    
    `st.cache_data` is a decorator that can be used to cache the results of data transformations. Here's an example of how to cache a NumPy array after applying a transformation:
    
    ```python
    import streamlit as st
    import numpy as np
    
    @st.cache_data
    def load_data():
        data = np.random.randn(1000, 1000)
    		return data * 2 + 1
    
    data = load_data()
    
    st.write(data)
    ```
    
    In the example above, the `load_data` function generates a 2D NumPy array using the `np.random.randn` function and then applies a transformation to it (multiply by 2 and add 1). The `@st.cache_data` decorator is applied to the function, so the first time the function is called, the array will be generated, the transformation will be applied, and the results will be cached. The next time the function is called, the cached results will be returned instead of regenerating the array and reapplying the transformation. This can greatly improve the performance of your app and reduce the computational resources required.
    
- Old text
    
    ### Caching an API call
    
    When working with data from external sources, it's common to make API calls to retrieve that data. However, making API calls can slow down the performance of your app, especially if the calls are made frequently or if the API has a rate limit. One way to improve performance is to cache the results of the API call so that they can be reused without having to make the call again.
    
    `st.cache_data` is a decorator that can be used to cache the results of an API call. Here's an example of how to cache the results of a GET request to an API endpoint using `st.cache_data` :
    
    ```python
    import streamlit as st
    import requests
    
    @st.cache_data
    def load_data():
        response = requests.get('https://jsonplaceholder.typicode.com/posts/1')
        return response.json()
    
    data = load_data()
    
    st.write(data)
    ```
    
    In the example above, the `load_data` function makes a GET request to the specified API endpoint and returns the response in JSON format. The `@st.cache_data` decorator is applied to the function, so the first time the function is called, the API call will be made and the results will be cached. The next time the function is called, the cached results will be returned instead of making another API call. This can greatly improve the performance of your app by reducing the number of API calls made.
    
    ### Caching a database query
    
    When working with data stored in a database, it's common to perform queries to retrieve that data. However, performing database queries can slow down the performance of your app, especially if the queries are complex or if the database has a high load. One way to improve performance is to cache the results of the database query so that they can be reused without having to perform the query again.
    
    `st.cache_data` is a decorator that can be used to cache the *results of a database query* (data), while `st.cache_resource` can be used to cache the *database connection* (a shared resource):
    
    ```python
    import streamlit as st
    import sqlite3
    
    @st.cache_resource # 👈 Cache the connection (shared resource)
    def init_connection():
        return sqlite3.connect('example.db')
    
    @st.cache_data # 👈 Cache the result of the query (data)
    def load_data(conn):
        return pd.read_sql_query("SELECT * from table", conn)
    
    conn = init_connection()
    data = load_data(conn)
    
    st.write(data)
    ```
    
    In the example above, the `load_data` function connects to a SQLite database, performs a SELECT query on the specified table, and returns the results as a Pandas DataFrame. The `@st.cache_data` decorator is applied to the function, so the first time the function is called, the query will be executed and the results will be cached. The next time the function is called, the cached results will be returned instead of executing the query again. This can greatly improve the 
    performance of your app and reduce the load on the database. Notice how we **did not** use `@st.cache_data` to cache the database connection (more on that [below](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e)).
    
    ### **Caching simple types (strings, lists, dicts, integers, etc.)**
    
    Simple types like strings, integers, floats, lists, tuples, and dicts can also be cached by `st.cache_data`, as they are serializable. Let's take a look examples of caching these types:
    
    ```python
    import streamlit as st
    
    @st.cache_data
    def cache_string():
        return "Hello World!"
    
    @st.cache_data
    def cache_list():
        return [1, 2, 3]
    
    @st.cache_data
    def cache_dict():
        return {"a": 1, "b": 2, "c": 3}
    
    @st.cache_data
    def cache_tuple():
        return (1, 2, 3)
    
    @st.cache_data
    def cache_int():
        return 1
    
    st.write(cache_string())
    st.write(cache_list())
    st.write(cache_dict())
    st.write(cache_tuple())
    st.write(cache_int())
    ```
    
    But what if:
    
    - Your data is very large, and you don't want exhaust your memory by creating a copy of it every time the app is rerun?
    - You want to cache an object that is not serializable (e.g. a database connection, ML model, etc.)?
    
    In these cases, you can use `st.cache_resource`.
    

## **st.cache_resource**

`st.cache_resource` is the right command to cache “resources” that should be available  globally across all users, sessions, and reruns. It has more limited use cases, especially caching database connections and ML models. 

### Usage

As an example for `st.cache_resource`, let’s look at a typical Streamlit app for machine learning. As a first step, we obviously need to load an ML model. We do this with [Huggingface’s transformers library](https://huggingface.co/docs/transformers/index):

```python
from transformers import pipeline
model = pipeline("sentiment-analysis")
```

If we put this code into a Streamlit app directly, the model will be loaded again and again at each rerun or user interaction. This poses two problems:

- Loading the model takes time, which slows down the app.
- Each session loads the model from scratch, which takes up a huge amount of memory.

Instead, it would make much more sense to load the model once and use that same object across all users and sessions. That’s exactly what `st.cache_resource` is for! Let’s add it to our app and process some text the user entered:

```python
import streamlit as st
from transformers import pipeline

@st.cache_resource
def load_model():
    return pipeline("sentiment-analysis")

query = st.text_input("Your query", value="I love Streamlit! 🎈")

# Classify the query text.
if query:
    result = model(query)[0]
    st.write(result)
```

If you run this app, you’ll see that `load_model` will only be called once when the app starts. Subsequent runs will reuse that same model that’s stored in the cache, saving time and memory! 

- Old text
    
    ~~allows you to cache the initialization of shared resources. This is useful for caching objects that you want to reuse across multiple runs, sessions, and users, including objects that are not serializable.~~
    
    ```python
    import streamlit as st
    
    @st.cache_resource
    def load_resource(): 
        # Initialization code here
        return resource
    
    resource = load_resource()
    ```
    
    By using `st.cache_resource`, the `resource` is only initialized once, and then reused every time the app is rerun. Every user will also get the same `resource`, meaning that the `resource` is a global resource that is shared across all users, reruns, and sessions.
    

### Behavior

Using `st.cache_resource` is very similar to using `st.cache_data`. But there are a few important differences in behavior: 

- `st.cache_resource` does **not** create a copy of the cached objects. Instead, it stores the object itself in the cache. This means all mutations will directly affect the cached object. You need to make sure that mutations on the cached object or concurrency issues do not lead to problems. See also … (link to section below).
- Objects cached with `st.cache_resource` are accessible globally by every session and user. To be clear, caching is also global for `st.cache_data`. But for the latter, every session and run gets its own copy, so there’s no chance of multiple sessions accessing the same object.
- Return values of functions do not need to be serializable. This is great for types that are not serializable by nature, e.g. some database connections, file handles, or threads. Caching these objects with `st.cache_data` is not possible.

- Old text
    
    `~~st.cache_resource` is used in a small number of common cases. Database connections and ML models are the foremost use cases. Objects like file handles and threads are less common use-cases.~~ 
    
    It is important to note that using `st.cache_resource` means that the object is now a global resource that is shared across all users, reruns, and sessions. You need to be careful about mutating the object, as this will affect all users.
    
    Suppose your app is running on a server with 4GB of memory, and you want to cache a large language model like GPT3 that takes up 3GB of memory. If you use `st.cache_data`, then every time the app is rerun, a copy of the model will be created in memory, resulting in an out of memory error. That is assuming that the model is serializable, which it often will not be. In this case, what you want is for the app to only load the model once in its lifetime, and then reuse that same object every time the app is rerun, even across multiple sessions and users. This is what `st.cache_resource` does.
    
    Or let's say your app connects to a database, and you want to cache the connection object. This is also a good use case for `st.cache_resource`, as you don't want to create a new connection every time the app is rerun, nor is the connection object serializable.
    

### Examples

**Database connections**

When connecting to a database, you usually want to create a connection object that is reused across multiple queries. It is not a good idea to create a new connection every time you want to query the database, as this is inefficient and can lead to connection errors. Let's take a look at an example of how to cache a database connection. In this example, we're connecting to a public Postgres database. But the same principles apply to other databases like MySQL, SQLite, etc.

```python
import psycopg2
import streamlit as st

# Initialize connection.
# Uses st.cache_resource to only run once.
@st.cache_resource
def init_connection():
    host = "hh-pgsql-public.ebi.ac.uk"
    database = "pfmegrnargs"
    user = "reader"
    password = "NWDMCE5xdipIjRrp"
    return psycopg2.connect(host=host, database=database, user=user, password=password)

conn = init_connection()
```

By using `st.cache_resource`, the connection object is only created once, and then reused every time the app is rerun. Every user will also get the same connection object, meaning that the connection object is a global resource that is shared across all users, reruns, and sessions.

**ML models**

Let's take a look at an example of how to cache an ML model. In this example, we're using a pre-trained [DistilBERT](https://huggingface.co/transformers/model_doc/distilbert.html) model from the 🤗 [Hugging Face](https://huggingface.co/) library. This model is used to classify text into one of 2 categories: "positive" or "negative". The model is cached using `st.cache_resource`, so that it is only loaded once, and then reused every time the app is rerun.

```python
import streamlit as st
from transformers import pipeline

# Initialize model.
# Uses st.cache_resource to only run once.
@st.cache_resource
def init_model():
    return pipeline(
        "sentiment-analysis",
        model="distilbert-base-uncased-finetuned-sst-2-english"
    )

model = init_model()

# Get user input.
text = st.text_input("Enter some text", value="I love Streamlit!")

# Classify text.
if text:
    result = model(text)[0]
    st.write(result)
```

By using `st.cache_resource`, the model is only loaded into memory once (on the first run), and then reused every time the app is rerun. Every user will also get the same model, meaning that the model is a global resource that is shared across all users, reruns, and sessions.

**File handles**

When working with files, it's common to open and close file handles frequently. However, opening and closing file handles can slow down the performance of your app, especially if the files are large or if the file operations are performed frequently. One way to improve performance is to cache the file handles so that they can be reused without having to constantly open and close the files:

```python
import streamlit as st

@st.cache_resource
def init_file():
    return open('large_file.txt', 'r')

file = init_file()

# Perform operations on the file
file_content = file.read()
st.write(file_content)
```

In the example above, the `init_file` function opens a file named 'large_file.txt' and returns the file handle. The `@st.cache_resource` decorator is applied to the function, so the first time the function is called, the file handle will be created and the results will be cached. The next time the function is called, the cached file handle will be returned instead of opening the file again.

<aside>
⚠️ It is worth noting that while caching global resources with `st.cache_resource` will improve performance and reduce the resources required by your app, it should be used with care, as it may lead to unexpected behavior if the object is not thread-safe or if it has mutable state. Learn more in [Mutation and concurrency issues](https://www.notion.so/Feature-draft-docs-for-st-cache_resource-and-st-cache_data-29a3e77c43a24bb7bba413dc7cbc097e).

</aside>

## **Deciding which caching decorator to use**

Deciding which caching decorator to use is not always trivial. It comes down to understanding the difference between “data” and “resource”. 

One way to conceptualize the difference is to consider a case where your app connects to a database, pulls data from it and writes data back. Pulling and writing stored *serializable* data to and from the database is what you use `@st.cache_data` for, while the *non-serializable* database connection itself should be cached with `@st.cache_resource`.

`@st.cache_resource` is used in a small number of common cases (database connections and ML models are the foremost use-cases; things like file handles, and threads are less common use-cases). `@st.cache_data` is *everything else*.

Here is a table of the most common use cases we’ve observed, their most common return types, and the corresponding caching decorator we recommend using:

| Use case | Return type | Decorator |
| --- | --- | --- |
| String literals, user input | str | st.cache_data |
| Floating-point numbers, numerical computations | float | st.cache_data |
| Integer values, counters | int | st.cache_data |
| Binary data, image data | bytes | st.cache_data |
| Boolean values, flags | bool | st.cache_data |
| Timestamps, dates | datetime.datetime | st.cache_data |
| Data manipulation, analysis | pandas.DataFrame | st.cache_data |
| Data manipulation, analysis | pandas.Series | st.cache_data |
| Numerical computations, image processing | numpy.ndarray | st.cache_data |
| Images, image processing | PIL.Image.Image | st.cache_data |
| Plotly figures | plotly.graph_objects.Figure | st.cache_data |
| Matplotlib figures | matplotlib.figure.Figure | st.cache_data |
| Altair figures | altair.Chart | st.cache_data |
| Database connections | pyodbc.Connection | st.cache_resource |
| MongoDB connections | pymongo.mongo_client.MongoClient | st.cache_resource |
| MySQL connections | mysql.connector.MySQLConnection | st.cache_resource |
| PostgreSQL connections | psycopg2.connection | st.cache_resource |
| SQLAlchemy connections | sqlalchemy.engine.base.Engine | st.cache_resource |
| SQLite connections | sqlite3.Connection | st.cache_resource |
| PyTorch models | torch.nn.Module | st.cache_resource |
| TensorFlow models | tensorflow.keras.Model | st.cache_resource |
| HuggingFace models | transformers.Pipeline | st.cache_resource |

Please note that the above table is a guideline, and the use case and return type may vary depending on the specific application or scenario. It is worth reiterating that:

- `st.cache_data` is used for serializable objects
- `st.cache_resource` is used for non-serializable objects such as database connections, ML models, file handles, and threads
- For very large serializable objects such as arrays/dataframes (> ca 100 million rows) it can make sense to use `st.cache_resource` to avoid copying. But make sure the object isn’t mutated!
- For other types of return objects that are not listed in the above table, you will have to evaluate them and decide which decorator to use based on the return object's serializability.

- **Old table (not part of actual docs; ignore!)**
    
    Here is a table to help you decide which caching decorator to use for different use cases:
    
    | Cached return object | Decorator |  |
    | --- | --- | --- |
    | Pandas dataframe | st.cache_data |  |
    | Numpy array | st.cache_data |  |
    | Simple types like str/int/float | st.cache_data |  |
    | Lists/tuples/dicts of the above | st.cache_data |  |
    | ML models, tokenizers, … | st.cache_resource |  |
    | Database/API connections | st.cache_resource |  |
    | Figure objects | Depends! Safer to use st.cache_data. But if figure objects are not serializable for some charting libraries, you’ll need touse st.cache_resource. |  |
    | Very large arrays/dataframes (> ca 100 million rows) | Can make sense to use st.cache_resource to avoid copying.But make sure the object isn’t mutated then! |  |
    
    Here is a table of the most common return types we've observed, and the corresponding caching decorator we recommend using:
    
    | Decorator | Return type |
    | --- | --- |
    | st.cache_data | str
    float
    int
    bytes
    bool
    datetime.datetime
    pandas.DataFrame
    pandas.Series
    numpy.ndarray
    numpy.float64
    numpy.int64
    PIL.Image.Image
    plotly.graph_objects.Figure
    matplotlib.figure.Figure
    altair.Chart |
    | st.cache_resource | pyodbc.Connection
    pymongo.mongo_client.MongoClient
    mysql.connector.MySQLConnection
    psycopg2.connection
    psycopg2.extensions.connection
    snowflake.connector.connection.SnowflakeConnection
    snowflake.snowpark.sessions.Session
    sqlalchemy.engine.base.Engine
    sqlite3.Connection
    torch.nn.Module
    tensorflow.keras.Model
    tensorflow.Module
    tensorflow.compat.v1.Session
    transformers.Pipeline
    transformers.PreTrainedTokenizertransformers.PreTrainedTokenizerFast
    transformers.PreTrainedTokenizerBase
    transformers.PreTrainedModel
    transformers.TFPreTrainedModel
    transformers.FlaxPreTrainedModel |
    

## Cache size and validation

The `ttl` and `max_entries` parameters for both decorators allow you control the size of the cache, while `st.cache_resource`'s `validate` parameter allows you to control cache validity

### The `ttl` (time-to-live) parameter

The `ttl` parameter is used to set the maximum number of seconds that a cached resource should be kept in the cache. This is useful for resources that may become stale after a certain amount of time, such as data from a remote API that is only updated every hour. If a resource is requested and it has been in the cache for longer than the specified `ttl`, it will be recomputed and the new value will be stored in the cache. The default value for `ttl` is `None`, which means that cache entries will not expire.

It is important to note that the `ttl` parameter is incompatible with `persist="disk"`. If `persist` is specified, the `ttl` parameter will be ignored.

Example usage:

```python
@st.cache_data(ttl=3600) # cache data for 1 hour
def get_api_data():
    # code to fetch data from remote API
    return data
```

<aside>
💡 Tip: We also support `timedelta` values for `ttl`, which lets you set more fluent `ttl` values. E.g. `ttl=timedelta(hours=1)`. But if you just pass a raw number, it will be seconds.

</aside>

### The `max_entries` parameter

The `max_entries` parameter is used to set the maximum number of entries that should be kept in the cache. This is useful for limiting the amount of memory that the cache uses, especially when caching large objects. When a new entry is added to a full cache, the oldest cached entry will be removed to make room for the new one. The default value for `max_entries` is `None`, which means that the cache is unbounded.

Example usage:

```python
@st.cache_data(max_entries=1000) # keep a maximum of 1000 entries in the cache
def get_large_dataframe():
    # code to generate a large dataframe
    return df
```

### Validating cached resources

TODO: @Tim Conkling ❄️ or @Johannes Rieke or @Joshua Carroll 

### The `show_spinner` parameter

The `show_spinner` parameter is used to control the display of a spinner when a "cache miss" occurs and the cached resource is being created. A "cache miss" occurs when the requested resource is not found in the cache and needs to be recomputed. By default, the spinner is enabled (`show_spinner=True`) to indicate to the user that something is happening. If set to `False`, the spinner will not be shown. The parameter also accepts a string, which will be used as the text for the spinner.

Example usage:

```python
@st.cache_data(show_spinner=False) # disable the spinner
def get_api_data():
    # code to fetch data from remote API
    return data

@st.cache_data(show_spinner="Fetching data from API") # use custom text for spinner
def get_api_data():
    # code to fetch data from remote API
    return data
```

### The `experimental_allow_widgets` **parameter**

Functions decorated with either decorator can also contain [input widgets](http://localhost:3000/library/api-reference/widgets)! Replaying input widgets is disabled by default. To enable it, you can set the `experimental_allow_widgets` parameter for to `True`.

Example usage:

```python
import streamlit as st

# Enable widget replay
@st.experimental_memo(experimental_allow_widgets=True)
def func():
    # Contains an input widget
    st.checkbox("Works!")

func()
```

Learn more about widget replay in the [st.cache_data](http://localhost:3000/library/api-reference/performance/st.cache_data) and [st.cache_resource](http://localhost:3000/library/api-reference/performance/st.cache_resource) API.

<aside>
⚠️ Setting this parameter to `True` may lead to excessive memory use since the widget value is treated as an additional input parameter to the cache. Every cache function will store a different cache value for every permutation of input parameters (just like any other cache function), AND *every permutation of widget values*. This can blow up your cache size in surprising ways.

</aside>

## **Mutation and concurrency issues**

It's important to be aware of mutation and concurrency issues when using `@st.cache_resource`, since it creates global resources that are shared across all users, reruns, and sessions. Note that since `@st.cache_data` returns a new copy of its data on each call, using it does not come with the same concerns.

Mutations refer to changes made to an object after it has been created. In the context of caching, it means modifying a cached object after it has been retrieved and returned by a function.

Concurrency refers to the ability of multiple users or processes to access and make changes to a shared resource at the same time. When it comes to caching, concurrency issues can arise when multiple users are accessing the same cached object simultaneously and making changes to it.

Mutating a cached resource can lead to unexpected and inconsistent results for different users, as changes made by one user will be reflected in the resource for all other users. Here's an example that clearly illustrates the consequences of mutating a cached resource when multiple users are accessing the app concurrently. Let's say you have a function that returns a list:

```python
import streamlit as st

@st.cache_resource
def create_list():
    l = [1, 2, 3]
    return l

l = create_list()
first_list_value = l[0]
l[0] = first_list_value + 1

st.write("l[0]=", l[0])
```

Let's say you're user A and you run the above app. You'll see the following output:

```python
l[0]= 2
```

Now, let's say another user B visits the app right after you. User B will instead see the following output:

```python
l[0]= 3
```

Now, if you're user A and you rerun the app immediately after user B, you'll see the following output:

```python
l[0]= 4
```

Why is this happening?

- When user A visits the app, `create_list()` is called and the list `[1, 2, 3]` is cached. This list is then returned to user A. Then, the first value of the list, 1, is assigned to first_list_value and l[0] is mutated to 2.
- When user B visits the app, `create_list()` returns the mutated list `[2, 2, 3]` to user B. Then, the first value of the list, 2, is assigned to first_list_value and l[0] is mutated to 3.
- When user A reruns the app, `create_list()` returns the mutated list `[3, 2, 3]` to user A. Then, the first value of the list, 3, is assigned to first_list_value and l[0] is mutated to 4.

If you think about it, this makes sense. User A and B are both using the same list object, and since the list object is mutated, user A's change to the list object is reflected in user B's app as well. 

In the above example, the list object is a global resource that is shared across all users, reruns, and sessions. This is why you need to be careful about mutating a cached resource when multiple users are accessing the app concurrently. If you had used `st.cache_data` instead of `st.cache_resource`, the list object would have been copied for each user, and the above example would have worked as expected (i.e. user A and B would have seen the same output):

```python
l[0]= 2
```

Now consider the less common use case where you have a function that returns a very large dataframe (> 100 million rows) that you want to cache. From the table in the prior section, you'd be inclined to use `st.cache_data` as the return type is a dataframe. But doing so may not be a good idea, as `st.cache_data` will create a new copy from the serialized data and return that with every run, which can lead to memory issues and cause your app to crash. Instead, you should use `st.cache_resource` to avoid copying the dataframe. However, this means that the dataframe is now a global resource that is shared across all users, reruns, and sessions. You need to be careful about mutating the dataframe, as this will affect all users.

## Frequently asked questions

1. What is the difference between `st.cache_data` and `st.cache_resource`?
- `st.cache_data` is used to cache serializable objects such as Pandas DataFrames, NumPy arrays, and simple types like strings, integers, and floats. These objects can be [pickled and unpickled](https://docs.python.org/3/library/pickle.html), so `st.cache_data` creates a new copy of the object from the serialized data and returns it with every run.
- `st.cache_resource` *must* be used to cache non-serializable objects such as database connections, ML models, file handles, and threads. These objects cannot be serialized, so `st.cache_resource` returns the same object with every run. But in some cases, you *may* want to use `st.cache_resource`for caching very large serializable objects so that your app doesn’t exhaust its memory by creating a copy of it every time the app is rerun.
1. Can I use `st.cache_data` for non-serializable objects?
    
    No, `st.cache_data` should only be used for serializable objects. If you are working with non-serializable objects, you *must* should use `st.cache_resource`.
    
2. Can I use `st.cache_resource` for simple types like strings, integers, and floats?
    
    No, you should use `st.cache_data` for simple types like strings, integers, and floats. `st.cache_resource` is meant for non-serializable objects only. However, in some few cases, it may make sense to use `st.cache_resource` to cache very large serializable objects. But be aware that caching the serializable object as a resource means that it is now a global resource that is shared across all users, reruns, and sessions. You need to be careful about mutating the object, as this will affect all users.
    
3. What happens if I mutate a cached resource?
    
    If you mutate a cached resource, the change will be reflected in the resource for all users, reruns, and sessions. This can lead to unexpected behavior if multiple users are accessing the app concurrently. To avoid this, you should make sure that the resource is not mutated after it is cached, or make sure that your cached resources are safe to mutate from multiple threads simultaneously.
    

# Migrating from st.cache

The caching commands described above were introduced in Streamlit 1.18.0. Before that, we had one catch-all command `st.cache`. Using it was often confusing, resulted in weird exceptions, and was slow. That’s why we replaced `st.cache` with the new commands in 1.18.0 (read more in this blog post). The new commands provide a more intuitive and efficient way to cache your data and resources, and are intended to replace `st.cache` in all new development.

If your app is still using `st.cache`, don’t despair! Here are a few notes on migrating:

- `st.cache` is deprecated. If your app uses it, new versions of Streamlit will show a deprecation warning.
- We will not remove `st.cache` soon, so you don’t need to worry about your 2-year-old app breaking. But we encourage you to try the new commands going forward – they will be way less annoying for you!
- Switching code to the new commands should be easy in most cases. To decide whether to use [`st.ca](http://st.ca)che_data` or `st.cache_resource`, read the documentation above. Streamlit will also recognize common use cases and show hints right in the deprecation warnings.
- Most parameters from `st.cache` are also present in the new commands, with a few exceptions:
    - `allow_output_mutation` does not exist anymore. You can safely delete it. Just make sure you use the right caching command for your use case.
    - `suppress_st_warning` does not exist anymore. You can safely delete it. Cached functions can now contain Streamlit commands and will replay them. If you want to use widgets inside cached functions, set `experimental_allow_widgets=True`.
    - `hash_funcs` does not exist anymore. You can exclude parameters from caching (and being hashed) by prepending them with an underscore: `_excluded_param`

If you have any questions or issues during the migration process, please reach out to us on the [forum](https://discuss.streamlit.io/) and we will be happy to assist you. 🎈 

---