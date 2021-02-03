# 队列

### 优先队列

```
自定义比较器
struct cmp{
    bool operator()(const ListNode* x,const ListNode* y)
    {
    	return x->val > y->val;
    }
}; 
priority_queue<ListNode *, vector<ListNode*>, cmp> p;
```

