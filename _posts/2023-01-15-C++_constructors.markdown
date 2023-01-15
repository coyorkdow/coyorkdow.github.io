---
layout: post
title: "A Quick Note of Copy and Move Control in C++"
date: 2023-01-15 +0800
categories: C++
---

This note is about the "rule of five", about the compiler behaviors on copy & move constructors & assignments.

---

- There are two different conceptions, **implicitly-declared** and **default**. They are different. Each copy/move constructor/assignment operator has an implicitly-declared edition and a default edition.

```C++
class Test {
 public:
  Test() = default;
};
```

We can directly use all of the four copy & move constructors & assignments on `Test`. What we use are **implicitly-declared**. But if we have `Test(const Test&) = default;` then we will invoke the **default** copy constructor instead of the implicitly-declared.

If we want use the default edition we should declare it as `default`.

- The compiler will always generate a copy constructor or assignment operator if there is no `user-declared` copy constructor or assignment operator. But sometimes the implicitly-declared copy constructor or assignment operator is declared as `delete`.
  <br /> For the class `T`
  - Copy Constructor
    1. If `T` has non-static data member that cannot be copied, or if `T` has directed or virtual base that cannot be copied.
    2. If `T` has non-static data member of rvalue reference type.
    3. If `T` has non-static data member whose destructor is unavailable, or if `T` has directed or virtual base whose destructor is unavailable. Unavailable means deleted or unaccessible.
    4. The scenario about the union or variant type.
    5. If `T` has `user-declared` move constructor or assignment operator.
  - Copy Assignment Operator
    1. If `T` has non-static data member that cannot be copy-assigned, or if `T` has directed or virtual base that cannot be copy-assigned. A const data member is non copy-assignable.
    2. If `T` has non-static data member of rvalue reference type.
    3. The scenario about the union or variant type.
    4. If `T` has `user-declared` move constructor or assignment operator.

- Declare copy constructor or move assignment operator does not affect each other. That is, to disallow copy we should declare both copy constructor and copy assignment operator as `delete`.

- A default constructor or assignment operator might still be available when its implicitly-declared edition is unavailable.

```C++
class Test {
 public:
  Test() = default;
  Test(Test&&) {} // or Test(Test&&) = default;, or Test(Test&&) = delete;
};
```

`Test` is now non copy-constructible and non copy-assignable. But if we have `Test(const Test&) = default;` then we can use the default copy constructor, even if the implicitly-declared is unavailable. And if we continue to have `Test& operator=(const Test&) = default;` the copy assignment on `Test` will be available too.

`Test(Test&&) = default;` or `Test(Test&&) = delete;` can also disallow the implicitly-declared copy.

- "Declared as default" and "declared as delete" are also treated as a kind of **user-declared**.

- Generally speaking. The behaviors of the user-declared may cause implicitly-declared edition unavailable, but it will not affect the default edition. Therefore, the conditions of default edition deleted are the conditions of implicitly-declared edition unavailable where the items about the user-declared behaviors are removed.

- The compiler won't always generate a move constructor or assignment operator even if there is no user-declared move constructor or assignment operator. Here are all these cases.
  <br /> For the class `T`
  - Move Constructor
    1. If `T` has user-declared copy constructor.
    2. If `T` has user-declared copy assignment operator.
    3. If `T` has user-declared move assignment operator.
    4. If `T` has user-declared destructor.
  - Move Assignment Operator
    1. If `T` has user-declared copy constructor.
    2. If `T` has user-declared copy assignment operator.
    3. If `T` has user-declared move constructor.
    4. If `T` has user-declared destructor.

It should be aware that in these cases, the complier doesn't declare it as `delete`, but doesn't generate the implicitly-declared at all. Which is different from the copy. And it is useful. For example.

```C++
class Test {
 public:
  Test() = default;
  ~Test() = default;
};
```

Since `Test` has user-declared destructor, there is no implicitly-declared move constructor. We can still have `Test t1; Test t2(std::move(t1));` because it will invoke the copy constructor. However if move constructor is deleted then it would cannot be compiled.

- The default move constructor is deleted in any of these conditions.
  <br /> For the class `T`
  1. If `T` has non-static data members that cannot be moved, or if `T` has direct or virtual base class that cannot be moved.
  2. If `T` has non-static data member whose destructor is unavailable, or if `T` has directed or virtual base whose destructor is unavailable. Unavailable means deleted or unaccessible.
  3. The scenario about the union or variant type.

- The default move assignment operator is deleted in any of these conditions.
  <br /> For the class `T`
  1. If `T` has non-static data members that cannot be moved, or if `T` has direct or virtual base class that cannot be move assigned. A const data member is non move-assignable.
  2. If `T` has non-static data member of reference type.

- A deleted default move constructor or a deleted move assignment operator is ignored by overload resolution. Therefore, when the move constructor or the move assignment operator is unavailable (there's no user-defined and there's no implicitly declared either. or user defines it as `default` while the default edition is deleted.), as long as the copy constructor or the assignment operator is available, trying invoke the move constructor or the move assignment operator will match the copy constructor or the copy move assignment operator, unless the move constructor or the move assignment operator is explicitly deleted.

```C++
class Unmovable {
 public:
  Unmovable() = default;
  Unmovable(const Unmovable&) = default;
  Unmovable& operator=(const Unmovable&) = default;
  Unmovable(Unmovable&&) = delete;
  Unmovable& operator=(Unmovable&&) = delete;
};

class Test {
 public:
  Test() = default;
  Test(const Test&) { std::cout << "copy constructor\n"; } 
  Test& operator=(const Test&) { std::cout << "copy assignment\n"; return *this; }
  Test(Test&&) = default; // the default move constructor is deleted.
  Test& operator=(Test&&) = default; // the default move assignment operator is deleted.
  Unmovable _;
};

int main() {
  Test t1;
  Test t2(std::move(t1)); // trying invoke the move constructor but actually invoke the copy.
  Test t3;
  t3 = std::move(t2); // trying invoke the move assignment operator but actually invoke the copy.
}
```

It will prints
```
copy constructor
copy assignment
```
