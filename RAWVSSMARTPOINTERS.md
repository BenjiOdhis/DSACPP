# Linked Lists in C++: Raw Pointers vs. Smart Pointers (`unique_ptr`)

A quick, practical reference comparing a singly linked list implemented with **manual memory** (`new`/`delete`) and with **modern C++ smart pointers** (`std::unique_ptr`). Use this to understand the trade-offs and to pick the right style for your project.

---

## TL;DR

| Topic       | Raw Pointers (`new`/`delete`)            | Smart Pointers (`std::unique_ptr`)        |
| ----------- | ---------------------------------------- | ----------------------------------------- |
| Ownership   | Implicit, **you** must track who deletes | Explicit, owned by the `unique_ptr` chain |
| Cleanup     | Manual `delete` everywhere               | Automatic (RAII) — no `delete` calls      |
| Safety      | Easy to leak/ double-free/ dangle        | Prevents leaks & most lifetime bugs       |
| Complexity  | Simple syntax; more discipline           | Slightly more syntax (`move`, `.get()`)   |
| When to use | Learning/low-level control               | **Default choice** in modern C++          |

---

## Singly Linked List — Raw Pointers Version

```cpp
// raw_list.cpp
#include <iostream>
using namespace std;

struct Node {
    int data;
    Node* next;                 // raw pointer to next node
};

Node* createNode(int value) {   // returns a heap-allocated node
    Node* n = new Node{value, nullptr};
    return n;                   // caller must eventually delete it
}

void printList(const Node* head) {
    const Node* cur = head;
    while (cur != nullptr) {
        cout << cur->data << " -> ";
        cur = cur->next;
    }
    cout << "null\n";
}

void freeList(Node* head) {     // walk the list and delete every node
    while (head) {
        Node* nxt = head->next;
        delete head;            // manual cleanup
        head = nxt;
    }
}

int main() {
    Node* node1 = createNode(3);
    Node* node2 = createNode(5);
    Node* node3 = createNode(13);
    Node* node4 = createNode(2);

    node1->next = node2;        // link: 3 -> 5 -> 13 -> 2
    node2->next = node3;
    node3->next = node4;

    printList(node1);

    freeList(node1);            // IMPORTANT: avoid leaks
    return 0;
}
```

**Compile & run**

```bash
g++ -std=c++17 -O2 raw_list.cpp -o raw_list && ./raw_list
```

**Pros**

* Minimal syntax; mirrors C semantics.
* Great for learning memory lifetimes.

**Cons**

* Easy to forget a `delete` (leak).
* Easy to double-free or dangle pointers after deletion.
* Harder to review/maintain in larger codebases.

---

## Singly Linked List — Smart Pointers (`std::unique_ptr`) Version

```cpp
// unique_list.cpp
#include <iostream>
#include <memory>
using namespace std;

struct Node {
    int data;
    unique_ptr<Node> next;            // ownership of the next node
    explicit Node(int value) : data(value), next(nullptr) {}
};

unique_ptr<Node> createNode(int value) {
    return make_unique<Node>(value);  // allocates & returns owning pointer
}

void printList(const Node* head) {    // read-only traversal with raw pointer
    const Node* cur = head;
    while (cur != nullptr) {
        cout << cur->data << " -> ";
        cur = cur->next.get();        // .get() returns raw pointer (no ownership)
    }
    cout << "null\n";
}

int main() {
    auto node1 = createNode(3);
    auto node2 = createNode(5);
    auto node3 = createNode(13);
    auto node4 = createNode(2);

    // Transfer ownership down the chain (unique_ptr is move-only)
    node1->next = move(node2);
    node1->next->next = move(node3);
    node1->next->next->next = move(node4);

    printList(node1.get());           // pass raw pointer for viewing

    // No manual delete: when node1 goes out of scope, it frees the entire chain.
    return 0;
}
```

**Compile & run**

```bash
g++ -std=c++17 -O2 unique_list.cpp -o unique_list && ./unique_list
```

**Pros**

* Automatic cleanup (RAII) — **no leaks** if you follow ownership.
* Clear intent: the list “owns” its nodes.
* Safer refactors and reviews.

**Cons**

* Must understand `move` semantics (`std::move` to transfer ownership).
* Need `.get()` when you only want to “look” at the pointer without owning it.

---

## Visualizing Ownership

**Raw pointers (manual):**

```
node1*  -> [3 | *] --> [5 | *] --> [13 | *] --> [2 | null]
(no automatic cleanup; you must free each node)
```

**unique_ptr chain (automatic):**

```
node1   -> [3 | unique_ptr] ==> [5 | unique_ptr] ==> [13 | unique_ptr] ==> [2 | null]
 (owns)           (owns)                 (owns)
Destroying node1 destroys the whole chain.
```

---

## Typical Operations (Side-by-Side)

### Insert at front

**Raw**

```cpp
void push_front(Node*& head, int v) { // reference to pointer so we can reassign it
    Node* n = new Node{v, nullptr};
    n->next = head;
    head = n;                          // new head
}
```

**unique_ptr**

```cpp
void push_front(unique_ptr<Node>& head, int v) {
    auto n = make_unique<Node>(v);
    n->next = move(head);              // new node takes ownership of old head
    head = move(n);                    // head now owns the new node
}
```

### Clear list

**Raw**

```cpp
freeList(head); // walk & delete (as shown above)
head = nullptr;
```

**unique_ptr**

```cpp
head.reset();   // instantly releases the whole chain via destructor cascade
```

---

## Common Pitfalls & How Smart Pointers Help

| Pitfall                    | Raw Pointers                 | `unique_ptr` Behavior                                                                              |
| -------------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------- |
| **Leak** (forgot `delete`) | Memory not reclaimed         | Impossible if last owner goes out of scope                                                         |
| **Double free**            | Crashes/UB if `delete` twice | Each object has exactly one owner                                                                  |
| **Dangling pointer**       | Use-after-free bugs          | Non-owning raw ptrs still possible, but ownership is explicit; avoid keeping long-lived non-owners |
| **Exception safety**       | Must `delete` in all paths   | Automatic cleanup on scope exit                                                                    |

---

## When to Learn/Use What

* **Learning path (recommended):**

  1. Practice with **raw pointers** (`new`/`delete`) to grasp heap, leaks, dangling, ownership.
  2. Move to **`std::unique_ptr`** (preferred in modern C++) for real projects.
  3. Learn **`std::shared_ptr`/`std::weak_ptr`** later for *shared* ownership cases.

* **Production guidance:** Prefer **smart pointers** (and RAII) by default. Use raw pointers for:

  * Non-owning observers (function params like `const Node*`),
  * Interfacing with legacy/low-level APIs,
  * Performance-critical paths where ownership is managed elsewhere.

---

## Quick Checklist

* [ ] Do I clearly know **who owns** each node?
* [ ] Could this function **throw**? (Smart pointers make cleanup automatic.)
* [ ] Do I need **nullable** and **movable** ownership? (`unique_ptr` fits.)
* [ ] Am I passing a pointer just to **read**? Use a **raw pointer/reference** param (`const Node*` / `const Node&`).
* [ ] Do I need **shared** ownership? Consider `shared_ptr` + **avoid cycles** (use `weak_ptr`).

---

## License

Feel free to copy this into your repo. Consider adding a short note like:

```
This reference is provided “as is”. Use at your own risk.
© <your-name>, MIT license (or your preferred license).
```
