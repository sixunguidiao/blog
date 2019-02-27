# 题目

输入n个整数， 找出其中最小的k个数字。例如输入[4,5,1,6,2,7,3,8]，则最小的k个数字是1,2,3,4。

# 思路1

把数组进行排序，排在最前面的k个数就是最小的k个数。代码略。

时间复杂度：O(NlogN)

# 思路2

从“数组中出现次数超过一半的数字”中我们可以得到启发，我们同样可以用partition函数来解决这个问题。我们基于数组的第k个数字来进行调整，使得所有小于第k个数字的数都排在它的左边，所有大于第k个数字的数都排在它的右边。这样调整之后，位于数组左边的k个数字就是最小的k个数字（但是这k个数字不一定是排好序的）。

```java
public List<Integer> GetLeastKNumbers(int[] nums, int k) {
    if (nums == null || nums.length == 0 || k < 1 || k > nums.length) return new ArrayList<>();
    List<Integer> result = new ArrayList<>();
    int lo = 0;
    int hi = nums.length - 1;
    int index = partition(nums, lo ,hi);
    
    while (index != k - 1) {
        if (index < k - 1) {
            index = partition(nums, index + 1, hi);
        } else {
            index = partition(nums, lo, index - 1);
        }
    }
    
    for (int i = 0; i < k; i++) {
        result.add(nums[i]);
    }
    
    return result;
}

private int partition(int[] nums, int lo, int hi) {
    if (lo == hi) return nums[lo];	// 由于下面的random.nextInt(hi - lo)要求hi - lo大于0，因此这里多加一个判断
    
    // 从当前数组中随机选择一个数作为基准数pivot
    Random random = new Random();
    int pivot_index = lo + random.nextInt(hi - lo);
    int pivot = nums[pivot_index];
    
    // 将pivot移到末尾
    swap(nums, pivot_index, hi);
    int store_index = lo;
    
    // 将所有小于pivot的数移到左边，此时，所有大于pivot的数自动聚集到右边
    for (int i = lo; i < hi; i++) {
        if (nums[i] < pivot) {
            swap(nums, store_index++, i);
        }
    }
    
    // 将pivot放到它最终的位置上
    swap(nums, store_index, hi);
    
    return stroe_index;
}

private void swap(int[] nums, int i, int j) {
    if (i != j) {
        int t = nums[i];
        nums[i] = nums[j];
        nums[j] = t;
    }
}
```

时间复杂度：O(NlogN)

# 思路3

由于是找最小的k个数，我们可以先用一个大顶堆保存数组的前k个数，之后从k+1开始遍历数组中剩下的数，如果当前数比堆顶元素小，将堆顶元素出堆，同时将当前元素进堆。遍历完成后堆中保存的即是最小的k个数。

注意，Java默认创建的是小顶堆。

```java
public List<Integer> GetLeastKNumbers(int[] nums, int k) {
    if (nums == null || nums.length == 0 || k < 1 || k > nums.length) return new ArrayList<>();
    // 创建一个大顶堆
    PriorityQueue<Integer> pq = new PriorityQueue<>(new Comparator<Integer>() {
        @Override
        public int compare(Integer o1, Integer o2) {
            return o2 - o1;
        }
    });

    for (int i = 0; i < k; i++) {
        pq.offer(nums[i]);
    }
    for (int i = k; i < nums.length; i++) {
        int top = pq.peek();
        if (nums[i] < top) {
            pq.poll();
            pq.offer(nums[i]);
        }
    }
    ArrayList<Integer> result = new ArrayList<>(pq);

    return result;
}
```

时间复杂度：O(NlogK)

# 解法比较

|                      | 基于partition函数的解法 | 基于堆或红黑树的解法 |
| :------------------: | :---------------------: | :------------------: |
|      时间复杂度      |          O(N)           |       O(NlogK)       |
| 是否需要修改输入数组 |           是            |          否          |
|  是否适用于海量数据  |           否            |          是          |

