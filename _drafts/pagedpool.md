---
layout: post
title:  "Paged pools"
---

Testing Jekyll features, images, code etc.

### Efficiency

When developing real time visualization code, eliminate techniques with inherent performance pitfalls.

Too much:

- Allocation
  - Deallocation
- Cache trashing
- Indirection
  - Small functions not inlined
- Copying

{% highlight c++ linenos %}

struct Page
{
  std::vector<unsigned char> store;
};

template <typename T>
struct PagedPool
{
  T * allocate();
  T * deallocate();

private:
  void addPage();

  // Storage for all our active pages.
  vector<Page> pages;
  
  // Number of items per page.
  size_t pageSize = 1024;
  
  // Stores a link to our first free element.
  T * freeHead = nullptr;
};

{% endhighlight %}

When needed, allocate a new page.

{% highlight c++ linenos %}

struct Header
{
  Header * next;
};

template <typename T>
T * PagedPool<T>::allocate()
{
  if (!freeHead) {
    addPage();
  }

  auto t = freeHead;

  freeHead = (T *)((Header *)freeHead)->next;

  return t;
}

template <typename T>
void PagedPool<T>::addPage()
{
  // Allocate a new page
  pages.emplace_back();

  auto & page = pages.back();

  page.store.resize(pageSize * sizeof(T));

  auto items = reinterpret_cast<T *>(page.store.data());

  for (size_t i = 0; i < pageSize - 1; ++i) {
    union
    {
      T * item;
      Header * header;
    } u, next;

    u.item = &items[i];
    next.item = &items[i + 1];

    u.header->next = next.header;
  }

  if (!freeHead) {
    ((Header *)&items[pageSize - 1])->next = nullptr;
  } else {
    ((Header *)&items[pageSize - 1])->next = (Header *)freeHead;
  }

  freeHead = &items[0];
}

{% endhighlight %}

Figure:

![Paged pool diagram](/assets/diagrams/PagedPool.png)