- 给定一个整形数组和一个目标值，返回数组中两数相加的的数字索引。(假定输入数组中数字不重复且输出唯一)
```
//Input: [2 5 7 9]  target = 7
//Output: [0 1]
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        
        unordered_map<int, int> result;
        
        for (int i = 0; i < nums.size(); i++) {
            if (result.find(target - nums[i]) != result.end()) {
                vector<int> ret;
                ret.push_back(result[target-nums[i]]);
                ret.push_back(i);
                return ret;
            }
            else
                result[nums[i]] = i;
        }
        
        return vector<int>{-1, -1};
    }
};
```

- 两条链表逆序存储着非负各位整数，将这两条链表相加输出一条链表
```
//Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
//Output: 7 -> 0 ->8
//Explanation: 342 + 465 = 807
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        ListNode* result = new ListNode(0);
        ListNode* root = result;
        int carry = 0;
        
        while (l1 != NULL || l2 != NULL) {
            if (l1 != NULL) {
                result->val += l1->val;
                l1 = l1->next;
            }
            if (l2 != NULL) {
                result->val += l2->val;
                l2 = l2->next;
            }
            carry = result->val/10;
            result->val %= 10;
            
            if (l1 != NULL || l2 != NULL || carry != 0) {
                result->next =  new ListNode(carry);   
            }
            result = result->next;
        }
        return root;
    }
};
```

- 给定一个字符串，找到其中非重复字符的最长子串
```
//Input1: "abcabcbb" Output1: "abc"
//Input2: "bbbbbbb"  Output2: "b"
//Input3: "pwwkew"   Output3: "wke"
class Solution {
public:
    //思路：利用额外空间存储当前字符出现过，i为此时字符串的下标，只要取得字串的开始下标即可
    int lengthOfLongestSubstring(string s) {
        vector<int> charIndexAsSize(256);               //将每一个字符的ascii值作为下标使用，且这个下标代表的就是字串的长度
        int start = 0, maxLen = 0;                      //代表连续非重复字串的起始下标
        
        for(int i = 0; i < s.size(); i++)
        {
            if(charIndexAsSize[s[i]] > start )          //出现重复字符，更新start
                start = charIndexAsSize[s[i]];          //更新start，令start=前一个重复字符的下标
            
            charIndexAsSize[s[i]] = i + 1;              //i+1表示的长度
            maxLen = max(maxLen, i + 1 - start);        //更新最大长度
        }
        return maxLen;
    }
};
```

- 给定n个非负整数：a1,a2,...an,每一个整数表示了一个坐标点(i, ai).每一个点可以画一天垂直的线，找到两条线，这两条线构成的容器能够装最多的水；
```
//装水容器需要满足木桶原理，加速算法的关键在于最大限度的排除不可能的情况
class Solution {
public:
    int maxArea(vector<int>& height) {
        int i = 0, j = height.size()-1;
        int area = 0;
        int min = 0;
        int ret = 0;
        while (i < j) {
            min = std::min(height[i], height[j]);
            area = min*(j-i);
            ret = std::max(ret, area);
            for (; height[i] <= min; i++);
            for (; height[j] <= min; j--);
        }
        return ret;
    }
};
```

- 给定一个包含n个整数的数组和一个整数，在数组中找到3个数字使得这三个数字的和最接近target
```
//Input: [-1, 2, 1, -4] target: 1
//最接近1的三数之和是2. (-1 + 2 + 1 = 2)

// 0 1 2 3 4 5 6 7 8
// 1 2             3
// sort(vector)
class Solution {
public:
    int threeSumClosest(vector<int>& nums, int target) {
        int size = nums.size();
        int first = 0, second = 1, third = size - 1;
        int closestSum = nums[0] + nums[1] + nums[2];
        int currentSum = 0;
        std::sort(nums.begin(), nums.end());
        
        for (; first < size - 2; first++) {
            second = first + 1;
            third = size - 1;
            while (second < third) {              
                currentSum = nums[first] + nums[second] + nums[third];
                 if (currentSum == target) return currentSum;
                closestSum = abs(target - closestSum) < abs(target - currentSum) ? closestSum : currentSum;
                
                if (currentSum < target) second++;
                else third--;
            }
        }
        
        return closestSum;
    }
};
```

- 给定一个数组和一个目标数字，在数组中找到a，b，c，d四个元素，使得a + b  + c + d = target，找到所有的情况
```
//Input: [1, 0, -1, 0, -2, 2] target: 0
//Output:
//[
// [-1, 0, 0, 1],
// [-2, -1, 1, 2],
// [-2, 0, 0, 2]
//]
class Solution {
  public:
    vector<vector<int>> fourSum(vector<int>& nums, int target) {
      vector<vector<int>> ret;
      int size = nums.size();
      int first = 0, second = 1, third = 2, four = size - 1;
      int current = target;
      sort(nums.begin(), nums.end());
      for (; first < size - 3 ; first++) {
        current -= nums[first];
        for (second = first + 1; second < size - 2; second++) {
          current -= nums[second];
          third = second + 1;
          four = size - 1;
          while(third < four) {
            if (nums[third] + nums[four] < current) third++;
            else if (nums[third] + nums[four] > current) four--;
            else {
              ret.push_back({nums[first], nums[second], nums[third], nums[four]});
              four--;
              while (four <= size - 1 && nums[four] == nums[four+1]) four--;
              while (third <= four && nums[third] == nums[third+1]) third++;
            }
          }
          current += nums[second];
          while (second <= third && nums[second] == nums[second+1]) second++;
        }
        current += nums[first];
        while (first <= second && nums[first] == nums[first+1]) first++;
      }

      return move(ret);
    }
};
```

- 给定一个仅包含'('和')'的字符串，找到最长有效的字符串
```
//Input: "(()"
//Output: 2
//Explanation: valid substring is "()"
//Input: ")()())"
//Output: 4
//Explanation: valid substring is "()()"

class Solution {
public:
    int longestValidParentheses(string s) {
        int size = s.size(), max = 0, sum = 0;
        for (int i = 0; i < size; i++) {
            if (s[i] == ')') continue;
            sum = 1;
            for (int j = i+1; j < size;j++) {
                if (s[j] == '(') sum +=1;
                else sum -= 1;
                
                if (sum == 0) {
                    max = max > (j-i+1) ? max : (j-i+1);
                    if (j+1<size && s[j+1] != '(') {
                        i = j;
                        break;
                    }
                }
            }
        }
        
        return max;
    }
};
```

- 给定一个排序数组和一个目标值，如果目标值在数组中出现，返回其下标，如果没有，返回目标值应该插入的位置
```
//Input: [1, 3, 5, 6] target: 5
//Output: 2
//Input: [1, 3, 5, 6] target: 2
//Output: 1
//Input: [1,3, 5, 6] target: 7
//Output: 4
//Input: [1,3, 5, 6] target: 0
//Output: 0

//二分查找
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
    
        if (target <= nums[0]) return 0;
        int size = nums.size();
        if (target > nums[size-1]) return size;
        if (target == nums[size-1]) return size-1;
        int first = 0, last = size - 1, middle = 0;
        while (first <= last) {
            middle = (first+last+1)>>1;
            if (nums[middle] < target) first = middle+1;
            else last = middle-1;                      
        }
        return first;
        
        //return lower_bound(nums.begin(), nums.end(), target) - nums.begin();
    }
};
```
