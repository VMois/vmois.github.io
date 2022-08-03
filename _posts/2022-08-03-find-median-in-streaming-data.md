---
layout: post
title: Two algorithms to find a median in streaming data
tags: [python,statistics,algorithms]
comments: false
readtime: true
slug: find-median-in-streaming-data
---

In this article, I want to describe two algorithms you can use to find a median on streaming data. Both of them calculate the median in one pass and are storage efficient.

## Motivation

Median (or 50th percentile) is a well-known metric. The "traditional" way of calculating the median is to take all numbers, sort them, and get a value in the middle. For modern computers, storing a couple of millions of numbers is easy. But what if we have much more data? Or we have limited space (think embedded systems)? In situations where resources are limited, two algorithms described below might help. They find a median in streaming data without storing all numbers. Curious? Let's get started!

## Running median algorithm

Running median algorithm is designed to find a median in streaming data. It returns an *approximate* median. The "running median" is not an actual name for this algorithm. I couldn't find an original name, so I will continue to call it "running median" for the rest of the article. Initially, I found an implementation of this algorithm on [John D. Cook's blog](https://www.johndcook.com/blog/skewness_kurtosis/). The algorithm is quite simple, so it will be easier to show the code straight away.

```python
class RunningMedian:
    n = 0
    _median = 0

    def push(self, new_number: int):
        self.n += 1
        weighted_average = (new_number - self._median) / self.n
        self._median += weighted_average
    
    def median(self):
        return self._median

# Usage
A = RunningMedian()
A.push(1)
A.push(2)
A.push(3)
A.push(4)

print(A.median())
```

As you can see, it is storage efficient. We only store two variables, `n` and `median`. In addition, the `push` method contains basic arithmetic operations, which means processing large amounts of numbers should be fast.

You can find more information about the running median algorithm in [this technical report](https://doi.org/10.2172/1028931) (p. 7, formula 1.1).

Moreover, you can calculate the median of a stream in parallel and later join results to get a joined median. Below you can find the example.

```python
A = RunningMedian()
# push numbers to A
# ...

B = RunningMedian()
# push numbers to B
# ...

total_n = A.n + B.n
median_A_B = A.median() + (B.n / total_n) * (B.median() - A.median())
```

You can find more details about calculating the median in parallel in [the same report presented above](https://doi.org/10.2172/1028931) (p. 8, formula 1.3).

Let's proceed to the next algorithm.

## Remedian algorithm

The Remedian algorithm can also find a median in streaming data. It returns an *approximate* median. I found a reference to this algorithm in [the Stackoverflow question](https://stackoverflow.com/a/729469). The Remedian algorithm works in the following way:

1. Create an array of a fixed odd size `b` called *buffer*;
2. Insert new numbers into a buffer;
3. If a buffer is full, calculate the median of the buffer, create a new buffer and add the computed median to a new buffer. Clear the previous buffer;
4. Insert new numbers into the previous buffer to reuse space;
5. To get a median:

- If all buffers are empty and the last buffer has only one value, the median of the stream is the first value from the last buffer;

OR

- If one or more buffers contain multiple values, calculate the weighted median of all values from all the buffers, and the weighted median becomes the median of a stream; For example, if you have two buffers of size `3`, each value from the first buffer will have a weight of `3^0` and each value from the second buffer will weight `3^1`, and so on;

You can find a nice diagram explaining algorithm steps in [this paper](https://www.sciencedirect.com/science/article/pii/S0304397513004519) (p. 4, Fig. 3).

{: .box-note}
**Note:** Check the [original Remedian algorithm paper](https://www.researchgate.net/publication/247974442_The_Remedian_A_Robust_Averaging_Method_for_Large_Data_Sets).

Let's take a look at the example implementation.

```python
import numpy as np

class Remedian:
    def __init__(self, buffer_size: int = 3):
        self.buffer_size = buffer_size
        self.buffers = []

    def _create_buffer(self):
        if self.i > len(self.buffers) - 1:
            new_buffer = [None] * self.buffer_size
            self.buffers.append(new_buffer)
    
    def _is_current_buffer_full(self):
        buffer = self.buffers[self.i]
        return buffer[-1] != None
    
    def _is_buffer_empty(self, position: int):
        buffer = self.buffers[position]
        return buffer[0] == None
    
    def _insert_number_into_buffer(self, number: int):
        buffer = self.buffers[self.i]

        for i in range(len(buffer)):
            if buffer[i] is None:
                buffer[i] = number
                break
        
        return self._is_current_buffer_full()
    
    def _calculate_current_buffer_median(self):
        buffer = self.buffers[self.i]
        return np.median(buffer)
    
    def _calculate_weighted_median(self, data, weights):
        # https://gist.github.com/tinybike/d9ff1dad515b66cc0d87
        data, weights = np.array(data).squeeze(), np.array(weights).squeeze()
        s_data, s_weights = map(np.array, zip(*sorted(zip(data, weights))))
        midpoint = 0.5 * sum(s_weights)
        if any(weights > midpoint):
            w_median = (data[weights == np.max(weights)])[0]
        else:
            cs_weights = np.cumsum(s_weights)
            idx = np.where(cs_weights <= midpoint)[0][-1]
            if cs_weights[idx] == midpoint:
                w_median = np.mean(s_data[idx:idx+2])
            else:
                w_median = s_data[idx+1]
        return w_median

    def _clear_current_buffer(self):
        self.buffers[self.i] = [None] * self.buffer_size
    
    def _should_calculate_weighted_median(self):
        for i in range(len(self.buffers) - 1):
            if not self._is_buffer_empty(i):
                return True
        return self.buffers[self.i][1] is not None
    
    def _get_values_from_buffer(self, position: int):
        buffer = self.buffers[position]
        return [n for n in buffer if n is not None]
    
    def push(self, number: int):
        self.i = 0
        if len(self.buffers) == 0:
            self._create_buffer()
        
        while self._insert_number_into_buffer(number):
            number = self._calculate_current_buffer_median()
            self._clear_current_buffer()
            self.i += 1
            self._create_buffer()
    
    def median(self):
        if len(self.buffers) == 0:
            return None
        
        if self._should_calculate_weighted_median():
            values = []
            weights = []
            for i in range(len(self.buffers)):
                buffer_values = self._get_values_from_buffer(i)
                buffer_weights = [self.buffer_size**i] * len(buffer_values)
                values.extend(buffer_values)
                weights.extend(buffer_weights)
            return self._calculate_weighted_median(values, weights)
        else:
            return self.buffers[self.i][0]

# Usage
A = Remedian()
A.push(1)
A.push(2)
A.push(3)
A.push(4)

print(A.median())
```

As you can see Remedian algorithm has a more complex implementation but still is storage efficient. For example, if we have 4 buffers of size 11, we can process `14 641` numbers!

## Testing accuracy and speed of the algorithms

It would be interesting to compare the accuracy and speed of the algorithms. I will use the dataset with [temperature readings from IoT devices](https://www.kaggle.com/datasets/atulanandjha/temperature-readings-iot-devices). Let's compare the speed and memory usage of the two algorithms above plus a textbook way of calculating median (sort and take middle value). How effective would they be in finding the median?

The benchmark code looks like this.

```python
import time
import sys

import pandas as pd

from running_median import RunningMedian
from remedian import Remedian

sensor_data = pd.read_csv('sensor_data_reducted.csv')
print(sensor_data.describe())

running_median = RunningMedian()
remedian = Remedian(11)  # buffer size can be adjusted here

running_median_time = time.time()
sensor_data['temp'].apply(running_median.push)
print("\nRunningMedian, "
      f"median: {running_median.median()}, "
      f"time: {time.time() - running_median_time}")


remedian_time = time.time()
sensor_data['temp'].apply(remedian.push)
print(f"\nRemedian, median: {remedian.median()}, "
      f"time: {time.time() - remedian_time}, buffer size: {remedian.buffer_size}, buffers number: {len(remedian.buffers)}")


def basic_median(data):
    n = len(data)
    s = sorted(data)
    if n % 2:
        return s[n // 2 - 1] / 2.0 + s[n // 2] / 2.0
    else:
        return s[n // 2]

temperatures = sensor_data['temp'].tolist()
default_time = time.time()
median = basic_median(temperatures)
print(f"\nBasic median, median: {median}, "
      f"time: {time.time() - default_time}, memory (in bytes): {sys.getsizeof(temperatures)}")

```

{: .box-note}
**Note:** You can run benchmark code for yourself. Here is a [Github source](https://github.com/VMois/median-in-stream).

`sensor_data_reducted` consists of 97606 rows. For each row, `temp` column contains temperature. Let's look into the results.

```bash
> python benchmark.py
count    97606.000000
mean        35.053931
std          5.699825
min         21.000000
25%         30.000000
50%         35.000000
75%         40.000000
max         51.000000
Name: temp, dtype: float64

RunningMedian, median: 35.05393111079263, time: 0.03059101104736328

Remedian, median: 37.0, time: 0.14257502555847168, buffer size: 11, buffers number: 5

Basic median, median: 35, time: 0.0037550926208496094, memory (in bytes): 780904
```

A few observations:

- The median value provided by `Series.describe()` is the same as from the RunningMedian algorithm. Is it possible `pandas` use the RunningMedian algorithm?

- RunningMedian and Remedian used little memory compared to the basic median (780904 bytes or 762KiB).

- RunningMedian is faster than the Remedian algorithm, but both are *significantly* slower than the basic median.

## Conclusion

This article looked into two interesting algorithms to find the median in streaming data. The idea to write this article came to me while working on a [web analytics tool to track visitors' time on a page](/build-web-analytics-project-from-scratch). I wanted an efficient way of calculating median time on a page. As it turned out, for most situations, the basic method of calculating median is sufficient :sweat_smile: But, in cases where memory usage matters, RunningMedian or Remedian algorithms are helpful. Thank you very much for reading the article! 