---
description: 希望小读者能够喜欢，我会慢慢维护起来的！
---

# 深入剖析 LRU 缓存机制实现

C语言实现代码：

```c
#include <stdio.h>
#include <stdlib.h>

// 定义一个结点 (用来实现一个双向链表)
typedef struct Node {
    struct Node *prev, *next;   // 前继指针，后继指针
    unsigned pageNumber; // 页面编号，也就是 key 值
} Node;

// 双向链表队列（为什么叫队列，可以看看文章中求错误页数目的类队列法）。
typedef struct DoubleLinkedQueue {
    unsigned count; // 双向链表中当前的页面数
    unsigned numberOfPages; // 双向链表中最多可以存储的页面数
    Node *head, *tail;     // 双向链表的头部和尾部结点
} DoubleLinkedQueue;

// 哈希表 (一个结点集合)
typedef struct Hash {
    int capacity; // 哈希表的容量
    Node** array; // 结点数组
} Hash;

// 创建一个新结点，并为新节点赋值
Node* newNode(unsigned pageNumber)
{
    // 分配内存空间并为页面编号赋值
    Node* temp = (Node*)malloc(sizeof(Node));
    temp->pageNumber = pageNumber;

    // 初始化结点的前继指针和后继指针
    temp->prev = temp->next = NULL;

    return temp;
}

// 创建一个空的双向链表
DoubleLinkedQueue* createDoubleLinkedQueue(int numberOfPages)
{
    DoubleLinkedQueue* queue = (DoubleLinkedQueue*)malloc(sizeof(DoubleLinkedQueue));

    // 双向链表为空
    queue->count = 0;
    queue->head = queue->tail = NULL;

    // 队列中最多可存储 
    queue->numberOfPages = numberOfPages;

    return queue;
}

// 创建一个容量为 capacity 的哈希表
Hash* createHash(int capacity)
{
    // 分配内存
    Hash* hash = (Hash*)malloc(sizeof(Hash));
    hash->capacity = capacity;

    // 创建一个指针数组以引用双向链表的节点
    hash->array = (Node**)malloc(hash->capacity * sizeof(Node*));

    // 将哈希表的所有 value 初始化为空
    int i;
    for (i = 0; i < hash->capacity; ++i)
        hash->array[i] = NULL;

    return hash;
}

// 检查LRU 缓存的容量是否达到了上限 内存中是否有可用插槽的功能
int isFullofLRUCache(DoubleLinkedQueue* queue)
{
    return queue->count == queue->numberOfPages;
}

// 检查双向链表是否为空
int isQueueEmpty(DoubleLinkedQueue* queue)
{
    return queue->rear == NULL;
}

// 删除双向链表中的一个结点
void deleteNode(DoubleLinkedQueue* queue)
{
    if (isQueueEmpty(queue))
        return;

    // 如果列表中仅有一个结点，则将双链表的头部结点置空
    if (queue->front == queue->rear)
        queue->front = NULL;

    // 更改队尾指针，删除队尾指针的前一个元素，也就是最久未被访问的页面
    Node* temp = queue->tail;
    queue->tail = queue->tail->prev;

    if (queue->tail)
        queue->tail->next = NULL;

    free(temp);

    // 当前双链表中元素的数目减1
    queue->count--;
}

// 在哈希表和双向链表中添加一个页面编号为 pageNumber 的结点
void Enqueue(DoubleLinkedQueue* queue, Hash* hash, unsigned pageNumber)
{
    // 如果双向链表已经满了，则删除伪队尾指针的前一个结点
    if (isFullofLRUCache(queue)) {
        // 从哈希表中删除编号为 pageNumber 的结点
        hash->array[queue->tail->pageNumber] = NULL;
        deleteNode(queue);
    }

    // 新建一个 key 为 pageNumber 的结点
    // 并且将新节点添加到队列的头部
    Node* temp = newNode(pageNumber);
    temp->next = queue->head;

    // 如果双向链表为空，改变双向链表的头指针和尾指针
    if (isQueueEmpty(queue))
        queue->tail = queue->head = temp;
    else // 双向链表不为空
    {
        queue->haed->prev = temp;
        queue->head = temp;
    }

    // 在哈希表中添加新节点
    hash->array[pageNumber] = temp;

    //  双向链表当前的容量加 1
    queue->count++;
}

// 请求页是否存在引用（是否在内存当中）:
// 1. 页面不在LRU缓存中, 将页面加载进内存并添加到双向链表队列的头部
// 2. 页面在LRU缓存中, 将页面移动到双向链表队列的头部
void ReferencePage(DoubleLinkedQueue* queue, Hash* hash, unsigned pageNumber)
{
    Node* reqPage = hash->array[pageNumber];

    // 页面不在LRU缓存中，将页面加载进内存
    if (reqPage == NULL)
        Enqueue(queue, hash, pageNumber);

    // 页面在LRU缓存中, 将页面移动到双向链表队列的头部
    else if (reqPage != queue->head) {
        // Unlink rquested page from its current location
        // in queue.
        reqPage->prev->next = reqPage->next;
        if (reqPage->next)
            reqPage->next->prev = reqPage->prev;

        // 如果请求页在队尾, 则修改队尾指针，并将结点移动到对头
        if (reqPage == queue->tail) {
            queue->tail = reqPage->prev;
            queue->tail->next = NULL;
        }

        // 将请求页移动到队列的头部
        reqPage->next = queue->head;
        reqPage->prev = NULL;

        // 修改当前头指针的前继结点
        reqPage->next->prev = reqPage;

        // 将请求页作为对头元素
        queue->head = reqPage;
    }
}

int main()
{
    DoubleLinkedQueue* q = createDoubleLinkedQueue(2);

    // 请求页面序号在 0 to 5 之间
    Hash* hash = createHash(6);

    // 请求页面序列 1, 2, 1, 3, 2, 4, 1, 3, 4
    ReferencePage(q, hash, 1);
    ReferencePage(q, hash, 2);
    ReferencePage(q, hash, 1);
    ReferencePage(q, hash, 3);
    ReferencePage(q, hash, 2);
    ReferencePage(q, hash, 4);
    ReferencePage(q, hash, 1);
    ReferencePage(q, hash, 3);
    ReferencePage(q, hash, 4);

    // 测试输出
    printf("%d ", q->head->pageNumber);
    printf("%d ", q->head->next->pageNumber);

    return 0;
}
```

