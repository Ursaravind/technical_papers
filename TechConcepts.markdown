# Technical Concepts Overview

## 1. Difference between `git pull` and `git fetch`
- **Git Fetch**: Downloads changes (commits, branches, tags) from a remote repository to your local repository but does not merge them into your working directory or current branch. You can review changes before deciding to merge.
- **Git Pull**: Combines `git fetch` and `git merge`. It downloads changes from the remote repository and immediately merges them into your current branch, updating your working directory.

## 2. What are Subqueries in SQL?
A subquery is a query nested within another SQL query, used to return data that will be used by the outer query. It is enclosed in parentheses and can be used in SELECT, WHERE, or FROM clauses to filter or compute results based on intermediate data.

## 3. What are Iterators?
Iterators are objects in programming (e.g., Python) that enable traversal through a sequence of elements, one at a time, without loading the entire sequence into memory. They implement `__iter__()` and `__next__()` methods to provide iteration functionality, often used in loops or comprehensions.

## 4. Flexbox in CSS
Flexbox (Flexible Box Layout) is a CSS layout model for arranging elements in a container. It allows responsive design by distributing space and aligning items dynamically. Key properties:
- `display: flex`: Enables flexbox on a container.
- `flex-direction`: Sets the main axis (row or column).
- `justify-content`: Aligns items along the main axis (e.g., `space-between`, `center`).
- `align-items`: Aligns items along the cross axis (e.g., `stretch`, `center`).
- `flex`: Defines how items grow, shrink, or maintain size.

## 5. Difference between `git merge` and `git rebase`
- **Git Merge**: Combines multiple branches by creating a merge commit, preserving the history of both branches. It maintains the original branch structure but can result in a more complex history.
- **Git Rebase**: Rewrites the commit history by moving the current branch’s commits onto the tip of another branch, creating a linear history. It avoids merge commits but alters history, which can complicate collaboration.

## 6. Python Memory Management and Garbage Collector
Python manages memory using a private heap for objects and reference counting to track object usage. When an object’s reference count drops to zero, it is deallocated. The **garbage collector** handles cyclic references (objects referencing each other) using a generational algorithm, periodically scanning for unreachable objects to free memory.

## 7. Difference between `localStorage` and `sessionStorage` in Web Development
- **localStorage**: Stores data with no expiration date; persists across browser sessions and tabs until explicitly cleared. Limited to ~5-10 MB, depending on the browser.
- **sessionStorage**: Stores data only for the duration of a single browser tab’s session; data is cleared when the tab closes. Also limited to ~5-10 MB.

## 8. What is `git stash`?
`git stash` temporarily saves uncommitted changes (modified tracked files and staged changes) in a stack, allowing you to switch branches or perform other tasks without committing. Use `git stash apply` to restore the changes and `git stash pop` to restore and remove them from the stash.

## 9. What are Generators in Python?
Generators are special iterators in Python that yield values one at a time using the `yield` keyword, allowing memory-efficient iteration over large datasets. Defined with a function or generator expression, they generate values lazily, pausing execution between yields, and cannot be reused once exhausted.