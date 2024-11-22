+++
title = "Re-visit Sorting algorithms with C++."
date = 2024-11-19
draft = false
[taxonomies]
  tags = ["C++", "Algorithm"]
[extra]
  toc = true
	keywords = "C++, algorithm, insert sort, bubble sort, heap sort, merge sort, quick sort"
+++

All kinds of sorting algorithms, re-visited!

## Easiest

### Bubble Sort

Bubble sort is the easiest sorting algorithm, it can be considered as an inferior version of the insert sort.

You should **never** use bubble sort in any applications. It has no advantages at all compared to the other sorting algorithms.

The basic concept is we traverse the vector and compare `[i]` and `[i-1]`, if they are not the order we want, we swap, and move on to the next pair.

We need to do at most `vec.size()-1` traversals for a vector `vec`.

```cpp
void bubble_sort(vector<int>& vec) {
    for (int i = 1; i < vec.size(); i++) {
        for (int j = 1; j < vec.size(); j++) {
            if (vec[j] < vec[j - 1]) {
                swap(vec[j], vec[j - 1]);
            }
        }
    }
}
```

### Bucket Sort

Bucket sort is probably also the easiest sorting algorithm, and it is also very useful for it runs in `O(N)` time.

It is optimal when the sorted value deviation is small.

```cpp
void bucket_sort(vector<int>& vec) {
    // If values in 0-100.
    auto bucket = array<int, 100>({0});
    for (auto v : vec) {
        bucket[v]++;
    }
    vec.clear();
    for (auto i = 0; i < 100; i++) {
        for (auto j = 0; j < bucket[i]; j++) {
            vec.push_back(i);
        }
    }
}
```

## Easy But Not Efficient

### Insertion Sort

It works really well when the sorted values are close to being fully sorted.

It is very similiar to the bubble sort, except for every traversal, we bubble values from back to front.

```cpp
void insert_sort(vector<int>& vec) {
    for (auto i = 1; i < vec.size(); i++) {
        for (auto j = i; j > 0 && vec[j - 1] > vec[j]; j--) {
            swap(vec[j - 1], vec[j]);
        }
    }
}
```

It works better than bubble sort although they have the same worst time complexity `O(N^2)`. The heap sort in the next category can be considered as the superior version of it.

### Shell Sort

Well, slightly optimized insertion sort, the only difference is we added an additional gaps for the bubble steps.

There are things like binary insertion sort, which is basically same as this.

```cpp
void shell_sort(vector<int>& vec) {
    auto gaps = vector<int>({4, 2, 1});
    for (auto gap : gaps) {
        for (auto i = 1; i < vec.size(); i++) {
            for (auto j = i; j >= gap && vec[j - gap] > vec[j]; j--) {
                swap(vec[j - gap], vec[j]);
            }
        }
    }
}
```

## Efficient

### Merge Sort

Divide and conquer sort, you need to think this recursively:

* Break a vector into 2 vectors (same length).
* Sort the 2 vectors independently so they are in order.
  * If one of the 2 vector is null, return the other vector directly.
* Merge back the 2 pieces with 2 pointer algorithm.

A special technique used in the following C++ code is we pre-defined a `buffer` vector to store the merged results, which speeds up the sort process a lot.

```cpp
// Buffer is used as internal buffer, so we don't allocate when sorting
void merge_sort(vector<int>& vec, vector<int>& buffer, int start_i, int end_i) {
    if (end_i - start_i <= 1) {
        return;
    }
    // Divide
    int mid = (start_i + end_i) / 2;
    merge_sort(vec, buffer, start_i, mid);
    merge_sort(vec, buffer, mid, end_i);
    // Merge
    buffer.clear();
    auto i = start_i;
    auto j = mid;
    while (true) {
        if (i >= mid) {
            copy(vec.begin() + j, vec.end() + end_i, buffer.end());
            break;
        }
        if (j >= end_i) {
            copy(vec.begin() + i, vec.end() + mid, buffer.end());
            break;
        }
        if (vec[i] <= vec[j]) {
            buffer.push_back(vec[i]);
            i++;
        } else {
            buffer.push_back(vec[j]);
            j++;
        }
    }
    copy(buffer.begin(), buffer.end(), vec.begin() + start_i);
}
```

#### Optimized Version

I have discussed this with one of my friend and he said we could avoid copying by reciprocally switch `buffer` and `vec` between recursive calls.

So this is a better version of the last C++ `merge_sort` code, without `clear()` and `copy()` at the end of each recursive call.

```cpp
void merge_sort(vector<int>& vec, vector<int>& buf, int start, int end) {
    if (end - start <= 1) {
        return;
    }
    // Divide
    int mid = (start + end) / 2;
    merge_sort(buf, vec, start, mid);
    merge_sort(buf, vec, mid, end);
    // Merge
    auto i = start;
    auto j = mid;
    auto offset = 0;
    while (true) {
        if (i >= mid) {
            copy(vec.begin() + j, vec.begin() + end,
                 buf.begin() + start + offset);
            break;
        }
        if (j >= end) {
            copy(vec.begin() + i, vec.begin() + mid,
                 buf.begin() + start + offset);
            break;
        }
        if (vec[i] <= vec[j]) {
            buf[start + offset] = vec[i];
            i++;
        } else {
            buf[start + offset] = vec[j];
            j++;
        }
        offset++;
    }
}
```

