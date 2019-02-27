# 题目

输入一个递增排序的数组和一个数字s，在数组中查找两个数，使得他们的和正好是s。如果有多对数字的和等于S，输出任意一对即可。例如输入数组[1,2,4,7,11,15]和数字15，输出[4,11]。

# 思路

由于数组是递增排序的，初始时用lo指示数组的第一个数，hi指示数组的最后一个数，当array[lo]+array[hi]<sum时，增加lo；当array[lo]+array[hi]>sum时，减小hi；当array[lo]+array[hi]==sum时，即找到了这两个数。

```java
public List<Integer> findNumbersWithSum(int[] nums, int s) {
    if (nums == null || nums.length == 0 || s < nums[0]) return new ArrayList<>();
    List<Integer> result = new ArrayList<>();
    int lo = 0;
    int hi = nums.length - 1;

    while (lo < hi) {
        int currSum = nums[lo] + nums[hi];
        if (currSum == s) {
            result.add(nums[lo]);
            result.add(nums[hi]);
            break;
        } else if (currSum < s) {
            lo++;
        } else {
            hi--;
        }
    }

    return result;
}
```

# 扩展

和为s的连续正数序列。输入一个正数sum，打印出所有和为s的连续正数序列（至少含有两个数）。例如输入15，由于1+2+3+4+5=4+5+6=7+8=15，所以结果打印出3个连续序列1-5,4-6,7-8。

# 思路

我们可以用small和big两个指针指示正数序列，small指示序列的第一个数，big指示序列的最后一个数。初始时small置为1，big置为2，然后，我们判断当前序列的和是大于还是小于sum：如果小于sum，我们增大big以让序列包含更多的数；如果小于sum，我们增大small以减小序列的和；如果等于sum我们则将当前序列打印出来。由于序列至少需要包含2个数，因此small至多只能增加到(1+s)/2。

```java
public List<List<Integer>> findContinuousSequence(int sum) {
    if (sum < 3) return new ArrayList<>();
    List<List<Integer>> result = new ArrayList<>();
    int small = 1;
    int big = 2;
    int mid = (1 + sum) / 2;
    int currSum = small + big;

    while (small < mid) {
        if (currSum == sum) {
            addSequence(small, big, result);
            // 添加完序列后继续增大big以使序列包含更多的数
            big++;
            currSum += big;
        } else if (currSum < sum) {
            big++;
            currSum += big;
        } else {
            currSum -= small;
            small++;
        }
    }

    return result;
}

// 将序列[small, ..., big]添加到result中
private void addSequence(int small, int big, List<List<Integer>> result) {
    List<Integer> temp = new ArrayList<>();
    for (int i = small; i <= big; i++) {
        temp.add(i);
    }
    result.add(temp);
}
```

