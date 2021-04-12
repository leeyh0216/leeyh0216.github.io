---
layout: post
title:  "LeetCode Weekly Contest 178"
date:   2020-03-01 17:30:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

# 참여 결과 및 후기

| Rank        | Score | Finish Time | Q1      | Q2      | Q3      | Q4 |
|-------------|-------|-------------|---------|---------|---------|----|
| 1746 / 9210 | 12    | 1:29:37     | 0:05:14 | 0:20:06 | 1:14:37 | -  |

시간 내에 3문제를 풀어 1746등을 기록했다. 4번 문제는 Contest가 끝나고 10분 뒤에 제출했는데 맞춰버렸다...

2019년 12월 중순부터 시작해서 매주 참여하고 있는데, 2주 전부터 한 두 문제 정도씩 못풀고, 푸는 시간도 전반적으로 늘어났다. 당연히 Rank도 추락하고 있다.

문제가 어려워졌다면 다른 사람도 못 풀 것이기 때문에 Rank가 내려가지 않을 것 같은데, Rank가 내려간 걸 보면 그냥 내가 못하는게 맞다.

평일에 2 ~ 3문제 씩 풀 때 좀 더 타이트하게 풀어야 겠다는 생각이 든다.

# Weekly Contest 178

## 1365. How Many Numbers Are Smaller Than the Current Number

### 문제 설명

입력 배열이 주어졌을 때 배열의 각 요소마다 배열 전체에서 자신보다 작은 값을 가진 요소들의 갯수를 채워 반환하는 문제였다.

1번 예시에서 \[8,1,2,2,3\] 입력 배열이 주어졌을 때, 첫번째 요소인 8은 자기보다 작은 수가 4개(1, 2, 2, 3)이기 때문에 4를 채워 반환하면 된다.

### 접근 방식

나는 다음과 같이 접근해서 풀었다.

1. 입력 배열(ex. \[8,1,2,2,3\])과 동일한 크기의 배열인 newNums 배열 선언
2. newNums 배열에 입력 배열의 값을 채우고 정렬(\[1,2,2,3,8\])
3. 요소 별로 자기보다 작은 값의 수를 저장할 Map 선언(map)
4. newNums 배열을 순회하며 첫번째로 등장하는 값들의 인덱스를 map에 추가
   1. 0번째 요소인 1의 경우 해당 값보다 작은 값이 존재하지 않으므로 0
   2. 1번째 요소인 2의 경우 처음 등장했으므로 1
   3. 2번째 요소는 1번째 요소와 같으므로 Skip
   4. 3번째 요소인 3의 경우 처음 등장했으므로 3
   5. 4번째 요소인 8의 경우 처음 등장했으므로 4
   6. 최종적으로 만들어진 Map: (1 -> 0, 2 -> 1, 3 -> 3, 8 -> 4)
5. nums를 순회하며 map에서 nums\[i\]를 키로 갖는 값을 넣어 반환한다.
   1. 0번째 요소인 8의 경우 4
   2. 1번째 요소인 1의 경우 0
   3. 2번째 요소인 2의 경우 1
   4. 3번째 요소인 2의 경우 1
   5. 4번째 요소인 3의 경우 3

### 소스 코드

{% highlight java %}
class Solution {
    public int[] smallerNumbersThanCurrent(int[] nums) {
        int[] newNums = new int[nums.length];
        Map<Integer, Integer> map = new HashMap<>();
        
        for(int i = 0; i < nums.length; i++)
            newNums[i] = nums[i];
        Arrays.sort(newNums);
        
        map.put(newNums[0], 0);
        int prev = newNums[0];
        for(int i = 1; i < newNums.length; i++){
            if(newNums[i] == prev)
                continue;
            else{
                map.put(newNums[i], i);
                prev = newNums[i];
            }
        }
        
        for(int i = 0; i < nums.length; i++)
            nums[i] = map.get(nums[i]);
        return nums;
    }
}
{% endhighlight %}

## 1366. Rank Teams by Votes

### 문제 설명

입력 배열에 각 팀에 투표를 한 문자열이 입력된다. 예를 들어 \["ABC","ACB","ABC","ACB","ACB"\]와 같은 배열이 입력 배열로 들어왔다고 생각해보자.

첫번째 배열 요소인 "ABC"는 A팀에 1등, B팀에 2등, C팀에 3등을 준다고 투표한 표이다.

