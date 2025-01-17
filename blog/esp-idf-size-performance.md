# How I Made an Open Source Python Script Run 50x Faster

Optimizing performance is an important aspect of software development, especially in embedded systems where resources are constrained. 
In this blog, I’ll walk you through how I achieved a **50x speedup** in `idf_size.py`, 
a Python script within the popular [Espressif IoT Development Framework (ESP-IDF)](https://github.com/espressif/esp-idf).

This transformation not only improved developer workflows but also highlighted the power of targeted optimizations.

## Background

`idf_size.py` plays a crucial role in analyzing memory usage for ESP32 firmware, focusing on the efficient utilization of limited resources like Instruction RAM (IRAM) and Data RAM (DRAM). 
While essential for optimizing firmware, the script's slow execution time, particularly when processing large `.map` files, hindered developer workflows and productivity.

### The Problem
Parsing a 15MB `.map` file took nearly 20 seconds, causing delays during builds and reducing productivity.
My goal was to eliminate this bottleneck while maintaining accuracy.

## Original Performance and Bottlenecks

### Legacy Code Performance
Running `idf_size.py` on a 15MB `.map` file yielded the following results:
```shell
anthony@linux:~/esp/esp-idf/tools$ time ./idf_size.py /tmp/firmware.map

real    0m19.440s
user    0m19.395s
sys     0m0.044s
```

An execution time nearing 20 seconds made the script impractical for frequent use.

### Bottlenecks Identified
Through careful analysis, the following issues were identified as the primary causes of slow performance in `idf_size.py`:

1. **Inefficient Regular Expressions:**

   The script relied on complex and redundant regex patterns, some with unnecessary wildcards, which significantly slowed down pattern matching.

2. **Redundant Regex Evaluations:**

   Multiple regexes were applied to the same line, duplicating effort and increasing processing time unnecessarily.

3. **Lack of Filtering Mechanisms:**

   Every line of the `.map` file was processed, even those irrelevant to memory usage analysis, adding unnecessary overhead.

By addressing these bottlenecks, I was able to dramatically reduce the runtime and make the script far more efficient.

## In-Depth Analysis of Optimizations

To tackle the identified bottlenecks, I implemented a series of targeted optimizations. 
Each optimization addressed specific inefficiencies in the script, from improving regular expression performance 
to minimizing unnecessary computations. These changes were iterative, with each step building upon the previous improvements 
to achieve the final result.

Below, I’ll detail the key optimizations that collectively reduced the execution time from nearly 20 seconds to just 0.385 seconds.

### Pre-Compiling Regexes
While some [have argued](https://stackoverflow.com/questions/452104/is-it-worth-using-pythons-re-compile) over the true performance gains of using `re.compile` 
due to the fact that the first time you call `re.match()`, it will perform a compilation and store it in a cache, there are still a few key benefits to using it here.
1. Using `re.compile` provides more readable code due to the fact that it allows you to assign a name to the compiled object and use it directly, rather than as a parameter in a function call:
       
    **_Original Code:_**
    ```python
    RE_MEMORY_SECTION = r"(?P<name>[^ ]+) +0x(?P<origin>[\da-f]+) +0x(?P<length>[\da-f]+)"
    m = re.match(RE_MEMORY_SECTION, line)
    ```

    **_Optimized Code:_**
    ```python
    RE_MEMORY_SECTION = re.compile(r"(?P<name>[^ ]+) +0x(?P<origin>[\da-f]+) +0x(?P<length>[\da-f]+)")
    m = RE_MEMORY_SECTION.match(line)
    ```
2. Pre-compiled regexes will bypass the cache entirely, meaning they will avoid having to make cache lookups on every usage, and are not limited by the maximum cache size available.

### Optimized Regex  
The most significant performance gain came from optimizing the `RE_SOURCE_LINE` regex. I noticed an extra unneeded wildcard after the symbol name, which slowed down the matching process. By removing this wildcard, the regex became immensely more efficient.
This optimization alone brought the execution time down to approximately 1.4 seconds! Despite this substantial gain, additional opportunities for optimization remained.

```python
                                       |
                                       V
RE_SOURCE_LINE = r"\s*(?P<sym_name>\S*).* +0x(?P<address>[\da-f]+) +0x(?P<size>[\da-f]+) (?P<archive>.+\.a)?\(?P<object_file>.+\.(o|obj))?\)"
```

### Combining Regexes
   I noticed that the code was using 2 distinct regexes which looked very similar.  I was able to optimize the performance even further by
   combining these two into a single regex, avoid the overhead of calling `regex.match()` twice. It was as simple as making the `archive`
   named group be optional. This change alone resulted in cutting the processing time in half even without removing the extra wildcard!

   **_Original Code_**
   ```python
   RE_SOURCE_LINE = r"\s*(?P<sym_name>\S*).* +0x(?P<address>[\da-f]+) +0x(?P<size>[\da-f]+) (?P<archive>.+\.a)\((?P<object_file>.+\.ob?j?)\)"
   m = re.match(RE_SOURCE_LINE, line, re.M)
   if not m:
      RE_SOURCE_LINE = r"\s*(?P<sym_name>\S*).* +0x(?P<address>[\da-f]+) +0x(?P<size>[\da-f]+) (?P<object_file>.+\.ob?j?)"
      m = re.match(RE_SOURCE_LINE, line)
   ```
   
   **_Optimized Code_**
   ```python
    RE_SOURCE_LINE = re.compile(r"\s*(?P<sym_name>\S*) +0x(?P<address>[\da-f]+) +0x(?P<size>[\da-f]+) (?P<archive>.+\.a)?\(?(?P<object_file>.+\.(o|obj))\)?")
    m = RE_SOURCE_LINE.match(line)
   ```

### Pre-filtering Mechanism
   Since I knew that the `RE_SOURCE_LINE` regex was the slowest part of the code, I wanted to design ways to avoid calling it. This is why I introduced a pre-filtering mechanism.
   This pre-filtering regex allowed me to quickly check if the line _might_ be a possible candidate, without running the full regex on it.
   Skipping lines that do not match the expected pattern significantly reduced the number of lines that had to be processed by the complete regex.

   ```python
   RE_PRE_FILTER = re.compile(r".*\.(o|obj)\)?")
   if not RE_PRE_FILTER.match(line):
     continue
   ```

### Late Start And Early Exit Mechanisms
I analyzed the `.map` file format to identify a start condition and a termination condition for the processing. 
The script now skips processing any lines before or after the relevant sections, reducing the total number of lines that need to be processed.

## Results

### Benchmark Analysis
Here’s a breakdown of how each individual optimization contributed to the total performance improvements:

| Pre-Filter | Early Exit | Optimized Regex | Avg Milliseconds |
|------------:|------------:|-------------:|------------:|
| ✅          | ❌          | ❌           | 6023       |
| ❌          | ✅          | ❌           | 4672       |
| ❌          | ❌          | ✅           | 675        |

### Final Performance

The optimized script:
```shell
anthony@linux:~/esp/esp-idf/tools$ time ./idf_size.py /tmp/firmware.map

real    0m0.385s
user    0m0.361s
sys     0m0.024s
```

Execution time reduced from 19.44 seconds to **0.385 seconds**!

## Real-World Impact

These optimizations made it feasible to run `idf_size.py` after every build, saving developers countless hours and enabling faster feedback loops.



## Conclusion

The optimizations applied to idf_size.py have transformed it from a slow, frustrating tool to a fast and efficient solution for analyzing memory usage. 
The dramatic reduction in execution time highlights the impact of targeted optimizations.

For further details and code changes, check out [PR #4518](https://github.com/espressif/esp-idf/pull/4518).
