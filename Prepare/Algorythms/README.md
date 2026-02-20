# Алгоритмы и структуры данных

Полезные ссылки:
- [NeetCode — структурированная подготовка + видео](https://neetcode.io/)
- [LeetCode — задачи для практики](https://leetcode.com/)
- [LeetCode Top Interview 150](https://leetcode.com/studyplan/top-interview-150/) — 150 задач от LeetCode
- [NeetCode 150](https://neetcode.io/practice) — 150 задач с разбором на видео
- [Blind 75 LeetCode](https://leetcode.com/discuss/general-discussion/460599/blind-75-leetcode-questions) — 75 классических задач
- [Visualgo — визуализация алгоритмов](https://visualgo.net/)
- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/)

---

## Оценка сложности алгоритмов (Big O)

**Big O** — описывает поведение алгоритма при увеличении входных данных (наихудший случай).

```
O(1)        < O(log n)  < O(n)  < O(n log n) < O(n²) < O(2ⁿ) < O(n!)
константа    логарифм    линейная  линейно-лог  квадрат  экспонента факториал
```

| Сложность | 10 элементов | 100 | 10,000 | Пример |
|-----------|-------------|-----|--------|--------|
| O(1) | 1 | 1 | 1 | Доступ к элементу массива |
| O(log n) | 3 | 7 | 14 | Бинарный поиск |
| O(n) | 10 | 100 | 10,000 | Линейный поиск |
| O(n log n) | 30 | 700 | 140,000 | Merge sort, Heap sort |
| O(n²) | 100 | 10,000 | 10⁸ | Bubble sort, вложенный цикл |
| O(2ⁿ) | 1024 | 10³⁰ | — | Все подмножества |
| O(n!) | 3.6M | — | — | Все перестановки |

**Пространственная сложность** (память) тоже оценивается Big O.

---

## Базовые структуры данных

### Массив (Array)

```go
arr := [5]int{1, 2, 3, 4, 5}  // фиксированный
sl  := []int{1, 2, 3, 4, 5}   // слайс (динамический)
```

| Операция | Сложность |
|----------|-----------|
| Доступ по индексу | O(1) |
| Поиск | O(n) |
| Вставка/удаление в конец | O(1) amortized |
| Вставка/удаление в середину | O(n) |

### Связный список (Linked List)

```go
type ListNode struct {
    Val  int
    Next *ListNode
}
```

| Операция | Сложность |
|----------|-----------|
| Доступ по индексу | O(n) |
| Вставка/удаление в начало | O(1) |
| Вставка/удаление в конец | O(1) с tail-pointer |
| Поиск | O(n) |

**Задачи:**
- [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) ⭐
- [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) ⭐
- [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) ⭐
- [19. Remove Nth Node From End](https://leetcode.com/problems/remove-nth-node-from-end-of-list/)
- [143. Reorder List](https://leetcode.com/problems/reorder-list/)
- [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) 🔥

### Стек (Stack) — LIFO

```go
// Используем слайс как стек
stack := []int{}
stack = append(stack, 1)        // push
val := stack[len(stack)-1]      // peek
stack = stack[:len(stack)-1]    // pop
```

**Задачи:**
- [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) ⭐
- [155. Min Stack](https://leetcode.com/problems/min-stack/) ⭐
- [739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures/)
- [84. Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) 🔥

### Очередь (Queue) — FIFO

```go
// Двусторонняя очередь (deque)
import "container/list"
q := list.New()
q.PushBack(1)
q.Front().Value
q.Remove(q.Front())
```

**Задачи:**
- [225. Implement Stack using Queues](https://leetcode.com/problems/implement-stack-using-queues/)
- [232. Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/)
- [239. Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) 🔥

### HashMap

```go
m := make(map[string]int)
m["key"] = 1
if v, ok := m["key"]; ok { use(v) }
delete(m, "key")
```

| Операция | Средний | Худший |
|----------|---------|--------|
| Get/Set/Delete | O(1) | O(n) |

**Задачи:**
- [1. Two Sum](https://leetcode.com/problems/two-sum/) ⭐
- [49. Group Anagrams](https://leetcode.com/problems/group-anagrams/) ⭐
- [128. Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/)
- [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/)

---

## Алгоритмы поиска

### Линейный поиск — O(n)

```go
func linearSearch(arr []int, target int) int {
    for i, v := range arr {
        if v == target {
            return i
        }
    }
    return -1
}
```

### Бинарный поиск — O(log n)

Требует **отсортированного** массива.

```go
func binarySearch(arr []int, target int) int {
    left, right := 0, len(arr)-1
    for left <= right {
        mid := left + (right-left)/2  // избегаем overflow vs (l+r)/2
        if arr[mid] == target {
            return mid
        } else if arr[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}
```

**Задачи:**
- [704. Binary Search](https://leetcode.com/problems/binary-search/) ⭐
- [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) ⭐
- [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)
- [4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/) 🔥

---

## Алгоритмы сортировки

### Bubble Sort — O(n²)

```go
func bubbleSort(arr []int) {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
}
```

### Merge Sort — O(n log n), стабильная

Разделяй и властвуй (Divide & Conquer):

```go
func mergeSort(arr []int) []int {
    if len(arr) <= 1 {
        return arr
    }
    mid := len(arr) / 2
    left := mergeSort(arr[:mid])
    right := mergeSort(arr[mid:])
    return merge(left, right)
}

func merge(left, right []int) []int {
    result := make([]int, 0, len(left)+len(right))
    i, j := 0, 0
    for i < len(left) && j < len(right) {
        if left[i] <= right[j] {
            result = append(result, left[i])
            i++
        } else {
            result = append(result, right[j])
            j++
        }
    }
    result = append(result, left[i:]...)
    result = append(result, right[j:]...)
    return result
}
```

### Quick Sort — O(n log n) avg, O(n²) worst

```go
func quickSort(arr []int, low, high int) {
    if low < high {
        pivot := partition(arr, low, high)
        quickSort(arr, low, pivot-1)
        quickSort(arr, pivot+1, high)
    }
}

func partition(arr []int, low, high int) int {
    pivot := arr[high]
    i := low - 1
    for j := low; j < high; j++ {
        if arr[j] <= pivot {
            i++
            arr[i], arr[j] = arr[j], arr[i]
        }
    }
    arr[i+1], arr[high] = arr[high], arr[i+1]
    return i + 1
}
```

### Heap Sort — O(n log n), in-place

```go
func heapSort(arr []int) {
    n := len(arr)
    // Строим max-heap
    for i := n/2 - 1; i >= 0; i-- {
        heapify(arr, n, i)
    }
    // Извлекаем элементы
    for i := n - 1; i > 0; i-- {
        arr[0], arr[i] = arr[i], arr[0]
        heapify(arr, i, 0)
    }
}

func heapify(arr []int, n, i int) {
    largest, l, r := i, 2*i+1, 2*i+2
    if l < n && arr[l] > arr[largest] { largest = l }
    if r < n && arr[r] > arr[largest] { largest = r }
    if largest != i {
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
    }
}
```

### Сравнение алгоритмов сортировки

| Алгоритм | Лучший | Средний | Худший | Память | Стабильная |
|---------|--------|---------|--------|--------|-----------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Да |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | **Да** |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | Нет |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | Нет |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Да |
| Go `slices.Sort` | — | O(n log n) | O(n log n) | O(log n) | Нет (pdqsort) |

**Задачи:**
- [912. Sort an Array](https://leetcode.com/problems/sort-an-array/)
- [215. Kth Largest Element](https://leetcode.com/problems/kth-largest-element-in-an-array/) ⭐

---

## Деревья (Trees)

### Бинарное дерево поиска (BST)

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}
```

**Свойство BST**: `left.Val < root.Val < right.Val` (рекурсивно).

| Операция | Средний | Худший (несбалансированное) |
|----------|---------|----------------------------|
| Поиск | O(log n) | O(n) |
| Вставка | O(log n) | O(n) |
| Удаление | O(log n) | O(n) |

### Обходы дерева (Tree Traversals)

```go
// Preorder: root → left → right
func preorder(root *TreeNode) []int {
    if root == nil { return nil }
    res := []int{root.Val}
    res = append(res, preorder(root.Left)...)
    res = append(res, preorder(root.Right)...)
    return res
}

// Inorder: left → root → right (даёт отсортированный порядок для BST!)
func inorder(root *TreeNode) []int {
    if root == nil { return nil }
    res := inorder(root.Left)
    res = append(res, root.Val)
    res = append(res, inorder(root.Right)...)
    return res
}

// Postorder: left → right → root
func postorder(root *TreeNode) []int {
    if root == nil { return nil }
    res := postorder(root.Left)
    res = append(res, postorder(root.Right)...)
    res = append(res, root.Val)
    return res
}

// BFS (Level-order) — обход по уровням
func levelOrder(root *TreeNode) [][]int {
    if root == nil { return nil }
    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        levelSize := len(queue)
        level := make([]int, levelSize)
        for i := 0; i < levelSize; i++ {
            node := queue[i]
            level[i] = node.Val
            if node.Left != nil  { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        queue = queue[levelSize:]
        result = append(result, level)
    }
    return result
}
```

### Итеративный DFS (со стеком)

```go
func inorderIterative(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    curr := root

    for curr != nil || len(stack) > 0 {
        for curr != nil {
            stack = append(stack, curr)
            curr = curr.Left
        }
        curr = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, curr.Val)
        curr = curr.Right
    }
    return result
}
```

### Задачи по деревьям

- [104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) ⭐
- [226. Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/) ⭐
- [100. Same Tree](https://leetcode.com/problems/same-tree/) ⭐
- [572. Subtree of Another Tree](https://leetcode.com/problems/subtree-of-another-tree/)
- [235. Lowest Common Ancestor of BST](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/) ⭐
- [102. Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) ⭐
- [543. Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) ⭐
- [124. Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/) 🔥
- [297. Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/) 🔥

### Сбалансированные деревья

**AVL Tree** — самобалансирующееся BST:
- `|height(left) - height(right)| <= 1` для каждого узла.
- Операции: O(log n) гарантировано.
- Дороже вставки/удаления (нужны ротации).

**Red-Black Tree** — другое самобалансирующееся BST:
- Менее строго сбалансировано, чем AVL.
- Быстрее вставки/удаления, чуть медленнее поиска.
- Используется в: `std::map` (C++), Java `TreeMap`, `database/sql` connection pools.

### Heap (Куча) — Priority Queue

**Min-Heap**: родитель ≤ детей. Корень — минимальный элемент.
**Max-Heap**: родитель ≥ детей. Корень — максимальный элемент.

```go
import "container/heap"

// Min-heap в Go
type MinHeap []int

func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

h := &MinHeap{3, 1, 4, 1, 5}
heap.Init(h)
heap.Push(h, 2)
min := heap.Pop(h).(int) // 1
```

| Операция | Сложность |
|----------|-----------|
| Push | O(log n) |
| Pop (min/max) | O(log n) |
| Peek (min/max) | O(1) |
| Build heap | O(n) |

**Задачи:**
- [703. Kth Largest in Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/) ⭐
- [1046. Last Stone Weight](https://leetcode.com/problems/last-stone-weight/)
- [973. K Closest Points](https://leetcode.com/problems/k-closest-points-to-origin/)
- [295. Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) 🔥

### Trie (Префиксное дерево)

Структура для хранения и поиска строк по префиксу:

```go
type TrieNode struct {
    children [26]*TrieNode
    isEnd    bool
}

type Trie struct {
    root *TrieNode
}

func (t *Trie) Insert(word string) {
    node := t.root
    for _, ch := range word {
        idx := ch - 'a'
        if node.children[idx] == nil {
            node.children[idx] = &TrieNode{}
        }
        node = node.children[idx]
    }
    node.isEnd = true
}

func (t *Trie) Search(word string) bool {
    node := t.root
    for _, ch := range word {
        idx := ch - 'a'
        if node.children[idx] == nil { return false }
        node = node.children[idx]
    }
    return node.isEnd
}
```

| Операция | Сложность |
|----------|-----------|
| Insert | O(m) — m = длина слова |
| Search | O(m) |
| StartsWith | O(m) |

**Задачи:**
- [208. Implement Trie](https://leetcode.com/problems/implement-trie-prefix-tree/) ⭐
- [211. Design Add and Search Words](https://leetcode.com/problems/design-add-and-search-words-data-structure/)
- [212. Word Search II](https://leetcode.com/problems/word-search-ii/) 🔥

---

## Графы (Graphs)

### Представление графа

```go
// Список смежности (adjacency list) — чаще используется
graph := map[int][]int{
    0: {1, 2},
    1: {0, 3},
    2: {0, 4},
    3: {1},
    4: {2},
}

// Матрица смежности — для плотных графов
adj := make([][]int, n)
for i := range adj {
    adj[i] = make([]int, n)
}
adj[0][1] = 1 // есть ребро 0→1
```

### BFS на графе — O(V + E)

```go
func bfs(graph map[int][]int, start int) []int {
    visited := make(map[int]bool)
    queue := []int{start}
    visited[start] = true
    result := []int{}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        result = append(result, node)

        for _, neighbor := range graph[node] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, neighbor)
            }
        }
    }
    return result
}
```

### DFS на графе — O(V + E)

```go
func dfs(graph map[int][]int, node int, visited map[int]bool, result *[]int) {
    visited[node] = true
    *result = append(*result, node)

    for _, neighbor := range graph[node] {
        if !visited[neighbor] {
            dfs(graph, neighbor, visited, result)
        }
    }
}
```

### Задачи BFS/DFS на графах

- [200. Number of Islands](https://leetcode.com/problems/number-of-islands/) ⭐ (DFS/BFS на сетке)
- [133. Clone Graph](https://leetcode.com/problems/clone-graph/) ⭐
- [695. Max Area of Island](https://leetcode.com/problems/max-area-of-island/)
- [130. Surrounded Regions](https://leetcode.com/problems/surrounded-regions/)
- [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) ⭐ (Multi-source BFS)
- [417. Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow/)
- [127. Word Ladder](https://leetcode.com/problems/word-ladder/) 🔥

### Алгоритм Дейкстры — кратчайший путь O((V+E) log V)

Для взвешенных графов с **неотрицательными** весами:

```go
import "container/heap"

type Item struct {
    node, cost int
}
type PQ []Item
func (pq PQ) Len() int            { return len(pq) }
func (pq PQ) Less(i, j int) bool  { return pq[i].cost < pq[j].cost }
func (pq PQ) Swap(i, j int)       { pq[i], pq[j] = pq[j], pq[i] }
func (pq *PQ) Push(x any)         { *pq = append(*pq, x.(Item)) }
func (pq *PQ) Pop() any {
    old := *pq; n := len(old); x := old[n-1]; *pq = old[:n-1]; return x
}

func dijkstra(graph map[int][][2]int, start, end int) int {
    dist := make(map[int]int)
    pq := &PQ{{start, 0}}
    heap.Init(pq)

    for pq.Len() > 0 {
        curr := heap.Pop(pq).(Item)
        node, cost := curr.node, curr.cost

        if node == end { return cost }
        if d, ok := dist[node]; ok && cost >= d { continue }
        dist[node] = cost

        for _, edge := range graph[node] {
            neighbor, weight := edge[0], edge[1]
            heap.Push(pq, Item{neighbor, cost + weight})
        }
    }
    return -1
}
```

### Алгоритм Беллмана-Форда — O(V * E)

Работает с **отрицательными** весами. Определяет отрицательные циклы.

```go
func bellmanFord(edges [][3]int, n, src int) []int {
    dist := make([]int, n)
    for i := range dist { dist[i] = 1<<31 - 1 }
    dist[src] = 0

    for i := 0; i < n-1; i++ {
        for _, e := range edges { // [u, v, weight]
            u, v, w := e[0], e[1], e[2]
            if dist[u] != 1<<31-1 && dist[u]+w < dist[v] {
                dist[v] = dist[u] + w
            }
        }
    }
    return dist
}
```

### Топологическая сортировка (DAG) — O(V + E)

Для ориентированных ациклических графов (DAG):

```go
// Алгоритм Кана (через in-degree)
func topoSort(n int, edges [][2]int) []int {
    inDegree := make([]int, n)
    graph := make([][]int, n)

    for _, e := range edges {
        graph[e[0]] = append(graph[e[0]], e[1])
        inDegree[e[1]]++
    }

    queue := []int{}
    for i, d := range inDegree {
        if d == 0 { queue = append(queue, i) }
    }

    result := []int{}
    for len(queue) > 0 {
        node := queue[0]; queue = queue[1:]
        result = append(result, node)
        for _, neighbor := range graph[node] {
            inDegree[neighbor]--
            if inDegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }
    if len(result) != n { return nil } // есть цикл
    return result
}
```

### Union-Find (Disjoint Set Union — DSU)

Для задач связности компонент:

```go
type DSU struct {
    parent, rank []int
}

func NewDSU(n int) *DSU {
    d := &DSU{parent: make([]int, n), rank: make([]int, n)}
    for i := range d.parent { d.parent[i] = i }
    return d
}

func (d *DSU) Find(x int) int {
    if d.parent[x] != x {
        d.parent[x] = d.Find(d.parent[x]) // path compression
    }
    return d.parent[x]
}

func (d *DSU) Union(x, y int) bool {
    px, py := d.Find(x), d.Find(y)
    if px == py { return false } // уже в одном множестве
    // union by rank
    if d.rank[px] < d.rank[py] { px, py = py, px }
    d.parent[py] = px
    if d.rank[px] == d.rank[py] { d.rank[px]++ }
    return true
}
```

**Задачи по графам:**
- [207. Course Schedule](https://leetcode.com/problems/course-schedule/) ⭐ (Топологическая сортировка)
- [210. Course Schedule II](https://leetcode.com/problems/course-schedule-ii/)
- [323. Number of Connected Components](https://leetcode.com/problems/number-of-connected-components-in-an-undirected-graph/)
- [684. Redundant Connection](https://leetcode.com/problems/redundant-connection/) (DSU)
- [743. Network Delay Time](https://leetcode.com/problems/network-delay-time/) ⭐ (Дейкстра)
- [787. Cheapest Flights](https://leetcode.com/problems/cheapest-flights-within-k-stops/) 🔥
- [269. Alien Dictionary](https://leetcode.com/problems/alien-dictionary/) 🔥

---

## Паттерны решения задач (Patterns)

### Two Pointers — O(n)

Два указателя движутся по массиву/строке:

```go
// Проверка палиндрома
func isPalindrome(s string) bool {
    l, r := 0, len(s)-1
    for l < r {
        if s[l] != s[r] { return false }
        l++; r--
    }
    return true
}

// Задачи
// 125. Valid Palindrome
// 167. Two Sum II (sorted)    ⭐
// 15. 3Sum                    ⭐
// 11. Container With Most Water ⭐
```

### Sliding Window — O(n)

Окно фиксированного или переменного размера:

```go
// Максимальная сумма подмассива длины k
func maxSumSubarray(arr []int, k int) int {
    sum, maxSum := 0, 0
    for i := 0; i < k; i++ { sum += arr[i] }
    maxSum = sum
    for i := k; i < len(arr); i++ {
        sum += arr[i] - arr[i-k]
        if sum > maxSum { maxSum = sum }
    }
    return maxSum
}

// Задачи:
// 121. Best Time to Buy Stock       ⭐
// 3. Longest Substring Without Repeating ⭐
// 424. Longest Repeating Char Replacement
// 76. Minimum Window Substring     🔥
```

### Fast & Slow Pointers (Floyd's Cycle Detection)

```go
// Определение цикла в Linked List
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast { return true }
    }
    return false
}

// Задачи:
// 141. Linked List Cycle             ⭐
// 142. Linked List Cycle II
// 287. Find the Duplicate Number    🔥
```

---

## Динамическое программирование (Dynamic Programming)

DP = разбиение задачи на подзадачи + кэширование решений.

**Признаки DP задачи:**
- Оптимальное значение (min/max/count).
- Повторяющиеся подзадачи.
- Оптимальная подструктура.

### 1D DP — Fibonacci / Climbing Stairs

```go
// Без мемоизации: O(2ⁿ)
// С мемоизацией: O(n) время, O(n) память
// Tabulation (bottom-up): O(n) время, O(1) память

func climbStairs(n int) int {
    if n <= 2 { return n }
    a, b := 1, 2
    for i := 3; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
```

**Задачи 1D DP:**
- [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) ⭐
- [198. House Robber](https://leetcode.com/problems/house-robber/) ⭐
- [322. Coin Change](https://leetcode.com/problems/coin-change/) ⭐
- [300. Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) ⭐
- [139. Word Break](https://leetcode.com/problems/word-break/)

### 2D DP — Unique Paths / Knapsack

```go
// Уникальные пути в сетке m×n
func uniquePaths(m, n int) int {
    dp := make([][]int, m)
    for i := range dp { dp[i] = make([]int, n) }
    for i := 0; i < m; i++ { dp[i][0] = 1 }
    for j := 0; j < n; j++ { dp[0][j] = 1 }
    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            dp[i][j] = dp[i-1][j] + dp[i][j-1]
        }
    }
    return dp[m-1][n-1]
}
```

**Задачи 2D DP:**
- [62. Unique Paths](https://leetcode.com/problems/unique-paths/)
- [1143. Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) ⭐
- [72. Edit Distance](https://leetcode.com/problems/edit-distance/) 🔥
- [309. Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)
- [518. Coin Change II](https://leetcode.com/problems/coin-change-ii/)

### Задачи на подстроки/подмассивы:
- [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/) ⭐
- [647. Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings/)

---

## Алгоритм Хаффмана (Huffman Coding)

Алгоритм **сжатия данных без потерь** (lossless compression). Автор: Дэвид Хаффман, 1952.

**Идея**: частые символы кодируются короткими битовыми кодами, редкие — длинными.

### Пример

```
Текст: "ABRACADABRA"
Частоты: A=5, B=2, R=2, C=1, D=1

Шаг 1: создаём узлы, сортируем по частоте
[C:1] [D:1] [B:2] [R:2] [A:5]

Шаг 2: объединяем два наименьших → новый узел
[CD:2] [B:2] [R:2] [A:5]

Шаг 3: снова объединяем наименьшие
[B+CD:4] [R:2] [A:5]
→ [B+R+CD:6] [A:5]
→ [A+B+R+CD:11]

Дерево:
       [11]
      /    \
   [A:5]  [6]
          /  \
       [B:2] [4]
             /  \
          [R:2] [CD:2]
                /   \
            [C:1] [D:1]

Коды:
A = 0         (1 бит, самый частый!)
B = 10        (2 бита)
R = 110       (3 бита)
C = 1110      (4 бита)
D = 1111      (4 бита)
```

**Без сжатия** (ASCII): 11 символов × 8 бит = 88 бит.
**С Хаффманом**: 5×1 + 2×2 + 2×3 + 1×4 + 1×4 = 5+4+6+4+4 = **23 бита**.

### Реализация на Go

```go
import "container/heap"

type HuffNode struct {
    char  rune
    freq  int
    left  *HuffNode
    right *HuffNode
}

type NodeHeap []*HuffNode

func (h NodeHeap) Len() int           { return len(h) }
func (h NodeHeap) Less(i, j int) bool { return h[i].freq < h[j].freq }
func (h NodeHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *NodeHeap) Push(x any)        { *h = append(*h, x.(*HuffNode)) }
func (h *NodeHeap) Pop() any {
    old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func BuildHuffmanTree(freq map[rune]int) *HuffNode {
    h := &NodeHeap{}
    for ch, f := range freq {
        heap.Push(h, &HuffNode{char: ch, freq: f})
    }

    for h.Len() > 1 {
        left := heap.Pop(h).(*HuffNode)
        right := heap.Pop(h).(*HuffNode)
        parent := &HuffNode{
            freq:  left.freq + right.freq,
            left:  left,
            right: right,
        }
        heap.Push(h, parent)
    }
    return heap.Pop(h).(*HuffNode)
}

// Генерация кодов (рекурсивный обход)
func GenerateCodes(node *HuffNode, prefix string, codes map[rune]string) {
    if node == nil { return }
    if node.left == nil && node.right == nil {
        codes[node.char] = prefix // листовой узел
        return
    }
    GenerateCodes(node.left, prefix+"0", codes)
    GenerateCodes(node.right, prefix+"1", codes)
}
```

### Применение Хаффмана

- **gzip, zlib** — используют Хаффман как часть алгоритма Deflate.
- **JPEG** — сжатие коэффициентов DCT.
- **MP3** — Хаффман для части данных.
- **HPACK (HTTP/2)** — сжатие заголовков.
- **ZIP** — часть алгоритма сжатия.

**Свойства:**
- Сложность построения дерева: O(n log n).
- Оптимально для посимвольного кодирования (prefix-free codes).
- Не оптимально для блоков символов (тогда — арифметическое кодирование).

---

## Рекурсия и Backtracking

### Backtracking (перебор с возвратом)

```go
// Все перестановки массива
func permute(nums []int) [][]int {
    result := [][]int{}
    var backtrack func(start int)
    backtrack = func(start int) {
        if start == len(nums) {
            perm := make([]int, len(nums))
            copy(perm, nums)
            result = append(result, perm)
            return
        }
        for i := start; i < len(nums); i++ {
            nums[start], nums[i] = nums[i], nums[start]
            backtrack(start + 1)
            nums[start], nums[i] = nums[i], nums[start] // отмена
        }
    }
    backtrack(0)
    return result
}
```

**Задачи:**
- [78. Subsets](https://leetcode.com/problems/subsets/) ⭐
- [39. Combination Sum](https://leetcode.com/problems/combination-sum/) ⭐
- [46. Permutations](https://leetcode.com/problems/permutations/) ⭐
- [79. Word Search](https://leetcode.com/problems/word-search/) ⭐
- [51. N-Queens](https://leetcode.com/problems/n-queens/) 🔥

---

## Топ LeetCode задач по темам (с ссылками)

### Arrays & Hashing
| # | Задача | Сложность |
|---|--------|----------|
| 217 | [Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) | Easy |
| 242 | [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Easy |
| 1 | [Two Sum](https://leetcode.com/problems/two-sum/) | Easy |
| 49 | [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Medium |
| 347 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Medium |
| 128 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | Medium |

### Two Pointers
| # | Задача | Сложность |
|---|--------|----------|
| 125 | [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Easy |
| 167 | [Two Sum II](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) | Medium |
| 15 | [3Sum](https://leetcode.com/problems/3sum/) | Medium |
| 11 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Medium |
| 42 | [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/) | Hard 🔥 |

### Sliding Window
| # | Задача | Сложность |
|---|--------|----------|
| 121 | [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) | Easy |
| 3 | [Longest Substring Without Repeating](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | Medium |
| 424 | [Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/) | Medium |
| 76 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) | Hard 🔥 |

### Binary Search
| # | Задача | Сложность |
|---|--------|----------|
| 704 | [Binary Search](https://leetcode.com/problems/binary-search/) | Easy |
| 74 | [Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix/) | Medium |
| 33 | [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) | Medium |
| 153 | [Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) | Medium |

### Trees
| # | Задача | Сложность |
|---|--------|----------|
| 226 | [Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/) | Easy |
| 104 | [Maximum Depth](https://leetcode.com/problems/maximum-depth-of-binary-tree/) | Easy |
| 543 | [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) | Easy |
| 110 | [Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree/) | Easy |
| 235 | [LCA of BST](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/) | Medium |
| 102 | [Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | Medium |
| 98 | [Validate BST](https://leetcode.com/problems/validate-binary-search-tree/) | Medium |
| 124 | [Max Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/) | Hard 🔥 |

### Graphs
| # | Задача | Сложность |
|---|--------|----------|
| 200 | [Number of Islands](https://leetcode.com/problems/number-of-islands/) | Medium |
| 133 | [Clone Graph](https://leetcode.com/problems/clone-graph/) | Medium |
| 994 | [Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) | Medium |
| 207 | [Course Schedule](https://leetcode.com/problems/course-schedule/) | Medium |
| 743 | [Network Delay Time](https://leetcode.com/problems/network-delay-time/) | Medium |
| 127 | [Word Ladder](https://leetcode.com/problems/word-ladder/) | Hard 🔥 |

### Dynamic Programming
| # | Задача | Сложность |
|---|--------|----------|
| 70 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | Easy |
| 198 | [House Robber](https://leetcode.com/problems/house-robber/) | Medium |
| 322 | [Coin Change](https://leetcode.com/problems/coin-change/) | Medium |
| 300 | [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) | Medium |
| 1143 | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) | Medium |
| 72 | [Edit Distance](https://leetcode.com/problems/edit-distance/) | Medium 🔥 |

---

## Ресурсы для подготовки

| Ресурс | Описание |
|--------|---------|
| [NeetCode 150](https://neetcode.io/practice) | 150 задач с разбором на видео по паттернам |
| [LeetCode Blind 75](https://leetcode.com/discuss/general-discussion/460599/blind-75-leetcode-questions) | 75 самых важных задач |
| [LeetCode Top Interview 150](https://leetcode.com/studyplan/top-interview-150/) | Официальный план от LeetCode |
| [Grind 169](https://www.techinterviewhandbook.org/grind75) | Расширенный список (169 задач) |
| [Visualgo](https://visualgo.net/) | Визуализация алгоритмов |
| [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) | Шпаргалка по сложностям |
| [CP Algorithms](https://cp-algorithms.com/) | Теория + реализации алгоритмов |