두번째 배열 요소도 위와 마찬가지로 A팀에 1등, C팀에 2등, B팀에 3등을 준다고 투표한 표이다.

전체 표를 합산해보면 아래와 같은 결과가 나온다.

* A팀: 1등 5표
* B팀: 2등 2표, 3등 3표
* C팀: 2등 3표, 3등 2표

결과적으로 A팀이 1등, C팀이 2등, B팀이 3등이므로 "ACB"와 같이 반환하면 된다. 

B팀과 C팀의 예시에서 두 팀 모두 1등은 0표씩을 기록했기 때문에 2등 표를 계산하여 등수를 매긴다. 만일 2등 표도 동일했다면 3등 표를 비교했어야 할 것이다. 만일 모든 표수가 동일하다면 알파벳 순서로 우위에 있는 팀이 더 등수가 높은 것으로 산정한다.

### 접근 방식

`Team` 클래스를 만들었다. `Team` 클래스에는 Team 이름과 몇 등을 몇 번 했는지 기록하는 배열이 존재한다. 또한 순위를 매기기 위해(정렬을 수행하기 위해) `Comparable<Team>`을 구현했으며, 아래와 같이 비교한다.

1. 비교 대상 Team과 1등 ~ 26등까지의 표수를 순차적으로 비교하여 더 많은 득표를 한 상황이 오면 -1(앞쪽으로 정렬), 더 적은 득표를 한 상황이 오면 1(뒤쪽으로 정렬)을 반환한다.
2. 만일 1등 ~ 26등까지의 표수가 모두 동일하다면 팀 이름을 기준으로 비교한 값을 반환한다.

### 소스 코드

{% highlight java %}
class Solution {
    class Vote implements Comparable<Vote>{
        char team;
        int[] votes = new int[26];
        
        public Vote(char team){
            this.team = team;
        }
        
        public void addVote(int pos){
            votes[pos]++;
        }
        
        public int compareTo(Vote other){
            for(int i = 0; i < 26; i++){
                if(votes[i] < other.votes[i])
                    return 1;
                else if(votes[i] > other.votes[i])
                    return -1;
            }
            
            if(team < other.team)
                return -1;
            else
                return 1;
        }
        
    }
    public String rankTeams(String[] votes) {
        Map<Character, Vote> teamVotes = new HashMap<>();
        for(int i = 0; i <votes.length; i++){
            for(int j = 0; j < votes[i].length(); j++){
                char currentTeam = votes[i].charAt(j);
                if(teamVotes.containsKey(currentTeam)){
                    Vote v = teamVotes.get(currentTeam);
                    v.addVote(j);
                }
                else{
                    Vote v = new Vote(currentTeam);
                    v.addVote(j);
                    teamVotes.put(currentTeam, v);
                }
            }
        }
        
        Iterator iter = teamVotes.keySet().iterator();
        List<Vote> list = new ArrayList<>();
        while(iter.hasNext()){
            list.add(teamVotes.get(iter.next()));
        }
        Collections.sort(list);
        StringBuilder sb = new StringBuilder();
        for(int i = 0; i < list.size(); i++){
            sb.append(list.get(i).team);
        }
        return sb.toString();
    }
}
{% endhighlight %}

## 1367. Linked List in Binary Tree

### 문제 설명

Linked List와 Tree가 주어진다. Tree의 Root에서 Leaf Node까지 연결하며 만들어낸 문자열이 Linked List에 포함된 문자열을 포함하면 true, 포함하지 않는다면 false를 반환하는 문제이다.

### 접근 방식

DFS로 Leaf Node까지 문자열을 생성해나가고, Leaf Node에서는 생성된 문자열을 List에 저장한다.

List에 저장된 문자열 중 Linked List의 문자열을 포함하는 문자열이 있다면 true를 반환하고 아닌 경우 false를 반환하도록 한다.

### 소스 코드

{% highlight java %}
class Solution {
    
    String target = "";
    List<String> list = new ArrayList<>();
    
    void visit(TreeNode node, String str){
        if(node == null)
            return;
        if(node.left == null && node.right == null){
            list.add(str + node.val);
            return;
        }
        visit(node.left, str + node.val);
        visit(node.right, str + node.val);
    }
    