At the beginning you should make sure `buf` is a copy of `vec`.

### Heap Sort

Heap sort is a little different, as it is very efficient for getting min or max one by one. It is used a lot to implement things like `priority_queue` data structure.

Compared to merge sort and quick sort, it has several advantages:

* Heap sort requires no extra space, which makes it suitable for large data structure.
* The heap data structure is good for inserting new value in the runtime, as well as popping the min/max value.
* It is an upgraded version of the **insertion sort**, works really well with nearly sorted data.

However it is slower than merge sort in most cases. If all we want is sort an array, merge sort is the winner here.

After the vector is **made into a heap** in a certain order (which takes `O(N)` and its proof is hard), read the min/max value will takes `O(1)`.

If the min/max element is popped or written over, the mantainence will only take `O(logN)` time.

Heap sort can be described as first making a vector into a heap, then pop the first element one by one. The pop is done by swapping head and tail elements, reduce the vector size by 1, then do sink the new top element.

Heap itself can be seen as a **complete binary tree** data structure. The time complexity for heap sort is very stable.

C++, since C++98 has `make_heap()` function, which is quite useful, and we are going to implement this `make_heap()`, `pop_heap()` function by hand.

#### `make_heap()`

We can define a helper function called `sink()`, it sinks a vector node to the correct location on the heap.

Then we sink every value from the vector end to begin, as a result, creating the **maximum/minimum heap** structure.

```cpp
void sink(vector<int>& vec, const int n, int i) {
    while (true) {
        auto min_i = i;
        auto min_val = vec[i];
        if (2 * i + 2 < n && vec[2 * i + 2] < min_val) {
            min_i = 2 * i + 2;
            min_val = vec[2 * i + 2];
        }
        if (2 * i + 1 < n && vec[2 * i + 1] < min_val) {
            min_i = 2 * i + 1;
            min_val = vec[2 * i + 1];
        }
        if (min_i == i) {
            return;
        } else {
            vec[min_i] = vec[i];
            vec[i] = min_val;
            i = min_i;
        }
    }
}

void make_heap(vector<int>& vec) {
    for (auto j = vec.size(); j > 0; j--) {
        auto i = j - 1;
        sink(vec, vec.size(), i);
    }
}
```

#### `pop_heap()`

`pop_heap()` will move the first element to the end of the vector, then do a heap update so that the heap is still valid.

This process can be taken as:

* Swap the begin and end element, and reduce the size of the `vec` by 1.
  * You can either directly remove the popped element or only move the popped item to the end.
  * The default C++ `pop_heap()` will only move the popped item to the end.
* Sink the new top element so the heap is still valid.
  * If you only moved at the first step, you need to remember the last element location.

Because we need to swap at first, we need an `empty()` check.

```cpp
void pop_heap(vector<int>& vec) {
    if (vec.empty()) {
        return;
    }
    swap(vec[0], vec[vec.size() - 1]);
    sink(vec, vec.size() - 1, 0);
}
```

#### `insert_heap()`

`insert_heap` is like reverse of `pop_heap()`, we append the element to the end of the vector, then bubble up the end element.

Because the bubble-up process doesn't break the heap validity, we don't need to sink afterwards.

For child index at `i` we can use `(i-1)/2` to get the parent index.

```cpp
void insert_heap(vector<int>& vec, int val) {
    vec.push_back(val);
    for (auto i = vec.size()-1; i > 0;) {
        if ((i - 1) / 2 >= 0 && vec[(i - 1) / 2] > vec[i]) {
            swap(vec[(i - 1) / 2], vec[i]);
            i = (i - 1) / 2;
        } else {
            break;
        }
    }
}
```

### Quick Sort

Quick sort can be considered as upgraded version of merge sort, although its worst time complexity is `O(N^2)`. It is also based on the **divide and conquer** concept.

However on average it performs better than the merge sort. And the space complexity is `O(N)` for the worst case.

The following code uses **Hoare partition algorithm**, which can be considered as 2 pointer method, and we ping-pong the pivot element to one of the pointer and increment the other pointer, until these 2 pointers across.

Notice it only works when we pick the first element as the pivot element. If random pivot is given, we need to swap the random pivot element and the first element beforing using this algorithm.

```cpp
int partition(vector<int>& vec, int low, int high) {
    auto pivot = vec[low];
    auto i = low - 1;
    auto j = high + 1;
    while (true) {
        do {
            i++;
        } while (vec[i] < pivot);
        do {
            j--;
        } while (vec[j] > pivot);
        if (i >= j) {
            return j;
        }
        swap(vec[i], vec[j]);
    }
}

void quick_sort(vector<int>& vec, int low, int high) {
    if (high <= low) {
        return;
    } else {
        auto mid = partition(vec, low, high);
        quick_sort(vec, low, mid - 1);
        quick_sort(vec, mid + 1, high);
    }
}
```