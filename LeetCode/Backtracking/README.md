# 回溯法题目汇总

对于在递归函数参数中有 start 并且循环从 start 开始的题 (如Subsets II, Combination Sum II)，如果题目中说数组中可能包含重复的数，但是要求结果不能有重复的，则我们在构造的过程中需要跳过重复元素，此时使用语句`if (i > start && nums[i] == nums[i - 1]) continue;`，这是因为在这类题中我们是从 start 开始构造的，如果在 start 之后我们发现当前数和前一个数相同，说明对于 start 这个位置来说选取当前数和选取前一个数的作用相同，而选取前一个数的结果我们已经得到了，为了避免重复此时就不应该再选取当前数了。而对于需要在递归函数循环中需要从头开始遍历数组并用 vis 记录某个元素是否被使用的题 (Permutations II)，就不能用上面这条语句了， 此时除了在当前数已经被使用过的情况下跳过外，如果当前数和前一个数相同并且前一个数还没有被使用过，我们也需要跳过，因为如果继续使用该数的话，那么接着会使用前一个数，而这和第一次使用前一个数再使用该数生成的结果相同。