    public boolean isSubPath(ListNode head, TreeNode root) {
        List<Integer> path = new ArrayList<>();
        StringBuilder sb = new StringBuilder();
        while(head != null){
            sb.append(head.val);
            head = head.next;
        }
        target = sb.toString();
        visit(root, "");
        for(int i = 0; i < list.size(); i++)
            if(list.get(i).contains(target))
                return true;
        return false;
    }
}
{% endhighlight %}

## 1368. Minimum Cost to Make at Least One Valid Path in a Grid

### 문제 설명

Grid를 순회하며 (0, 0) 지점에서 (m - 1, n - 1) 지점까지 가는 비용을 계산하는 방법이다.

여기서 비용이라는 용어가 나오는데, 각 Cell에는 Cell에서 이동 가능한 방향이 나온다. 이 이동 방향을 전혀 수정하지 않고 (m - 1, n - 1)까지 도달하면 비용이 0이 된다. 방향을 1번 수정할 때마다 비용이 1이 추가되고, 하나의 Cell에서 방향은 1번 밖에 바꿀 수 없다.

### 접근 방식

BFS로 풀이하였다.

1. (0, 0) 지점의 Cell을 Queue에 넣는다.
2. Queue에서 Item을 꺼낸다.
3. Item의 x, y가 (m - 1, n - 1)이라면 현재까지의 최소 비용과 비교하여 갱신을 수행한다.
4. Item의 x, y가 (m - 1, n - 1)이 아니라면 원래 방향과 다른 세 방향으로의 Item을 만들어 Queue에 추가한다.
5. 3 ~ 4의 과정을 반복한다.
6. 기록된 최소 비용을 반환한다.

3, 4 과정에서 주의해야 할 점은 x, y가 Grid를 벗어났을 때의 처리와 방문했던 지점을 다시 방문하지 않게 하는 부분이 중요하다.

다만 네 방향으로 움직이기 때문에 하나의 Cell은 최대 4번 방문할 수 있는데, 해당 Cell에 기록된 최소 비용과 현재 방문에서의 비용을 비교하여 현재 방문에서의 비용이 더 적은 경우에만 Cell을 방문하도록 처리하면 된다.

### 소스 코드

{% highlight java %}
class Solution {
    class Item{
        int yPos = 0;
        int xPos = 0;
        int cost = 0;
        
        public Item(int xPos, int yPos, int cost){
            this.xPos = xPos;
            this.yPos = yPos;
            this.cost = cost;
        }
    }
    
    int[][] visited;
    int[] x = new int[]{1, -1, 0, 0};
    int[] y = new int[]{0, 0, 1, -1};
    
    public int minCost(int[][] grid) {
        visited = new int[grid.length][grid[0].length];
        for(int i = 0; i < visited.length; i++){
            for(int j = 0; j < visited[i].length; j++)
                visited[i][j] = Integer.MAX_VALUE;
        }
        Queue<Item> queue = new LinkedList<>();
        queue.offer(new Item(0, 0, 0));
        
        int min = Integer.MAX_VALUE;
        while(!queue.isEmpty()){
            Item next = queue.poll();
            if(visited[next.yPos][next.xPos] <= next.cost)
                continue;
            visited[next.yPos][next.xPos] = next.cost;
            if(next.yPos == grid.length - 1 && next.xPos == grid[0].length - 1){
                min = Math.min(min, next.cost);
                continue;
            }
            int nextXPos = next.xPos + x[grid[next.yPos][next.xPos] - 1], nextYPos = next.yPos + y[grid[next.yPos][next.xPos] - 1];
            if(nextXPos >= 0 && nextYPos >= 0 && nextYPos < grid.length && nextXPos < grid[0].length){
                queue.offer(new Item(nextXPos, nextYPos, next.cost));
            }
            for(int i = 1; i <= 3; i++){
                nextXPos = next.xPos + x[(grid[next.yPos][next.xPos] - 1 + i) % 4];
                nextYPos = next.yPos + y[(grid[next.yPos][next.xPos] - 1 + i) % 4];
                if(nextXPos >= 0 && nextYPos >= 0 && nextYPos < grid.length && nextXPos < grid[0].length){
                    queue.offer(new Item(nextXPos, nextYPos, next.cost + 1));
                }
            }
        }
        return min;
        
    }
}
{% endhighlight %}