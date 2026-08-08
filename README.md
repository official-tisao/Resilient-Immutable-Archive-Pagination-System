# Resilient Immutable Archive Pagination System

A pagination strategy for high-volume blogs, news platforms, and content archives designed to prevent historical pagination pages from becoming stale whenever new content is published.

## Author

**Author:** Saheed Oluwatosin Tiamiyu  
**Organization:** TISAO Technologies LLC  
**Website:** https://tisaotechnologies.com  
**LinkedIn:** https://www.linkedin.com/in/tiamiyu-saheed-oluwatosin/  
**GitHub:** https://github.com/official-tisao

**Immutable Archive Pagination** is an architectural strategy proposed and publicly documented by Saheed Oluwatosin Tiamiyu through TISAO Technologies LLC for constructing append-only, stable archive pagination using immutable post-ID boundaries.

---

## The Problem

Traditional pagination usually puts the newest content on page 1:

```text
/posts/page/1
/posts/page/2
/posts/page/3
...
```

Assume a news website has:

```text
20,000 posts
20 posts per page
1,000 archive pages
```

Publishing one new post changes page 1.

The previous last item from page 1 moves to page 2, page 2 pushes an item to page 3, and the shift may continue across the entire archive.

Conceptually:

```text
NEW POST
   ↓
Page 1 changes
   ↓
Page 2 changes
   ↓
Page 3 changes
   ↓
...
Page 1000 changes
```

Although only one article was published, hundreds or thousands of archive URLs may now contain different content.

This creates unnecessary:

- search-engine recrawling;
- CDN cache invalidation;
- SSR work;
- database queries;
- static-page regeneration;
- infrastructure load.

---

## The Idea

**Immutable Archive Pagination** reverses the physical pagination model.

Instead of:

```text
Page 1 = newest posts
```

it uses:

```text
Page 1 = earliest archive segment
Page 2 = next archive segment
Page 3 = next archive segment
...
Page N = newest archive segment
```

New content is appended only to the highest-numbered page.

For a page size of 20:

```text
Page 1  [20 posts] SEALED
Page 2  [20 posts] SEALED
Page 3  [20 posts] SEALED
Page 4  [ 7 posts] OPEN
                        ↑
                  new posts append here
```

Once Page 4 reaches 20 qualifying posts, it is permanently sealed.

The next qualifying post creates Page 5.

### Core invariant

> Once an archive page is sealed, future publications or deletions must never cause unrelated content to cross that page boundary.

---

## Stable ID Boundaries

The strategy assumes that posts have immutable, monotonically increasing IDs, such as:

```sql
id BIGINT AUTO_INCREMENT PRIMARY KEY
```

Instead of storing every page-to-post relationship, each archive page stores only:

```text
archive
page_number
start_post_id
end_post_id
```

Example:

```text
archive     page    start_id    end_id
---------------------------------------
tag:tinubu    1       10031       10076
tag:tinubu    2       10081       10142
tag:tinubu    3       10151       10203
tag:tinubu    4       10211       NULL
```

The final row represents the current OPEN page.

---

## Page Retrieval

A sealed archive page can be reconstructed using its ID boundaries and archive predicate:

```sql
SELECT p.*
FROM posts p
WHERE p.id >= :startId
  AND p.id <= :endId
  AND <archive predicate>
ORDER BY p.id ASC;
```

For example:

```sql
WHERE p.id BETWEEN :startId AND :endId
  AND tag = 'tinubu'
```

The IDs do not need to be consecutive.

If the global stream contains:

```text
10031 Tinubu
10032 Sports
10033 Business
10034 Tinubu
...
10076 Tinubu
```

and `10076` is the twentieth qualifying Tinubu post, the page boundary becomes:

```text
10031 → 10076
```

The archive predicate filters unrelated posts inside that range.

---

## Why No Page-Membership Table?

A conventional implementation could store:

```text
archive_page_id
post_id
```

for every archive entry.

At large scale this can create millions or billions of additional relationship records across:

```text
tags
categories
authors
topics
global feeds
```

This strategy instead derives membership from:

```text
ID boundary + archive predicate
```

Therefore the pagination metadata grows approximately with the number of **pages**, not the number of individual archive entries.

With one million qualifying articles and a page size of 20:

```text
1,000,000 articles
       ↓
50,000 page-boundary records
```

---

## Deletion

Historical pages are never rebalanced.

Suppose Page 12 contains 20 posts and one article is deleted.

It becomes:

```text
Page 12 → 19 visible posts
Page 13 → unchanged
Page 14 → unchanged
...
```

The system does **not** move a post from Page 13 into Page 12.

This keeps deletion localized to the page containing the deleted content.

---

## Latest Content

The physical archive numbering increases chronologically:

```text
1 → 2 → 3 → ... → N
oldest                newest
```

But users can still see newest content first.

For example:

```text
/t/tinubu
```

acts as the dynamic latest-content view.

If:

```text
Page 52 = 20 posts, SEALED
Page 53 = 7 posts, OPEN
```

the landing page can show:

```text
7 newest posts from Page 53
+
13 newest posts from Page 52
=
20 latest posts
```

User navigation can then move backward:

```text
Latest
  ↓
Page 52
  ↓
Page 51
  ↓
Page 50
```

---

## Cache Architecture

The model naturally creates two cache classes.

### Mutable

```text
/
/posts
/t/tinubu
/category/sports
/t/tinubu/page/{OPEN_PAGE}
```

These receive shorter TTLs and targeted invalidation.

### Sealed

```text
/t/tinubu/page/1
/t/tinubu/page/2
...
/t/tinubu/page/N-1
```

These pages can receive very long CDN retention because normal publication does not change them.

A sealed page can also be statically generated and removed from the normal SSR path:

```text
Search Bot / Browser
        ↓
       CDN
        ↓
Static sealed archive
```

---

## No Large OFFSET Queries

Traditional pagination may eventually require queries such as:

```sql
LIMIT 20 OFFSET 500000
```

Immutable Archive Pagination instead performs bounded indexed retrieval:

```sql
WHERE id BETWEEN :startId AND :endId
```

The system already knows where each historical page starts and ends.

---

## Complexity

Let:

```text
P = number of historical pagination pages
A = number of archive streams affected by a publication
```

Traditional newest-first pagination can logically invalidate:

```text
O(P)
```

archive pages after one publication.

Immutable Archive Pagination affects approximately:

```text
O(A)
```

For one archive stream, the historical-page impact is effectively:

```text
O(1)
```

regardless of whether the archive contains 100 pages or 1,000,000 pages.

---

## Architecture

```text
                   New Post
                      │
                      ▼
              Archive Evaluation
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
       Global      Category       Tag
       Archive      Archive      Archive
         │            │            │
         ▼            ▼            ▼
      OPEN PAGE    OPEN PAGE    OPEN PAGE
         │            │            │
         └──── reaches page size ──┘
                      │
                      ▼
                    SEALED
                      │
                      ▼
              Static Generation
                      │
                      ▼
                Object Storage
                      │
                      ▼
                     CDN
```

---

## Technical Documentation

The detailed system design, database model, deletion semantics, cache strategy, concurrency considerations, taxonomy handling, and implementation notes are available in:

**[TECHNICAL.md](./TECHNICAL.md)**

---

## Status

This repository currently documents the architecture and algorithm.

Future work may include:

- reference implementation;
- Java/Spring Boot implementation;
- benchmarks;
- traditional pagination comparison;
- CDN experiments;
- database performance tests;
- large-scale simulation.

---

## License

See [LICENSE](./LICENSE) and [NOTICE](./NOTICE) for licensing and attribution requirements.
