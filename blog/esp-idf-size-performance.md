# How I Increased the Performance of an Open Source Python Script by 50x

I want to take a chance to explore in depth how I increased the performance of a python script by **50x** (Yes, 50 times faster) within
 a very popular open source repository with over 14.3k stars on GitHub.

## Background

The `idf_size.py` script plays a critical role in the Espressif IoT Development Framework ([espressif/esp-idf](https://github.com/espressif/esp-idf)) by providing detailed insights into memory usage, including Instruction RAM (`IRAM`) and Data RAM (`DRAM`) usage. Optimizing these memory sections are essential for the ESP32's performance: `IRAM` is used for executing time-critical code, while `DRAM` stores variables and application data. Since both `IRAM` and `DRAM` are limited resources within the ESP32 microcontroller, efficient utilization is vital to maximizing performance.

![image](https://github.com/user-attachments/assets/bce7e422-0144-4537-b3dd-1162e955e119)
_ESP32 internal memory (SRAM) layout_
   (_image source: [espressif](https://developer.espressif.com/blog/esp32-programmers-memory-model/)_)


However, there’s a catch: as important as knowning the memory usage is, `idf_size.py`'s performance issues had long been a source of frustration. Slow execution times, especially with large map files, wasted valuable developer time, and made frequent memory analysis a tedious task. Recognizing this issue, I set out to enhance the script's performance, and the results were nothing short of transformative.

## Legacy Code Performance

The following is a real comparision based on a 15MB firmware.map file for an IoT project based around home automation that I was working on at the time.

**Input file size**
```shell
anthony@linux:~/esp/esp-idf/tools$ du -h /tmp/firmware.map
15M    /tmp/firmware.map
```

**Legacy Code Performance**
```shell
anthony@linux:~/esp/esp-idf/tools$ time ~/esp/esp-idf-master/tools/idf_size.py /tmp/firmware.map
Total sizes:
 DRAM .data size:   14132 bytes
 DRAM .bss  size:   35128 bytes
Used static DRAM:   49260 bytes (  75320 available, 39.5% used)
Used static IRAM:  108605 bytes (  22467 available, 82.9% used)
      Flash code:  923103 bytes
    Flash rodata:  300984 bytes
Total image size:~1346824 bytes (.bin may be padded larger)

real    0m19.440s
user    0m19.395s
sys     0m0.044s
```

As you can clearly see, it took almost **20 seconds** for the software to parse my file. Something is definitely not right!

#### In-Depth Analysis of Optimizations

During my optimization process, I experimented with various methods to improve the script's speed. Below is an in-depth analysis of each method and how they affected the resulting speed:

1. **Fixed Regex**:
   The most significant performance gain came from fixing the `RE_SOURCE_LINE` regex. I noticed an extra unneeded wildcard after the symbol name, which slowed down the matching process. By removing this wildcard, the regex became immensly more efficient.

   ```python
                                          |
                                          V
   RE_SOURCE_LINE = r"\s*(?P<sym_name>\S*).* +0x(?P<address>[\da-f]+) +0x(?P<size>[\da-f]+) (?P<archive>.+\.a)?\(?P<object_file>.+\.(o|obj))?\)"
   ```

2. **Combining Regexes**:
   Combining the CMake and source file regexes into one reduced the processing time by almost half. This optimization alone brought the execution time down to approximately 1.4 seconds.

3. **Pre-filtering Mechanism**:
   Introducing a pre-filtering mechanism to quickly skip lines that do not match the expected pattern significantly reduced the number of lines processed by the complex regex.

   ```python
   RE_PRE_FILTER = re.compile(r".*\.(o|obj)\)?")
   ```

4. **Early Exit Mechanism**:
   By modifying the script to exit early once the relevant sections of the map file were processed, I avoided unnecessary work on irrelevant parts of the map file.

   ```python
   for line in map_file:
       if line.strip() == "Cross Reference Table":
           break
   ```

#### Resulting Optimized Performance

**My optimized version**
```shell
anthony@linux:~/esp/esp-idf/tools$ time ./idf_size.py /tmp/firmware.map
Total sizes:
 DRAM .data size:   14132 bytes
 DRAM .bss  size:   35128 bytes
Used static DRAM:   49260 bytes (  75320 available, 39.5% used)
Used static IRAM:  108605 bytes (  22467 available, 82.9% used)
      Flash code:  923103 bytes
    Flash rodata:  300984 bytes
Total image size:~1346824 bytes (.bin may be padded larger)

real    0m0.385s
user    0m0.361s
sys     0m0.024s
```

**Results**:
- Execution time reduced from **19.4 seconds** to **0.4 seconds**. This is a substantial improvement.

#### Benchmarking Results

I conducted extensive benchmarking to measure the impact of each optimization. The following table summarizes the results:

| Pre-Filter | Early Exit | Optimized Regex | Avg Milliseconds |
|------------:|------------:|-------------:|------------:|
| ✅          | ✅          | ✅           | 375        |
| ❌          | ✅          | ✅           | 561        |
| ✅          | ❌          | ✅           | 505        |
| ❌          | ❌          | ✅           | 675        |
| ✅          | ✅          | ❌           | 1291       |
| ❌          | ✅          | ❌           | 4672       |
| ✅          | ❌          | ❌           | 6023       |
| ❌          | ❌          | ❌           | 9294       |

As you can see, the fixed regex provided the most significant performance gain. Without the fixed regex, the pre-filter and early exit mechanisms made a much larger difference. Combining all three optimizations resulted in the best performance, reducing the execution time to just 375 milliseconds!

#### Conclusion

I am pleased with the results of these performance improvements. Reducing the execution time from 19.4 seconds to just 0.4 seconds is a remarkable achievement. This optimization not only enhances the script's efficiency but also improves the overall developer experience when working with large map files in the Espressif IoT Development Framework.

Thank you for reading. I hope these improvements make your development process smoother and more enjoyable. Feel free to check out the [PR #4518](https://github.com/espressif/esp-idf/pull/4518) for more details and to see the code changes.

Let's continue to make our tools faster and better together!

---
