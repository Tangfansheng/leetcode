## leetcode数据结构与算法

### 三数之和

```java
//感受就是做题前一定要考虑方法适不适合

public class threeSum {
    public List<List<Integer>> threeSum(int[] nums) {
        if (nums == null || nums.length < 3) {
            return null;
        }
        //考虑arr[i]时 去剩余的部分找是否有和为0-arr[i]的二元组
        List<List<Integer>> res = new ArrayList<>();
        HashSet<String> set = new HashSet<>();//去重
        HashMap<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            map.put(nums[i], i);
        }
        for (int i = 0; i < nums.length; i++) {
            List<Integer> list = new ArrayList<>();
            int[] arr = twoSum(nums, i, map, set);
            if (arr != null) {
                for (int value : arr) {
                    list.add(value);
                }
                res.add(list);
            }
        }
        return res;
    }

    public int[] twoSum(int[] nums, int index, HashMap<Integer, Integer> map, HashSet<String> set) {
        //不需要判空
        int target = 0 - nums[index];
        int[] res = new int[3];
        for (int i = 0; i < nums.length && i!=index ; i++) {
            int aim = target - nums[i];
            if (map.containsKey(aim) && map.get(aim) != index && map.get(aim) != i) {
                res[0] = nums[i];
                res[1] = nums[map.get(aim)];
                res[2] = nums[index];
                Arrays.sort(res);
                String str = res[0] + "_" + res[1] + "_" + res[2];
                if (!set.contains(str)) {
                    set.add(str);
                    return res;
                }

            }
        }
        return null;
    }
}

```

这道题一开始想就是从TwoSum改过来 就延续了用Hashmap<nums[i]. i>记录值的这种方法。但这是一个挺显而易见的问题，key会被覆盖掉，对于[6,-3,-3]这种答案，在存map里面的时候会存在覆盖的问题啊。难过，侥幸过了188个算例，改了半天终于想到是这方法根本不行。



